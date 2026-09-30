# TP — Séance 3 : PostgreSQL en pratique

## Objectifs du TP

À la fin de ce TP, vous devez savoir :

- lancer PostgreSQL avec Docker Compose et vous y connecter en `psql` ;
- écrire le DDL d'un schéma relationnel avec ses contraintes ;
- constater qu'une contrainte vous protège réellement d'une donnée incohérente ;
- écrire des requêtes avec jointures, agrégats et pagination ;
- mesurer l'effet d'un index avec `EXPLAIN ANALYZE` ;

Tout se fait en SQL, **écrit par vous** : vous devez connaître cette syntaxe pour pouvoir relire
et corriger un schéma, et tout ce qui suivra. Le cours et la
[documentation PostgreSQL](https://www.postgresql.org/docs/16/) sont vos références. Pas encore de
Python : la liaison avec FastAPI est le sujet de la séance 4.

## Prérequis

- Docker Desktop lancé.

---

## Étape 0 — Lancer PostgreSQL

Créez un dossier `tp-db/` et écrivez-y un `docker-compose.yml` avec un seul service `db` :

- image `postgres:16` ;
- utilisateur, mot de passe et base : `gearshare` (variables d'environnement de l'image officielle) ;
- port `5432` exposé sur la machine ;
- un **volume nommé** pour les données ;
- le dossier local `sql/init/` monté en lecture seule sur `/docker-entrypoint-initdb.d` (les
  scripts qui s'y trouvent sont joués au premier démarrage, sur une base vide) ;
- un **healthcheck** basé sur `pg_isready`.

Le §9 du cours vous donne la structure. Créez le dossier `sql/init/` (vide pour l'instant), puis
démarrez la base et connectez-vous :

```bash
docker compose up -d
docker compose ps          # le service db doit être "healthy"
docker compose exec db psql -U gearshare -d gearshare
```

Dans `psql`, affichez la version du serveur, listez les tables (`\dt`, aucune pour l'instant) et
quittez (`\q`).

> 💡 Si vous préférez un client graphique, DBeaver ou l'extension PostgreSQL de VS Code se
> connectent sur `localhost:5432`. Mais **sachez vous servir de `psql`** : sur un serveur distant,
> c'est souvent tout ce que vous aurez.

---

## Étape 1 — Créer le schéma

Créez `sql/init/01-schema.sql` avec les tables `users`, `items` et `reservations`. Le cahier des
charges :

**`users`**
- `id` : clé primaire auto-incrémentée
- `email` : obligatoire, unique, max 255 caractères
- `password_hash` : obligatoire
- `display_name` : obligatoire, max 80 caractères
- `created_at` : horodatage avec fuseau, valeur par défaut = maintenant

**`items`**
- `id` : clé primaire auto-incrémentée
- `owner_id` : obligatoire, référence `users(id)`, suppression en cascade
- `titre` : obligatoire, max 120 caractères
- `description` : optionnelle, texte libre
- `tarif_jour` : obligatoire, montant exact, strictement positif
- `disponible` : booléen, obligatoire, défaut vrai
- `created_at` : comme pour `users`

**`reservations`**
- `id` : clé primaire auto-incrémentée
- `item_id` : obligatoire, référence `items(id)`, suppression en cascade
- `borrower_id` : obligatoire, référence `users(id)` — aucun comportement de suppression
  n'est précisé : n'en ajoutez pas
- `date_debut`, `date_fin` : dates obligatoires
- `statut` : obligatoire, défaut `'active'`, uniquement parmi `active`, `annulee`, `terminee`
- une contrainte **nommée** garantissant `date_fin > date_debut`

Appliquez-le. Comme le volume existe déjà (l'étape 0 a démarré la base à vide), le script d'init
**ne se rejouera pas tout seul**. Deux options :

```bash
# Option A : repartir de zéro (le script d'init se joue)
docker compose down -v && docker compose up -d

# Option B : appliquer le fichier à chaud
docker compose exec -T db psql -U gearshare -d gearshare < sql/init/01-schema.sql
```

Faites l'option A au moins une fois : c'est le mécanisme que vous utiliserez pour le script de
seed du projet.

**Vérifiez** avec `\dt`, puis `\d items` et `\d reservations` : vous devez voir les colonnes, la
clé primaire, les contraintes `CHECK` et les clés étrangères. Si une contrainte manque, corrigez
maintenant.

---

## Étape 2 — Insérer des données, et se faire refuser

Créez `sql/init/02-seed.sql`, qui insère exactement les données suivantes (les résultats attendus
aux étapes suivantes en dépendent). Les `id` sont ceux que la base doit générer, dans cet ordre.

**Utilisateurs** — mots de passe : n'importe quelle chaîne factice, ce ne sont pas encore de vrais
hashs.

| id | email | display_name |
|---|---|---|
| 1 | alice@esiee.fr | Alice |
| 2 | bob@esiee.fr | Bob |
| 3 | chloe@esiee.fr | Chloé |

**Items**

| id | propriétaire | titre | description | tarif_jour |
|---|---|---|---|---|
| 1 | Alice | Vélo de ville | Cadre alu, panier avant | 8.50 |
| 2 | Alice | Perceuse | Avec jeu de mèches | 5.00 |
| 3 | Bob | Tente 2 places | *(aucune)* | 12.00 |
| 4 | Bob | Appareil photo | Reflex + objectif 50mm | 20.00 |

**Réservations**

| id | item | emprunteur | date_debut | date_fin | statut |
|---|---|---|---|---|---|
| 1 | Vélo de ville | Bob | 2026-09-01 | 2026-09-05 | terminee |
| 2 | Tente 2 places | Alice | 2026-09-10 | 2026-09-12 | active |
| 3 | Appareil photo | Chloé | 2026-09-15 | 2026-09-20 | active |

Appliquez-le, puis **essayez délibérément de casser la base**. Écrivez une instruction pour
chacun des cas suivants, et notez le message d'erreur exact et le nom de la contrainte violée :

1. un deuxième utilisateur avec l'email `alice@esiee.fr` ;
2. un item dont le propriétaire n'existe pas (`owner_id = 999`) ;
3. un item à tarif négatif ;
4. une réservation qui se termine avant de commencer ;
5. une réservation au statut `en_attente_peut_etre`.

Les cinq doivent échouer. Consignez les noms de contraintes : c'est ce que vous intercepterez en
Python (séance 4) pour renvoyer un `409` propre au lieu d'un `500`.

Puis testez la suppression d'un utilisateur : comptez les items de Bob, puis supprimez Bob. La
base refuse. Lisez attentivement le message : il vous dit pourquoi et nomme la contrainte qui
bloque. Vérifiez que rien n'a été supprimé, pas même ses items.

Voyons maintenant ce qui se passerait sans ce garde-fou. Écrivez les deux instructions
`ALTER TABLE` qui suppriment la clé étrangère de `borrower_id`, puis la recréent, sous le même nom,
avec `ON DELETE CASCADE`. Supprimez à nouveau Bob, recomptez ses items et regardez ce que contient
la table `reservations`. Pour chaque ligne disparue, notez qui l'a perdue et pourquoi : par quelle
clé étrangère la cascade est passée.

**Question à laquelle vous devez répondre par écrit** : quel comportement voulez-vous pour
GearShare ? Supprimer un compte doit-il effacer l'historique des prêts que d'autres utilisateurs
ont faits sur son matériel ? Et ses propres emprunts ? Notez votre décision — c'est un choix de
conception, pas un détail.

Rechargez ensuite les données (`docker compose down -v && docker compose up -d`).

---

## Étape 3 — Interroger

Écrivez les requêtes suivantes dans un fichier `sql/requetes.sql`, avec leur résultat en
commentaire. Toutes doivent être testées.

1. Les items disponibles à moins de 10 €/jour, triés par tarif croissant.
2. Les items dont le titre contient « velo », insensible à la casse. Regardez le résultat de près.
3. Chaque item avec le nom de son propriétaire (jointure).
4. Le nombre de réservations par item, **y compris les items jamais réservés** (le piège du cours).
5. Pour chaque utilisateur : son nom, son nombre d'items publiés, le tarif moyen de ses items.
   Uniquement ceux qui ont publié au moins un item.
6. Les réservations actives dont la date de début est postérieure au 1er septembre 2026, avec le
   titre du matériel et le nom de l'emprunteur.
7. Les 2 items les plus chers, en sautant le premier (pagination).
8. Le chiffre d'affaires potentiel de chaque réservation active : tarif journalier × nombre de
   jours. (Indice : la différence de deux `DATE` donne un entier en PostgreSQL.)

La requête 4 est celle sur laquelle je vous attends. Si votre résultat ne contient que les items
réservés, ou si la Perceuse apparaît avec 1 réservation, corrigez.

---

## Étape 4 — Transactions

Ouvrez **deux** sessions `psql` en parallèle (deux terminaux), A et B.

**Isolation.** Dans A, ouvrez une transaction, passez le tarif du Vélo de ville à 99.00 et
relisez-le. Sans valider, relisez-le depuis B. Puis annulez dans A et relisez des deux côtés.
Recommencez en validant au lieu d'annuler. Notez à chaque fois ce que voit B.

**Atomicité.** Dans une seule transaction, écrivez :

1. une réservation valide du Vélo de ville par Chloé du 1er au 4 novembre 2026 ;
2. le passage du Vélo de ville à `disponible = FALSE` ;
3. une réservation qui échoue volontairement (emprunteur inexistant) ;
4. une validation de la transaction.

Vérifiez ensuite : la réservation de novembre existe-t-elle ? Le vélo est-il indisponible ?
Notez aussi ce que répond `psql` à la validation, et ce qu'il répond si vous tapez une autre
commande entre l'erreur et la validation. C'est exactement ce que vous devrez gérer côté
SQLAlchemy en séance 4.

---

## Étape 5 — Index et `EXPLAIN ANALYZE`

**Générez du volume** : insérez 200 000 items appartenant à Alice, de titre `Item de test 1`,
`Item de test 2`, …, `Item de test 200000`, avec un tarif aléatoire entre 1 et 51 €. Une seule
instruction suffit : cherchez `generate_series` et `random()` dans la documentation. Lancez
ensuite `ANALYZE items;` pour mettre à jour les statistiques du planificateur.

**Mesurez** avec `EXPLAIN ANALYZE` la recherche de l'item de titre exact `Item de test 123456`.
Relevez le type de nœud et le temps d'exécution.

Créez un index sur `titre`, relancez `ANALYZE`, remesurez. Consignez les deux chiffres dans
`MESURES.md` avec le facteur d'accélération.

Mesurez ensuite deux cas plus subtils :

- (a) la recherche des titres qui **se terminent** par `123456` (joker en tête avec `LIKE`) ;
- (b) le filtre `disponible = TRUE`, **après** avoir créé un index sur `disponible` et relancé
  `ANALYZE`. Mesurez ensuite `disponible = FALSE`, avec le même index.

Expliquez en une phrase chacun des résultats dans `MESURES.md`. (Indice pour (a) : un B-tree
est trié par le début de la chaîne. Indice pour (b) : comptez combien de lignes vérifient chacune
des deux conditions. Question bonus : l'index sur `titre` sert-il pour un `LIKE` avec le joker
**en fin** de chaîne ? Mesurez avant de répondre.)

---

## Étape 6 — Étendre le schéma, et le reviewer

Le schéma actuel ne couvre pas tout GearShare : il manque les **avis** (note + commentaire après
un prêt terminé) et les **tags** de matériel.

### 6.1. Les règles métier

Voici les règles à ajouter au schéma existant (`users`, `items`, `reservations`) :

1. Après une réservation au statut `terminee`, l'emprunteur peut laisser un avis :
   une note entière de 1 à 5 et un commentaire libre optionnel.
2. Un avis est lié à exactement une réservation, et une réservation ne peut avoir
   qu'un seul avis.
3. Un matériel peut porter plusieurs tags (ex. `velo`, `sport`) ; un tag peut être
   porté par plusieurs matériels. Un tag a un libellé unique en minuscules.
4. Un même matériel ne peut pas avoir deux réservations actives qui se chevauchent.

Contraintes techniques : PostgreSQL 16, contraintes nommées, `TIMESTAMPTZ` pour les horodatages,
index sur les clés étrangères.

### 6.2. Écrivez le DDL

Dans `sql/init/03-schema-v2.sql`, écrivez le DDL qui traduit ces règles (`CREATE TABLE`,
`ALTER TABLE`, `CREATE INDEX`). Avant d'écrire, identifiez pour chaque règle le mécanisme qui la
garantit : type, `NOT NULL`, `UNIQUE`, `CHECK`, clé étrangère, table de liaison… ou rien, si la
base ne peut pas la garantir seule.

### 6.3. Reviewez

Relisez votre DDL avec la grille suivante, corrigez ce qui doit l'être et consignez le résultat
dans `REVIEW-schema.md` :

| Point de contrôle | OK / KO | Ce que j'ai corrigé |
|---|---|---|
| Chaque table a une clé primaire | | |
| Les clés étrangères sont déclarées avec `REFERENCES` | | |
| Les `ON DELETE` sont explicites et cohérents avec le métier | | |
| La règle « un seul avis par réservation » est garantie par une contrainte `UNIQUE` | | |
| La note est bornée par un `CHECK (note BETWEEN 1 AND 5)` | | |
| La table de liaison item/tag a une clé primaire composite | | |
| Le libellé de tag est `UNIQUE` | | |
| Les contraintes sont nommées | | |
| Des index existent sur les clés étrangères | | |
| Le non-chevauchement est réellement garanti (contrainte) ou explicitement laissé à l'applicatif | | |
| Aucun type flottant pour un montant, aucun `TIMESTAMP` sans fuseau | | |

Puis **prouvez que ça marche** : écrivez dans `sql/tests-contraintes.sql` au moins quatre
`INSERT` qui doivent échouer (note à 6, deux avis sur la même réservation, deux tags de même
libellé, un avis sur une réservation inexistante) et vérifiez qu'ils échouent tous — et pour la
bonne raison : lisez le nom de la contrainte dans chaque message.

Sur le non-chevauchement : garantissez-le par une contrainte d'exclusion (`EXCLUDE`, voir la
documentation PostgreSQL), ou documentez explicitement dans `REVIEW-schema.md` que vous le gérerez
en applicatif et pourquoi. Ce que je ne veux pas, c'est que la question soit passée sous silence.

---

## Ce que vous devez me rendre à la fin de la séance 3

Dans votre dépôt de groupe, un dossier `tp-db/` contenant :

1. **`docker-compose.yml`** avec volume nommé, healthcheck et init.
2. **`sql/init/01-schema.sql`** et **`sql/init/02-seed.sql`** : `docker compose down -v && up -d`
   doit reconstruire une base complète et peuplée.
3. **`sql/requetes.sql`** : les 8 requêtes de l'étape 3, testées, avec leur résultat en commentaire.
4. **`MESURES.md`** : les mesures `EXPLAIN ANALYZE` avant/après index et vos deux explications.
5. **`sql/init/03-schema-v2.sql`** (version corrigée) et **`REVIEW-schema.md`**.
6. **`sql/tests-contraintes.sql`** : les insertions qui doivent échouer.
7. Votre réponse écrite à la question de l'étape 2 sur la suppression d'un utilisateur (2–3 lignes
   dans `REVIEW-schema.md` suffisent).

## Pour aller plus loin

- Créez une vue qui expose le catalogue enrichi (item + propriétaire + note moyenne) et
  interrogez-la comme une table.
- Écrivez un trigger qui bascule automatiquement une réservation en `terminee` quand `date_fin`
  est dépassée, et discutez : est-ce une bonne idée de mettre cette logique en base plutôt que
  dans l'application ?
- Comparez `EXPLAIN ANALYZE` d'une jointure sur `owner_id` avec et sans index sur cette clé
  étrangère, sur les 200 000 lignes générées.
