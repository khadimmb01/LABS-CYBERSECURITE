# DNS statique avec BIND9

TP réalisé en avril 2026 : configuration d'un serveur DNS statique avec BIND9 sur Ubuntu/Debian, pour résoudre les noms internes du domaine `monlab.sn` dans un réseau de laboratoire isolé.

**Auteur :** Khadim Mbacké Thiao
**Outils :** Netplan, BIND9, `named-checkconf`, `named-checkzone`, `nslookup`

| Paramètre | Valeur |
|---|---|
| Nom d'hôte | serveurdns |
| IP du serveur DNS | 192.168.56.30 |
| Passerelle | 192.168.56.1 |
| Réseau | 192.168.56.0/24 |
| Domaine interne | monlab.sn |
| Zone directe | `/etc/bind/direct` |
| Zone inverse | `/etc/bind/reverse` |

## Objectif

Mettre en place un DNS qui résout les noms vers les adresses IP (zone directe) et les adresses IP vers les noms (zone inverse). Le lab est isolé : pas de forwarding vers l'extérieur et pas de pare-feu.

## Étapes

1. IP statique avec Netplan
2. Déclaration des zones dans `named.conf.local`
3. Création de la zone directe (enregistrement A)
4. Création de la zone inverse (enregistrement PTR)
5. Vérification de la syntaxe, redémarrage et statut de BIND9
6. Tests avec `nslookup`

## 1. IP statique (Netplan)

Un serveur DNS doit garder la même adresse, sinon les clients ne le retrouvent plus. Le nom du fichier Netplan varie selon l'installation (`ls /etc/netplan/`).

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.56.30/24
      routes:
        - to: default
          via: 192.168.56.1
      nameservers:
        addresses:
          - 192.168.56.30
```

Le serveur utilise son propre BIND9 comme DNS. On applique puis on vérifie :

```bash
sudo netplan apply
ip addr show enp0s3
```

## 2. Déclaration des zones

Le fichier `named.conf.local` indique à BIND9 quelles zones il gère et où se trouvent leurs fichiers.

```bash
sudo nano /etc/bind/named.conf.local
```

```
// Zone directe : nom -> IP
zone "monlab.sn" {
    type master;
    file "/etc/bind/direct";
};

// Zone inverse : IP -> nom
zone "56.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/reverse";
};
```

Le nom de la zone inverse s'écrit à l'envers : le réseau 192.168.56 devient `56.168.192.in-addr.arpa`. Les noms des fichiers de zone sont libres.

## 3. Zone directe

On part du modèle fourni par BIND9 :

```bash
sudo cp /etc/bind/db.local /etc/bind/direct
sudo nano /etc/bind/direct
```

```
$TTL    604800
@       IN SOA  monlab.sn. root.monlab.sn. (
                2024042601 ; Serial
                604800     ; Refresh
                86400      ; Retry
                2419200    ; Expire
                604800 )   ; Negative Cache TTL
;
@           IN NS   serveurdns.monlab.sn.
;
; Enregistrement A : serveur DNS
serveurdns  IN A    192.168.56.30
```

- **SOA** : décrit la zone et son responsable. Le champ Serial doit être incrémenté à chaque modification.
- **NS** : serveur de noms de la zone.
- **A** : associe le nom `serveurdns` à son adresse IP.

Seul le serveur DNS est déclaré pour l'instant. D'autres machines seront ajoutées plus tard.

## 4. Zone inverse

On copie la zone directe puis on la transforme :

```bash
sudo cp /etc/bind/direct /etc/bind/reverse
sudo nano /etc/bind/reverse
```

```
$TTL    604800
@       IN SOA  monlab.sn. root.monlab.sn. (
                2024042601 ; Serial
                604800     ; Refresh
                86400      ; Retry
                2419200    ; Expire
                604800 )   ; Negative Cache TTL
;
@       IN NS   serveurdns.monlab.sn.
;
; Enregistrement PTR : dernier octet de l'IP -> nom complet
30      IN PTR  serveurdns.monlab.sn.
```

Dans un enregistrement PTR, on n'écrit que le dernier octet de l'IP (`30`), car le réseau 192.168.56 est déjà dans le nom de la zone. L'enregistrement A copié de la zone directe est supprimé, il n'a pas sa place ici. Le point final du nom complet est obligatoire.

## 5. Vérification et redémarrage

On contrôle la syntaxe avant de redémarrer :

```bash
sudo named-checkconf
sudo named-checkzone monlab.sn /etc/bind/direct
sudo named-checkzone 56.168.192.in-addr.arpa /etc/bind/reverse
```

`named-checkconf` ne doit rien afficher, et chaque zone doit se terminer par `OK`. Puis :

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

Le service doit être `active (running)`. S'il est `failed`, on relance `named-checkconf` pour trouver l'erreur de syntaxe.

## 6. Tests avec nslookup

**Résolution directe (nom vers IP) :**

```bash
nslookup serveurdns.monlab.sn 192.168.56.30
```

Résultat attendu : `Address: 192.168.56.30`

**Résolution inverse (IP vers nom) :**

```bash
nslookup 192.168.56.30 192.168.56.30
```

Résultat attendu : `30.56.168.192.in-addr.arpa name = serveurdns.monlab.sn.`

| Commande | Résultat attendu |
|---|---|
| `nslookup serveurdns.monlab.sn 192.168.56.30` | `Address: 192.168.56.30` |
| `nslookup 192.168.56.30 192.168.56.30` | `name = serveurdns.monlab.sn.` |
| `systemctl status bind9` | `active (running)` |
| `named-checkconf` | Aucune sortie, pas d'erreur |

## Fichiers modifiés

| Fichier | Action |
|---|---|
| `/etc/netplan/50-cloud-init.yaml` | IP statique, passerelle, DNS 192.168.56.30 |
| `/etc/bind/named.conf.local` | Déclaration des zones directe et inverse |
| `/etc/bind/direct` | Zone directe : SOA, NS, A |
| `/etc/bind/reverse` | Zone inverse : SOA, NS, PTR |

## Points d'attention

- À chaque modification d'un fichier de zone, il faut incrémenter le champ **Serial** puis relancer `sudo systemctl restart bind9`.
- Le serveur de noms déclaré dans l'enregistrement NS doit avoir un enregistrement A dans la zone, sinon `named-checkzone` signale une erreur.
- Le nom dans l'enregistrement PTR doit correspondre au nom de l'enregistrement A, sinon la résolution inverse renvoie un nom différent de celui attendu.

## Compétences mobilisées

- Configuration d'une IP statique avec Netplan
- Installation et configuration d'un serveur DNS BIND9
- Création de zones directe et inverse, enregistrements A, NS, SOA et PTR
- Vérification de la syntaxe avec `named-checkconf` et `named-checkzone`
- Tests de résolution avec `nslookup`

## Suite

Dans le prochain TP, les enregistrements ne seront plus saisis à la main : un serveur DHCP (`isc-dhcp-server`) mettra à jour automatiquement les zones de BIND9 grâce au DNS dynamique (DDNS).

| | DNS statique (ce TP) | DNS dynamique (prochain TP) |
|---|---|---|
| Enregistrements | Saisis manuellement | Créés automatiquement par le DHCP |
| Mise à jour | Manuelle + redémarrage | Automatique (DDNS) |
| Cas d'usage | Serveurs à IP fixe | Postes clients à IP variable |
