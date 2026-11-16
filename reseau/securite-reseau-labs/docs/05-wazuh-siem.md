# TP 6 : Déploiement du SIEM Wazuh

[← Retour au sommaire](../README.md)

## Objectif

pfSense, Suricata, les journaux SSH et ceux de Windows alertent chacun dans leur coin : l'analyste doit consulter cinq endroits pour
reconstituer une attaque. Un **SIEM** (*Security Information and Event Management*) centralise tout :

1. il **collecte** les logs de toutes les sources ;
2. il les **normalise** dans un format commun ;
3. il **corrèle** les événements pour révéler une attaque en plusieurs étapes ;
4. il **alerte** ;
5. il **archive** pour les investigations.

La corrélation est la vraie valeur ajoutée : un scan, des échecs SSH, une connexion réussie puis un fichier déposé sont anodins isolément,
mais forment une chaîne d'attaque quand ils viennent de la même adresse.

## Architecture de Wazuh

| Composant | Rôle |
|---|---|
| **Manager** | Analyse les logs et applique les règles de détection |
| **Indexer** | Base de données des événements et des alertes |
| **Dashboard** | Interface web de l'analyste |
| **Agent** | Collecteur installé sur chaque machine surveillée (logs, intégrité des fichiers, conformité) ; il parle au Manager sur le port 1514 |

Dans ce lab, les trois premiers composants sont réunis dans la VM Wazuh fournie au format OVA (**All-in-One**), en `192.168.10.10`.

## 1. VM Wazuh

- Import de l'OVA dans VMware, carte réseau en **Host-only**.
- La VM a d'abord reçu une adresse du DHCP (`192.168.10.104`). Plutôt que de la fixer dans la VM, une **réservation DHCP** a été créée
  dans pfSense (**Services > DHCP Server > LAN > DHCP Static Mappings**) : l'adresse MAC de la VM reçoit toujours `192.168.10.10`.
- Accès au dashboard depuis la VM Windows : `https://192.168.10.10` (certificat auto-signé, identifiants par défaut de l'OVA
  à changer hors d'un lab).

![Réservation DHCP](../images/tp6-wazuh/01-reservation-dhcp-wazuh.png)

![Adresse IP de la VM Wazuh](../images/tp6-wazuh/02-console-wazuh-adresse-ip.png)

## 2. Agent sur Ubuntu Server

Le dashboard (**Endpoints > Deploy new agent**) génère la commande selon le système, l'adresse du serveur et le nom de l'agent.
Paramètres : DEB amd64, serveur `192.168.10.10`, nom `ubuntu-server`, groupe `default`.

![Formulaire de déploiement](../images/tp6-wazuh/03-deploiement-agent-ubuntu.png)

Sur Ubuntu Server :

```bash
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.10-1_amd64.deb \
  && sudo WAZUH_MANAGER='192.168.10.10' WAZUH_AGENT_NAME='ubuntu-server' dpkg -i ./wazuh-agent_4.14.10-1_amd64.deb

sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Les deux blocs du dashboard ne font pas la même chose : le premier **installe** l'agent, le second le **démarre**.

![Installation de l'agent sur Ubuntu](../images/tp6-wazuh/04-installation-agent-ubuntu.png)

![Agent ubuntu-server actif](../images/tp6-wazuh/05-agent-ubuntu-actif.png)

## 3. Agent sur Windows 11

Même démarche, avec la commande PowerShell du dashboard, exécutée dans un **PowerShell administrateur** :

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.10-1.msi -OutFile $env:tmp\wazuh-agent
msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='192.168.10.10' WAZUH_AGENT_NAME='windows-workstation'
NET START WazuhSvc
Get-Service WazuhSvc        # Status = Running
```

![Installation de l'agent Windows](../images/tp6-wazuh/06-installation-agent-windows.png)

![Deux agents actifs](../images/tp6-wazuh/07-deux-agents-actifs.png)

## 4. Premiers événements

Dans **Threat Hunting**, on filtre par agent (`agent.name: ubuntu-server`) et par plage de temps.

![Vue générale de Threat Hunting](../images/tp6-wazuh/08-threat-hunting-vue-generale.png)

**Événements générés sur Ubuntu** : connexions SSH avec un mauvais mot de passe vers un utilisateur inexistant (`ssh fakeuser@127.0.0.1`),
création d'un fichier dans `/tmp`, `sudo` avec un mauvais mot de passe.

![Échecs SSH sur Ubuntu](../images/tp6-wazuh/09-ubuntu-ssh-echecs.png)

![Alertes d'authentification en échec](../images/tp6-wazuh/10-alertes-authentification-echec.png)

La règle **5710** (`sshd: Attempt to login using a non-existent user`, niveau 5) s'est déclenchée. Le détail d'une alerte contient
l'agent, la règle, le niveau, la description et les champs décodés depuis le log d'origine.

![Détail d'une alerte Ubuntu](../images/tp6-wazuh/11-detail-alerte-ubuntu.png)

**Événements générés sur Windows** : connexion à une ressource réseau avec de faux identifiants
(`net use \\127.0.0.1\C$ /user:FakeAdmin WrongPassword`). Wazuh remonte la règle **60122** (`Logon Failure - Unknown user or bad password`,
niveau 5), issue de l'événement Windows 4625.

![Connexion échouée sur Windows](../images/tp6-wazuh/14-windows-connexion-echouee.png)

![Événements Windows](../images/tp6-wazuh/12-evenements-windows.png)

![Détail d'une alerte Windows](../images/tp6-wazuh/13-detail-alerte-windows.png)

## 5. Logs de pfSense en Syslog

Un agent ne s'installe pas sur pfSense. C'est pfSense qui **envoie** ses logs en Syslog vers le manager.

**Côté pfSense** : **Status > System Logs > Settings > Remote Logging Options**, activation, serveur `192.168.10.10:514`,
contenu *Everything*.

![Remote Logging dans pfSense](../images/tp6-wazuh/15-pfsense-remote-logging.png)

**Côté Wazuh** : un second bloc `<remote>` en mode `syslog` dans `/var/ossec/etc/ossec.conf`, qui n'accepte que l'adresse de pfSense
(extrait dans [`configs/wazuh`](../configs/wazuh/ossec-syslog-remote.xml)). Puis `sudo systemctl restart wazuh-manager`.

![Bloc remote syslog](../images/tp6-wazuh/16-ossec-conf-remote-syslog.png)

![Manager actif](../images/tp6-wazuh/17-wazuh-manager-actif.png)

**Vérification en trois temps**, parce que les logs de pfSense n'apparaissaient pas dans le dashboard :

```bash
sudo ss -ulpn | grep 514                          # le manager écoute-t-il sur le port 514 ?
sudo tcpdump -i any -n udp port 514 -c 5          # des paquets de 192.168.10.1 arrivent-ils ?
sudo grep filterlog /var/ossec/logs/archives/archives.log | tail -n 3   # après <logall>yes</logall>
```

Résultat : `wazuh-remoted` écoute sur `192.168.10.10:514`, les paquets Syslog de pfSense arrivent, et `archives.log` contient des
lignes `filterlog` (le journal du pare-feu). Les logs sont donc **reçus**. Ils n'apparaissent pas dans le dashboard parce que le
dashboard n'affiche que les événements qui **déclenchent une règle d'alerte** ; ces lignes n'en déclenchent aucune.
L'option `logall` a ensuite été remise à `no`, car elle fait grossir le disque très vite.

![Option logall dans ossec.conf](../images/tp6-wazuh/18-ossec-conf-logall.png)

## 6. Un troisième agent par clonage

Clonage complet de la VM Ubuntu Server (VMware, *Full clone*), adresse changée en `192.168.10.30` (le sujet proposait `.21`).

![Netplan du clone](../images/tp6-wazuh/19-netplan-ubuntu-server-2.png)

![Console d'ubuntu-server-2](../images/tp6-wazuh/20-ubuntu-server-2.png)

Le clone a copié l'agent avec la **même identité** et la même clé que l'original : Wazuh les aurait confondus. Il faut le réenregistrer :

```bash
sudo systemctl stop wazuh-agent
sudo rm /var/ossec/etc/client.keys
sudo /var/ossec/bin/agent-auth -m 192.168.10.10 -A ubuntu-server-2     # doit afficher "Valid key received"
sudo systemctl start wazuh-agent
```

![Trois agents actifs](../images/tp6-wazuh/21-trois-agents-actifs.png)

## 7. Inventaire des agents

| ID | Agent | IP | Système | Version | Statut |
|---|---|---|---|---|---|
| 001 | `ubuntu-server` | 192.168.10.20 | Ubuntu 26.04.1 LTS | 4.14.10 | Active |
| 002 | `windows-workstation` | 192.168.10.50 | Windows 11 Home (10.0.26300) | 4.14.10 | Active |
| 003 | `ubuntu-server-2` | 192.168.10.30 | Ubuntu 26.04.1 LTS | 4.14.10 | Active |

Les trois agents sont dans le groupe `default` et rattachés au nœud `node01`.

## 8. Premier rapport de sécurité

Le rapport PDF est généré depuis **Threat Hunting** (le champ *Report source* de la page Reporting ne proposait aucun tableau de bord).
Fichier : [`reports/wazuh-threat-hunting-report.pdf`](../reports/wazuh-threat-hunting-report.pdf).

| Indicateur (période de 24 heures) | Valeur |
|---|---|
| Alertes au total | 3 780 |
| Alertes de niveau 12 ou plus | 0 |
| Échecs d'authentification | 18 |
| Réussites d'authentification | 62 |
| Agents concernés | `ubuntu-server`, `windows-workstation`, `wazuh-server`, `ubuntu-server-2` |

Observations :

- Le pic d'environ 2 800 alertes vers 15h vient surtout d'`ubuntu-server`. Les règles les plus fréquentes sont des événements système
  (détection d'anomalies *rootcheck*, installations de paquets `dpkg`) : l'analyse initiale d'un nouvel agent produit beaucoup de bruit.
- Les modules de conformité (SCA) évaluent les machines avec des référentiels CIS. Pour Ubuntu 26.04, Wazuh a utilisé un référentiel
  *Amazon Linux 2023*, faute de référentiel adapté. Windows 11 obtient un score de conformité inférieur à 30 %, ce qui est attendu pour un poste non durci.
- Les tests de ce TP (règles 5710 et 60122) figurent dans le résumé des alertes.

## Problèmes rencontrés

| Problème | Cause | Correction |
|---|---|---|
| La VM Wazuh recevait une adresse aléatoire (`.104`) | DHCP de pfSense | Réservation DHCP par adresse MAC vers `192.168.10.10` |
| L'agent se connectait à la mauvaise adresse | Dans la commande générée, `WAZUH_MANAGER` contenait l'adresse de la machine elle-même (`.20` sur Ubuntu, `.50` sur Windows) | Adresse du manager corrigée en `192.168.10.10` avant d'exécuter la commande |
| Le dashboard affichait « server is not ready yet » | Le service `wazuh-manager` était en échec (*timeout* au démarrage) alors que l'indexer et le dashboard tournaient | `sudo systemctl start wazuh-manager`, puis vérification de l'état des trois services |
| Aucun log pfSense visible dans le dashboard | Les lignes reçues ne déclenchent aucune règle d'alerte | Preuve de réception par `ss`, `tcpdump` et `archives.log` (`logall`) |
| Le sujet cherche `agent.name: pfSense` | pfSense n'a pas d'agent : ses logs sont rattachés au serveur Wazuh | Chercher `predecoder.program_name: filterlog` ou `location: 192.168.10.1` |
| Deux agents avec la même identité après le clonage | Le clone copie `client.keys` | Suppression de la clé, puis `agent-auth` avec un nouveau nom |
| Le rapport ne se générait pas | Champ *Report source* vide | Génération depuis le bouton de Threat Hunting |

## À retenir

- Un SIEM ne montre que ce qui **déclenche une règle** : « je ne vois rien » ne veut pas dire « rien n'est reçu ». Vérifier la chaîne
  maillon par maillon (port à l'écoute, paquets reçus, archives, règles).
- Une commande générée par un outil se **relit** avant de s'exécuter : l'adresse du serveur était fausse deux fois.
- Cloner une machine ne suffit pas à créer une nouvelle identité d'agent.
- Les journaux du pare-feu, d'Ubuntu et de Windows se retrouvent enfin au même endroit.
