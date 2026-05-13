# Exercice 2 — DHCP : observer DORA paquet par paquet

**Durée estimée :** 45 min
**Objectif :** capturer un échange DHCP complet (Discover / Offer / Request /
ACK), identifier les options portées par chaque message, et comprendre
pourquoi DHCP utilise un broadcast L2 alors qu'IP n'est pas encore
configuré.

## Manipulation

Côté `dhcp-server`, démarrez une capture filtrée sur les ports DHCP (67/68)&nbsp;:

```bash
docker exec -it lab_dhcp_server tcpdump -i eth0 -nn -e -v port 67 or port 68
```

> Note&nbsp;: `-e` affiche les adresses MAC, indispensables pour comprendre
> le broadcast L2.

Côté `client`, déclenchez une nouvelle demande de bail&nbsp;:

```bash
docker exec lab_client bash -c "dhclient -r eth0 2>/dev/null; dhclient -v eth0"
```

Observez les **4 paquets** DORA dans la capture, puis arrêtez tcpdump (Ctrl+c).

Affichez aussi les journaux applicatifs du serveur&nbsp;:

```bash
docker logs --tail 40 lab_dhcp_server
```

## À rendre — répondez directement dans ce fichier

### 1. Tableau DORA

Complétez en vous appuyant sur **votre propre capture**&nbsp;:

| Étape       | Émetteur (IP src) | Destinataire (IP dst) | MAC src / dst | Options DHCP notables |
| ----------- | ----------------- | --------------------- | ------------- | --------------------- |
| 1. Discover | `0.0.0.0`         | `255.255.255.255`     | `66:c6:cb:25:f7:2a` → `ff:ff:ff:ff:ff:ff` | option 53 = 1 (Discover), option 55 = netmask, router, dns-server, domain-name, hostname |
| 2. Offer    | `172.20.1.2`      | `255.255.255.255`     | serveur → `ff:ff:ff:ff:ff:ff` | option 53 = 2 (Offer), option 54 = 172.20.1.2, option 51 = 12h, option 3 = 172.20.1.254, option 6 = 1.1.1.1 / 8.8.8.8 |
| 3. Request  | `0.0.0.0`         | `255.255.255.255`     | `66:c6:cb:25:f7:2a` → `ff:ff:ff:ff:ff:ff` | option 53 = 3 (Request), option 54 = 172.20.1.2, option 50 = 172.20.1.172 |
| 4. ACK      | `172.20.1.2`      | `255.255.255.255`     | serveur → `ff:ff:ff:ff:ff:ff` | option 53 = 5 (ACK), option 51 = 12h, option 58 = T1 6h, option 59 = T2 10h30m, option 3 = 172.20.1.254, option 6 = 1.1.1.1 / 8.8.8.8 |
### 2. Configuration finale du client

```bash
docker exec lab_client ip -4 addr show eth0
docker exec lab_client ip route
docker exec lab_client cat /etc/resolv.conf   # peut être vide si non géré par dhclient
```

Notez **l'IP attribuée, le masque, la passerelle, les DNS, la durée de bail**.

> 💬 **Votre réponse :**
>
> - **IP attribuée** : `172.20.1.172/24` (dynamique DHCP)
- **Masque** : `255.255.255.0` (`/24`)
- **Passerelle (GW)** : `172.20.1.1`
- **DNS** : `1.1.1.1` et `8.8.8.8`
- **Domaine** : `lab.local`
- **Durée de bail** : 12h (T1 = 6h, T2 = 10h30)

### 3. Questions de réflexion

**Question 1.** Pourquoi le client utilise-t-il **`0.0.0.0` comme IP
source** pour le Discover, alors que c'est une adresse non routable&nbsp;?
Que se passerait-il avec n'importe quelle autre adresse&nbsp;?

> 💬 **Votre réponse :**
>
> Le client n'a pas encore d'IP au moment du Discover — il ne peut donc pas en mettre une valide. `0.0.0.0` est la convention DHCP pour signifier "émetteur sans adresse". Si le client utilisait une IP arbitraire, les routeurs pourraient rejeter le paquet car l'adresse source ne correspondrait à aucune interface connue sur le réseau.

**Question 2.** Pourquoi le **Request** est-il **rediffusé en broadcast**
alors que le client connaît déjà l'IP du serveur après l'Offer&nbsp;?

> 💬 **Votre réponse :**
>
> Parce qu'il peut y avoir plusieurs serveurs DHCP sur le réseau qui ont tous envoyé une Offer. Le broadcast permet d'informer simultanément tous les serveurs que le client a choisi l'un d'eux — les autres retirent alors leur offre et libèrent l'adresse réservée.

**Question 3.** À quoi sert le **transaction ID (xid)** présent dans les
4 paquets&nbsp;? Que se passerait-il s'il était omis dans un réseau avec
plusieurs serveurs DHCP&nbsp;?

> 💬 **Votre réponse :**
>
> Le xid est un identifiant aléatoire généré par le client pour associer chaque Offer/ACK à son propre Discover/Request. Sans lui, dans un réseau avec plusieurs clients et serveurs DHCP, un client ne pourrait pas distinguer quelle réponse lui est destinée — il risquerait d'accepter la configuration d'un autre client.

**Question 4.** Que renvoie le serveur si vous demandez explicitement une
adresse hors du pool (essayez `dhclient -v -s 172.20.1.99 eth0`)&nbsp;?
Justifiez.

> 💬 **Votre réponse :**
>
> Le serveur renvoie un **DHCPNAK** (refus). L'adresse `172.20.1.99` est hors du pool configuré (`172.20.1.100 -- 172.20.1.200`), le serveur ne peut donc pas l'attribuer et refuse explicitement la demande pour éviter des conflits d'adresses.

**Question 5.** La directive `dhcp-authoritative` est active sur notre
serveur. Quel est son effet **comportemental** sur les NAK&nbsp;?

> 💬 **Votre réponse :**
>
> Avec `dhcp-authoritative`, le serveur envoie immédiatement un DHCPNAK si un client demande une adresse qu'il ne reconnaît pas (bail expiré, mauvais réseau, hors pool). Sans cette directive, le serveur ignorerait silencieusement la demande et attendrait, ce qui allongerait inutilement le délai de reconfiguration du client.

### 4. Renouvellement de bail (T1/T2)

Le bail est de 12&nbsp;h, T1 (renouvellement) à 6&nbsp;h, T2 (rebind) à 10&nbsp;h30.
En **2-3 phrases**, décrivez la différence entre un renouvellement T1 et
un rebind T2 (destinataire du paquet, comportement attendu).

> 💬 **Votre réponse :**
>
> À T1 (6h), le client envoie un DHCPREQUEST en **unicast directement au serveur** qui lui a attribué le bail pour tenter de le renouveler — c'est une négociation silencieuse entre les deux. Si le serveur ne répond pas, le client attend T2 (10h30) et envoie cette fois un DHCPREQUEST en **broadcast**, car il cherche n'importe quel serveur DHCP disponible pour rebind son adresse. Si aucun serveur ne répond avant l'expiration du bail (12h), le client perd son adresse et recommence un cycle DORA complet.
