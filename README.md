# TP Virtualisation – Infrastructure ESXi Imbriquée avec vCenter

Configuration complète : ESXi, vCenter, pfSense, Active Directory

---

## Table des matières

1. Prérequis matériels et logiciels
2. Objectif du TP
3. Architecture
4. Préparation de l'hôte
5. Création de la VM ESXi imbriqué
6. Déploiement vCenter Server (VCSA)
7. Déploiement pfSense
8. Déploiement Windows Server 2022 (AD + DNS)
9. Intégration du Client Windows 10
10. Serveur Web DMZ (Ubuntu Server)
11. Tests finaux et validation
12. VMotion - Migration à chaud (Optionnel)
13. Gestion des Snapshots
14. Bonnes pratiques TP imbriqué
15. Plan d'adressage récapitulatif

---

## 1. Prérequis matériels et logiciels

### 1.1 Matériel nécessaire

| Composant    | Minimum               | À l'aise                  |
|--------------|-----------------------|---------------------------|
| Processeur   | Intel i5 / AMD Ryzen 5 | Intel i7 / Ryzen 7       |
| RAM          | 16 Go                 | 32 Go (recommandé)        |
| Disque dur   | 200 Go libre          | 500 Go SSD                |
| Réseau       | Carte Ethernet        | —                         |

### 1.2 Logiciels requis

- VMware Workstation Pro 17+
- ISO VMware ESXi 8.x
- ISO vCenter Server Appliance 8.x (VCSA)
- ISO pfSense 2.7+
- ISO Windows Server 2022
- ISO Windows 10/11 ou Ubuntu Desktop
- ISO Ubuntu Server 22.04 ou Debian 12

### 1.3 Connaissances préalables

- Concepts de virtualisation
- Administration Windows Server (AD, DNS)
- Administration Linux (Apache/Nginx)
- Réseaux : VLANs, segmentation, firewall
- VMware Workstation

---

## 2. Objectif du TP

Mettre en place une infrastructure d'entreprise complète, comprenant :

- ESXi imbriqué (hyperviseur dans Workstation)
- vCenter Server
- pfSense (firewall, segmentation)
- Windows Server (AD + DNS + File)
- Machine client (Windows ou Linux)
- Serveur web DMZ (Linux)

> **Note :** Les VM internes seront créées dans VMware Workstation, pas directement dans ESXi, pour contourner la nested virtualization.

---

## 3. Architecture

### 3.1 Schéma réseau logique
```
                     Internet
                        │
                        │
                   ┌────▼────┐
                   │ pfSense │
                   │  WAN    │ (NIC1 - NAT)
                   └────┬────┘
                        │
            ┌───────────┴───────────┐
            │                       │
       ┌────▼────┐             ┌────▼────┐
       │   LAN   │             │   DMZ   │
       │ VLAN 20 │             │ VLAN 30 │
       └────┬────┘             └────┬────┘
            │                       │
       ┌────▼────┐             ┌────▼────┐
       │WinServer│             │ Web DMZ │
       │ + Client│             │ Ubuntu  │
       └─────────┘             └─────────┘
```

- NIC1 (NAT) → Management (ESXi + vCenter) et accès Internet
- NIC2 (Host-Only) → Production interne (LAN/DMZ)

### 3.2 Plan d'adressage IP

| Zone       | Réseau           | VLAN | IP Type  | Exemple                                        |
|------------|------------------|------|----------|------------------------------------------------|
| Management | 192.168.140.0/24 | 10   | Statique | ESXi 192.168.140.150 / vCenter 192.168.140.155 |
| LAN        | 192.168.20.0/24  | 20   | Statique | Windows Server 192.168.20.10 / Client 192.168.20.20 |
| DMZ        | 192.168.30.0/24  | 30   | Statique | Web 192.168.30.10                              |
| WAN        | DHCP / NAT       | 99   | DHCP     | pfSense WAN                                    |

---

## 4. Préparation de l'hôte

### 4.1 BIOS / Virtualisation

- Activer VT-x (Intel) ou AMD-V (AMD)
- Enregistrer et redémarrer

### 4.2 VMware Workstation

- Installer Workstation Pro 17+
- Créer dossier `C:\ISO\` et y placer tous les ISO nécessaires

---

## 5. Création de la VM ESXi imbriqué

### 5.1 Paramètres VM

| Paramètre           | Valeur                                                  |
|---------------------|---------------------------------------------------------|
| Nom                 | ESXi-TP-Entreprise                                      |
| Type                | Custom (Advanced)                                       |
| Guest OS            | VMware ESXi 8.x                                         |
| Firmware            | UEFI                                                    |
| CPU                 | 4 vCPU                                                  |
| RAM                 | 16 Go                                                   |
| Disques             | 50 Go (OS) + 200 Go (Datastore)                         |
| NIC 1               | NAT (Management)                                        |
| NIC 2               | Host-Only (Production)                                  |
| ISO                 | VMware-ESXi-8.x.iso                                     |
| Nested virtualization | vhv.enable=TRUE, hypervisor.cpuid.v0=FALSE            |

### 5.2 Installation d'ESXi

1. Monter ISO et démarrer la VM
2. Installer ESXi sur Disque 1 (50 Go)
3. Définir mot de passe root : `Admin!2026`
4. Configurer Management Network sur NIC1 (`192.168.140.150`)
5. Configurer DNS : `192.168.140.2`, `8.8.8.8`
6. Hostname : `esxi01.entreprise.local`

> **📸 Capture 1.1** – Page de connexion ESXi (`https://192.168.140.150`)

![Capture 1.1 - Page de connexion ESXi](captures/1.1-esxi-login.png)

> **📸 Capture 1.2** – Dashboard ESXi avec version 8.x visible

![Capture 1.2 - Dashboard ESXi](captures/1.2-esxi-dashboard.png)

> **📸 Capture 1.4** – Configuration réseau Management (IP 192.168.140.150)

![Capture 1.4 - Configuration réseau ESXi](captures/1.4-esxi-network.png)

### 5.3 Datastore

Ajouter Disque 2 (200 Go) comme datastore. Dans ESXi Web UI :
```
Storage → Datastores → New Datastore → VMFS → Sélectionner Disque 2 → Nom : datastore-VM
```

> **📸 Capture 1.3** – Datastore `datastore-VM` visible (200 Go)

![Capture 1.3 - Datastore ESXi](captures/1.3-esxi-datastore.png)

---

## 6. Déploiement vCenter Server (VCSA)

### 6.1 Stage 1 – Déploiement de l'appliance

1. Monter ISO VCSA
2. Depuis ton PC hôte → `vcsa-ui-installer\win32\installer.exe`
3. Stage 1 : déployer sur ESXi
   - IP : `192.168.140.155`
   - Datastore : `datastore-VM`
   - Deployment Size : Tiny

### 6.2 Stage 2 – Configuration de vCenter

1. Après Stage 1 → cliquer sur **Continue** pour Stage 2
2. **Time synchronization** : Cocher *Synchronize time with the ESXi host*
3. **SSH Access** : Cocher *Enable SSH*
4. **SSO (Single Sign-On)** :
   - Sélectionner : *Create a new SSO domain*
   - Domain : `vsphere.local`
   - Site Name : `Default-First-Site`
   - Mot de passe : `P@ssword123`
5. **CEIP** : décocher si souhaité
6. Cliquer sur **Finish** → Stage 2 (~20 min)

### 6.3 Vérification de vCenter

- URL : `https://192.168.140.155`
- Username : `administrator@vsphere.local`
- Password : `P@ssword123`

> **📸 Capture 2.1** – Interface vSphere Client connectée

![Capture 2.1 - vSphere Client](captures/2.1-vcenter-login.png)

### 6.4 Configuration des Port Groups (VLANs)

| Port Group   | VLAN | Usage                    |
|--------------|------|--------------------------|
| Management   | 10   | ESXi / vCenter           |
| LAN-VLAN20   | 20   | Windows Server / Client  |
| DMZ-VLAN30   | 30   | Web Server               |
| WAN-VLAN99   | 99   | pfSense WAN              |

Dans vSphere : `Hôte → Configure → Networking → Port groups → Add Networking`

> **📸 Capture 2.2** – Datacenter `Lab-Entreprise` créé

![Capture 2.2 - Datacenter](captures/2.2-vcenter-datacenter.png)

> **📸 Capture 2.3** – Hôte ESXi ajouté sous le Datacenter

![Capture 2.3 - Hôte ESXi](captures/2.3-vcenter-host.png)

> **📸 Capture 2.4** – Les 4 Port Groups créés (LAN-VLAN20, DMZ-VLAN30, WAN-VLAN99, Management)

![Capture 2.4 - Port Groups](captures/2.4-vcenter-portgroups.png)

---

## 7. Déploiement pfSense

### 7.1 Préparation de l'ISO dans vSphere

1. Connecte-toi à ton vCenter
2. Va dans `Storage → datastore-VM → Files → New Folder` (nommer : ISO)
3. Upload `pfSense-CE-2.7.2-RELEASE-amd64.iso`

### 7.2 Création de la VM pfSense

| Paramètre    | Valeur               |
|--------------|----------------------|
| CPU          | 2 vCPU               |
| RAM          | 2 Go                 |
| Disque       | 20 Go                |
| NIC 1 (WAN)  | WAN-VLAN99           |
| NIC 2 (LAN)  | LAN-VLAN20           |
| NIC 3 (DMZ)  | DMZ-VLAN30           |
| ISO          | pfSense-CE-2.7.2.iso |

### 7.3 Installation et Configuration

**Assignation des interfaces (Option 1)**

- VLANs : Tapez `n`
- WAN : `vmx0`
- LAN : `vmx2`
- OPT1 (DMZ) : `vmx1`
- Validation : Tapez `y`

> **📸 Capture 3.1** – Console pfSense avec les 3 interfaces (WAN/LAN/OPT1)

![Capture 3.1 - Console pfSense](captures/3.1-pfsense-console.png)

**Configuration des adresses IP (Option 2)**

LAN (Interface 2) :
- Adresse IPv4 : `192.168.20.1` / Masque : `24`
- DHCP : `y` (192.168.20.10 → 192.168.20.50)

DMZ (Interface 3) :
- Adresse IPv4 : `192.168.30.1` / Masque : `24`
- DHCP : `y` (192.168.30.10 → 192.168.30.50)

### 7.4 Règles de Pare-feu

**A. Configuration de l'interface LAN**

1. Aller dans `Firewall > Rules > LAN`
2. Cliquer sur **Add**
3. Configurer : Action : Pass / Protocol : Any / Source : Any / Destination : Any
4. **Save** puis **Apply Changes**

> **📸 Capture 3.4** – Règle firewall LAN → Any

![Capture 3.4 - Règle LAN](captures/3.4-pfsense-rule-lan.png)

**B. Configuration de l'interface WAN**

1. Aller dans `Interfaces > WAN`
2. Décocher : *Block private networks and loopback addresses*
3. Décocher : *Block bogon networks*
4. **Save** et **Apply Changes**

> **📸 Capture 3.6** – WAN : cases "Block private networks" décochées

![Capture 3.6 - WAN pfSense](captures/3.6-pfsense-wan.png)

**C. Configuration de l'interface DMZ**

Répéter la même procédure que pour le LAN (`Firewall > Rules > DMZ → Add → Pass/Any`)

> **📸 Capture 3.5** – Règle firewall DMZ → Any

![Capture 3.5 - Règle DMZ](captures/3.5-pfsense-rule-dmz.png)

### 7.5 Accès à l'interface Web

- URL : `https://192.168.20.1`
- Utilisateur : `admin` / Mot de passe : `P@ssword123`

> **📸 Capture 3.2** – Dashboard Web pfSense

![Capture 3.2 - Dashboard pfSense](captures/3.2-pfsense-dashboard.png)

> **📸 Capture 3.3** – Interfaces : WAN + LAN (192.168.20.1) + DMZ (192.168.30.1)

![Capture 3.3 - Interfaces pfSense](captures/3.3-pfsense-interfaces.png)

---

## 8. Déploiement Windows Server 2022 (AD + DNS)

### 8.1 Création de la VM

| Paramètre | Valeur              |
|-----------|---------------------|
| Nom       | SRV-AD-01           |
| CPU       | 2 vCPU              |
| RAM       | 4 Go                |
| Disque    | 60 Go               |
| Réseau    | LAN-VLAN20          |
| ISO       | Windows Server 2022 |

### 8.2 Configuration réseau

- Adresse IP : `192.168.20.10`
- Masque : `255.255.255.0`
- Passerelle : `192.168.20.1`
- DNS : `127.0.0.1` (après installation AD)

> **📸 Capture 4.3** – `ipconfig /all` avec IP 192.168.20.10 et DNS 127.0.0.1

![Capture 4.3 - ipconfig SRV-AD-01](captures/4.3-winserver-ipconfig.png)

### 8.3 Installation Active Directory

1. Ouvrir le **Gestionnaire de serveur**
2. Cliquer sur `Gérer > Ajouter des rôles et fonctionnalités`
3. Cocher **Services de domaine Active Directory (AD DS)**
4. Cliquer sur **Installer**

> **📸 Capture 4.1** – Gestionnaire de serveur avec rôle AD DS installé

![Capture 4.1 - AD DS installé](captures/4.1-winserver-adds.png)

### 8.4 Promotion en contrôleur de domaine

1. Cliquer sur le **drapeau jaune** en haut du Gestionnaire
2. Cliquer sur *Promouvoir ce serveur en contrôleur de domaine*
3. Sélectionner **Ajouter une nouvelle forêt**
4. Nom de domaine racine : `entreprise.local`
5. Définir le mot de passe de restauration (DSRM)
6. Le serveur va redémarrer

> **📸 Capture 4.2** – Console ADUC avec domaine `entreprise.local`

![Capture 4.2 - ADUC](captures/4.2-winserver-aduc.png)

### 8.5 Configuration des redirecteurs DNS

1. Ouvrir `dnsmgmt.msc`
2. Clic droit sur le serveur → **Propriétés** → onglet **Redirecteurs**
3. Ajouter :
   - `192.168.20.1` (pfSense)
   - `8.8.8.8` (DNS Google)

> **📸 Capture 4.4** – Console DNS avec zone `entreprise.local`

![Capture 4.4 - DNS zone](captures/4.4-winserver-dns.png)

> **📸 Capture 4.5** – Redirecteurs DNS configurés (192.168.20.1 + 8.8.8.8)

![Capture 4.5 - Redirecteurs DNS](captures/4.5-winserver-dns-redirecteurs.png)

---

## 9. Intégration du Client Windows 10

### 9.1 Création de la VM

| Paramètre | Valeur       |
|-----------|--------------|
| Nom       | CLI-WIN10-01 |
| CPU       | 2 vCPU       |
| RAM       | 2 Go         |
| Disque    | 40 Go        |
| Réseau    | LAN-VLAN20   |
| ISO       | Windows 10   |

### 9.2 Configuration réseau

- Adresse IP : `192.168.20.20`
- Masque : `255.255.255.0`
- Passerelle : `192.168.20.1`
- DNS : `192.168.20.10`

> **📸 Capture 5.1** – `ipconfig /all` avec IP 192.168.20.20 et DNS 192.168.20.10

![Capture 5.1 - ipconfig CLI-WIN10](captures/5.1-win10-ipconfig.png)

### 9.3 Jonction au domaine

1. `Propriétés système → Modifier → Domaine`
2. Saisir : `entreprise.local`
3. Authentifier avec les identifiants AD
4. Redémarrer le poste

> **📸 Capture 5.2** – Jonction au domaine `entreprise.local` réussie

![Capture 5.2 - Jonction domaine](captures/5.2-win10-domain-join.png)

### 9.4 Vérification
```cmd
whoami
```

Résultat attendu : `entreprise\utilisateur`

> **📸 Capture 5.3** – `whoami` → `entreprise\jean.dupont`

![Capture 5.3 - whoami](captures/5.3-win10-whoami.png)

> **📸 Capture 5.4** – Ordinateur visible dans la console ADUC

![Capture 5.4 - ADUC Computers](captures/5.4-win10-aduc-computers.png)

---

## 10. Serveur Web DMZ (Ubuntu Server)

### 10.1 Création de la VM

| Paramètre | Valeur               |
|-----------|----------------------|
| Nom       | SRV-WEB-DMZ          |
| CPU       | 2 vCPU               |
| RAM       | 2 Go                 |
| Disque    | 30 Go                |
| Réseau    | DMZ-VLAN30           |
| ISO       | Ubuntu Server 22.04  |

### 10.2 Configuration réseau

- Adresse IP : `192.168.30.10`
- Masque : `255.255.255.0`
- Passerelle : `192.168.30.1`
- DNS : `8.8.8.8`

### 10.3 Installation du serveur Web

Mise à jour des dépôts :
```bash
sudo apt update
```

Installation d'Apache2 :
```bash
sudo apt install apache2 -y
```

Création de la page HTML personnalisée :
```bash
echo "<h1>TP DMZ REUSSI - Serveur Web</h1>" | sudo tee /var/www/html/index.html
```

Vérification du service :
```bash
sudo systemctl status apache2
```

> **📸 Capture 6.1** – `sudo systemctl status apache2` → Active (running)

![Capture 6.1 - Apache2 status](captures/6.1-ubuntu-apache-status.png)

### 10.4 Test de validation

Depuis CLI-WIN10-01 : ouvrir le navigateur → `http://192.168.30.10`

> **📸 Capture 6.2** – Page web "TP DMZ REUSSI" depuis le navigateur

![Capture 6.2 - Page web DMZ](captures/6.2-ubuntu-webpage.png)

> **📸 Capture 6.3** – `ping 8.8.8.8` depuis Ubuntu → succès

![Capture 6.3 - Ping 8.8.8.8](captures/6.3-ubuntu-ping.png)

---

## 11. Tests finaux et validation

### 11.1 Tests de connectivité

Depuis le client Windows (CMD) :
```cmd
ping 192.168.20.10          # Test vers AD
ping 192.168.30.10          # Test vers DMZ
nslookup entreprise.local   # Test DNS
tracert 192.168.30.10       # Trace du chemin réseau
```

Depuis Ubuntu DMZ :
```bash
ping 192.168.20.1           # Test vers pfSense DMZ
ping 8.8.8.8                # Test connectivité Internet
curl http://localhost        # Test Apache local
sudo ss -tlnp | grep 80     # Vérifier port HTTP
```

> **📸 Capture 7** – Tests de validation réseau (Connectivité LAN, DMZ et résolution DNS)

![Capture 7 - Tests validation](captures/7-tests-validation.png)

Cette capture regroupe l'ensemble des tests de validation de l'infrastructure réseau depuis le poste client CLI-WIN10-01 :

- **Test 1 - Ping LAN → AD** : `ping 192.168.20.10` → Succès, confirme la connectivité entre le client et le contrôleur de domaine sur le VLAN 20.
- **Test 2 - Ping LAN → DMZ** : `ping 192.168.30.10` → Succès, confirme que le trafic traverse correctement le firewall pfSense vers la zone DMZ (VLAN 30).
- **Test 3 - Résolution DNS** : `nslookup entreprise.local` → Résolution réussie, confirme le bon fonctionnement du serveur DNS Active Directory.
- **Test 4 - Traceroute** : `tracert 192.168.30.10` → Le chemin passe par `192.168.20.1` (pfSense), confirmant le routage inter-VLAN via le firewall.

### 11.2 Validation de l'infrastructure

| Test                  | Résultat attendu                    |
|-----------------------|-------------------------------------|
| Ping LAN ↔ DMZ        | Succès depuis client                |
| Ping DMZ → LAN        | Bloqué par pfSense (si configuré)   |
| HTTP LAN → DMZ        | Page web accessible                 |
| Résolution DNS        | `entreprise.local` résolu           |
| vCenter accessible    | `https://192.168.140.155` répond    |
| ESXi accessible       | `https://192.168.140.150` répond    |

---

## 12. VMotion - Migration à chaud (Optionnel)

> **Note :** Cette section nécessite un deuxième hôte ESXi (ESXi-02 : 192.168.140.151)

### 12.1 Préparation du stockage

- Sur l'hôte cible (192.168.140.151), créer un nouveau datastore VMFS
- Nom : `Datastore-ESXi02`

### 12.2 Configuration du réseau

- Sur l'hôte cible, créer les mêmes Port Groups (LAN-VLAN20, DMZ-VLAN30, etc.)
- Activer vMotion sur les adaptateurs VMkernel des deux hôtes

### 12.3 Lancement de la migration

1. Clic droit sur la VM → **Migrer**
2. Type : *Modifier la ressource de calcul et le stockage*
3. Sélectionner l'hôte cible (`192.168.140.151`)
4. Sélectionner le datastore : `Datastore-ESXi02`
5. Vérifier les réseaux / Priorité : Élevée
6. **Terminer**

> **📸 Capture 8.1** – Paramétrage de la migration vMotion

![Capture 8.1 - vMotion paramétrage](captures/8.1-vmotion-setup.png)

> **📸 Capture 8.2** – Assistant de migration lancé (type : calcul + stockage)

![Capture 8.2 - vMotion assistant](captures/8.2-vmotion-assistant.png)

### 12.4 Vérification

- La tâche *Relocate VM* doit être à 100%
- La VM doit apparaître sous l'hôte `.151`

> **📸 Capture 8.3** – VM affichée sous l'hôte .151 après migration

![Capture 8.3 - VM après migration](captures/8.3-vmotion-result.png)

---

## 13. Gestion des Snapshots

### 13.1 Qu'est-ce qu'un snapshot ?

Un snapshot (instantané) est une copie de l'état d'une VM à un instant précis.

Utilité :
- Sauvegarder avant une modification critique
- Tester une mise à jour (rollback si problème)
- Créer des points de restauration

> ⚠️ **Attention :** Les snapshots ne sont PAS des sauvegardes ! Ne pas les conserver longtemps (impact performance).

### 13.2 Création d'un snapshot

1. Sélectionner la VM (ex: SRV-AD-01)
2. Clic droit → `Snapshots → Take Snapshot`
3. Configuration :
   - Name : `Avant_Config_GPO`
   - Description : *État du serveur AD avant configuration des GPO*
   - Cocher : *Snapshot the virtual machine's memory*
   - Cocher : *Quiesce guest file system*
4. Cliquer sur **OK**

> **📸 Capture 9.1** – Création du snapshot (fenêtre Take Snapshot)

![Capture 9.1 - Take Snapshot](captures/9.1-snapshot-create.png)

### 13.3 Restaurer un snapshot

1. Clic droit sur la VM → `Snapshots → Manage Snapshots`
2. Sélectionner le snapshot désiré
3. Cliquer sur **Restore** → Confirmer avec **Yes**

> **📸 Capture 9.2** – Gestionnaire de snapshots avec snapshot visible

![Capture 9.2 - Manage Snapshots](captures/9.2-snapshot-manager.png)

> **📸 Capture 9.3** – Snapshot restauré avec succès

![Capture 9.3 - Snapshot restauré](captures/9.3-snapshot-restored.png)

### 13.4 Bonnes pratiques

| Recommandation          | Explication                                   |
|-------------------------|-----------------------------------------------|
| Durée de vie courte     | Max 24-72h pour éviter impact performance     |
| Avant modifications     | Créer systématiquement avant mise à jour      |
| Pas pour backup         | Un snapshot dépend du disque original         |
| Surveiller l'espace     | Les snapshots consomment de l'espace disque   |
| Documenter              | Utiliser nom et description clairs            |
| Consolider régulièrement| Supprimer les snapshots obsolètes             |

---

## 14. Bonnes pratiques TP imbriqué

- NIC1 = NAT → Management / Internet
- NIC2 = Host-Only → Production / LAN / DMZ
- 2 disques → 1 pour OS ESXi, 1 pour datastore
- `vhv.enable = TRUE`, `hypervisor.cpuid.v0 = FALSE`
- VLANs séparés pour Management / LAN / DMZ / WAN
- RAM / CPU suffisants (16 Go RAM / 4 vCPU ESXi minimum)

---

## 15. Plan d'adressage récapitulatif

### 15.1 Réseau Management (192.168.140.0/24)

| Machine  | IP               | Gateway          | DNS     | Utilisateur                    | Mot de passe |
|----------|------------------|------------------|---------|--------------------------------|--------------|
| ESXi-01  | 192.168.140.150  | 192.168.140.2    | 8.8.8.8 | root                           | Admin!2026   |
| ESXi-02  | 192.168.140.151  | 192.168.140.2    | 8.8.8.8 | root                           | Admin!2026   |
| vCenter  | 192.168.140.155  | 192.168.140.2    | 8.8.8.8 | administrator@vsphere.local    | P@ssword123  |

### 15.2 Réseau LAN (192.168.20.0/24 - VLAN 20)

| Machine      | IP             | Gateway        | DNS            | Utilisateur    | Mot de passe |
|--------------|----------------|----------------|----------------|----------------|--------------|
| pfSense LAN  | 192.168.20.1   | -              | -              | admin          | P@ssword123  |
| SRV-AD-01    | 192.168.20.10  | 192.168.20.1   | 127.0.0.1      | Administrateur | Admin2026    |
| CLI-WIN10-01 | 192.168.20.20  | 192.168.20.1   | 192.168.20.10  | -              | -            |

### 15.3 Réseau DMZ (192.168.30.0/24 - VLAN 30)

| Machine      | IP             | Gateway        | DNS     | Utilisateur | Mot de passe |
|--------------|----------------|----------------|---------|-------------|--------------|
| pfSense DMZ  | 192.168.30.1   | -              | -       | -           | -            |
| SRV-WEB-DMZ  | 192.168.30.10  | 192.168.30.1   | 8.8.8.8 | admin-web   | admin2026    |

### 15.4 Interfaces pfSense

| Interface    | Réseau  | IP             | DHCP                    |
|--------------|---------|----------------|-------------------------|
| WAN (vmx0)   | NAT     | DHCP (auto)    | -                       |
| LAN (vmx2)   | VLAN 20 | 192.168.20.1   | 192.168.20.10-50        |
| DMZ (vmx1)   | VLAN 30 | 192.168.30.1   | 192.168.30.10-50        |

---

## Conclusion

Ce TP vous a permis de mettre en place une infrastructure d'entreprise complète, incluant :

- Un hyperviseur ESXi imbriqué avec vCenter Server pour la gestion centralisée
- Une segmentation réseau avec VLANs (Management, LAN, DMZ, WAN)
- Un firewall pfSense pour sécuriser et isoler les différents réseaux
- Un contrôleur de domaine Active Directory avec DNS
- Un poste client Windows joint au domaine
- Un serveur web en DMZ (isolation sécurisée)
- La gestion des snapshots pour la sauvegarde et la restauration
- La migration à chaud (VMotion) entre deux hôtes ESXi

Cette infrastructure constitue la base d'une architecture d'entreprise moderne et vous permet de comprendre les concepts essentiels de la virtualisation, de la segmentation réseau et de la sécurité informatique.

**Félicitations pour avoir complété ce TP ! 🎉**
