# Exercice 1 — OSI à travers une vraie capture de paquets

**Durée estimée :** 45 min
**Objectif :** relier chaque couche du modèle OSI à un champ concret observable
dans une capture réseau effectuée sur le lab.

## Préparation

Configurez le client (s'il ne l'a pas déjà été par DHCP — voir exercice 2)&nbsp;:

```bash
docker exec lab_client bash -c "ip route del default 2>/dev/null; ip route add default via 172.20.1.254"
```

Vérifiez l'accès au site « public »&nbsp;:

```bash
docker exec lab_client curl -s http://172.20.0.10/whoami
```

## Manipulation

**Étape 1 — supprimez une éventuelle capture précédente** (sinon échec en
*Permission denied* car tcpdump abandonne ses privilèges vers l'utilisateur
`tcpdump` après ouverture du fichier) :

```bash
docker exec lab_client rm -f /tmp/http.pcap
```

**Étape 2 — lancez la capture côté client.** Les flags importants&nbsp;:
`-U` (écriture non bufferisée, indispensable si la capture est interrompue),
`-c 30` (s'arrête automatiquement après 30 paquets, ≈ 2 à 3 requêtes HTTP) :

```bash
docker exec lab_client tcpdump -i eth0 -U -w /tmp/http.pcap -nn -c 30 host 172.20.0.10
```

> ⚠️ Cette commande **bloque** le terminal tant que les 30 paquets ne sont
> pas capturés. Ouvrez un **second terminal** pour l'étape 3.

**Étape 3 — dans un second terminal, déclenchez du trafic** *pendant* que
tcpdump tourne&nbsp;:

```bash
docker exec lab_client curl -v http://172.20.0.10/
docker exec lab_client curl -v http://172.20.0.10/whoami
docker exec lab_client curl -s http://172.20.0.10/        # complément pour atteindre 30 paquets
```

tcpdump s'arrête seul dès le 30e paquet et affiche un résumé du type
`30 packets captured / 0 packets dropped by kernel`.

**Étape 4 — vérifiez que la capture est exploitable** :

```bash
docker exec lab_client capinfos /tmp/http.pcap
```

`Number of packets` doit être > 0. Si la capture est vide, voir la section
*Pièges fréquents* en bas de cet énoncé.

**Étape 5 — analysez la capture**. Trois vues utiles&nbsp;:

```bash
# Vue compacte : une ligne par paquet (utile pour repérer les n° de frames)
docker exec lab_client tshark -r /tmp/http.pcap

# Vue détaillée d'un paquet précis (ex. la requête GET = frame n°4)
docker exec lab_client tshark -r /tmp/http.pcap -V -Y 'frame.number == 4'

# Vue détaillée complète (pager : utilisez 'q' pour quitter)
docker exec -it lab_client sh -c "tshark -r /tmp/http.pcap -V | less"
```

> `-V` produit la décomposition complète couche par couche
> (Frame → Ethernet → IP → TCP → HTTP). Le filtre `-Y` cible un paquet
> par son numéro pour éviter d'avoir à scroller dans toute la trace.

## Visualisation assistée : `osi_inspect.py`

La sortie brute de `tshark -V` est dense (plusieurs centaines de lignes par
paquet). Un script Python est fourni dans ce dossier pour vous présenter,
pour chaque trame, un **tableau structuré par couche OSI** avec en plus
une **colonne d'explication pédagogique** pour chaque champ.

### Lister les trames

Depuis la racine du dépôt (l'hôte, pas l'intérieur du conteneur) :

```bash
./lab/exercises/osi_inspect.py
```

Vous obtenez la même vue compacte que `tshark` mais sans avoir à taper la
commande complète. Repérez la trame qui vous intéresse — typiquement la
trame portant `GET /` (ligne marquée `HTTP 141 GET / HTTP/1.1`).

### Détailler une trame

```bash
./lab/exercises/osi_inspect.py 4         # trame n°4 — la requête HTTP
./lab/exercises/osi_inspect.py 1         # trame n°1 — le SYN du handshake
./lab/exercises/osi_inspect.py 8         # trame n°8 — la réponse 200 OK
```

Le script affiche, pour chaque couche OSI **présente** dans la trame, les
champs clés avec **3 informations** :

| Colonne       | Contenu                                       |
| ------------- | --------------------------------------------- |
| `Champ`       | Nom du champ tel qu'extrait par tshark        |
| `Valeur`      | Valeur réelle observée dans **votre** capture |
| `Explication` | À quoi sert ce champ, comment l'interpréter   |

C'est exactement la matière dont vous avez besoin pour remplir le tableau
*« Couche / Élément observé / Valeur exemple »* demandé dans la section
suivante.

### Réutilisation pour les exercices suivants

Le script est générique. Pour disséquer une capture DHCP (exercice 2) ou
NAT (exercice 3), pointez-le vers le bon conteneur et le bon fichier :

```bash
./lab/exercises/osi_inspect.py 1 --pcap /tmp/dhcp.pcap --container lab_dhcp_server
./lab/exercises/osi_inspect.py 3 --pcap /tmp/nat.pcap  --container lab_nat_router
```

### Travail demandé avec ce script

1. Lancez `./lab/exercises/osi_inspect.py` pour obtenir la liste des trames.
2. Identifiez **une trame contenant du HTTP** (typiquement la requête `GET /`)
   et **une trame de contrôle TCP** (SYN, ACK seul, ou FIN).
3. Lancez le script avec le n° de chaque trame et **copiez la sortie**
   dans le README de votre fork (bloc de code).
4. Pour chacune des deux trames, **comptez et nommez** les couches OSI
   visibles (utilisez la ligne `Pile présente : …` en en-tête). Expliquez
   en 1 phrase pourquoi la couche 7 est absente sur la trame de contrôle TCP.

> 💬 **Votre réponse (sorties du script + analyse) :**
>
> _Remplacez ce texte par votre réponse._

## À rendre — répondez directement dans ce fichier

Pour **chaque couche OSI**, donnez **un exemple concret extrait de votre
capture** (champ, valeur observée). Justifiez en 1-2 phrases.

| Couche OSI         | Élément observé dans la capture | Valeur exemple |
| ------------------ | ------------------------------- | -------------- |
| 7 — Application    | _ex. méthode HTTP_              | `GET /whoami HTTP/1.1` |C'est la couche que l'application "comprend". La méthode GET indique l'action voulue, l'URI désigne la ressource, le Host permet le virtual hosting
| 6 — Présentation   | _ex. encodage / Content-Type_   | …   text/html (encodage du payload)    L6 s'occupe du format des données : comment interpréter les bytes       |
| 5 — Session        | _ex. Keep-Alive, cookies_       | …   keep-alive (réutilise la connexion)     L5 gère le maintien d'un dialogue continu entre client et serveur.      |
| 4 — Transport      | _ex. port TCP, flags_           | …  40844 → 80, flags AP (ACK+PUSH)  L4 multiplexe plusieurs services sur une même IP via les ports (80=HTTP, 443=HTTPS, 22=SSH).          |
| 3 — Réseau         | _ex. IP source / destination_   | …  172.20.1.50 → 172.20.0.10            L3 est end-to-end : ces IPs traversent tous les routeurs jusqu'à la destination. Le TTL=64 est décrémenté à chaque saut   |
| 2 — Liaison        | _ex. adresses MAC_              | …   e2:08:b8:3b:49:30 → 92:fa:d9:67:0b:21 L2 est point-à-point sur le segment LAN. La MAC destination est celle du prochain saut  |           |
| 1 — Physique       | _non visible — pourquoi&nbsp;?_ | … tcpdump capture à partir de L2    L1 = signaux électriques/optiques/radio           |

## Questions de réflexion

**Question 1.** Pourquoi l'**adresse MAC source** observée n'est-elle
**pas** celle du serveur `internet` mais celle du `nat-router`&nbsp;? Que
vous apprend cette observation sur la portée de chaque couche&nbsp;?

> 💬 **Votre réponse :**
>
> La MAC destination observée (92:fa:d9:67:0b:21) est celle du **nat-router**, et non
celle du serveur `lab_internet`. C'est normal : les adresses MAC ont une **portée
strictement locale** au segment L2 (un seul lien).

Le client (172.20.1.50) et le serveur (172.20.0.10) sont sur **deux réseaux IP
différents** (172.20.1.0/24 et 172.20.0.0/24). Le client ne peut pas joindre le
serveur directement en L2 — il doit passer par sa passerelle (nat-router).


**Question 2.** Vous capturez sur `eth0` du client (côté LAN). Dans votre
trace, l'**IP source** sortante est `172.20.1.50`. Pourtant, `curl /whoami`
rapporte que le serveur perçoit `172.20.0.254`. Expliquez cette différence
et indiquez **où** il faudrait capturer pour voir l'IP réécrite.
*Astuce&nbsp;:* `docker exec lab_nat_router tcpdump -i any -nn -c 10 host 172.20.0.10`.

> 💬 **Votre réponse :**
>
> La capture se fait sur `eth0` du **client**, donc **avant** que le paquet ne
traverse le nat-router. À ce moment, l'IP source est encore celle d'origine :
172.20.1.50.

Le nat-router applique ensuite une règle iptables **MASQUERADE** : il **réécrit**
l'IP source en 172.20.0.254 (son IP côté réseau `lab_internet`) et mémorise la
traduction dans sa table conntrack. C'est pour cela que le serveur perçoit
172.20.0.254.

Pour voir l'IP **réécrite**, il faut capturer **sur le nat-router** (interface
côté réseau internet) ou **directement sur le serveur** :

    docker exec lab_nat_router tcpdump -i any -nn -c 10 host 172.20.0.10

Cela illustre que le NAT modifie la couche 3 (IP source) mais sans toucher au
contenu applicatif (couche 7).

**Question 3.** Lancez `curl -v https://...` vers un site HTTPS public
(depuis l'hôte, pas le lab). Quelle couche change visiblement par
rapport au HTTP du lab&nbsp;? Quelles couches **disparaissent** de votre
visibilité&nbsp;?

> 💬 **Votre réponse :**
>
> En lançant `curl -v https://www.google.com`, on observe les changements suivants
dans la capture par rapport au HTTP du lab :

**Ce qui change visiblement :**
- Un **TLS handshake** s'intercale entre la connexion TCP et la requête HTTP
  (ClientHello, ServerHello, échange de certificats, dérivation de clés).
- La **couche 6 (Présentation)** devient active et chiffre tout ce qui est
  au-dessus (HTTP).

**Ce qui disparaît de la visibilité :**
- **L7 (HTTP)** : méthode, URI, headers, body — tout est chiffré.
- **L6 applicative** (Content-Type, encodage) — chiffré.
- **L5** (Connection, cookies) — chiffré.

**Ce qui reste visible :**
- L2 (MAC), L3 (IP source/dest), L4 (ports TCP, flags).

C'est ce qui permet aux firewalls et aux routeurs de fonctionner même en HTTPS :
ils n'ont besoin que des couches basses pour acheminer.

**Question 4.** La couche 5 (Session) est très peu visible dans une
capture HTTP/1.1. Donnez **deux mécanismes applicatifs** qui jouent le
rôle de la couche session, et expliquez pourquoi ils sont implémentés
« plus haut »&nbsp;dans la pile.

> 💬 **Votre réponse :**
>
> En HTTP/1.1, la couche 5 (Session) est très peu visible parce qu'HTTP est
**stateless** par design — chaque requête est indépendante. La notion de session
est donc déléguée à l'applicatif. Deux mécanismes courants :

1. **Cookies HTTP** (`Set-Cookie` / `Cookie`) : le serveur émet un identifiant
   de session qu'il stocke côté serveur. Le client le renvoie à chaque requête.

2. **Tokens (JWT, OAuth)** : un jeton signé contient l'état de session
   (utilisateur, droits, expiration). Le client l'envoie en header
   `Authorization: Bearer ...`.

**Pourquoi "plus haut" dans la pile ?**

- La **couche 5 OSI** a été largement abandonnée en pratique. TCP (couche 4)
  gère déjà la session bas-niveau (handshake, FIN, numéros de séquence).
- Les besoins de session **applicative** (qui est connecté ? quels droits ?
  panier d'achat ?) sont **spécifiques à chaque application** — ils ne
  rentreraient pas dans un protocole générique de couche 5.
- Implémenter au niveau application permet de **traverser les proxies,
  load-balancers, CDN** sans casser la session (un cookie suit le HTTP, peu
  importe combien de TCP différents servent les requêtes).

## Pièges fréquents

* **Capture vide (`Number of packets: 0`)** — vous avez lancé `curl` *avant*
  tcpdump, ou tcpdump a été tué avant d'écrire son buffer. Solution : utilisez
  bien `-U -c 30` (étape 2) et déclenchez le trafic *après* le message
  `listening on eth0…`.
* **`tcpdump: /tmp/http.pcap: Permission denied`** — un fichier appartenant à
  l'utilisateur `tcpdump` (créé par une capture précédente) bloque
  l'écriture. Solution : `docker exec lab_client rm -f /tmp/http.pcap`.
* **`tshark … | less` n'affiche rien** — vous êtes dans un environnement
  sans TTY (script, pipeline). Retirez `| less` ou utilisez
  `docker exec -it lab_client sh -c "… | less"`.
