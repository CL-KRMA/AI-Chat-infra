# Cluster K3s sur AWS avec Terraform

Ce projet Terraform configure une infrastructure complète pour déployer un cluster Kubernetes K3s sur AWS avec support Ollama (llama3, llava, gemma4).

## 📋 Table des matières

- [Vue d'ensemble](#vue-densemble)
- [Architecture](#architecture)
- [Prérequis](#prérequis)
- [Installation](#installation)
- [Configuration](#configuration)
- [Déploiement](#déploiement)
- [Accès au cluster](#accès-au-cluster)
- [Installation K3s](#installation-k3s)
- [Ressources créées](#ressources-créées)
- [Maintenance](#maintenance)
- [Nettoyage](#nettoyage)

## 🎯 Vue d'ensemble

Ce projet Terraform déploie un cluster Kubernetes K3s sur AWS composé de :
- **1 nœud Master** (contrôleur du cluster + Ollama)
- **2 nœuds Worker** (exécution des workloads applicatifs)

Tous les nœuds utilisent des instances EC2 **r5.xlarge** (4 vCPU, 32GB RAM) avec **50 GB** de stockage SSD chiffré (gp3) et bénéficient d'adresses IP élastiques (EIP) fixes.

### Pourquoi r5.xlarge ?

Le nœud master fait tourner Ollama avec 3 modèles IA :

| Modèle | RAM utilisée |
|---|---|
| llama3:8b | ~5GB |
| llava:7b | ~4GB |
| gemma4:12b | ~7.6GB |
| OS + K3s + Ollama | ~4GB |
| **Total** | **~21GB** |

Le `r5.xlarge` avec 32GB RAM offre 11GB de marge confortable.

### En clair, ce code fait quoi ?

**Il crée automatiquement 3 serveurs dans le cloud AWS prêts à fonctionner ensemble comme un cluster Kubernetes.**

À l'exécution, le code :
1. **Crée un réseau privé** (VPC avec DNS activé) pour que les 3 serveurs communiquent entre eux
2. **Crée 3 serveurs Ubuntu** avec chacun 50 GB de disque chiffré
3. **Configure les règles de sécurité** pour autoriser le trafic nécessaire (SSH, ports Kubernetes, HTTP/HTTPS)
4. **Ajoute une clé SSH** pour se connecter en ligne de commande
5. **Assigne des adresses IP élastiques fixes** — les IPs ne changent pas après redémarrage

## 🏗️ Architecture

```
Internet
    │
    ▼
Internet Gateway
    │
    ▼
┌─────────────────────────────────────────────┐
│  VPC (192.168.2.0/24) — DNS activé          │
│                                              │
│  ┌────────────────────────────────────────┐ │
│  │  Subnet public (us-east-1a)            │ │
│  │                                        │ │
│  │  ┌──────────────────────────────────┐  │ │
│  │  │ Master (192.168.2.10)            │  │ │
│  │  │ r5.xlarge — 32GB RAM             │  │ │
│  │  │ Ollama + K3s control-plane       │  │ │
│  │  └──────────────────────────────────┘  │ │
│  │                                        │ │
│  │  ┌──────────────────────────────────┐  │ │
│  │  │ Worker-1 (192.168.2.11)          │  │ │
│  │  │ r5.xlarge — 32GB RAM             │  │ │
│  │  │ Workloads applicatifs            │  │ │
│  │  └──────────────────────────────────┘  │ │
│  │                                        │ │
│  │  ┌──────────────────────────────────┐  │ │
│  │  │ Worker-2 (192.168.2.12)          │  │ │
│  │  │ r5.xlarge — 32GB RAM             │  │ │
│  │  │ Workloads applicatifs            │  │ │
│  │  └──────────────────────────────────┘  │ │
│  └────────────────────────────────────────┘ │
│                                              │
│  Security Group (K3s + HTTP/HTTPS)           │
│  3 × Elastic IPs fixes                       │
└─────────────────────────────────────────────┘
```

## 📦 Prérequis

### Outils nécessaires

- **Terraform** >= 1.0 ([Installation](https://www.terraform.io/downloads.html))
- **AWS CLI** >= 2.0 ([Installation](https://aws.amazon.com/fr/cli/))
- **Compte AWS** avec permissions suffisantes
- **Clé SSH** générée localement

### Permissions AWS requises

Votre utilisateur AWS doit avoir accès à :
- EC2 (instances, security groups, key pairs, EIP)
- VPC (VPC, subnets, internet gateways, route tables)

## 🚀 Installation

### 1. Cloner le projet

```bash
git clone https://github.com/CL-KRMA/aws-terraform-config
cd aws-terraform-config
```

### 2. Générer une clé SSH

```bash
mkdir -p keys
ssh-keygen -t rsa -b 4096 -f keys/ma-cle-ssh -N ""
```

Cette commande crée :
- `keys/ma-cle-ssh` (clé privée — **ne jamais partager ni commiter**)
- `keys/ma-cle-ssh.pub` (clé publique — utilisée par Terraform)

> Assure-toi que `keys/` est dans ton `.gitignore`

### 3. Configurer les credentials AWS

```bash
aws configure
```

Entrez :
- AWS Access Key ID
- AWS Secret Access Key
- Region : `us-east-1`
- Format de sortie : `json`

## ⚙️ Configuration

### Variables disponibles

| Variable | Valeur par défaut | Description |
|---|---|---|
| `region` | `us-east-1` | Région AWS |
| `instance_type` | `r5.xlarge` | Type d'instance EC2 |
| `ami_id` | `ami-08c40ec9ead489470` | Ubuntu 22.04 LTS |
| `volume_size` | `50` | Taille du disque en GB |
| `key_name` | `ma-cle-ssh` | Nom de la clé SSH |
| `project_name` | `k3s-cluster` | Préfixe des ressources AWS |
| `allowed_ssh_cidr` | `0.0.0.0/0` | CIDR autorisé pour SSH |

> **Sécurité** : En production, remplace `allowed_ssh_cidr` par ton IP (`x.x.x.x/32`) pour restreindre l'accès SSH.

### Personnalisation via terraform.tfvars

```hcl
# terraform.tfvars
project_name     = "mon-projet"
allowed_ssh_cidr = "203.0.113.0/32"  # ton IP uniquement
```

## 🎬 Déploiement

### Étape 1 : Initialiser Terraform

```bash
terraform init
```

### Étape 2 : Planifier le déploiement

```bash
terraform plan -out=tfplan
```

Vérifie que **19 ressources** vont être créées.

### Étape 3 : Appliquer la configuration

```bash
terraform apply tfplan
```

Comptez **5-10 minutes** pour que tout soit opérationnel.

### Étape 4 : Récupérer les informations

```bash
terraform output
```

Exemple de résultat :
```
nodes = {
  master   = { private_ip = "192.168.2.10", public_ip = "54.x.x.x" }
  worker-1 = { private_ip = "192.168.2.11", public_ip = "54.x.x.x" }
  worker-2 = { private_ip = "192.168.2.12", public_ip = "54.x.x.x" }
}
cluster_summary = {
  master  = "54.x.x.x"
  workers = ["54.x.x.x", "54.x.x.x"]
  region  = "us-east-1"
}
```

## 🔌 Accès au cluster

```bash
# Master
ssh -i keys/ma-cle-ssh ubuntu@<MASTER_PUBLIC_IP>

# Worker 1
ssh -i keys/ma-cle-ssh ubuntu@<WORKER1_PUBLIC_IP>

# Worker 2
ssh -i keys/ma-cle-ssh ubuntu@<WORKER2_PUBLIC_IP>
```

## ☸️ Installation K3s

### Sur le master

```bash
ssh -i keys/ma-cle-ssh ubuntu@<MASTER_PUBLIC_IP>

curl -sfL https://get.k3s.io | sh -

# Vérifier que K3s tourne
sudo kubectl get nodes

# Récupérer le token pour les workers
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Sur chaque worker

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.2.10:6443 \
  K3S_TOKEN=<TOKEN> sh -
```

### Vérifier le cluster

```bash
# Depuis le master
sudo kubectl get nodes

# Résultat attendu
# NAME       STATUS   ROLES                  AGE
# master     Ready    control-plane,master   3m
# worker-1   Ready    <none>                 2m
# worker-2   Ready    <none>                 1m
```

### Configurer kubectl en local

```bash
# Depuis le master, copier le kubeconfig
sudo cat /etc/rancher/k3s/k3s.yaml

# En local — remplace 127.0.0.1 par l'IP publique du master
mkdir -p ~/.kube
nano ~/.kube/config

# Tester
kubectl get nodes
```

## 📦 Ressources créées

### Réseau
| Ressource | Description |
|---|---|
| `aws_vpc.main` | VPC avec DNS activé |
| `aws_subnet.public` | Subnet public us-east-1a |
| `aws_internet_gateway.main` | Accès internet |
| `aws_route_table.public` | Routage vers IGW |

### Sécurité — ports ouverts
| Port | Protocole | Source | Usage |
|---|---|---|---|
| 22 | TCP | `allowed_ssh_cidr` | SSH |
| ICMP | — | `allowed_ssh_cidr` | Ping |
| 6443 | TCP | VPC interne | K3s API Server |
| 8472 | UDP | VPC interne | K3s VXLAN (Flannel) |
| 10250 | TCP | VPC interne | Kubelet |
| 30000-32767 | TCP | 0.0.0.0/0 | NodePorts |
| 80 | TCP | 0.0.0.0/0 | HTTP |
| 443 | TCP | 0.0.0.0/0 | HTTPS |

### Instances EC2
| Nœud | IP privée | Type | Disque | Rôle |
|---|---|---|---|---|
| master | 192.168.2.10 | r5.xlarge | 50GB gp3 chiffré | control-plane + Ollama |
| worker-1 | 192.168.2.11 | r5.xlarge | 50GB gp3 chiffré | worker |
| worker-2 | 192.168.2.12 | r5.xlarge | 50GB gp3 chiffré | worker |

> Chaque instance a une **Elastic IP fixe** — l'IP ne change pas après redémarrage.

## 🔧 Maintenance

### Voir l'état actuel

```bash
terraform show
terraform output
```

### Modifier une ressource

```bash
# Éditer main.tf puis
terraform plan
terraform apply
```

### Arrêter les instances pour économiser

```bash
aws ec2 stop-instances --instance-ids <ID1> <ID2> <ID3>
```

> Les EIP attachées à des instances arrêtées sont gratuites.

## 🗑️ Nettoyage

```bash
terraform destroy
```

Confirme en tapant `yes`. Toutes les ressources sont supprimées.

## 📊 Coûts estimés

| Ressource | Quantité | Coût/mois |
|---|---|---|
| r5.xlarge on-demand | 3 | ~$720 |
| EIP attachées | 3 | $0 |
| gp3 50GB | 3 | ~$12 |
| **Total** | | **~$732/mois** |

> 💡 **Pour réduire les coûts** : utilise des **Spot Instances** (économie de 60-70%, soit ~$290/mois) ou éteins les instances quand tu ne les utilises pas.

## 🐛 Dépannage

### "Permission denied (publickey)"
```bash
chmod 600 keys/ma-cle-ssh
```

### "AWS credentials not found"
```bash
aws configure
```

### Les workers ne rejoignent pas le master
- Vérifie que le DNS est activé sur le VPC
- Vérifie que le port 6443 est accessible depuis le subnet interne
- Vérifie que le token est correct

### Les instances mettent longtemps à démarrer
Normal — AWS a besoin de 5-10 minutes pour initialiser les ressources.

## 📚 Ressources

- [Documentation Terraform AWS](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Documentation K3s](https://docs.k3s.io/)
- [Documentation Ollama](https://ollama.com/library)
- [Pricing AWS EC2](https://aws.amazon.com/fr/ec2/pricing/on-demand/)

## 📝 Licence

Ce projet est libre d'utilisation.

## 👤 Auteur

Créé pour un cluster K3s avec Ollama — Juin 2026
