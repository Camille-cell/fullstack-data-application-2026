# TP — Séance 5 : sécuriser l'application

## Ce que vous allez faire

À la fin de la séance 4, votre application a trois services et une vraie base de données, mais
n'importe qui peut publier, modifier ou supprimer le matériel de n'importe qui. Dans ce TP, vous
ajoutez les comptes utilisateurs, la connexion par JWT et les règles de propriété, côté backend
**et** côté frontend.

## Objectifs du TP

À la fin de ce TP, vous devez savoir :

- hacher et vérifier un mot de passe correctement ;
- implémenter l'inscription et la connexion avec émission d'un JWT ;
- écrire une dépendance `get_current_user` et protéger des routes ;
- vérifier la propriété d'une ressource et renvoyer `403` ;
- brancher la connexion sur le frontend : cookie, transmission du jeton, retour à la page de
  connexion sur un `401` ;
- exploiter une faille volontairement introduite, pour comprendre ce que vous protégez.

## Prérequis

- Le TP 2 de la séance 4 terminé : `docker compose up` démarre frontend, backend et base, les
  ressources `items` et `reservations` fonctionnent, et un utilisateur de démonstration existe en
  base.
- La table `users` existe déjà : elle vient du schéma de la séance 3, et votre migration initiale
  de la séance 4 l'a créée avec sa colonne `password_hash`. Ce TP ne fait que la remplir et la
  lire ; aucune migration n'est à écrire.
- Docker Desktop lancé.

Vous travaillez dans le même dépôt que celui de la séance 4.

---

## Étape 0 — Dépendances et configuration

1. **Dépendances du backend.** Ajoutez à `backend/requirements.txt` la bibliothèque PyJWT,
   l'extra `email` de Pydantic (pour valider les adresses email) et `python-multipart` (pour lire
   le formulaire de connexion OAuth2). Fixez les versions majeures, comme pour les autres.
2. **Le secret.** Ajoutez `JWT_SECRET` et `JWT_EXPIRE_MINUTES` (60) à votre `.env.example`, avec
   une valeur factice et explicite pour le secret. Dans votre `.env`, mettez un vrai secret,
   généré par la commande donnée juste après cette liste.
3. **La configuration du backend.** Ajoutez ces deux variables à votre classe de configuration.
   Il est préférable que le backend **refuse de démarrer** si `JWT_SECRET` est absent, plutôt que d'utiliser une
   valeur par défaut.
4. **La configuration du frontend.** Le frontend reçoit une seule nouvelle variable, non secrète :
   `COOKIE_SECURE`, à `false` en local. Ajoutez-la dans la section `environment:` du service
   `frontend` de votre `docker-compose.yml`, pas par le `.env`.

Pour générer le secret :

```bash
openssl rand -hex 32
```

**Vérifiez** que le frontend ne connaît toujours pas le secret, et que le `.env` n'est pas suivi
par Git :

```bash
docker compose up -d --build
docker compose exec frontend env | grep -i jwt     # aucune ligne attendue
git status                                          # le .env ne doit PAS apparaître
```

---

## Étape 1 — Le hachage des mots de passe

Écrivez `backend/app/core/security.py` avec deux fonctions, `hash_password` et `verify_password`,
en suivant le cours :

- PBKDF2-HMAC-SHA256 de la bibliothèque standard, 600 000 itérations, sel de 16 octets tiré avec
  `secrets` ;
- un condensat **auto-descriptif** : algorithme, coût, sel et condensat dans une seule chaîne ;
- `verify_password` relit le coût dans la chaîne stockée, pas dans la constante ;
- la comparaison finale se fait en temps constant ;
- un condensat malformé fait renvoyer `False`, jamais une exception.

**Vérifiez** en écrivant `backend/tests/test_security.py`, qui n'a besoin ni de base ni de client
HTTP. Il peut par exemple montrer que :

- deux hachages du même mot de passe sont différents ;
- le bon mot de passe est accepté, un mauvais refusé ;
- un condensat malformé (par exemple `x`) est refusé sans erreur ;
- un condensat produit avec un autre nombre d'itérations reste vérifiable.

```bash
docker compose run --rm backend pytest tests/test_security.py -v
```

**Vérifiez votre compréhension** (inutile de rédiger les réponses ; prenez le temps d'en discuter en binôme) :

1. Pourquoi deux hachages du même mot de passe sont-ils différents ?
2. Comment `verify_password` retrouve-t-il le sel, alors qu'aucune colonne ne le stocke ?
3. Combien de temps prend un appel à `hash_password` sur votre machine ? Est-ce un problème, ou
   l'objectif ?
4. Que se passerait-il pour l'utilisateur de démonstration de la séance 4, dont le
   `password_hash` vaut `x`, si `verify_password` levait une exception sur ce condensat ?

---

## Étape 2 — L'inscription

Écrivez la verticale, en respectant vos couches :

- **schémas** (`app/schemas/user.py`) : un schéma de création (email validé, mot de passe de 12 à
  128 caractères, nom affiché), un schéma de lecture **sans** `password_hash`, et le schéma de la
  réponse de connexion (`access_token`, `token_type`, `expires_in`) ;
- **repository** (`app/repositories/user.py`) : récupérer un utilisateur par identifiant et par
  email, en ajouter un ;
- **exception métier** : une nouvelle exception « email déjà utilisé », traduite en `409` par votre
  gestionnaire d'exceptions de la séance 4. Vérifiez qu'elle est bien dans la table de
  correspondance, sinon elle sortira en `400` ;
- **service** (`app/services/auth.py`) : l'inscription, qui lève cette exception si l'email est
  pris et hache le mot de passe avant l'ajout ;
- **router** (`app/routers/auth.py`) : `POST /auth/register`, qui répond `201` avec l'utilisateur
  créé.

**Vérifiez** depuis `http://localhost:8000/docs` ou avec `curl` :

| Cas | Attendu |
|---|---|
| Inscription valide | `201`, l'utilisateur est renvoyé |
| Même email une seconde fois | `409` |
| Email invalide (`pas-un-email`) | `422` |
| Mot de passe de 4 caractères | `422` |
| Mot de passe de 200 caractères | `422` |

```bash
curl -i -X POST http://localhost:8000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "alice@esiee.fr", "password": "motdepasse-alice", "display_name": "Alice"}'

docker compose exec db psql -U gearshare -d gearshare -c "SELECT id, email, password_hash FROM users;"
```

Normalement, le mot de passe n'apparaît nulle part en clair, et la réponse JSON ne contient ni
`password` ni `password_hash` : relisez-la attentivement.

Créez de la même façon un second compte, `bob@esiee.fr` : vous en aurez besoin à l'étape 5.

---

## Étape 3 — La connexion et le JWT

Complétez `app/core/security.py` avec la création et le décodage du jeton : HS256, `sub` sous
forme de chaîne, `iat`, `exp`, et une liste explicite d'algorithmes au décodage. Puis :

- dans le service, une méthode qui authentifie un utilisateur à partir de son email et de son mot
  de passe, et renvoie l'utilisateur ou rien ;
- `POST /auth/login`, qui lit un formulaire OAuth2 (le champ `username` porte l'email) et renvoie
  le jeton, ou un `401` avec l'en-tête `WWW-Authenticate: Bearer`.

**Vérifiez**, et gardez le jeton dans une variable :

```bash
curl -i -X POST http://localhost:8000/auth/login \
  -d "username=alice@esiee.fr&password=motdepasse-alice"

TOKEN_ALICE=$(curl -s -X POST http://localhost:8000/auth/login \
  -d "username=alice@esiee.fr&password=motdepasse-alice" \
  | python3 -c "import json, sys; print(json.load(sys.stdin)['access_token'])")
```

Puis **lisez son contenu**, sans aucune clé :

```bash
echo "$TOKEN_ALICE" | python3 -c "import base64, sys; p = sys.stdin.read().split('.')[1]; print(base64.urlsafe_b64decode(p + '=' * (-len(p) % 4)))"
```

(Vous pouvez aussi coller le jeton sur [jwt.io](https://jwt.io).)

**Vérifiez votre compréhension** :

5. Quels claims contient votre payload ? Y a-t-il quoi que ce soit qui ne devrait pas y être ?
6. Vous venez de lire le jeton sans clé. Qu'est-ce que la signature garantit, alors ?
7. À quelle date expire votre jeton ? Convertissez `exp` avec `date -r <exp>` (macOS) ou
   `date -d @<exp>` (Linux).

Testez enfin un mauvais mot de passe, puis un email inexistant. Les deux devraient renvoyer `401`
avec **exactement le même message** : comparez les deux réponses.

---

## Étape 4 — `get_current_user` et les routes protégées

Écrivez la dépendance `get_current_user` dans `app/api/deps.py`, et son alias `CurrentUser` :

- elle réutilise votre fonction de décodage, sans réécrire `jwt.decode` ;
- elle relit l'utilisateur **en base**, par le repository ;
- quelle que soit la cause (pas de jeton, jeton malformé, expiré, utilisateur disparu), elle
  renvoie le même `401` avec l'en-tête `WWW-Authenticate: Bearer`.

Ajoutez `GET /auth/me`, qui renvoie l'utilisateur connecté. Puis protégez vos routes :

| Route | Protection |
|---|---|
| `GET /items`, `GET /items/{id}` | Publiques |
| `POST /items` | Authentifié ; le propriétaire **vient du jeton** |
| `PUT`, `PATCH`, `DELETE /items/{id}` | Authentifié et propriétaire |
| Toutes les routes `/reservations` | Authentifié ; l'emprunteur **vient du jeton** |
| `GET /auth/me` | Authentifié |

Pensez à **retirer `owner_id` du schéma de création d'un item, et `borrower_id` de celui d'une
réservation** : tant qu'ils y sont, n'importe qui peut agir au nom de n'importe qui.

**Vérifiez** les trois cas :

```bash
# Sans jeton : 401 et en-tête WWW-Authenticate: Bearer
curl -i -X POST http://localhost:8000/items \
  -H "Content-Type: application/json" \
  -d '{"titre": "Vélo", "tarif_jour": 8.5}'

# Avec un jeton bidon : 401
curl -i -X POST http://localhost:8000/items \
  -H "Authorization: Bearer nimportequoi" \
  -H "Content-Type: application/json" \
  -d '{"titre": "Vélo", "tarif_jour": 8.5}'

# Avec le jeton d'Alice : 201
curl -i -X POST http://localhost:8000/items \
  -H "Authorization: Bearer $TOKEN_ALICE" \
  -H "Content-Type: application/json" \
  -d '{"titre": "Vélo", "tarif_jour": 8.5}'

docker compose exec db psql -U gearshare -d gearshare -c "SELECT id, titre, owner_id FROM items;"
```

Le dernier item devrait appartenir à Alice. Vérifiez aussi dans Swagger : le bouton « Authorize »
permet de se connecter, et un cadenas apparaît sur les routes protégées.

---

## Étape 5 — La propriété et le `403`

Récupérez le jeton de Bob comme à l'étape 3 (`TOKEN_BOB`). Avec ce jeton, supprimez l'item
d'Alice :

```bash
curl -i -X DELETE http://localhost:8000/items/<id-de-l-item-d-alice> \
  -H "Authorization: Bearer $TOKEN_BOB"
```

Si vous obtenez `204`, c'est une faille, et c'est tout à fait normal à ce stade. Ajoutez la
vérification de propriété **dans le service**, avec l'exception métier « accès refusé » de la
séance 4, et vérifiez qu'elle sort bien en `403`. Faites de même pour la modification d'un item
et pour l'annulation d'une réservation, autorisée à l'emprunteur et au propriétaire de l'item.

---

## Étape 6 — Les tests

Il reste à faire prouver la matrice par votre suite de tests. Si vous n'avez pas branché vos tests sur
la base de test en séance 4 (c'était dans « Pour aller plus loin »), commencez par là : créez la
base, ajoutez `TEST_DATABASE_URL` à votre configuration et les fixtures du cours de la séance 4.

```bash
docker compose exec db createdb -U gearshare gearshare_test
```

Puis, dans `tests/conftest.py`, ajoutez :

- une fixture `bob`, sur le modèle de `alice` ;
- deux fixtures `client_alice` et `client_bob`, qui portent chacune le jeton de leur utilisateur.
  Prenez soin d'en faire **des instances distinctes** du client de test : le cours explique ce
  qui se passe si elles partagent la fixture `client`.

Écrivez ensuite :

- pour chaque ligne de la matrice, y compris les règles que vous y avez ajoutées, trois tests : anonyme, propriétaire, autre utilisateur ;
- un test qui envoie l'`owner_id` de Bob dans le corps avec le jeton d'Alice, et vérifie que
  l'item créé appartient à Alice ;
- un test paramétré sur le login : mauvais mot de passe et email inexistant renvoient le même
  `401` et le même corps ;
- un test du service, sans HTTP, qui vérifie que la suppression de l'item d'un autre lève
  l'exception « accès refusé ».

```bash
docker compose run --rm backend pytest -v
```

**Une expérience pour juger vos tests** : retirez temporairement `CurrentUser` de la route
`DELETE /items/{id}`, et relancez la suite. Combien de tests échouent ? S'il n'y en a aucun, votre
suite ne protège pas cette route : complétez-la. Remettez ensuite la protection.

---

## Étape 7 — La connexion côté frontend

Le backend sait émettre et vérifier un jeton. Branchez maintenant le frontend, en suivant le cours
(section « Côté frontend »).

1. **Le client d'API.** Ajoutez la connexion et l'inscription à `ApiClient`, et faites passer le
   jeton **en paramètre** à chaque méthode qui en a besoin, plutôt que de le ranger dans
   l'instance : le cours explique pourquoi.
2. **La lecture du cookie.** Une petite dépendance lit le cookie `access_token` de la requête et
   le fournit aux routes.
3. **Les pages de connexion et d'inscription.** Formulaires, routes `GET` et `POST`, schéma
   Post/Redirect/Get. En cas de succès, la connexion pose le cookie : `HttpOnly`, `SameSite=Lax`,
   `Secure` selon `COOKIE_SECURE`, et une durée égale à `expires_in`. En cas d'échec, elle
   réaffiche le formulaire avec l'email saisi, jamais le mot de passe.
4. **La déconnexion.** Un bouton dans un formulaire `POST`, qui supprime le cookie, plutôt qu'un lien.
5. **Le retour après connexion.** La page de connexion accepte un paramètre `suivant` ; vérifiez
   qu'il désigne bien une page de votre site avant d'y rediriger.
6. **Le `401`.** Votre gestionnaire d'exceptions de la séance 4 renvoie vers la page de connexion
   quand le backend répond `401`, et supprime le cookie au passage.
7. **Les formulaires existants.** La réservation et la création d'un item transmettent le jeton.
   Supprimez l'`owner_id` provisoire marqué `# TODO séance 5` en séance 4.
8. **Le gabarit commun.** Il affiche « Se connecter » ou un bouton « Se déconnecter » selon la
   présence du cookie.

**Vérifiez dans le navigateur** :

- connectez-vous, puis ouvrez les outils de développement (onglet *Application* ou *Stockage*,
  rubrique *Cookies*) : le cookie `access_token` est là, avec `HttpOnly` et `SameSite=Lax` ;
- déconnecté, essayez de réserver : vous arrivez sur la page de connexion ;
- ouvrez `http://localhost:8080/connexion?suivant=https://example.com` et connectez-vous : vous
  restez normalement sur votre site ;
- **le test le plus instructif** : connectez-vous en tant qu'Alice dans une fenêtre normale, et en tant que
  Bob dans une fenêtre de navigation privée. Créez un item dans chacune, puis vérifiez en base que
  chaque item a le bon propriétaire.

```bash
docker compose exec db psql -U gearshare -d gearshare -c "SELECT id, titre, owner_id FROM items ORDER BY id DESC LIMIT 5;"
```

Si les deux items appartiennent à la même personne, le jeton est partagé entre les utilisateurs
quelque part dans votre frontend.

---

## Étape 8 — Exploiter vos propres failles

Trois expériences courtes. Le but est de *voir* le problème, pas seulement d'en entendre parler.

### 8.1. Forger un jeton avec un secret faible

Mettez temporairement `JWT_SECRET=secret` dans votre `.env`, puis redémarrez le backend :

```bash
docker compose up -d backend
```

Sur [jwt.io](https://jwt.io), collez le jeton d'Alice, remplacez son `sub` par l'identifiant de
Bob, et saisissez `secret` comme clé de signature. Utilisez le jeton obtenu sur `GET /auth/me`.

8. Qui êtes-vous pour l'API ? Que se passerait-il si votre vrai `JWT_SECRET` était commité dans le
   dépôt ?

Remettez ensuite votre vrai secret, et redémarrez.

### 8.2. Le jeton expiré

Mettez `JWT_EXPIRE_MINUTES=1`, redémarrez, connectez-vous dans le navigateur, attendez un peu plus
d'une minute, puis rechargez une page qui demande d'être connecté. Vous devriez revenir sur la page
de connexion, et `GET /auth/me` avec ce jeton devrait renvoyer `401`. Remettez `60`.

### 8.3. L'utilisateur supprimé

Inscrivez un troisième compte, `carol@esiee.fr`, sans lui créer d'item ni de réservation.
Récupérez son jeton, puis supprimez-la en base :

```bash
docker compose exec db psql -U gearshare -d gearshare -c "DELETE FROM users WHERE email = 'carol@esiee.fr';"
```

Rappelez `GET /auth/me` avec son ancien jeton. Attendu : `401`. Si vous obtenez `200` ou `500`,
votre `get_current_user` ne relit pas correctement l'utilisateur en base.

---

## Étape 9 — La relecture finale

Avant de rendre, relisez votre code avec cette liste, et corrigez ce qui ne va pas :

- [ ] `JWT_SECRET` hors du code, 32 octets aléatoires ou plus
- [ ] `.env` absent du dépôt **et de l'historique** Git
- [ ] Le frontend ne reçoit pas `JWT_SECRET`
- [ ] Hachage PBKDF2-HMAC-SHA256, 600 000 itérations, sel tiré par `secrets`
- [ ] Comparaison des condensats en temps constant
- [ ] `password_hash` absent de tous les schémas de lecture
- [ ] Payload du JWT limité à `sub`, `iat`, `exp`
- [ ] `exp` présent **et** vérifié
- [ ] Liste explicite d'algorithmes au décodage
- [ ] `get_current_user` relit l'utilisateur en base
- [ ] Même `401` quelle que soit la cause, avec `WWW-Authenticate`
- [ ] Toutes les routes d'écriture protégées
- [ ] L'identité vient du jeton, jamais du corps
- [ ] Propriété vérifiée dans le service, `403` renvoyé
- [ ] Message de login identique pour un email inconnu et un mauvais mot de passe
- [ ] Longueur minimale et maximale du mot de passe
- [ ] Aucun mot de passe dans les logs
- [ ] Jeton passé en paramètre au client d'API, jamais stocké dans l'instance
- [ ] Cookie `HttpOnly`, `SameSite=Lax`, `Secure` configurable
- [ ] Aucune modification par `GET` dans le frontend, déconnexion comprise
- [ ] Paramètre `suivant` limité aux pages du site

Terminez par une recherche de secrets oubliés :

```bash
git grep -niE "secret|password|token" -- '*.py' '*.yml' '*.yaml' | grep -v "settings\.\|password_hash\|hash_password\|verify_password"
git log --all --oneline -- .env
```

Si la première commande affiche encore des lignes, expliquez-les en une phrase ; la seconde
n'affiche normalement rien.

---

## Ce que vous devez me rendre à la fin du TP

Sur le dépôt Git de votre projet :

1. **Inscription et connexion fonctionnelles**, mots de passe hachés en base.
2. **`get_current_user`** et toutes les routes d'écriture protégées ; l'identité tirée du jeton.
3. **La propriété vérifiée** dans les services, avec `403` sur violation.
4. **Les tests** : la matrice d'autorisation, le login et la règle de propriété, qui passent avec
   `docker compose run --rm backend pytest`.
5. **Le frontend connecté** : connexion, inscription, déconnexion, retour à la page de connexion
   sur un `401`, et des formulaires qui transmettent le jeton.
6. **`.env.example` à jour** (avec un `JWT_SECRET` factice) et **`.env` absent du dépôt**.

## Pour aller plus loin

- **Rejeter les champs inconnus** : configurez vos schémas de création pour refuser tout champ
  qu'ils ne déclarent pas. Que renvoie alors le test de l'étape 6 qui envoie un `owner_id` ?
- **Re-hacher au login** : si le condensat stocké utilise moins d'itérations que la valeur
  actuelle, recalculez-le avec le mot de passe que vous venez de vérifier. Écrivez le test qui le
  prouve.
- **L'attaque temporelle sur le login** : mesurez le temps de réponse de `/auth/login` pour un
  email inexistant et pour un mauvais mot de passe. Corrigez l'écart en hachant un mot de passe
  factice quand l'utilisateur n'existe pas, et mesurez à nouveau.
- **Le jeton CSRF** : ajoutez un jeton CSRF à vos formulaires, en plus de `SameSite=Lax`, et
  documentez ce qu'il apporte.
- **Le refresh token** : un jeton d'accès de 15 minutes et un jeton de rafraîchissement de 7 jours,
  stocké en base et révocable. Expliquez ce que cela change en cas de vol de jeton.
- **La limitation de débit** : 5 tentatives de connexion par minute et par adresse IP. Discutez de
  ses limites derrière un proxy.
