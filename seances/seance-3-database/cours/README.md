# Cours — Séance 3 : PostgreSQL et le modèle relationnel (2h)

L'API de la séance 1 a un défaut rédhibitoire : elle oublie tout à chaque redémarrage. Cette
séance est consacrée à la **persistance** — et plus précisément à PostgreSQL, la base de données
imposée pour votre projet.

Aujourd'hui, on fait du SQL « à la main », sans ORM. C'est volontaire : vous ne pourrez pas
reviewer sérieusement le SQL qu'un ORM (ou un agent) génère à votre place si vous ne savez pas
lire du SQL. L'ORM arrive en séance 4.

- [Cours — Séance 3 : PostgreSQL et le modèle relationnel (2h)](#cours--séance-3--postgresql-et-le-modèle-relationnel-2h)
  - [Objectifs pédagogiques](#objectifs-pédagogiques)
  - [1. Pourquoi une base de données ?](#1-pourquoi-une-base-de-données-)
  - [2. Le modèle relationnel](#2-le-modèle-relationnel)
    - [2.1. Tables, lignes, colonnes](#21-tables-lignes-colonnes)
    - [2.2. Clé primaire](#22-clé-primaire)
    - [2.3. Clé étrangère et intégrité référentielle](#23-clé-étrangère-et-intégrité-référentielle)
    - [2.4. Cardinalités](#24-cardinalités)
    - [2.5. Un peu de normalisation](#25-un-peu-de-normalisation)
  - [3. Les types PostgreSQL utiles](#3-les-types-postgresql-utiles)
  - [4. SQL : définir le schéma (DDL)](#4-sql--définir-le-schéma-ddl)
  - [5. SQL : manipuler les données (DML)](#5-sql--manipuler-les-données-dml)
  - [6. SQL : interroger](#6-sql--interroger)
    - [6.1. SELECT, WHERE, ORDER BY](#61-select-where-order-by)
    - [6.2. Les jointures](#62-les-jointures)
    - [6.3. Agrégats](#63-agrégats)
  - [7. Transactions et ACID](#7-transactions-et-acid)
  - [8. Index et performances](#8-index-et-performances)
  - [9. Fonctions, procédures stockées et triggers](#9-fonctions-procédures-stockées-et-triggers)
    - [9.1. Fonction ou procédure ?](#91-fonction-ou-procédure-)
    - [9.2. Exemple : `created_at` et `updated_at` gérés par la base](#92-exemple--created_at-et-updated_at-gérés-par-la-base)
    - [9.3. Code en base : avec parcimonie](#93-code-en-base--avec-parcimonie)
  - [10. PostgreSQL dans Docker Compose](#10-postgresql-dans-docker-compose)
  - [11. Synthèse](#11-synthèse)

## Objectifs pédagogiques

À la fin de ce cours, vous devez être capables de :

- expliquer ce qu'apporte une base relationnelle par rapport à un stockage naïf ;
- concevoir un schéma correct : clés primaires, clés étrangères, contraintes, cardinalités ;
- écrire du SQL de création, d'insertion, de mise à jour et d'interrogation, jointures comprises ;
- expliquer ce qu'est une transaction et pourquoi ACID compte ;
- savoir quand créer un index, et lire un `EXPLAIN ANALYZE` ;
- distinguer fonction et procédure stockée, et lire un trigger ;
- lancer PostgreSQL avec Docker Compose et vous y connecter avec `psql`.

## 1. Pourquoi une base de données ?

Reprenons le `FAKE_DB: dict[int, dict]` de la séance 1. Ses problèmes :

| Problème | Conséquence |
|---|---|
| En mémoire | Tout est perdu au redémarrage du conteneur |
| Mono-processus | Deux instances de l'API n'ont pas les mêmes données |
| Pas de contrainte | Rien n'empêche deux utilisateurs avec le même email |
| Pas d'atomicité | Un crash au milieu d'une opération laisse un état incohérent |
| Recherche linéaire | Trouver un item par titre parcourt tout le dictionnaire |
| Pas de concurrence | Deux requêtes simultanées peuvent réserver le même créneau |

Un **SGBD** (Système de Gestion de Base de Données) résout tout cela d'un coup. Un SGBD
relationnel, en particulier, apporte quatre choses :

1. **la persistance** — les données survivent au processus ;
2. **l'intégrité** — le SGBD refuse les données incohérentes, même si votre code a un bug ;
3. **la concurrence** — plusieurs clients écrivent en même temps sans se corrompre ;
4. **l'interrogation déclarative** — vous décrivez *ce que* vous voulez, le moteur décide *comment*
   l'obtenir.

Le point 2 est celui qu'on sous-estime le plus. Une contrainte `UNIQUE` sur `users.email` est une
garantie **absolue** : aucun bug applicatif, aucune requête concurrente, aucun script d'import
mal écrit ne créera un doublon. Une vérification en Python (`if User.query.filter_by(email=...)`)
n'offre aucune de ces garanties, parce que deux requêtes simultanées peuvent passer le test toutes
les deux avant qu'aucune n'ait inséré.

> Pour votre projet : **mettez les invariants dans la base**, pas seulement dans le code Python.
> Le code Python vous sert à produire un beau message d'erreur ; la base vous sert à garantir
> qu'on ne peut pas se tromper.

Une note sur le NoSQL, que vous croiserez : MongoDB, Redis, Elasticsearch répondent à d'autres
besoins (schéma très mouvant, cache, recherche plein texte). Ils n'offrent généralement pas
l'intégrité référentielle ni les transactions multi-documents avec la même rigueur. Pour une
application métier structurée comme la vôtre — utilisateurs, matériel, réservations, avec des
relations fortes entre eux — **le relationnel est le bon choix par défaut**, et c'est celui qui
est imposé.

## 2. Le modèle relationnel

Formalisé par **Edgar F. Codd** chez IBM en 1970, le modèle relationnel repose sur une idée
simple : représenter les données sous forme de relations (des tables), et les manipuler avec une
algèbre bien définie. C'est ce qui a donné SQL, et c'est resté le socle de l'industrie depuis
cinquante ans.

### 2.1. Tables, lignes, colonnes

Une **table** représente un type d'entité. Chaque **ligne** (ou *tuple*) est une occurrence,
chaque **colonne** (ou *attribut*) est une propriété typée.

```text
             table "items"
┌────┬──────────────┬────────────┬───────────┐
│ id │ titre        │ tarif_jour │ owner_id  │  ← colonnes
├────┼──────────────┼────────────┼───────────┤
│  1 │ Vélo         │       8.50 │         3 │  ← ligne
│  2 │ Perceuse     │       5.00 │         1 │
└────┴──────────────┴────────────┴───────────┘
```

### 2.2. Clé primaire

La **clé primaire** identifie de façon unique et immuable une ligne. Elle est obligatoire, non
nulle, unique.

Utilisez un identifiant technique auto-incrémenté (`id`) plutôt qu'une donnée métier. L'email d'un
utilisateur semble unique — jusqu'au jour où quelqu'un veut le changer, et où toutes les lignes qui
le référencent doivent être mises à jour.

```sql
CREATE TABLE users (
    id    BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE
);
```

`BIGSERIAL` = entier 64 bits auto-incrémenté. Sur PostgreSQL récent, la forme moderne équivalente
est `BIGINT GENERATED ALWAYS AS IDENTITY` ; les deux sont acceptées, `BIGSERIAL` reste la plus
courante et c'est ce que génèrent la plupart des ORM.

### 2.3. Clé étrangère et intégrité référentielle

Une **clé étrangère** matérialise un lien vers une autre table, et le SGBD garantit que ce lien
pointe toujours vers quelque chose d'existant.

```sql
CREATE TABLE items (
    id         BIGSERIAL PRIMARY KEY,
    titre      VARCHAR(120) NOT NULL,
    tarif_jour NUMERIC(8, 2) NOT NULL CHECK (tarif_jour > 0),
    owner_id   BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE
);
```

Désormais :

- impossible d'insérer un item avec `owner_id = 999` si l'utilisateur 999 n'existe pas ;
- `ON DELETE CASCADE` : supprimer un utilisateur supprime automatiquement ses items.

Le comportement `ON DELETE` est un **choix métier** que vous devez faire consciemment :

| Clause | Effet à la suppression du parent |
|---|---|
| `ON DELETE RESTRICT` (défaut) | Refuse la suppression tant qu'il reste des enfants |
| `ON DELETE CASCADE` | Supprime aussi les enfants |
| `ON DELETE SET NULL` | Met la clé étrangère à `NULL` (la colonne doit être nullable) |

Pour GearShare : supprimer un utilisateur doit-il supprimer ses réservations passées, ou faut-il
les conserver pour l'historique ? Il n'y a pas de bonne réponse universelle — il y a une réponse
que vous devez **décider et documenter**. C'est le genre de choix qu'un agent fera à votre place,
silencieusement, si vous ne le spécifiez pas.

### 2.4. Cardinalités

| Relation | Comment on la modélise |
|---|---|
| **1–N** (un user a N items) | Clé étrangère `owner_id` dans la table `items` |
| **N–N** (un item a N tags, un tag est sur N items) | Table de liaison `item_tags(item_id, tag_id)` |
| **1–1** (un user a un profil) | Clé étrangère `UNIQUE` dans l'une des deux tables |

Une table de liaison :

```sql
CREATE TABLE item_tags (
    item_id BIGINT NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    tag_id  BIGINT NOT NULL REFERENCES tags(id)  ON DELETE CASCADE,
    PRIMARY KEY (item_id, tag_id)
);
```

La clé primaire **composite** `(item_id, tag_id)` garantit qu'un tag n'est associé qu'une fois au
même item.

### 2.5. Un peu de normalisation

La normalisation consiste à éliminer les redondances pour éviter les incohérences. Sans entrer
dans la théorie complète, retenez deux réflexes :

- **Ne stockez jamais deux fois la même information.** Si le nom du propriétaire apparaît à la fois
  dans `users.nom` et dans `items.owner_nom`, les deux finiront par diverger. Stockez `owner_id`
  et faites une jointure.
- **Une colonne contient une valeur atomique.** `tags VARCHAR(255)` contenant `"velo,sport,ete"`
  est ingérable : vous ne pouvez ni indexer, ni chercher correctement, ni garantir la cohérence.
  Faites une table.

La dénormalisation volontaire (dupliquer pour aller plus vite) existe, mais c'est une optimisation
qu'on fait **après** avoir mesuré un problème, pas par défaut.

## 3. Les types PostgreSQL utiles

| Type | Usage | Remarque |
|---|---|---|
| `BIGSERIAL` / `BIGINT` | Identifiants | `BIGSERIAL` pour la clé primaire |
| `VARCHAR(n)` / `TEXT` | Chaînes | `TEXT` sans limite ; `VARCHAR(n)` documente une contrainte |
| `NUMERIC(p, s)` | **Montants** | Exact. À utiliser pour tout ce qui est argent |
| `REAL` / `DOUBLE PRECISION` | Mesures scientifiques | **Jamais pour de l'argent** (arrondis binaires) |
| `BOOLEAN` | Vrai/faux | `TRUE` / `FALSE` |
| `DATE` | Date sans heure | |
| `TIMESTAMPTZ` | Date + heure **avec fuseau** | À préférer systématiquement à `TIMESTAMP` |
| `JSONB` | Données semi-structurées | Indexable ; à réserver aux cas réellement variables |
| `UUID` | Identifiant global | Utile si les ids doivent être imprévisibles |

Deux pièges classiques :

1. **`FLOAT` pour des euros.** `0.1 + 0.2 != 0.3` en binaire. Un tarif journalier, un total de
   facture, un solde : `NUMERIC`.
2. **`TIMESTAMP` sans fuseau.** Vos utilisateurs, votre serveur et votre base peuvent être dans des
   fuseaux différents, et l'heure d'été existe. `TIMESTAMPTZ` stocke en UTC et convertit ; c'est le
   seul choix raisonnable.

## 4. SQL : définir le schéma (DDL)

Le **DDL** (*Data Definition Language*) décrit la structure.

```sql
CREATE TABLE users (
    id            BIGSERIAL PRIMARY KEY,
    email         VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    display_name  VARCHAR(80)  NOT NULL,
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE TABLE items (
    id          BIGSERIAL PRIMARY KEY,
    owner_id    BIGINT       NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    titre       VARCHAR(120) NOT NULL,
    description TEXT,
    tarif_jour  NUMERIC(8, 2) NOT NULL CHECK (tarif_jour > 0),
    disponible  BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE TABLE reservations (
    id          BIGSERIAL PRIMARY KEY,
    item_id     BIGINT NOT NULL REFERENCES items(id) ON DELETE CASCADE,
    borrower_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    date_debut  DATE   NOT NULL,
    date_fin    DATE   NOT NULL,
    statut      VARCHAR(20) NOT NULL DEFAULT 'active',
    CONSTRAINT dates_coherentes CHECK (date_fin > date_debut),
    CONSTRAINT statut_valide    CHECK (statut IN ('active', 'annulee', 'terminee'))
);
```

Observez tout ce que ce schéma **interdit** :

- un item sans propriétaire (`NOT NULL` + `REFERENCES`) ;
- un tarif négatif ou nul (`CHECK`) ;
- deux comptes avec le même email (`UNIQUE`) ;
- une réservation qui se termine avant de commencer (`CHECK` sur deux colonnes) ;
- un statut fantaisiste (`CHECK ... IN`).

Chacune de ces contraintes est un bug que vous n'aurez jamais, quelle que soit la qualité du code
applicatif. **Nommez vos contraintes** (`CONSTRAINT dates_coherentes ...`) : quand elles se
déclenchent, le message d'erreur devient exploitable, et vous pourrez le traduire en `409` propre.

Modifier une table existante :

```sql
ALTER TABLE items ADD COLUMN categorie VARCHAR(50);
ALTER TABLE items ALTER COLUMN description SET NOT NULL;
ALTER TABLE items DROP COLUMN categorie;
```

⚠️ En production, on ne fait pas d'`ALTER TABLE` à la main : on utilise des **migrations
versionnées**. C'est le sujet de la séance 4 (Alembic).

## 5. SQL : manipuler les données (DML)

```sql
-- Insertion, avec récupération de l'id généré
INSERT INTO users (email, password_hash, display_name)
VALUES ('alice@esiee.fr', '$pbkdf2-sha256$...', 'Alice')
RETURNING id;

-- Insertion multiple
INSERT INTO items (owner_id, titre, tarif_jour) VALUES
    (1, 'Vélo de ville', 8.50),
    (1, 'Perceuse', 5.00),
    (2, 'Tente 2 places', 12.00);

-- Mise à jour
UPDATE items SET disponible = FALSE WHERE id = 2;

-- Suppression
DELETE FROM items WHERE id = 3;
```

Le `RETURNING` est une spécificité PostgreSQL très pratique : il évite un second aller-retour pour
récupérer l'identifiant généré.

> ⚠️ **`UPDATE` et `DELETE` sans `WHERE` s'appliquent à toute la table.** C'est la première façon
> de détruire des données en production. Prenez l'habitude d'écrire d'abord le `WHERE`, puis le
> reste — ou de tester avec un `SELECT` avant.

## 6. SQL : interroger

Les exemples de cette section sont exécutés dans `psql` sur le petit jeu de données ci-dessous
(schéma de la section 4). Rejouez-le chez vous : lire un résultat est aussi important que savoir
écrire la requête.

<details>
<summary>Jeu de données utilisé pour les exemples</summary>

```sql
INSERT INTO users (email, password_hash, display_name) VALUES
    ('alice@esiee.fr', '$pbkdf2-sha256$...', 'Alice'),
    ('bob@esiee.fr',   '$pbkdf2-sha256$...', 'Bob'),
    ('chloe@esiee.fr', '$pbkdf2-sha256$...', 'Chloé'),
    ('david@esiee.fr', '$pbkdf2-sha256$...', 'David');

INSERT INTO items (owner_id, titre, tarif_jour, disponible) VALUES
    (1, 'Vélo de ville',    8.50, TRUE),
    (1, 'Perceuse',         5.00, FALSE),
    (2, 'Tente 2 places',  12.00, TRUE),
    (3, 'Vélo électrique', 15.00, TRUE),
    (2, 'Vélo enfant',      4.00, TRUE),
    (1, 'Ponceuse',         7.00, TRUE);

INSERT INTO reservations (item_id, borrower_id, date_debut, date_fin, statut) VALUES
    (1, 2, '2026-09-01', '2026-09-05', 'terminee'),
    (1, 3, '2026-09-10', '2026-09-12', 'active'),
    (3, 1, '2026-09-15', '2026-09-20', 'active'),
    (4, 4, '2026-09-02', '2026-09-03', 'annulee');
```

</details>

### 6.1. SELECT, WHERE, ORDER BY

```sql
SELECT id, titre, tarif_jour
FROM items
WHERE disponible = TRUE
  AND tarif_jour <= 10
  AND titre ILIKE '%vélo%'
ORDER BY tarif_jour ASC
LIMIT 20 OFFSET 0;
```

Résultat :

```text
 id |     titre     | tarif_jour
----+---------------+------------
  5 | Vélo enfant   |       4.00
  1 | Vélo de ville |       8.50
(2 rows)
```

Lisez-le ligne par ligne : la *Perceuse* (5 €) est écartée car indisponible, le *Vélo électrique*
car trop cher (15 €), la *Tente* car son titre ne contient pas « vélo ». Le tri croissant place le
vélo enfant en premier.

`ILIKE` est un `LIKE` insensible à la casse (spécifique PostgreSQL). `%` remplace n'importe quelle
suite de caractères. Insensible à la casse, mais **pas aux accents** : `titre ILIKE '%velo%'`
renvoie `(0 rows)` sur ce jeu de données, car `é` ≠ `e`. Pour une vraie recherche tolérante, il
faut l'extension `unaccent` — un détail qu'un agent oublie volontiers.

`LIMIT` / `OFFSET` implémentent la pagination — exactement les `limit` / `skip` de vos routes
FastAPI de la séance 1.

### 6.2. Les jointures

C'est le cœur du relationnel : recomposer l'information éclatée entre plusieurs tables.

```sql
SELECT
    i.id,
    i.titre,
    i.tarif_jour,
    u.display_name AS proprietaire
FROM items AS i
JOIN users AS u ON u.id = i.owner_id
WHERE i.disponible = TRUE;
```

Résultat :

```text
 id |      titre      | tarif_jour | proprietaire
----+-----------------+------------+--------------
  1 | Vélo de ville   |       8.50 | Alice
  3 | Tente 2 places  |      12.00 | Bob
  4 | Vélo électrique |      15.00 | Chloé
  5 | Vélo enfant     |       4.00 | Bob
  6 | Ponceuse        |       7.00 | Alice
(5 rows)
```

La table `items` ne stocke que `owner_id = 1` ; c'est la jointure qui va chercher « Alice » dans
`users`. Notez aussi que, sans `ORDER BY`, l'ordre des lignes n'est **pas garanti** : ici il suit
l'ordre d'insertion, mais rien ne vous le promet.

| Type | Effet |
|---|---|
| `INNER JOIN` (ou `JOIN`) | Uniquement les lignes qui ont une correspondance des deux côtés |
| `LEFT JOIN` | Toutes les lignes de gauche ; `NULL` à droite s'il n'y a pas de correspondance |
| `RIGHT JOIN` | L'inverse (rare, on réécrit en `LEFT JOIN`) |
| `FULL OUTER JOIN` | Tout des deux côtés |

Le `LEFT JOIN` sert dès que la relation est optionnelle :

```sql
-- Tous les items, avec le nombre de réservations (0 compris)
SELECT i.titre, COUNT(r.id) AS nb_reservations
FROM items AS i
LEFT JOIN reservations AS r ON r.item_id = i.id
GROUP BY i.id, i.titre
ORDER BY nb_reservations DESC;
```

Résultat :

```text
      titre      | nb_reservations
-----------------+-----------------
 Vélo de ville   |               2
 Tente 2 places  |               1
 Vélo électrique |               1
 Vélo enfant     |               0
 Ponceuse        |               0
 Perceuse        |               0
(6 rows)
```

Même requête en remplaçant `LEFT JOIN` par `JOIN` :

```text
      titre      | nb_reservations
-----------------+-----------------
 Vélo de ville   |               2
 Tente 2 places  |               1
 Vélo électrique |               1
(3 rows)
```

Au passage : la réservation annulée du *Vélo électrique* est comptée. Si seules les réservations
actives vous intéressent, le filtre `r.statut = 'active'` doit aller **dans le `ON`**, pas dans un
`WHERE` — sinon il retransforme votre `LEFT JOIN` en `INNER JOIN`.

Avec un `INNER JOIN`, les items jamais réservés disparaissent du résultat. C'est une erreur
extrêmement fréquente — et un bug que produisent régulièrement les agents quand on leur demande
« la liste des items avec leur nombre de réservations ».

### 6.3. Agrégats

```sql
SELECT
    u.display_name,
    COUNT(i.id)        AS nb_items,
    AVG(i.tarif_jour)  AS tarif_moyen,
    MAX(i.tarif_jour)  AS tarif_max
FROM users AS u
LEFT JOIN items AS i ON i.owner_id = u.id
GROUP BY u.id, u.display_name
HAVING COUNT(i.id) > 0
ORDER BY nb_items DESC;
```

Résultat :

```text
 display_name | nb_items |     tarif_moyen     | tarif_max
--------------+----------+---------------------+-----------
 Alice        |        3 |  6.8333333333333333 |      8.50
 Bob          |        2 |  8.0000000000000000 |     12.00
 Chloé        |        1 | 15.0000000000000000 |     15.00
(3 rows)
```

Trois choses à remarquer :

- **David n'apparaît pas** : grâce au `LEFT JOIN`, il existe bien un groupe pour lui (avec
  `nb_items = 0`), mais le `HAVING COUNT(i.id) > 0` l'élimine après le regroupement ;
- l'item indisponible (la *Perceuse* d'Alice) est compté : aucun `WHERE` ne l'exclut ;
- `AVG` sur un `NUMERIC` renvoie 16 décimales. Pour une API, arrondissez côté SQL
  (`ROUND(AVG(i.tarif_jour), 2)`) plutôt que de laisser le frontend s'en charger.

Deux règles :

- toute colonne du `SELECT` qui n'est pas dans une fonction d'agrégat doit être dans le `GROUP BY` ;
- `WHERE` filtre **avant** le regroupement, `HAVING` filtre **après**.

## 7. Transactions et ACID

Une **transaction** regroupe plusieurs opérations en une unité indivisible.

```sql
BEGIN;
    INSERT INTO reservations (item_id, borrower_id, date_debut, date_fin)
    VALUES (1, 2, '2026-09-01', '2026-09-05');

    UPDATE items SET disponible = FALSE WHERE id = 1;
COMMIT;
```

Si la seconde instruction échoue, un `ROLLBACK` annule aussi la première. Sans transaction, vous
auriez une réservation sur un item resté marqué disponible.

**ACID** résume les quatre garanties :

| Propriété | Signification |
|---|---|
| **A**tomicité | Tout ou rien |
| **C**ohérence | La base passe d'un état valide à un autre (contraintes respectées) |
| **I**solation | Les transactions concurrentes ne se marchent pas dessus |
| **D**urabilité | Une transaction validée survit à une coupure de courant |

Retenez surtout ceci pour votre projet : **toute opération métier qui touche plusieurs tables doit
être dans une transaction**. Créer une réservation, décrémenter un stock, écrire une ligne
d'historique : une seule unité. En séance 4, la session SQLAlchemy gérera cela pour vous — mais il
faut savoir ce qu'elle fait.

L'isolation mérite un mot. Deux requêtes qui réservent le même créneau au même instant peuvent
toutes deux constater « le créneau est libre » avant qu'aucune n'ait écrit. C'est une **race
condition**, et aucun `if` en Python ne la corrige. Les parades sont côté base : une contrainte
`UNIQUE` (ou `EXCLUDE`) qui rend le doublon impossible, ou un verrou explicite
(`SELECT ... FOR UPDATE`). Pour GearShare, la contrainte est la solution la plus simple et la plus
robuste.

## 8. Index et performances

Sans index, une recherche parcourt toute la table (*sequential scan*). Sur 100 lignes c'est
instantané ; sur 10 millions, c'est une catastrophe.

### 8.1. Qu'est-ce qu'un index ?

Pensez à l'index à la fin d'un livre technique. Pour trouver où l'on parle de « transaction »,
vous ne relisez pas les 400 pages : vous allez à la lettre T, la liste est triée, et elle vous
renvoie directement aux pages 112 et 245. Un index de base de données fait exactement cela.

Concrètement, un **index** est une **structure de données séparée de la table**, stockée sur
disque à côté d'elle, qui contient :

- les valeurs d'une (ou plusieurs) colonne(s), **triées** ;
- pour chaque valeur, un pointeur vers l'emplacement physique de la ligne dans la table.

Par défaut, PostgreSQL utilise un **B-tree** : un arbre équilibré dont chaque nœud couvre une
plage de valeurs. Pour trouver `owner_id = 3` parmi un million de lignes, le moteur descend
l'arbre en 3 ou 4 étapes (au lieu de lire le million de lignes), puis suit les pointeurs vers les
lignes concernées.

```text
Table items (ordre d'insertion)          Index idx_items_owner (trié par owner_id)
+----+----------+-----------------+      +----------+------------------------+
| id | owner_id | titre           |      | owner_id | lignes                 |
+----+----------+-----------------+      +----------+------------------------+
|  1 |        1 | Vélo de ville   |      |        1 | -> id 1, id 2, id 6    |
|  2 |        1 | Perceuse        |      |        2 | -> id 3, id 5          |
|  3 |        2 | Tente 2 places  |      |        3 | -> id 4                |
|  4 |        3 | Vélo électrique |      +----------+------------------------+
|  5 |        2 | Vélo enfant     |
|  6 |        1 | Ponceuse        |
+----+----------+-----------------+
```

Parce qu'il est trié, un B-tree accélère :

- les égalités (`WHERE owner_id = 3`) ;
- les plages (`WHERE date_debut BETWEEN '2026-09-01' AND '2026-09-30'`, `<`, `>=`...) ;
- les tris (`ORDER BY created_at DESC LIMIT 20` : les lignes sont lues déjà dans l'ordre).

Ce qu'un B-tree fait avec les recherches de texte partielles (`LIKE`), vous le mesurerez vous-même
au TP.

Ce qu'il faut retenir sur son coût :

- un index **occupe de la place** : sur une table de 100 000 items, l'index sur `owner_id` pèse
  environ 700 Ko pour une table de 7,5 Mo ;
- il doit être **maintenu à chaque écriture** : chaque `INSERT`, chaque `UPDATE` de la colonne
  indexée, chaque `DELETE` met aussi l'index à jour ;
- c'est le **planificateur** de PostgreSQL qui décide de s'en servir ou non. Créer un index ne
  garantit pas qu'il sera utilisé : si la requête ramène une grosse partie de la table, lire la
  table en entier reste moins cher.

Enfin, vous avez déjà des index sans le savoir : PostgreSQL en crée un automatiquement pour chaque
`PRIMARY KEY` (`items_pkey`) et chaque contrainte `UNIQUE` (`users_email_key`). C'est d'ailleurs
ainsi qu'il vérifie l'unicité rapidement.

### 8.2. Créer des index

```sql
CREATE INDEX idx_items_owner ON items (owner_id);
CREATE INDEX idx_reservations_item_dates ON reservations (item_id, date_debut);
CREATE UNIQUE INDEX idx_users_email ON users (email);
```

Le troisième est redondant avec notre schéma : la contrainte `email ... UNIQUE` a déjà créé un
index unique. Il est là pour montrer la syntaxe — ne dupliquez pas les index dans votre projet.

L'index composite `(item_id, date_debut)` sert les requêtes qui filtrent sur `item_id`, ou sur
`item_id` **et** `date_debut`. Il ne sert quasiment pas à une requête qui filtre uniquement sur
`date_debut` : l'ordre des colonnes compte, comme dans un annuaire trié par nom puis prénom.

**Quand créer un index :**

- sur toutes vos **clés étrangères** (`owner_id`, `item_id`, `borrower_id`) — PostgreSQL ne les
  indexe **pas** automatiquement, contrairement aux clés primaires ;
- sur les colonnes utilisées dans les `WHERE` fréquents ;
- sur les colonnes de tri (`ORDER BY`) des listes paginées.

**Quand ne pas en créer :**

- sur une petite table (le moteur ira plus vite en la parcourant) ;
- sur une colonne peu sélective (un booléen qui vaut `TRUE` pour 90 % des lignes) ;
- partout « au cas où » : chaque index ralentit les `INSERT`/`UPDATE` et consomme de l'espace.

### 8.3. Mesurer avec `EXPLAIN ANALYZE`

Pour mesurer plutôt que deviner :

```sql
EXPLAIN ANALYZE
SELECT * FROM items WHERE owner_id = 3;
```

`EXPLAIN` affiche le plan d'exécution choisi par le planificateur ; `ANALYZE` **exécute
réellement** la requête et ajoute les mesures. Attention donc : un `EXPLAIN ANALYZE DELETE ...`
supprime vraiment les lignes.

Exemple sur une table `items` de 100 000 lignes réparties entre 1 000 propriétaires, **sans index**
sur `owner_id` :

```text
                                              QUERY PLAN
------------------------------------------------------------------------------------------------------
 Seq Scan on items  (cost=0.00..2185.07 rows=99 width=74) (actual time=0.004..3.415 rows=104 loops=1)
   Filter: (owner_id = 3)
   Rows Removed by Filter: 99902
 Planning Time: 0.168 ms
 Execution Time: 3.446 ms
(5 rows)
```

Comment le lire :

- `Seq Scan on items` : parcours complet de la table ;
- `cost=0.00..2185.07` : coût **estimé** par le planificateur, dans une unité arbitraire (utile
  seulement pour comparer deux plans) ;
- `rows=99` (estimé) contre `actual ... rows=104` (réel) : les statistiques sont bonnes ;
- `Rows Removed by Filter: 99902` : le signal d'alarme. PostgreSQL a lu 100 006 lignes pour en
  garder 104 ;
- `Execution Time: 3.446 ms` : le temps réellement passé.

Même requête **après** `CREATE INDEX idx_items_owner ON items (owner_id);` :

```text
                                                        QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on items  (cost=5.06..295.45 rows=99 width=74) (actual time=0.070..0.156 rows=104 loops=1)
   Recheck Cond: (owner_id = 3)
   Heap Blocks: exact=99
   ->  Bitmap Index Scan on idx_items_owner  (cost=0.00..5.04 rows=99 width=0) (actual time=0.058..0.059 rows=104 loops=1)
         Index Cond: (owner_id = 3)
 Planning Time: 0.195 ms
 Execution Time: 0.340 ms
(7 rows)
```

Le plan se lit **de l'intérieur vers l'extérieur** (la ligne la plus indentée s'exécute d'abord) :

1. `Bitmap Index Scan on idx_items_owner` : l'index fournit la liste des emplacements des 104
   lignes ;
2. `Bitmap Heap Scan on items` : PostgreSQL va lire ces lignes dans la table, en visitant 99 blocs
   (`Heap Blocks: exact=99`) au lieu de la table entière.

Le coût estimé passe de 2185 à 295, le temps de 3,4 ms à 0,34 ms : **10 fois plus rapide**, et
l'écart grandit avec la taille de la table.

Vous croiserez trois types de parcours :

| Nœud | Signification |
|---|---|
| `Seq Scan` | Lecture de toute la table ; normal sur une petite table ou une requête peu sélective |
| `Index Scan` | L'index donne chaque ligne une par une ; typique d'une recherche très sélective |
| `Bitmap Index Scan` + `Bitmap Heap Scan` | L'index repère d'abord toutes les lignes, puis la table est lue bloc par bloc ; typique quand plusieurs dizaines ou centaines de lignes correspondent |

Par exemple, une recherche par clé primaire donne un `Index Scan` pur :

```text
EXPLAIN ANALYZE SELECT * FROM items WHERE id = 4242;

 Index Scan using items_pkey on items  (cost=0.29..8.31 rows=1 width=74) (actual time=0.009..0.010 rows=1 loops=1)
   Index Cond: (id = 4242)
 Planning Time: 0.203 ms
 Execution Time: 0.030 ms
```

Sur le petit jeu de données de la section 6 (6 items), vous obtiendrez un `Seq Scan` **même avec
l'index** : lire une seule page de table coûte moins cher que passer par l'index. Ce n'est pas un
bug, c'est le planificateur qui fait son travail.

Faites l'exercice au TP : créez quelques milliers de lignes, mesurez, ajoutez l'index, remesurez.
Le chiffre est plus convaincant que le discours.

## 9. Fonctions, procédures stockées et triggers

PostgreSQL ne se contente pas de stocker : il peut **exécuter du code** côté base. Vous n'en
écrirez pas beaucoup dans le projet, mais vous devez savoir les lire.

### 9.1. Fonction ou procédure ?

| | Fonction (`CREATE FUNCTION`) | Procédure (`CREATE PROCEDURE`) |
|---|---|---|
| Renvoie une valeur | Oui (scalaire, ligne, table, ou `trigger`) | Non (paramètres `INOUT` possibles) |
| Appel | Dans une requête : `SELECT ma_fonction(...)` | Instruction dédiée : `CALL ma_procedure(...)` |
| Transactions | S'exécute dans la transaction de l'appelant | Peut faire `COMMIT` / `ROLLBACK` en interne |
| Usage typique | Calcul, trigger | Traitement par lots, maintenance |

Une fonction simple, en SQL pur :

```sql
CREATE FUNCTION montant_reservation(p_tarif NUMERIC, p_debut DATE, p_fin DATE)
RETURNS NUMERIC
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT p_tarif * (p_fin - p_debut);
$$;

SELECT montant_reservation(8.50, '2026-09-01', '2026-09-05');  -- 34.00
```

Une procédure, qui archive les réservations passées :

```sql
CREATE PROCEDURE terminer_reservations_passees()
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE reservations
    SET statut = 'terminee'
    WHERE statut = 'active' AND date_fin < CURRENT_DATE;
END;
$$;

CALL terminer_reservations_passees();
```

`$$ ... $$` délimite simplement le corps (pour éviter d'échapper les apostrophes). `plpgsql` est le
langage procédural de PostgreSQL : variables, `IF`, boucles.

### 9.2. Exemple : `created_at` et `updated_at` gérés par la base

Cas d'usage le plus fréquent dans un projet web : horodater chaque ligne. `DEFAULT now()` suffit
pour la création, mais il a deux limites :

- rien n'empêche le code applicatif d'envoyer une valeur arbitraire pour `created_at` ;
- `DEFAULT` ne joue qu'à l'insertion : il ne met jamais à jour `updated_at`.

On confie donc les deux colonnes à une **fonction trigger**, appelée automatiquement par
PostgreSQL avant chaque `INSERT` et chaque `UPDATE` :

```sql
ALTER TABLE items ADD COLUMN updated_at TIMESTAMPTZ NOT NULL DEFAULT now();

CREATE FUNCTION set_timestamps()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        NEW.created_at := now();              -- à l'insertion : date de création imposée
    ELSE
        NEW.created_at := OLD.created_at;     -- à la mise à jour : jamais modifiable
    END IF;
    NEW.updated_at := now();
    RETURN NEW;
END;
$$;

CREATE TRIGGER items_set_timestamps
BEFORE INSERT OR UPDATE ON items
FOR EACH ROW
EXECUTE FUNCTION set_timestamps();
```

À lire attentivement :

- **`NEW`** est la ligne telle qu'elle va être écrite, **`OLD`** la ligne avant modification
  (inexistante à l'`INSERT`). Un trigger `BEFORE` peut modifier `NEW` avant l'écriture.
- **`TG_OP`** vaut `'INSERT'`, `'UPDATE'` ou `'DELETE'` : une seule fonction couvre les deux cas.
- **`FOR EACH ROW`** : la fonction s'exécute une fois par ligne touchée.
- La fonction est **générique** : elle ne mentionne aucune table. Vous créez un trigger par table
  (`users`, `items`, `reservations`) qui la réutilise.

Vérifiez le comportement :

```sql
INSERT INTO items (owner_id, titre, tarif_jour, created_at)
VALUES (1, 'Scie sauteuse', 6.00, '1999-01-01')     -- valeur ignorée par le trigger
RETURNING created_at, updated_at;

UPDATE items SET tarif_jour = 7.00 WHERE titre = 'Scie sauteuse'
RETURNING created_at, updated_at;                    -- seul updated_at a bougé
```

> 💡 `now()` renvoie l'heure de **début de la transaction**, pas l'heure exacte de l'instruction :
> toutes les lignes écrites dans une même transaction ont le même horodatage. C'est en général ce
> qu'on veut. Si vous avez besoin de l'heure réelle, c'est `clock_timestamp()`.

### 9.3. Code en base : avec parcimonie

Un trigger est **invisible** depuis le code Python : quelqu'un qui lit votre service ne voit pas
que `updated_at` change tout seul. Ma règle pour le projet :

- **oui** pour des invariants techniques simples et universels (horodatage, audit) ;
- **non** pour la logique métier (calcul de prix, règles de réservation) : elle va dans la couche
  service, où elle est lisible, testable avec pytest et reviewable en PR.

En séance 4, retenez aussi ceci : **Alembic ne détecte ni les fonctions ni les triggers** en
autogénération. Ils s'écrivent à la main dans une migration (`op.execute(...)`), avec leur
`DROP` correspondant dans `downgrade()`. Si un agent vous en génère un, c'est un point de review
obligatoire.

## 10. PostgreSQL dans Docker Compose

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: gearshare
      POSTGRES_PASSWORD: gearshare
      POSTGRES_DB: gearshare
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./sql/init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U gearshare -d gearshare"]
      interval: 5s
      timeout: 3s
      retries: 10

volumes:
  pgdata:
```

Trois points à comprendre :

1. **Le volume nommé `pgdata`** est ce qui rend vos données persistantes entre les
   `docker compose down` / `up`. Sans lui, la base repart vide à chaque fois. Inversement,
   `docker compose down -v` supprime le volume — c'est votre bouton « repartir de zéro ».
2. **`/docker-entrypoint-initdb.d`** : les fichiers `.sql` ou `.sh` de ce dossier sont exécutés
   automatiquement, **uniquement au tout premier démarrage**, quand le volume est vide. Si vous
   modifiez votre script d'init et que rien ne change, c'est que le volume existe déjà : faites
   `docker compose down -v`.
3. **Le `healthcheck`** permet aux autres services d'attendre que la base soit réellement prête :

```yaml
  api:
    build: ./backend
    depends_on:
      db:
        condition: service_healthy
```

Sans cette condition, `depends_on` attend seulement que le conteneur soit *démarré*, pas que
PostgreSQL accepte les connexions — et votre API plante au démarrage une fois sur deux. C'est un
grand classique des projets rendus.

Pour vous connecter :

```bash
# Depuis l'hôte, si le port est publié
psql -h localhost -U gearshare -d gearshare

# Ou directement dans le conteneur (pas besoin de psql installé)
docker compose exec db psql -U gearshare -d gearshare
```

Quelques méta-commandes `psql` à connaître : `\dt` (lister les tables), `\d items` (décrire une
table), `\di` (lister les index), `\x` (affichage vertical, très utile), `\q` (quitter).

> 🔒 Le mot de passe en clair dans le `docker-compose.yml` est acceptable en développement local
> uniquement. Pour votre projet, ces valeurs iront dans un fichier `.env` **non commité**, avec un
> `.env.example` commité qui documente les variables attendues. On formalise cela en séance 4.

## 11. Synthèse

- Une base relationnelle apporte persistance, **intégrité**, concurrence et interrogation
  déclarative. L'intégrité est ce qu'on sous-estime le plus.
- Mettez les invariants **dans le schéma** : `NOT NULL`, `UNIQUE`, `CHECK`, clés étrangères.
- Choisissez vos types : `NUMERIC` pour l'argent, `TIMESTAMPTZ` pour les dates.
- Maîtrisez `JOIN` (et sachez quand il faut un `LEFT JOIN`), `GROUP BY` / `HAVING`, `LIMIT`.
- Une opération métier multi-tables = une transaction.
- Indexez les clés étrangères et les colonnes de recherche ; mesurez avec `EXPLAIN ANALYZE`.
- Fonction = renvoie une valeur, appelée dans une requête ; procédure = `CALL`. Un trigger
  `BEFORE INSERT OR UPDATE` gère proprement `created_at` / `updated_at` ; la logique métier, elle,
  reste dans le code applicatif.
- Dans Compose : volume nommé pour la persistance, `healthcheck` + `condition: service_healthy`
  pour l'ordre de démarrage, `/docker-entrypoint-initdb.d` pour l'initialisation.

Passez au [TP](../tp/README.md).

En séance 4, on relie enfin les deux moitiés : l'API de la séance 1 et la base de cette séance 3,
avec SQLAlchemy, Alembic et une vraie architecture en couches.
