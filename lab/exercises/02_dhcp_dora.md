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

| Étape | Émetteur (IP src) | Destinataire (IP dst) | MAC src / dst | Options DHCP notables |
|-------|-------------------|----------------------|----------------|----------------------|
| 1. Discover | 0.0.0.0 | 255.255.255.255 | e2:08:b8:3b:49:30 / ff:ff:ff:ff:ff:ff | option 53=Discover (1), option 50=Requested-IP=172.20.1.175, option 12=Hostname="client", option 55=Parameter-Request (subnet, gateway, DNS, etc.) |
| 2. Offer | 172.20.1.2 | 172.20.1.175 | 3a:d3:cc:c7:25:d4 / e2:08:b8:3b:49:30 | option 53=Offer (2), option 54=Server-ID=172.20.1.2, option 51=Lease=43200s (12h), option 58=T1=21600s (6h), option 59=T2=37800s (10h30), option 1=Subnet-Mask=255.255.255.0, option 3=Gateway=172.20.1.254, option 6=DNS=1.1.1.1,8.8.8.8 |
| 3. Request | 0.0.0.0 | 255.255.255.255 | e2:08:b8:3b:49:30 / ff:ff:ff:ff:ff:ff | option 53=Request (3), option 54=Server-ID=172.20.1.2 (serveur choisi), option 50=Requested-IP=172.20.1.175 |
| 4. ACK | 172.20.1.2 | 172.20.1.175 | 3a:d3:cc:c7:25:d4 / e2:08:b8:3b:49:30 | option 53=ACK (5), option 54=Server-ID=172.20.1.2, option 51=Lease=43200s, options identiques à l'Offer (subnet, gateway, DNS, T1, T2) |

### 2. Configuration finale du client

```bash
docker exec lab_client ip -4 addr show eth0
docker exec lab_client ip route
docker exec lab_client cat /etc/resolv.conf   # peut être vide si non géré par dhclient
```

Notez **l'IP attribuée, le masque, la passerelle, les DNS, la durée de bail**.

> 💬 **Votre réponse :**
>
> **Configuration obtenue par DHCP** :
- IP : **172.20.1.175**
- Masque : **/24 (255.255.255.0)**
- Passerelle : **172.20.1.254**
- DNS : **1.1.1.1** et **8.8.8.8**
- Domain : **lab.local**
- Bail : **43200 secondes (12h)**, T1 à 6h, T2 à 10h30

### 3. Questions de réflexion

**Question 1.** Pourquoi le client utilise-t-il **`0.0.0.0` comme IP
source** pour le Discover, alors que c'est une adresse non routable&nbsp;?
Que se passerait-il avec n'importe quelle autre adresse&nbsp;?

> 💬 **Votre réponse :**
>
> Le client utilise 0.0.0.0 comme IP source parce qu'**il n'a pas encore d'IP attribuée**
au moment où il envoie son Discover. C'est précisément la raison pour laquelle il
demande à un serveur DHCP : il n'a rien.

0.0.0.0 est l'adresse "non spécifiée" définie par l'IETF (RFC 1122) pour signaler
ce cas : "je suis un hôte sans identité IP".

Si le client utilisait n'importe quelle autre adresse :
- Une IP **hors du subnet** serait droppée par les équipements L3 du réseau ;
- Une IP **dans le subnet mais non assignée** pourrait entrer en conflit avec un
  autre hôte (envoi de paquets bizarres, blacklisting par switches stricts) ;
- Une IP **valide mais utilisée par un autre** créerait un conflit ARP.

0.0.0.0 est donc un signal univoque : *"je suis un nouveau venu, configure-moi"*.

**Question 2.** Pourquoi le **Request** est-il **rediffusé en broadcast**
alors que le client connaît déjà l'IP du serveur après l'Offer&nbsp;?

> 💬 **Votre réponse :**
>
> Le Request est rediffusé en broadcast pour deux raisons complémentaires :

**1. Informer les autres serveurs DHCP** : sur un réseau avec plusieurs serveurs,
chacun envoie son propre Offer. Le client n'en accepte qu'un (via l'option 54
Server-ID dans son Request). Le broadcast permet aux serveurs **non choisis** de
voir le Server-ID retenu et de **libérer** l'IP qu'ils avaient pré-réservée.
Sans cela, ils garderaient cette adresse "en attente" inutilement.

**2. Le client n'a toujours pas d'IP officielle** : au moment du Request, l'IP est
proposée mais pas encore confirmée. Le client ne peut pas envoyer en unicast
classique avec une IP source valide. Le broadcast contourne ce problème :
n'importe quel équipement L2 peut le transmettre sans connaître l'adresse du
client.

**Question 3.** À quoi sert le **transaction ID (xid)** présent dans les
4 paquets&nbsp;? Que se passerait-il s'il était omis dans un réseau avec
plusieurs serveurs DHCP&nbsp;?

> 💬 **Votre réponse :**
>
> Le **transaction ID (xid)** est un identifiant unique 32 bits choisi aléatoirement
par le client au début du DORA. Il apparaît identique dans les 4 paquets pour les
**corréler entre eux** : Discover, Offer, Request et ACK partagent le même xid
(dans ma capture : 0xa27534c).

**Rôles** :
- Permet au client de **rejeter les réponses qui ne correspondent pas** à sa
  transaction (par exemple, l'Offer destinée à un autre client passant en
  broadcast).
- Permet au serveur de **lier** une Request à l'Offer précédente.
- Sépare les transactions concurrentes (un même client peut relancer un DORA
  si timeout — chaque tentative a son propre xid).

**Sans xid (ou avec plusieurs serveurs DHCP)** : un client pourrait accepter
l'Offer d'un autre serveur destinée à un autre client (vol de bail), ou
confondre deux Offers reçues pour deux Requests différents. Le réseau
deviendrait imprévisible — c'est l'équivalent d'envoyer des courriers sans
numéro de référence.

**Question 4.** Que renvoie le serveur si vous demandez explicitement une
adresse hors du pool (essayez `dhclient -v -s 172.20.1.99 eth0`)&nbsp;?
Justifiez.

> 💬 **Votre réponse :**
>
> **Observation** : la commande `dhclient -v -s 172.20.1.99 eth0` génère plusieurs
DHCPDISCOVER en unicast vers 172.20.1.99 avec un intervalle croissant (6s, 15s),
puis échoue. **Aucun NAK n'est reçu**, **aucune réponse non plus**.

**Justification** : l'option `-s 172.20.1.99` force le client à envoyer son
Discover **uniquement** à cette adresse. Or 172.20.1.99 n'est ni dans le pool
(qui va de 172.20.1.100 à 172.20.1.200 d'après les logs dnsmasq) ni l'IP du
serveur DHCP réel (172.20.1.2). **Personne n'écoute** sur cette adresse →
aucune réponse possible.

Note importante : un NAK serait envoyé si un **vrai serveur DHCP** recevait une
Request pour une adresse qu'il ne peut pas allouer (par exemple, une IP hors
pool ou déjà attribuée à un autre client). Ici, on n'obtient même pas de NAK
parce qu'aucun serveur ne traite la requête.

**Question 5.** La directive `dhcp-authoritative` est active sur notre
serveur. Quel est son effet **comportemental** sur les NAK&nbsp;?

> 💬 **Votre réponse :**
>
> La directive `dhcp-authoritative` (dans `/etc/dnsmasq.conf`) déclare le serveur
comme **autorité officielle** pour le sous-réseau qu'il gère. Son effet
comportemental principal concerne les **NAK** :

**Sans dhcp-authoritative** (mode par défaut, prudent) :
- Si un client demande à renouveler une IP que le serveur **ne reconnaît pas**
  (par exemple parce que le serveur a redémarré et perdu ses baux), le serveur
  **ignore silencieusement** la requête.
- Le client retente, finit par expirer, puis refait un DORA complet — mais
  cela peut prendre du temps.

**Avec dhcp-authoritative** (notre cas) :
- Le serveur **envoie immédiatement un NAK** quand il ne reconnaît pas l'IP
  demandée ou si elle est hors pool.
- Le client abandonne son IP courante et refait un DORA tout de suite.

**Effet net** : convergence plus rapide vers un état cohérent, au prix
d'éventuels NAK envoyés "à tort" si plusieurs serveurs coexistent (ce qui est
proscrit dans un réseau bien conçu — un seul serveur DHCP autoritaire par
subnet).

### 4. Renouvellement de bail (T1/T2)

Le bail est de 12&nbsp;h, T1 (renouvellement) à 6&nbsp;h, T2 (rebind) à 10&nbsp;h30.
En **2-3 phrases**, décrivez la différence entre un renouvellement T1 et
un rebind T2 (destinataire du paquet, comportement attendu).

> 💬 **Votre réponse :**
>
> **T1 (renouvellement, à 50% du bail = 6h)** : le client envoie un **DHCPREQUEST
en unicast** au serveur qui a délivré le bail (172.20.1.2 dans notre cas). Il
demande simplement à prolonger son bail. Si le serveur répond ACK, le bail est
prolongé de 12h supplémentaires. Le réseau reste stable.

**T2 (rebind, à 87.5% du bail = 10h30)** : si T1 a échoué (par exemple, le
serveur d'origine est down), le client passe en mode **rebind**. Il envoie un
DHCPREQUEST **en broadcast** à *n'importe quel* serveur DHCP du réseau. Si un
autre serveur peut prendre le relais (configuration multi-serveurs ou serveur
revenu en ligne), il répondra ACK ou NAK.

**Différence clé** : T1 = unicast ciblé au serveur connu ; T2 = broadcast
ouvert. T1 est la voie normale, T2 est le filet de sécurité.
