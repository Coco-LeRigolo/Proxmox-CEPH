# **Rapport de Projet : Cluster Hyper-convergé Proxmox & Ceph**

_Équipe : Paulin, Armand, Zakari, Maxime, Corentin_

**Sujet :** Mise en place d'un cluster de virtualisation haute disponibilité (3 nœuds) avec stockage distribué Ceph.

Sommaire :

[**1\. Architecture et Plan d'Adressage 2**](#_10111rg143cd)

  - [Schéma Logique](#_ocrid1wnc88k)

  - [Plan d'Adressage IP](#_ed7nm5xf90qg)

[**2\. Mise en Œuvre Technique (Accomplissements & Procédure)**](#_xjglbqx62ki5)

  - [A. Préparation des Nœuds](#_1azfn4xw83ig)

  - [B. Installation et Configuration de Ceph (Stockage Distribué)](#_7nqqp0ype4gm)

  - [C. Gestion du Stockage (OSD & Pools)](#_kstgfmzfhhsf)

  - [D. Validation Fonctionnelle (Déploiement VM)](#_7ghhe6foh1k9)

[**3\. Retours d'Expérience et Incidents**](#_f1p36qm6wi7h)

  - [Incident Majeur #1 : Corruption de Cluster (Duplication de Hostname)](#_6lt37pf25b12)

  - [Incident #2 : Disques invisibles pour Ceph (OSD)](#_2f3ty2ghnz0q)

  - [Incident #3 : Résolution DNS (Mises à jour impossibles)](#_j1v9wmliq6gj)

  - [Incident #4 : Perte d'un OSD et Résilience du Stockage](#_y4jvroq7mmjr)

[**4\. Limitations et Reste à faire**](#_tn0zftybu501)

[**5\. Conclusion Générale**](#_pz1v068kdz88)

## 

# **1\. Architecture et Plan d'Adressage**

Pour ce projet, nous avons déployé une infrastructure composée de trois serveurs physiques et d'un poste d'administration, reliés via un switch central.

## **Schéma Logique**
![alt text]( https://github.com/Coco-LeRigolo/Proxmox-CEPH/blob/main/Image1.png "Schéma Logique")

### 

## **Plan d'Adressage IP**

Voici la configuration réseau appliquée à notre laboratoire (Réseau 192.168.100.0/24) :

| **Équipement** | **Hôte (Hostname)** | **Adresse IP** | **Passerelle (GW)** | **Rôle** |
| --- | --- | --- | --- | --- |
| **SERV 1** | pve | 192.168.100.25 | .254 | Nœud Proxmox 1 |
| --- | --- | --- | --- | --- |
| **SERV 2** | pve-paulin | 192.168.100.30 | .254 | Nœud Proxmox 2 |
| --- | --- | --- | --- | --- |
| **SERV 3** | pve-zakry | 192.168.100.35 | .254 | Nœud Proxmox 3 |
| --- | --- | --- | --- | --- |
| **PC ADMIN** | PC_ADM | 192.168.100.21 | .254 | Gestion & Accès Web |
| --- | --- | --- | --- | --- |

**Note sur l'infrastructure :** Nous avons choisi une topologie en étoile. Le port 1/1 du switch dessert le Serveur 1, le 1/2 le Serveur 2, etc.

# **2\. Mise en Œuvre Technique (Accomplissements & Procédure)**

Nous avons réussi à déployer un cluster fonctionnel avec stockage partagé. Voici les étapes clés validées, accompagnées des procédures techniques appliquées.

## **A. Préparation des Nœuds**

**Contexte :** L'installation de base de Proxmox VE (PVE) a été réalisée sur les trois serveurs. Nous avons veillé à la cohérence entre les interfaces physiques connectées au switch et les ponts réseaux (vmbr0) configurés dans l'OS.

**Procédure Technique : Standardisation des Nœuds** Avant de créer le cluster, nous avons harmonisé la configuration via le terminal (Shell) pour éviter les conflits d'identité et de réseau.

1.  **Correction de la résolution DNS (/etc/resolv.conf)**
    - Nécessaire pour télécharger les paquets si le DNS par défaut échoue.
    - Commande :
    ```
    nano /etc/resolv.conf
    ```
    - Action : Ajout manuel de
    ```
    nameserver 8.8.8.8
    ```
2.  **Validation du nom d'hôte unique (Hostname)**
    - Crucial pour éviter la corruption du cluster (cf. Incident #1).
    - Commande :
    ```
    hostnamectl set-hostname \[nom_du_noeud\]
    ```
    - Vérification : Contrôle dans /etc/hosts que l'IP locale pointe vers le bon nom.

### **B. Installation et Configuration de Ceph (Stockage Distribué)**

**Contexte :** Pour permettre la haute disponibilité et la migration des VMs, nous avons déployé Ceph en mode "Hyper-convergé".

- **Version :** "Reef" (PVE 8) sur les 3 nœuds.
- **Réseau :** Utilisation du Public Network (192.168.100.0/24) pour le trafic Ceph faute de lien dédié.
- **Monitors (MON) :** 3 moniteurs (un par nœud).
- **Managers (MGR) :** Ajout sur deux nœuds.

**Procédure Technique : Déploiement Ceph**

1.  **Installation (Via Interface Web sur chaque nœud)**
    - Chemin : **Datacenter > \[Nœud\] > Ceph > Install**.
    - Sélection : Version Reef (18.2) / Repository : No-Subscription.
2.  **Configuration du Cluster (Nœud 1 uniquement)**
    - Chemin : **Configuration > Create Configuration**.
    - Réseaux : Sélectionner 192.168.100.0/24 pour _Public_ et _Cluster Network_.  
        
3.  **Ajout des Monitors (Nœuds 2 et 3)**
    - Chemin : **Ceph > Monitor > Create**.
    - Action : Sélectionner le nœud courant pour atteindre le quorum de 3.

**Procédure Technique : Déploiement Ceph**

**1\. Installation (Via Interface Web sur chaque nœud) :**

- **Datacenter > \[Nœud\] > Ceph > Install**.
- Sélection version : **Reef (18.2)**.
- Repository : **No-Subscription**.

**2\. Configuration du Cluster (Sur le Nœud 1 uniquement) :**

- Onglet **Configuration > Create Configuration**.
- **Public Network :** 192.168.100.0/24.
- **Cluster Network :** 192.168.100.0/24 (Même réseau sélectionné).

**3\. Ajout des Monitors (Sur Nœud 2 et 3) :**

- **Ceph > Monitor > Create >** Sélectionner le nœud courant pour atteindre le quorum de 3.

## **C. Gestion du Stockage (OSD & Pools)**

**Contexte :** Nous avons intégré un disque dur dédié par serveur pour créer un pool nommé vm-storage avec une réplication de 3/2 (Size=3, Min_Size=2).

**Procédure Technique : Préparation des Disques et Pools**

1.  **Nettoyage Bas Niveau (CLI)**
    - _Action nécessaire car Proxmox rejetait les disques contenant des signatures LVM._
    - Identifier le disque :
    ```
    fdisk -l
    ```
    - Supprimer les signatures :
    ```
    wipefs -a /dev/sdb
    ```
    - Remise à zéro (si nécessaire) :
    ```
    dd if=/dev/zero of=/dev/sdb bs=1M count=100
    ```
    - Initialisation Ceph :
    ```
    ceph-volume lvm zap /dev/sdb
    ```        
2.  **Création du Pool (Interface Web)**
    - Chemin : **Datacenter > Ceph > Pools > Create**.
    - Paramètres : Name vm-storage, Size 3, Min. Size 2.
    - **Add Storage :** Case cochée (lien automatique RBD).

Résultat:

![alt text]( https://github.com/Coco-LeRigolo/Proxmox-CEPH/blob/main/Image2.png "Résultat OSD")

## **D. Validation Fonctionnelle (Déploiement VM)**

Pour valider le bon fonctionnement de notre infrastructure, nous avons effectué un test grandeur nature :

1.  **Création d'une VM maître :** Installation d'une machine virtuelle Linux de base sur le stockage Ceph.
2.  **Duplication :** Nous avons cloné cette VM pour en disposer d'une copie sur chaque nœud du cluster.

Cette opération a confirmé que le pool Ceph vm-storage est bien accessible en lecture/écriture par tous les hyperviseurs et que la réplication des données s'effectue correctement.

**Procédure Technique : Test de Réplication**

**1\. Création de la VM :**

- Create **VM > Onglet Disques > Storage** : Choisir **ceph-shared** (le stockage RBD créé précédemment).

**2\. Clonage :**

- Clic droit sur la **VM > Clone**.
- **Mode :** Full Clone (pour une copie indépendante).
- **Target Node :** Sélection d'un autre serveur (ex: pve-armand).
    - _Résultat : La VM démarre sur le second nœud en accédant aux mêmes données réparties via le réseau._

# **3\. Retours d'Expérience et Incidents**

C'est la partie la plus critique du projet. Nous avons rencontré plusieurs obstacles techniques majeurs qui nous ont permis de mieux comprendre le fonctionnement interne de Proxmox.

## **Incident Majeur #1 : Corruption de Cluster (Duplication de Hostname)**

**Le Problème (Contexte) :** Lors de la tentative de création du cluster, les deux premiers serveurs possédaient le même nom d'hôte par défaut (pve). La commande de jonction a échoué car Corosync (le service de quorum) ne pouvait pas distinguer les machines. Cela a provoqué une corruption de la base de données du cluster (pmxcfs) car les deux nœuds tentaient d'écrire simultanément au même endroit.

**Solution :** Le cluster étant irrécupérable, nous avons dû réinstaller le nœud fautif pour lui attribuer une identité unique.

**Procédure Technique : Réinstallation et Correction d'Identité**

1.  **Préparation du Support :**
    - Insérer la clé USB d'installation de Proxmox VE dans le serveur concerné.
    - Redémarrer et booter sur la clé.
2.  **Configuration Critique (Installeur Graphique) :**
    - Lancer l'installation jusqu'à l'étape **Management Network Configuration**.
    - **Hostname (FQDN) :** Au lieu de laisser la valeur par défaut (pve.local), saisir impérativement un nom unique.
    - _Action :_ Saisie de paulin.local.
    - **IP Address :** Réassigner l'IP statique (ex: 192.168.100.12).
3.  **Retour à la normale et Jonction :**
    - Terminer l'installation et redémarrer.
    - Se connecter à la nouvelle interface web (https://192.168.100.12:8006).
    - Aller dans **Datacenter > Cluster > Join Cluster**.
    - Coller les informations du premier nœud et valider.
    - Résultat : Le cluster accepte le nœud car les noms sont désormais distincts.

## **Incident #2 : Disques invisibles pour Ceph (OSD)**

**Le Problème (Contexte) :** Lors de l'étape "Create OSD" dans l'interface, les disques durs destinés au stockage n'apparaissaient pas dans la liste. **Cause :** Les disques contenaient des résidus de partitions (signatures LVM) d'une installation précédente. Par sécurité, Proxmox masque les disques non vierges.

**Procédure Technique : Nettoyage Bas Niveau (CLI)**

1.  **Accès au Terminal (Shell) :**
    - Sélectionner le **nœud concerné > Shell**.
2.  **Identification et Nettoyage :**
    - Identifier le disque via la commande lsblk (ex: /dev/sdb).
    - Effacer les signatures de fichiers:
    ```
    wipefs -a /dev/sdb
    ```
    - Supprimer la table de partition (remise à zéro):
    ```
    dd if=/dev/zero of=/dev/sdb bs=1M count=100
    ```
    - "Zapper" le disque pour Ceph (commande spécifique):
    ```
    ceph-volume lvm zap /dev/sdb
    ```
3.  **Finalisation (Interface Web) :**
    - Retourner dans **Ceph > OSD > Create: OSD**.
    - Résultat : Le disque est désormais visible ("Unused") et sélectionnable.

## **Incident #3 : Résolution DNS (Mises à jour impossibles)**

**Le Problème (Contexte) :** Impossible de télécharger les paquets Ceph ou d'effectuer les mises à jour système. Le serveur renvoyait l'erreur Temporary failure in name resolution. La configuration DNS de l'interface ne s'appliquant pas correctement au système sous-jacent.

**Procédure Technique : Configuration DNS Manuelle**

1.  **Accès au fichier de configuration :**
    - Ouvrir le Shell du nœud.
    - Éditer le fichier avec nano:
    ```
    nano /etc/resolv.conf
    ```
2.  **Application du correctif :**
    - Ajouter ou modifier la ligne pour pointer vers un DNS public (Google):
      ```
      nameserver 8.8.8.8
      ```
        - Sauvegarder (Ctrl+O) et quitter (Ctrl+X).
3.  **Validation :**
    - Lancer la commande apt update. La connexion aux serveurs Proxmox doit fonctionner immédiatement.

## **Incident #4 : Perte d'un OSD et Résilience du Stockage**

**Le Problème (Contexte) :** Le cluster a perdu la communication avec un disque (OSD.0) suite à un conflit de métadonnées ("Locked/Busy"), bien que le disque physique soit sain. **Accomplissement :** Cet incident a permis de valider la résilience : aucune VM n'a planté grâce à la réplication sur les autres nœuds. Nous avons dû simuler un remplacement de disque à chaud.

**Procédure Technique : Remplacement et Reconstruction**

1.  **Suppression Logique (Interface Web) :**
    - Aller dans **Ceph > OSD**.
    - Sélectionner l'OSD défaillant (down).
    - Menu **More > Destroy**. (L'OSD est retiré de la configuration logicielle).
2.  **Remise à zéro du disque (Shell) :**

- Comme pour l'incident #2, le disque est bloqué. Il faut le forcer:
  ```
  wipefs -a /dev/sda  
  dd if=/dev/zero of=/dev/sda bs=1M count=100
  ```
    - _(Note : Remplacer /dev/sda par le disque concerné)_.

1.  **Recréation (Interface Web) :**
    - **Ceph > OSD > Create: OSD**.
    - Sélectionner le disque (désormais vierge).
    - Valider.
2.  **Observation de la Résilience :**
    - Le nouvel OSD démarre. Ceph lance automatiquement le "Backfill" (copie des données depuis les deux autres nœuds vers le nouveau disque).
    - Attendre le retour à l'état HEALTH_OK.

# **4\. Limitations et Reste à faire**

Faute de temps ou de matériel, certaines parties du projet initial n'ont pas pu être finalisées :

1.  **Réseau dédié Ceph :** Nous faisons passer le trafic de réplication de stockage sur le même câble que le trafic utilisateur et administration. En production, cela créerait des goulots d'étranglement (latence). Il aurait fallu une seconde carte réseau par serveur.
2.  **Haute Disponibilité (HA) des VMs :** Le stockage est prêt (Ceph), mais nous n'avons pas eu le temps de configurer les _groupes HA_ pour que les VMs redémarrent automatiquement sur le Nœud 2 si le Nœud 1 s'éteint brutalement.
3.  **Tests de charge :** Nous n'avons pas pu simuler une coupure électrique d'un nœud pendant une écriture disque pour valider la robustesse du Min_Size: 2.

# **5\. Conclusion Générale**

Ce projet nous a permis de passer de la théorie à la pratique en déployant une infrastructure hyper-convergée complète, similaire à celles utilisées en production dans les entreprises.

**Bilan Technique :** Nous avons réussi à mettre en œuvre un cluster Proxmox de 3 nœuds interconnectés, supporté par un stockage distribué Ceph tolérant aux pannes. L'architecture réseau, bien que contrainte par le matériel disponible (réseau unique pour le trafic et le stockage), est fonctionnelle et opérationnelle.

**Bilan Pédagogique :** Au-delà de l'installation ("le chemin heureux"), c'est la gestion des incidents qui a été la source principale d'apprentissage.

1.  La corruption du cluster par duplication de _hostname_ nous a appris l'importance critique de l'identification unique dans les systèmes distribués.
2.  Les problèmes de partitions disques nous ont forcés à maîtriser les outils bas niveau Linux (wipefs, fdisk, dd).
3.  Enfin, la perte et la reconstruction d'un OSD Ceph nous ont permis de voir la "magie" de l'auto-guérison du cluster en temps réel.

En conclusion, ce laboratoire est une réussite : nous disposons aujourd'hui d'une plateforme robuste capable de survivre à la perte d'un serveur ou d'un disque, validant ainsi les principes de haute disponibilité et de résilience visés par le projet.
