# Capture — Reboot du design de `pouchdb-http-router`

## 1. Identité, finalité et portée du change

Cette Capture concerne le reboot architectural de `pouchdb-http-router`. Elle conserve l'état de design nécessaire pour reconstruire ultérieurement `docs/design/README.md`, `contract.md`, `architecture.md`, `environment.md` et `decisions.md`.

La Capture est une projection de travail non normative. Elle n'institue ni identifiant ni étape OpenSpec. Les futurs documents de design devront être relus et institués séparément.

La finalité établie est de fournir la surface HTTP nécessaire pour connecter et synchroniser efficacement **2..n nœuds PouchDB**. Le routeur transporte le protocole requis entre PouchDB ; il n'expose pas PouchDB comme base Web généraliste et ne cherche pas à reproduire CouchDB.

Le nom courant `pouchdb-http-router` décrit le mécanisme sans élargir le contrat. Le mot `http` ne signifie pas « API distante complète » : en V1, la capacité garantie est la synchronisation.

## 2. Autorité et articulation documentaire à reconstruire

La documentation cible doit séparer les responsabilités suivantes :

- `contract.md` fixe le problème résolu, la surface garantie, les non-objectifs et les propriétés attendues ;
- `architecture.md` décrit la structure qui réalise ce contrat et ses invariants internes ;
- `environment.md` décrit l'enveloppe d'exécution, les intégrations, les preuves et les conditions de benchmark ;
- `decisions.md` conserve les arbitrages, alternatives écartées, hypothèses jusqu'à falsification et compromis ;
- `README.md` indexe ces quatre documents, explique leur articulation et indique où se trouve chaque information normative.

La suite de tests prouve certains comportements, mais ne définit pas le contrat des futurs documents de design.

## 3. Critères et contraintes matériels

**C-FIDELITY — Fidélité PouchDB.** Le comportement exposé doit être celui qu'attend le replicator PouchDB courant sur la surface retenue. Les différences volontaires avec CouchDB doivent être explicites plutôt que masquées.

**C-PERFORMANCE — Performance utile.** Le routeur doit rester une couche mince. Les optimisations protocolaires qui évitent un fallback coûteux, notamment `_bulk_get`, priment sur les micro-optimisations internes. Les comparaisons de performance doivent isoler autant que possible le coût du routeur de celui du framework hôte.

**C-SIMPLICITY — Contrainte minimale.** Le routeur hérite de l'enveloppe imposée par le PouchDB serveur courant et n'ajoute pas de runtime, framework, stockage, parser ou lifecycle spéculatif. Une facilité d'implémentation n'est pas une raison suffisante pour élargir la surface HTTP.

**C-INTEGRATION — Intégrabilité directe.** Le même handler doit fonctionner sous `node:http` et directement sous Express, avec la frontière applicative précisée par C-STORAGE et R-DB-EXISTS-BOUNDARY.

**C-STORAGE — Stockage opaque, existence fournie par l'application.** Le routeur reçoit un constructeur ou preset PouchDB et la fonction `databaseExists(name)` de l'application. Il ne connaît ni LevelDB, ni chemin de fichiers, ni adapter concret. `PouchDB.defaults(...)` reste le mécanisme naturel de configuration du stockage. La position antérieure « constructeur/preset seul, sans apport applicatif » est supersédée pour Q-DB-CREATION ; aucune factory générale de bases n'est requise en V1. L'application est responsable de la vérité de `databaseExists` dans l'espace de noms qu'elle expose, notamment pour les bases externes ou préexistantes ; le routeur n'en inventorie aucune.

**C-STATEFUL — Serveur persistant.** Le modèle est un processus serveur stateful et long-lived. Les environnements FaaS, edge ou Workers sans stockage local persistant sont hors cible tant que PouchDB ne change pas lui-même cette enveloppe.

## 4. Contrat produit courant

### R-SYNC-SCOPE — Surface nécessaire à 2..n nœuds

Une primitive entre dans la V1 lorsqu'elle est nécessaire à la connexion ou à la synchronisation correcte et performante des nœuds PouchDB actuels. Les orientations historiques écartées sont rappelées dans R-SYNC-FIRST.

**R-ROUTE-TABLE — D1 explicitement validée le 2026-09-27.** L'audit de `pouchdb-replication`, `pouchdb-adapter-http`, `pouchdb-checkpointer` et `pouchdb-changes-filter` au commit PouchDB `27de91f1105a8074ddd06f5a23156dd99c4eb016` fonde la table cible V1 ci-dessous, désormais retenue dans son ensemble. Une synchronisation bidirectionnelle est composée de deux réplications unidirectionnelles ; la topologie 2..n n'ajoute pas de primitive HTTP. Cette validation fixe le contrat de surface, non la preuve qu'un routeur V1 l'implémente déjà.

| Capacité sémantique | Couples méthode/chemin | Rôle |
| --- | --- | --- |
| information/existence de base | `GET /:database` | setup et `info()` de la source, dont `update_seq` |
| création de base | `PUT /:database` | setup automatique après 404, sauf `skip_setup` |
| changements | `GET` et `POST /:database/_changes` | one-shot/live en longpoll ; `POST` pour `doc_ids` et `selector` |
| révisions manquantes | `POST /:database/_revs_diff` | calcul de réplication |
| écriture groupée | `POST /:database/_bulk_docs` | écriture répliquée avec `new_edits:false` |
| lecture groupée | `POST /:database/_bulk_get` | lecture de révisions avec `revs:true`, `latest:true` ; trajet performant |
| lecture documentaire de repli | `GET /:database/:documentId` et `GET /:database/_design/:designId` | shim de `_bulk_get` après échec, y compris documents de design |
| lecture de pièce jointe | `GET /:database/:documentId/:attachmentPath+` et `GET /:database/_design/:designId/:attachmentPath+` | récupération binaire, y compris pièces jointes des documents de design et noms contenant `/` |
| document local de checkpoint | `GET` et `PUT /:database/_local/:localId` | lecture/écriture sur source et/ou cible selon l'option de checkpoint |

La table compte **treize formes de route**, dont onze sur le trajet `_bulk_get` fonctionnel et deux pour le repli. Les URL de base émises par l'adapter ont une barre finale : les formes `/:database/` et `/:database` relèvent des mêmes opérations. Le matcher traite les chemins réservés `_local` et `_design` avant les formes génériques, capture les segments avant décodage et restitue intégralement les noms de pièces jointes. Ces détails de matching sont fondés sur l'encodeur de l'adapter ; leur réalisation exacte reste à vérifier en conception/implémentation.

**R-DB-CREATION — Q-DB-CREATION explicitement validée le 2026-09-28.** À stockage et options PouchDB équivalents, lorsque les requêtes franchissent les limites HTTP configurées et qu'aucune politique de hooks ne change ou ne refuse l'opération, la synchronisation via le routeur doit produire les mêmes résultats et issues qu'une synchronisation PouchDB directe. Un `413` lié au plafond de body ou un refus applicatif explicite peut donc empêcher une sync directe pourtant valide ; la fidélité n'est pas une promesse inconditionnelle malgré ces frontières. La conformité fonctionnelle ne dispense pas de respecter le dialogue HTTP du client. `GET /:database` consulte la base sans en demander la création : base absente connue → `404 not_found`, base existante → informations de base. `PUT /:database` demande la création : `201` seulement après ouverture utilisable, `412` si la base est déjà connue comme existante et utilisable. Les autres routes sync ne créent pas implicitement une base absente et répondent `404` lorsqu'elle est connue comme absente, sous réserve des étapes antérieures du pipeline (parsing et hooks). Une erreur de stockage que le routeur ne peut qualifier ne devient ni `404` ni `412`. Le setup HTTP PouchDB ordinaire peut faire `GET` 404 → `PUT` 201 ou, lors d'une course gagnée par un autre appel, accepter `412` ; `skip_setup` ne crée pas par ce mécanisme. Le routeur n'est responsable que de la vérité qu'il peut établir à sa frontière applicative et de la concurrence entre requêtes qu'il traite lui-même ; les incohérences ou choix de gestion du stockage par l'application ne sont pas compensés par une découverte universelle des bases.

**R-BULK-GET-FALLBACK — choix validé pour D1.** `_bulk_get` appartient à la V1 pour préserver les performances ; les deux lectures documentaires préservent le repli du client PouchDB courant après un premier échec de `_bulk_get`. Ce repli, introduit avec `_bulk_get` en 5.0.0 et élargi jusqu'à toutes les erreurs en 6.1.1, n'est pas une compatibilité réservée aux anciens clients. L'échec est mémorisé par URL de base ; le client passe ensuite par des `GET` avec `open_revs`, `revs` et `latest`. Le compromis est l'exposition de lectures directes de documents sans promesse d'API documentaire généraliste. L'alternative de limiter le contrat au seul trajet `_bulk_get` fonctionnel a été explicitement écartée le 2026-09-27.

L'option `id()` du client tente `GET /` pour obtenir un UUID ; si celui-ci manque ou si la lecture échoue, l'adapter se rabat sur l'URL de la base. Aucune route root n'est nécessaire. `live` répète `_changes?feed=longpoll` avec heartbeat/timeout et annulation ; checkpoints désactivés ou limités à la source/cible n'ajoutent pas de routes. Les filtres `doc_ids`/`selector` passent par `POST _changes`, les filtres nommés et `_view` par `_changes` sans endpoint HTTP `_view`. **D1 inclut les filtres nommés et par vue : la source doit lire localement le document de design et exécuter sa fonction de filtre ou de map côté serveur**, comme le plugin PouchDB. Cette frontière de confiance et son coût sont acceptés au titre des capacités de synchronisation, sans ajouter une API distante de vues. Les filtres fonctionnels fournis au replicator s'exécutent côté client.

### R-NOT-REMOTE-DB — Frontière négative

La V1 ne garantit pas une API distante généraliste. Restent hors cible sauf besoin nouveau institué :

- destruction et maintenance distante de base, dont `DELETE /:db` et `_compact` ;
- `_all_docs` comme API de consultation distante ;
- création, mise à jour ou suppression documentaire hors opérations requises par la synchronisation ;
- écriture ou suppression directe de pièces jointes ;
- consultation distante de vues persistantes, `_temp_view` et `_view_cleanup` (le filtre `_view` de `_changes` reste dans la synchronisation) ;
- root welcome, `_session`, administration CouchDB, Fauxton, `_users`, `_replicator`, cluster/configuration et autres surfaces CouchDB ;
- `HEAD` générique historique ;
- `multipart/related` historique.

Ces exclusions n'amputent pas PouchDB : les opérations qui peuvent rester locales s'exécutent sur chaque nœud. Un appel HTTP direct avec `curl` ou `fetch` peut naturellement fonctionner sur les routes présentes, mais il ne crée pas un contrat produit supplémentaire.

Le profil de synchronisation n'est pas une ACL. `_changes`, `_bulk_get` et `_bulk_docs` donnent déjà des capacités de lecture et d'écriture importantes. L'authentification périphérique et les hooks/politiques applicatives restent nécessaires lorsque le réseau n'est pas de confiance.

### R-CAPABILITIES — Extension future sans promesse actuelle

Décision D13 explicitement validée le 2026-09-27 : chaque déclaration de route porte dès la V1 la métadonnée interne `capability: "sync"`. Cette classification prépare l'ajout éventuel de `api` sans établir de profil `api`, d'option publique `mode` ni de seconde surface HTTP en V1. Toute capacité supplémentaire exige un besoin réel et une décision séparée.

## 5. Architecture courante

### R-HANDLER — Frontière HTTP native

L'hypothèse conservée jusqu'à falsification est un handler unique `(req, res)` fondé seulement sur les primitives `node:http` nécessaires :

- requête : `method`, `url`, `headers` et stream lisible ;
- réponse : `statusCode`, `setHeader()`, `headersSent`, `write()` et `end()`.

Le même handler doit fonctionner sous `http.createServer(handler)` et `app.use('/prefix', handler)`. Express est une intégration de référence par la documentation, les tests et le benchmark, pas une dépendance architecturale. Aucun wrapper Express n'est prévu tant qu'un cas réel ne falsifie pas le montage direct.

Le point de montage appartient à l'hôte. `routerPrefix` était un artefact de la catch-all Next.js et doit disparaître. Sous Express, le handler consomme l'URL relative fournie au point de montage ; sous Node natif, le dispatch éventuel appartient à l'application.

Le cœur ne doit utiliser aucune commodité Next.js/Express telle que `req.query`, `req.params`, `req.locals`, `res.locals`, `res.status()`, `res.json()` ou `res.send()`. Il ne doit pas inventer en remplacement une abstraction HTTP universelle, un `CanonicalRequest` ou une API d'adapters.

### R-PIPELINE — Traitement sémantique

Le pipeline cible est :

1. analyser l'URL et sélectionner une déclaration de route précompilée ;
2. produire `operation`, `database` et paramètres nommés ;
3. parser query et body suivant le mode déclaré (`none`, `json`, `raw`) ;
4. exécuter séquentiellement les hooks `before` ;
5. sauf court-circuit, résoudre le handle PouchDB en cache ;
6. exécuter l'opération PouchDB ;
7. normaliser succès ou erreur attendue en réponse sémantique ;
8. exécuter séquentiellement les hooks `after` ;
9. écrire status, headers et payload sur la réponse HTTP native.

**R-BEFORE-ORDER — D3 explicitement validée le 2026-09-27.** Le body est lu et parsé selon le mode de la route avant les hooks `before`. Ceux-ci s'exécutent avant la résolution/création du handle PouchDB et avant l'opération. Un refus par `before` ne déclenche donc aucun accès PouchDB pour cette requête ; il ne signifie pas qu'aucun handle de cette base n'était déjà ouvert dans le cache par une requête antérieure. Une erreur de lecture ou de parsing intervient avant `before` et ne peut être refusée par ce hook.

La déclaration interne d'une route doit rester petite : opération sémantique, capacité `sync` en V1, méthode, chemin, mode de body et handler. Les matchers sont compilés une fois à l'initialisation.

Le pathname brut est matché avant décodage. Seuls les paramètres capturés sont décodés, afin qu'un `/` encodé dans un identifiant de document ou de pièce jointe ne devienne pas prématurément un séparateur structurel. Les paramètres positionnels et leur reconstruction depuis une query Next.js disparaissent.

La réponse sémantique publique et interne reste orientée protocole — `{ status, headers?, body? }`, avec `status` requis — et non abstraction HTTP générale. Objets/tableaux et `null` deviennent du JSON, `Buffer` reste binaire, les chaînes restent texte/raw et `undefined` n'émet aucun body. De petits utilitaires internes peuvent assurer la sémantique JSON, binaire et erreur avant l'écriture native.

### R-OPERATIONS — Identité sémantique

Les hooks voient une opération stable, pas le pattern de route ni les anciens noms historiques. Deux transports HTTP d'une même capacité peuvent partager une opération ; la méthode brute reste consultable sur `req` si nécessaire.

**O-OPERATION-VOCAB — nomenclature D2 explicitement validée le 2026-09-27** : `database.info`, `database.create`, `changes.read`, `revisions.diff`, `documents.bulkRead`, `documents.bulkWrite`, `attachment.read`, `localDocument.read`, `localDocument.write` et `document.read` (pour les deux chemins de repli). Ces identifiants plats décrivent la ressource et l'action ; `GET`/`POST _changes` partagent `changes.read`, comme les deux chemins de pièce jointe partagent `attachment.read` et les deux chemins documentaires partagent `document.read`. Le vocabulaire des paramètres est `database`, `documentId`, `designId`, `attachmentId` et `localId` ; `documentId` doit être reconstitué comme `_design/…` sur le chemin d'un document de design, et `attachmentId` comprend ses éventuels `/`. `localDocument` n'affirme pas qu'un document `_local` est forcément un checkpoint ; `sync.push`/`sync.pull` ne sont pas déductibles d'une seule requête HTTP. Leur place dans le contexte public est fixée par R-HOOK-CONTEXT.

### R-HOOKS — Extension générique V1

Deux phases génériques `before` et `after` sont retenues en V1. Les hooks sont du code de confiance dans le processus, asynchrone si nécessaire et exécuté séquentiellement dans l'ordre déclaré. Aucun DSL de filtrage route/méthode, aucun drapeau `skip*` et aucun type de hook spécialisé n'est prévu ; R-GENERIC-HOOKS explique cet arbitrage.

**R-HOOK-CONTEXT — contrat d'extension validé le 2026-09-27.** Chaque requête a un contexte unique, partagé par les hooks `before` puis `after`. Il expose `req`, `operation`, `database`, `params`, `query`, `body`, `state`, `db`, `response` et `committed`, sans `res` brut ni déclaration de route. Les champs sont disponibles selon la phase :

- `req` est consultable sans consommer à nouveau son stream ; `operation` est immuable ; `database`, `params`, `query` et `body` sont parsés et transformables dans `before`, puis exposent dans `after` les valeurs effectivement transmises ;
- `state` est un objet propre à la requête, modifiable dans les deux phases pour communiquer entre hooks ;
- `db` est absent dans `before`, puis fournit dans `after` le handle utilisé, ou reste absent si `before` a court-circuité ; un refus ne provoque aucun accès PouchDB pour cette requête ;
- `response` est absente dans `before` : seul le retour d'une réponse court-circuite ; dans `after`, elle désigne la réponse sémantique courante, modifiable avant émission de ce qui reste à écrire ;
- `committed` est un booléen en lecture seule porté par le contexte, non par `response`, et reflète l'envoi effectif des headers HTTP (`false` dans `before` ; dans `after`, état réel du transport). Remplacer `response` ne peut pas le réinitialiser ;
- si des hooks modifient `database` ou les paramètres de ciblage, le contrôle d'autorisation doit porter sur la cible finalement transmise à la résolution du handle ; un contrôle sur une cible ensuite remplacée ne garantit pas le refus avant accès à la base ;
- retourner `undefined` conserve le contexte courant, y compris ses mutations ; une réponse retournée par `before` court-circuite les hooks `before` restants et l'opération sans résoudre `db`, puis passe par tous les `after` ; une réponse complète retournée par `after` remplace la réponse courante, puis les autres `after` s'exécutent. Une mutation directe de `response` dans `after` persiste si le hook retourne `undefined` ;
- les erreurs PouchDB attendues sont normalisées avant `after`, afin que celui-ci voie une réponse sémantique plutôt qu'une exception interne ;
- sans hooks, le chemin nominal évite boucle et `await` spécifiques ; le contexte existe déjà pour le pipeline.

Le compromis est explicite : un hook d'autorisation sémantique intervient après parsing et peut donc en payer le coût. Une authentification périmétrique qui doit refuser avant lecture du body reste placée devant le routeur dans l'application hôte. La V1 accepte de matérialiser les bodies JSON/raw nécessaires avant `before` ; elle ne vise pas une API générique de streaming entrant.

**R-BODY-LIMIT — défaut V1 explicitement validé le 2026-09-27.** Les routes qui parsèment un body `json` ou `raw` appliquent pendant la lecture, avant `before` et avant tout accès PouchDB pour cette requête, un plafond de **64 MiB par requête** par défaut. L'option explicite `bodyLimit` permet de l'adapter au déploiement ; une configuration invalide doit être rejetée et aucun défaut inférieur du parser ne doit s'appliquer implicitement. Un dépassement renvoie HTTP `413` sans invoquer `before`. Les routes de mode `none` n'accumulent pas de body. Le chiffre est un compromis produit raisonnable faute de données pour le falsifier, et reste révisable : ce n'est pas une norme PouchDB ni une garantie que tout document ou lot valide sera synchronisable. Un `_bulk_docs` ou une pièce jointe en base64 peut dépasser le plafond ; le rehausser augmente la mémoire consommée par le parsing, également sous concurrence. Une limite plus basse dans l'hôte ou un proxy peut prévaloir ; l'authentification périmétrique précoce reste de la responsabilité de l'hôte.

`after` ne promet pas d'intercepter chaque octet. Sur `_changes`, un premier heartbeat peut avoir engagé status et headers avant le résultat final : `committed = true`. `after` peut alors observer la réponse et modifier le payload final encore non émis, seulement dans les limites du status et des headers effectivement envoyés (notamment le type de contenu). Une modification incompatible doit être refusée explicitement, jamais ignorée. Si une erreur survient après engagement sans réponse HTTP fidèle possible, le transport est interrompu au lieu de feindre un nouveau statut ; `after` n'est pas garanti sur cette issue tardive ni sur la déconnexion. Si `committed = false`, `after` peut encore remplacer status, headers et body avant l'écriture native.

### R-DB-LIFECYCLE — Handles PouchDB persistants

Le routeur maintient en interne un handle PouchDB par nom logique de base et le réutilise. Ce modèle évite les allocations/listeners d'un `new PouchDB()` par requête et respecte le partage de connexion de l'adapter LevelDB. Le cache appartient à l'instance du routeur et n'est pas une API publique.

**R-DB-EXISTS-BOUNDARY — signature validée le 2026-09-28.** Les options publiques pertinentes sont `PouchDB: PouchDBConstructor` (constructeur ou preset `PouchDB.defaults(...)`) et `databaseExists: (name: string) => boolean | Promise<boolean>`. Cette dernière est une fonction applicative d'existence *logique* dans l'espace exposé ; elle ne crée pas la base, renvoie `false` uniquement pour une absence connue et échoue si l'application ne sait pas répondre. Le routeur ne traduit pas cet échec en absence. Le `name` est la valeur finale de `context.database` après les hooks `before`. Pour une lecture sans handle en cache, le routeur interroge `databaseExists` et ne construit PouchDB que si elle répond `true` ; pour `PUT /:database`, une existence connue ou un handle déjà en cache conduit à `412`, tandis qu'une absence connue autorise la construction, l'attente de l'ouverture utilisable, la mise en cache puis `201`. Le cache évite les vérifications répétées pour les handles connus ; sa validité et son invalidation restent soumises à R-DB-LIFECYCLE. Une promesse par nom, interne au routeur, coordonne une création en cours et empêche deux `PUT` reçus par cette instance d'annoncer tous deux `201` ; une lecture concurrente attend son issue avant de conclure. Aucun verrou global ni coordination interprocessus ne sont institués. La vérification d'existence suivie d'une construction n'est pas atomique face à un créateur extérieur : l'application assume cette frontière et doit coordonner son propre usage du stockage si nécessaire.

**R-ENCODED-NAMES — vérification de compatibilité, sans décision de remappage.** Le paramètre d'URL décodé alimente `context.database` et `databaseExists` ; l'ancien `pouchdb-express-router` réencode `req.params.db` avant de construire son handle PouchDB. Pour les noms de bases comportant des caractères encodés, la correspondance entre nom logique, nom physique préexistant et handle construit par le nouveau routeur doit être vérifiée. Les URL de sync déjà fixées par D1 ne changent pas ; cette vérification n'institue ni une règle générale de réencodage ni une promesse de compatibilité de tout le stockage historique.

**R-DB-LIFECYCLE — Q-DB-LIFECYCLE-DETAILS explicitement validée le 2026-09-28.** Un handle en cache est invalidé sur ses événements PouchDB `closed` et `destroyed`, y compris une destruction signalée sur ce handle après `destroy()` appelé sur un autre handle du même nom. L'invalidation retire uniquement le handle concerné ; l'accès suivant repasse par `databaseExists` et ouvre au besoin un nouveau handle. Une erreur ordinaire d'opération n'invalide pas le cache. Une fermeture d'un autre handle ne signale pas nécessairement `closed` sur le handle du routeur : la coordination de cet usage extérieur relève de l'application, comme une destruction du stockage sans événement PouchDB. Pas de LRU, TTL, pool ni verrou global justifié par cette décision.

**R-ROUTER-CLOSE — arrêt explicite validé pour la V1.** `router.close()` est asynchrone et ferme une seule fois les ressources appartenant à cette instance : dès l'engagement de la fermeture, le routeur ne démarre plus de traitement ; il termine ou annule ses `_changes` actifs, laisse finir les autres opérations déjà engagées, puis ferme les handles qu'il a créés et conservés. Son achèvement signifie que ces ressources sont libérées ; il ne détruit pas les bases, ne ferme pas le serveur HTTP et ne prend pas en charge les handles créés par l'application. Le serveur peut être arrêté ou le handler retiré par l'hôte selon son intégration ; les flux longs imposent de coordonner cet arrêt avec `router.close()`. R-DB-LIFECYCLE / R-ROUTER-CLOSE rappelle pourquoi l'orientation antérieure sans fermeture publique a été remplacée.

La création implicite d'une base absente est tranchée par R-DB-CREATION et R-DB-EXISTS-BOUNDARY. `skip_setup` ne règle pas l'ouverture d'un adapter local ; aucun `pouchdb-all-dbs`, catalogue universel ou inspection du stockage n'est ajouté au routeur.

### R-CHANGES — Longpoll et ressources

`_changes` doit préserver le longpoll et le heartbeat nécessaires au client PouchDB. Lors d'une déconnexion client ou de R-ROUTER-CLOSE, le routeur annule le feed `db.changes()`, efface heartbeat et timeout, puis libère les ressources associées. Cette gestion est un invariant interne de transport, pas une extension publique des hooks.

## 6. Dépendances et contraintes éliminées

`path-to-regexp` reste adapté au matching et à l'extraction des paramètres ; les matchers doivent être précompilés.

`body-parser` est conservable tant qu'il opère directement sur les streams Node et ne déforme pas le contrat. Ses limites par défaut de `100kb` (JSON/raw) doivent être remplacées explicitement pour appliquer R-BODY-LIMIT.

`multiparty` est présumé supprimé : le client PouchDB courant encode les attachments documentaires en JSON/base64 ou utilise la route binaire dédiée, et aucun besoin actuel de `multipart/related` n'a été trouvé. Cette suppression enlève une dépendance, les fichiers temporaires et un lifecycle événementiel fragile. La décision reste falsifiable si la surface sync courante ou ses tests démontrent un besoin multipart.

Next.js, React et les conventions catch-all ne font pas partie du reboot. CORS, Helmet, authentification générique, rate limiting et observabilité applicative appartiennent à l'application hôte ou au reverse proxy, pas au routeur ni aux intégrations de benchmark.

## 7. Environnement, validation et benchmark

### R-RUNTIME — Enveloppe courante

Node.js est le runtime serveur supporté parce que l'écosystème PouchDB serveur courant l'impose. Le routeur ne cherche ni à élargir ni à réduire cette enveloppe. Bun et Deno ne reçoivent ni code, ni CI, ni abstraction dédiés tant que PouchDB ne les supporte pas réellement ; l'usage sobre de primitives `node:http` évite cependant de les bloquer gratuitement.

Express sert de preuve d'intégrabilité et de référence comparative. Fastify et les frameworks exposant les objets Node bruts sont des intégrations plausibles mais non promises. Hono, Web `Request`/`Response`, Bun/Deno natifs et serverless restent hors cible sans falsification ou besoin démontré.

Les streams imposent deux règles d'intégration : le body ne doit pas avoir été consommé irréversiblement avant le handler, et les ressources persistantes de `_changes` doivent être annulées à la déconnexion.

### R-VALIDATION — Preuve à deux étages

**R-TEST-COVERAGE — Q-TEST-COVERAGE explicitement validée le 2026-09-28.** La preuve principale est une suite propre de synchronisations de bout en bout utilisant le vrai replicator PouchDB contre le routeur. Les fixtures et leurs vérifications utilisent directement les handles PouchDB locaux du serveur ; seul le dialogue de synchronisation passe par HTTP. Couvrir push, pull, sync bidirectionnelle, live, conflits, pièces jointes, filtres et checkpoints, avec comparaison pertinente aux résultats d'une sync directe. La suite ne doit pas requérir de routes HTTP de préparation ou de nettoyage hors de la surface V1.

Compléter cette preuve par des tests HTTP ciblés des statuts, du trajet `_bulk_get` et de son repli, de l'encodage des chemins, des hooks, du lifecycle et des invariants de transport. La suite upstream sert à identifier des scénarios et risques, sans être exécutée comme gate supplémentaire ni recopiée massivement. Son profil `minimumForPouchDB` vise l'API PouchDB nécessaire à sa suite générale, plus large que la sync seule. Même les tests upstream de réplication préparent parfois leurs données par `remote.put()`/`remote.bulkDocs()` et nettoient par `remote.destroy()` ; leur sélection par nom ne retire pas ces appels HTTP hors contrat. Une suite verte ne prouve pas à elle seule que tous les trajets et propriétés de la V1 sont couverts, notamment quand le client emprunte silencieusement le fallback de `_bulk_get`.

Pour D1, vérifier séparément le trajet `_bulk_get` effectif sans shim, le repli documentaire, les pièces jointes de documents de design, les filtres nommés/`_view`, l'encodage des chemins et les checkpoints. Cela inclut R-ENCODED-NAMES : tester sur une base préexistante aux noms encodés l'accord entre URL, `databaseExists` et handle PouchDB, en particulier face au réencodage pratiqué par l'ancien routeur Express. Ces vérifications ne rouvrent pas à elles seules la table de routes.
Pour Q-DB-CREATION, comparer des synchronisations PouchDB directes et via HTTP avec mêmes adapter/options et états de départ (résultat, révisions, conflits, checkpoints et issues pertinentes), dans les limites de transport admises par R-BODY-LIMIT, puis contrôler séparément les statuts HTTP d'une base absente, préexistante, créée et des requêtes concurrentes. La seule résolution de `db.info()` côté client avec `skip_setup` n'est pas une preuve de statut HTTP : dans le code audité, cette méthode peut résoudre avec un objet d'erreur sur `404`. Les essais isolés en mémoire ne prouvent ni le futur routeur ni les garanties d'un autre stockage.
Pour R-DB-LIFECYCLE/R-ROUTER-CLOSE, vérifier l'éviction après `closed` et `destroyed`, la réouverture après changement connu, l'arrêt pendant `_changes`, l'attente des opérations déjà engagées et la libération des handles lors du retrait du handler dans un processus qui continue de tourner. L'absence d'événement sur un autre handle fermé par l'application ne prouve pas la capacité du routeur à détecter sa fermeture.

### R-BENCHMARK — Comparaison de référence

**R-BENCHMARK-MATRIX — Q-BENCHMARK-MATRIX explicitement validée le 2026-09-28.** La campagne mesure la même charge de synchronisation avec le vrai replicator PouchDB sur trois cibles : A, `pouchdb-express-router` sous Express ; B, nouveau routeur sous Express ; C, même nouveau handler sous `node:http`. Trois lectures en résultent :

- `pouchdb-express-router` contre le nouveau routeur monté sous Express : coût/bénéfice de l'implémentation à environnement comparable ;
- nouveau routeur sous Express contre le même handler sous `node:http` : coût de l'enveloppe Express ;
- `pouchdb-express-router` contre le nouveau routeur sous `node:http` : résultat système final.

La charge part de notre suite de sync, dans un banc de mesure distinct. Les trois scénarios retenus pour commencer sont la réplication initiale en pull, la réplication initiale en push et la sync bidirectionnelle incrémentale avec checkpoints. Les autres cas (live, filtres, conflits, pièces jointes) sont d'abord testés fonctionnellement ; leur entrée dans le benchmark exige une question de performance concrète. Les volumes précis se calibrent pour obtenir des mesures stables sans durée excessive : ils ne sont pas figés ici.

Même client PouchDB, versions, options, seed et états initiaux équivalents sur A/B/C ; une seule cible travaille à la fois. Préparer les fixtures directement sur les handles locaux, puis mesurer de l'engagement de la réplication à son achèvement ; vérifications et nettoyage sont hors chronométrage. Échauffer chaque cible symétriquement, remettre l'état PouchDB à zéro entre mesures et traiter les caches selon un régime explicité et comparable. Les serveurs peuvent rester chauds sur des ports distincts, sans contamination entre passages. Une charge passe une fois par chacun des six ordres `ABC`, `ACB`, `BAC`, `BCA`, `CAB`, `CBA` : 18 passages par charge équilibrent la position des cibles ; répéter le bloc entier en cas de bruit excessif. Conserver les mesures brutes, la médiane et la dispersion ; pas de retries masquant un échec.

A→B mesure le résultat de la substitution Express ; pour les charges qui lisent des révisions distantes, il inclut la différence de protocole : l'ancien routeur n'offre pas `_bulk_get` et le client peut se replier sur des lectures individuelles. Publier les volumes et nombres d'appels HTTP avec les durées, notamment `_bulk_get` et ce repli ; ne pas attribuer A→B au seul surcoût du handler. Le client PouchDB mémorise la prise en charge de `_bulk_get` par URL de base : préciser pour chaque passage si sa détection initiale est incluse dans la mesure ou déjà acquise par l'échauffement, et ne pas mélanger ces régimes dans une comparaison. B→C estime l'effet de l'enveloppe Express sur le même handler ; A→C donne le résultat système final. Une différence fonctionnelle empêchant une charge commune doit être signalée et traitée avant d'interpréter ses mesures, non masquée par des parcours distincts.

Le développement local valide seulement le harness par un passage ou un sous-ensemble rapide. La CI fonctionnelle vérifie la correction. Le benchmark de référence est une action manuelle dédiée, reproductible par SHA/seed/environnement, qui exécute la campagne complète et publie mesures individuelles et statistiques. Les résultats historiques Next.js/Express éclairent l'origine du reboot mais ne constituent pas la mesure du nouveau design.

## 8. Arbitrages structurants à préserver dans `decisions.md`

**R-BOUNDARY — `node:http` plutôt que Web `Request`/`Response`.** Une abstraction Web universelle aurait ajouté conversions et couches alors que PouchDB serveur est aujourd'hui Node. Le handler natif couvre Node et Express directement et reste l'hypothèse la plus simple jusqu'à falsification.

**R-NO-EXPRESS-WRAPPER — Montage direct plutôt qu'intégration logicielle.** Express accepte les objets Node et le handler. Les exemples, tests et benchmarks suffisent tant qu'aucun besoin réel de wrapper n'apparaît.

**R-SYNC-FIRST — Synchronisation plutôt qu'API distante complète.** L'orientation transitoire « toutes les APIs PouchDB distantes » a été rejetée après audit du replicator : elle entraînerait vues, temp views, compaction, maintenance et potentiellement plugins sans répondre au besoin central. La surface sync conserve les capacités locales de PouchDB et réduit fortement le code exposé.

**R-NOT-COUCHDB — PouchDB plutôt que clone CouchDB.** Le protocole CouchDB n'est utilisé que là où PouchDB en dépend. Les API d'administration CouchDB et la compatibilité serveur générale sont hors cible.

**R-OPERATION — Opération sémantique plutôt que route publique.** Le reboot n'est lié ni aux anciens noms, ni aux patterns de chemins. L'opération fournit aux hooks une identité stable pendant que routage et transport restent internes.

L'arbitrage **R-BEFORE-ORDER** privilégie l'inspection et la transformation du body avec refus avant accès PouchDB, au prix d'une lecture/parsing avant l'autorisation sémantique ; l'authentification précoce relève de l'hôte.

L'arbitrage **R-BULK-GET-FALLBACK** accepte deux routes de lecture supplémentaires pour conserver le fallback réel du client ; une réplication verte ne prouve pas que le trajet groupé performant fonctionne.

**R-GENERIC-HOOKS — Deux hooks génériques plutôt que forks spécialisés.** `before`/`after` couvrent autorisation, instrumentation, validation et filtrage sans multiplier les API dédiées. L'ancienne plomberie `req.locals`/`res.locals`, `onRequest`/`onResponse` et `skip*` n'est pas conservée.

**R-HOOK-CONTEXT** conserve un seul contexte mutable contrôlé par le routeur ; `committed`, état du transport irréversible, reste séparé de la réponse remplaçable. Ce choix évite à la fois un wrapper HTTP généraliste et une fausse promesse de modification des headers après heartbeat.

**R-BODY-LIMIT — Plafond configurable plutôt qu'absence de défaut ou alignement CouchDB.** D3 fait payer le parsing avant `before` ; un plafond fini protège les ressources. `1 MiB` et les défauts génériques parser/proxy risquent de bloquer `_bulk_docs`, alors que les 4 GiB autorisés pour une requête CouchDB ne conviennent pas à un body matérialisé en mémoire ; la limite CouchDB par document est une autre notion, hors pièces jointes. Le défaut révisable de `64 MiB` doit être confronté aux tailles réelles et à la mémoire sous concurrence. Un `413` peut bloquer la réplication jusqu'à ajustement ; réduire les lots ne résout pas un document isolé trop gros, d'où la qualification de fidélité dans R-DB-CREATION.

**R-CACHED-HANDLE — Un handle par base plutôt qu'un handle par requête.** Le modèle suit le lifecycle réel de PouchDB/LevelDB, évite les réallocations et les listeners persistants, sans ajouter eviction ou pool non justifiés.

**R-DB-LIFECYCLE / R-ROUTER-CLOSE — Événements et fermeture explicite plutôt que cache lié au processus.** `closed`/`destroyed` permettent d'écarter les handles devenus inutilisables ; une fermeture extérieure sans événement sur le handle du routeur ne peut pas être inférée. PouchDB conserve des ressources tant que ses handles restent ouverts : retirer le handler d'un processus vivant sans `router.close()` laisserait le cache privé sans propriétaire capable de les libérer. Ce besoin a remplacé l'orientation initiale « pas de fermeture publique en V1 », sans justifier LRU ni gestion globale du stockage. `router.close()` ne remplace pas `server.close()` ni la coordination des autres handles par l'hôte.

**R-DB-CREATION / R-DB-EXISTS-BOUNDARY — Vérité HTTP avec responsabilité bornée.** Ouvrir PouchDB pour découvrir une absence pendant `GET` peut créer la base et mentir sur l'état. Une simple liste des handles du routeur ignorerait les bases externes. Le prédicat applicatif d'existence, consulté avant construction, préserve `404`/`201`/`412` sur ce que le routeur peut connaître. L'alternative `acquireDatabase(name, {create})`, retournant handle et résultat typé, a été écartée en V1 : elle imposait aux intégrations de fournir leurs handles quand constructeur/preset plus prédicat suffisent. La vérification suivie de la construction n'est pas atomique face à des créateurs extérieurs ; le routeur coordonne ses propres appels et l'application son espace de stockage.

**R-NO-MULTIPART — JSON/raw plutôt que multipart historique.** La surface PouchDB courante ne justifie plus `multipart/related`; le conserver imposerait parser, fichiers temporaires et lifecycle supplémentaires.

**R-PROOF — Contrat d'abord, tests comme preuve.** La suite upstream, puis l'idée d'en retenir seulement les tests inchangés, ont été écartées comme gates : leur périmètre et leurs fixtures HTTP dépassent la V1, tandis que leur adaptation coûte cher pour une preuve limitée. La suite propre exerce la sync avec le vrai replicator ; des tests HTTP ciblés révèlent ce qu'une sync verte pourrait masquer. L'upstream reste une source de scénarios.

**R-PERF-METHOD — Benchmark décomposé plutôt qu'attribution globale.** Nos scénarios de sync remplacent la suite upstream comme charge commune. Lorsque la charge lit des révisions distantes, A→B mesure aussi le bénéfice de `_bulk_get` ; B→C compare le même handler sous deux enveloppes. Les chiffres Express/Next.js antérieurs motivent le reboot sans prouver son résultat.

## 9. Sources matérielles

**S-EXPLORATION — source primaire désignée.** `ChatGPT-EXPL_-_Specs.md`, conversation fournie le 2026-09-27, contient les repères A1 à A93, dont A93, estimation intermédiaire remplacée par D1. Elle mêle décisions humaines et propositions du modèle : seules les adoptions explicites, les conséquences nécessaires des éléments déjà établis ou les preuves indépendantes fondent le design courant. Les positions finales A89/A92 remplacent l'orientation API-first A72-A80.

**S-CODE — evidence technique rapportée dans S-EXPLORATION.** Code courant et historique de `jpbourgeon/pouchdb-http-router`/ancien nom, notamment routage, hooks, `_changes`, dépendances, harness et benchmarks. Le dépôt `main` ne contient pas encore les cinq documents de design cibles ni une Capture existante identifiée.

**S-POUCHDB — evidence amont rapportée dans S-EXPLORATION.** Replicator, adapter HTTP, preset/plugins standards et suites PouchDB courants ; ils fondent l'inventaire sync, les fallbacks, les limites de la suite et l'absence de besoin multipart actuel.

**S-UPSTREAM-AUDIT — preuve technique primaire acquise pour D1/D2.** `apache/pouchdb` au commit `27de91f1105a8074ddd06f5a23156dd99c4eb016` (https://github.com/apache/pouchdb/tree/27de91f1105a8074ddd06f5a23156dd99c4eb016), notamment `packages/node_modules/pouchdb-replication/src/{replicate.js,getDocs.js,sync.js}`, `pouchdb-adapter-http/src/index.js`, `pouchdb-checkpointer/src/index.js`, `pouchdb-generate-replication-id/src/index.js`, `pouchdb-changes-filter/src/{index.js,evalFilter.js,evalView.js}` et `pouchdb-utils/src/bulkGetShim.js`. Audit statique des émissions HTTP et des branches de repli, non validation d'un routeur V1 exécuté. Les notes de version primaires PouchDB 5.0.0 (https://pouchdb.com/2015/10/06/pouchdb-5.0.0-five-years-of-pouchdb.html), 5.2.0 (https://pouchdb.com/2016/01/13/pouchdb-5.2.0-a-better-build-system-with-rollup.html) et 6.1.1 (https://pouchdb.com/2017/01/05/pouchdb-6.1.1.html) fondent la chronologie du fallback. Dans la discussion présente, l'utilisateur a validé le 2026-09-27 le choix d'inclure les deux routes de repli ; cette preuve technique n'institue pas, à elle seule, le vocabulaire D2.

**S-BODY-LIMIT — validation utilisateur et preuves techniques, le 2026-09-27.** L'utilisateur a retenu le plafond par défaut de `64 MiB`, raisonnable faute de données contraires et révisable, après avoir écarté l'absence de plafond. S-UPSTREAM-AUDIT (`replicate.js`, `pouchdb-adapter-http/src/index.js`) étaye les lots de 100 et `_bulk_docs` JSON avec attachments en base64. Les documentations CouchDB (https://docs.couchdb.org/en/stable/config/http.html#max-http-request-size ; https://docs.couchdb.org/en/stable/config/couchdb.html#max-document-size), Express (https://expressjs.com/en/resources/middleware/body-parser/) et nginx (https://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size) distinguent plafonds de requête, de document, de parser et de proxy ; aucun n'institue la valeur V1.

**S-DB-CREATION — validation utilisateur et essais techniques, le 2026-09-28.** L'utilisateur a validé la fidélité de la sync dans la responsabilité du routeur, les statuts d'une base absente et la signature `PouchDB` + `databaseExists`. Des essais isolés montrent que `createIfMissing:false` n'empêche pas la création en mémoire, qu'une ouverture LevelDB absente n'offre qu'une `OpenError` non typée et qu'`errorIfExists:true` peut être trompé par un store déjà ouvert. Un prototype mémoire a obtenu les statuts attendus sur les bases connues, mais un registre limité aux créations internes a mal classé une base créée à l'extérieur : l'application doit connaître son espace de noms. Ces essais n'ont pas validé le futur routeur. Code PouchDB : S-UPSTREAM-AUDIT (`pouchdb-adapter-http/src/index.js`, `pouchdb-adapter-leveldb-core/src/index.js`) ; API (https://pouchdb.com/api.html) et protocole (https://docs.couchdb.org/en/stable/replication/protocol.html) étayent le dialogue sans instituer la décision.

**S-DB-LIFECYCLE — validation utilisateur et preuves techniques, le 2026-09-28.** L'utilisateur a validé l'éviction sur `closed`/`destroyed` et `router.close()` asynchrone en V1. S-UPSTREAM-AUDIT (`pouchdb-core/src/{constructor.js,setup.js,adapter.js}`, `pouchdb-adapter-leveldb-core/src/index.js`) et la documentation PouchDB (https://pouchdb.com/api.html) étayent propagation des événements et libération des connexions/listeners par `db.close()`. Des essais bornés avec PouchDB 9/LevelDB ont observé la notification du handle caché après destruction externe et des lectures périmées ; fermer un autre handle partagé n'a pas produit `closed` sur celui du routeur. Ces observations ne sont pas généralisées à tous les adapters. `server.close()` (https://nodejs.org/api/http.html#serverclosecallback) ne ferme ni ces handles ni les longpolls actifs ; ces preuves motivent sans instituer la décision.

**S-D1 — validation utilisateur le 2026-09-27.** L'utilisateur a validé les deux routes de repli, la table finale de treize formes et les filtres nommés/`_view` exécutés localement sur la source ; la sémantique de la base absente est décidée séparément dans S-DB-CREATION.

**S-D2 — validation utilisateur le 2026-09-27.** L'utilisateur a explicitement validé les identifiants d'opération et noms de paramètres ; S-HOOK-CONTEXT établit séparément le contexte complet.

**S-D3 — validation utilisateur le 2026-09-27.** L'utilisateur a validé le parsing avant `before` et le refus avant accès PouchDB pour la requête ; les limites de body et le contexte des hooks ont été décidés séparément.

**S-HOOK-CONTEXT — validation utilisateur le 2026-09-27.** L'utilisateur a validé le contexte partagé, les retours de `before`/`after`, `context.committed` distinct de `response` et les restrictions après engagement. La documentation `node:http` (https://nodejs.org/api/http.html#responseheaderssent ; https://nodejs.org/api/http.html#responsewritechunk-encoding-callback) étaye la frontière technique sans instituer ce contrat.

**S-TEST-COVERAGE — validation utilisateur et code upstream, le 2026-09-28.** L'utilisateur a validé notre suite de sync de bout en bout, les fixtures locales et les tests HTTP ciblés ; il a écarté les tests upstream inchangés comme gate supplémentaire. Au commit S-UPSTREAM-AUDIT, `TESTING.md`, `tests/integration/{test.replication.js,test.sync.js,utils.js}` et `bin/test-node.sh` montrent la préparation/fin de tests par des appels HTTP hors V1 ; `pouchdb-server/packages/node_modules/express-pouchdb/lib/index.js` montre que `minimumForPouchDB` inclut all-docs, compaction et vues. Ces constats motivent sans instituer le choix.

**S-BENCHMARK-MATRIX — validation utilisateur le 2026-09-28.** Après exploration des trois cibles, des trois charges issues de notre suite, de l'exclusion des fixtures et assertions du chronométrage et de l'alternance équilibrée des six ordres, l'utilisateur a explicitement validé Q-BENCHMARK-MATRIX et demandé sa Capture. La documentation Node du benchmark (https://nodejs.org/api/bench.html) étaye la conservation des échantillons bruts, l'échauffement et la séparation de la mesure et de son setup, sans imposer de runner particulier ni instituer cette décision. Les volumes à calibrer restent une question d'exécution, pas une question de design ouverte.

**S-COHERENCE — validation utilisateur de la réconciliation le 2026-09-28.** L'utilisateur a validé le contrôle de cohérence et demandé de qualifier la fidélité directe par les limites et interventions applicatives, de préserver deux vérifications matérielles (noms encodés et état du repli `_bulk_get`) et de réduire les répétitions sans changer les décisions. Sources techniques primaires au commit S-UPSTREAM-AUDIT : `pouchdb-adapter-http/src/index.js` mémorise le support de `_bulk_get` par URL ; dans `pouchdb-express-router` au commit `1b2e666d2fd791b8ec38518c239342316f6102cc`, `lib/routes/db.js` réencode `req.params.db` avant construction PouchDB. Ces constats appellent des tests ciblés sans établir une nouvelle règle de nommage ou modifier la matrice validée.

**S-REFERENCE — comparaison rapportée dans S-EXPLORATION.** `express-pouchdb`/`pouchdb-server` pour la surface historique, le cache de handles, `_changes`, le montage Express et les compromis du profil `minimumForPouchDB`. Cette référence éclaire le design mais n'a pas autorité pour élargir le contrat.

**S-BENCHMARK — état expérimental antérieur rapporté dans S-EXPLORATION.** Campagne et documentation du benchmark Express/Next.js. Elles instituent une méthode de comparaison reproductible et motivent la décomposition future, mais leurs valeurs ne prouvent pas les performances du reboot.
