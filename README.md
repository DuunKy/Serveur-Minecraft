<div align="center">
  <img src="https://capsule-render.vercel.app/api?type=waving&color=gradient&customColorList=76C828,4CAF50,2E7D32&height=250&section=header&text=Mon%20Serveur%20Minecraft&fontSize=60&animation=fadeIn&fontAlignY=38&desc=Sécurité%20%E2%80%A2%20Conteneurisé%20%E2%80%A2%20Automatisé&descAlignY=60&descSize=20&fontColor=ffffff" alt="Header Minecraft Nature" />
</div>

<div align="center">
  <a href="https://git.io/typing-svg">
    <img src="https://readme-typing-svg.demolab.com?font=VT323&weight=500&size=38&pause=800&color=33FF33&center=true&vCenter=true&random=false&width=850&height=100&lines=Init+Secure+Block+Protocol...;Loading+WireGuard+Tunnel...%5BOK%5D;Applying+OpnSense+Rules...%5BOK%5D;Mounting+Stateful+Volumes...%5BOK%5D;%3E%3E+SECURE+HYBRID+INFRA+READY+%3C%3C" alt="Typing SVG" />
  </a>
  <br>
</div>


<div align="center">
  <img src="https://img.shields.io/badge/WireGuard-88171A?style=for-the-badge&logo=wireguard&logoColor=white" alt="WireGuard" />
  &nbsp; <img src="https://img.shields.io/badge/OpnSense-D94F00?style=for-the-badge&logo=opnsense&logoColor=white" alt="OpnSense" />
  &nbsp;
  <img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker" />
  &nbsp;
  <img src="https://img.shields.io/badge/Debian-A81D33?style=for-the-badge&logo=debian&logoColor=white" alt="Debian" />
</div>


## 📋 Aperçu

Ce projet démontre la mise en œuvre d'une **infrastructure hybride sécurisée** (Cloud + On-Premise) pour l'hébergement de services stateful critiques.
L'objectif principal est de créer un serveur minecraft entre ami en exposant un service hébergé localement (HomeLab) tout en masquant l'IP résidentielle et en appliquant une politique de **Zero Trust** sur les accès administratifs.

L'architecture repose sur un tunnel chiffré traversant un pare-feu périmétrique, avec une ségrégation stricte des flux via un **Bastion SSH**.

## 🏗️ Architecture de mon réseau

```mermaid
graph LR
    subgraph Cloud ["☁️ Cloud (VPS)"]
        PublicIP[("🌐 Public IP")]
        WG_S["WireGuard Server"]
        PublicIP --> WG_S
    end

    subgraph Home ["🏠 Mon HomeLab"]
        OpnSense["🔥 OpnSense (Firewall)"]
        
        subgraph DMZ ["🔒 Security Zone"]
            Bastion["🛡️ SSH Bastion\n(Fail2Ban + Key Only)"]
        end
        
        subgraph AppNet ["🎮 App Network"]
            Debian["🐧 Debian Server"]
            Docker[["🐳 Docker (Minecraft)"]]
        end
    end

    %% Flux de données
    User(("Joueur")) -- "TCP/UDP 25565" --> PublicIP
    Admin(("Admin")) -- "SSH (Port 22)" --> PublicIP

    %% Connexions internes
    WG_S == "VPN Tunnel (Encrypted)" ==> OpnSense
    
    %% Routing du jeu
    OpnSense -.->|"Port Forwarding"| Docker
    
    %% Routing Admin
    OpnSense --> Bastion
    Bastion -->|"SSH Jump"| Debian

    classDef cloud fill:#e1f5fe,stroke:#01579b,stroke-width:2px;
    classDef home fill:#fff3e0,stroke:#e65100,stroke-width:2px;
    classDef sec fill:#ffebee,stroke:#b71c1c,stroke-width:2px;
    
    class Cloud cloud;
    class Home home;
    class Bastion sec;
```
## Stateful Management (Les données persistentes)

1. **Isolation du volume** : Les données (data/world etc.), les logs et les configurations du serveur sont découplés/séparé/indépendant du conteneur via des **Docker Volumes**. Le conteneur est éphémère mais la donnée elle est persistante.

2. **Backup**

La stratégie de sauvegarde repose sur un modèle "Pull" (l'hôte récupère les données) pour garantir qu'une compromission du serveur n'affecte pas l'intégrité des archives existantes. (Car j'ai déjà eu un problème de corruption et c'est pas cool)

### 🔄 Comment est fait la sauvegarde !

```mermaid
sequenceDiagram
    participant Cron as 🕒 Planificateur (Cron Linux)
    participant Script as 📜 Script Bash
    participant MC as 🎮 Serveur Minecraft
    participant Host as 💻 Hôte Windows (PowerShell)

    Note over Cron, MC: 1. Préparation & Intégrité (Côté Serveur)
    Cron->>Script: Déclenchement Backup (Toutes les 2h)
    Script->>MC: RCON "save-off" (Arrêt écriture disque)
    Script->>MC: RCON "save-all" (Force sauvegarde RAM->Disque)
    Script->>Script: Compression des Données (ZIP/TAR)
    Script->>MC: RCON "save-on" (Reprise écriture)

    Note over Host, Script: 2. Récupération Sécurisée (Côté Client "Pull")
    Host->>Script: Requête SCP (Port 22 SSH)
    Script-->>Host: Authentification (Clé SSH Ed25519)
    Script-->>Host: Transfert de l'archive chiffrée
    Host->>Host: Vérification & Stockage (Disque Local)
```
Je mettrai les petit script dans le repo bien sûr !

## 🔐 Durcissement de la Sécurité (Défense en Profondeur)

### 1. Segmentation Réseau
* **Tunneling VPS (Ingress / Entrant) :** L'adresse IP publique du domicile n'est jamais exposée directement sur Internet. Le VPS agit comme un "fusible" (point d'entrée jetable) et masque l'infrastructure réelle.
* **Isolation (VLAN) :** Le serveur Debian est strictement cloisonné du reste du réseau domestique via des règles de pare-feu **OpnSense**. En cas de compromission du serveur de jeu, le réseau personnel reste protégé.

### 2. Contrôle d'Accès (Bastion SSH)
* **Zero Direct Access :** Aucun accès SSH direct n'est autorisé sur le serveur d'application depuis l'extérieur.
* **Bastion Durci :** Toute administration passe obligatoirement par un serveur rebond (*Jump Server*).
    * Authentification par clés cryptographiques **Ed25519** uniquement.
    * `PermitRootLogin no` : Connexion directe en tant que *root* désactivée.

### 3. Sécurité des Conteneurs (Docker Security)
* **User Namespace Remapping :** Le processus Docker s'exécute avec un UID spécifique (**non-root**). Cela limite considérablement l'impact sur l'hôte en cas d'évasion de conteneur (*Container Breakout*).
* **Resource Limits :** Des quotas stricts (CPU & RAM) sont définis dans le `docker-compose.yml` pour empêcher tout déni de service (DoS) qui pourrait surcharger la machine hôte.


## Les technologies utilisés

| Couche (Layer) | Technologie | Rôle & Usage Spécifique |
| :--- | :--- | :--- |
| **🌐 Ingress / Réseau** | **WireGuard** | Tunneling chiffré (VPN) Site-to-Site pour masquer l'IP réelle. |
| **🔥 Securité périmétrique** | **OpnSense** | Pare-feu, NAT, VLAN Tagging et Inspection de paquets (DPI). |
| **🐧 Système d'exploitation** | **Debian / Linux** | Hôte du serveur d'application (Optimisé & Durci). |
| **🐳 Conteneurisation** | **Docker & Compose** | Orchestration du service Minecraft et isolation des processus. |
| **🛡️ Contrôle d'accès** | **OpenSSH** | Accès sécurisé par clés **Ed25519** (Password Auth désactivé). |
| **🤖 Automatisation (Server)** | **Bash & Cron** | Script de sauvegarde : Freeze I/O (`save-off`), Compression & Rotation. |
| **⚡ Automatisation (Client)** | **PowerShell** | Script "Pull" : Récupération sécurisée via **SCP** vers l'hôte Windows. |
| **🎮 Protocole** | **RCON** | Communication directe avec le serveur de jeu pour la gestion d'état (Save/Stop). |

## 🔧 Problèmes & Troubleshooting

Ce projet a nécessité de résoudre plusieurs problématiques techniques liées à l'encapsulation réseau et à la persistance des données.

### 1. Network Fragmentation (WireGuard MTU)
* **Problème :** Instabilité de la connexion et perte de paquets (packet loss) observée lors du passage dans le tunnel VPN. Certains paquets de jeu (UDP) étaient droppés.
* **Analyse :** Le tunnel WireGuard ajoute un *overhead* (en-tête) aux paquets. Avec un MTU par défaut de 1500 (Ethernet standard), les paquets encapsulés dépassaient la taille limite, causant de la fragmentation ou du rejet.
* **Solution :** Calcul et ajustement du **MTU (Maximum Transmission Unit)** à `1420`  sur l'interface WireGuard pour laisser de la place aux en-têtes du protocole.
    ```bash
    # Exemple de fix dans wg2.conf
    MTU = 1420
    ```

### 2. Corruption de données & Atomicité (l'invisible)
* **Problème :** Lors des premiers tests, copier le dossier `/world` pendant que le serveur tournait résultait en des "chunks" corrompus (fichiers ouverts ou en cours d'écriture).
* **Solution :** Implémentation du mécanisme de **Safe-Backup**. Le script force le serveur à vider sa RAM sur le disque (`save-all`) et bloque l'écriture (`save-off`) le temps de la compression `tar.gz`.

### 3. Config Management (JSON & Mods)
* **Problème :** Conflits d'IDs et incompatibilités entre certains mods, empêchant le démarrage du conteneur.
* **Intervention Dev :** * Analyse des logs de crash (Stack Traces Java).
    * Développement de correctifs dans les fichiers `.json` de configuration pour harmoniser les dépendances.
    * Création d'un environnement "Custom" optimisé pour nos besoins spécifiques, plutôt qu'un modpack générique. (changement taux de drops/ spawn de certains mobs dans des biomes etc.)

Merci d'avoir pris le temps de lire !