# TP Virtualisation – Infrastructure ESXi Imbriquée avec vCenter

**Configuration complète : ESXi, vCenter, pfSense, Active Directory**

---

## Table des matières

1. [Prérequis matériels et logiciels](#1-prérequis-matériels-et-logiciels)
2. [Objectif du TP](#2-objectif-du-tp)
3. [Architecture](#3-architecture)
4. [Préparation de l'hôte](#4-préparation-de-lhôte)
5. [Création de la VM ESXi imbriqué](#5-création-de-la-vm-esxi-imbriqué)
6. [Déploiement vCenter Server (VCSA)](#6-déploiement-vcenter-server-vcsa)
7. [Déploiement pfSense](#7-déploiement-pfsense)
8. [Déploiement Windows Server 2022 (AD + DNS)](#8-déploiement-windows-server-2022-ad--dns)
9. [Intégration du Client Windows 10](#9-intégration-du-client-windows-10)
10. [Serveur Web DMZ (Ubuntu Server)](#10-serveur-web-dmz-ubuntu-server)
11. [Tests finaux et validation](#11-tests-finaux-et-validation)
12. [VMotion - Migration à chaud (Optionnel)](#12-vmotion---migration-à-chaud-optionnel)
13. [Gestion des Snapshots](#13-gestion-des-snapshots)
14. [Bonnes pratiques TP imbriqué](#14-bonnes-pratiques-tp-imbriqué)
15. [Plan d'adressage récapitulatif](#15-plan-dadressage-récapitulatif)

---

## 1. Prérequis matériels et logiciels

### 1.1 Matériel nécessaire

| Composant | Minimum | À l'aise |
|-----------|---------|----------|
| Processeur | Intel i5 / AMD Ryzen 5 | Intel i7 / Ryzen 7 |
| RAM | 16 Go | 32 Go (recommandé) |
| Disque dur | 200 Go libre | 500 Go SSD |
| Réseau | Carte Ethernet | — |

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

Mettre en place une **infrastructure d'entreprise complète**, comprenant :

- ESXi imbriqué (hyperviseur dans Workstation)
- vCenter Server
- pfSense (firewall, segmentation)
- Windows Server (AD + DNS + File)
- Machine client (Windows ou Linux)
- Serveur web DMZ (Linux)

> **Note :** Les VM internes seront créées dans VMware Workstation, **pas directement dans ESXi**, pour contourner la nested virtualization.

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

- **NIC1 (NAT)** → Management (ESXi + vCenter) et accès Internet
- **NIC2 (Host-Only)** → Production interne (LAN/DMZ)

### 3.2 Plan d'adressage IP

| Zone | Réseau | VLAN | IP Type | Exemple |
|------|--------|------|---------|---------|
| Management | 192.168.140.0/24 | 10 | Statique | ESXi 192.168.140.150 / vCenter 192.168.140.155 |
| LAN | 192.168.20.0/24 | 20 | Statique | Windows Server 192.168.20.10 / Client 192.168.20.20 |
| DMZ | 192.168.30.0/24 | 30 | Statique | Web 192.168.30.10 |
| WAN | DHCP / NAT | 99 | DHCP | pfSense WAN |

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

| Paramètre | Valeur |
|-----------|--------|
| Nom | ESXi-TP-Entreprise |
| Type | Custom (Advanced) |
| Guest OS | VMware ESXi 8.x |
| Firmware | UEFI |
| CPU | 4 vCPU |
| RAM | 16 Go |
| Disques | 50 Go (OS) + 200 Go (Datastore) |
| NIC 1 | NAT (Management) |
| NIC 2 | Host-Only (Production) |
| ISO | VMware-ESXi-8.x.iso |
| Nested virtualization | `vhv.enable=TRUE, hypervisor.cpuid.v0=FALSE` |

### 5.2 Installation d'ESXi

1. Monter ISO et démarrer la VM
2. Installer ESXi sur **Disque 1 (50 Go)**
3. Définir mot de passe root : **Admin!2026**
4. Configurer **Management Network** sur NIC1 (192.168.140.150)
5. Configurer DNS : 192.168.140.2, 8.8.8.8
6. Hostname : esxi01.entreprise.local

### 5.3 Datastore

1. Ajouter **Disque 2 (200 Go)** comme datastore
2. Dans ESXi Web UI :
   ```
   Storage → Datastores → New Datastore → VMFS → Sélectionner Disque 2 → Nom : datastore-VM
   ```

---

## 6. Déploiement vCenter Server (VCSA)

### 6.1 Stage 1 – Déploiement de l'appliance

1. Monter ISO VCSA
2. Depuis ton PC hôte → `vcsa-ui-installer\win32\installer.exe`
3. Stage 1 : déployer sur ESXi
   - IP : 192.168.140.155
   - Datastore : datastore-VM
   - Deployment Size : Tiny

### 6.2 Stage 2 – Configuration de vCenter

**Objectif :** finaliser la configuration et activer les services.

1. Après Stage 1 → cliquer sur **Continue** pour Stage 2

2. **Time synchronization** :
   - Cocher `Synchronize time with the ESXi host` → vCenter synchronise son horloge avec l'hôte ESXi

3. **SSH Access** :
   - Cocher `Enable SSH` → permet l'accès console pour dépannage

4. **SSO (Single Sign-On)** :
   - Sélectionner : `Create a new SSO domain`
   - Domain : `vsphere.local`
   - Site Name : `Default-First-Site` (laisser par défaut)
   - Mot de passe pour `administrator@vsphere.local` : P@ssword123

5. **CEIP** : décocher si tu ne souhaites pas envoyer les données à VMware

6. Cliquer sur **Finish** → Stage 2 (~20 min)
   - vCenter configure ses services internes, base de données, et SSO.

### 6.3 Vérification de vCenter

1. Ouvre un navigateur → `https://192.168.140.155`
2. Login :
   - Username : `administrator@vsphere.local`
   - Password : `P@ssword123`
3. Tu accèdes au **vSphere Client Web**

### 6.4 Configuration des Port Groups (VLANs)

| Port Group | VLAN | Usage |
|------------|------|-------|
| Management | 10 | ESXi / vCenter |
| LAN-VLAN20 | 20 | Windows Server / Client |
| DMZ-VLAN30 | 30 | Web Server |
| WAN-VLAN99 | 99 | pfSense WAN |

Dans vSphere : Hôte → **Configure** → **Networking** → **Port groups** → **Add Networking**

---

## 7. Déploiement pfSense

### 7.1 Préparation de l'ISO dans vSphere

1. Connecte-toi à ton **vCenter**
2. Va dans l'onglet **Storage** → Sélectionne **datastore-VM**
3. Clique sur **Files** → **New Folder** (Nomme-le `ISO`)
4. Upload `pfSense-CE-2.7.2-RELEASE-amd64.iso`

### 7.2 Création de la VM pfSense

| Paramètre | Valeur |
|-----------|--------|
| CPU | 2 vCPU |
| RAM | 2 Go |
| Disque | 20 Go |
| NIC 1 (WAN) | WAN-VLAN99 |
| NIC 2 (LAN) | LAN-VLAN20 |
| NIC 3 (DMZ) | DMZ-VLAN30 |
| ISO | pfSense-CE-2.7.2-RELEASE-amd64.iso |

### 7.3 Installation et Configuration

#### Assignation des interfaces (Option 1)

- **VLANs** : Tapez **`n`**
- **WAN** : **`vmx0`**
- **LAN** : **`vmx2`**
- **OPT1 (DMZ)** : **`vmx1`**
- **Validation** : Tapez **`y`**

#### Configuration des adresses IP (Option 2)

**LAN (Interface 2) :**
- Adresse IPv4 : `192.168.20.1`
- Masque : `24`
- DHCP : `y` (192.168.20.10 → 192.168.20.50)

**DMZ (Interface 3) :**
- Adresse IPv4 : `192.168.30.1`
- Masque : `24`
- DHCP : `y` (192.168.30.10 → 192.168.30.50)

### 7.4 Règles de Pare-feu

#### A. Configuration de l'interface LAN

1. Aller dans **Firewall > Rules > LAN**
2. Cliquer sur **Add**
3. Configurer :
   - **Action** : `Pass`
   - **Protocol** : `Any`
   - **Source** : `Any`
   - **Destination** : `Any`
4. Cliquer sur **Save** puis **Apply Changes**

#### B. Configuration de l'interface WAN

1. Aller dans **Interfaces > WAN**
2. **Décocher** : `Block private networks and loopback addresses`
3. **Décocher** : `Block bogon networks`
4. Cliquer sur **Save** et **Apply Changes**

#### C. Configuration de l'interface DMZ

Répéter la même procédure que pour le LAN (**Firewall > Rules > DMZ** → Add → Pass/Any)

### 7.5 Accès à l'interface Web

- **URL** : `https://192.168.20.1`
- **Utilisateur** : `admin`
- **Mot de passe par défaut** : `pfsense`
- **Nouveau mot de passe** : `P@ssword123` (à changer lors de la première connexion)

---

## 8. Déploiement Windows Server 2022 (AD + DNS)

### 8.1 Création de la VM

| Paramètre | Valeur |
|-----------|--------|
| Nom | SRV-AD-01 |
| CPU | 2 vCPU |
| RAM | 4 Go |
| Disque | 60 Go |
| Réseau | LAN-VLAN20 |
| ISO | Windows Server 2022 |

### 8.2 Configuration réseau

- **Adresse IP** : `192.168.20.10`
- **Masque** : `255.255.255.0`
- **Passerelle** : `192.168.20.1`
- **DNS** : `127.0.0.1` (après installation AD)

### 8.3 Installation Active Directory

1. Ouvrir le **Gestionnaire de serveur**
2. Cliquer sur **Gérer > Ajouter des rôles et fonctionnalités**
3. Cocher **Services de domaine Active Directory (AD DS)**
4. Cliquer sur **Installer**

### 8.4 Promotion en contrôleur de domaine

1. Cliquer sur le **drapeau jaune** en haut du Gestionnaire
2. Cliquer sur **Promouvoir ce serveur en contrôleur de domaine**
3. Sélectionner **Ajouter une nouvelle forêt**
4. Nom de domaine racine : **`entreprise.local`**
5. Définir le mot de passe de restauration (DSRM)
6. Le serveur va redémarrer

### 8.5 Configuration des redirecteurs DNS

1. Ouvrir le **Gestionnaire de DNS** (`dnsmgmt.msc`)
2. Clic droit sur le nom du serveur → **Propriétés**
3. Aller dans l'onglet **Redirecteurs**
4. Ajouter :
   - `192.168.20.1` (pfSense)
   - `8.8.8.8` (DNS Google)

---

## 9. Intégration du Client Windows 10

### 9.1 Création de la VM

| Paramètre | Valeur |
|-----------|--------|
| Nom | CLI-WIN10-01 |
| CPU | 2 vCPU |
| RAM | 2 Go |
| Disque | 40 Go |
| Réseau | LAN-VLAN20 |
| ISO | Windows 10/11 |

### 9.2 Configuration réseau

- **Adresse IP** : `192.168.20.20`
- **Masque** : `255.255.255.0`
- **Passerelle** : `192.168.20.1`
- **DNS** : `192.168.20.10` (Serveur AD)

### 9.3 Jonction au domaine

1. Propriétés système → **Modifier** → **Domaine**
2. Saisir : **`entreprise.local`**
3. Authentifier avec les identifiants AD
4. Redémarrer le poste

### 9.4 Vérification

Commande : `whoami`

Résultat attendu : **`entreprise\utilisateur`**

---

## 10. Serveur Web DMZ (Ubuntu Server)

### 10.1 Création de la VM

| Paramètre | Valeur |
|-----------|--------|
| Nom | SRV-WEB-DMZ |
| CPU | 2 vCPU |
| RAM | 2 Go |
| Disque | 30 Go |
| Réseau | DMZ-VLAN30 |
| ISO | Ubuntu Server 22.04 |

### 10.2 Configuration réseau

- **Adresse IP** : `192.168.30.10`
- **Masque** : `255.255.255.0`
- **Passerelle** : `192.168.30.1`
- **DNS** : `8.8.8.8`

### 10.3 Installation du serveur Web

**Mise à jour des dépôts :**
```bash
sudo apt update
```

**Installation d'Apache2 :**
```bash
sudo apt install apache2 -y
```

**Création de la page HTML personnalisée :**
```bash
echo "<h1>TP DMZ REUSSI - Serveur Web</h1>" | sudo tee /var/www/html/index.html
```

**Vérification du service :**
```bash
sudo systemctl status apache2
```

### 10.4 Test de validation

Depuis le client Windows (CLI-WIN10-01) :
- Ouvrir le navigateur
- Aller à : **`http://192.168.30.10`**
- **Résultat attendu :** La page web personnalisée s'affiche

---

## 11. Tests finaux et validation

### 11.1 Tests de connectivité

**Depuis le client Windows (CMD) :**
```bash
ping 192.168.20.10          # Test vers AD
ping 192.168.30.10          # Test vers DMZ
nslookup entreprise.local   # Test DNS
tracert 192.168.30.10       # Trace du chemin réseau
```

**Depuis Ubuntu DMZ :**
```bash
ping 192.168.20.1           # Test vers pfSense DMZ
ping 8.8.8.8                # Test connectivité Internet
curl http://localhost       # Test Apache local
sudo ss -tlnp | grep 80     # Vérifier port HTTP
```

### 11.2 Validation de l'infrastructure

| Test | Résultat attendu |
|------|------------------|
| Ping LAN ↔ DMZ | Succès depuis client |
| Ping DMZ → LAN | Bloqué par pfSense (si configuré) |
| HTTP LAN → DMZ | Page web accessible |
| Résolution DNS | entreprise.local résolu |
| vCenter accessible | https://192.168.140.155 répond |
| ESXi accessible | https://192.168.140.150 répond |

---

## 12. VMotion - Migration à chaud (Optionnel)

> **Note :** Cette section nécessite un deuxième hôte ESXi (ESXi-02 : 192.168.140.151)

### 12.1 Préparation du stockage

1. Sur l'hôte cible (192.168.140.151), créer un nouveau datastore
2. Type : VMFS
3. Nom : Datastore-ESXi02

### 12.2 Configuration du réseau

1. Sur l'hôte cible, créer les mêmes Port Groups (LAN-VLAN20, DMZ-VLAN30, etc.)
2. Activer vMotion sur les adaptateurs VMkernel des deux hôtes

### 12.3 Lancement de la migration

1. Clic droit sur la VM → **Migrer**
2. Type : **Modifier la ressource de calcul et le stockage**
3. Sélectionner l'hôte cible (192.168.140.151)
4. Sélectionner le datastore : Datastore-ESXi02
5. Vérifier les réseaux
6. Priorité : Élevée
7. Terminer

### 12.4 Vérification

- La tâche **Relocate VM** doit être à 100%
- La VM doit apparaître sous l'hôte .151
- Vérifier que l'hôte et le datastore ont changé dans l'onglet **Résumé**

---

## 13. Gestion des Snapshots

### 13.1 Qu'est-ce qu'un snapshot ?

Un **snapshot** (instantané) est une copie de l'état d'une VM à un instant précis.

**Utilité :**
- Sauvegarder avant une modification critique
- Tester une mise à jour (rollback si problème)
- Créer des points de restauration

⚠️ **Attention :** Les snapshots ne sont PAS des sauvegardes ! Ne pas les conserver longtemps (impact performance).

### 13.2 Création d'un snapshot

1. Sélectionner la VM (ex: SRV-AD-01)
2. Clic droit → **Snapshots → Take Snapshot**
3. Configuration :
   - Name : `Avant_Config_GPO`
   - Description : `État du serveur AD avant configuration des GPO`
   - Cocher : `Snapshot the virtual machine's memory`
   - Cocher : `Quiesce guest file system`
4. Cliquer sur **OK**

### 13.3 Restaurer un snapshot

1. Clic droit sur la VM → **Snapshots → Manage Snapshots**
2. Sélectionner le snapshot désiré
3. Cliquer sur **Restore**
4. Confirmer avec **Yes**

### 13.4 Bonnes pratiques

| Recommandation | Explication |
|----------------|-------------|
| Durée de vie courte | Max 24-72h pour éviter impact performance |
| Avant modifications | Créer systématiquement avant mise à jour |
| Pas pour backup | Un snapshot dépend du disque original |
| Surveiller l'espace | Les snapshots consomment de l'espace disque |
| Documenter | Utiliser nom et description clairs |
| Consolider régulièrement | Supprimer les snapshots obsolètes |

---

## 14. Bonnes pratiques TP imbriqué

- **NIC1 = NAT** → Management / Internet
- **NIC2 = Host-Only** → Production / LAN / DMZ
- **2 disques** → 1 pour OS ESXi, 1 pour datastore
- **vhv.enable = TRUE, hypervisor.cpuid.v0 = FALSE**
- **VLANs séparés** pour Management / LAN / DMZ / WAN
- **RAM / CPU suffisants** (16 Go RAM / 4 vCPU ESXi minimum)

---

## 15. Plan d'adressage récapitulatif

### 15.1 Réseau Management (192.168.140.0/24)

| Machine | IP | Gateway | DNS | Utilisateur | Mot de passe |
|---------|-----|---------|-----|-------------|--------------|
| ESXi-01 | 192.168.140.150 | 192.168.140.2 | 8.8.8.8 | root | Admin!2026 |
| ESXi-02 | 192.168.140.151 | 192.168.140.2 | 8.8.8.8 | root | Admin!2026 |
| vCenter | 192.168.140.155 | 192.168.140.2 | 8.8.8.8 | administrator@vsphere.local | P@ssword123 |

### 15.2 Réseau LAN (192.168.20.0/24 - VLAN 20)

| Machine | IP | Gateway | DNS | Utilisateur | Mot de passe |
|---------|-----|---------|-----|-------------|--------------|
| pfSense LAN | 192.168.20.1 | - | - | admin | P@ssword123 |
| SRV-AD-01 | 192.168.20.10 | 192.168.20.1 | 127.0.0.1 | Administrateur | Admin2026 |
| CLI-WIN10-01 | 192.168.20.20 | 192.168.20.1 | 192.168.20.10 | - | - |

### 15.3 Réseau DMZ (192.168.30.0/24 - VLAN 30)

| Machine | IP | Gateway | DNS | Utilisateur | Mot de passe |
|---------|-----|---------|-----|-------------|--------------|
| pfSense DMZ | 192.168.30.1 | - | - | - | - |
| SRV-WEB-DMZ | 192.168.30.10 | 192.168.30.1 | 8.8.8.8 | admin-web | admin2026 |

### 15.4 Interfaces pfSense

| Interface | Réseau | IP | DHCP |
|-----------|--------|-----|------|
| WAN (vmx0) | NAT | DHCP (auto) | - |
| LAN (vmx2) | VLAN 20 | 192.168.20.1 | 192.168.20.10-50 |
| DMZ (vmx1) | VLAN 30 | 192.168.30.1 | 192.168.30.10-50 |

---

## Conclusion

Ce TP vous a permis de mettre en place une **infrastructure d'entreprise complète**, incluant :

- Un hyperviseur ESXi imbriqué avec vCenter Server pour la gestion centralisée
- Une segmentation réseau avec VLANs (Management, LAN, DMZ, WAN)
- Un firewall pfSense pour sécuriser et isoler les différents réseaux
- Un contrôleur de domaine Active Directory avec DNS
- Un poste client Windows joint au domaine
- Un serveur web en DMZ (isolation sécurisée)
- La gestion des snapshots pour la sauvegarde et la restauration
- La migration à chaud (VMotion) entre deux hôtes ESXi

Cette infrastructure constitue la base d'une **architecture d'entreprise moderne** et vous permet de comprendre les concepts essentiels de la virtualisation, de la segmentation réseau et de la sécurité informatique.

**Félicitations pour avoir complété ce TP !** 🎉
