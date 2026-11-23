[English](API.md) · **Français**

# API HTTP v1 de mcproxy

L'API de données de mcproxy : du JSON sur HTTPS et des événements en direct (Server-Sent Events), servis
par le service mcproxy lui-même. Tout client peut l'utiliser : une application de bureau, une page web
(`fetch` / `EventSource`), Home Assistant, des scripts (`curl`). Aucun client n'a de traitement
particulier : tout ce qu'un client affiche passe par ces endpoints.

**Sommaire**

1. [Démarrage rapide](#démarrage-rapide)
2. [Conventions](#conventions)
3. [Liste des endpoints](#liste-des-endpoints)
4. [Référence des endpoints](#référence-des-endpoints)
5. [Événements en direct](#événements-en-direct)
6. [Notions](#notions)
7. [Où se trouve chaque réglage](#où-se-trouve-chaque-réglage)

---

## Démarrage rapide

### URL de base

```
https://<proxy>:5080/api/v1
```

`<proxy>` est l'adresse de la machine qui exécute mcproxy, par exemple `192.168.1.50`. Les exemples
ci-dessous utilisent cette adresse.

### Certificat

L'API utilise HTTPS avec un certificat auto-signé que le proxy crée à son premier démarrage. Vérifiez-le
une fois, puis épinglez-le :

1. Sur la machine du proxy, `mcproxy api` affiche l'empreinte SHA-256 du certificat.
2. Sur le client, lisez l'empreinte que présente le serveur et comparez :

   ```bash
   openssl s_client -connect 192.168.1.50:5080 </dev/null 2>/dev/null \
     | openssl x509 -noout -fingerprint -sha256
   ```

   (`openssl` l'affiche en majuscules avec des deux-points ; `mcproxy api` et `GET /api/v1` la donnent en
   hexadécimal minuscule.)
3. Épinglez-le. Avec `curl`, ignorez la vérification (`-k`, acceptable sur un LAN de confiance) ou épinglez la clé publique :

   ```bash
   # once: the pin of the server's public key
   PIN=$(openssl s_client -connect 192.168.1.50:5080 </dev/null 2>/dev/null \
     | openssl x509 -pubkey -noout | openssl pkey -pubin -outform der \
     | openssl dgst -sha256 -binary | base64)

   curl -k --pinnedpubkey "sha256//$PIN" https://192.168.1.50:5080/api/v1/status
   ```

Les applications devraient épingler le certificat à la première utilisation, comme SSH : afficher
l'empreinte, demander une fois à l'utilisateur, la retenir, et refuser ensuite un certificat différent (en disant pourquoi).

### Jeton (facultatif)

Quand un jeton d'API est défini sur le proxy, chaque requête doit le fournir :

```bash
curl -k -H "Authorization: Bearer <token>" https://192.168.1.50:5080/api/v1/status
```

`EventSource` ne peut pas définir d'en-têtes : passez plutôt `?token=<token>` dans l'URL (il peut être
encodé en pourcentage). Si aucun jeton n'est configuré, aucun en-tête n'est nécessaire.

### Premiers appels

**Quel est ce serveur ?**

```bash
curl -k https://192.168.1.50:5080/api/v1
```

```json
{
  "api": "mcproxy",
  "version": 1,
  "server": "<mcproxy version>",
  "modules": [],
  "tls_fingerprint": "5e8c1f0a9b7d3e2c4a6f8b0d1e3c5a7f9b2d4e6f8a0c1e3b5d7f9a2c4e6b8d0f",
  "endpoints": ["/api/v1/status", "/api/v1/nodes", "/api/v1/messages", "..."]
}
```

**La radio est-elle connectée ?**

```bash
curl -k https://192.168.1.50:5080/api/v1/status
```

```json
{
  "server": { "version": "<mcproxy version>", "uptime_secs": 86400, "modules": [] },
  "radio": {
    "connected": true,
    "connected_secs": 3600,
    "link": "ble",
    "node_name": "Base Station",
    "battery_mv": 4100,
    "path_id_bytes": 1,
    "...": "..."
  },
  "queue": { "waiting": 0, "in_flight": null, "held_acks": 0 },
  "...": "..."
}
```

(L'objet complet se trouve sous [`GET /api/v1/status`](#get-apiv1status).)

**Les deux messages les plus récents**

```bash
curl -k "https://192.168.1.50:5080/api/v1/messages?limit=2"
```

```json
{
  "items": [
    {
      "id": 1234,
      "ts": 1793600400,
      "kind": "channel",
      "channel": "Public",
      "channel_id": "3f9a1c7e5b2d4f60",
      "channel_slot": 0,
      "channel_kind": "public",
      "from": "Hilltop",
      "from_key": null,
      "to": null,
      "to_key": null,
      "with": null,
      "outgoing": false,
      "text": "Good signal up here today",
      "text_state": "shown",
      "hops": 2,
      "snr_db": 6.25,
      "rssi_dbm": -98,
      "path": ["7c", "a1"],
      "path_nodes": [],
      "heard_count": 3,
      "echo_of": null,
      "path_id_bytes": 1,
      "client_id": null,
      "delivery": null,
      "scoped": false,
      "region": null,
      "cli": false
    },
    {
      "id": 1233,
      "ts": 1793600100,
      "kind": "dm",
      "channel": null,
      "channel_id": null,
      "channel_slot": null,
      "channel_kind": null,
      "from": "Base Station",
      "from_key": null,
      "to": "Valley Relay",
      "to_key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
      "with": "7c4d2e8f1a3b",
      "outgoing": true,
      "text": "Are you back online?",
      "text_state": "shown",
      "hops": null,
      "snr_db": null,
      "rssi_dbm": null,
      "path": null,
      "path_nodes": null,
      "heard_count": null,
      "echo_of": null,
      "path_id_bytes": null,
      "client_id": "6f1c2a9e-4b7d-4e2a-9c1f-0a3b5d7e9f11",
      "delivery": {
        "state": "delivered", "route": "direct", "timeout_ms": 4200, "rtt_ms": 2650,
        "at": 1793600103, "repeats": 0, "attempt": 1, "attempts_max": 3
      },
      "scoped": false,
      "region": null,
      "cli": false
    }
  ],
  "next": "1233",
  "newer": null
}
```

(`path_nodes` est abrégé ici ; il contient les [sauts résolus](#sauts-résolus).)

### Envoyer un message

Créez un `id` par message que l'utilisateur envoie (un UUID convient) et gardez-le pour chaque nouvel essai
HTTP de ce message : le même `id` n'est jamais envoyé deux fois.

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/messages \
  -H "Content-Type: application/json" \
  -d '{"id": "6f1c2a9e-4b7d-4e2a-9c1f-0a3b5d7e9f12", "channel": "Public", "text": "Hello from the hills"}'
```

```json
{
  "sent": true,
  "message_id": 1235,
  "client_id": "6f1c2a9e-4b7d-4e2a-9c1f-0a3b5d7e9f12"
}
```

Un DM utilise `"to": "Valley Relay"` (le nom d'un contact ou le début de sa clé) au lieu de `channel`.
Voir [`POST /api/v1/messages`](#post-apiv1messages).

### Écouter les événements en direct

```bash
curl -k -N "https://192.168.1.50:5080/api/v1/events?types=message,message_status"
```

```
event: message
id: 5012
data: {"id":1236,"ts":1793600500,"kind":"channel","channel":"#hills","from":"Hilltop","text":"Anyone on?", ...}
```

Dans un navigateur (le certificat doit avoir été accepté une fois) :

```js
const url = "https://192.168.1.50:5080/api/v1/events?types=message,status&token=" + encodeURIComponent(token);
const events = new EventSource(url);
events.addEventListener("message", (e) => showMessage(JSON.parse(e.data)));
events.addEventListener("status", () => refreshStatus());
events.addEventListener("lagged", () => reloadEverything());
```

Voir [Événements en direct](#événements-en-direct).

---

## Conventions

| Sujet | Règle |
|---|---|
| Format | JSON uniquement, UTF-8, `Content-Type: application/json`. Les noms de champs sont en `snake_case`. Chaque enregistrement est un objet avec des champs nommés (`{"ts": .., "packets": ..}`), jamais un tuple. |
| Unités | Dans le nom du champ : `battery_mv`, `snr_db`, `rssi_dbm`, `noise_dbm`, `temperature_c`, `distance_km`, `uptime_secs`, `size_bytes`, `age_ms`. Les valeurs sont des nombres bruts ; les clients les mettent en forme. |
| Temps | Secondes unix entières (UTC) dans les champs nommés `ts` ou `*_at`. Les durées sont en `*_secs` (ou `*_ms`). |
| Valeurs manquantes | `null`, jamais `0` ni `""`. Un champ est présent même quand il vaut `null` (les exceptions sont indiquées pour chaque endpoint). Les entiers pouvant être nuls sont des nombres JSON ou `null`, jamais des chaînes. |
| Clés des noeuds | Un noeud est sa clé publique, 64 caractères hexadécimaux minuscules (`key`). Un `{key}` dans un chemin d'URL accepte la clé complète ou un préfixe unique d'au moins 2 caractères hexadécimaux. Pas hexadécimal ou trop court : `400 bad_request` ; aucune correspondance : `404 not_found` ; plusieurs correspondances : `409 ambiguous`. La recherche porte sur les noeuds entendus et les contacts archivés. |
| Type de noeud | `"chat"`, `"repeater"`, `"room"`, `"sensor"` ou `"unknown"` (`"self"` pour notre propre compagnon dans les vues du mesh). |
| Ids de chemin | Chaînes hexadécimales minuscules, une par saut (`["7c", "a1"]`) ; 1 à 3 octets chacune. Voir [Taille des ids de chemin](#taille-des-ids-de-chemin). |
| Plage de temps | Les endpoints d'historique acceptent `?since=<unix>&until=<unix>`, ou `?hours=<n>` (en remontant depuis maintenant ; `hours=0` = tout ; plus de 878400, soit 100 ans : 400, en texte brut). `since` l'emporte sur `hours`. La valeur par défaut de `hours` est donnée pour chaque endpoint (généralement 24). |
| Pagination | Les listes répondent `{"items": [...], "next": <cursor or null>}`. Passez `?cursor=<next>` pour la page suivante et `?limit=<n>` (500 par défaut, 5000 au plus, sauf indication contraire de l'endpoint). **Les curseurs sont des chaînes** (`"500"`, `"1233"`) ; renvoyez-les sans les modifier. `next: null` = dernière page. Un curseur est une position : une liste qui change entre deux pages (un noeud entendu à nouveau remonte en tête de `/nodes`) peut répéter ou sauter un élément à la jonction ; dédoublonnez donc par clé ou par id. |
| Erreurs | Le statut HTTP indique le genre d'erreur ; le corps dit pourquoi : `{"error": {"code": "not_found", "message": "no node matches 'a1b2'"}}`. `code` est un mot stable destiné aux machines, `message` est destiné aux personnes. Voir le tableau ci-dessous. |
| Stabilité | Des champs peuvent être **ajoutés** au sein de la v1. Un champ n'est jamais renommé, supprimé ni doté d'un autre sens dans la v1 (ce serait `/api/v2`). Les clients ignorent les champs qu'ils ne connaissent pas, et traitent une valeur d'énumération inconnue (un nouveau `link`, `kind`, `type`) comme inconnue, pas comme une erreur. |

**Statuts d'erreur**

| Statut | `code` typique | Signification |
|---|---|---|
| 400 | `bad_request` (ou un code précis) | Un paramètre ou un champ du corps est manquant ou incorrect. Rien n'a été fait. |
| 401 | `unauthorized` | Jeton manquant ou incorrect. |
| 403 | `forbidden` | L'adresse du client n'est pas dans la liste autorisée du proxy. (Certains endpoints utilisent aussi 403 pour `refused` / `not_admin`.) |
| 404 | `not_found` (ou un code précis) | Ce noeud, ce message, ce canal ou cet endpoint n'existe pas. |
| 409 | `ambiguous`, `busy`, ... | Un préfixe de clé correspond à plusieurs noeuds, ou la requête est en conflit avec l'état actuel. |
| 500 | `internal` | Erreur inattendue du serveur. |
| 503 | `unavailable`, `radio_down`, `no_store`, `no_data`, ... | Le compagnon n'est pas connecté ou n'a pas répondu, le stockage des messages est désactivé, la base de données ne peut pas être ouverte, ou le proxy est en cours d'arrêt. |
| 504 | `no_answer`, `unknown` | Pas de réponse à temps du mesh ou du compagnon. |

Chaque endpoint peut aussi répondre 401, 403 et 500 ; les endpoints qui lisent la base de données
répondent `503 no_data` quand elle ne peut pas être ouverte. Ces erreurs ne sont pas répétées dans chaque
tableau d'erreurs, ni les erreurs de `{key}` ci-dessus.

### Transport et accès

- **Où :** servie par le service mcproxy lui-même (section de configuration `[api]` : `enabled = true`,
  `listen = "0.0.0.0:5080"`). Chaque requête lit la base de données par sa propre connexion en lecture
  seule, de sorte que les requêtes de l'API ne ralentissent jamais la liaison radio.
- **HTTPS** par défaut (`[api] tls = true`) : les requêtes et les réponses sont chiffrées. Au premier
  démarrage, le proxy crée un certificat auto-signé (`api-cert.pem` et `api-key.pem` dans `~/.config/mcproxy` de
  l'utilisateur du service, à côté de la clé des messages, lisibles par cet utilisateur seul, conservés lors des mises à jour). Il
  n'expire jamais. `mcproxy api` affiche son empreinte SHA-256 et `GET /api/v1` la donne
  (`tls_fingerprint`, `null` sans TLS). Pour votre propre certificat :
  `[api] cert = "/path/fullchain.pem"`, `key = "/path/key.pem"`. `tls = false` = HTTP simple (par
  exemple derrière votre propre proxy inverse). Les navigateurs acceptent le certificat auto-signé une fois, à la main.
- Le port de l'application MeshCore (5000) est un service différent : TCP simple, protocole compagnon MeshCore.
- **Accès :** la liste d'adresses `[server] allow` s'applique (vide = toute adresse qui peut l'atteindre).
  `[api] token` facultatif (par défaut : aucun) : s'il est défini, chaque requête doit fournir
  `Authorization: Bearer <token>` ou `?token=`. Sans l'un ni l'autre, l'API est ouverte au réseau (le
  proxy écrit un avertissement dans les logs au démarrage).
- **Navigateurs :** CORS est ouvert (`Access-Control-Allow-Origin: *`). Une requête préliminaire (`OPTIONS`) autorise les
  méthodes `GET, POST, PUT, PATCH, DELETE, OPTIONS` et les en-têtes `Authorization, Content-Type,
  Last-Event-ID`, avec une mise en cache de 86400 s. Quand un jeton est défini, c'est lui qui assure la protection.
- **Chaque réponse** porte `X-Content-Type-Options: nosniff` et `Cache-Control: no-store`.
- **Vie privée :** les messages sont fournis avec leur texte (les nôtres et ceux des autres, tous les canaux et tous les DM) aussi longtemps
  que le stockage le garde. Les clés des canaux et les mots de passe de connexion enregistrés ne sont jamais servis.
- **Les requêtes lentes** sont écrites dans les logs par le proxy : une réponse qui a pris 3 s ou plus, et un client
  parti avant sa réponse. La chaîne de requête (qui peut contenir le jeton) n'est pas écrite dans les logs.
- **Les chemins inconnus** répondent `404 not_found` ("no such endpoint").
- **Les corps** doivent être du JSON valide envoyé avec `Content-Type: application/json` ; sinon, la requête est
  rejetée avant d'atteindre l'endpoint (400 ou 415, avec un corps en texte brut au lieu de l'objet
  d'erreur JSON). Il en va de même pour un paramètre de chemin du mauvais type (un `{id}` de message
  non numérique).

---

## Liste des endpoints

| Méthode | Chemin | Ce qu'il fait |
|---|---|---|
| **Découverte et état** | | |
| GET | [`/api/v1`](#get-apiv1) | Nom du serveur, version de l'API, empreinte du certificat |
| GET | [`/api/v1/status`](#get-apiv1status) | État en direct : radio, stockage, contacts, file, applications |
| **Messages** | | |
| GET | [`/api/v1/messages`](#get-apiv1messages) | Messages reçus et envoyés, les plus récents d'abord |
| POST | [`/api/v1/messages`](#post-apiv1messages) | Envoyer un message (ou en renvoyer un) |
| DELETE | [`/api/v1/messages`](#delete-apiv1messages) | Supprimer une conversation DM du stockage |
| DELETE | [`/api/v1/messages/{id}`](#delete-apiv1messagesid) | Supprimer un message stocké |
| GET | [`/api/v1/messages/{id}/heard`](#get-apiv1messagesidheard) | Chaque copie entendue du paquet d'un message |
| **Canaux** | | |
| GET | [`/api/v1/channels`](#get-apiv1channels) | Canaux connus, avec leur trafic |
| POST | [`/api/v1/channels`](#post-apiv1channels) | Ajouter un canal sur le compagnon |
| PATCH | [`/api/v1/channels/{id}`](#patch-apiv1channelsid) | Définir la région propre à un canal |
| DELETE | [`/api/v1/channels/{id}`](#delete-apiv1channelsid) | Retirer un canal du compagnon |
| **Noeuds** | | |
| GET | [`/api/v1/nodes`](#get-apiv1nodes) | Noeuds entendus et contacts |
| GET | [`/api/v1/nodes/{key}`](#get-apiv1nodeskey) | Un noeud |
| PATCH | [`/api/v1/nodes/{key}`](#patch-apiv1nodeskey) | Favori, suivi, "à moi", chemin des DM |
| GET | [`/api/v1/nodes/{key}/adverts`](#get-apiv1nodeskeyadverts) | Chaque annonce entendue d'un noeud |
| GET | [`/api/v1/nodes/{key}/heard`](#get-apiv1nodeskeyheard) | Comment nous entendons un noeud au fil du temps |
| GET | [`/api/v1/nodes/{key}/status`](#get-apiv1nodeskeystatus) | Réponses d'état d'un répéteur / salon |
| GET | [`/api/v1/nodes/{key}/telemetry`](#get-apiv1nodeskeytelemetry) | Valeurs de télémétrie d'un noeud |
| **Événements en direct** | | |
| GET | [`/api/v1/events`](#get-apiv1events) | Flux Server-Sent Events |
| **Mesh et statistiques** | | |
| GET | [`/api/v1/mesh/summary`](#get-apiv1meshsummary) | Totaux de paquets pour les graphiques |
| GET | [`/api/v1/mesh/packets`](#get-apiv1meshpackets) | Chaque paquet entendu, décodé |
| GET | [`/api/v1/mesh/graph`](#get-apiv1meshgraph) | Le mesh sous forme de graphe de noeuds et de liens |
| GET | [`/api/v1/mesh/delivery`](#get-apiv1meshdelivery) | Le sort de nos DM, par contact et par route |
| GET | [`/api/v1/mesh/hours`](#get-apiv1meshhours) | Trafic par heure de la journée |
| GET | [`/api/v1/mesh/replay`](#get-apiv1meshreplay) | Paquets d'une fenêtre de temps avec leur trajet dans le mesh |
| GET | [`/api/v1/mesh/adverts`](#get-apiv1meshadverts) | Heures brutes des annonces de chaque noeud |
| GET | [`/api/v1/messages/stats`](#get-apiv1messagesstats) | Nombre de messages par heure, jour, canal, expéditeur |
| GET | [`/api/v1/messages/senders`](#get-apiv1messagessenders) | Tous ceux qui ont écrit |
| **Actions sur les noeuds** | | |
| POST | [`/api/v1/nodes/{key}/login`](#post-apiv1nodeskeylogin) | Se connecter à un répéteur / salon / capteur |
| DELETE | [`/api/v1/nodes/{key}/login`](#delete-apiv1nodeskeylogin) | Oublier un mot de passe enregistré |
| POST | [`/api/v1/nodes/{key}/cli`](#post-apiv1nodeskeycli) | Exécuter une commande CLI sur un noeud |
| GET | [`/api/v1/nodes/{key}/cli`](#get-apiv1nodeskeycli) | Commandes envoyées à un noeud et leurs réponses |
| GET | [`/api/v1/cli/commands`](#get-apiv1clicommands) | Les commandes CLI qui existent |
| GET | [`/api/v1/nodes/{key}/neighbours`](#get-apiv1nodeskeyneighbours) | La table des voisins d'un répéteur, à la dernière demande |
| POST | [`/api/v1/nodes/{key}/neighbours`](#post-apiv1nodeskeyneighbours) | Demander la table des voisins maintenant |
| GET | [`/api/v1/nodes/{key}/owner`](#get-apiv1nodeskeyowner) | Firmware, nom et texte du propriétaire, à la dernière demande |
| POST | [`/api/v1/nodes/{key}/owner`](#post-apiv1nodeskeyowner) | Les demander maintenant |
| POST | [`/api/v1/nodes/{key}/status`](#post-apiv1nodeskeystatus) | Demander son état à un noeud maintenant |
| POST | [`/api/v1/nodes/{key}/telemetry`](#post-apiv1nodeskeytelemetry) | Demander sa télémétrie à un noeud maintenant |
| POST | [`/api/v1/trace`](#post-apiv1trace) | Tracer un chemin, SNR par saut |
| POST | [`/api/v1/nodes/{key}/restore`](#post-apiv1nodeskeyrestore) | Remettre un contact archivé sur le compagnon |
| GET | [`/api/v1/nodes/{key}/paths`](#get-apiv1nodeskeypaths) | Comment les chemins d'un noeud ont changé |
| **Régions** | | |
| GET | [`/api/v1/regions`](#get-apiv1regions) | Noms de régions connus |
| PUT | [`/api/v1/regions`](#put-apiv1regions) | Remplacer les noms ajoutés à la main |
| POST | [`/api/v1/regions/fetch`](#post-apiv1regionsfetch) | Demander leurs régions aux répéteurs proches |
| **Réglages** | | |
| GET | [`/api/v1/settings`](#get-apiv1settings) | Réglages du proxy qu'un client peut changer |
| PATCH | [`/api/v1/settings`](#patch-apiv1settings) | Les changer |
| **Logs** | | |
| GET | [`/api/v1/logs`](#get-apiv1logs) | Le log du service |
| **Compagnon** | | |
| PATCH | [`/api/v1/companion/settings`](#patch-apiv1companionsettings) | Réglages d'ajout automatique et taille des ids de chemin |
| POST | [`/api/v1/companion/advert`](#post-apiv1companionadvert) | Envoyer notre annonce maintenant |
| PATCH | [`/api/v1/companion/region`](#patch-apiv1companionregion) | La région par défaut du compagnon |
| GET | [`/api/v1/companion/history`](#get-apiv1companionhistory) | Notre compagnon au fil du temps |

---

## Référence des endpoints

## Découverte et état

### GET /api/v1

Nomme le serveur et liste ce qu'il offre ; n'a besoin ni de base de données ni de radio.

```bash
curl -k https://192.168.1.50:5080/api/v1
```

```json
{
  "api": "mcproxy",
  "version": 1,
  "server": "<mcproxy version>",
  "modules": [],
  "tls_fingerprint": "5e8c1f0a9b7d3e2c4a6f8b0d1e3c5a7f9b2d4e6f8a0c1e3b5d7f9a2c4e6b8d0f",
  "endpoints": [
    "/api/v1/status", "/api/v1/companion/history", "/api/v1/nodes", "/api/v1/nodes/{key}",
    "/api/v1/messages", "/api/v1/channels", "/api/v1/events"
  ]
}
```

| Champ | Type | Signification |
|---|---|---|
| `api` | string | Toujours `"mcproxy"`. |
| `version` | integer | Version de l'API (`1`). |
| `server` | string | La version de mcproxy. |
| `modules` | array de string | Modules facultatifs intégrés à ce programme, `[]` s'il n'y en a pas. Un client n'affiche les parties d'un module que s'il est listé. |
| `tls_fingerprint` | string ou null | SHA-256 du certificat (DER), en hexadécimal minuscule ; `null` sans TLS. |
| `endpoints` | array de string | Chaque chemin d'endpoint servi (paramètres de chemin sous la forme `{key}` / `{id}`). |

### GET /api/v1/status

L'état en direct du proxy (ce que `mcproxy status` affiche).

```bash
curl -k https://192.168.1.50:5080/api/v1/status
```

```json
{
  "server": { "version": "<mcproxy version>", "uptime_secs": 86400, "modules": [] },
  "radio": {
    "connected": true,
    "connected_secs": 3600,
    "link": "ble",
    "serial_port": "",
    "host": "",
    "address": "AA:BB:CC:DD:EE:FF",
    "ble_name": "Base Station (AA:BB:CC:DD:EE:FF)",
    "node_name": "Base Station",
    "node_key": "e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4",
    "reconnects": 2,
    "last_down_reason": null,
    "battery_mv": 4100,
    "battery_at": 1793600000,
    "temperature_c": 6.5,
    "temperature_at": 1793600000,
    "frames_from_radio": 1234,
    "frames_to_radio": 567,
    "path_id_bytes": 1,
    "auto_add": {
      "mode": "selected", "chat": true, "repeater": true, "room": false, "sensor": false,
      "overwrite_oldest": true, "max_hops": 0
    }
  },
  "store": {
    "messages": 10, "metadata_rows": 900, "devices": 3, "size_bytes": 1048576,
    "packets": 50000, "node_rows": 3000,
    "contacts_archived": 420, "contacts_archived_removed": 70,
    "retention": {
      "text_days": 365, "metadata_days": 0, "packets_days": 180, "nodes_days": 365, "contacts_days": 0
    }
  },
  "contacts": { "cached": 347, "fresh": true, "max": 350, "prune_days": 0 },
  "queue": {
    "waiting": 0,
    "in_flight": { "cmd": "GET_BATT_AND_STORAGE", "owner": "#1", "age_ms": 120 },
    "held_acks": 0
  },
  "polite": { "enabled": true, "burst": 4, "gap_secs": 3, "waiting": 0, "acks_open": 0, "acks_open_max": 6 },
  "clients": [
    { "id": 3, "address": "192.168.1.20", "app": "MeshCore", "connected_secs": 120,
      "frames_in": 10, "frames_out": 20, "pending": 0 }
  ],
  "tracked": [
    { "key": "a1b2c3d4e5f6", "name": "Hilltop", "result": "ok", "at": 1793600000 }
  ]
}
```

**`server`**

| Champ | Type | Signification |
|---|---|---|
| `version` | string | Version de mcproxy. |
| `uptime_secs` | integer | Depuis le démarrage du service. |
| `modules` | array de string | Modules facultatifs intégrés à ce programme, `[]` s'il n'y en a pas. |

**`radio`** (le compagnon)

| Champ | Type | Signification |
|---|---|---|
| `connected` | bool | Le compagnon est connecté en ce moment. |
| `connected_secs` | integer | Depuis le dernier établissement (ou la dernière perte) de la liaison. |
| `link` | string | `"ble"` (Bluetooth), `"usb"` (câble) ou `"tcp"` (un compagnon avec Wi-Fi ou Ethernet). Traitez les autres valeurs comme inconnues. |
| `serial_port` | string | Le port USB configuré ; `""` = trouvé automatiquement, ou autre liaison. |
| `host` | string | L'adresse réseau configurée du compagnon, `"host"` ou `"host:port"` (utile avec `tcp`). |
| `address` | string | L'adresse Bluetooth configurée (utile avec `ble`). |
| `ble_name` | string | Le nom d'affichage du compagnon pour toute liaison, par exemple `"Base Station (AA:BB:CC:DD:EE:FF)"`, `"Base Station (USB ttyACM0)"`, `"Base Station (TCP 192.168.1.50:5000)"` (le champ garde son ancien nom). |
| `node_name` | string | Le nom de noeud du compagnon. |
| `node_key` | string ou null | Sa clé publique. |
| `reconnects` | integer | Reconnexions de la liaison depuis le démarrage. |
| `last_down_reason` | string ou null | Pourquoi la liaison est tombée la dernière fois. |
| `battery_mv`, `battery_at` | integer ou null | Dernière mesure de la batterie et son heure. |
| `temperature_c`, `temperature_at` | number / integer ou null | Dernière température de l'historique du compagnon et son heure. |
| `frames_from_radio`, `frames_to_radio` | integer | Compteurs de trames. |
| `path_id_bytes` | integer ou null | Taille des ids de chemin avec laquelle notre compagnon envoie (son mode de hachage de chemin, 1-3) ; `null` = firmware antérieur à v10 ou pas encore lu. Voir [Taille des ids de chemin](#taille-des-ids-de-chemin). |
| `auto_add` | object ou null | Quels contacts le compagnon ajoute de lui-même ; `null` = pas encore lu. Voir [Réglages d'ajout automatique](#réglages-dajout-automatique). |

**`store`** (`null` quand le stockage des messages est désactivé)

| Champ | Type | Signification |
|---|---|---|
| `messages` | integer | Textes de messages stockés. |
| `metadata_rows` | integer | Lignes de messages stockées (métadonnées). |
| `devices` | integer | Appareils (applications par adresse) connus du stockage. |
| `size_bytes` | integer | Taille de la base de données. |
| `packets`, `node_rows` | integer | Paquets et lignes d'historique des noeuds stockés. |
| `contacts_archived` | integer | Contacts dans l'[archive des contacts](#archive-des-contacts-et-autres-compagnons). |
| `contacts_archived_removed` | integer | Parmi eux, combien ont quitté la liste du compagnon. |
| `retention.text_days`, `metadata_days`, `packets_days`, `nodes_days`, `contacts_days` | integer | Conservation par type de données, en jours (0 = pour toujours). |

Les compteurs du stockage sont mis en cache pendant 15 s au plus.

**`contacts`**

| Champ | Type | Signification |
|---|---|---|
| `cached` | integer | Contacts dans le cache que le proxy garde de la liste du compagnon. |
| `fresh` | bool | Le cache est vérifié et à jour (`false` = en cours d'actualisation depuis la radio). |
| `max` | integer ou null | Combien de contacts le compagnon peut contenir. |
| `prune_days` | integer | Suppression automatique des contacts non entendus pendant ce nombre de jours (0 = désactivé). |

**`queue`**

| Champ | Type | Signification |
|---|---|---|
| `waiting` | integer | Commandes en attente du compagnon. |
| `in_flight` | object ou null | La commande que le compagnon traite : `cmd` = son nom (tel que `mcproxy watch` l'affiche), `owner` = qui l'a envoyée, `age_ms` = depuis son envoi. `null` = inactif. |
| `held_acks` | integer | ACK gardés pour les applications du port 5000 qui ne sont pas connectées (voir [Remise au moins une fois](#remise-au-moins-une-fois-et-ack-gardés)). |

Valeurs de `owner` (stables) : `"#<id>"` = une application (l'`id` dans `clients`), `"proxy"` = le proxy lui-même
(contrôle de santé, historique, lecture des messages), `"api"` = cette API (`POST /api/v1/messages`, renvois,
favoris).

**`polite`** (voir [Envoyer sans saturer](#envoyer-sans-saturer))

| Champ | Type | Signification |
|---|---|---|
| `enabled` | bool | L'envoi sans saturer est activé. |
| `burst` | integer | Commandes radio envoyées d'un coup avant que l'espacement commence. |
| `gap_secs` | integer | Intervalle entre les commandes radio espacées. |
| `waiting` | integer | Commandes radio qui attendent leur tour en ce moment. |
| `acks_open`, `acks_open_max` | integer | DM qui attendent leur ACK en ce moment, et le maximum simultané. |

**`clients`** (applications connectées sur le port MeshCore)

| Champ | Type | Signification |
|---|---|---|
| `id` | integer | Id de connexion (comme dans `in_flight.owner`). |
| `address` | string | L'adresse IP de l'application. |
| `app` | string ou null | Le nom de l'application, une fois qu'elle l'a indiqué. |
| `connected_secs` | integer | Depuis sa connexion. |
| `frames_in`, `frames_out` | integer | Trames reçues d'elle / envoyées vers elle. |
| `pending` | integer ou null | Messages en attente pour cet appareil dans le stockage. |

**`tracked`** (noeuds dont l'état et la télémétrie sont demandés régulièrement)

| Champ | Type | Signification |
|---|---|---|
| `key` | string | L'entrée suivie telle qu'enregistrée (un préfixe de clé). |
| `name` | string | Son nom (l'entrée elle-même s'il est inconnu). |
| `result` | string | Résultat du dernier passage, en texte lisible (`"waiting for the first round"` avant le premier). |
| `at` | integer ou null | Quand ce résultat est arrivé ; `null` avant le premier passage. |

| Statut | Code | Quand |
|---|---|---|
| 503 | `unavailable` | Le proxy est en cours d'arrêt. |

## Messages

### GET /api/v1/messages

Messages reçus et envoyés, **les plus récents d'abord**, paginés par id de message.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Plage de temps (voir [Conventions](#conventions)). |
| `kind` | string | tous | `dm` ou `channel`. |
| `channel` | string | tous | Un id de canal ; sinon un nom (tous les canaux de ce nom) ; sinon un numéro d'emplacement (le canal qui s'y trouve maintenant). Voir [Ids et emplacements des canaux](#ids-et-emplacements-des-canaux). |
| `with` | string hex | aucun | Une seule conversation DM, **dans les deux sens** : la clé d'un contact ou son début (au moins 4 hex). Filtré avant la pagination. |
| `from` | string | aucun | Reçus de cet expéditeur seulement : un nom (sans tenir compte des majuscules) ou un préfixe de clé. |
| `limit` | integer | 200 | De 1 à 2000. |
| `cursor` | string | aucun | Continuer en dessous de cet id de message (donné par `next`). |
| `after` | integer | aucun | Donner à la place la page **juste au-dessus** de cet id de message (les `limit` messages juste plus récents que lui), toujours les plus récents d'abord. |

`channel` et `with` sont appliqués dans la requête elle-même : les pages d'un canal calme sont donc aussi
rapides que celles d'un canal actif. `with` se combine avec `cursor` et `after`.

**Défiler dans les deux sens.** Sans `after` : `next` = continuer en dessous (à passer comme `cursor`), `newer` vaut
`null`. Avec `after` : `newer` = continuer au-dessus (à repasser comme `after` ; `null` quand la page
atteint le plus récent), `next` vaut `null`. Un client garde une fenêtre de quelques pages : `cursor=<oldest
shown>` charge les plus anciens, `after=<newest shown>` charge les plus récents, l'extrémité opposée est abandonnée. Les nouveaux messages arrivent
sous forme d'événements `message` : une fenêtre placée sur les plus récents les ajoute, une fenêtre remontée plus haut les ignore et les lit
avec `after` quand elle redescend.

```bash
curl -k "https://192.168.1.50:5080/api/v1/messages?with=7c4d&limit=50"
```

```json
{
  "items": [
    {
      "id": 1240,
      "ts": 1793600900,
      "kind": "dm",
      "channel": null,
      "channel_id": null,
      "channel_slot": null,
      "channel_kind": null,
      "from": "Valley Relay",
      "from_key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
      "to": null,
      "to_key": null,
      "with": "7c4d2e8f1a3b",
      "outgoing": false,
      "text": "Back online, battery replaced",
      "text_state": "shown",
      "hops": 1,
      "snr_db": 7.5,
      "rssi_dbm": -91,
      "path": ["a1"],
      "path_nodes": [
        {
          "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
          "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
          "name": "Hilltop", "kind": "repeater", "lat": 12.3, "lon": 45.6,
          "confidence": 0.9, "choices": 1
        }
      ],
      "heard_count": 1,
      "echo_of": null,
      "path_id_bytes": 1,
      "client_id": null,
      "delivery": null,
      "scoped": false,
      "region": null,
      "cli": false
    }
  ],
  "next": null,
  "newer": null
}
```

| Champ | Type | Signification |
|---|---|---|
| `id` | integer | Id du message stocké. |
| `ts` | integer | Quand le proxy l'a reçu (les nôtres : quand il a été stocké). |
| `kind` | string | `"dm"` ou `"channel"`. |
| `channel` | string ou null | Le nom du canal (d'après son id, quel que soit son emplacement actuel). `null` quand `channel_id` l'est, ou quand le nom n'est pas connu. |
| `channel_id` | string ou null | 16 hex. Regroupez une conversation avec ce champ. `null` pour un DM, et pour un message de canal dont le canal ne peut pas être déterminé (voir [Ids et emplacements des canaux](#ids-et-emplacements-des-canaux)). |
| `channel_slot` | integer ou null | L'emplacement actuel de son canal sur le compagnon ; `null` = il n'y est pas, ou `channel_id` vaut `null`. |
| `channel_kind` | string ou null | `"public"`, `"hashtag"` (`#name`) ou `"private"` (aussi quand le nom est inconnu) ; `null` quand `channel_id` l'est. |
| `from` | string | DM : le nom du contact (ou un préfixe de sa clé). Canal : le nom que l'expéditeur a mis devant le texte. Les nôtres : le nom de notre noeud. |
| `from_key` | string ou null | La clé complète (64 hex) : celle de l'expéditeur d'un DM quand c'est un contact connu, celle de l'expéditeur sur un canal quand son nom correspond à exactement un noeud. `null` quand elle n'est pas connue, et pour les nôtres. |
| `to` | string ou null | DM que nous avons envoyés : le contact destinataire (son nom, ou le début de sa clé quand le nom n'est pas connu). |
| `to_key` | string ou null | DM que nous avons envoyés : la clé complète de ce contact (64 hex), `null` quand elle n'est pas connue. Absent sur les proxys antérieurs à 0.2.1. |
| `with` | string ou null | DM, dans les deux sens : les 6 premiers octets (12 hex) de la clé de l'autre partie, toujours renseigné, même pour quelqu'un qui n'est pas un contact. **Regroupez les conversations DM avec ce champ, jamais par un nom** (deux contacts peuvent avoir le même) ; il peut être donné comme `?with=`. `null` pour les messages de canal. Absent sur les proxys antérieurs à 0.2.1. |
| `outgoing` | bool | Envoyé par nous (depuis n'importe quelle app ou `POST /api/v1/messages`). |
| `text` | string ou null | Toujours renseigné (les nôtres aussi), `null` une fois que le texte a expiré ou s'il ne peut pas être lu. |
| `text_state` | string | `"shown"`, `"expired"` (supprimé par la durée de conservation ; les métadonnées restent), `"data"` (un datagramme de canal : des données binaires qu'une app a envoyées sur le canal, pas du texte ; voir `data`) ou `"unreadable"` (stocké, mais la clé de chiffrement n'est pas disponible). Absent sur les anciens proxys. |
| `data` | object ou null | Un datagramme de canal (`text_state` `"data"`) : `{"type": 65535, "hex": "0a1b2c"}`. `type` = le type de données donné par l'app qui envoie (0-65535 ; MeshCore réserve 0, 65535 sert aux expériences) ; `hex` = ses octets. `null` pour tout message texte. |
| `hops` | integer ou null | Nombre de sauts du paquet. `null` pour nos propres messages (affichez plutôt `delivery.route`). |
| `snr_db` | number ou null | SNR du message reçu. |
| `rssi_dbm` | integer ou null | RSSI de la première copie entendue de son paquet (messages reçus). |
| `path` | array ou null | **Le chemin de l'expéditeur** : les ids tels qu'ils sont dans le paquet, d'après la première copie entendue. Voir [Chemins des messages](#chemins-des-messages). |
| `path_nodes` | array ou null | Chaque id de `path` [résolu](#sauts-résolus), dans le même ordre, avec l'expéditeur comme contexte quand il est connu. |
| `heard_count` | integer ou null | Combien de fois nous avons entendu le paquet qui le portait ; pour nos messages de canal, combien de répétitions nous en avons entendues (écho compris). `null` = le paquet n'a pas été associé (DM que nous avons envoyés : toujours). |
| `echo_of` | integer ou null | Sur un message de canal reçu qui est **notre propre message revenant** par le mesh : l'`id` de notre message. Masquez-le ou fusionnez-le avec le nôtre. Exact (empreinte du paquet). À partir de 0.2.1, une telle copie n'est plus stockée : seulement sur les lignes plus anciennes. |
| `path_id_bytes` | integer ou null | La taille d'id de chemin que son paquet avait réellement, d'après une copie entendue (les nôtres : la taille avec laquelle le compagnon l'a envoyé). `null` tant qu'aucune copie n'a été entendue ; pour les nôtres, affichez alors `radio.path_id_bytes` de `/status`. |
| `client_id` | string ou null | Pour les messages envoyés par `POST /api/v1/messages`, l'`id` que le client lui a donné. |
| `delivery` | object ou null | Messages que nous avons envoyés : voir [États de remise](#états-de-remise-et-nouveaux-essais-des-dm). `null` pour les messages reçus et avant que le compagnon ait répondu. |
| `scoped` | bool | Reçu : son paquet a été envoyé dans une région (portée flood). |
| `region` | string ou null | Reçu : le nom de cette région, quand elle correspond à une région connue. |
| `cli` | bool | Une commande CLI de noeud ou sa réponse, envoyée par une app via le port 5000, pas un message de discussion : gardez-la hors des discussions, des compteurs de non-lus et des notifications. |

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | `kind` autre que `dm` / `channel` ; `with` pas en hex ou plus court que 4. |
| 404 | `not_found` | `channel` ne correspond à aucun canal. |
| 409 | `ambiguous` | Les clés de contact de plusieurs conversations commencent par `with`. |

### POST /api/v1/messages

Envoie un message par le compagnon. **Pas de doublons, par conception** : voir
[Envoyer sans doublons](#envoyer-sans-doublons).

**Corps : un nouveau message**

| Champ | Type | Obligatoire | Signification |
|---|---|---|---|
| `id` | string | oui | L'id du client pour ce message : lettres, chiffres, `- _ . :`, au plus 64 caractères (un UUID convient). Créé une fois par message envoyé par l'utilisateur, jamais par essai HTTP. |
| `text` | string | oui | Non vide, sans caractère NUL. Au plus 160 octets pour un DM ; sur un canal, 160 moins le préfixe `"<name>: "` du compagnon. |
| `channel` | string ou integer | l'un de `channel` / `to` | L'id du canal (tel que `GET /api/v1/channels` le donne ; à préférer : il est le même quel que soit l'emplacement du canal, donc une liste d'emplacements périmée atteint le bon canal ou obtient `not_found`, jamais un autre canal), son nom (sans tenir compte des majuscules) ou son numéro d'emplacement. |
| `to` | string | l'un de `channel` / `to` | Le nom exact d'un contact (sans tenir compte des majuscules) ou le début de sa clé publique (hex, au moins 4). |
| `route` | string | non | DM seulement : `"auto"` (par défaut : le chemin stocké par le compagnon, flood s'il n'y en a pas), `"flood"` (le chemin stocké est d'abord oublié), `"path"` (avec `path` ; stocké d'abord). Comme avec l'appui long sur Envoyer dans l'app MeshCore, le compagnon garde le changement. Les renvois passent quand même en flood avant le dernier essai. |
| `path` | array de string hex | avec `route: "path"` | Ids des répéteurs, premier saut d'abord, 1-3 octets chacun, tous de la même taille ; `[]` = direct. |

**Corps : un renvoi** (voir [Renvoyer](#renvoyer))

| Champ | Type | Obligatoire | Signification |
|---|---|---|---|
| `id` | string | oui | Un **nouvel** id de client pour cet essai. |
| `resend_of` | integer | oui | L'id du message stocké à renvoyer. S'utilise sans `text`, `to` ni `channel`. |
| `route` | string | non | Comme ci-dessus (renvoi sur un canal : seulement `auto`). |

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/messages \
  -H "Content-Type: application/json" \
  -d '{"id": "b7e2c9d4-1f3a-4c5e-8d7b-2a4c6e8f0a13", "to": "Valley Relay", "text": "See you at the ridge", "route": "auto"}'
```

**Envoyé** (HTTP 202) :

```json
{
  "sent": true,
  "message_id": 1241,
  "client_id": "b7e2c9d4-1f3a-4c5e-8d7b-2a4c6e8f0a13"
}
```

**Non envoyé** (il n'est jamais parti ; vous pouvez l'envoyer de nouveau sans risque avec le même `id`) :

```json
{
  "sent": false,
  "code": "not_sent",
  "reason": "not_contact",
  "error": "\"Valley Relay\" is not a contact of the companion - a DM needs it in the contacts",
  "client_id": "b7e2c9d4-1f3a-4c5e-8d7b-2a4c6e8f0a13"
}
```

**Inconnu** (HTTP 504 ; il a été transmis au compagnon mais aucune réponse n'est venue, il est donc peut-être parti) :

```json
{
  "sent": false,
  "code": "unknown",
  "error": "the companion did not answer",
  "message_id": 1243,
  "client_id": "b7e2c9d4-1f3a-4c5e-8d7b-2a4c6e8f0a13"
}
```

| Champ | Type | Signification |
|---|---|---|
| `sent` | bool | Il est parti. |
| `message_id` | integer ou null | Le message stocké (`null` quand le stockage des messages est désactivé). Pour un renvoi, l'id stocké qui a été renvoyé. Présent aussi avec `unknown` : le message est stocké avec l'état de remise `"unknown"` et peut être renvoyé avec `resend_of` (absent quand le proxy lui-même n'a pas répondu à temps, et sur les proxys antérieurs à 0.2.1). |
| `client_id` | string | L'`id` de la requête (présent dans chaque réponse). |
| `code` | string | `"not_sent"` ou `"unknown"`. |
| `reason` | string | Avec `not_sent` : pourquoi (voir plus bas). |
| `error` | string | Explication lisible. |

La réponse arrive une fois que le compagnon a répondu (en général en moins d'une seconde ; au plus
`[server] command_timeout_secs` + 5 s, plus le temps pendant lequel l'[envoi sans saturer](#envoyer-sans-saturer) peut le retenir, plus jusqu'à 12 s de
[nouveaux essais si occupé](#nouveaux-essais-si-occupé)).

**Résultats**

| Statut | Corps | Quand |
|---|---|---|
| 202 | `sent: true` | Le compagnon l'a accepté. |
| 400 | `{"error": {...}}` | Corps mal formé : `id` absent ou incorrect, pas de `text`, à la fois `channel` et `to` ou aucun des deux, `text` avec NUL, `route` / `path` incorrect, `resend_of` avec `text` / `to` / `channel`. Rien n'est envoyé. |
| 400 | `not_sent`, raison `bad_request` | Texte vide, ou `route` sur un message de canal. |
| 400 | `not_sent`, raison `too_long` | Texte trop long. |
| 404 | `not_sent`, raison `not_found` | Canal absent du compagnon (aussi quand il a été retiré avant le tour du message) ; renvoi : aucun message de ce genre envoyé d'ici, ou son canal n'est plus sur le compagnon. |
| 404 | `not_sent`, raison `not_contact` | Destinataire du DM absent des contacts du compagnon (un DM a besoin de sa clé, et l'autre partie a besoin de la nôtre pour le lire). |
| 409 | `not_sent`, raison `ambiguous` | Plusieurs contacts correspondent à `to`. |
| 409 | `not_sent`, raison `not_dm`, `not_failed`, `expired`, `too_old` | Renvoi refusé (voir [Renvoyer](#renvoyer)). |
| 503 | `not_sent`, raison `radio_down` | Le compagnon n'est pas connecté. |
| 503 | `not_sent`, raison `busy` | La file des commandes est pleine, la file de paquets du compagnon est restée pleine après les nouveaux essais, ou (canal) les canaux du compagnon sont encore en cours de lecture après la connexion. |
| 503 | `not_sent`, raison `refused` | Le compagnon a dit non. |
| 503 | `not_sent`, raison `no_store` | Renvoi alors que le stockage des messages est désactivé. |
| 503 | `{"error": {"code": "unavailable"}}` | Le proxy est en train de s'arrêter. |
| 504 | `unknown` | Pas de réponse à temps : il est peut-être parti. Ne renvoyez pas automatiquement : demandez à l'utilisateur, puis renvoyez-le avec `resend_of` (son `message_id`) et un nouvel `id`. Sans `message_id`, cherchez d'abord le message par son `client_id`. |

Après un envoi, exactement un événement `message` avec `"client_id"` et `"outgoing": true` suit (aussi
quand aucune copie n'est stockée ; son `id` vaut alors `null`). Un client le rattache à son envoi en attente même si la
réponse HTTP a été perdue : cet événement signifie qu'il est parti ou, quand le compagnon n'a pas répondu, qu'il est
peut-être parti (un événement `message_status` avec l'état `"unknown"` suit). Les apps MeshCore sur le port 5000 en reçoivent une copie
chacune quand les copies sont activées (voir [Stockage et copies](#stockage-et-copies)).

Sur la machine du proxy : `mcproxy send --channel Public "text"` / `mcproxy send --to "Valley Relay" "text"`.

### DELETE /api/v1/messages

Supprime une conversation DM entière (dans les deux sens) du stockage du proxy.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `with` | string hex | obligatoire | La clé du contact ou son début (hex, au moins 4). Un début doit correspondre à un seul contact. |

```bash
curl -k -X DELETE "https://192.168.1.50:5080/api/v1/messages?with=7c4d2e8f"
```

```json
{ "ok": true, "deleted": 42 }
```

Un événement `messages_deleted` suit. Voir [Stockage et copies](#stockage-et-copies) pour savoir ce que touche
une suppression.

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | Pas de `with`, ou pas en hex / plus court que 4. |
| 409 | `ambiguous` | Les clés de plusieurs contacts commencent par `with`. Rien n'a été supprimé. |
| 503 | `refused` | Le stockage des messages est désactivé, ou la suppression a échoué. |
| 503 | `unavailable` | Le proxy n'a pas répondu à temps. |

### DELETE /api/v1/messages/{id}

Supprime un message stocké.

```bash
curl -k -X DELETE https://192.168.1.50:5080/api/v1/messages/1240
```

```json
{ "ok": true, "deleted": 1 }
```

Un événement `messages_deleted` suit.

| Statut | Code | Quand |
|---|---|---|
| 400 | (texte brut) | `{id}` n'est pas un nombre. |
| 404 | `not_found` | Aucun message de ce genre. |
| 503 | `refused` | Le stockage des messages est désactivé, ou la suppression a échoué. |
| 503 | `unavailable` | Le proxy n'a pas répondu à temps. |

### GET /api/v1/messages/{id}/heard

Chaque fois que nous avons entendu le paquet qui portait un message (la première copie et les répétitions par d'autres
répéteurs), les plus anciennes d'abord ; pour nos propres messages, les répétitions entendues une fois associées.

```bash
curl -k https://192.168.1.50:5080/api/v1/messages/1234/heard
```

```json
{
  "matched": true,
  "items": [
    {
      "ts": 1793600400, "snr_db": 6.25, "rssi_dbm": -98, "hops": 2, "route": "flood",
      "path": ["7c", "a1"],
      "via": {
        "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
        "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
        "name": "Hilltop", "kind": "repeater", "lat": 12.3, "lon": 45.6,
        "confidence": 0.92, "choices": 1
      }
    }
  ]
}
```

| Champ | Type | Signification |
|---|---|---|
| `matched` | bool | `false` = aucun paquet n'a été associé à ce message (`items` vaut `[]`). |
| `items[].via` | [saut](#sauts-résolus) ou null | Le répéteur qui nous l'a transmis ; `null` = entendu directement de l'expéditeur. |

Le paquet est associé par canal (ou expéditeur + nous pour les DM) et nombre de sauts parmi ceux entendus dans la
minute précédente : [au mieux](#données-déduites).

| Statut | Code | Quand |
|---|---|---|
| 404 | `not_found` | Aucun message de ce genre. |

## Canaux

### GET /api/v1/channels

Tous les canaux connus, par id, avec leur trafic : d'abord les emplacements du compagnon (dans l'ordre des emplacements, selon la dernière
lecture), puis les canaux déjà vus sur un compagnon (`slot: null`, `on_companion: false` ; `name: null`
quand il n'est pas connu).

```bash
curl -k https://192.168.1.50:5080/api/v1/channels
```

```json
{
  "items": [
    { "id": "3f9a1c7e5b2d4f60", "slot": 0, "name": "Public", "kind": "public",
      "on_companion": true, "messages": 812, "last_at": 1793600400, "region": null },
    { "id": "d41e8a2c6b9f0357", "slot": 1, "name": "#hills", "kind": "hashtag",
      "on_companion": true, "messages": 95, "last_at": 1793600500, "region": "eu-se" },
    { "id": "9b0c2e4a6d8f1357", "slot": null, "name": "Hikers", "kind": "private",
      "on_companion": false, "messages": 12, "last_at": 1793000000, "region": null }
  ],
  "next": null
}
```

| Champ | Type | Signification |
|---|---|---|
| `id` | string ou null | Id du canal, 16 hex (voir [Ids et emplacements des canaux](#ids-et-emplacements-des-canaux)). |
| `slot` | integer ou null | Emplacement sur le compagnon connecté ; `null` = il n'y est pas. |
| `name` | string ou null | Nom du canal. |
| `kind` | string | `"public"`, `"hashtag"` ou `"private"`. |
| `on_companion` | bool | Actuellement sur le compagnon connecté. |
| `messages`, `last_at` | integer / integer ou null | Messages encore stockés (depuis toujours, dans la limite de conservation des métadonnées) et heure du plus récent. Pour une période, utilisez `by_channel` de `GET /api/v1/messages/stats`. |
| `region` | string ou null | La [région](#régions) propre au canal, gardée par le proxy : un nom, `"*"` (aucune région), ou `null` (la valeur par défaut du compagnon s'applique). |

Les clés des canaux ne sont jamais données.

### POST /api/v1/channels

Ajoute un canal sur le compagnon, dans son premier emplacement vide (un canal qui s'y trouve déjà n'est jamais remplacé).

| Champ | Type | Obligatoire | Signification |
|---|---|---|---|
| `name` | string | oui | `"#name"` pour un canal hashtag, `"Public"`, ou n'importe quel nom pour un canal privé. De 1 à 31 octets, sans caractères de contrôle. |
| `key` | string | canaux privés | La clé du canal, 32 caractères hex (telle que les apps la partagent). |

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/channels \
  -H "Content-Type: application/json" -d '{"name": "#hills"}'
```

```json
{ "ok": true, "slot": 1, "name": "#hills", "id": "d41e8a2c6b9f0357" }
```

La réponse arrive une fois que le compagnon l'a pris ; un événement `channels` suit.

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | Pas de nom, nom hors de 1-31 octets ou avec des caractères de contrôle, clé qui n'a pas 32 hex. |
| 400 | `needs_key` | Un canal privé sans sa clé. |
| 409 | `exists` | Le compagnon a déjà ce canal. |
| 409 | `full` | Aucun emplacement vide. |
| 409 | `busy` | Les emplacements sont encore en cours de lecture après la connexion ; ou le stockage des messages est désactivé et aucune app n'est connectée sur le port 5000 (les messages en attente sur le compagnon seraient perdus). |
| 503 | `radio_down` | Le compagnon n'est pas connecté. |
| 503 | `refused` | Le compagnon l'a refusé (ou n'a pas répondu). |

### PATCH /api/v1/channels/{id}

Définit la [région](#régions) propre à un canal, gardée par le proxy pour toutes les apps (une par canal, la
dernière définie l'emporte).

`{id}` est l'id du canal, 16 caractères hex, tel que `GET /api/v1/channels` le donne.

| Champ | Type | Obligatoire | Signification |
|---|---|---|---|
| `region` | string ou null | oui | Un nom de région, `"*"` (aucune région), ou `null` (la valeur par défaut du compagnon). |

```bash
curl -k -X PATCH https://192.168.1.50:5080/api/v1/channels/d41e8a2c6b9f0357 \
  -H "Content-Type: application/json" -d '{"region": "eu-se"}'
```

```json
{ "ok": true, "region": "eu-se" }
```

Un événement `channels` `{"channel_id", "region"}` suit.

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | `{id}` n'a pas 16 hex, nom de région incorrect. |
| 503 | `no_store` | Le stockage des messages est désactivé. |
| 503 | `unavailable` | L'enregistrement a échoué, ou pas de réponse à temps. |

### DELETE /api/v1/channels/{id}

Retire un canal du compagnon ; son emplacement est vidé.

`{id}` est l'id du canal ; sinon son nom ; sinon son numéro d'emplacement. Il n'est retiré que si son emplacement contient encore
ce canal quand la commande part.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `messages` | string | `keep` | `keep` garde ses messages stockés ; `delete` les supprime aussi du proxy (par id de canal), une fois que le compagnon a confirmé le retrait. |

```bash
curl -k -X DELETE "https://192.168.1.50:5080/api/v1/channels/d41e8a2c6b9f0357?messages=keep"
```

```json
{ "ok": true, "slot": 1, "name": "#hills", "id": "d41e8a2c6b9f0357" }
```

Un événement `channels` suit ; avec `messages=delete`, les messages sont supprimés une fois que le compagnon montre
que le canal n'y est plus, et un événement `channel_messages_deleted` suit. Si le compagnon ne le montre
pas (son emplacement contient encore le canal, ou la liaison a été perdue avant), les messages sont gardés et
aucun événement de ce type n'arrive. Le réglage de région du canal est gardé : il appartient à l'identité du canal et revient si
le canal est ajouté de nouveau.

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | `messages` autre que `keep` / `delete`. |
| 404 | `not_found` | Le compagnon n'a pas ce canal. |
| 409 | `busy` | Comme pour `POST /api/v1/channels`. |
| 503 | `radio_down` | Le compagnon n'est pas connecté. |
| 503 | `refused` | Le compagnon l'a refusé (ou n'a pas répondu). |

## Noeuds

### GET /api/v1/nodes

Noeuds entendus sur le mesh et contacts du compagnon, les plus récemment entendus d'abord.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=0` (tout) | Fenêtre portant sur le dernier moment où un noeud a été entendu. |
| `kind` | string | tous | `chat`, `repeater`, `room` ou `sensor`. |
| `contacts` | string | voir plus bas | Quels contacts du compagnon hors de la fenêtre sont ajoutés : `favorites` ou `all`. |
| `mine` | bool | `false` | `true` = seulement [vos propres noeuds](#vos-propres-noeuds), quelle que soit la fenêtre. |
| `limit`, `cursor` | | 500 | Pagination (voir [Conventions](#conventions)). |

**Ce qui est listé, dans cet ordre :**

1. Les noeuds entendus dans la fenêtre, les plus récemment entendus d'abord (`in_window: true`).
2. Les contacts du compagnon hors de la fenêtre : **les favoris toujours**, et tous les contacts avec
   `contacts=all` ou quand la fenêtre couvre tout (`hours=0`, par défaut) (`in_window: false`).
3. Quand tous les contacts sont listés : les contacts archivés qui n'ont jamais été entendus (`contact: false`,
   `left_companion_at` renseigné). Voir [Archive des contacts et autres compagnons](#archive-des-contacts-et-autres-compagnons).

Avec `mine=true` : seulement vos propres noeuds, les plus récemment entendus d'abord, puis ceux jamais entendus. Seule
la page demandée est construite : des pages plus petites coûtent donc moins.

```bash
curl -k "https://192.168.1.50:5080/api/v1/nodes?kind=repeater&hours=24&limit=100"
```

```json
{
  "items": [
    {
      "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "name": "Hilltop",
      "kind": "repeater",
      "first_seen_at": 1790000000,
      "last_seen_at": 1793600300,
      "adverts": 412,
      "snr_db": 8.25,
      "rssi_dbm": -88,
      "hops": 0,
      "via": null,
      "lat": 12.3,
      "lon": 45.6,
      "distance_km": 4.2,
      "contact": true,
      "favorite": true,
      "knows_us": true,
      "out_path": [],
      "access": { "role": "admin", "firmware": 1, "at": 1793500000 },
      "mine": true,
      "saved_login": true,
      "path_id_bytes": 1,
      "path_id_bytes_at": 1793600300,
      "left_companion_at": null,
      "other_companion": null,
      "heard": true,
      "in_window": true
    },
    {
      "key": "c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2",
      "name": "Valley Relay",
      "kind": "repeater",
      "first_seen_at": null,
      "last_seen_at": null,
      "adverts": 0,
      "snr_db": null,
      "rssi_dbm": null,
      "hops": null,
      "via": null,
      "lat": null,
      "lon": null,
      "distance_km": null,
      "contact": true,
      "favorite": true,
      "knows_us": null,
      "out_path": ["a1"],
      "access": null,
      "mine": false,
      "saved_login": false,
      "path_id_bytes": null,
      "path_id_bytes_at": null,
      "left_companion_at": null,
      "other_companion": null,
      "heard": false,
      "in_window": false,
      "companion_heard_at": 1793400000
    }
  ],
  "next": null
}
```

**Objet noeud**

| Champ | Type | Signification |
|---|---|---|
| `key` | string | Clé publique, 64 hex. |
| `name` | string | Nom du noeud. |
| `kind` | string | Type du noeud. |
| `first_seen_at`, `last_seen_at` | integer ou null | Première et dernière annonce entendue ; `null` = jamais entendu. |
| `adverts` | integer | Annonces entendues. |
| `snr_db`, `rssi_dbm`, `hops` | number / integer ou null | D'après la dernière annonce entendue. |
| `via` | [saut](#sauts-résolus) ou null | Le répéteur par lequel nous avons entendu son annonce en dernier, résolu avec tout le chemin de l'annonce comme contexte ; `null` = direct. |
| `lat`, `lon` | number ou null | Position. |
| `distance_km` | number ou null | Depuis la position de notre compagnon. |
| `contact` | bool | Dans la liste de contacts du compagnon. |
| `favorite` | bool ou null | Favori sur le compagnon lui-même (ses indicateurs de contact, tels que l'app MeshCore les montre) ; `null` quand ce n'est pas un contact. |
| `knows_us` | true ou null | `true` quand il est prouvé que le noeud a notre clé ; `null` = inconnu. Voir [Données déduites](#données-déduites). |
| `out_path` | array ou null | Contacts : le chemin que le compagnon utilise pour lui envoyer des DM, ids des répéteurs, premier saut d'abord ; `[]` = direct ; `null` = aucun connu (les DM partent en flood) ou pas un contact. Il n'y a pas de date de mise à jour : le compagnon n'en garde pas. |
| `access` | object ou null | Notre rôle d'après la dernière réponse de connexion : `{"role", "firmware", "at"}` ; `null` = aucune connexion vue. Voir [Rôles et connexions](#rôles-et-connexions). |
| `mine` | bool | L'un de [vos propres noeuds](#vos-propres-noeuds). |
| `saved_login` | bool | Un mot de passe de connexion est enregistré pour lui. |
| `path_id_bytes`, `path_id_bytes_at` | integer ou null | Le mode de hash de chemin propre au noeud (1-3), d'après sa dernière annonce flood, et quand. Voir [Taille des ids de chemin](#taille-des-ids-de-chemin). |
| `left_companion_at` | integer ou null | Quand il a quitté la liste de contacts du compagnon ; `null` = il y est encore, ou n'a jamais été un contact. |
| `other_companion` | string ou null | La clé d'un autre compagnon dont il est un contact (gardé à part) ; `null` sinon. |
| `heard` | bool | `false` = un contact dont le proxy n'a jamais entendu d'annonce : les valeurs viennent de sa trame de contact. |
| `in_window` | bool | `false` = listé parce que c'est un contact / favori, pas entendu dans la fenêtre. |
| `companion_heard_at` | integer ou null | **Seulement sur les éléments avec `heard: false`** : quand le compagnon a mis à jour le contact pour la dernière fois. |

Les contacts suivent le compagnon (le cache de contacts du proxy). Les noms et clés des contacts archivés sont
utilisés partout où une clé est résolue (expéditeurs de DM, préfixes `{key}`).

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | `kind` inconnu ; `contacts` autre que `all` / `favorites`. |

### GET /api/v1/nodes/{key}

Un seul noeud : l'[objet noeud](#get-apiv1nodes) (aussi pour un contact jamais entendu, ou un contact
archivé), plus ses dernières réponses et son chemin DM résolu.

```bash
curl -k https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4
```

```json
{
  "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "name": "Hilltop",
  "kind": "repeater",
  "first_seen_at": 1790000000,
  "last_seen_at": 1793600300,
  "adverts": 412,
  "snr_db": 8.25,
  "rssi_dbm": -88,
  "hops": 0,
  "via": null,
  "lat": 12.3,
  "lon": 45.6,
  "distance_km": 4.2,
  "contact": true,
  "favorite": true,
  "knows_us": true,
  "out_path": [],
  "access": { "role": "admin", "firmware": 1, "at": 1793500000 },
  "mine": true,
  "saved_login": true,
  "path_id_bytes": 1,
  "path_id_bytes_at": 1793600300,
  "left_companion_at": null,
  "other_companion": null,
  "tracked": true,
  "latest": {
    "status": { "at": 1793590000, "battery_mv": 4100, "uptime_secs": 86400, "noise_floor": -110 },
    "owner": { "at": 1793000000, "firmware": "v1.9.1", "owner": "Mountain club" },
    "neighbours": { "at": 1793500000, "total": 12 }
  },
  "out_path_nodes": []
}
```

| Champ | Type | Signification |
|---|---|---|
| `tracked` | bool | Son état et sa télémétrie lui sont demandés régulièrement. |
| `latest.status` | object ou null | Dernière réponse d'état : `at`, `battery_mv`, `uptime_secs`, `noise_floor` (chacun peut valoir `null`). `null` = n'a jamais répondu. |
| `latest.owner` | object ou null | Dernière réponse sur le propriétaire : `at`, `firmware`, `owner`. |
| `latest.neighbours` | object ou null | Dernière table des voisins : `at`, `total`. |
| `out_path_nodes` | array ou null | `out_path` [résolu](#sauts-résolus), dans le même ordre (premier saut d'abord), lu comme le chemin de retour du contact vers nous pour que chaque id ait ses voisins comme contexte. `null` quand `out_path` vaut `null` ; `[]` = direct. |

`heard` et `in_window` n'apparaissent ici que pour un contact jamais entendu (tous deux `false`, avec
`companion_heard_at`) ; un noeud entendu n'a ni l'un ni l'autre.

| Statut | Code | Quand |
|---|---|---|
| 404 | `not_found` | Noeud inconnu. |

### PATCH /api/v1/nodes/{key}

Modifie un noeud : son favori et son chemin DM sur le compagnon, et son suivi et « mine » sur le
proxy. N'importe quelle partie des champs :

| Champ | Type | Signification |
|---|---|---|
| `favorite` | bool | Met / retire le contact des favoris **sur le compagnon** (réécrit avec seulement son indicateur de favori changé, comme le fait l'app MeshCore). Les apps MeshCore le voient la prochaine fois qu'elles lisent les contacts. |
| `track` | bool | Demander à ce répéteur / salon / capteur son état et sa télémétrie toutes les `[observer] track_interval_mins` (par défaut 720, toutes les 12 heures ; une requête LoRa chacun), ou arrêter. Se connecte en invité (avec le mot de passe enregistré du noeud, sinon `[observer] track_password`) quand il ne répond pas. Enregistré dans `[observer] track`, comme `mcproxy track add / remove`. Résultats : `GET /api/v1/nodes/{key}/status` et `/telemetry`. |
| `mine` | bool ou null | `true` l'ajoute à [vos propres noeuds](#vos-propres-noeuds), `false` le garde hors de la liste même avec une connexion admin, `null` revient au mode automatique. |
| `path` | array ou null | Le chemin que le compagnon utilise pour les DM vers ce contact : `null` = l'oublier (le prochain DM part en flood), `[]` = direct, `["a1", "b2"]` = par ces répéteurs, premier saut d'abord (1-3 octets chacun, tous de la même taille). Stocké sur le compagnon (les apps MeshCore le voient). |

```bash
curl -k -X PATCH https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4 \
  -H "Content-Type: application/json" -d '{"favorite": true, "track": true}'
```

```json
{
  "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "track": true,
  "favorite": true
}
```

La réponse contient `key` et chaque champ modifié, avec la valeur obtenue (`mine` tel qu'il est maintenant,
`out_path` pour `path`). Les champs sont appliqués un par un (`track`, `mine`, `favorite`, `path`) :
quand l'un échoue, ceux d'avant restent modifiés. Un événement `node` suit un changement de `favorite`,
`mine` ou `path`.

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | Un autre champ ; `favorite` / `track` pas un bool ; `mine` ni bool ni null ; `path` incorrect ; `track: true` sur un noeud de discussion (seuls les répéteurs, salons et capteurs répondent). |
| 404 | `not_contact` | `favorite`, `path` ou `track: true` sur un noeud qui n'est pas un contact du compagnon. |
| 503 | `no_store` | `mine` alors que le stockage des messages est désactivé. |
| 503 | `unavailable` | Le compagnon n'est pas connecté, a refusé ou n'a pas répondu. |

### GET /api/v1/nodes/{key}/adverts

Toutes les annonces entendues d'un noeud, les plus anciennes d'abord.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Plage de temps. |
| `limit`, `cursor` | | 500 | Pagination. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/nodes/c3d4e5f6/adverts?hours=48"
```

```json
{
  "items": [
    {
      "ts": 1793590000, "snr_db": 5.5, "rssi_dbm": -97, "hops": 1, "route": "flood",
      "path": ["a1"],
      "path_nodes": [
        { "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
          "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
          "name": "Hilltop", "kind": "repeater", "lat": 12.3, "lon": 45.6,
          "confidence": 0.95, "choices": 1 }
      ],
      "via": { "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "...": "..." },
      "advert_ts": 1793589990,
      "repeat": false
    }
  ],
  "next": null
}
```

| Champ | Type | Signification |
|---|---|---|
| `route` | string | `"flood"` / `"flood+transport"` = une annonce flood ; `"direct"` / `"direct+transport"` = une annonce zéro saut (envoyée en direct, longueur de chemin 0, entendue seulement par les noeuds à portée). |
| `path` | array | Ids de chemin tels que reçus. |
| `path_nodes` | array | Chaque id de `path` [résolu](#sauts-résolus), dans le même ordre, avec le chemin et l'expéditeur de cette annonce comme contexte ; `[]` = entendue directement. |
| `via` | saut ou null | Le dernier répéteur avant nous ; `null` = direct. |
| `advert_ts` | integer ou null | L'horloge de l'expéditeur, d'après le contenu de l'annonce (une valeur par annonce distincte). |
| `repeat` | bool | La même annonce entendue de nouveau par un autre chemin (même `advert_ts` que la ligne précédente). |

### GET /api/v1/nodes/{key}/heard

Comment nous entendons ce noeud au fil du temps, même sans annonces : chaque paquet qu'il nous a relayé (il est le
dernier saut d'un chemin flood) plus ses propres annonces entendues directement.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Plage de temps. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/heard?hours=2"
```

```json
{
  "packets": 230,
  "avg_snr_db": 7.4,
  "avg_rssi_dbm": -90.2,
  "same_hash_nodes": 1,
  "per_hour": [
    { "ts": 1793595600, "packets": 120, "avg_snr_db": 7.6, "avg_rssi_dbm": -89.8 },
    { "ts": 1793599200, "packets": 110, "avg_snr_db": 7.2, "avg_rssi_dbm": -90.6 }
  ],
  "by_type": [
    { "type": "advert", "packets": 90 },
    { "type": "channel", "packets": 80 }
  ]
}
```

| Champ | Type | Signification |
|---|---|---|
| `per_hour` | array | [Série complète](#séries-horaires-complètes), une ligne par heure. |
| `same_hash_nodes` | integer | Autres noeuds connus dont la clé commence par le même octet. Un dernier saut n'est qu'un id court, leurs relais peuvent donc être comptés ici aussi : affichez une indication quand > 0. |

Les moyennes sont arrondies à une décimale.

### GET /api/v1/nodes/{key}/status

Réponses d'état d'un répéteur ou d'un serveur de salon, les plus anciennes d'abord.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Plage de temps. |
| `limit`, `cursor` | | 500 | Pagination. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/status?hours=168"
```

```json
{
  "items": [
    {
      "ts": 1793590000, "battery_mv": 4100, "noise_dbm": -110, "rssi_dbm": -92, "snr_db": 6.5,
      "tx_queue": 0, "recv": 51200, "sent": 20480,
      "recv_flood": 48000, "recv_direct": 3200, "sent_flood": 19000, "sent_direct": 1480,
      "airtime_tx_secs": 3600, "airtime_rx_secs": 7200,
      "dups_flood": 9000, "dups_direct": 120, "errors": 0, "recv_errors": 14, "uptime_secs": 864000
    }
  ],
  "next": null
}
```

| Champ | Type | Signification |
|---|---|---|
| `noise_dbm` | integer ou null | Le bruit du noeud. |
| `rssi_dbm`, `snr_db` | | Du dernier paquet que le noeud a reçu. |
| `tx_queue` | integer ou null | Paquets dans sa file d'envoi. |
| `recv`, `sent`, `recv_flood`, `recv_direct`, `sent_flood`, `sent_direct` | integer ou null | Compteurs de paquets. |
| `airtime_tx_secs`, `airtime_rx_secs` | integer ou null | Temps d'antenne. |
| `dups_flood`, `dups_direct` | integer ou null | Doublons écartés. |
| `errors` | integer ou null | Événements d'erreur. |
| `recv_errors` | integer ou null | Erreurs de réception. |
| `uptime_secs` | integer ou null | Depuis le démarrage du noeud. |

Chaque valeur peut être `null` (tous les firmwares ne la donnent pas).

### GET /api/v1/nodes/{key}/telemetry

Toutes les valeurs de télémétrie d'un noeud, les plus anciennes d'abord, une ligne par valeur.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Plage de temps. |
| `limit`, `cursor` | | 500 | Pagination. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/telemetry"
```

```json
{
  "items": [
    { "ts": 1793590000, "channel": 1, "kind": "voltage", "value": 4.1 },
    { "ts": 1793590000, "channel": 1, "kind": "temperature", "value": 6.5 },
    { "ts": 1793590000, "channel": 2, "kind": "humidity", "value": 71.0 }
  ],
  "next": null
}
```

| Champ | Type | Signification |
|---|---|---|
| `channel` | integer | Le canal du capteur (Cayenne LPP). |
| `kind` | string | `"voltage"`, `"temperature"`, `"humidity"`, ... (le type LPP). |
| `value` | number | La valeur, dans l'unité habituelle du type. |

## Mesh et statistiques

### GET /api/v1/mesh/summary

Totaux des paquets entendus, pour des graphiques.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Période. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/mesh/summary?hours=24"
```

```json
{
  "packets": 5000,
  "avg_snr_db": 6.5,
  "avg_rssi_dbm": -95.0,
  "by_type": [
    { "type": "advert", "packets": 900, "avg_snr_db": 5.0 },
    { "type": "channel", "packets": 700, "avg_snr_db": 6.8 }
  ],
  "per_hour": [
    { "ts": 1793595600, "packets": 200, "messages": 35, "avg_snr_db": 6.0,
      "by_type": { "advert": 60, "dm": 5, "channel": 30, "path": 40, "ack": 65 } },
    { "ts": 1793599200, "packets": 0, "messages": 0, "avg_snr_db": null, "by_type": {} }
  ],
  "via": [
    { "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "name": "Hilltop", "kind": "repeater", "lat": 12.3, "lon": 45.6,
      "confidence": 0.98, "choices": 1, "packets": 800, "avg_snr_db": 7.0, "id_bytes": 1 }
  ]
}
```

| Champ | Type | Signification |
|---|---|---|
| `avg_snr_db`, `avg_rssi_dbm` | number ou null | Moyennes simples sur les paquets (voir [SNR et RSSI](#snr-et-rssi)). |
| `by_type` | array | Paquets par type (`type` `null` = non décodé), les plus nombreux d'abord. Types : `request`, `response`, `dm`, `ack`, `advert`, `channel`, `channel-data`, `anon-request`, `path`, `trace`, `multipart`, `control`, `raw`, `other`. |
| `per_hour` | array | [Complet](#séries-horaires-complètes). |
| `per_hour[].by_type` | object | Paquets de cette heure par type (mêmes noms ; `"unknown"` pour un type non décodé). Un type non entendu cette heure-là est omis (à lire comme 0). La somme des valeurs donne `packets`. |
| `per_hour[].messages` | integer | Paquets `dm` + `channel` entendus cette heure-là (tous les paquets entendus, pas seulement ceux qui nous sont destinés ; les répétitions comptent une fois par réception). La part des messages dans le trafic est `messages / packets`. |
| `via` | array | Le **dernier id du chemin des paquets flood** : le répéteur dont notre compagnon a entendu le paquet. Un [saut résolu](#sauts-résolus) plus `packets`, `avg_snr_db` et `id_bytes` (1-3). Voir ci-dessous. |

**Lire `via`.** Les paquets à routage direct ne sont pas comptés (leur chemin liste les sauts encore à
venir). Il y a une entrée par chaîne d'id : cela compte donc des ids, pas des répéteurs. Le même
répéteur apparaît une fois par taille d'id utilisée par le mesh (`id_bytes`), un id de 1 octet peut
désigner plusieurs répéteurs (résolu comme le dernier saut avant nous : `confidence`, `could_be`), et
les ids qu'aucun répéteur entendu ne possède (`choices: 0`) viennent souvent d'autres mesh ou de
paquets abîmés qui ont passé le contrôle de la radio. Pour les « répéteurs entendus directement »,
comptez les entrées avec `choices` > 0 et au moins quelques paquets, ou utilisez
`GET /api/v1/mesh/graph` (liens vers `"self"`).

### GET /api/v1/mesh/packets

Tous les paquets entendus, décodés, du plus ancien au plus récent, paginés par id de paquet.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `since_id` | integer | 0 | Paquets dont l'id est supérieur à celui-ci. |
| `limit` | integer | 1000 | 1 à 5000. |

`next` = le dernier `id` de la page (une chaîne), ou `null` quand la page n'est pas pleine : renvoyez-le
dans `since_id`. Cet endpoint n'accepte pas de période.

```bash
curl -k "https://192.168.1.50:5080/api/v1/mesh/packets?since_id=0&limit=2"
```

```json
{
  "items": [
    {
      "id": 98001, "ts": 1793600400, "snr_db": 6.25, "rssi_dbm": -98,
      "route": "flood", "type": "channel", "version": 0, "hops": 2, "hash_size": 1,
      "path": ["7c", "a1"],
      "path_nodes": [],
      "last_hop": "a1", "size_bytes": 64,
      "node_key": null, "dst": null, "src": null, "channel_hash": "3f", "ack": null,
      "scoped": false, "region": null
    },
    {
      "id": 98002, "ts": 1793600402, "snr_db": 8.0, "rssi_dbm": -90,
      "route": "flood", "type": "advert", "version": 0, "hops": 0, "hash_size": 1,
      "path": [], "path_nodes": [],
      "last_hop": null, "size_bytes": 110,
      "node_key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "dst": null, "src": null, "channel_hash": null, "ack": null,
      "scoped": true, "region": "eu"
    }
  ],
  "next": "98002"
}
```

| Champ | Type | Signification |
|---|---|---|
| `id`, `ts`, `rssi_dbm`, `version`, `hops`, `size_bytes` | integer | Id du paquet, heure, RSSI, version du contenu, nombre de sauts, taille. |
| `snr_db` | number | SNR. |
| `route` | string | `"flood"`, `"flood+transport"`, `"direct"`, `"direct+transport"`. |
| `type` | string | Type de contenu (voir `mesh/summary` `by_type`). |
| `hash_size` | integer | Taille des ids de chemin de ce paquet, 1-3. |
| `path` | array | Ids de chemin, hexa en minuscules. |
| `path_nodes` | array | Chaque id de `path` [résolu](#sauts-résolus), dans le même ordre. Paquets flood : le trajet suivi (origine, répéteurs, nous), chaque id étant résolu avec ses voisins et l'expéditeur (`node_key`, annonces) comme contexte. Paquets directs : leur chemin liste les sauts encore à venir, chaque id est donc résolu seul. `[]` = pas de chemin. |
| `last_hop` | string ou null | Dernier id du chemin. |
| `node_key` | string ou null | Annonces : la clé de l'expéditeur. |
| `dst`, `src` | string ou null | Hash de destination / de source (DM, requêtes). |
| `channel_hash` | string ou null | Hash du canal (le premier octet de l'id du canal). |
| `ack` | string ou null | Code ACK. |
| `scoped` | bool | Envoyé dans une région (portée flood). |
| `region` | string ou null | Nom de cette région, quand elle correspond à une région connue. |

### GET /api/v1/mesh/graph

Le mesh sous forme de graphe, construit au fil du temps à partir de tout ce qui est entendu : chaque
chemin de paquet (deux noeuds voisins dans un chemin s'entendent), les tables de voisins des répéteurs et les traces.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since` | integer | `hours=720` (30 jours) | Fenêtre ; `hours=0` = tout. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/mesh/graph?hours=720"
```

```json
{
  "nodes": [
    { "id": "self", "key": null, "name": "Base Station", "kind": "self", "lat": 12.31, "lon": 45.62,
      "hash": null, "confidence": 1.0, "choices": 1, "packets": 0, "last_seen": null },
    { "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "name": "Hilltop", "kind": "repeater", "lat": 12.3, "lon": 45.6,
      "hash": "a1", "confidence": 0.92, "choices": 1, "packets": 1200, "last_seen": 1793600400 },
    { "id": "h:5d", "key": null, "name": null, "kind": "unknown", "lat": null, "lon": null,
      "hash": "5d", "confidence": 0.0, "choices": 0, "packets": 14, "last_seen": 1793500000 }
  ],
  "links": [
    { "from": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "to": "self",
      "heard_forward": 800, "heard_back": 0, "snr_forward_db": 7.0, "snr_back_db": null,
      "last_seen": 1793600400, "sources": ["paths"] }
  ],
  "shared_ids": [
    { "id": "7c", "nodes": [
      { "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f", "name": "Valley Relay",
        "last_seen": 1793600000, "distance_km": 9.8 },
      { "key": "7c11aa22bb33cc44dd55ee66ff7788997c11aa22bb33cc44dd55ee66ff778899", "name": "Ridge Repeater",
        "last_seen": 1793400000, "distance_km": 31.5 }
    ] }
  ],
  "repeaters": [
    { "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "name": "Hilltop", "kind": "repeater", "path_id_bytes": 1 }
  ]
}
```

Voir [Objets du graphe du mesh](#objets-du-graphe-du-mesh) pour `nodes` et `links`.

| Champ | Type | Signification |
|---|---|---|
| `shared_ids` | array | Ids de chemin correspondant à plusieurs répéteurs entendus : sur une route directe, les deux peuvent relayer le paquet (le firmware relaie dès que l'id correspond), et les cartes ne peuvent pas savoir avec certitude lequel un chemin désignait. |
| `repeaters` | array | Tous les répéteurs et salons entendus (depuis toujours, pas seulement dans la fenêtre), avec leur propre mode de hash de chemin (`path_id_bytes` 1-3 ou `null`), pour construire des routes côté client. |

### GET /api/v1/mesh/delivery

`GET /api/v1/mesh/delivery?hours=720[&node=<key>]` : le sort des DM que nous avons envoyés, depuis
n'importe quelle application, comptés par contact et par route utilisée par le compagnon. Chaque essai
compte, nouveaux essais compris, sous la route réellement prise par cet essai. Compteurs bruts : la
pondération des routes (par lien, selon le signal) revient au client. `node` = seulement les contacts
dont la clé commence ainsi (hexa). Par défaut 30 jours, `hours=0` = tout ; au plus 2000 lignes, celles
avec le plus d'essais d'abord.

```json
{ "paths": [
    { "to": "<contact key>", "name": "Alice", "path": ["a1", "7c"],
      "sent": 12, "delivered": 10, "failed": 2, "rtt_ms_median": 2650,
      "last_at": 1793600103, "last_delivered_at": 1793600103 }
  ],
  "since": 1791008103 }
```

| Champ | Type | Signification |
|---|---|---|
| `to` | string | La clé publique du contact ; les 12 premiers caractères hexa de la clé quand ce n'est pas un contact du compagnon. |
| `name` | string ou null | Son nom, si une de ses annonces a été entendue. |
| `path` | array ou null | La route : `null` = en flood, `[]` = directe (aucun répéteur), sinon les ids de chemin (comme dans le paquet : 1-3 octets en hexa), premier saut d'abord. C'est le chemin que le compagnon avait pour le contact au départ de l'essai ; un essai dont le chemin n'est pas connu n'est pas compté. |
| `sent` | integer | Essais par cette route. |
| `delivered`, `failed` | integer | Essais dont l'ACK est revenu (même en retard) / qui n'en ont pas eu à temps. Les autres sont encore en attente. |
| `rtt_ms_median` | integer ou null | Aller-retour médian de l'ACK pour les essais remis. |
| `last_at`, `last_delivered_at` | integer ou null | Le dernier essai par cette route, et le dernier remis. |

Conservé aussi longtemps que l'historique des noeuds (`[observer] node_history_days`).

### GET /api/v1/mesh/hours

Trafic par heure de la journée (heure locale de la machine du proxy).

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since` | integer | `hours=720` | Fenêtre ; `hours=0` = tout. |
| `node` | string | aucun | Une clé de noeud ou un préfixe : seulement ce que nous avons entendu de ce noeud (ses propres annonces entendues directement, et ce qu'il nous a relayé). |

Avec `node` : le profil journalier du noeud (un répéteur solaire qui se tait la nuit, les meilleures
heures pour le joindre). Sans : tout le trafic, plus notre bruit par heure. Les meilleures heures et les
heures silencieuses sont à déduire de `hours` par le client.

```bash
curl -k "https://192.168.1.50:5080/api/v1/mesh/hours?hours=720&node=a1b2c3d4"
```

```json
{
  "days": 30.0,
  "hours": [
    { "hour": 0, "packets": 310, "packets_per_day": 10.3, "snr_median_db": 6.5, "snr_low_db": 2.0,
      "snr_high_db": 9.5, "rssi_avg": -91.2, "noise_floor_avg": null }
  ],
  "week": [
    [12, 9, 8, 7, 6, 8, 14, 20, 25, 27, 26, 28, 30, 29, 27, 26, 25, 24, 22, 20, 18, 16, 14, 13]
  ]
}
```

| Champ | Type | Signification |
|---|---|---|
| `days` | number | Durée de la fenêtre en jours (au moins 1). |
| `hours` | array | 24 lignes, `hour` 0-23. |
| `hours[].snr_low_db`, `snr_high_db` | number ou null | SNR aux 10e / 90e centiles. |
| `hours[].rssi_avg` | number ou null | RSSI moyen (dBm). |
| `hours[].noise_floor_avg` | number ou null | Notre bruit (dBm) ; seulement sans `node`. |
| `week` | array | 7 lignes (lundi d'abord) de 24 nombres de paquets (abrégé dans l'exemple). |

| Statut | Code | Quand |
|---|---|---|
| 400 / 404 / 409 | | `node` incorrect, inconnu ou ambigu. |

### GET /api/v1/mesh/replay

Tous les paquets d'une fenêtre avec les noeuds traversés, pour une carte animée.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `until` | integer | maintenant | Fin de la fenêtre. |
| `since` | integer | `until` - `hours` | Début de la fenêtre. |
| `hours` | integer | 1 | Durée de la fenêtre quand `since` n'est pas donné (au moins 1). |
| `limit` | integer | 5000 | 1 à 20000 paquets. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/mesh/replay?hours=1"
```

```json
{
  "since": 1793596800,
  "until": 1793600400,
  "nodes": {
    "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90":
      { "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
        "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
        "name": "Hilltop", "kind": "repeater", "lat": 12.3, "lon": 45.6 },
    "self": { "id": "self", "key": null, "name": "Base Station", "kind": "self", "lat": 12.31, "lon": 45.62 }
  },
  "packets": [
    { "id": 98001, "ts": 1793600400, "type": "channel", "route": "flood", "snr": 6.25,
      "chain": ["h:7c", "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "self"],
      "confidence": [0.0, 0.92, 1.0] }
  ],
  "more": false
}
```

| Champ | Type | Signification |
|---|---|---|
| `nodes` | object | Chaque noeud des chaînes, une seule fois, par id : `id`, `key`, `name`, `kind`, `lat`, `lon` (sans statistiques). |
| `packets[].chain` | array d'ids | Origine (si connue), répéteurs, puis `"self"`. Un chemin n'est suivi que pour les paquets flood (pas pour les paquets directs ni les traces). |
| `packets[].confidence` | array de number | Le degré de certitude de chaque `chain[i]` (origine et nous : 1). Voir [Sauts résolus](#sauts-résolus). |
| `more` | bool | La fenêtre contenait plus de `limit` paquets : redemandez à partir du dernier `ts`. |

### GET /api/v1/mesh/adverts

Les heures brutes des annonces de chaque noeud entendu dans la fenêtre, pour que le client trouve les
minuteries (une annonce ne contient pas son intervalle) et les annonces envoyées à la main.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since` | integer | `hours=720` | Fenêtre portant sur le moment où nous avons entendu chaque annonce pour la première fois ; `hours=0` = tout. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/mesh/adverts?hours=720"
```

```json
{
  "nodes": [
    {
      "node": { "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
                "name": "Hilltop", "kind": "repeater" },
      "flood_ts": [1793514000, 1793557200, 1793600400],
      "local_ts": [1793580000],
      "last_heard_at": 1793600410
    }
  ],
  "since": 1791008400
}
```

| Champ | Type | Signification |
|---|---|---|
| `flood_ts` | array d'integer | Annonces flood : l'horloge de l'expéditeur tirée de l'annonce elle-même (exacte, une valeur par annonce distincte, quel que soit le délai en route), de la plus ancienne à la plus récente, les 500 dernières. |
| `local_ts` | array d'integer | Annonces zéro saut (envoyées en direct, entendues seulement à portée), de la même façon. |
| `last_heard_at` | integer | Quand nous avons entendu une de ses annonces pour la dernière fois. |
| `since` | integer | Début de la fenêtre utilisée. |

### GET /api/v1/messages/stats

Nombres de messages pour des graphiques.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Période. |
| `channel` | string | tous | Un id de canal, sinon un nom, sinon un numéro d'emplacement (les DM sont alors exclus). |
| `from` | string | tous | Un nom d'expéditeur (casse indifférente) ou un préfixe de clé : messages reçus seulement. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/messages/stats?hours=48"
```

```json
{
  "per_hour": [ { "ts": 1793595600, "dm": 2, "channel": 14 } ],
  "per_day": [ { "ts": 1793577600, "dm": 9, "channel": 120 } ],
  "by_channel": [
    { "channel": "Public", "channel_id": "3f9a1c7e5b2d4f60", "messages": 90 },
    { "channel": "#hills", "channel_id": "d41e8a2c6b9f0357", "messages": 30 }
  ],
  "by_sender": [
    { "name": "Hilltop", "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "messages": 25 },
    { "name": "Trail Walker", "key": null, "messages": 11 }
  ]
}
```

| Champ | Type | Signification |
|---|---|---|
| `per_hour` | array | [Complet](#séries-horaires-complètes), `ts` = début de l'heure. |
| `per_day` | array | Complet de la même façon, `ts` = minuit UTC. |
| `by_channel` | array | Par canal, les plus nombreux d'abord (`channel` vaut `""` quand son nom n'est pas connu ; `channel_id` vaut `null` pour les messages dont le canal ne peut pas être déterminé). |
| `by_sender` | array | Par expéditeur, les plus nombreux d'abord (`key` `null` quand elle n'est pas connue avec exactitude). Les DM que nous avons envoyés ne sont pas comptés ; nos messages de canal comptent sous le nom de notre noeud (`key` `null`). |

| Statut | Code | Quand |
|---|---|---|
| 404 | `not_found` | `channel` ne correspond à aucun canal. |

### GET /api/v1/messages/senders

Tous ceux qui ont écrit, du plus récent au plus ancien. Les messages que nous avons envoyés ne sont pas comptés.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=0` (tout) | Période. |
| `limit`, `cursor` | | 500 | Pagination. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/messages/senders"
```

```json
{
  "items": [
    { "name": "Hilltop", "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "messages": 40, "dm": 4, "channel": 36, "channels": ["#hills", "Public"],
      "first_at": 1790000000, "last_at": 1793600400 }
  ],
  "next": null
}
```

| Champ | Type | Signification |
|---|---|---|
| `key` | string ou null | Expéditeurs de DM : la clé du contact ; expéditeurs sur un canal : la clé quand exactement un noeud entendu porte ce nom. |
| `channels` | array de string | Canaux où il a écrit (noms, ou ids quand le nom n'est pas connu). |

## Actions sur les noeuds

Les répéteurs, les serveurs de salon et les capteurs ont une CLI, pas une discussion. Notre rôle sur un
noeud décide de ce que nous pouvons y faire : voir [Rôles et connexions](#rôles-et-connexions). Chaque action
ci-dessous passe par LoRa et peut prendre de quelques secondes à une minute ou plus : affichez un état d'attente.

### POST /api/v1/nodes/{key}/login

Se connecte à un noeud. Sans réponse, la connexion est essayée jusqu'à 3 fois, la dernière en flood (le
compagnon oublie la route, comme pour le dernier essai d'un DM) : un échec peut donc prendre une minute ou plus.

| Champ | Type | Obligatoire | Signification |
|---|---|---|---|
| `password` | string | non | Au plus 32 octets. Un mot de passe vide marche quand le noeud nous a dans sa liste d'accès, ou a un mot de passe invité vide (par défaut sur un répéteur ; sur un serveur de salon, c'est en général `hello` par défaut). Sur un serveur de salon, le mot de passe invité donne `read_write`. |
| `save` | bool | non | `true` garde ce mot de passe une fois la connexion réussie (exige un mot de passe) ; `false` oublie celui enregistré une fois la connexion réussie ; absent = pas de changement. |
| `use_saved` | bool | non | `true` se connecte avec le mot de passe enregistré ; s'utilise sans `password` ni `save`. |

Le corps est facultatif (pas de corps = connexion avec mot de passe vide).

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/login \
  -H "Content-Type: application/json" -d '{"use_saved": true}'
```

```json
{ "ok": true, "role": "admin", "firmware": 1, "saved_login": true }
```

| Champ | Type | Signification |
|---|---|---|
| `role` | string ou null | `"admin"`, `"read_write"`, `"read_only"`, `"guest"` ; `null` = un noeud plus ancien qui ne le dit pas. |
| `firmware` | integer ou null | Le niveau de firmware du noeud, tiré de la réponse à la connexion. |
| `saved_login` | bool | Un mot de passe est maintenant enregistré pour lui. |

Un événement `node` suit. Le mot de passe va au noeud et n'est pas écrit dans le log.

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | Mot de passe de plus de 32 octets ; `save: true` sans mot de passe ; `use_saved` avec `password` ou `save`. |
| 403 | `refused` | Le noeud a refusé la connexion. |
| 404 | `not_contact` | Pas un contact du compagnon. |
| 409 | `no_saved` | `use_saved` sans mot de passe enregistré. |
| 409 | `busy` | Une connexion à ce noeud est en cours. |
| 503 | `radio_down` | Le compagnon n'est pas connecté, ou a été perdu pendant la connexion. |
| 503 | `no_store` | Le stockage des messages est désactivé. |
| 503 | `unavailable` | Le proxy est en cours d'arrêt. |
| 504 | `no_answer` | Hors de portée, ou mauvais mot de passe : un répéteur n'y répond jamais, et un serveur de salon avec `allow.read.only` désactivé non plus, on ne peut donc pas distinguer les deux cas. Une connexion avec mot de passe vide sur un répéteur qui a un mot de passe invité finit de la même façon, sauf s'il nous a déjà dans sa liste d'accès. |

### DELETE /api/v1/nodes/{key}/login

Oublie le mot de passe enregistré du noeud.

```bash
curl -k -X DELETE https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/login
```

```json
{ "ok": true }
```

Un événement `node` suit.

| Statut | Code | Quand |
|---|---|---|
| 503 | `no_store` | Le stockage des messages est désactivé. |
| 503 | `unavailable` | L'enregistrement a échoué, ou le proxy est en cours d'arrêt. |
| 504 | `no_answer` | Pas de réponse du proxy à temps. |

### POST /api/v1/nodes/{key}/cli

Envoie une commande CLI à un noeud, comme le fait meshcore-cli (le type de message CLI, une fois), et
répond avec sa réponse. Les commandes passent une à la fois par noeud (les autres attendent : une réponse
ne dit pas à quelle commande elle répond). Une réponse arrive en quelques secondes, ou en dizaines de
secondes sur plusieurs sauts ; l'envoi sans saturer peut la retarder encore.

| Champ | Type | Obligatoire | Signification |
|---|---|---|---|
| `command` | string | oui | 1 à 160 octets, une seule ligne (pas de caractères de contrôle). |

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/cli \
  -H "Content-Type: application/json" -d '{"command": "ver"}'
```

```json
{ "ok": true, "id": 57, "reply": "v1.9.1 (Build: 01-Jan-2026)" }
```

Un événement `cli` suit chaque commande. **Ne renvoyez jamais une commande de vous-même :** le compagnon
donne à chacune un nouvel horodatage, le noeud l'exécute donc à nouveau.

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | Pas de commande, pas 1 à 160 octets, ou caractères de contrôle : non envoyée. |
| 403 | `not_admin` | Notre dernière connexion a donné un autre rôle : non envoyée (le noeud l'ignorerait). |
| 403 | `refused` | Le compagnon ne l'a pas acceptée : non envoyée. |
| 404 | `not_found` / 409 `ambiguous` | La clé ne correspond à aucun noeud, ou à plusieurs : non envoyée. |
| 404 | `not_contact` | Pas un contact du compagnon : non envoyée. |
| 409 | `not_supported` | Un noeud de discussion : non envoyée. |
| 409 | `busy` | Trop de commandes en attente (32) : non envoyée. |
| 503 | `radio_down` | Le compagnon n'est pas connecté : non envoyée. |
| 503 | `no_store` | Le stockage des messages est désactivé (les commandes en ont besoin pour leur historique) : non envoyée. |
| 503 | `unavailable` | Le proxy est en cours d'arrêt : non envoyée. |
| 504 | `no_answer` | Pas de réponse à temps : hors de portée, ou pas admin. **Elle a pu être exécutée.** |
| 504 | `unknown` | Le compagnon s'est déconnecté pendant que la commande attendait ou était en cours. **Elle a pu être exécutée.** |

### GET /api/v1/nodes/{key}/cli

Les commandes envoyées d'ici à un noeud et leurs réponses, de la plus récente à la plus ancienne. Tous les
appareils voient cette liste. Conservée chiffrée, aussi longtemps que les textes des messages.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `limit` | integer | 200 | 1 à 2000. |
| `cursor` | string | aucun | Continuer en dessous de cet id de commande (tiré de `next`). |

```bash
curl -k https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/cli
```

```json
{
  "items": [
    { "id": 58, "ts": 1793600600, "state": "ok", "command": "password ***", "reply": "***" },
    { "id": 57, "ts": 1793600500, "state": "ok", "command": "ver", "reply": "v1.9.1 (Build: 01-Jan-2026)" }
  ],
  "next": null
}
```

| Champ | Type | Signification |
|---|---|---|
| `state` | string | `"ok"`, `"no_answer"`, `"refused"` ou `"unknown"`. |
| `command`, `reply` | string ou null | Les commandes contenant un secret (`password`, `set guest.password`, ...) apparaissent comme `password ***` avec la réponse `***` ; seul le `POST` qui en a envoyé une reçoit sa vraie réponse. `null` quand le texte ne peut pas être lu. |

### GET /api/v1/cli/commands

Les commandes CLI des noeuds MeshCore, dans l'ordre d'affichage, pour une liste de choix.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `kind` | string | tous | `repeater`, `room` ou `sensor` : seulement les commandes de ce genre de noeud. |
| `serial` | bool | `false` | Inclure les commandes qui ne marchent que sur la console USB du noeud lui-même. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/cli/commands?kind=repeater"
```

```json
{
  "items": [
    { "usage": "set tx <dbm>", "about": "Set the transmit power (1-22 dBm)", "section": "Radio",
      "only": null, "needs": null, "danger": true, "secret": false, "serial": false },
    { "usage": "get radio.rxgain", "about": "RX boosted gain state", "section": "Radio",
      "only": null, "needs": "firmware 1.14.1", "danger": false, "secret": false, "serial": false }
  ]
}
```

| Champ | Type | Signification |
|---|---|---|
| `only` | array ou null | Les genres de noeud qui l'ont ; `null` = tous. |
| `needs` | string ou null | Tous les noeuds ne l'ont pas (une version de firmware, des cartes prises en charge). |
| `danger` | bool | Demander avant l'envoi : elle redémarre le noeud, le coupe du réseau ou change sa façon d'émettre. |
| `secret` | bool | Elle contient un secret (masqué dans l'historique). |
| `serial` | bool | Seulement sur la console USB du noeud lui-même (omise sauf avec `serial=true`). |

### GET /api/v1/nodes/{key}/neighbours

La table de voisins qu'un répéteur a donnée la dernière fois qu'on la lui a demandée : les noeuds qu'**il** entend directement, les plus forts d'abord.

```bash
curl -k https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/neighbours
```

```json
{
  "asked_at": 1793500000,
  "total": 12,
  "items": [
    { "prefix": "7c4d2e8f", "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
      "name": "Valley Relay", "kind": "repeater", "us": false,
      "heard_at": 1793499400, "heard_secs_ago": 600, "snr_db": 9.25 },
    { "prefix": "e5f60718", "key": "e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4",
      "name": "Base Station", "kind": "chat", "us": true,
      "heard_at": 1793499880, "heard_secs_ago": 120, "snr_db": 6.0 }
  ]
}
```

| Champ | Type | Signification |
|---|---|---|
| `asked_at` | integer ou null | Quand la table a été lue ; `null` = jamais (alors `total` vaut `null` et `items` vaut `[]`). |
| `total` | integer ou null | Combien de voisins il a (au plus 88 sont lus). |
| `items[].key`, `name`, `kind` | string ou null | Résolus à partir des noeuds et contacts que nous connaissons ; `null` si inconnu. |
| `items[].us` | bool | C'est notre compagnon. |
| `items[].heard_at` | integer | Quand le répéteur a entendu celui-ci pour la dernière fois. |

### POST /api/v1/nodes/{key}/neighbours

Demande maintenant au répéteur sa table de voisins et répond comme `GET`. Requêtes LoRa : une pour 11
entrées, plus une connexion invité quand il ne nous connaît pas ; peut prendre une minute. Redemander au
même noeud pendant qu'une requête est en route attend la même réponse (pas de second paquet).

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/neighbours
```

La réponse a la même forme que [`GET /api/v1/nodes/{key}/neighbours`](#get-apiv1nodeskeyneighbours).
Un événement `node` suit.

| Statut | Code | Quand |
|---|---|---|
| 404 | `not_contact` | Pas un contact du compagnon. |
| 503 | `unavailable` | Le compagnon n'est pas connecté, le stockage des messages est désactivé, ou le proxy est en cours d'arrêt. |
| 504 | `no_answer` | Hors de portée, ou la connexion a été refusée (un mot de passe peut être enregistré avec l'endpoint de connexion). |

### GET /api/v1/nodes/{key}/owner

La version de firmware, le nom et le texte du propriétaire (défini par son admin) qu'un noeud a donnés la dernière fois qu'on les lui a demandés.

```bash
curl -k https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/owner
```

```json
{ "asked_at": 1793000000, "firmware": "v1.9.1", "name": "Hilltop", "owner": "Mountain club" }
```

Tous les champs valent `null` si on ne les lui a jamais demandés.

### POST /api/v1/nodes/{key}/owner

Demande maintenant au noeud et répond comme `GET`. Les répéteurs répondent ; les serveurs de salon répondent
selon leur firmware. Redemander pendant qu'une requête est en route attend la même réponse.

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/owner
```

La réponse a la même forme que [`GET /api/v1/nodes/{key}/owner`](#get-apiv1nodeskeyowner). Un
événement `node` suit.

| Statut | Code | Quand |
|---|---|---|
| 404 | `not_contact` | Pas un contact du compagnon. |
| 503 | `unavailable` | Le compagnon n'est pas connecté, le stockage des messages est désactivé, ou le proxy est en cours d'arrêt. |
| 504 | `no_answer` | Hors de portée, ou la connexion a été refusée. |

### POST /api/v1/nodes/{key}/status

Demande maintenant son état au noeud (une requête LoRa, plus une connexion invité quand il ne nous connaît
pas ; peut prendre une minute) et répond comme [`GET /api/v1/nodes/{key}/status`](#get-apiv1nodeskeystatus),
avec cette seule réponse.

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/status
```

```json
{
  "items": [
    { "ts": 1793600700, "battery_mv": 4100, "noise_dbm": -110, "rssi_dbm": -92, "snr_db": 6.5,
      "tx_queue": 0, "recv": 51300, "sent": 20500, "recv_flood": 48100, "recv_direct": 3200,
      "sent_flood": 19020, "sent_direct": 1480, "airtime_tx_secs": 3605, "airtime_rx_secs": 7210,
      "dups_flood": 9010, "dups_direct": 120, "errors": 0, "recv_errors": 14, "uptime_secs": 864700 }
  ],
  "next": null
}
```

Un événement `node` suit.

| Statut | Code | Quand |
|---|---|---|
| 404 | `not_contact` | Pas un contact du compagnon. |
| 503 | `unavailable` | Le compagnon n'est pas connecté, le stockage des messages est désactivé, ou le proxy est en cours d'arrêt. |
| 504 | `no_answer` | Hors de portée, ou la connexion a été refusée. |

### POST /api/v1/nodes/{key}/telemetry

Demande maintenant sa télémétrie au noeud et répond comme
[`GET /api/v1/nodes/{key}/telemetry`](#get-apiv1nodeskeytelemetry), avec cette seule réponse. Un invité
n'obtient que la télémétrie de base : batterie et température du MCU.

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/a1b2c3d4/telemetry
```

```json
{
  "items": [
    { "ts": 1793600700, "channel": 1, "kind": "voltage", "value": 4.1 },
    { "ts": 1793600700, "channel": 1, "kind": "temperature", "value": 6.5 }
  ],
  "next": null
}
```

Un événement `node` suit. Mêmes erreurs que pour `POST /api/v1/nodes/{key}/status`.

### POST /api/v1/trace

Envoie une trace le long d'une liste de répéteurs (en direct, un paquet LoRa par saut) ; chacun note le SNR
auquel il l'a entendue.

| Champ | Type | Obligatoire | Signification |
|---|---|---|---|
| `path` | array de string hex | oui | Au moins un id de répéteur, tous de la même taille : 1 ou 2 octets chacun. |
| `round_trip` | bool | non (par défaut `true`) | Ajouter le retour (`a1 > b2 > a1`), pour que la réponse nous revienne. |

Pour un contact répéteur, utilisez son `out_path` plus son propre id ; pour un noeud de discussion, son
`out_path` (il ne répète pas). La réponse arrive une fois la trace revenue (jusqu'à environ une minute ; la
requête attend au plus 120 s).

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/trace \
  -H "Content-Type: application/json" -d '{"path": ["a1", "7c"], "round_trip": true}'
```

```json
{
  "path": ["a1", "7c", "a1"],
  "hops": [
    { "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
      "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "name": "Hilltop",
      "kind": "repeater", "lat": 12.3, "lon": 45.6, "confidence": 0.95, "choices": 1, "snr_db": 8.5 },
    { "hash": "7c", "id": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
      "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f", "name": "Valley Relay",
      "kind": "repeater", "lat": 12.4, "lon": 45.7, "confidence": 0.8, "choices": 2,
      "could_be": [
        { "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
          "name": "Valley Relay", "last_seen": 1793600000, "distance_km": 9.8 },
        { "key": "7c11aa22bb33cc44dd55ee66ff7788997c11aa22bb33cc44dd55ee66ff778899",
          "name": "Ridge Repeater", "last_seen": 1793400000, "distance_km": 31.5 }
      ],
      "snr_db": 4.25 },
    { "hash": "a1", "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "...": "...", "snr_db": 5.0 }
  ],
  "final_snr_db": 7.75
}
```

| Champ | Type | Signification |
|---|---|---|
| `path` | array | Le chemin envoyé (avec le retour). |
| `hops` | array | Chaque saut [résolu](#sauts-résolus) plus `snr_db` = comment ce saut l'a entendue. |
| `final_snr_db` | number | Comment nous avons entendu le dernier. |

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | Pas de chemin, chemin vide, pas de l'hexa, sauts de tailles différentes, ou pas 1 ou 2 octets. |
| 503 | `unavailable` | Le compagnon n'est pas connecté ou l'a refusée, ou le proxy est en cours d'arrêt. |
| 504 | `no_answer` | Un saut hors de portée, ou un mauvais chemin. |

### POST /api/v1/nodes/{key}/restore

Remet un contact archivé sur le compagnon (sa dernière trame de contact : nom, type, drapeaux, chemin ; le
compagnon le marque comme modifié à l'instant). Marche aussi pour un contact encore sur le compagnon (il
est mis à jour).

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/nodes/7c4d2e8f/restore
```

```json
{
  "ok": true,
  "replaced": { "key": "b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1", "name": "Old Rover" }
}
```

| Champ | Type | Signification |
|---|---|---|
| `replaced` | object ou null | Le contact que le compagnon a retiré pour faire de la place (liste pleine et remplacement du plus ancien activé) ; `name` peut être `null`. |

| Statut | Code | Quand |
|---|---|---|
| 404 | `not_archived` | Pas dans l'archive des contacts. |
| 409 | `contacts_full` | La liste est pleine et le remplacement du plus ancien est désactivé (ou il ne reste que des favoris). |
| 503 | `unavailable` | Le compagnon n'est pas connecté, a refusé, ou n'a pas répondu. |

### GET /api/v1/nodes/{key}/paths

Comment les chemins d'un noeud ont changé : le chemin que notre compagnon utilise pour le joindre, et les
chemins que ses annonces ont pris pour nous parvenir.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since` | integer | `hours=720` | Fenêtre ; `hours=0` = tout. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/nodes/7c4d2e8f/paths"
```

```json
{
  "node": { "id": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
            "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
            "name": "Valley Relay", "kind": "repeater", "lat": 12.4, "lon": 45.7 },
  "current": ["a1"],
  "outgoing": [
    { "at": 1793000000, "mode": "flood", "path": [], "hops": [] },
    { "at": 1793100000, "mode": "path", "path": ["a1"], "hops": [ { "hash": "a1", "name": "Hilltop", "...": "..." } ] }
  ],
  "incoming": [
    { "from": 1793000000, "to": 1793600000, "adverts": 12, "path": ["a1"],
      "hops": [ { "hash": "a1", "name": "Hilltop", "...": "..." } ], "snr_avg_db": 4.5 }
  ],
  "incoming_paths": [
    { "path": ["a1"], "hops": [ { "hash": "a1", "name": "Hilltop", "...": "..." } ], "adverts": 40, "last": 1793600000 }
  ]
}
```

| Champ | Type | Signification |
|---|---|---|
| `current` | array ou null | Le `out_path` que le compagnon a maintenant ; `null` = aucun (DM en flood) ou pas un contact. |
| `outgoing` | array | Chaque changement du chemin que notre compagnon utilise pour le joindre (enregistré depuis que cette fonction existe ; la première ligne est le chemin qu'il avait alors). `mode` : `"flood"`, `"direct"` ou `"path"`. `hops` dans l'ordre du chemin, de nous vers l'extérieur. |
| `incoming` | array | Périodes pendant lesquelles ses annonces nous sont parvenues par le même chemin (`hops` de lui vers nous). |
| `incoming_paths` | array | Les 10 chemins les plus utilisés. |

Les `hops` sont des [sauts résolus](#sauts-résolus).

## Régions

Un message flood peut être limité à une région (firmware v1.15 ou plus récent). Un nom de région fait 1 à 29
caractères parmi `a-z`, `0-9`, `-` (un `#` initial est accepté ; les majuscules passent en minuscules). Voir [Régions](#régions-portées-flood).

### GET /api/v1/regions

Tous les noms de région que le proxy connaît, et leur provenance.

```bash
curl -k https://192.168.1.50:5080/api/v1/regions
```

```json
{
  "items": [
    { "name": "eu", "source": "companion" },
    { "name": "eu-se", "source": "channel" },
    { "name": "hills", "source": "repeater",
      "served_by": [ { "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "name": "Hilltop" } ],
      "fetched_at": 1793500000 }
  ],
  "next": null,
  "fetching": false
}
```

| Champ | Type | Signification |
|---|---|---|
| `source` | string | `"companion"` (sa région par défaut), `"channel"` (la région propre à un canal), `"user"` (ajoutée à la main), `"repeater"` (un répéteur la dessert). Un nom connu de plusieurs sources garde la première. |
| `served_by`, `fetched_at` | array / integer | Seulement pour les noms donnés par des répéteurs : quels répéteurs, et quand ils ont été récupérés pour la dernière fois. |
| `fetching` | bool | Une récupération est en cours. |

| Statut | Code | Quand |
|---|---|---|
| 503 | `unavailable` | Le proxy est en cours d'arrêt. |

### PUT /api/v1/regions

Remplace les noms de région ajoutés à la main ; répond avec la liste complète comme le fait `GET`.

| Champ | Type | Obligatoire | Signification |
|---|---|---|---|
| `items` | array | oui | Au plus 200 entrées, chacune `{"name": "..."}` (une simple chaîne est acceptée aussi). Les doublons sont retirés. |

```bash
curl -k -X PUT https://192.168.1.50:5080/api/v1/regions \
  -H "Content-Type: application/json" -d '{"items": [{"name": "eu"}, {"name": "eu-se"}]}'
```

La réponse a la même forme que [`GET /api/v1/regions`](#get-apiv1regions).

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | Pas de `items`, plus de 200, ou un nom incorrect. |
| 503 | `no_store` | Le stockage des messages est désactivé. |
| 503 | `unavailable` | L'enregistrement a échoué, ou pas de réponse à temps. |

### POST /api/v1/regions/fetch

Demande leurs régions aux répéteurs à portée directe. Réponse immédiate ; des événements `regions` suivent, le
dernier avec `"state": "done"` (relisez alors `GET /api/v1/regions`). Seuls les répéteurs enregistrés avec une
route zéro saut sont interrogés (les routes ne sont jamais modifiées) ; les autres sont listés comme `not_asked`.

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/regions/fetch
```

```json
{ "ok": true, "state": "discovering", "wait_secs": 10 }
```

| Statut | Code | Quand |
|---|---|---|
| 409 | `busy` | Une récupération est en cours. |
| 409 | `unsupported` | Le firmware est trop ancien pour découvrir les répéteurs. |
| 503 | `radio_down` | Le compagnon n'est pas connecté. |
| 503 | `no_store` | Le stockage des messages est désactivé. |

## Réglages

### GET /api/v1/settings

Les réglages du proxy qu'un client peut afficher et modifier.

```bash
curl -k https://192.168.1.50:5080/api/v1/settings
```

```json
{
  "hide_own_text": true,
  "mirror_sent_channel": true,
  "mirror_sent_dm": false,
  "mirror_available": true
}
```

| Champ | Type | Par défaut | Signification |
|---|---|---|---|
| `hide_own_text` | bool | `true` | Garder le texte des messages envoyés par notre noeud hors du log du proxy (fichiers de log, journal, `GET /api/v1/logs`, événements `log` : seulement sa taille, pas d'hexadécimal). La vue en direct (événements `activity`, application de terminal, `mcproxy watch`), `GET /api/v1/messages` et les événements `message` l'affichent, et le stockage le garde. Affiché comme un seul interrupteur, par exemple "Masquer mes messages dans le log du proxy". Sur la machine du proxy : `mcproxy text own show / hide` ; anciens fichiers de log : `mcproxy logs --purge-own`. |
| `mirror_sent_channel`, `mirror_sent_dm` | bool | `true` / `false` | Copies de ce qu'un appareil envoie, pour les autres appareils sur le port 5000. Voir [Stockage et copies](#stockage-et-copies). Sur la machine du proxy : `mcproxy mirror channel / dm on / off`. |
| `mirror_available` | bool | | Lecture seule. `false` quand le stockage des messages est désactivé : les copies sont impossibles ; afficher les deux interrupteurs comme désactivés. |

### PATCH /api/v1/settings

Modifie les réglages : envoyez les champs à modifier. Appliqué aussitôt (aussi dans l'application de terminal et le log)
et enregistré dans la configuration du proxy ; répond avec le nouvel objet des réglages. Rien ne change
quand un champ est incorrect.

```bash
curl -k -X PATCH https://192.168.1.50:5080/api/v1/settings \
  -H "Content-Type: application/json" -d '{"mirror_sent_dm": true}'
```

```json
{
  "hide_own_text": true,
  "mirror_sent_channel": true,
  "mirror_sent_dm": true,
  "mirror_available": true
}
```

Seuls `hide_own_text`, `mirror_sent_channel` et `mirror_sent_dm` peuvent être modifiés. Toute personne autorisée à
joindre l'API peut les modifier : les mêmes personnes peuvent déjà lire chaque message par le port des applications
(5000), donc cela n'expose rien de plus. La liste d'adresses autorisées et le jeton restreignent les deux.

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | Un champ inconnu, ou une valeur qui n'est pas un bool. |
| 500 | `internal` | La configuration n'a pas pu être enregistrée. |

## Logs

### GET /api/v1/logs

Le log du service, du plus ancien au plus récent.

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `lines` | integer | 500 | 1 à 10000. |
| `filter` | string | `all` | `all`, `apps`, `radio` ou `problems`. |
| `grep` | string | aucun | Seulement les lignes qui contiennent ce texte. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/logs?lines=2&filter=problems"
```

```json
{
  "items": [
    { "ts": 1793600000, "level": "WARN", "text": "radio: link lost, reconnecting" },
    { "ts": 1793600012, "level": "INFO", "text": "radio: connected" }
  ],
  "next": null
}
```

| Champ | Type | Signification |
|---|---|---|
| `ts` | integer ou null | Heure de la ligne ; `null` (avec `level` à `null`) pour une ligne qui n'est pas une entrée de log. |
| `level` | string ou null | `"INFO"`, `"WARN"`, `"ERROR"`, ... |
| `text` | string | Le message. |

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | `filter` inconnu. |

## Compagnon

Ce sont les réglages propres au compagnon : voir [C'est le compagnon qui décide](#cest-le-compagnon-qui-décide).

### PATCH /api/v1/companion/settings

Modifie quels contacts le compagnon ajoute de lui-même, et sa taille des ids de chemin. Écrit dans le compagnon
(enregistré dedans, comme le fait l'application), puis relu dans `GET /api/v1/status`. Un ou plusieurs de ces champs :

| Champ | Type | Signification |
|---|---|---|
| `add` | string | `"all"` ou `"selected"`. |
| `chat`, `repeater`, `room`, `sensor` | bool | Avec `selected` : quels types sont ajoutés. |
| `overwrite_oldest` | bool | Quand la liste est pleine, remplacer le plus ancien contact non favori. Aussi `mcproxy contacts --overwrite-oldest on / off`. |
| `max_hops` | integer | 0 à 64 : 0 = toute distance, 1 = entendus directement seulement, N = jusqu'à N-1 sauts. |
| `path_id_bytes` | integer | 1 à 3 : la taille des ids de chemin avec laquelle notre compagnon envoie (son mode de hash de chemin). |

Voir [Réglages d'ajout automatique](#réglages-dajout-automatique).

```bash
curl -k -X PATCH https://192.168.1.50:5080/api/v1/companion/settings \
  -H "Content-Type: application/json" -d '{"add": "selected", "repeater": true, "max_hops": 3}'
```

```json
{ "ok": true }
```

Un événement `status` suit une fois les réglages relus.

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | Un autre champ, une mauvaise valeur. |
| 409 | `unsupported` | Firmware sans ces réglages. |
| 503 | `unavailable` | Le compagnon n'est pas connecté, a refusé ou n'a pas répondu. |

### POST /api/v1/companion/advert

Notre compagnon envoie maintenant sa propre annonce, comme les boutons Advert de l'application MeshCore. Attend son tour comme
toute commande (voir [Envoyer sans saturer](#envoyer-sans-saturer)).

| Champ | Type | Obligatoire | Signification |
|---|---|---|---|
| `flood` | bool | oui | `true` = par les répéteurs ; `false` = zéro saut (seuls les noeuds à portée l'entendent). |

```bash
curl -k -X POST https://192.168.1.50:5080/api/v1/companion/advert \
  -H "Content-Type: application/json" -d '{"flood": true}'
```

```json
{ "ok": true }
```

Répondu une fois que le compagnon l'a envoyée.

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | Pas de `flood`, ou pas un bool. |
| 503 | `unavailable` | Le compagnon n'est pas connecté, a refusé ou n'a pas répondu à temps. |

### PATCH /api/v1/companion/region

Définit la [région](#régions-portées-flood) par défaut du compagnon, enregistrée sur le compagnon.

| Champ | Type | Obligatoire | Signification |
|---|---|---|---|
| `region` | string ou null | oui | Un nom de région, ou `null` = aucune. |

```bash
curl -k -X PATCH https://192.168.1.50:5080/api/v1/companion/region \
  -H "Content-Type: application/json" -d '{"region": "eu"}'
```

```json
{ "ok": true, "region": "eu" }
```

`region` est ce que le compagnon indique à la relecture.

| Statut | Code | Quand |
|---|---|---|
| 400 | `bad_request` | Mauvais nom de région. |
| 409 | `unsupported` | Firmware sans régions (v1.15 ou plus récent nécessaire), ou le compagnon n'a pas indiqué sa région. |
| 503 | `radio_down` | Le compagnon n'est pas connecté. |
| 503 | `unavailable` | Le compagnon l'a refusée ou n'a pas répondu. |

### GET /api/v1/companion/history

Notre propre compagnon au fil du temps, du plus ancien au plus récent (une ligne toutes les 15 minutes).

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `hours`, `since`, `until` | integer | `hours=24` | Plage de temps. |
| `limit`, `cursor` | | 500 | Pagination. |

```bash
curl -k "https://192.168.1.50:5080/api/v1/companion/history?hours=1"
```

```json
{
  "items": [
    { "ts": 1793597700, "battery_mv": 4100, "temperature_c": 6.5, "noise_dbm": -112,
      "recv": 10240, "sent": 812, "lat": 12.31, "lon": 45.62 }
  ],
  "next": null
}
```

| Champ | Type | Signification |
|---|---|---|
| `recv`, `sent` | integer ou null | Compteurs de paquets depuis le démarrage du compagnon (une baisse = un redémarrage). |
| `noise_dbm` | integer ou null | Le bruit du compagnon. |
| `lat`, `lon` | number ou null | Sa position telle qu'elle y est réglée. |

---

## Événements en direct

### GET /api/v1/events

Un flux Server-Sent Events (`text/event-stream` ; dans les navigateurs `new EventSource(url)`).

| Paramètre | Type | Par défaut | Signification |
|---|---|---|---|
| `types` | string | tous | Noms des événements à recevoir, séparés par des virgules, par exemple `packet,status`. `lagged` est envoyé quel que soit `types`. |
| `token` | string | aucun | Le jeton de l'API, pour les clients qui ne peuvent pas définir d'en-tête (peut être encodé en pourcentage). |

```bash
curl -k -N "https://192.168.1.50:5080/api/v1/events?types=node,status"
```

```
event: node
id: 7301
data: {"key":"a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90"}

event: status
id: 7302
data: {}

: keep-alive
```

**Règles du flux**

- Chaque événement a un nom (`event:`), une ligne JSON `data:` et un `id:` croissant.
- Un commentaire de maintien est envoyé toutes les 15 s.
- Les clients se reconnectent d'eux-mêmes (les navigateurs le font). Le flux ne rejoue **pas** les événements manqués : un
  en-tête `Last-Event-ID` est accepté (CORS l'autorise) mais ignoré. Après une reconnexion, relisez les
  endpoints REST pour ce que vous affichez.
- Un client qui lit trop lentement reçoit un événement `lagged` avec le nombre d'événements qui lui ont échappé
  (le proxy garde environ 2048 événements en tampon) : relisez les endpoints REST, comme après une reconnexion. Un client
  qui ne connaît pas `lagged` peut l'ignorer.
- Les événements que personne n'écoute ne sont pas construits : un client ne voit que ce qui s'est passé pendant qu'il était connecté.

| Événement | Quand | Contenu |
|---|---|---|
| `packet` | Un paquet a été entendu. | Même objet qu'un élément de `GET /api/v1/mesh/packets`, `path_nodes` compris (un noeud entendu pour la première fois est résolu en moins d'une minute). |
| `activity` | Une ligne lisible de ce qui passe (ce que montre `mcproxy watch` ; propre texte masqué selon la configuration). | `{"ts", "at", "kind", "text"}` |
| `status` | L'état en direct a changé. | `{}` : relisez `GET /api/v1/status`. |
| `node` | Nouvelles données pour ce noeud (annonce, état, télémétrie, voisins, propriétaire, connexion), ou il a été ajouté à la liste de contacts du compagnon, y a été modifié (favori, chemin, nom) ou en a été retiré (par n'importe quelle application, par le compagnon lui-même ou par une restauration), ou sa marque "mine" ou son mot de passe enregistré a changé. Suivi de `status` quand le nombre de contacts a changé. | `{"key"}` |
| `contacts` | Plus de 50 contacts ont changé d'un coup (par exemple un autre compagnon connecté, une restauration). Suivi de `status`. | `{"changed": n}` : relisez `GET /api/v1/nodes` au lieu de lire noeud par noeud. |
| `message` | Un message est arrivé ou a été envoyé. | Un élément de `GET /api/v1/messages`, voir ci-dessous. |
| `message_status` | La remise d'un de nos messages a changé (état, essai, répétitions). | `{"id", "client_id", "delivery"}` |
| `messages_deleted` | Une conversation DM ou un message a été supprimé du stockage. | `{"with", "id", "deleted"}` : retirez-les de ce que vous affichez. |
| `channel_messages_deleted` | Un canal a été retiré avec ses messages. | `{"id", "slot", "deleted"}` : retirez les messages de ce canal (par `id`). |
| `log` | Une ligne a été écrite dans le log du service (tous les niveaux, comme `filter=all`). | Même objet qu'un élément de `GET /api/v1/logs`. |
| `reset` | `mcproxy reset` a effacé des parties des données stockées. | `{"parts": [...]}` (`"messages"`, `"mesh"`, `"contacts"`, `"history"`) : retirez ce que vous en affichez et relisez. Aussi envoyé avec toutes les parties quand un autre compagnon se connecte. |
| `cli` | Une commande CLI de noeud envoyée d'ici est terminée, quel que soit celui qui l'a envoyée. | `{"id", "node", "name", "command", "reply", "state"}` (comme dans `GET /api/v1/nodes/{key}/cli`, secrets masqués). |
| `channels` | Un canal a été défini sur le compagnon (ajouté, modifié ou retiré, par n'importe quelle application ou par l'API), ou la région propre d'un canal a changé. | `{"slot"}` : relisez `GET /api/v1/channels`. Ou `{"channel_id", "region"}` : définie par l'API ou la console, ou apprise d'une application MeshCore qui envoie dans ce canal avec sa propre région (une région par canal pour tous, la dernière définie l'emporte). |
| `regions` | Les étapes de `POST /api/v1/regions/fetch`. | `{"state": "discovering", "wait_secs"}`, `{"state": "asking"}`, puis `{"state": "done", ...}` ; après `done`, relisez `GET /api/v1/regions`. |
| `lagged` | Ce client a lu le flux trop lentement. | `{"missed": n}` |

**L'événement `message`.** Le même objet qu'un élément de `GET /api/v1/messages`, avec ces différences :
`ts` est le moment où le proxy l'a reçu ; `delivery` est encore `null` (des événements `message_status` suivent pour les nôtres) ;
`path` et `rssi_dbm` sont `null` (le paquet est associé juste après : relisez le message, ou
`GET /api/v1/messages/{id}/heard`) ; `text_state`, `data`, `heard_count`, `echo_of`, `path_id_bytes` et
`path_nodes` ne sont pas inclus. Pour un message envoyé par `POST /api/v1/messages`, il porte
`client_id`, et `id` est `null` quand aucune copie n'est stockée.

**Exemples de contenu**

`packet`
```json
{
  "id": 98003, "ts": 1793600420, "snr_db": 5.75, "rssi_dbm": -99, "route": "flood", "type": "advert",
  "version": 0, "hops": 1, "hash_size": 1, "path": ["a1"],
  "path_nodes": [ { "hash": "a1", "name": "Hilltop", "kind": "repeater", "confidence": 0.95, "choices": 1, "...": "..." } ],
  "last_hop": "a1", "size_bytes": 110,
  "node_key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
  "dst": null, "src": null, "channel_hash": null, "ack": null, "scoped": false, "region": null
}
```

`activity` (`at` = heure locale sur le proxy ; `kind` est `"app"`, `"radio"`, `"store"`,
`"frame"` ou `"event"`)
```json
{ "ts": 1793600420, "at": "14:20:20.512", "kind": "radio", "text": "radio -> all  ADVERT Hilltop" }
```

`status`
```json
{}
```

`node`
```json
{ "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90" }
```

`contacts`
```json
{ "changed": 120 }
```

`message`
```json
{
  "id": 1242, "ts": 1793600430, "kind": "channel", "channel": "#hills", "channel_slot": 1,
  "channel_id": "d41e8a2c6b9f0357", "channel_kind": "hashtag", "from": "Hilltop", "from_key": null,
  "to": null, "to_key": null, "with": null, "outgoing": false, "text": "Anyone on?", "hops": 1, "snr_db": 6.0, "rssi_dbm": null,
  "path": null, "client_id": null, "delivery": null, "scoped": false, "region": null, "cli": false
}
```

`message_status`
```json
{
  "id": 1241,
  "client_id": "b7e2c9d4-1f3a-4c5e-8d7b-2a4c6e8f0a13",
  "delivery": { "state": "sent", "route": "flood", "timeout_ms": 9800, "rtt_ms": null, "at": null,
                "repeats": 0, "attempt": 1, "attempts_max": 3 }
}
```

`messages_deleted` (`with` = la clé ou son début tel que donné, en minuscules ; ou `id` pour un seul message)
```json
{ "with": "7c4d2e8f", "id": null, "deleted": 42 }
```

`channel_messages_deleted`
```json
{ "id": "d41e8a2c6b9f0357", "slot": 1, "deleted": 95 }
```

`log`
```json
{ "ts": 1793600440, "level": "INFO", "text": "api: message b7e2c9d4-1f3a-4c5e-8d7b-2a4c6e8f0a13 sent" }
```

`reset`
```json
{ "parts": ["messages", "mesh", "contacts", "history"] }
```

`cli`
```json
{
  "id": 57,
  "node": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "name": "Hilltop",
  "command": "ver",
  "reply": "v1.9.1 (Build: 01-Jan-2026)",
  "state": "ok"
}
```

`channels` (un emplacement a changé)
```json
{ "slot": 1 }
```

`channels` (la région propre d'un canal a changé)
```json
{ "channel_id": "d41e8a2c6b9f0357", "region": "eu-se" }
```

`regions` (début, puis demande)
```json
{ "state": "discovering", "wait_secs": 10 }
```
```json
{ "state": "asking" }
```

`regions` (fin)
```json
{
  "state": "done",
  "error": null,
  "found": ["eu", "hills"],
  "repeaters": [
    { "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "name": "Hilltop",
      "regions": ["eu", "hills"] },
    { "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f", "name": "Valley Relay",
      "error": "no answer" }
  ],
  "not_asked": [
    { "key": "c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2", "name": "Ridge Repeater",
      "reason": "route", "error": "its route on the companion is not zero hop (the proxy does not change it)" },
    { "key": "5d6e7f80", "name": null, "reason": "not_contact", "error": "not a contact of the companion" }
  ]
}
```

Dans `done` : `error` est renseigné quand la récupération s'est terminée trop tôt (le compagnon s'est déconnecté, ou a refusé la
découverte) ; `repeaters[]` a `regions` pour un répéteur qui a répondu ou `error` pour un répéteur qui n'a pas répondu ;
`not_asked[].reason` est `"route"` (sa route sur le compagnon n'est pas à zéro saut) ou `"not_contact"`
(alors `key` est l'id court donné par la découverte et `name` est `null`).

`lagged`
```json
{ "missed": 312 }
```

---

## Notions

### Sauts résolus

Un chemin désigne un répéteur par les 1-3 premiers octets de sa clé, donc un id peut correspondre à plusieurs répéteurs.
Le proxy résout chaque id d'après le contexte (tables de voisins, qui nous entendons directement, distance, entendu
depuis combien de temps) et fournit le résultat avec ses données brutes. Chaque `via`, `path_nodes`,
`out_path_nodes`, saut de trace et saut de chemin de cette API est un objet de ce type :

```json
{
  "hash": "7c",
  "id": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
  "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f",
  "name": "Valley Relay",
  "kind": "repeater",
  "lat": 12.4,
  "lon": 45.7,
  "confidence": 0.8,
  "choices": 2,
  "could_be": [
    { "key": "7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f7c4d2e8f1a3b5c6d7e8f9a0b1c2d3e4f", "name": "Valley Relay",
      "last_seen": 1793600000, "distance_km": 9.8 },
    { "key": "7c11aa22bb33cc44dd55ee66ff7788997c11aa22bb33cc44dd55ee66ff778899", "name": "Ridge Repeater",
      "last_seen": 1793400000, "distance_km": 31.5 }
  ]
}
```

| Champ | Type | Signification |
|---|---|---|
| `hash` | string | L'id tel qu'il était dans le paquet. |
| `id` | string | La clé du noeud, ou `"h:<hex>"` quand aucun répéteur entendu n'a cet id (`"self"` pour notre compagnon dans les vues du mesh). |
| `key`, `name` | string ou null | `null` pour un id `"h:"`. |
| `kind` | string | Type de noeud (`"unknown"` pour un id `"h:"`). |
| `lat`, `lon` | number ou null | Position. |
| `confidence` | number | 0..1. 0 = aucun répéteur entendu n'a cet id. Une seule correspondance sur 1 octet vaut 0.9 (ce peut être un répéteur que nous n'avons jamais entendu). |
| `choices` | integer | Combien de répéteurs / salons entendus partagent l'id. |
| `could_be` | array | Seulement quand `choices` > 1 : chaque candidat, avec `last_seen` et `distance_km` (ou `null`). |

La façon de présenter un saut (sûr / probable / supposé) est le choix du client.

### Taille des ids de chemin

Tous les ids de saut dans le chemin d'un paquet ont la même taille, de 1 à 3 octets (le mode de hash de chemin du firmware).
**La taille est propre à chaque paquet** : le noeud qui envoie un paquet la choisit, et chaque répéteur ajoute son id à
cette taille (MeshCore `sendFlood` / `routeRecvPacket`). Les compagnons utilisent 1 octet par défaut, donc dans un mesh dont
les répéteurs utilisent 3 octets, les paquets des compagnons laissés par défaut portent quand même des ids de 1 octet. Les chemins de
cette API sont montrés exactement tels qu'ils sont arrivés par radio.

- `/status` `radio.path_id_bytes` : la taille avec laquelle notre compagnon envoie (réglée avec
  `PATCH /api/v1/companion/settings`).
- Le `path_id_bytes` d'un noeud (+ `path_id_bytes_at`) : la taille des paquets qu'**il** envoie, lue dans
  sa dernière annonce **flood** (l'expéditeur choisit la taille et les répéteurs la gardent). Les annonces à zéro saut ne
  l'indiquent pas (envoyées en direct avec une longueur de chemin de 0, ce qui se lit comme 1 octet), donc un noeud entendu seulement à zéro saut
  reste `null`. Elle ne dit rien de l'aspect des paquets des autres quand il les relaie : il ajoute son
  id à leur taille.
- Le `path_id_bytes` d'un message : la taille que son paquet avait vraiment.
- Le `hash_size` d'un paquet.

### Chemins des messages

`path`, `path_nodes` et `rssi_dbm` d'un message reçu sont **le chemin de l'expéditeur** (ids comme dans le
paquet, et chacun résolu, avec l'expéditeur comme contexte quand il est connu) et le signal, d'après la première
copie de son paquet entendue par le proxy. Ils sont gardés par le proxy qu'une application ait été connectée ou non,
donc un client qui se reconnecte plus tard les obtient quand même. Ils sont `null` quand le paquet n'a pas été associé,
et pour les messages qui ont attendu dans la radio pendant que le proxy lui-même n'était pas connecté : la file hors
ligne du compagnon ne garde que le nombre de sauts (firmware), donc ce chemin est perdu partout. Sur nos propres
messages ils sont `null` : voir `delivery`, et `GET /api/v1/messages/{id}/heard` pour leurs répétitions.

### SNR et RSSI

`avg_rssi_dbm` / `avg_snr_db` sont de simples moyennes sur les paquets. Leur somme ne donne pas le
bruit : le SNR LoRa sature vers +10 dB sur les paquets forts alors que le RSSI continue de monter, et sous le
bruit le RSSI reste proche du bruit alors que le SNR devient négatif.

### Séries horaires complètes

Chaque liste `per_hour` est **complète** : une ligne pour chaque heure de la plage, de la plus ancienne à la plus récente, `ts` = le
début de l'heure sur une limite d'heure UTC (`ts % 3600 == 0`). Une heure sans rien a des compteurs
à zéro et des moyennes `null` (dans `mesh/summary` : `{"ts", "packets": 0, "messages": 0, "avg_snr_db":
null, "by_type": {}}`). Avec `hours=0`, la plage commence aux premières données stockées. Les listes `per_day`
sont complètes de la même façon, `ts` = minuit UTC. Une série n'a jamais plus de 20000 lignes.

### Ids et emplacements des canaux

**Les canaux sont connus par leur id, pas par leur emplacement.** Un canal *est* sa clé secrète (celle de Public est
fixe, celle d'un canal `#name` vient de son nom, celle d'un canal privé est aléatoire). `channel_id` = les 8 premiers
octets du SHA-256 de cette clé, en hexadécimal (16 caractères) : le même sur chaque radio qui a le
canal, dans n'importe quel emplacement ; à sens unique, donc il ne révèle rien de la clé (ses 2 premiers caractères sont le
hash de canal dans les paquets).

- Chaque message de canal est stocké avec l'id de son canal, donc il garde son canal quand les canaux sont
  réordonnés ou qu'un autre compagnon est connecté.
- Un message dont le canal ne peut pas être déterminé (stocké par une ancienne version sans son id) a
  `channel_id: null` et n'appartient à aucun canal : jamais deviné d'après son emplacement, qui peut contenir un autre
  canal maintenant. Il n'est pas dans `GET /api/v1/channels` et aucun `?channel=` ne le trouve.
- `channel` (un nom) vient de l'id, quel que soit l'emplacement actuel du canal ; `channel_slot` est
  l'emplacement qu'il a maintenant. Regroupez une conversation par `channel_id`.
- `?channel=` prend un id de canal, sinon un nom (tous les canaux de ce nom), sinon un numéro d'emplacement (le
  canal qui s'y trouve maintenant). `POST /api/v1/messages` les prend dans le même ordre : envoyez l'id.
- **Un envoi va dans son canal ou nulle part.** Il garde l'id du canal pendant qu'il attend, et son emplacement est
  recherché, puis lu dans le compagnon, juste avant son départ. Un canal qui a disparu d'ici là
  donne `not_found`.
- Les applications MeshCore sur le port 5000 ne connaissent que les emplacements. Le proxy garde la liste que chaque application a lue, et
  envoie et transmet les messages selon le canal que cette application désigne, même après que des canaux ont été déplacés par
  une autre application. Les messages d'un canal qu'une application n'a pas encore sont gardés pour elle jusqu'à ce qu'elle relise ses
  canaux ; tout le reste continue entre-temps. Un canal retiré puis ajouté de nouveau avec la même
  clé est le même canal, dans n'importe quel emplacement : son historique et ce qui a été gardé lui appartiennent de nouveau. Un message dont le canal ne peut pas être déterminé est stocké
  (`channel_id: null`) et n'est donné à aucune application.

### Envoyer sans doublons

`POST /api/v1/messages` est conçu pour qu'un message parte au plus une fois :

- Le client crée un `id` par message que l'utilisateur envoie, jamais par essai HTTP.
- Le même `id` de nouveau : pendant l'envoi, la requête attend le même résultat ; une fois **sent**
  ou **unknown**, elle reçoit ce résultat (pendant 30 minutes, et tant que sa copie stockée
  existe) et rien n'est envoyé une deuxième fois. Après **not_sent**, le même `id` peut être envoyé de nouveau.
- Une requête = un message. Un message de canal est une seule commande au compagnon. Un DM n'est renvoyé que
  tant qu'aucun ACK n'est arrivé, comme le font les applications MeshCore (voir [États de remise](#états-de-remise-et-nouveaux-essais-des-dm)) ;
  la réponse HTTP reflète le premier essai.
- `unknown` signifie qu'il est peut-être parti : ne renvoyez jamais automatiquement ; demandez à l'utilisateur, puis renvoyez-le
  avec `resend_of` (le même message, un nouvel essai) et un nouvel `id`.

### États de remise et nouveaux essais des DM

`delivery` sur nos propres messages (et dans les événements `message_status`) :

```json
{ "state": "delivered", "route": "direct", "timeout_ms": 4200, "rtt_ms": 2650, "at": 1793600103,
  "repeats": 0, "attempt": 1, "attempts_max": 3, "path": ["a1", "7c"] }
```

| Champ | Type | Signification |
|---|---|---|
| `state` | string | `"sending"`, `"sent"`, `"delivered"`, `"failed"` ou `"unknown"` (il a été confié au compagnon, qui n'a pas répondu : il est peut-être parti). Il devient `"sent"` ou `"delivered"` si l'essai suivant d'une application, ou une répétition entendue, le montre ; il peut être renvoyé comme un message en échec. |
| `route` | string ou null | `"flood"` ou `"direct"`, tel que le compagnon l'a envoyé. |
| `path` | array ou null | DM : les ids du chemin par lequel l'essai est passé (celui qui a été remis, sinon le dernier), premier saut en premier ; `[]` = direct sans répéteur ; `null` = en flood, inconnu, ou message de canal. Par contact et par route au fil du temps : `GET /api/v1/mesh/delivery`. |
| `timeout_ms` | integer ou null | L'estimation par le compagnon du délai de l'ACK (DM). |
| `rtt_ms`, `at` | integer ou null | Temps aller-retour et heure de l'ACK (DM). |
| `repeats` | integer | Messages de canal : répétitions entendues. |
| `attempt`, `attempts_max` | integer | Quel essai, sur combien. |

**DM.** `sent` une fois que le compagnon l'a envoyé (`route`, `timeout_ms`), `delivered` quand l'ACK du
destinataire est revenu (`rtt_ms`, `at`), `failed` quand aucun ACK n'est arrivé à temps (l'estimation + 20 %). Un ACK tardif
le fait quand même passer à `delivered`. Les DM envoyés par `POST /api/v1/messages` sont renvoyés comme le font les
applications MeshCore : jusqu'à `attempts_max` = 3 essais, `sending` pendant qu'un renvoi est en route, le même
horodatage et le même texte à chaque fois. Le firmware transmet chaque essai qui atteint le destinataire à son
application, donc il ne voit un seul message que si cette application compare l'horodatage et le texte (ce proxy le fait). Avant
le dernier essai, le compagnon oublie la route, donc il part en flood. L'attente de l'ACK d'un essai
commence quand il quitte la file ([envoyer sans saturer](#envoyer-sans-saturer) peut le retenir), et un ACK annule
un essai qui y attend encore. Un essai refusé parce que la table de paquets du compagnon est pleine est
suivi du suivant après une pause, au lieu d'échouer. Une application qui renvoie son propre DM apparaît comme
l'`attempt` suivant du même message.

**Messages de canal.** Il n'existe pas d'ACK : `sent`, et `repeats` = combien de fois nous avons entendu des répéteurs le
relayer dans les 30 premières s (au mieux : associé par canal et taille de paquet, puis par contenu). `repeats: 0`
un moment après l'envoi = aucun répéteur n'a été entendu en train de le relayer : il peut être renvoyé avec `resend_of`
(`sending`, puis `sent` de nouveau, `attempt` continuant de compter et `repeats` compté à partir du renvoi).

Un message envoyé est suivi pendant 15 minutes après son dernier changement. Au démarrage (et quand les
données d'un compagnon reviennent), les DM restés à `sent` ou `sending` passent à `failed` : plus rien ne les
suit.

### Renvoyer

`POST /api/v1/messages` avec `resend_of` renvoie un message stocké. C'est l'essai suivant de ce
message, pas un nouveau message : rien de nouveau n'est stocké, aucun événement `message` ne suit, et
`GET /api/v1/messages` garde une seule ligne. `delivery` passe à `sending`, puis aux états habituels par
`message_status` sur l'id stocké, `attempt` et `attempts_max` continuant de compter. `message_id` dans la
réponse est cet id stocké. Chaque tentative demande un nouvel `id`.

**Un DM** part avec le même horodatage et le même texte et un nouveau numéro d'essai, donc une application qui les
compare affiche un seul message si elle l'a déjà (un ACK perdu au retour est courant) ; les autres affichent une
répétition. Il part une seule fois, jamais réessayé de lui-même, quel que soit son âge. `route` fonctionne comme pour un nouveau message.

**Un message de canal** peut être renvoyé tant qu'aucun répéteur n'a été entendu en train de le relayer (`repeats: 0`) et pendant
10 minutes après son envoi ; `route` doit être `auto`. Il part comme exactement le même paquet (mêmes
canal, horodatage et texte ; le compagnon ajoute le même nom) : les noeuds qui l'ont déjà entendu l'écartent
comme un paquet qu'ils ont transporté, donc personne ne l'affiche deux fois, et ceux qui l'ont manqué le reçoivent maintenant. Les noeuds
se souviennent des 160 derniers paquets qu'ils ont vus : sur un mesh très chargé, même 10 minutes peuvent être trop longues pour certains
d'entre eux.

| Raison | Statut | Signification |
|---|---|---|
| `not_found` | 404 | Aucun message de ce type envoyé d'ici ; ou (canal) le canal n'est plus sur le compagnon. |
| `not_contact` | 404 | (DM) Le destinataire n'est plus un contact du compagnon. |
| `not_dm` | 409 | Ni un DM ni un message de canal. |
| `not_failed` | 409 | (DM) Pas en échec : `delivered`, ou encore suivi. (Canal) Un répéteur a été entendu en train de le relayer, ou il est en cours d'envoi. |
| `expired` | 409 | Son texte a disparu, ou la copie stockée est peut-être tronquée (un texte proche de la limite de 160 octets précédé du préfixe de copie de DM) : envoyez-le comme un nouveau message. |
| `too_old` | 409 | (Canal) Envoyé il y a plus de 10 minutes : les noeuds ne s'en souviennent plus, donc un renvoi pourrait s'afficher deux fois ; envoyez un nouveau message. |
| `no_store` | 503 | Le stockage des messages est désactivé. |

### Nouveaux essais si occupé

Le compagnon refuse un message tant que sa file de paquets est pleine (16 paquets ; un message de canal reçoit alors
la réponse "not found" du firmware, un DM "table full"). Rien n'est parti, donc le proxy remet la
même commande en file après 2, 4 et 6 s avant de répondre `not_sent` avec `reason: "busy"`. La réponse HTTP
de `POST /api/v1/messages` peut donc prendre jusqu'à 12 s de plus.

### Envoyer sans saturer

L'envoi sans saturer (`[polite]` dans la configuration, activé par défaut ; `mcproxy polite on / off`) s'applique à toutes les
applications, à la console et à cette API ensemble :

- Les commandes qui passent par radio (messages, annonces, demandes aux répéteurs) partent aussitôt jusqu'à `burst`,
  puis une toutes les `gap_secs` ; entre-temps les autres commandes ne sont pas retenues.
- Au plus `acks_open_max` DM attendent leur ACK en même temps : le compagnon ne reconnaît que les ACK de ses
  8 derniers DM, donc un de plus lui ferait manquer celui du plus ancien.
- Les DM que le proxy renvoie font une pause de 1, 2, 4 s plus une part aléatoire entre les essais.
- Rien n'est refusé pour cela : une requête met seulement plus de temps à recevoir sa réponse (`POST /api/v1/messages` et
  `POST /api/v1/companion/advert` attendent).
- Une application sur le port 5000 qui envoie exactement le même message deux fois en moins d'une minute (mêmes chat, texte,
  horodatage et essai) reçoit la réponse du compagnon au premier, et il part une seule fois ; un renvoi
  manuel ou un nouvel essai de DM est un nouvel envoi et part toujours. Par cette API, le même `id` n'envoie déjà
  jamais deux fois.

`GET /api/v1/status` `polite` montre les réglages et ce qui attend maintenant.

### Un message, une ligne

Un message reçu est connu par son propre id : son canal (ou son expéditeur), l'horodatage de son expéditeur, son
type et son texte. Une copie d'un message déjà stocké, quel que soit le moment où elle arrive (une répétition tardive, un DM
que son expéditeur a renvoyé, notre propre message de canal qui revient), n'est pas stockée de nouveau : pas de deuxième ligne, pas
d'événement `message`, rien pour les applications sur le port 5000. Tous les clients voient le même historique. `heard_count`
compte quand même les copies entendues. Les messages arrivés deux fois avant cet ajout gardent les deux lignes.

### Remise au moins une fois et ACK gardés

Les applications sur le port MeshCore (5000) reçoivent chaque message stocké au moins une fois : un message compte comme
remis à une application seulement quand cette application demande le suivant, donc après une connexion perdue une application peut
voir un message deux fois. Les ACK des DM envoyés par une application qui n'est plus connectée sont gardés pour elle (jusqu'à
24 heures) et remis quand elle se reconnecte ; `/status` `queue.held_acks` les compte. Cette API n'est
limitée par rien de tout cela : `GET /api/v1/messages` fournit tout l'historique stocké.

### Stockage et copies

- Chaque message reçu ou envoyé (depuis une application sur le port 5000 ou par `POST /api/v1/messages`) est
  stocké avec son texte (chiffré) et, pour les nôtres, la commande sous laquelle il est parti.
- `mirror_sent_channel` / `mirror_sent_dm` (`[store]` dans la configuration) décident seulement si les **autres applications sur le port 5000** reçoivent une
  copie de ce qu'un appareil envoie. Exemple : un message envoyé sur Public depuis un téléphone atteint une tablette et une
  application de bureau comme un message reçu (marqué "↪ <nom>") ; les copies de DM s'affichent dans la conversation du contact
  avec un préfixe. Désactivé = les autres applications ne reçoivent aucune copie de ce genre. Une application MeshCore générique voit donc
  exactement ce que le compagnon lui montrerait ; l'historique, les interfaces et `resend_of` voient tout.
- Supprimer un message dans une application ne le supprime jamais du stockage du proxy (les autres appareils le reçoivent quand même) ;
  vider une conversation est un choix d'affichage de cette application.
- `DELETE /api/v1/messages` et `DELETE /api/v1/messages/{id}` ne touchent que le stockage du proxy : le
  compagnon, le contact et les copies que les applications ont déjà restent. Rien d'autre dans le stockage ne pointe vers une
  ligne de message, et le proxy abandonne ce qu'il suivait en mémoire pour eux (suivi de la remise, étiquetage
  tardif du canal). Les clients retirent les messages supprimés (et tout état non lu les concernant) de leur
  propre cache à la réception de `messages_deleted` ; une application MeshCore générique n'est pas concernée, sauf qu'elle ne les reçoit
  plus lors d'une synchronisation ultérieure.
- La durée de conservation (textes, métadonnées, paquets, historique des noeuds, contacts archivés) se règle sur la machine du proxy
  (`mcproxy retention`) ; un texte expiré par `[store] retention_days` laisse le message avec
  `text: null`, `text_state: "expired"` (les métadonnées restent, dans la limite de `[store] metadata_retention_days`).
- Les anciens proxys avaient un réglage `[api] message_text` qui pouvait retenir les textes au moment de les fournir ; les
  textes étaient quand même stockés et s'affichent une fois le proxy mis à jour.

### Données déduites

Certaines valeurs sont déduites de ce que le proxy a entendu, et ne sont pas des faits rapportés par la radio :

- `knows_us` : `true` quand il est prouvé que le noeud a notre clé (il nous a envoyé un DM, ou a confirmé un des
  nôtres). Un DM ne fonctionne que dans les deux sens : il nous faut le noeud comme contact pour envoyer, et il lui faut nous avoir comme
  contact pour le lire, donc "ne peut peut-être pas vous lire" tant que c'est `null`.
- `via`, `path_nodes` et chaque [saut résolu](#sauts-résolus) : un id court résolu d'après le contexte, avec
  `confidence`.
- `same_hash_nodes` dans `GET /api/v1/nodes/{key}/heard` : des relais d'autres noeuds ayant le même premier
  octet peuvent être comptés.
- `repeats` de nos messages de canal et `GET /api/v1/messages/{id}/heard` : associés par canal (ou
  expéditeur + nous pour les DM), taille et nombre de sauts parmi les paquets entendus peu avant : au mieux.
- `echo_of` est exact (empreinte du paquet).

### Vos propres noeuds

`mine` = marqué comme vôtre à la main, ou sinon votre dernière connexion à ce noeud a donné `admin`. Réglez-le avec
`PATCH /api/v1/nodes/{key}` `{"mine": true | false | null}` : `true` ajoute un noeud, `false` le garde hors de
la liste même avec une connexion admin, `null` revient à l'automatique. `GET /api/v1/nodes` avec
`mine=true` ne liste que les vôtres. Gardé par compagnon par le proxy, donc tous les clients voient la même liste ; `mcproxy reset`
n'y touche pas.

### Rôles et connexions

`access` sur un noeud : `{"role", "firmware", "at"}` d'après sa dernière réponse de connexion, par n'importe qui (une application aussi) ;
`null` = aucune connexion vue. `role` est `"admin"`, `"read_write"`, `"read_only"`, `"guest"` ou `null` (un
noeud plus ancien qui ne le dit pas). `firmware` est le niveau de firmware du noeud.

| Rôle | Commandes | État, voisins, propriétaire | Télémétrie | Salon : publier |
|---|---|---|---|---|
| `admin` | oui | oui | complète | oui |
| `read_write` | non | oui | complète | oui |
| `read_only` | non | oui | complète | non |
| `guest` | non | oui | de base (batterie, température du MCU) | non |

**Mots de passe enregistrés.** `"save": true` sur une connexion garde le mot de passe une fois la connexion réussie,
`"save": false` oublie celui qui est enregistré une fois qu'elle a réussi, et sans `save` rien ne change. Une connexion
qui échoue ne change rien. `{"use_saved": true}` se connecte avec le mot de passe enregistré.
`DELETE /api/v1/nodes/{key}/login` l'oublie. Les mots de passe sont stockés chiffrés, par compagnon, et
partagés par tous les clients ; aucun endpoint n'en renvoie (chaque noeud a `saved_login`). Le proxy utilise aussi le
mot de passe enregistré d'un noeud quand il se connecte de lui-même (pour demander l'état, les voisins et ainsi de suite), avant son
mot de passe général de suivi. `mcproxy reset --mesh` les efface.

**Messages CLI.** Une commande ou une réponse envoyée par une application par le port 5000 reste dans
`GET /api/v1/messages` avec `"cli": true` (les applications en ont besoin) : gardez-les hors des chats, des compteurs de non lus et des
notifications. Les réponses aux commandes envoyées par `POST /api/v1/nodes/{key}/cli` ne sont pas du tout des
messages.

### Archive des contacts et autres compagnons

**Archive des contacts.** Le proxy garde chaque contact que le compagnon a eu (le compagnon en contient
environ 350). `left_companion_at` = quand un contact a quitté la liste du compagnon. Quand tous les contacts
sont listés (`contacts=all` ou `hours=0`), les contacts archivés jamais entendus sont listés aussi (`contact: false`,
`favorite: null`, `out_path: null`, `heard: false`, `left_companion_at` renseigné). Combien de temps un contact reste
après son départ : `[store] contact_archive_days` (0 = toujours), `mcproxy retention --contacts N`. `/status`
montre `store.contacts_archived`, `store.contacts_archived_removed` et `store.retention.contacts_days`.
`POST /api/v1/nodes/{key}/restore` en remet un.

**Un autre compagnon connecté.** Le proxy reconnaît son compagnon à sa clé publique, et chaque compagnon
a ses propres données (messages, contacts, canaux, mesh, historique). Quand un autre se connecte, les
données du précédent sont mises de côté (`data/companions/<key>.db`) et le nouveau commence vide, ou
retrouve ses propres données s'il a déjà été connecté. Rien n'est modifié sur aucune des deux radios. Un événement `reset` (toutes les parties) et un événement `status`
suivent : relisez tout. Un noeud qui est un contact de l'autre compagnon indique la clé de celui-ci dans
`other_companion`. `mcproxy reset` efface les données du compagnon connecté.

### Réglages d'ajout automatique

Quels contacts le compagnon ajoute de lui-même (les réglages "auto add" de l'application MeshCore), dans
`GET /api/v1/status` `radio.auto_add` :

```json
{ "mode": "selected", "chat": true, "repeater": true, "room": false, "sensor": false,
  "overwrite_oldest": true, "max_hops": 0 }
```

| Champ | Type | Signification |
|---|---|---|
| `mode` | string | `"all"` = chaque noeud entendu est ajouté et les interrupteurs de type ne s'appliquent pas ; `"selected"` = seulement les types activés. |
| `chat`, `repeater`, `room`, `sensor` | bool ou null | Les interrupteurs de type ; `null` sur un firmware qui ne les indique pas. |
| `overwrite_oldest` | bool ou null | Quand la liste est pleine, le plus ancien contact non favori est remplacé. Les favoris ne le sont jamais : le firmware les saute en choisissant le plus ancien (`BaseChatMesh::allocateContactSlot`) ; si tous les contacts sont des favoris, rien n'est ajouté. |
| `max_hops` | integer | 0 = toute distance, 1 = entendus directement seulement, N = jusqu'à N-1 sauts. |

`auto_add` est `null` tant qu'il n'a pas été lu. Le proxy garde de toute façon chaque noeud entendu (`GET /api/v1/nodes`, l'archive
des contacts), quoi que le compagnon ajoute. Modifiez-les avec
`PATCH /api/v1/companion/settings`.

### C'est le compagnon qui décide

Les réglages du compagnon (contacts, favoris, chemins, canaux, ajout automatique, taille des ids de chemin, région
par défaut, paramètres radio, nom) lui appartiennent : le proxy ne les modifie jamais de lui-même, il transmet seulement
les modifications demandées par l'API ou la CLI. Quand une application les modifie par le proxy (les
interrupteurs de l'application MeshCore, une application de bureau, `mcproxy`), le proxy les relit dans le compagnon et
envoie un événement `status`, donc chaque client montre ce qu'a la radio. Une modification faite avec la radio
connectée directement à un téléphone est lue à la prochaine connexion du proxy.

### Régions (portées flood)

Un message flood peut être limité à une région (firmware v1.15 ou plus récent ; "aucune région" pour un seul envoi demande
la v1.16). Les noms de région que le proxy connaît viennent de la région par défaut du compagnon, des régions propres
des canaux, des noms ajoutés à la main (`PUT /api/v1/regions`) et des noms que les répéteurs fournissent
(`POST /api/v1/regions/fetch`).

- La **région par défaut du compagnon** est enregistrée sur le compagnon (`PATCH /api/v1/companion/region`).
- La **région propre d'un canal** est gardée par le proxy, la même pour toutes les applications (`PATCH /api/v1/channels/{id}`,
  la vue Chat de la console, ou apprise d'une application MeshCore qui envoie dans ce canal avec sa propre région ;
  une région par canal, la dernière définie l'emporte). `"*"` = aucune région, `null` = celle du compagnon par défaut.
  Un message envoyé dans ce canal part dans cette région, puis la région par défaut du compagnon revient.
- Les messages et paquets reçus portent `scoped` (envoyé dans une région) et `region` (son nom, quand elle
  correspond à une région connue).

### Objets du graphe du mesh

Le graphe (`GET /api/v1/mesh/graph`) se construit au fil du temps. Les ids de chemin sont résolus en noeuds comme dans un
[saut résolu](#sauts-résolus).

**Noeud**

```json
{ "id": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90",
  "key": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "name": "Hilltop",
  "kind": "repeater", "lat": 12.3, "lon": 45.6, "hash": "a1", "confidence": 0.92, "choices": 1,
  "packets": 1200, "last_seen": 1793600400 }
```

| Champ | Type | Signification |
|---|---|---|
| `id` | string | Clé, `"h:<hex>"` (un id qu'aucun répéteur entendu n'a) ou `"self"`. |
| `kind` | string | Type de noeud, ou `"self"`. |
| `hash` | string ou null | L'id de chemin sous lequel il a été vu. |
| `confidence` | number | La moyenne de toutes les fois où un id de chemin a été résolu vers lui (1 pour `self` et les noeuds vus seulement comme origines, 0 pour les ids `"h:"`). |
| `choices` | integer | Répéteurs entendus qui partagent son id. |
| `packets`, `last_seen` | integer / integer ou null | Paquets auxquels il a pris part, et quand il a été vu pour la dernière fois. |
| `could_be` | array | Seulement quand `choices` > 1, comme dans un saut résolu. |

**Lien**

```json
{ "from": "a1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90", "to": "self",
  "heard_forward": 800, "heard_back": 0, "snr_forward_db": 7.0, "snr_back_db": null,
  "last_seen": 1793600400, "sources": ["paths", "neighbours"] }
```

| Champ | Type | Signification |
|---|---|---|
| `heard_forward` | integer | Nombre de fois où `to` a entendu `from` (paquets passés dans ce sens). |
| `heard_back` | integer | Nombre de fois où `from` a entendu `to`. |
| `snr_forward_db`, `snr_back_db` | number ou null | SNR mesuré du côté qui reçoit (notre réception, une table de voisins ou une trace) ; `null` quand il n'a jamais été mesuré. |
| `sources` | array | `"paths"`, `"neighbours"`, `"traces"`. |

Les liens à sens unique et la force des liens sont des calculs du client. Ne signalez jamais un lien avec `"self"` comme étant
à sens unique : notre compagnon n'entend jamais ses propres émissions. Un chemin n'est enchaîné que pour les paquets flood,
jamais pour les paquets directs ni les traces.

---

## Où se trouve chaque réglage

- **Compagnon** = enregistré dans la radio elle-même : chaque application le voit (par le proxy ou directement en
  BLE / USB), et il survit sans le proxy et à une réinstallation du proxy. Le proxy envoie seulement la
  commande.
- **Proxy** = le fichier de configuration ou la base de données du proxy : seuls les clients de ce proxy le voient ; la radio
  n'en sait rien.
- **Client seulement** = chaque application le garde ; jamais envoyé.

| Réglage / données | Stocké sur | Modifié par |
|---|---|---|
| Liste des contacts (qui est un contact) | Compagnon | Applications ; `POST /api/v1/nodes/{key}/restore` ; `mcproxy prune` |
| Favori d'un contact | Compagnon (indicateurs du contact) | `PATCH /api/v1/nodes/{key}` `{"favorite"}` |
| Vos propres noeuds (`mine`) | Proxy (par compagnon) | `PATCH /api/v1/nodes/{key}` `{"mine"}` |
| Mots de passe de connexion enregistrés | Proxy (par compagnon, chiffrés, jamais renvoyés) | `POST` / `DELETE /api/v1/nodes/{key}/login` |
| Chemin d'un contact (`out_path`, la taille de ses ids) | Compagnon (contact) | `PATCH /api/v1/nodes/{key}` `{"path"}` |
| Ajout automatique : tous / choisis, types, sauts max (`radio.auto_add`) | Compagnon | `PATCH /api/v1/companion/settings` |
| Notre taille des ids de chemin (`path_id_bytes`, mode de hash de chemin) | Compagnon | `PATCH /api/v1/companion/settings` |
| Remplacer le plus ancien quand la liste est pleine (`radio.auto_add.overwrite_oldest`) | Compagnon | `PATCH /api/v1/companion/settings` `{"overwrite_oldest"}` ; `mcproxy contacts --overwrite-oldest` |
| La région par défaut du compagnon | Compagnon | `PATCH /api/v1/companion/region` |
| Canaux (noms, clés, emplacements) | Compagnon (le proxy garde une sauvegarde) | Applications ; `POST /api/v1/channels` ; `DELETE /api/v1/channels/{id}` ; `mcproxy restore` |
| Nom du noeud, position, paramètres radio (fréquence, bande passante, SF, CR, puissance TX), modes de télémétrie, règle de position dans les annonces, ACK multiples, PIN BLE | Compagnon | Applications seulement (transmis tel quel ; pas d'endpoint API) |
| La région propre d'un canal | Proxy (base de données), la même pour toutes les applications | `PATCH /api/v1/channels/{id}` `{"region"}` ; la vue Chat de la console ; apprise d'une application MeshCore qui y envoie |
| Noms de région ajoutés à la main | Proxy (base de données) | `PUT /api/v1/regions` |
| Copies des messages envoyés (`mirror_sent_channel`, `mirror_sent_dm`) | Proxy (configuration) | `PATCH /api/v1/settings` ; `mcproxy mirror` |
| Propre texte dans les logs (`hide_own_text`) | Proxy (configuration) | `PATCH /api/v1/settings` ; `mcproxy text own` |
| Envoyer sans saturer (`[polite]` : activé / désactivé, rafale, intervalle) | Proxy (configuration) | `mcproxy polite on` / `off` |
| Ce que les applications sur le port 5000 rattrapent (`sync_days`, `sync_per_channel`). Cette API n'est pas limitée : `GET /api/v1/messages` fournit tout l'historique | Proxy (configuration) | `mcproxy sync` |
| Durée de conservation (textes, métadonnées, paquets, historique des noeuds, contacts archivés) | Proxy (configuration) | `mcproxy retention` |
| Nettoyage automatique des contacts (`prune_days`) | Proxy (configuration) ; les retraits se font sur le compagnon | `mcproxy prune --auto` |
| Noeuds suivis (`[observer] track`) | Proxy (configuration) | `PATCH /api/v1/nodes/{key}` `{"track"}` ; `mcproxy track` |
| Messages, paquets, historique des noeuds, réponses voisins / propriétaire, traces, historique CLI, archive des contacts, chronologie des chemins | Proxy (base de données) | Enregistré |
| Jeton de l'API, certificat, adresses autorisées, ports | Proxy (configuration, `~/.config/mcproxy`) | Sur la machine du proxy seulement (`mcproxy api`) |
| Chats vidés, messages masqués / supprimés dans une vue, marques de lecture, filtres, tri, disposition, thème | Client seulement | L'application |
