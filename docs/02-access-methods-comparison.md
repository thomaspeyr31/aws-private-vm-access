# Rapport — Accès aux VM AWS en réseau privé depuis macOS

## 1. Contexte

Ce rapport compare les méthodes permettant d'accéder, depuis un Mac, à des instances EC2 situées dans un sous-réseau privé d'un VPC AWS (sans IP publique). Pour chaque méthode : principe technique, prérequis, mise en œuvre, cas d'usage et sécurité.

## 2. Tableau de synthèse

| Méthode | Port ouvert sur internet | Authentification | Complexité | Coût relatif | Idéal pour |
|---|---|---|---|---|---|
| SSM Session Manager | Aucun | IAM | Faible | Gratuit | Accès admin régulier, 1 à N instances |
| EC2 Instance Connect Endpoint | Aucun | IAM | Faible | Très faible | SSH/RDP natif, sans agent |
| Bastion Host | 22/3389 (restreint par IP) | Clé SSH | Moyenne | Faible | Compatibilité avec outils SSH classiques |
| AWS Client VPN | Aucun | Certificat / SAML | Moyenne-élevée | Élevé | Accès réseau complet, plusieurs ressources |
| Site-to-Site VPN | Aucun | IPsec | Élevée | Moyen | Connecter un réseau entier (bureau/maison) |
| AWS Direct Connect | Aucun | Physique + IAM | Très élevée | Très élevé | Volumes niveau entreprise |
| VPN auto-géré (WireGuard) | Port VPN dédié | Clé pré-partagée | Moyenne | Faible | Contrôle total, budget réduit |

## 3. Détail par méthode

### 3.1 AWS Systems Manager – Session Manager

**Principe** — chaque connexion passe par l'API AWS SSM, authentifiée par IAM. Aucun port entrant à ouvrir.

```
Mac --(HTTPS + IAM)--> API SSM --(canal chiffré)--> Agent SSM sur la VM (subnet privé)
```

**Prérequis techniques**
- SSM Agent actif sur la VM (préinstallé sur les AMI Amazon Linux et Ubuntu récentes)
- Rôle IAM attaché à l'instance avec la policy managée `AmazonSSMManagedInstanceCore`
- Si le subnet n'a pas de route internet (pas de NAT Gateway) : 3 VPC Endpoints à créer — `com.amazonaws.<region>.ssm`, `ssmmessages`, `ec2messages`
- Côté Mac : AWS CLI v2 + plugin `session-manager-plugin`
- Droits IAM utilisateur : `ssm:StartSession`, `ssm:TerminateSession`, `ssm:ResumeSession`

**Mise en œuvre**
```bash
brew install awscli
brew install --cask session-manager-plugin

aws ssm start-session --target i-0123456789abcdef0
```

Pour utiliser un vrai client SSH (recommandé), dans `~/.ssh/config` :
```
Host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
```
Puis simplement : `ssh ubuntu@i-0123456789abcdef0`

Port forwarding (ex. atteindre du RDP sur une VM Windows) :
```bash
aws ssm start-session --target i-0123456789abcdef0 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["3389"],"localPortNumber":["9999"]}'
```

**Sécurité**
- Aucun port ouvert publiquement
- Contrôle d'accès fin par IAM (par instance, tag ou utilisateur)
- Traçabilité native : CloudTrail journalise chaque session ; le contenu peut être streamé vers CloudWatch Logs / S3, chiffrable par KMS
- Compatible avec une exigence MFA via condition IAM (`aws:MultiFactorAuthPresent`)

**Cas d'usage** — solution par défaut pour de l'administration ponctuelle ou régulière d'un petit nombre de VM.

### 3.2 EC2 Instance Connect Endpoint (EIC Endpoint)

**Principe** — un point de terminaison managé par AWS, créé dans le VPC, fait office de proxy pour tunneliser du SSH/RDP standard vers une VM sans IP publique, sans agent logiciel à installer.

```
Mac --(client SSH/RDP standard, tunnel via AWS)--> EIC Endpoint (dans le VPC) --> VM (subnet privé)
```

**Prérequis techniques**
- Créer un EC2 Instance Connect Endpoint dans un subnet du VPC
- Security Group de la VM cible : autoriser le trafic entrant depuis le Security Group de l'EIC Endpoint
- Droits IAM : `ec2-instance-connect:OpenTunnel`

**Mise en œuvre**
```bash
aws ec2 create-instance-connect-endpoint --subnet-id subnet-xxxxxxxx --security-group-ids sg-xxxxxxxx

aws ec2-instance-connect ssh --instance-id i-0123456789abcdef0 --connection-type eice
```

**Sécurité**
- Mêmes avantages réseau que SSM (zéro port public), sans dépendance à l'agent SSM
- Accès conditionné à IAM, journalisable via CloudTrail
- Tunnel temporaire, initié à la demande

**Cas d'usage** — alternative légère à SSM pour du SSH/RDP natif sans agent, ou quand SSM Agent n'est pas installé. À ne pas confondre avec l'« EC2 Instance Connect » historique, réservé aux instances avec IP publique.

### 3.3 Bastion Host (Jump Server)

**Principe** — une instance dans un subnet public sert de relais SSH/RDP vers les VM privées.

```
Mac --(SSH port 22)--> Bastion (subnet public, IP publique) --(SSH interne)--> VM (subnet privé)
```

**Prérequis techniques**
- Instance EC2 en subnet public, avec IP publique ou Elastic IP
- Security Group du bastion : port 22/3389 ouvert uniquement depuis l'IP de papa — jamais `0.0.0.0/0`
- Security Group de la VM privée : port 22/3389 ouvert uniquement depuis le Security Group du bastion

**Mise en œuvre**
```bash
ssh -J ubuntu@<ip-publique-bastion> ubuntu@<ip-privee-vm>
```

**Sécurité**
- Méthode la plus exposée : un hôte directement joignable depuis internet
- Durcissement indispensable : clé SSH uniquement (pas de mot de passe), fail2ban, mises à jour régulières
- Moins auditable nativement que SSM (pas de journalisation de session intégrée)

**Cas d'usage** — pertinent si des outils existants imposent un accès SSH classique. Sans contrainte particulière, SSM ou l'EIC Endpoint sont préférables aujourd'hui.

### 3.4 AWS Client VPN

**Principe** — service VPN managé (compatible OpenVPN). Une fois connecté, le Mac obtient une adresse dans le VPC et peut joindre toutes les ressources autorisées, pas seulement une VM.

```
Mac --(tunnel VPN chiffré TLS)--> Client VPN Endpoint --(routage VPC)--> VM(s), bases de données, etc.
```

**Prérequis techniques**
- Endpoint Client VPN associé à un ou plusieurs subnets
- Authentification : certificats mutuels (via ACM Private CA) ou fédération SAML/Active Directory
- Règles d'autorisation définissant les CIDR du VPC accessibles
- Client macOS : application officielle AWS Client VPN, ou OpenVPN/Tunnelblick

**Sécurité**
- Chiffrement TLS 1.2+ de bout en bout
- Peut être couplé à une MFA via le fournisseur d'identité SAML
- Option « split-tunnel » recommandée : seul le trafic vers le VPC transite par AWS

**Cas d'usage** — accès régulier à plusieurs ressources du VPC (plusieurs VM, une base de données interne, un outil web interne). Coût plus élevé (facturation à l'heure de connexion).

### 3.5 Site-to-Site VPN

Connecte un réseau entier (ex. le domicile de papa, avec un routeur compatible IPsec) au VPC, plutôt qu'un poste individuel.
- Nécessite un Customer Gateway (routeur/pare-feu IPsec : pfSense, Ubiquiti, Cisco…)
- Deux tunnels IPsec créés côté AWS pour la redondance
- Disproportionné pour un seul Mac ; pertinent seulement si plusieurs appareils du réseau domicile doivent accéder au VPC.

### 3.6 AWS Direct Connect

Liaison physique dédiée entre une infrastructure on-premise et AWS.
- Nécessite un point de présence Direct Connect ou un partenaire réseau
- Coût et délais de mise en place de niveau entreprise (plusieurs semaines)
- Mentionné pour exhaustivité ; sans intérêt pour l'usage décrit.

### 3.7 Alternative : VPN auto-géré (WireGuard/OpenVPN sur EC2)

Installer soi-même WireGuard ou OpenVPN sur une petite instance EC2.
- Avantage : coût très faible, contrôle total
- Inconvénient : maintenance entièrement à charge (mises à jour, clés, monitoring), pas de journalisation IAM/CloudTrail native
- Cas d'usage : profil à l'aise en administration système, cherchant à minimiser les coûts récurrents.

### 3.8 Cas avancé : accès multi-utilisateurs via EIC Endpoint (mode « salle de classe »)

Un seul EIC Endpoint peut servir de point d'entrée pour plusieurs utilisateurs, chacun restreint à sa propre VM — utile pour un environnement de formation ou de test partagé.

**Mécanisme d'isolation** — l'action `ec2-instance-connect:OpenTunnel` cible l'ARN de l'endpoint (pas celui de l'instance) ; l'accès à une VM précise se restreint via une condition sur les tags, pas via le champ `Resource`. Modèle ABAC (une seule policy pour tout le monde) :

```json
{
  "Effect": "Allow",
  "Action": ["ec2-instance-connect:OpenTunnel", "ec2-instance-connect:SendSSHPublicKey"],
  "Resource": "arn:aws:ec2:eu-west-1:111122223333:instance-connect-endpoint/eice-xxxxxxxx",
  "Condition": {
    "StringEquals": { "aws:ResourceTag/Etudiant": "${aws:PrincipalTag/Etudiant}" }
  }
}
```

Chaque utilisateur et sa VM assignée partagent le même tag ; une seule policy suffit pour tous.

**Bonus sécurité** — avec `aws ec2-instance-connect ssh --connection-type eice`, AWS génère et pousse une clé SSH éphémère à chaque connexion (`SendSSHPublicKey`). Aucune clé SSH à distribuer : tout est piloté par IAM.

**Réseau** — Security Group de l'endpoint autorisé sur chaque VM (une seule règle pour tout le groupe) ; ne pas autoriser les VM entre elles, pour éviter qu'un utilisateur rebondisse vers la VM d'un autre.

## 4. Normes et bonnes pratiques de sécurité

### 4.1 Contrôle d'accès (IAM)
- Principe du moindre privilège : policies restreintes à des instances/tags précis, jamais `*`
- Préférer des rôles IAM (identifiants temporaires STS) à des utilisateurs avec clés d'accès permanentes
- MFA activé sur le compte root et tout compte ayant accès aux VM

### 4.2 Réseau
- Ne jamais autoriser `0.0.0.0/0` sur les ports d'administration (22, 3389)
- Défense en profondeur : Security Groups (avec état, niveau instance) + NACL (sans état, niveau subnet)
- Isoler les VM sensibles dans des subnets sans route vers un Internet Gateway

### 4.3 Chiffrement
- SSM/EIC : chiffrement en transit, logs de session chiffrables via KMS
- SSH : clés Ed25519 ou RSA 4096 bits, authentification par mot de passe désactivée
- VPN : TLS 1.2 minimum, certificats renouvelés périodiquement

### 4.4 Journalisation et audit
- CloudTrail : historique des appels API (ouverture/fermeture de session, etc.)
- VPC Flow Logs : visibilité réseau
- Alertes CloudWatch/EventBridge sur connexions atypiques (horaires inhabituels, IP inconnue)

### 4.5 Référentiels reconnus
- **CIS AWS Foundations Benchmark** : recommandations sur IAM, journalisation, réseau
- **AWS Well-Architected Framework**, pilier Sécurité : gestion des identités, contrôles de détection, protection de l'infrastructure

## 5. Points pratiques complémentaires

- **IP dynamique** — le Bastion perd son intérêt sécurité si l'IP source change (FAI résidentiel) ou en cas de connexion depuis un autre lieu. SSM, EIC Endpoint et Client VPN reposent sur une identité (IAM/certificat), pas sur une IP source.
- **Coût dans la durée** — SSM et EIC Endpoint sont gratuits ; Client VPN et Site-to-Site sont facturés à l'heure, même sans connexion active.
- **Combiner les méthodes** — rien n'empêche d'utiliser SSM en usage principal et de garder un Bastion en secours.
- **Prérequis souvent oublié** — pour SSM/EIC, un profil AWS configuré côté Mac (`aws configure`) avec des identifiants IAM personnels est nécessaire, en plus des outils eux-mêmes.

## 6. Recommandation

Pour une administration occasionnelle de quelques VM depuis un Mac personnel :

1. **SSM Session Manager** en premier choix — gratuit, aucun port exposé, traçabilité native
2. **EIC Endpoint** si l'agent SSM n'est pas souhaité — SSH natif tout aussi sécurisé
3. **Bastion** seulement en cas de contrainte de compatibilité avec des outils existants
4. **Client VPN** si l'accès doit s'étendre régulièrement à plusieurs ressources du VPC
