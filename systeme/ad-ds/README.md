# Déploiement d’un domaine Active Directory avec Windows Server 2022

Lab réalisé le 28 septembre 2026 sous VMware Workstation : AD DS, DNS, unités d’organisation, groupes, partages SMB, GPO et jonction d’un poste Windows 11.

**Auteur :** Khadim Mbacké Thiao
**Environnement :** Windows Server 2022 Standard Evaluation (DC-1, 192.168.10.2) et Windows 11 Professionnel 25H2 (IT_KMT_00, 192.168.10.6)
**Domaine :** lab.local (NetBIOS : LAB), réseau isolé 192.168.10.0/24

## Objectif

Reproduire un réseau d’entreprise simple : un annuaire centralisé, un DNS fiable, des droits d’accès par service et des lecteurs réseau attribués automatiquement aux utilisateurs.

![Schéma de l’infrastructure cible](images/00-schema-architecture.png)

**Arborescence de l’annuaire :**
- `OU_Users` : une OU par service (DG, DAF, IT, RH, Commercial)
- `OU_Grpe` : un groupe par service (GRPE_*)
- `OU_DA` : groupes de droits d’accès, `DA_DC-1_<service>_LECTURE` et `_MODIF`
- `OU_Partage` : objets « Dossier partagé »
- `OU_PC` : ordinateurs du domaine

## 1. Réseau du serveur

Le futur contrôleur de domaine reçoit une adresse IP fixe : 192.168.10.2/24. Son DNS préféré est 127.0.0.1, car il hébergera lui-même le DNS. Pas de passerelle, le réseau est isolé.

![Configuration IPv4 statique de DC-1](images/01-ipv4-serveur.png)
*Configuration IPv4 statique de DC-1*

## 2. Installation des rôles AD DS et DNS

Avec l’assistant « Ajout de rôles et de fonctionnalités » (installation basée sur un rôle, serveur DC-1), j’ai coché uniquement **Services AD DS** et **Serveur DNS** pour limiter la surface d’attaque. L’installation se termine par « Configuration requise » : les rôles sont là, mais le serveur n’est pas encore contrôleur de domaine.

![Sélection des rôles AD DS et DNS](images/02-roles-ad-dns.png)
*Sélection des rôles AD DS et DNS*

![Installation réussie, promotion à faire](images/03-installation-reussie.png)
*Installation réussie, promotion à faire*

## 3. Promotion en contrôleur de domaine

J’ai créé une **nouvelle forêt** `lab.local`, puisque c’est le premier contrôleur du domaine.

![Nom de domaine racine](images/04-nom-domaine.png)
*Nom de domaine racine*

Paramètres choisis : niveaux fonctionnels Windows Server 2016 (le maximum proposé), serveur DNS et catalogue global cochés, RODC décoché, et un mot de passe DSRM pour la restauration de l’annuaire.

![Options du contrôleur de domaine](images/05-options-dc.png)
*Options du contrôleur de domaine*

L’assistant génère le script PowerShell équivalent (`Install-ADDSForest`). Le mot de passe DSRM n’y apparaît pas.

![Script PowerShell généré par l’assistant](images/06-script-powershell.png)
*Script PowerShell généré par l’assistant*

La vérification des prérequis réussit, avec deux avertissements sans gravité : la délégation DNS est impossible (lab.local n’a pas de zone parente) et Windows Server 2022 bloque par défaut les algorithmes de chiffrement faibles de type NT 4.0. Le serveur redémarre ensuite automatiquement.

![Vérification des prérequis](images/07-verification-prerequis.png)
*Vérification des prérequis*

![Tableau de bord après redémarrage](images/08-tableau-de-bord.png)
*Tableau de bord après redémarrage*

## 4. DNS

Active Directory dépend entièrement du DNS. J’ai vérifié la résolution avec `nslookup`, puis créé une **zone de recherche inversée** pour 192.168.10.0/24 (10.168.192.in-addr.arpa), intégrée à Active Directory, avec uniquement les **mises à jour dynamiques sécurisées** : seuls les clients authentifiés peuvent modifier les enregistrements.

![Résolution de lab.local](images/09-nslookup-domaine.png)
*Résolution de lab.local*

![Zone inverse : ID réseau 192.168.10](images/10-zone-inverse-id.png)
*Zone inverse : ID réseau 192.168.10*

![Mises à jour dynamiques sécurisées](images/11-maj-dynamiques.png)
*Mises à jour dynamiques sécurisées*

L’enregistrement PTR de DC-1 est bien présent et la résolution inverse fonctionne.

![Enregistrement PTR de DC-1](images/12-enregistrement-ptr.png)
*Enregistrement PTR de DC-1*

![Résolution inverse](images/13-nslookup-inverse.png)
*Résolution inverse*

## 5. Annuaire : OU, utilisateurs et groupes

J’ai séparé utilisateurs, groupes, droits d’accès, partages et ordinateurs dans des OU distinctes, car les GPO se lient aux OU et non aux conteneurs par défaut. Une sous-OU par service permet de cibler précisément les stratégies de groupe.

![OU principales sous OU_lab.local](images/14-ou-principales.png)
*OU principales sous OU_lab.local*

![Sous-OU par service et création d’un utilisateur](images/15-sous-ou-utilisateur.png)
*Sous-OU par service et création d’un utilisateur*

Les comptes sont créés directement dans l’OU de leur service, avec l’identifiant `prenom-nom`. Seule l’option « changer le mot de passe à la prochaine ouverture de session » est cochée. Cocher aussi « n’expire jamais » provoque une erreur : les deux options sont incompatibles.

![Création de l’utilisateur](images/16-creation-utilisateur.png)
*Création de l’utilisateur*

![Erreur : options de mot de passe incompatibles](images/17-conflit-mot-de-passe.png)
*Erreur : options de mot de passe incompatibles*

Pour les droits, j’ai créé un groupe de sécurité par service, plus deux groupes d’accès par dossier (lecture et modification). Les droits sont attribués à des groupes, jamais à des individus : l’utilisateur est membre de GRPE_IT.

![Groupes de droits d’accès dans OU_DA](images/18-groupes-da.png)
*Groupes de droits d’accès dans OU_DA*

![Membres du groupe GRPE_IT](images/19-membres-grpe-it.png)
*Membres du groupe GRPE_IT*

## 6. Dossiers et partages SMB

Un dossier par service est partagé avec l’assistant « Nouveau partage » (profil SMB Rapide, chemin personnalisé). J’ai activé l’énumération basée sur l’accès : chaque utilisateur ne voit que les dossiers auxquels il a droit.

![Dossiers des services](images/20-dossiers-services.png)
*Dossiers des services*

![Emplacement du partage](images/21-chemin-partage.png)
*Emplacement du partage*

![Paramètres de partage](images/22-parametres-partage.png)
*Paramètres de partage*

La restriction se fait au niveau **NTFS**, pas au niveau du partage (Tout le monde en contrôle total) : c’est le plus restrictif des deux qui s’applique. Pour DIRECTION GENERALE, j’ai désactivé l’héritage en **convertissant** les droits hérités en droits explicites, pour ne pas me verrouiller hors du dossier.

![Désactivation de l’héritage](images/23-heritage.png)
*Désactivation de l’héritage*

Deux groupes reçoivent les droits : `DA_DC-1_DG_MODIF` (Modification, sans Contrôle total, donc sans pouvoir changer les permissions) et `DA_DC-1_DG_LECTURE` (lecture seule).

![Permission Modification](images/24-permission-modif.png)
*Permission Modification*

![Permission Lecture](images/25-permission-lecture.png)
*Permission Lecture*

![Liste finale des permissions NTFS](images/26-permissions-ntfs.png)
*Liste finale des permissions NTFS*

![Partage créé](images/27-partage-cree.png)
*Partage créé*

## 7. Publication des partages et partage commun

Les partages sont référencés dans l’annuaire sous forme d’objets « Dossier partagé » dans `OU_Partage`. Le partage COMMUN reçoit le groupe « Utilisateurs du domaine », car il est destiné à tous.

![Objets Dossier partagé dans OU_Partage](images/28-dossiers-partages-ad.png)
*Objets Dossier partagé dans OU_Partage*

![Permissions du partage COMMUN](images/29-partage-commun.png)
*Permissions du partage COMMUN*

## 8. GPO : mappage automatique des lecteurs

Plutôt que de connecter les lecteurs à la main, j’ai utilisé les préférences de stratégie de groupe (Configuration utilisateur, Mappages de lecteurs) : le lecteur suit l’utilisateur sur n’importe quel poste. Une GPO par service (J, K, L, M, N) et une GPO commune (O), nommées `GPO_DC-1_MAP_<lettre>_<service>`, créées une fois puis liées aux OU concernées.

![Création d’un lecteur mappé dans la GPO](images/30-gpo-lecteur-mappe.png)
*Création d’un lecteur mappé dans la GPO*

![Liaison des GPO à OU_Commercial](images/31-liaison-gpo.png)
*Liaison des GPO à OU_Commercial*

## 9. Jonction du poste Windows 11

Le client est configuré avec une IP fixe (192.168.10.6) et le DNS du contrôleur (192.168.10.2), condition indispensable pour retrouver les services Active Directory. Il est ensuite renommé `IT_KMT_00` et joint à `lab.local` avec un compte autorisé, puis redémarré.

![Configuration IP du client](images/32-ipconfig-client.png)
*Configuration IP du client*

![Changement de nom et de domaine](images/33-nom-et-domaine.png)
*Changement de nom et de domaine*

![Jonction réussie](images/34-jonction-reussie.png)
*Jonction réussie*

L’objet ordinateur apparaît côté serveur, dans le conteneur Computers.

![IT_KMT_00 dans Active Directory](images/35-ordinateur-dans-ad.png)
*IT_KMT_00 dans Active Directory*

## 10. Validation

Le poste propose la connexion au domaine LAB.LOCAL. Un utilisateur se connecte et doit choisir son mot de passe dès la première connexion, grâce à l’option cochée à l’étape 5. Les lecteurs IT (K:) et COMMUN (O:) apparaissent sans aucune manipulation : les GPO s’appliquent.

![Écran de connexion au domaine](images/36-connexion-domaine.png)
*Écran de connexion au domaine*

![Changement de mot de passe à la première connexion](images/38-changement-mot-de-passe.png)
*Changement de mot de passe à la première connexion*

![Lecteurs réseau mappés par les GPO](images/37-lecteurs-reseau.png)
*Lecteurs réseau mappés par les GPO*

## Compétences mobilisées

- Déploiement d’un contrôleur de domaine : rôles AD DS et DNS, promotion, forêt, DSRM
- DNS : zones directes et inverses intégrées à AD, mises à jour sécurisées, `nslookup`
- Conception d’annuaire : OU par fonction et par service, conventions de nommage
- Gestion des accès : groupes de service et d’accès, moindre privilège, permissions NTFS
- GPO : mappage de lecteurs, liaison aux OU
- Administration de postes clients : configuration réseau, jonction au domaine

## Limites du lab et améliorations

- Un seul contrôleur de domaine : il en faudrait au moins deux en production
- Nom de domaine en `.local` : à remplacer par un sous-domaine d’un nom DNS détenu
- Base AD, journaux, SYSVOL et données des partages sur le disque système : à placer sur un volume dédié
- Chiffrement SMB désactivé : à activer pour des données sensibles
- Poste client dans le conteneur Computers : à déplacer dans `OU_PC` pour lui appliquer des GPO ordinateur
- Événements d’avertissement après la promotion (DNS, DFSR) : à examiner dans l’Observateur d’événements
- Création des OU, utilisateurs et groupes : automatisable en PowerShell
