<div id="top">

<!-- STYLE D'EN-TÊTE : CLASSIQUE -->
<div align="center">

<img src="Browsermania.png" width="30%" style="position: relative; top: 0; right: 0;" alt="Logo du projet"/>

# BROWSERMANIA

<em>Libérez votre parcours  avec une orchestration sans effort.</em>

<!-- BADGES -->
<img src="https://img.shields.io/github/last-commit/sony-level/Browsermania?style=flat&logo=git&logoColor=white&color=0080ff" alt="dernier-commit">
<img src="https://img.shields.io/github/languages/top/sony-level/Browsermania?style=flat&color=0080ff" alt="langage-principal-repo">
<img src="https://img.shields.io/github/languages/count/sony-level/Browsermania?style=flat&color=0080ff" alt="nombre-langages-repo">

<em>Construit avec les outils et technologies :</em>

<img src="https://img.shields.io/badge/Markdown-000000.svg?style=flat&logo=Markdown&logoColor=white" alt="Markdown">
<img src="https://img.shields.io/badge/GNU%20Bash-4EAA25.svg?style=flat&logo=GNU-Bash&logoColor=white" alt="GNU%20Bash">
<img src="https://img.shields.io/badge/YAML-CB171E.svg?style=flat&logo=YAML&logoColor=white" alt="YAML">

</div>
<br>

---

## 📄 Table des matières

- [Présentation](#-présentation)
- [Démarrage rapide](#-démarrage-rapide)
    - [Prérequis](#-prérequis)
    - [Installation](#-installation)
    - [Utilisation](#-utilisation)
    - [Tests](#-tests)

---

## ✨ Présentation

Browsermania est votre outil incontournable pour lancer et gérer facilement des clusters Kubernetes avec des technologies de pointe.

**Pourquoi Browsermania ?**

Ce projet simplifie le déploiement d'environnements d'orchestration de conteneurs robustes, permettant aux développeurs de se concentrer sur la création d'applications plutôt que sur la gestion de l'infrastructure. Les fonctionnalités principales incluent :

- 🚀 **Installation automatisée de Kubernetes :** Simplifie la configuration et réduit les erreurs manuelles.
- 🛡️ **Intégration de Containerd et Cilium :** Améliore les performances et la sécurité avec un runtime moderne et des solutions réseau avancées.
- ⚖️ **Répartition de charge avec MetalLB :** Gère efficacement la distribution du trafic et l'attribution d'IP pour vos applications.
- 💾 **Stockage persistant pour MySQL :** Garantit la persistance des données pour les applications à état, essentiel pour la fiabilité.
- 🌐 **Déploiement complet de la stack applicative :** Déploie rapidement une API backend et une application frontend, accélérant la productivité des développeurs.

---

## 🚀 Démarrage rapide

### 📋 Prérequis

Ce projet nécessite les dépendances suivantes :

- **Langage de programmation :** Shell
- **Gestionnaire de paquets :** Bash

### ⚙️ Installation

Construisez Browsermania à partir des sources et installez les dépendances :

1. **Clonez le dépôt :**

    ```sh
    ❯ git clone https://github.com/sony-level/Browsermania
    ```

2. **Accédez au dossier du projet :**

    ```sh
    ❯ cd Browsermania
    ```

3. **Installez les dépendances :**

**Avec [bash](https://www.gnu.org/software/bash/) :**

```sh
❯ chmod +x launch.sh
````
### Script de configuration d'un cluster Kubernetes avec Containerd, Cilium et Cilium CLI

Ce script shell configure un nœud (maître ou worker) pour un cluster Kubernetes basé sur Containerd avec le CNI Cilium.

## 🛠️ Pré-requis

- Ubuntu/Debian
- kata-container installer
- Droits `sudo`

## 📜 Script Bash

```bash
#!/bin/bash
# Script pour configurer un cluster Kubernetes avec containerd, Cilium et Cilium CLI

# Exécuter sur tous les nœuds
echo "Mise à jour du système et installation des dépendances..."
sudo apt update && sudo apt upgrade -y
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo apt install -y curl apt-transport-https ca-certificates software-properties-common

echo "Activation de l'IP forwarding..."
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

echo "Installation de containerd..."
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

echo "Ajout du dépôt Kubernetes..."
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update

echo "Installation de kubeadm, kubelet et kubectl..."
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

echo "Configuration de kubelet pour utiliser containerd..."
cat <<EOF | sudo tee /etc/default/kubelet
KUBELET_EXTRA_ARGS=--container-runtime-endpoint=unix:///var/run/containerd/containerd.sock
EOF

sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable kubelet
sudo systemctl restart kubelet

echo "Installation de Cilium CLI..."
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Initialisation uniquement sur le nœud maître
if [ "$1" == "master" ]; then
  echo "Initialisation du nœud maître Kubernetes..."
  sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket unix:///var/run/containerd/containerd.sock

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
fi

```

Une fois ce script executé et le cluster initialisé, lancer :
```bash
    ❯ cilium install
    ❯ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml (ou une version plus récente)
    ❯ kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
puis appliquer le fichier de deploiement .yaml.