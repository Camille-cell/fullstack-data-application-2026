# TP 2 — Séance 4 : brancher le backend sur PostgreSQL

## Ce que vous allez faire

À la fin du [TP 1](tp1-frontend.md), votre application a deux services : un frontend qui sert des
pages HTML et un backend qui garde tout en mémoire. Dans ce TP, vous ajoutez le troisième service,
**PostgreSQL**, et vous restructurez le backend en couches pour qu'il y lise et écrive ses
données.

Le frontend, lui, ne doit presque pas changer : il ne connaît que l'API. Chaque modification que
vous serez obligés d'y faire révélera un changement du contrat de l'API ; vous les noterez.

Ce TP est **guidé, pas dicté** : je vous dis quoi construire, dans quel ordre, et comment vérifier
que ça marche. Le code, c'est vous qui l'écrivez, en vous appuyant sur le
[cours 2](../cours/cours2-backend-db.md), qui vous donne tous les éléments nécessaires. Les seules commandes données telles quelles sont celles à lancer
dans le terminal.

## Objectifs du TP

À la fin de ce TP, vous devez savoir :

- structurer un backend FastAPI en couches (routers / services / repositories / models / schemas) ;
- configurer SQLAlchemy et injecter une session par requête ;
- créer, relire et corriger une migration Alembic ;
- externaliser toute la configuration en variables d'environnement ;
- orchestrer backend + frontend + base dans un seul `docker compose up`, et choisir comment les
  migrations s'y appliquent ;
- repérer un N+1 dans les logs SQL et le corriger ;
- écrire une verticale complète avec ses règles métier, et la relire avec une grille de review.

C'est le TP le plus long du module. **Ce que vous produisez ici est le socle de votre projet** :
soignez-le, vous allez vivre dedans pendant trois séances.

## Prérequis

- Les TP des séances 1 et 2 terminés.
- Le TP 1 de cette séance terminé : dépôt organisé en `backend/` et `frontend/`.
- Docker Desktop lancé.

---

## Étape 0 — L'arborescence cible

Repartez du dépôt du TP 1. Le dossier `frontend/` ne bouge pas ; c'est le dossier `backend/` qui
se restructure en couches. Voici la cible :

```text
gearshare/
├── backend/
│   ├── alembic/
│   ├── alembic.ini
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py
│   │   ├── core/
│   │   │   ├── config.py
│   │   │   └── exceptions.py
│   │   ├── db/
│   │   │   └── session.py
│   │   ├── api/
│   │   │   └── deps.py
│   │   ├── models/
│   │   ├── schemas/
│   │   ├── repositories/
│   │   ├── services/
│   │   └── routers/
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/          # celui du TP 1, inchangé
├── .env.example
├── .env               # ← dans le .gitignore
├── .gitignore
└── docker-compose.yml
```

Créez les dossiers vides, puis :

1. **Dépendances.** Ajoutez à `backend/requirements.txt` les quatre bibliothèques dont le backend
   a désormais besoin : SQLAlchemy 2.0, le driver PostgreSQL `psycopg` (version 3, avec son
   extra `binary`), Alembic et `pydantic-settings`. Fixez les versions majeures, comme pour
   FastAPI. Gardez `pytest` et `httpx`, ajoutés en séance 2.
2. **`.gitignore`.** Il doit au minimum exclure le `.env`, les caches Python et pytest, et votre
   environnement virtuel.

**Faites ce `.gitignore` maintenant.** Un `.env` commité en séance 4, c'est un secret exposé dans
l'historique Git jusqu'au rendu. Vérifiez-le :

```bash
echo "TEST=1" > .env
git status          # le .env ne doit PAS apparaître
```

---

## Étape 1 — Configuration et session

Trois fichiers à écrire :

- **`app/core/config.py`** : une classe de configuration `pydantic-settings` qui lit au moins
  l'URL de la base et un booléen d'affichage du SQL (désactivé par défaut). Elle est instanciée
  une seule fois, au niveau du module.
- **`app/db/session.py`** : l'engine SQLAlchemy et la fabrique de sessions, construits à partir de
  cette configuration. Activez la vérification des connexions mortes (`pool_pre_ping`).
- **`app/api/deps.py`** : la dépendance `get_db`, qui ouvre une session par requête, la valide si
  tout s'est bien passé, l'annule en cas d'exception, et la ferme **dans tous les cas**. Exposez
  aussi l'alias `DbSession` pour ne pas répéter `Depends(get_db)` partout.

Puis le fichier **`.env.example`**, commité, qui liste toutes les variables nécessaires avec des
valeurs de développement : les trois variables attendues par l'image `postgres`
(`POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`), `DATABASE_URL` au format
`postgresql+psycopg://utilisateur:motdepasse@hote:port/base`, et `SQL_ECHO`. Copiez-le :

```bash
cp .env.example .env
```

**Vérifiez votre compréhension** : quel hôte mettez-vous dans `DATABASE_URL`, `db` ou `localhost` ?
Pourquoi ? Si vous hésitez, relisez le cours de la séance 0 sur le réseau Docker.

**Point de contrôle** : dans `get_db`, que se passe-t-il si une route lève une exception après
avoir ajouté un objet à la session ? L'objet est-il en base ? Notez votre réponse, vous la
vérifierez à l'étape 4.

---

## Étape 2 — Les modèles

Écrivez `app/models/base.py`, puis `user.py`, `item.py` et `reservation.py`, en traduisant
**exactement** le schéma SQL de la séance 3. Le modèle `Item` du cours vous sert de référence pour
la syntaxe ; les deux autres sont à écrire entièrement.

Points à ne pas rater :

- `index=True` sur toutes les clés étrangères ;
- `Numeric(8, 2)` pour `tarif_jour` (jamais `Float` pour de l'argent) ;
- `DateTime(timezone=True)` avec une valeur par défaut côté serveur pour `created_at` ;
- l'unicité de `User.email` ;
- les contraintes `CHECK` de la séance 3 dans `__table_args__` (tarif positif, dates cohérentes,
  statut valide), **nommées** ;
- les `relationship(back_populates=...)` des deux côtés de chaque lien.

Enfin, faites en sorte que `app/models/__init__.py` importe `Base` et les trois modèles. Ce
fichier n'est pas cosmétique : relisez dans le cours la partie sur Alembic pour comprendre ce qui
se passe sans lui.

---

## Étape 3 — Alembic

Initialisez Alembic depuis `backend/` :

```bash
alembic init alembic
```

Puis branchez-le sur votre application, dans `alembic/env.py` :

- l'URL de connexion doit venir de votre configuration, **pas** de `alembic.ini` ;
- `target_metadata` doit pointer sur les métadonnées de votre `Base`, avec les modèles importés.

Dans `alembic.ini`, laissez `sqlalchemy.url` vide : ce fichier est commité, votre mot de passe n'a
rien à y faire.

Pour lancer Alembic dans un conteneur, ajoutez temporairement le service `db` à votre
`docker-compose.yml` (vous le compléterez à l'étape 6 ; le cours en donne un modèle).
Démarrez la base seule, puis générez la migration :

```bash
docker compose up -d db
docker compose run --rm backend alembic revision --autogenerate -m "schema initial"
```

> 💡 Si vous exécutez Alembic depuis votre machine plutôt que dans le conteneur, `DATABASE_URL`
> doit pointer sur `localhost` et non `db`, et le port de la base doit être publié. Le plus simple
> est de tout faire dans le conteneur.

**Ouvrez le fichier généré dans `alembic/versions/` et lisez-le en entier.** Vérifiez :

- [ ] les trois tables sont créées dans le bon ordre (`users` avant `items`) ;
- [ ] les clés étrangères sont présentes, avec le bon `ondelete` ;
- [ ] les contraintes `CHECK` sont là, avec leurs noms ;
- [ ] les index sur les clés étrangères sont créés ;
- [ ] le `downgrade()` supprime bien tout, dans l'ordre inverse.

S'il manque quelque chose, ne corrigez pas la migration : corrigez le **modèle**, supprimez le
fichier, régénérez.

Appliquez, inspectez, puis testez l'aller-retour :

```bash
docker compose run --rm backend alembic upgrade head
docker compose exec db psql -U gearshare -d gearshare -c "\dt"
docker compose exec db psql -U gearshare -d gearshare -c "\d items"
docker compose run --rm backend alembic downgrade base
docker compose run --rm backend alembic upgrade head
```

Le cycle complet doit passer sans erreur. Si `downgrade` échoue, corrigez la migration maintenant :
une migration non réversible est un problème que vous traînerez tout le projet.

### 3bis — Le piège du renommage

Exercice court mais important. Insérez d'abord deux items en base avec `psql` (il vous faudra un
utilisateur pour la clé étrangère). Puis renommez, dans le modèle `Item`, la colonne `tarif_jour`
en `prix_jour`, et générez :

```bash
docker compose run --rm backend alembic revision --autogenerate -m "renommage tarif_jour"
```

Lisez la migration générée. Que fait-elle exactement ? Que deviendraient vos deux items si vous
l'appliquiez ? Notez votre prédiction, puis **corrigez la migration à la main** pour qu'elle
renomme la colonne sans perte (le cours vous donne l'opération Alembic à utiliser).
N'oubliez pas le `downgrade()`.

Appliquez, et vérifiez que les tarifs sont toujours là :

```bash
docker compose run --rm backend alembic upgrade head
docker compose exec db psql -U gearshare -d gearshare -c "SELECT id, titre, prix_jour FROM items;"
```

Puis **revenez en arrière** (`downgrade -1`, supprimez le fichier de migration, remettez
`tarif_jour` dans le modèle) : c'était un exercice, on garde le nom initial.

Consignez dans `docs/review-migrations.md` ce que vous avez observé : la migration générée, votre
prédiction, la correction. C'est un livrable.

---

## Étape 4 — Une verticale complète : `items`

Écrivez la ressource `items` couche par couche, **dans cet ordre**, en vérifiant chaque fichier
avant de passer au suivant :

1. **`app/core/exceptions.py`** : une exception métier de base, et une sous-classe par famille
   d'erreur — ressource introuvable, conflit métier, accès refusé. Elles transportent un message
   lisible. Aucune ne connaît HTTP.
2. **`app/schemas/item.py`** : un schéma de création, un schéma de modification (tous les champs
   optionnels) et un schéma de lecture, construit depuis un objet SQLAlchemy.
3. **`app/repositories/item.py`** : une classe `ItemRepository` qui sait récupérer un item,
   lister (avec pagination, recherche texte et filtre sur la disponibilité), ajouter et supprimer.
   **Aucun `commit()`** : relisez pourquoi dans le cours.
4. **`app/services/item.py`** : une classe `ItemService` qui s'appuie sur le repository et lève
   l'exception « introuvable » quand l'item n'existe pas. Pas d'autre règle pour l'instant
   (l'ownership viendra en séance 5).
5. **`app/routers/items.py`** : les cinq routes du CRUD, avec `response_model` et les codes de
   statut de la séance 1. Le router ne fait que traduire : schéma en entrée, appel au service,
   schéma en sortie.

Dans `app/main.py`, incluez le router, gardez la route `/health`, et branchez la traduction des
exceptions métier en codes HTTP. À vous de choisir entre un `try`/`except` dans chaque
route et un gestionnaire d'exceptions global. Justifiez ce choix en une
phrase dans `docs/review-migrations.md`.

**Vérifiez** :

```bash
docker compose up -d --build
curl -X POST http://localhost:8000/items \
  -H "Content-Type: application/json" \
  -d '{"owner_id": 1, "titre": "Vélo de ville", "tarif_jour": 8.5}'

curl http://localhost:8000/items
curl -i http://localhost:8000/items/9999          # attendu : 404 avec {"detail": "..."}
docker compose exec db psql -U gearshare -d gearshare -c "SELECT * FROM items;"
```

La dernière commande est celle qui compte : **la donnée est-elle réellement en base ?** Redémarrez
ensuite tout et refaites un `GET /items` : les données doivent survivre.

```bash
docker compose restart
curl http://localhost:8000/items
```

Revenez enfin sur le point de contrôle de l'étape 1 : provoquez une erreur après un ajout (par
exemple un `POST` avec un `owner_id` qui n'existe pas) et vérifiez en base que rien n'a été écrit.
Quel code HTTP obtenez-vous ? Est-ce le bon ? Si c'est une `500`, demandez-vous quelle couche
devrait détecter ce cas.

---

## Étape 5 — Débusquer un N+1

Mettez `SQL_ECHO=true` dans votre `.env` et redémarrez le backend.

Faites évoluer la représentation d'un item pour qu'elle inclue son propriétaire (la page liste du
frontend pourra l'afficher). Pas de nouvelle route : c'est la ressource `items` qui s'enrichit, et
`GET /items` renvoie désormais un champ `owner` en plus.

Pour cela, créez un petit schéma de lecture du propriétaire, limité à son identifiant et à son nom
affiché, et ajoutez-le comme champ du schéma de lecture d'un item. Ce schéma ne doit contenir **ni
`email` ni `password_hash`** : n'exposez jamais un utilisateur complet dans la représentation
d'une autre ressource.

Insérez une vingtaine d'items en une seule commande SQL (indice : `INSERT ... SELECT` avec
`generate_series`), appelez `GET /items?limit=50`, et **comptez les requêtes SQL dans les logs** :

```bash
curl -s "http://localhost:8000/items?limit=50" > /dev/null
docker compose logs --since 30s backend | grep -c "SELECT"
```

Combien de requêtes pour une seule page ? D'où viennent-elles ? Le N+1 est ici caché dans la
sérialisation : trouvez quelle ligne de votre code le déclenche, sans que vous ayez écrit de
boucle. Corrigez-le dans la méthode de liste de `ItemRepository` (le cours vous donne l'option de
chargement à utiliser), puis recomptez.

Notez les deux chiffres, avant et après, dans `docs/review-migrations.md` (ou un
`docs/notes-perf.md`).

> N'oubliez pas de remettre `SQL_ECHO=false` avant de rendre : les logs SQL en continu sont
> inexploitables, et en production ils peuvent contenir des données personnelles.

---

## Étape 6 — Compose à trois services et migrations au démarrage

### 6.1. Les trois services

Complétez le `docker-compose.yml` :

- le service `db` reçoit ses identifiants par substitution depuis le `.env`, stocke ses données
  dans un volume nommé, et expose un `healthcheck` ;
- le service `backend` reçoit `env_file: .env` et n'attend pas seulement que la base soit
  démarrée, mais qu'elle soit **prête** ;
- le service `frontend` reste celui du TP 1 : **pas de `env_file`**, il n'a besoin que de
  `API_BASE_URL` (relisez pourquoi dans le [cours 1](../cours/cours1-frontend.md)).

### 6.2. Votre choix : comment appliquer les migrations ?

Jusqu'ici, vous lanciez `alembic upgrade head` à la main. Sur votre rendu, je ferai `git clone`,
`cp .env.example .env`, `docker compose up`, et rien d'autre : les migrations doivent donc
s'appliquer **toutes seules**, avant que l'API ne reçoive sa première requête.

Le cours présente deux approches. **C'est à votre groupe de choisir**, et de justifier :

| | Option 1 : `entrypoint.sh` | Option 2 : service `migrate` |
|---|---|---|
| Où vit la commande | Dans l'image du backend, exécutée avant `uvicorn` | Dans un service Compose dédié, construit depuis la même image |
| Ce qui attend quoi | Le backend attend la base | `migrate` attend la base, le backend attend la fin de `migrate` |
| Ce que vous devez écrire | Un script shell, et le `Dockerfile` qui l'utilise | Un service de plus, et la bonne `condition` dans `depends_on` |

Avant de trancher, répondez par écrit, dans `docs/choix-migrations.md`, à ces questions :

1. Que se passe-t-il au **deuxième** `docker compose up`, quand la base est déjà à jour ?
2. Si une migration échoue, que voit-on dans `docker compose ps` et dans les logs ? Le backend
   démarre-t-il quand même ?
3. En production, que se passerait-il si trois instances du backend démarraient en même temps
   sur une base pas encore migrée ? (Inutile de le tester en local : le port `8000` publié
   empêche de lancer plusieurs instances.)
4. Votre backend tourne en `--reload` pendant le développement : votre option fonctionne-t-elle
   toujours ?

Puis écrivez votre choix et sa justification en quelques lignes. Il n'y a pas de mauvaise
réponse, il y a des réponses non justifiées.

### 6.3. Vérifiez depuis zéro

```bash
docker compose down -v          # ⚠️ supprime aussi le volume : la base repart vide
docker compose up --build
```

Un seul `up` doit : démarrer Postgres, attendre qu'il soit prêt, appliquer les migrations, démarrer
l'API, démarrer le frontend. Regardez l'ordre des lignes dans les logs pour le confirmer, puis :

```bash
docker compose ps -a
docker compose exec db psql -U gearshare -d gearshare -c "SELECT * FROM alembic_version;"
```

Si vous devez lancer plusieurs commandes dans le bon ordre pour que ça marche, ce n'est pas fini.

### 6.4. Le frontend sur la vraie base

Ouvrez le frontend sur `http://localhost:8080` et refaites le parcours du TP 1 : liste, détail,
création d'un item.

- La **liste** et le **détail** fonctionnent sans modification du frontend : le contrat de lecture
  n'a pas changé (ou presque : regardez comment s'affiche `tarif_jour`, devenu un `Numeric`).
- La **création** échoue avec un message « owner_id : Field required ». Le contrat a changé : un
  item a désormais un propriétaire. Votre frontend du TP 1 affiche proprement l'erreur `422` du
  backend, c'est exactement ce qu'on attendait de lui.

Corrigez de façon **provisoire** : insérez un utilisateur de démonstration en base, puis faites
envoyer `owner_id` par le frontend, en le fixant dans la route de création.

```bash
docker compose exec db psql -U gearshare -d gearshare \
  -c "INSERT INTO users (email, display_name, password_hash) VALUES ('demo@esiee.fr', 'Démo', 'x');"
```

Marquez cette valeur d'un commentaire `# TODO séance 5` : en séance 5, le propriétaire sera
l'utilisateur connecté, déduit du jeton par le backend, et ce champ disparaîtra du formulaire.

Enfin, redémarrez tout (`docker compose restart`) et rechargez la liste : **les items sont toujours
là**. C'est la différence avec l'étape 6 du TP 1.

Notez dans `docs/review-migrations.md` chaque modification que vous avez dû faire dans
`frontend/`, et le changement de contrat qui l'a provoquée.

---

## Étape 7 — La verticale `reservations`

Vous avez construit `items` couche par couche. Construisez maintenant `reservations` de la même
façon, en suivant **exactement** l'architecture de `items` : mêmes conventions de nommage, même
découpage, mêmes exceptions métier. Le modèle `Reservation` existe déjà (étape 2).

### 7.1. Le contrat

Routes attendues :

| Méthode et chemin | Rôle | Codes attendus |
|---|---|---|
| `POST /reservations` | Créer une réservation | `201`, `404`, `409`, `422` |
| `GET /reservations` | Lister, filtrable par `item_id`, paginé | `200` |
| `GET /reservations/{id}` | Détail | `200`, `404` |
| `PATCH /reservations/{id}` | Annuler, avec le corps `{"statut": "annulee"}` | `200`, `404`, `409` |

### 7.2. Les règles métier

Elles vivent toutes dans le service, et nulle part ailleurs :

- l'item doit exister, sinon « introuvable » ;
- l'item doit être disponible, sinon « conflit » ;
- aucune réservation **active** de cet item ne doit chevaucher le créneau demandé, sinon
  « conflit » ;
- on ne peut annuler qu'une réservation au statut `active`, sinon « conflit ».

La détection de chevauchement est une requête : elle va donc dans le repository, et le service
l'appelle. Avant de l'écrire, dessinez sur papier les cinq cas de figure entre un créneau existant
et un créneau demandé : **avant**, **après**, **chevauchement partiel au début**, **chevauchement
partiel à la fin**, **englobant**. Puis trouvez la condition la plus simple qui renvoie « vrai »
exactement pour les trois cas de conflit. Indice : elle tient en deux comparaisons, et il est plus
facile de raisonner sur les cas où il n'y a **pas** de conflit.

Décidez aussi, et notez-le : deux réservations jointives (l'une finit le jour où l'autre commence)
sont-elles en conflit ? Votre requête doit refléter cette décision.

### 7.3. Vérifiez

Écrivez dans `docs/cas-tests-chevauchement.md` les cinq cas de figure (plus le cas jointif), avec
pour chacun le créneau existant, le créneau demandé et le code attendu. Puis rejouez-les un par un
depuis `http://localhost:8000/docs` ou avec `curl`, et notez le résultat obtenu. Ce fichier est
votre plan de tests pour le service : le cours 2 montre comment l'automatiser avec
`parametrize` sur la base de test.

Contrôlez aussi la migration : si votre modèle `Reservation` a évolué depuis le schéma initial,
générez une migration, relisez-la et testez son `downgrade()`.

### 7.4. Relisez

Échangez votre verticale `reservations` avec un autre binôme de votre groupe (ou relisez-la vous-
même le lendemain, à froid), et remplissez `docs/review-couches.md` :

| Point de contrôle | OK / KO | Ce que j'ai corrigé |
|---|---|---|
| Aucun SQL ni requête SQLAlchemy dans le router | | |
| Aucune `HTTPException` dans le service (uniquement les exceptions métier) | | |
| Aucun `commit()` dans le repository | | |
| `response_model` sur les 4 routes | | |
| Schéma de lecture construit depuis l'objet SQLAlchemy, sans champ interne | | |
| La requête de chevauchement couvre les cinq cas, y compris l'englobant | | |
| Pas de f-string dans une requête SQL | | |
| Pas de N+1 sur `GET /reservations` (si la relation `item` est chargée) | | |
| Migration relue, `downgrade()` testé | | |
| Aucune dépendance ajoutée sans justification | | |

Sur la requête de chevauchement, soyez exigeants : c'est l'erreur la plus fréquente de ce TP. Une
condition qui teste seulement « le début demandé tombe dans un créneau existant » rate le cas où
la nouvelle réservation **englobe** complètement une réservation existante.

### 7.5. Le frontend

La page de détail d'un item, écrite au TP 1, doit toujours permettre de réserver et d'annuler.
Mais le backend renvoie maintenant `409` pour un créneau déjà pris ou un item indisponible, et
votre frontend ne savait gérer que le `422` : ajoutez le `409` aux erreurs affichées sur le
formulaire de réservation (le cours 1 montre comment). Vérifiez dans le navigateur
qu'une réservation en conflit affiche un message clair, et non une page d'erreur.

---

## Ce que vous devez me rendre à la fin du TP 2

Sur le dépôt Git de votre projet :

1. **`docker compose up` fonctionne depuis un clone propre** : `git clone`,
   `cp .env.example .env`, `docker compose up` → API sur `:8000`, frontend sur `:8080`, base
   migrée automatiquement et accessible.
2. **Le backend en couches** : `models/`, `schemas/`, `repositories/`, `services/`, `routers/`,
   avec au moins les ressources `items` et `reservations` complètes.
3. **Les migrations Alembic** dans `alembic/versions/`, avec un `downgrade()` testé.
4. **`.env.example` commité, `.env` absent du dépôt.** Je vérifierai l'historique Git.
5. **Le frontend du TP 1 branché sur la base** : liste, détail, création d'un item, réservation et
   annulation, avec des données qui survivent à un redémarrage.
6. **`docs/choix-migrations.md`** : vos réponses aux quatre questions de l'étape 6.2, votre choix
   (`entrypoint.sh` ou service `migrate`) et sa justification.
7. **`docs/review-couches.md`** : la review de la verticale `reservations`, avec les corrections
   apportées.
8. **`docs/review-migrations.md`** : ce que vous avez observé sur le renommage de colonne, votre
   choix de traduction des exceptions, le N+1 (les deux chiffres avant/après), et les
   modifications imposées au frontend par les changements de contrat.
9. **`docs/cas-tests-chevauchement.md`** : les cas de chevauchement, le code attendu et le
   résultat obtenu.

## Pour aller plus loin

- **Branchez votre suite de la séance 2 sur la base de test** : créez `gearshare_test`, ajoutez les
  fixtures `engine`, `db` et `client` du cours 2, remplacez la remise à zéro en
  mémoire par la transaction annulée, et vérifiez que vos tests de contrat de la séance 2 passent
  **sans modification**. Puis automatisez `docs/cas-tests-chevauchement.md` en un test paramétré
  sur `ReservationService`. C'est le premier chantier de la séance 5, autant prendre de l'avance.
- Implémentez **l'autre** option de migration que celle choisie à l'étape 6, sur une branche, et
  comparez le comportement sur les quatre questions.
- Écrivez un script `seed.py` qui peuple la base de données de démonstration, et branchez-le sur
  une commande `docker compose run --rm backend python -m app.seed`.
- Mettez en place un middleware qui logue la durée de chaque requête, et repérez la route la plus
  lente.
