Mise en œuvre d'un serveur de fichiers Windows Server 2025 dans un domaine Active Directory `lab.lan`, avec gestion des permissions par groupes de sécurité.
 
---
 
### Objectifs
 
- Centraliser les fichiers bureautiques sur un serveur de fichiers
- Mettre en œuvre le protocole **SMB** (Server Message Block) pour le partage
- Appliquer des permissions différenciées selon les groupes métier
### Environnement de lab
 
| Élément | Détails |
|---|---|
| Hyperviseur | VirtualBox |
| Réseau | Internal Network — `172.16.10.0/24` |
| Serveur | **LABSRV** — Windows Server 2025 (DC, DNS, DHCP, File Server) — `172.16.10.1` |
| Client | **CLI01** — Windows 11 Pro (joint au domaine) |
| Domaine | `lab.lan` |
 
---
 
## 2. Structure Active Directory
 
### Organisation des OU et groupes
 
```
lab.lan
└── OU Departements
    ├── Groupe RH (Domain Local, Security)
    ├── Groupe Comptabilite (Domain Local, Security)
    ├── Groupe Direction (Domain Local, Security)
    ├── User user.rh           -> membre du groupe RH
    ├── User user.compta       -> membre du groupe Comptabilite
    └── User user.dir          -> membre du groupe Direction
```
 
> **Note** — Le groupe AD est nommé `Comptabilite` sans accent (bonne pratique pour éviter les problèmes de compatibilité avec PowerShell et les chemins UNC), mais le **dossier sur le disque** s'appelle `Comptabilité` avec accent comme le demande la consigne.
 
### Création via l'interface Active Directory Users and Computers
 
> Étapes réalisées en GUI.
 
![OU Departements avec ses groupes et users](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/OU-and-Users.png)
 
---
 
## 3. Installation du rôle File Server
 
### Procédure en GUI
 
1. Ouvrir le **Server Manager**
2. `Manage` -> `Add Roles and Features`
3. Choisir **Role-based or feature-based installation**
4. Sélectionner le serveur `LABSRV`
5. Cocher **File and Storage Services** -> **File and iSCSI Services** -> **File Server**
6. Suivre l'assistant jusqu'à la fin
7. 

![File server ok](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/Sharing-Docs-activated.png)
 
---
 
## 4. Création de l'arborescence et du partage
 
### Création des dossiers
 
> Réalisé via le **File Explorer** : clic droit -> `New` -> `Folder`
 
Arborescence créée :
 
```
C:\
└── Documents_Entreprise\
    ├── RH\
    ├── Comptabilité\
    └── Direction\
```
 

 
### Création du partage
 
> Réalisé via le **Server Manager** -> `File and Storage Services` -> `Shares` -> `New Share` -> `SMB Share - Quick`.
 
- **Local path** : `C:\Documents_Entreprise`
- **Share name** : `Docs`
- **Remote path** : `\\LABSRV\Docs`

 
---
 
## 5. Configuration des permissions NTFS
 
### Matrice des permissions cible
 
| Dossier | Principal | Permissions |
|---|---|---|
| `Documents_Entreprise` | Domain Users | Read & Execute |
| `Documents_Entreprise` | Direction | Modify |
| `Documents_Entreprise\RH` | RH | Modify |
| `Documents_Entreprise\RH` | Direction | Modify |
| `Documents_Entreprise\Comptabilité` | Comptabilite | Modify |
| `Documents_Entreprise\Comptabilité` | Direction | Modify |
| `Documents_Entreprise\Direction` | Direction | Modify |
 
> **Note sur la traduction "lecture/écriture" -> NTFS** : la consigne demande un accès en "lecture/écriture". J'ai utilisé la permission NTFS **Modify** qui regroupe Read, Read & execute, List folder contents, Write, et le droit de suppression. C'est l'équivalent standard de "lecture/écriture" en environnement Windows. La permission **Full Control** a été évitée car elle inclut le droit de modifier les permissions, ce qui n'est pas souhaité pour des utilisateurs métier.

![]


 
### Configuration en GUI
 
Pour chaque sous-dossier (`RH`, `Comptabilité`, `Direction`) :
 
1. **Clic droit** sur le dossier -> `Properties` -> onglet `Security`
2. Bouton `Advanced` -> **Disable inheritance** -> choisir `Convert inherited permissions into explicit permissions`
3. **Remove** les entrées trop larges (`Users`, `Domain Users` si présent)
4. **Conserver** les entrées système : `Administrators`, `SYSTEM`, `CREATOR OWNER`
5. **Add** le ou les groupes métier avec la permission **Modify**
Pour la racine `Documents_Entreprise` :
- Inheritance **enabled** (par défaut)
- Ajout de **Domain Users** avec Read & Execute
- Ajout de **Direction** avec Modify

 
![Screenshot : permissions NTFS sur RH](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/Permissions-RH-on-server.png)
 
![Screenshot : permissions NTFS sur Comptabilité](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/Permissions-Comptabilite-on-server.png)
 
[Screenshot : permissions NTFS sur Direction](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/Permissions-direction-on-server.png)
 
---
 
## 6. Vérification des partages en PowerShell
 
### Lister tous les partages SMB du serveur
 
```powershell
Get-SmbShare
```
 
### Vérification des permissions par fichier en Powershell
 
```powershell
Get-Acl | Format-List
```
Permissions sur Documents Entrerprise
![Permission Documents Entreprise](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/Powershell-Permissions-Documents_Entreprise.png)

Permissions dossier RH
![Permission dossier RH](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/Powershell-Permissions-Documents_Entreprise-RH.png)

Permissions dossier Comptabilité
![Permissions Compta](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/Powershell-Permissions-Documents_Entreprise-Comptabilite.png)

Permissions dossier Direction
![Permissions Direction](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/Powershell-Permissions-Documents_Entreprise-Direction.png)
 
### Permissions de partage (Share)
 
```powershell
Get-SmbShareAccess -Name "Docs"
```
 
Résultat obtenu :
 
```
Name AccountName        AccessControlType AccessRight
---- -----------        ----------------- -----------
Docs Everyone           Allow             Full
```
 
> **Note importante** — Les permissions de partage SMB sont configurées à **Everyone : Full** par défaut sur un SMB Share Quick. Cela ne signifie PAS que tout le monde peut tout faire : la **sécurité effective est l'intersection** des permissions Share et NTFS (la plus restrictive gagne).
>
> En mettant le partage en grand ouvert et en filtrant tout au niveau **NTFS**, on suit la pratique recommandée : **un seul endroit où gérer les droits**, donc moins d'erreurs et de configurations divergentes. Les vraies restrictions sont définies par les ACL NTFS de l'étape 5.
 
 
---
 
## 7. Configuration du lecteur réseau côté client
 
### Mapping du lecteur Z: via PowerShell
 
> Réalisé sur **CLI01** (Windows 11 Pro), connecté en tant qu'utilisateur du domaine.
 
```powershell
New-PSDrive -Name "Z" -PSProvider FileSystem -Root "\\LABSRV\Docs" -Persist
```
 
### Vérification du mapping
 
```powershell
Get-PSDrive -PSProvider FileSystem
```
 
> **À noter** — Le mapping `-Persist` est lié au profil utilisateur. Chaque utilisateur de test devra exécuter la commande au moins une fois dans sa session.
 
 
---
 
## 8. Tests d'accès avec différents utilisateurs
 
### Méthodologie
 
Pour chaque utilisateur, le test suit le même protocole :
 
1. **Connexion** à CLI01 avec l'utilisateur testé
2. **Vérification de l'identité** : `whoami` et `whoami /groups`
3. **Mapping** du lecteur Z: vers `\\LABSRV\Docs`
4. **Tests d'écriture** dans chaque sous-dossier via `New-Item`
5. **Capture** des succès et des échecs (`Access denied`)


### Test 1 : user.rh (membre du groupe RH)

Mapping et tests en tant qu'user du groupe RH
![Mapping RH](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/Mapping-and-tests-user-rh.png)


### Test 2 : user.compta (membre du groupe Comptabilite)

Mapping et tests en tant qu'user du groupe Comptabilité
![Mapping RH](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/Mapping-and-tests-user-compta.png)

 
### Test 3 : user.dir (membre du groupe Direction)

Mapping et tests en tant qu'user du groupe Direction
![Mapping Direction](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/Mapping-and-tests-user-direction.png)

![Mapping Direction2](https://github.com/Camille-Cybear/Atelier-partage-fichiers/blob/c25b3866afce54ec869dcb6f95cb8064a41b1942/Pictures/Mapping-and-tests-user-direction2.png)
 
---
 
### Matrice des accès observés
 
| Utilisateur | Groupe AD | `Z:\` (racine) | `Z:\RH` | `Z:\Comptabilité` | `Z:\Direction` |
|---|---|---|---|---|---|
| `user.rh` | RH | Read | Modify | Access denied | Access denied |
| `user.compta` | Comptabilite | Read | Access denied | Modify | Access denied |
| `user.dir` | Direction | Modify | Modify | Modify | Modify |
