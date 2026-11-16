# TP 2 : Règles de filtrage avancées et NAT

[← Retour au sommaire](../README.md)

## Objectif

Traduire la politique de sécurité de TechSenegal SA en règles pfSense :

- les employés naviguent sur le web (HTTP/HTTPS), envoient des e-mails et résolvent les noms (DNS) ;
- toute autre connexion sortante est interdite ;
- depuis Internet, seul le serveur web est joignable, sur le port 8080, redirigé vers le serveur interne ;
- seul le poste de l'administrateur accède au pare-feu.

## Comment lire une règle

Une règle pfSense se lit comme une phrase :

> **Autoriser ou bloquer** [protocole] **de** [source] **vers** [destination] [sur tel port].

- La **source** est la machine qui **commence** la conversation. La **destination** est celle qu'elle cherche à joindre.
- La réponse revient toute seule : pfSense est *stateful*, il retient chaque connexion autorisée dans sa table d'états.
- Le **port source** est choisi au hasard par la machine, on ne filtre donc presque jamais dessus. Le **port de destination**
  identifie le service (443 pour HTTPS, 53 pour le DNS).
- Le protocole décrit le mode de transport : TCP (connexion fiable), UDP (messages sans connexion, comme le DNS) ou ICMP (ping).
- Une règle s'applique sur l'interface **par laquelle le paquet entre** dans pfSense : les règles des postes du LAN
  vont dans l'onglet LAN.
- Les règles sont lues **de haut en bas** et la **première qui correspond gagne**. Les exceptions vont donc au-dessus
  des règles générales. Si rien ne correspond, le trafic est bloqué par une règle implicite.

## 1. Alias

Un alias donne un nom à une adresse ou à un port. Si l'adresse change, on ne modifie qu'un seul endroit.

| Alias | Type | Valeur | Rôle |
|---|---|---|---|
| `Serveur_web_Apache` | Host | 192.168.10.20 | Serveur web interne |
| `Poste_Admin` | Host | 192.168.10.50 | Poste administrateur |
| `Kali_test` | Host | 192.168.10.99 | Machine d'attaque simulée |
| `Port_web` | Port | 80, 443 | HTTP et HTTPS |
| `Port_Email` | Port | 25, 465, 587, 143, 993 | SMTP, SMTPS, IMAP, IMAPS |

![Alias créés](../images/tp2-filtrage-nat/01-alias-firewall.png)

Pour que la règle d'administration fonctionne, la VM Windows reçoit l'adresse fixe `192.168.10.50`.

![Adresse IP fixe du poste admin](../images/tp2-filtrage-nat/02-ip-statique-poste-admin.png)

## 2. Règles LAN

Ordre **final** (de haut en bas) :

| # | Action | Protocole | Source | Destination | Port | Rôle |
|---|---|---|---|---|---|---|
| 1 | Block | Any | `Kali_test` | Any | Any | Isoler la machine d'attaque (journalisée) |
| 2 | Pass | TCP | `Poste_Admin` | 192.168.10.1 | 443 | Accès de l'admin au WebGUI |
| 3 | Pass | TCP | LAN subnets | Any | `Port_web` | Navigation, avec la planification `Heures_Travail` |
| 4 | Pass | TCP | LAN subnets | Any | `Port_Email` | Messagerie |
| 5 | Pass | TCP/UDP | LAN subnets | Any | 53 | Résolution DNS |
| 6 | Block | Any | LAN subnets | Any | Any | *Deny-all* explicite, journalisé |

La règle « Default allow LAN to any » a été supprimée. L'**Anti-Lockout Rule** a d'abord été conservée, puis
désactivée plus tard pour une raison expliquée dans la section des problèmes.

Premier ordre, avec le Block Kali en bas de liste :

![Règles LAN, ordre initial](../images/tp2-filtrage-nat/03-regles-lan-ordre-initial.png)

Block Kali remonté sous l'Anti-Lockout Rule :

![Block Kali remonté](../images/tp2-filtrage-nat/04-block-kali-remonte.png)

Ordre final, Anti-Lockout désactivée :

![Règles LAN, ordre final](../images/tp2-filtrage-nat/05-regles-lan-ordre-final.png)

## 3. NAT

- **NAT sortant** : passé du mode automatique au mode **hybride**. pfSense garde ses règles automatiques et on peut en ajouter.
- **Port Forward** : le port 8080 du WAN est redirigé vers le port 80 de `Serveur_web_Apache`, avec l'option
  *Add associated filter rule* qui crée la règle WAN correspondante.
- Un second Port Forward envoie `WAN:9443` vers `192.168.10.20:8443` (mini-défi).
- Le **NAT 1:1** n'est pas testable dans ce lab : il demande plusieurs adresses IP publiques.

![Les deux Port Forward](../images/tp2-filtrage-nat/19-port-forward-8080-9443.png)

![Règles WAN créées automatiquement](../images/tp2-filtrage-nat/20-regles-wan-automatiques.png)

## 4. Préparation d'Ubuntu Server

Adresse fixe `192.168.10.20` avec Netplan (fichier complet dans [`configs/ubuntu`](../configs/ubuntu/netplan-ubuntu-server.yaml)),
puis Apache avec une page d'accueil personnalisée pour reconnaître le serveur pendant les tests.

![Configuration Netplan](../images/tp2-filtrage-nat/21-netplan-ubuntu-server.png)

![Adresse IP d'Ubuntu](../images/tp2-filtrage-nat/22-ubuntu-adresse-ip.png)

![Apache installé et actif](../images/tp2-filtrage-nat/24-apache-installe.png)

![Page d'accueil personnalisée](../images/tp2-filtrage-nat/25-page-apache-personnalisee.png)

## 5. Tests de validation

| Test | Machine | Destination | Résultat attendu | Résultat obtenu |
|---|---|---|---|---|
| Navigation HTTP | Windows | example.com:80 | Connexion réussie | OK |
| Navigation HTTPS | Windows | google.com:443 | Connexion réussie | OK |
| DNS | Windows | google.com | Résolution | OK |
| FTP bloqué | Windows | ftp.gnu.org:21 | Timeout | OK |
| Telnet bloqué | Windows | towel.blinkenlights.nl:23 | Timeout | OK |
| Kali vers Internet | Kali | 8.8.8.8 | Bloqué | OK |
| Kali vers pfSense | Kali | 192.168.10.1 | Bloqué | OK |
| Kali vers WebGUI | Kali | 192.168.10.1:443 | Bloqué | OK après correction (voir ci-dessous) |
| Admin vers WebGUI | Windows | 192.168.10.1:443 | Page de login | OK |
| Port Forward 8080 | Machine hôte | IP du WAN:8080 | Page Apache | OK |
| Port Forward 9443 | Machine hôte | IP du WAN:9443 | Listing du service 8443 | OK |

![Test HTTP](../images/tp2-filtrage-nat/06-test-http.png)

![Test HTTPS](../images/tp2-filtrage-nat/07-test-https.png)

![Test DNS](../images/tp2-filtrage-nat/08-test-dns.png)

![FTP et Telnet bloqués](../images/tp2-filtrage-nat/09-test-ftp-telnet-bloques.png)

![Kali, adresse IP](../images/tp2-filtrage-nat/10-kali-adresse-ip.png)

![Kali, ping bloqué](../images/tp2-filtrage-nat/11-kali-ping-bloque.png)

![Test du Port Forward 8080](../images/tp2-filtrage-nat/26-test-port-forward-8080.png)

![Service sur le port 8443](../images/tp2-filtrage-nat/27-serveur-http-8443.png)

![Test du Port Forward 9443](../images/tp2-filtrage-nat/28-test-port-forward-9443.png)

## 6. Lire les logs du firewall

Chaque règle avec l'option **Log** écrit dans **Status > System Logs > Firewall**. Une ligne contient l'heure, l'interface
d'entrée, l'action (Pass ou Block), l'IP et le port source, l'IP et le port de destination, et le protocole.
La description de la règle indique laquelle s'est appliquée.

![Logs des blocages de Kali](../images/tp2-filtrage-nat/12-logs-firewall-block-kali.png)

Le Port Forward apparaît dans les logs avec la destination **après traduction** : `192.168.10.20:80` pour le port 8080
et `192.168.10.20:8443` pour le port 9443.

![Logs du Port Forward](../images/tp2-filtrage-nat/29-logs-port-forward.png)

La table des états (**Diagnostics > States**) montre chaque connexion active avec son état TCP, par exemple `ESTABLISHED`.

![Table des états](../images/tp2-filtrage-nat/16-table-des-etats.png)

## 7. Mini-défi : navigation limitée aux heures de travail

Création de la planification `Heures_Travail` (lundi au vendredi, 08h00 à 18h00), associée à la règle de navigation.
En dehors de ce créneau, le trafic web tombe sur le *deny-all*.

![Planification créée](../images/tp2-filtrage-nat/17-schedule-heures-travail.png)

![Planification associée à la règle web](../images/tp2-filtrage-nat/18-schedule-sur-regle-web.png)

## Problèmes rencontrés

### Kali atteignait encore le WebGUI malgré la règle Block

Premiers tests : Kali semblait bloqué partout. En réalité, il n'avait pas encore reçu l'adresse `192.168.10.99` de l'alias
et il était arrêté par le *deny-all*, pas par la règle Block Kali. Une fois l'adresse fixée, deux causes sont apparues :

1. La règle **Pass web** (LAN vers Any) était **au-dessus** du Block Kali. Kali fait partie du LAN, donc cette règle correspondait la première.
2. L'**Anti-Lockout Rule**, placée toujours tout en haut, autorise n'importe quelle machine du LAN à joindre le WebGUI.

Corrections : Block Kali remonté en tête de liste, Anti-Lockout désactivée (**System > Advanced > Admin Access**),
puis table des états réinitialisée (**Diagnostics > States > Reset States**).

![Kali obtient la page de login](../images/tp2-filtrage-nat/13-kali-acces-webgui-anti-lockout.png)

![Désactivation de l'Anti-Lockout](../images/tp2-filtrage-nat/14-desactivation-anti-lockout.png)

![Kali, timeout vers le WebGUI](../images/tp2-filtrage-nat/15-kali-webgui-timeout.png)

> Après la désactivation, seul `Poste_Admin` accède au WebGUI. Si l'adresse de la VM Windows change, on se verrouille hors de
> pfSense ; le shell de la console (option 8) permet alors de rétablir l'accès.

### `apt update` échouait sur Ubuntu

La planification `Heures_Travail` coupait la navigation en dehors des heures de bureau (le test avait lieu un dimanche).
Les mises à jour passent par HTTP, elles tombaient donc sur le *deny-all*. La planification a été retirée le temps de
l'installation d'Apache, puis remise.

![apt update bloqué](../images/tp2-filtrage-nat/23-apt-update-bloque.png)

### Pas de trace du Port Forward dans les logs

La règle WAN créée automatiquement ne journalise pas par défaut. La case **Log** a été cochée sur les deux règles WAN.

### Adresse de Kali perdue au redémarrage

Le profil réseau modifié n'était pas celui qui se reconnectait au démarrage, Kali repassait en DHCP. Correction : modifier le
profil existant (mode manuel, connexion automatique) et supprimer les profils en double.

### Heure des logs décalée d'une heure

pfSense était réglé sur `Africa/Dakar` alors que la VM suivait l'heure de Paris. Le fuseau a été changé en `Europe/Paris`.
Les planifications horaires suivent l'heure de pfSense.

## À retenir

- L'**ordre des règles** compte autant que leur contenu : les exceptions au-dessus des règles générales.
- Une règle de blocage n'a de valeur que si elle est **testée avec la bonne adresse source**.
- L'Anti-Lockout Rule protège de l'erreur humaine, mais elle contourne les règles écrites pour le WebGUI.
- Un test négatif réussi (« c'est bloqué ») peut avoir une cause qui n'est pas celle qu'on croit : toujours vérifier dans les logs
  **quelle règle** s'est appliquée.
