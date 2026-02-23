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
- ISO pfSense 2.6.0
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

> **Capture 1.1** – Page de connexion ESXi (`https://192.168.140.150`)

![Capture 1.1](1.1%20Page%20de%20connexion%20ESXi.PNG)

> **Capture 1.2** – Dashboard ESXi avec version 8.x visible

![Capture 1.2](1.2%20Dashboard%20ESXi%20avec%20version%208.x%20visible.png)

> **Capture 1.3** – Configuration réseau Management (IP 192.168.140.150)

![Capture 1.3](1.3%20Configuration%20r%C3%A9seau%20Management%20(IP%20192.168.140.150).PNG)

### 5.3 Datastore

Ajouter Disque 2 (200 Go) comme datastore. Dans ESXi Web UI :
```
Storage → Datastores → New Datastore → VMFS → Sélectionner Disque 2 → Nom : datastore-VM
```

> **Capture 1.4** – Datastore `datastore-VM` visible (200 Go)

!![Capture 1.4](1.4%20Datastore%20datastore-VM%20visible%20(200%20Go).PNG)

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

> **Capture 2.1** – Interface vSphere Client connectée

![Capture 2.1](2.1%20Interface%20vSphere%20Client%20connect%C3%A9e.PNG)

### 6.4 Configuration des Port Groups (VLANs)

| Port Group   | VLAN | Usage                    |
|--------------|------|--------------------------|
| Management   | 10   | ESXi / vCenter           |
| LAN-VLAN20   | 20   | Windows Server / Client  |
| DMZ-VLAN30   | 30   | Web Server               |
| WAN-VLAN99   | 99   | pfSense WAN              |

Dans vSphere : `Hôte → Configure → Networking → Port groups → Add Networking`

> **Capture 2.2** – Datacenter `Lab-Entreprise` créé

![Capture 2.2](2.2%20Datacenter%20Lab-Entreprise%20cr%C3%A9%C3%A9.PNG)

> **Capture 2.3** – Hôte ESXi ajouté sous le Datacenter

![Capture 2.3](2.3%20H%C3%B4te%20ESXi%20ajout%C3%A9%20sous%20le%20Datacenter.PNG)

> **Capture 2.4** – Les 4 Port Groups créés (LAN-VLAN20, DMZ-VLAN30, WAN-VLAN99, Management)

![Capture 2.4](2.4%20Les%204%20Port%20Groups%20cr%C3%A9%C3%A9s%20(LAN-VLAN20,%20DMZ-VLAN30,%20WAN-VLAN99,%20Management).PNG)

---

## 7. Déploiement pfSense

### 7.1 Préparation de l'ISO dans vSphere

1. Connecte-toi à ton vCenter
2. Va dans `Storage → datastore-VM → Files → New Folder` (nommer : ISO)
3. Upload `pfSense-CE-2.6.0-RELEASE-amd64.iso`

### 7.2 Création de la VM pfSense

| Paramètre    | Valeur               |
|--------------|----------------------|
| CPU          | 2 vCPU               |
| RAM          | 2 Go                 |
| Disque       | 20 Go                |
| NIC 1 (WAN)  | WAN-VLAN99           |
| NIC 2 (LAN)  | LAN-VLAN20           |
| NIC 3 (DMZ)  | DMZ-VLAN30           |
| ISO          | pfSense-CE-2.6.0.iso |

### 7.3 Installation et Configuration

**Assignation des interfaces (Option 1)**

- VLANs : Tapez `n`
- WAN : `vmx0`
- LAN : `vmx1`
- OPT1 (DMZ) : `vmx2`
- Validation : Tapez `y`

> **Capture 3.1** – Console pfSense avec les 3 interfaces (WAN/LAN/OPT1)

![Capture 3.1](3.1%20Console%20pfSense%20avec%20les%203%20interfaces%20(WANLANOPT1).png)

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

> **Capture 3.2** – Règle firewall LAN → Any

![Capture 3.2](3.2%20R%C3%A8gle%20firewall%20LAN%20%E2%86%92%20Any.png)

**B. Configuration de l'interface WAN**

1. Aller dans `Interfaces > WAN`
2. Décocher : *Block private networks and loopback addresses*
3. Décocher : *Block bogon networks*
4. **Save** et **Apply Changes**

> **Capture 3.3** – WAN : cases "Block private networks" décochées

![Capture 3.3](3.3%20WAN%20%20cases%20Block%20private%20networks%20d%C3%A9coch%C3%A9es.png)

**C. Configuration de l'interface DMZ**

Répéter la même procédure que pour le LAN (`Firewall > Rules > DMZ → Add → Pass/Any`)

> **Capture 3.4** – Règle firewall DMZ → Any

![Capture 3.4](3.4%20R%C3%A8gle%20firewall%20DMZ%20%E2%86%92%20Any.png)

### 7.5 Accès à l'interface Web

- URL : `https://192.168.20.1`
- Utilisateur : `admin` / Mot de passe : `P@ssword123`

> **Capture 3.5** – Dashboard Web pfSense

![Capture 3.5](3.5%20Dashboard%20Web%20pfSense.png)

> **Capture 3.6** – Interfaces : WAN + LAN (192.168.20.1) + DMZ (192.168.30.1)

![Capture 3.6](3.6%20Interfaces%20%20WAN%20+%20LAN%20(192.168.20.1)%20+%20DMZ%20(192.168.30.1).png)

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

> **Capture 4.1** – `ipconfig /all` avec IP 192.168.20.10 et DNS 127.0.0.1

![Capture 4.1](4.1%20ipconfig%20all%20avec%20IP%20192.168.20.10%20et%20DNS%20127.0.0.1.png)

### 8.3 Installation Active Directory

1. Ouvrir le **Gestionnaire de serveur**
2. Cliquer sur `Gérer > Ajouter des rôles et fonctionnalités`
3. Cocher **Services de domaine Active Directory (AD DS)**
4. Cliquer sur **Installer**

> **Capture 4.2** – Gestionnaire de serveur avec rôle AD DS installé

![Capture 4.2](4.2%20Gestionnaire%20de%20serveur%20avec%20r%C3%B4le%20AD%20DS%20install%C3%A9.PNG)

### 8.4 Promotion en contrôleur de domaine

1. Cliquer sur le **drapeau jaune** en haut du Gestionnaire
2. Cliquer sur *Promouvoir ce serveur en contrôleur de domaine*
3. Sélectionner **Ajouter une nouvelle forêt**
4. Nom de domaine racine : `entreprise.local`
5. Définir le mot de passe de restauration (DSRM)
6. Le serveur va redémarrer
7. Vérification de l'inventaire : Ouvrir la console Utilisateurs et ordinateurs Active Directory (ADUC) et vérifier dans le dossier Computers que l'ordinateur client (CLI-WIN10-01) est bien listé, confirmant ainsi la communication entre le client et le domaine.

> **Capture 4.3** – Console ADUC avec domaine `entreprise.local`

![Capture 4.3](4.3%20Console%20ADUC%20avec%20domaine%20entreprise.local.PNG)

### 8.5 Configuration des redirecteurs DNS

1. Ouvrir `dnsmgmt.msc`
2. Clic droit sur le serveur → **Propriétés** → onglet **Redirecteurs**
3. Ajouter :
   - `192.168.20.1` (pfSense)
   - `8.8.8.8` (DNS Google)

> **Capture 4.4** – Console DNS avec zone `entreprise.local`

![Capture 4.4](4.4%20Console%20DNS%20avec%20zone%20entreprise.local.png)

> **Capture 4.5** – Redirecteurs DNS configurés (192.168.20.1 + 8.8.8.8)

![Capture 4.5](4.5%20Redirecteurs%20DNS%20configur%C3%A9s%20(192.168.20.1%20+%208.8.8.8).png)

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

> **Capture 5.1** – `ipconfig /all` avec IP 192.168.20.20 et DNS 192.168.20.10

![Capture 5.1](5.1%20ipconfig%20all%20avec%20IP%20192.168.20.20%20et%20DNS%20192.168.20.10.png)

### 9.3 Jonction au domaine

1. `Propriétés système → Modifier → Domaine`
2. Saisir : `entreprise.local`
3. Authentifier avec les identifiants AD
4. Redémarrer le poste

> **Capture 5.2** – Jonction au domaine `entreprise.local` réussie

![Capture 5.2](5.2%20client%20Windows%2010%20%E2%86%92%20Jonction%20au%20domaine%20entreprise.local%20r%C3%A9ussie.png)

### 9.4 Vérification
```cmd
whoami
```

Résultat attendu : `entreprise\utilisateur`

> **Capture 5.3** – `whoami` → `entreprise\jean.dupont`

![Capture 5.3](5.3%20whoami%20%E2%86%92%20entreprise-jean.dupont.png)

> **Capture 5.4** – Ordinateur visible dans la console ADUC

![Capture 5.4](5.4%20Ordinateur%20visible%20dans%20la%20console%20ADUC.png)

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

> **Capture 6.1** – `sudo systemctl status apache2` → Active (running)

![Capture 6.1](6.1%20sudo%20systemctl%20status%20apache2%20%E2%86%92%20Active%20(running).png)

### 10.4 Test de validation

Depuis CLI-WIN10-01 : ouvrir le navigateur → `http://192.168.30.10`

> **Capture 6.2** – Page web "TP DMZ REUSSI" depuis le navigateur

![Capture 6.2](6.2%20Page%20web%20TP%20DMZ%20REUSSI%20depuis%20le%20navigateur.png)

> **Capture 6.3** – `ping 8.8.8.8` depuis Ubuntu → succès

![Capture 6.3](6.3%20ping%208.8.8.8%20depuis%20Ubuntu%20%E2%86%92%20succ%C3%A8s.png)

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

> **Capture 7.1** – Tests de validation réseau (Connectivité LAN, DMZ et résolution DNS)

![Capture 7.1](7.1%20Tests%20de%20validation%20r%C3%A9seau%20%20Connectivit%C3%A9%20LAN,%20DMZ%20et%20r%C3%A9solution%20DNS.png)

Cette capture regroupe l'ensemble des tests de validation de l'infrastructure réseau depuis le poste client CLI-WIN10-01 :

- **Test 1 - Ping LAN → AD** : `ping 192.168.20.10` → Succès, confirme la connectivité entre le client et le contrôleur de domaine sur le VLAN 20.
- **Test 2 - Ping LAN → DMZ** : `ping 192.168.30.10` → Succès, confirme que le trafic traverse correctement le firewall pfSense vers la zone DMZ (VLAN 30).
- **Test 3 - Résolution DNS** : `nslookup entreprise.local` → Résolution réussie, confirme le bon fonctionnement du serveur DNS Active Directory.
- **Test 4 - Traceroute** : `tracert 192.168.30.10` → Le chemin passe par `192.168.20.1` (pfSense), confirmant le routage inter-VLAN via le firewall.

Depuis Ubuntu DMZ :

```bash
ping 192.168.30.1           # Test vers passerelle DMZ (pfSense)
ping 8.8.8.8                # Test connectivité Internet
curl http://localhost        # Test Apache local
sudo ss -tlnp | grep 80     # Vérifier port HTTP
```
> **Capture 7.2** – Tests de connectivité depuis le serveur Web DMZ (Ubuntu)

![Capture 7.2](7.2%20Tests%20de%20connectivité%20depuis%20le%20serveur%20Web%20DMZ%20(Ubuntu).png)

Cette capture regroupe l'ensemble des tests de validation effectués directement depuis le serveur SRV-WEB-DMZ pour confirmer son isolation et son accès aux ressources nécessaires :

* **Test 1 - Ping vers la passerelle** : `ping 192.168.20.1` → Succès, confirme que le serveur peut joindre l'interface LAN du firewall pfSense pour le routage.
* **Test 2 - Connectivité Internet** : `ping 8.8.8.8` → Succès, confirme que la DMZ dispose d'un accès vers l'extérieur pour les mises à jour et les services.
* **Test 3 - Test du service Web local** : `curl http://localhost` → Succès, le serveur retourne bien la balise `TP DMZ REUSSI`, confirmant que le serveur Apache2 est fonctionnel.
* **Test 4 - État des ports** : `ss -tlnp | grep 80` → Le port 80 est bien en écoute (LISTEN), confirmant que le service est prêt à recevoir des requêtes HTTP.

### 11.2 Validation de l'infrastructure

| Test | Résultat attendu | État |
| :--- | :--- | :--- |
| **Ping LAN ↔ DMZ** | Succès depuis le client Windows |  OK |
| **Ping DMZ → LAN** | Bloqué par pfSense (Isolation) |  OK |
| **HTTP LAN → DMZ** | Page "TP DMZ REUSSI" accessible |  OK |
| **Résolution DNS** | `entreprise.local` résolu par l'AD |  OK |
| **Accès vCenter** | `https://192.168.140.155` opérationnel |  OK |
| **Accès ESXi** | `https://192.168.140.150` & `...151` répondent |  OK |

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

> **Capture 8.1** – Paramétrage de la migration vMotion

![Capture 8.1](8.1%20vSphere%20Client%20%20Param%C3%A9trage%20de%20la%20migration%20vMotion..png)

> **Capture 8.2** – Assistant de migration lancé (type : calcul + stockage)

![Capture 8.2](8.2%20Assistant%20de%20migration%20lanc%C3%A9%20(type%20%20calcul%20+%20stockage).png)

### 12.4 Vérification

- La tâche *Relocate VM* doit être à 100%
- La VM doit apparaître sous l'hôte `.151`

> **Capture 8.3** – VM affichée sous l'hôte .151 après migration

![Capture 8.3](8.3%20VM%20affich%C3%A9e%20sous%20l'h%C3%B4te%20.151%20apr%C3%A8s%20migration.png)

---

## 13. Gestion des Snapshots

### 13.1 Qu'est-ce qu'un snapshot ?

Un snapshot (instantané) est une copie de l'état d'une VM à un instant précis.

Utilité :
- Sauvegarder avant une modification critique
- Tester une mise à jour (rollback si problème)
- Créer des points de restauration

> **Attention :** Les snapshots ne sont PAS des sauvegardes ! Ne pas les conserver longtemps (impact performance).

### 13.2 Création d'un snapshot

1. Sélectionner la VM (ex: SRV-AD-01)
2. Clic droit → `Snapshots → Take Snapshot`
3. Configuration :
   - Name : `Avant_Config_GPO`
   - Description : *État du serveur AD avant configuration des GPO*
   - Cocher : *Snapshot the virtual machine's memory*
   - Cocher : *Quiesce guest file system*
4. Cliquer sur **OK**

> **Capture 9.1** – Création du snapshot (fenêtre Take Snapshot)

![Capture 9.1](9.1%20Cr%C3%A9ation%20du%20snapshot%20(fen%C3%AAtre%20Take%20Snapshot).PNG)

### 13.3 Restaurer un snapshot

1. Clic droit sur la VM → `Snapshots → Manage Snapshots`
2. Sélectionner le snapshot désiré
3. Cliquer sur **Restore** → Confirmer avec **Yes**

> **Capture 9.2** – Gestionnaire de snapshots avec snapshot visible

![Capture 9.2](9.2%20Gestionnaire%20de%20snapshots%20avec%20snapshot%20visible.PNG)

> **Capture 9.3** – Snapshot restauré avec succès

![Capture 9.3](9.3%20Snapshot%20restaur%C3%A9%20avec%20succ%C3%A8s.%20V%C3%A9rification%20du%20rollback.PNG)

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


