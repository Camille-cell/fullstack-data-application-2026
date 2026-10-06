# Cours 1 — Séance 4 : l'architecture de l'application et le frontend

Vous avez une API sans mémoire (séance 1) que seul un développeur sait utiliser, avec `/docs` ou
`curl`. Ce premier cours pose l'**architecture globale** de votre application (frontend, backend,
base de données, et la configuration de chacun) puis vous apprend à écrire le **frontend** : une
seconde application FastAPI qui sert des pages HTML et ne parle au backend que par son API.

Le [cours 2](cours2-backend-db.md) traite ensuite l'intérieur du backend et son branchement sur
PostgreSQL.

- [Cours 1 — Séance 4 : l'architecture de l'application et le frontend (45 min)](#cours-1--séance-4--larchitecture-de-lapplication-et-le-frontend-45-min)
  - [Objectifs pédagogiques](#objectifs-pédagogiques)
  - [1. L'architecture de l'application](#1-larchitecture-de-lapplication)
    - [1.1. Vue d'ensemble : trois services, une seule direction](#11-vue-densemble--trois-services-une-seule-direction)
    - [1.2. Configurer chaque service : les variables d'environnement](#12-configurer-chaque-service--les-variables-denvironnement)
    - [1.3. Les couches du frontend](#13-les-couches-du-frontend)
  - [2. Le frontend : une application FastAPI qui sert du HTML](#2-le-frontend--une-application-fastapi-qui-sert-du-html)
    - [2.1. Le principe : le rendu côté serveur](#21-le-principe--le-rendu-côté-serveur)
    - [2.2. La structure du projet frontend](#22-la-structure-du-projet-frontend)
    - [2.3. HTML : l'essentiel](#23-html--lessentiel)
    - [2.4. Les templates Jinja2](#24-les-templates-jinja2)
    - [2.5. Le client d'API](#25-le-client-dapi)
    - [2.6. Les routes de pages](#26-les-routes-de-pages)
    - [2.7. Les formulaires : recevoir, transmettre, rediriger](#27-les-formulaires--recevoir-transmettre-rediriger)
    - [2.8. Un peu de CSS](#28-un-peu-de-css)
    - [2.9. Ce que le frontend ne fait pas](#29-ce-que-le-frontend-ne-fait-pas)
  - [3. Synthèse](#3-synthèse)

## Objectifs pédagogiques

À la fin de ce cours, vous devez être capables de :

- décrire l'architecture globale de l'application (frontend, backend, base) et la responsabilité
  de chaque service ;
- configurer chaque service par variables d'environnement, en ne lui donnant que ce dont il a
  besoin ;
- écrire un frontend FastAPI qui sert des pages HTML (templates Jinja2, formulaires, un peu de
  CSS) et ne communique avec le backend que par l'API ;
- traiter un formulaire avec le schéma Post/Redirect/Get et traduire les erreurs de l'API en
  messages lisibles ;
- reviewer un frontend généré par un agent.

## 1. L'architecture de l'application

Avant d'écrire la moindre ligne, il faut savoir **où** chaque morceau de code va vivre. Votre
application a deux niveaux d'architecture, à ne pas confondre :

- l'**architecture globale** : quels services tournent, qui parle à qui, qui est responsable de
  quoi — c'est l'objet de cette partie ;
- l'**architecture interne du backend** : comment le code de l'API est découpé en couches — c'est
  l'objet du [cours 2](cours2-backend-db.md).

Cette partie ne contient volontairement aucun code : c'est le plan de l'application. Le code vient
ensuite, couche par couche. Si vous ne savez pas expliquer ce plan à l'oral, vous ne saurez pas
non plus l'expliquer à un agent.

### 1.1. Vue d'ensemble : trois services, une seule direction

Votre projet tourne sous la forme de **trois services**, chacun dans son propre conteneur, avec
ses propres dépendances et son propre cycle de vie :

```text
┌─────────────┐  HTML   ┌─────────────────┐  JSON   ┌─────────────────┐  SQL   ┌──────────────┐
│ Navigateur  │ ──────► │    frontend     │ ──────► │     backend     │ ─────► │  PostgreSQL  │
│ (l'humain)  │ ◄────── │ FastAPI + HTML  │ ◄────── │ FastAPI, API    │ ◄───── │              │
│             │         │     :8080       │         │ REST  :8000     │        │    :5432     │
└─────────────┘         └─────────────────┘         └─────────────────┘        └──────────────┘
                          présentation                règles métier             stockage et
                                                      et persistance            intégrité
```

Le navigateur n'est pas un service que vous déployez, mais il fait partie du tableau : c'est lui
qui affiche les pages et envoie les formulaires.

**Client et serveur : un rôle, pas une machine.** Dans tout échange HTTP, le **client** est celui
qui envoie la requête, le **serveur** celui qui la reçoit et répond. Ce n'est pas une propriété
d'un programme, c'est le rôle qu'il joue dans un échange donné. Le frontend en est l'exemple
parfait : il est **serveur** pour le navigateur (il reçoit ses requêtes et lui renvoie du HTML),
et **client** du backend (il lui envoie des requêtes et reçoit du JSON). De la même façon, le
backend est serveur pour le frontend et client de PostgreSQL. Votre API, elle, peut avoir autant
de clients que nécessaire : le frontend, mais aussi `curl`, la page `/docs`, un script d'import ou
le `TestClient` de vos tests.

| Service | Sa responsabilité | Il parle à | Il ne fait **jamais** |
|---|---|---|---|
| **frontend** | Produire les pages HTML, recevoir les formulaires, appeler l'API, transformer les réponses (et les erreurs) en quelque chose de lisible pour un humain | le navigateur (en entrée), le backend (en sortie) | accéder à la base, décider d'une règle métier |
| **backend** | Exposer l'API REST, valider toute entrée, appliquer les règles métier et les droits d'accès, lire et écrire en base | le frontend (et tout autre client), la base | produire du HTML, faire confiance à ce que le client lui envoie |
| **db** | Stocker les données de façon durable, garantir leur intégrité (clés étrangères, contraintes, unicité, transactions) | le backend uniquement | être joignable depuis l'extérieur |

Détaillons chacun.

**Le frontend est la couche de présentation.** Quand on ouvre la page « Objets disponibles »,
c'est le frontend qui reçoit la requête du navigateur. Il ne sait pas lui-même quels objets
existent : il le demande au backend, reçoit du JSON, et l'injecte dans un gabarit HTML qu'il
renvoie au navigateur. Quand on soumet un formulaire de réservation, le frontend récupère les
champs, les transmet au backend, puis affiche soit une confirmation, soit un message d'erreur
compréhensible (« ce créneau est déjà réservé » plutôt que « 409 »). Il peut faire de petites
vérifications de confort (un champ obligatoire, une date de fin après la date de début) pour
éviter un aller-retour inutile, mais **ces vérifications ne protègent rien** : elles améliorent
l'expérience, c'est tout.

**Le backend est la source de vérité.** C'est lui qui sait qu'un objet indisponible ne peut pas
être réservé, que deux réservations ne peuvent pas se chevaucher, qu'un utilisateur ne peut
modifier que ses propres objets. Il revalide **tout** ce qu'il reçoit, même si le frontend l'a
déjà vérifié, parce que le frontend n'est pas son seul client possible : n'importe qui peut
appeler l'API avec `curl`, un script ou une autre application. Le backend publie son **contrat**
(la documentation OpenAPI accessible sur `/docs`) : c'est ce contrat, et rien d'autre, que le
frontend a le droit de connaître.

**La base de données est la dernière ligne de défense.** Elle ne connaît pas vos règles métier
au sens large, mais elle garantit ce qu'elle peut garantir : une réservation pointe vers un objet
qui existe, un tarif est positif, un email est unique, une transaction est appliquée en entier ou
pas du tout. Si un bug du backend tente d'écrire une donnée incohérente, c'est elle qui refuse.
Elle n'est accessible qu'au backend, sur le réseau interne de Docker Compose : aucun port n'a
besoin d'être ouvert vers l'extérieur.

**Suivez une action de bout en bout.** On veut réserver une perceuse du 12 au 14 :

1. Le navigateur envoie le formulaire au **frontend**.
2. Le frontend lit les champs et appelle l'API du **backend** : « crée une réservation pour
   l'objet 42, du 12 au 14 ».
3. Le backend valide le format de la demande, vérifie que l'objet existe, qu'il est disponible et
   que le créneau est libre, puis enregistre la réservation dans la **base**.
4. La base vérifie ses contraintes et confirme l'écriture.
5. Le backend répond au frontend : « réservation créée », avec ses données en JSON.
6. Le frontend redirige le navigateur vers une page de confirmation, en HTML.

Si le créneau est pris, l'étape 3 s'arrête : le backend répond par une erreur `409`, et le
frontend la traduit en message lisible sur le formulaire, sans jamais savoir *comment* le backend
a détecté le conflit.

**Pourquoi séparer autant ?**

- **Un contrat explicite.** Puisque le frontend ne passe que par l'API, celle-ci doit être
  complète, cohérente et documentée. Vous ne pouvez pas « tricher » en allant lire la base
  directement.
- **Plusieurs clients possibles.** Demain, une application mobile, un script d'import ou un
  partenaire peuvent utiliser la même API, sans rien dupliquer. C'est l'architecture de la plupart
  des applications d'entreprise.
- **Des évolutions indépendantes.** Vous pouvez refaire toute l'interface sans toucher au
  backend, ou optimiser une requête SQL sans toucher au frontend, tant que le contrat ne change
  pas.
- **Une surface d'attaque réduite.** La base n'est jamais exposée ; le seul point d'entrée vers
  les données est un backend qui contrôle tout.
- **Une validation en profondeur.** Trois niveaux de contrôle, chacun avec son rôle : confort
  (frontend), règles (backend), intégrité (base).

**La règle d'or : les appels vont dans un seul sens.** Le frontend appelle le backend, le backend
appelle la base. Jamais l'inverse, et jamais de raccourci : un frontend qui ouvre une connexion à
PostgreSQL « juste pour cette page » détruit tout ce qui précède. C'est un point que je vérifierai
dans vos projets.

### 1.2. Configurer chaque service : les variables d'environnement

Chaque service a besoin de **configuration** : l'adresse de la base, un mot de passe, l'URL du
backend, une clé secrète. Cette configuration ne s'écrit **jamais dans le code** ni dans l'image
Docker : elle est fournie au conteneur, au démarrage, sous forme de **variables
d'environnement**. La même image peut ainsi tourner sur votre machine, dans la CI et en
production ; seules les variables changent. C'est le principe *12-factor* qu'on détaille en
[section 7 du cours 2](cours2-backend-db.md#7-configuration--le-12-factor).

Voici ce dont chaque service a besoin :

| Service | Variable | Rôle | Secret ? |
|---|---|---|---|
| **db** | `POSTGRES_USER` | Utilisateur PostgreSQL créé au premier démarrage | non |
| | `POSTGRES_PASSWORD` | Son mot de passe | **oui** |
| | `POSTGRES_DB` | Nom de la base créée au premier démarrage | non |
| **backend** | `DATABASE_URL` | Adresse complète de la base : driver, utilisateur, mot de passe, hôte `db`, port, nom de base | **oui** (contient le mot de passe) |
| | `JWT_SECRET` | Clé de signature des jetons (séance 5) | **oui** |
| | `JWT_EXPIRE_MINUTES` | Durée de validité d'un jeton | non |
| | `SQL_ECHO` | Affiche le SQL généré, pour le débogage | non |
| **frontend** | `API_BASE_URL` | Adresse du backend, `http://backend:8000` dans Compose | non |

Les variables du service `db` sont lues par l'image officielle `postgres`, et seulement **au
premier démarrage**, quand le volume de données est vide. Si vous changez le mot de passe dans
`.env` après coup, la base existante garde l'ancien : il faut supprimer le volume pour la
réinitialiser.

**D'où viennent ces valeurs ?** Docker Compose propose trois mécanismes, qu'il ne faut pas
confondre :

- **`environment:`** dans `docker-compose.yml` : la valeur est écrite directement dans le fichier,
  service par service. C'est adapté aux valeurs **non secrètes** qui décrivent l'architecture
  elle-même, comme `API_BASE_URL: http://backend:8000`.
- **`env_file:`** : Compose charge toutes les variables d'un fichier (typiquement `.env`) et les
  injecte dans le conteneur. C'est le moyen de fournir les secrets sans les écrire dans le YAML.
- **la substitution `${VARIABLE}`** : avant même de lancer les conteneurs, Compose lit
  automatiquement le fichier `.env` placé à côté du `docker-compose.yml` et remplace les
  `${...}` du YAML par leurs valeurs. C'est ainsi que les identifiants de la base sont transmis au
  service `db` sans être écrits en clair dans le fichier commité.

Le fichier `.env` contient donc les vraies valeurs, **n'est jamais commité** (il est dans le
`.gitignore`), et un fichier `.env.example`, lui commité, liste toutes les variables attendues
avec des valeurs factices. Un nouveau membre du groupe copie `.env.example` en `.env`, ajuste, et
lance `docker compose up`.

Trois règles à respecter :

- **Chaque service ne reçoit que ce dont il a besoin.** Le frontend n'a besoin que de
  `API_BASE_URL` : il ne doit recevoir ni `DATABASE_URL`, ni `JWT_SECRET`. Lui donner tout le
  fichier `.env` « par simplicité », c'est offrir le mot de passe de la base à un service qui
  n'a aucune raison de le connaître, et qui est le plus exposé des trois.
- **Les noms d'hôte sont des noms de service.** Entre conteneurs, la base s'appelle `db` et le
  backend `backend`, résolus par le DNS interne de Compose. `localhost` désigne le conteneur
  lui-même : une `DATABASE_URL` pointant sur `localhost` fonctionne quand vous lancez le backend
  sur votre machine, et échoue dans Docker. Depuis votre machine, c'est l'inverse : vous passez
  par `localhost` et les ports publiés.
- **Les valeurs doivent rester cohérentes.** L'utilisateur, le mot de passe et le nom de base
  contenus dans `DATABASE_URL` doivent être ceux de `POSTGRES_USER`, `POSTGRES_PASSWORD` et
  `POSTGRES_DB`. Une incohérence ne se voit qu'au démarrage du backend, par un refus de
  connexion.

Enfin, chaque application vérifie sa configuration **au démarrage** : si une variable obligatoire
manque, elle refuse de démarrer avec un message clair, plutôt que de planter au premier appel. On
verra comment dans le [cours 2](cours2-backend-db.md#7-configuration--le-12-factor) (section 7), ainsi que le
`docker-compose.yml` complet à trois services (section 8).

### 1.3. Les couches du frontend

Le frontend a lui aussi un découpage interne, plus léger que celui du backend puisqu'il n'a ni
règle métier ni base de données :

- les **routes de pages** : une URL vue par le navigateur, une page HTML renvoyée ;
- le **client d'API** : le seul module qui sait parler au backend (adresses, appels HTTP,
  traduction des erreurs) ;
- les **templates** : le HTML, avec des emplacements pour les données ;
- les **fichiers statiques** : la feuille de style, les images.

Chaque élément a une seule responsabilité : une route de page ne construit pas d'appel HTTP à la
main, elle passe par le client d'API ; un template n'appelle rien, il affiche ce qu'on lui donne.
Vous retrouverez le même principe, plus poussé, dans les couches du backend (cours 2). Le détail
du frontend est l'objet de la partie suivante.

## 2. Le frontend : une application FastAPI qui sert du HTML

Votre frontend est une **deuxième application FastAPI**, distincte du backend. Au lieu de
renvoyer du JSON, ses routes renvoient des **pages HTML** construites côté serveur à partir de
templates. Vous connaissez déjà FastAPI : vous n'apprenez ici que la partie « présentation ».

### 2.1. Le principe : le rendu côté serveur

```text
Navigateur                      frontend (:8080)                  backend (:8000)
    │  GET /items                     │                                 │
    │ ──────────────────────────────► │  GET /items                     │
    │                                 │ ──────────────────────────────► │
    │                                 │ ◄────────────────────────────── │
    │                                 │  200 [{"id": 1, ...}, ...]      │
    │                                 │                                 │
    │                                 │  remplit le template            │
    │ ◄────────────────────────────── │  items/list.html                │
    │  200 <html>...</html>           │                                 │
```

Le navigateur ne parle **qu'au frontend** et ne reçoit que du HTML tout prêt. C'est le frontend,
côté serveur, qui appelle le backend. Deux conséquences agréables :

- **pas de JavaScript à écrire** : tout reste en Python, vos pages fonctionnent avec du HTML et
  des formulaires standard ;
- **pas de CORS à configurer** : le navigateur n'appelle jamais l'API directement, l'appel
  frontend → backend est un appel de serveur à serveur, auquel les règles CORS du navigateur ne
  s'appliquent pas.

### 2.2. La structure du projet frontend

```text
frontend/
├── Dockerfile
├── pyproject.toml          # fastapi, uvicorn, jinja2, httpx, python-multipart
└── app/
    ├── main.py             # création de l'app, montage des fichiers statiques
    ├── api_client.py       # le SEUL module qui appelle le backend
    ├── templating.py       # configuration Jinja2, partagée par les routes
    ├── routers/
    │   └── items.py        # routes de pages
    ├── templates/
    │   ├── base.html       # squelette commun à toutes les pages
    │   ├── erreur.html
    │   └── items/
    │       ├── list.html
    │       └── detail.html
    └── static/
        └── style.css
```

Quatre dépendances en plus de FastAPI : `uvicorn` pour servir l'application, `jinja2` pour les
templates, `httpx` pour appeler le backend, `python-multipart` pour lire les formulaires HTML.
Aucune dépendance vers SQLAlchemy ou un driver PostgreSQL : **si l'une d'elles apparaît dans le
`pyproject.toml` du frontend, il y a un problème d'architecture.**

### 2.3. HTML : l'essentiel

HTML décrit la **structure** d'une page : des titres, des paragraphes, des listes, des tableaux,
des formulaires. Il ne décrit pas son apparence (c'est le rôle de CSS, section 2.8). Un document
HTML minimal :

```html
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>GearShare — Objets</title>
  <link rel="stylesheet" href="/static/style.css">
</head>
<body>
  <header>
    <nav>
      <a href="/">GearShare</a>
      <a href="/items">Objets</a>
    </nav>
  </header>

  <main>
    <h1>Objets disponibles</h1>
    <p>Empruntez du matériel entre voisins.</p>
  </main>

  <footer>
    <p>Projet Fullstack data application — ESIEE</p>
  </footer>
</body>
</html>
```

- `<head>` contient les métadonnées (encodage, titre de l'onglet, feuille de style) ; `<body>`
  contient ce qui s'affiche.
- `<meta charset="utf-8">` évite les accents cassés ; la balise `viewport` rend la page lisible
  sur mobile.
- `<header>`, `<nav>`, `<main>`, `<footer>` sont des balises **sémantiques** : elles décrivent le
  rôle de chaque zone. Préférez-les à des `<div>` anonymes, votre CSS et l'accessibilité y gagnent.

Les balises dont vous aurez besoin pour le projet tiennent dans ce tableau :

| Balise | Usage |
|---|---|
| `<h1>` à `<h3>` | Titres, un seul `<h1>` par page |
| `<p>` | Paragraphe |
| `<a href="...">` | Lien vers une autre page (navigation, toujours en `GET`) |
| `<ul>` / `<li>` | Liste |
| `<table>`, `<thead>`, `<tbody>`, `<tr>`, `<th>`, `<td>` | Tableau de données |
| `<form>`, `<label>`, `<input>`, `<textarea>`, `<select>`, `<button>` | Formulaire |
| `<div>`, `<section>`, `<article>` | Regroupement de contenu |

**Les formulaires** méritent qu'on s'y arrête, car ce sont eux qui transportent les actions de
l'utilisateur :

```html
<form method="post" action="/items/42/reservations">
  <label for="date_debut">Du</label>
  <input type="date" id="date_debut" name="date_debut" required>

  <label for="date_fin">Au</label>
  <input type="date" id="date_fin" name="date_fin" required>

  <button type="submit">Réserver</button>
</form>
```

- `method` vaut `get` ou `post` : **un formulaire HTML ne connaît que ces deux méthodes**. Pas de
  `PUT`, de `PATCH` ni de `DELETE` côté navigateur ; c'est le frontend qui appellera la bonne
  méthode sur l'API. Gardez pour autant des URL de ressources, sans verbe : un formulaire posté
  sur `/reservations/7` avec un champ caché `statut=annulee` devient, côté backend,
  `PATCH /reservations/7` avec `{"statut": "annulee"}`. Une URL comme `/reservations/7/annuler`
  réintroduit le verbe que REST bannit (séance 1).
- `action` est l'URL **du frontend** qui reçoit le formulaire.
- `name` est la clé sous laquelle la valeur sera envoyée. Un champ sans `name` n'est pas transmis.
- `<label for="...">` relié à l'`id` du champ : cliquer sur le libellé active le champ, et les
  lecteurs d'écran savent ce que le champ attend.
- `type="date"`, `type="number"`, `type="email"`, `type="password"` et `required` donnent une
  validation gratuite dans le navigateur. Rappelez-vous : **c'est du confort, pas de la
  sécurité.** Le backend revalide tout.

Règle à respecter : **un lien ne modifie jamais rien.** Toute action qui crée, modifie ou supprime
passe par un formulaire en `POST`. Un navigateur, un moteur de recherche ou un antivirus peuvent
suivre un lien tout seul ; ils ne soumettent pas un formulaire.

### 2.4. Les templates Jinja2

Écrire le HTML à la main dans des chaînes Python devient vite illisible. Un **template** est un
fichier HTML avec des emplacements que le serveur remplit au moment de la requête. FastAPI
s'appuie sur **Jinja2** pour cela.

La syntaxe tient en trois éléments :

| Syntaxe | Rôle | Exemple |
|---|---|---|
| `{{ ... }}` | Afficher une valeur | `{{ item.titre }}` |
| `{% ... %}` | Instruction (boucle, condition, bloc) | `{% for item in items %}` |
| `{# ... #}` | Commentaire, absent du HTML produit | `{# TODO : pagination #}` |

**L'héritage de templates** évite de recopier l'en-tête et le pied de page sur chaque page. Le
template de base définit le squelette et des **blocs** à remplir :

```html
{# app/templates/base.html #}
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>{% block title %}GearShare{% endblock %}</title>
  <link rel="stylesheet" href="{{ url_for('static', path='style.css') }}">
</head>
<body>
  <header>
    <nav>
      <a href="/" class="logo">GearShare</a>
      <a href="/items">Objets</a>
    </nav>
  </header>
  <main>
    {% block content %}{% endblock %}
  </main>
</body>
</html>
```

Chaque page **étend** ce squelette et ne remplit que ses blocs :

```html
{# app/templates/items/list.html #}
{% extends "base.html" %}

{% block title %}Objets — GearShare{% endblock %}

{% block content %}
<h1>Objets disponibles</h1>

{% if items %}
  <table>
    <thead>
      <tr><th>Objet</th><th>Tarif / jour</th><th>Statut</th></tr>
    </thead>
    <tbody>
      {% for item in items %}
        <tr>
          <td><a href="/items/{{ item.id }}">{{ item.titre }}</a></td>
          <td>{{ item.tarif_jour }} €</td>
          <td>{{ "Disponible" if item.disponible else "Indisponible" }}</td>
        </tr>
      {% endfor %}
    </tbody>
  </table>
{% else %}
  <p>Aucun objet pour le moment.</p>
{% endif %}
{% endblock %}
```

### 2.5. Le client d'API

**Qu'est-ce qu'un client ?** Dès qu'une application en appelle une autre, une question se pose :
où écrire le code qui fait ces appels ? La réponse habituelle est un **client** : un composant
qui est la **porte d'entrée unique** vers l'autre application. Tout le reste du code passe par
lui, et lui seul sait comment l'atteindre.

Un client a toujours les mêmes responsabilités :

- **savoir où est l'autre application** : son adresse, ses paramètres de connexion ;
- **parler son protocole** : construire les requêtes HTTP, les URL, les en-têtes, le JSON ;
- **traduire ses réponses** dans le langage de votre application : des données Python en cas de
  succès, une exception claire en cas d'échec, que l'erreur vienne de l'autre application ou du
  réseau ;
- **offrir des opérations nommées d'après le métier** (`list_items`, `create_reservation`) plutôt
  que d'après la technique (`GET /items`, `POST /reservations`).

Vous en utilisez déjà sans le savoir : `httpx.Client` est un client HTTP générique, le
`TestClient` de FastAPI est un client de votre API pour les tests, et un driver PostgreSQL est un
client de la base. Ce qu'on écrit ici est un client **spécifique** : il ne sait parler qu'à votre
backend, mais il le fait dans les termes de votre application.

Techniquement, un client peut prendre plusieurs formes : un **module** Python qui expose des
fonctions, une **classe** dont on crée une instance, voire un paquet complet quand l'API est grosse
(c'est ce que publient Stripe, GitHub ou AWS pour leurs propres API). L'essentiel n'est pas la
forme, c'est la règle : **un seul point d'entrée**.

Pour notre projet, on choisit **une classe**, `ApiClient`. Elle regroupe en un seul objet l'état
du client (l'adresse du backend, la connexion HTTP, le timeout) et les opérations qu'il propose ;
sa configuration est passée explicitement au constructeur plutôt que lue en cachette au moment de
l'import ; et on peut en créer une autre instance, pointant ailleurs ou remplacée par une fausse,
dans les tests.

Les routes de pages ne manipulent donc jamais `httpx` directement : elles appellent les méthodes
de `ApiClient`.

```python
# app/api_client.py
import os
from datetime import date
from typing import Any

import httpx


class ApiError(Exception):
    """Erreur renvoyée par le backend, ou backend injoignable."""

    def __init__(self, status_code: int, detail: Any) -> None:
        super().__init__(f"{status_code}: {detail}")
        self.status_code = status_code
        self.detail = detail


class ApiClient:
    """Porte d'entrée unique du frontend vers l'API du backend."""

    def __init__(self, base_url: str, timeout: float = 10) -> None:
        self._http = httpx.Client(base_url=base_url, timeout=timeout)

    def _send(self, method: str, url: str, **kwargs: Any) -> Any:
        try:
            response = self._http.request(method, url, **kwargs)
        except httpx.RequestError as exc:
            raise ApiError(503, "Le service est momentanément indisponible.") from exc
        if response.is_error:
            try:
                detail = response.json().get("detail")
            except ValueError:
                detail = response.text
            raise ApiError(response.status_code, detail)
        return response.json() if response.content else None

    def list_items(self, limit: int = 20) -> list[dict[str, Any]]:
        return self._send("GET", "/items", params={"limit": limit})

    def get_item(self, item_id: int) -> dict[str, Any]:
        return self._send("GET", f"/items/{item_id}")

    def create_reservation(self, item_id: int, date_debut: date, date_fin: date) -> dict[str, Any]:
        return self._send(
            "POST",
            "/reservations",
            json={"item_id": item_id, "date_debut": date_debut.isoformat(), "date_fin": date_fin.isoformat()},
        )


api_client = ApiClient(base_url=os.environ["API_BASE_URL"])
```

La dernière ligne crée **l'instance unique** utilisée par toute l'application. C'est le seul
endroit où l'adresse du backend est lue.

Ce qu'apporte ce découpage :

- **une seule adresse à configurer** : `API_BASE_URL`, lue dans l'environnement
  (`http://backend:8000` dans Docker Compose, voir la section 1.2) ;
- **une seule connexion réutilisée** : l'instance garde son `httpx.Client`, qui réutilise les
  connexions au backend d'une requête à l'autre au lieu d'en ouvrir une à chaque appel ;
- **un timeout systématique** : sans lui, un backend bloqué bloque aussi le frontend, indéfiniment ;
- **une seule exception à gérer** dans les routes, `ApiError`, qu'il s'agisse d'une erreur `4xx`
  du backend ou d'un backend arrêté ;
- **un seul endroit à modifier** en séance 5, quand il faudra ajouter le jeton d'authentification
  à chaque appel.

### 2.6. Les routes de pages

Une route de page fait toujours la même chose : appeler le client d'API, puis rendre un template
avec le résultat.

```python
# app/templating.py
from pathlib import Path

from fastapi.templating import Jinja2Templates

BASE_DIR = Path(__file__).parent

templates = Jinja2Templates(directory=BASE_DIR / "templates")
```

```python
# app/main.py
from fastapi import FastAPI, Request
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles

from app.api_client import ApiError
from app.routers import items
from app.templating import BASE_DIR, templates

app = FastAPI(title="GearShare — frontend")
app.mount("/static", StaticFiles(directory=BASE_DIR / "static"), name="static")
app.include_router(items.router)


@app.exception_handler(ApiError)
def api_error_page(request: Request, exc: ApiError) -> HTMLResponse:
    messages = {
        404: "Cette page n'existe pas ou plus.",
        503: "Le service est momentanément indisponible. Réessayez dans un instant.",
    }
    message = messages.get(exc.status_code, "Une erreur est survenue.")
    return templates.TemplateResponse(
        request, "erreur.html", {"message": message}, status_code=exc.status_code
    )
```

```python
# app/routers/items.py
from fastapi import APIRouter, Request
from fastapi.responses import HTMLResponse

from app.api_client import api_client
from app.templating import templates

router = APIRouter()


@router.get("/items", response_class=HTMLResponse)
def list_items_page(request: Request) -> HTMLResponse:
    items = api_client.list_items()
    return templates.TemplateResponse(request, "items/list.html", {"items": items})


@router.get("/items/{item_id}", response_class=HTMLResponse)
def item_detail_page(request: Request, item_id: int) -> HTMLResponse:
    item = api_client.get_item(item_id)
    return templates.TemplateResponse(request, "items/detail.html", {"item": item})
```

Notez :

- `TemplateResponse` reçoit la **requête** en premier argument : Jinja2 en a besoin pour
  `url_for`, notamment pour retrouver les fichiers statiques ;
- le **gestionnaire d'exceptions global** évite de répéter un `try/except` dans chaque route de
  lecture : un objet inexistant (`404` renvoyé par le backend) affiche une page « introuvable »
  propre, un backend arrêté une page « service indisponible » — jamais une stack trace ;
- le frontend ne **décide** rien : si l'objet n'existe pas, c'est le backend qui l'a dit.

### 2.7. Les formulaires : recevoir, transmettre, rediriger

Le traitement d'un formulaire suit un schéma fixe, appelé **Post/Redirect/Get** :

1. le navigateur envoie le formulaire en `POST` au frontend ;
2. le frontend appelle l'API ;
3. **en cas de succès**, il répond par une **redirection `303`** vers une page en `GET` ;
4. **en cas d'erreur**, il réaffiche le formulaire avec un message et les valeurs déjà saisies.

```python
# app/routers/items.py (suite)
from datetime import date
from typing import Annotated

from fastapi import Form
from fastapi.responses import RedirectResponse, Response

from app.api_client import ApiError

MESSAGES_RESERVATION = {
    409: "Ce créneau n'est pas disponible.",
    422: "Les dates saisies ne sont pas valides.",
}


@router.post("/items/{item_id}/reservations")
def reserver(
    request: Request,
    item_id: int,
    date_debut: Annotated[date, Form()],
    date_fin: Annotated[date, Form()],
) -> Response:
    try:
        api_client.create_reservation(item_id, date_debut, date_fin)
    except ApiError as exc:
        if exc.status_code not in MESSAGES_RESERVATION:
            raise  # 404, 503... : le gestionnaire global s'en charge
        item = api_client.get_item(item_id)
        return templates.TemplateResponse(
            request,
            "items/detail.html",
            {
                "item": item,
                "erreur": MESSAGES_RESERVATION[exc.status_code],
                "saisie": {"date_debut": date_debut, "date_fin": date_fin},
            },
            status_code=exc.status_code,
        )
    return RedirectResponse(f"/items/{item_id}?reservation=ok", status_code=303)
```

Pourquoi rediriger plutôt que renvoyer directement une page de succès ? Si la réponse au `POST`
est une page, un rafraîchissement du navigateur **renvoie le formulaire** : on réserve deux
fois. Après une redirection `303`, la page affichée est le résultat d'un `GET` ; la rafraîchir ne
fait que la recharger.

Côté template, le message d'erreur et les valeurs saisies se réaffichent simplement :

```html
{# extrait de app/templates/items/detail.html #}
{% if erreur %}
  <p class="alerte alerte-erreur">{{ erreur }}</p>
{% endif %}
{% if request.query_params.get("reservation") == "ok" %}
  <p class="alerte alerte-succes">Votre réservation est enregistrée.</p>
{% endif %}

<form method="post" action="/items/{{ item.id }}/reservations">
  <label for="date_debut">Du</label>
  <input type="date" id="date_debut" name="date_debut"
         value="{{ saisie.date_debut if saisie else '' }}" required>
  <label for="date_fin">Au</label>
  <input type="date" id="date_fin" name="date_fin"
         value="{{ saisie.date_fin if saisie else '' }}" required>
  <button type="submit">Réserver</button>
</form>
```

Quatre remarques :

- `Form()` indique à FastAPI de lire le champ dans le corps du formulaire. Un formulaire HTML
  n'envoie pas du JSON mais ses champs encodés sous la forme
  `date_debut=2026-10-12&date_fin=2026-10-14` (format `application/x-www-form-urlencoded`), et
  FastAPI ne sait pas décoder ce format seul : il s'appuie sur le paquet `python-multipart`. Sans
  lui, l'application refuse de démarrer dès qu'une route utilise `Form()`. Le backend, qui ne
  reçoit que du JSON, n'en a pas besoin. Le typage `date` convertit ensuite la chaîne envoyée par
  le navigateur en vraie date ;
- si un utilisateur contourne le navigateur et envoie une date illisible, FastAPI la rejette
  lui-même ; ce cas ne concerne pas un utilisateur normal, le champ `type="date"` l'en empêche ;
- le frontend renvoie le **même code** que le backend (`409`, `422`) avec la page réaffichée :
  c'est plus honnête pour le navigateur, et plus facile à tester ;
- la création de réservation sera protégée par authentification en séance 5. Pour l'instant,
  l'appel passe sans jeton ; le client d'API est l'endroit où vous l'ajouterez.

**Traduire les erreurs, pas les inventer.** Le frontend ne doit jamais afficher le JSON brut du
backend, ni un code HTTP seul. Il doit traduire, en partant de ce que l'API renvoie réellement :

| Code renvoyé par l'API | Ce que voit l'utilisateur |
|---|---|
| `401` | Retour à la page de connexion (séance 5) |
| `403` | « Vous n'avez pas le droit de faire cette action. » |
| `404` | Page « introuvable » |
| `409` | Message sur le formulaire : « créneau déjà réservé », « email déjà utilisé »… |
| `422` | Message sur le formulaire, champs concernés signalés |
| `5xx` ou backend injoignable | Page « service indisponible », sans détail technique |

### 2.8. Un peu de CSS

CSS décrit l'**apparence** : couleurs, espacements, mise en page. Le design n'est pas évalué dans
ce module ; la **lisibilité**, si. Une feuille de style d'une cinquantaine de lignes suffit
largement. Elle se place dans `app/static/style.css` et elle est chargée par le `<link>` du
template de base.

Une règle CSS associe un **sélecteur** (à quoi elle s'applique) à des **déclarations** (ce qu'elle
change) :

| Sélecteur | Cible | Exemple |
|---|---|---|
| `main` | toutes les balises `<main>` | `main { max-width: 960px; }` |
| `.alerte` | les éléments avec `class="alerte"` | `.alerte { padding: 0.75rem; }` |
| `nav a` | les liens situés dans un `<nav>` | `nav a { text-decoration: none; }` |

Une base raisonnable pour le projet :

```css
/* app/static/style.css */
:root {
  --couleur-principale: #1d4ed8;
  --couleur-texte: #1f2937;
  --couleur-bordure: #d1d5db;
}

body {
  margin: 0;
  font-family: system-ui, sans-serif;
  line-height: 1.5;
  color: var(--couleur-texte);
}

main {
  max-width: 960px;
  margin: 0 auto;      /* centre le contenu */
  padding: 1rem;
}

nav {
  display: flex;       /* aligne les liens sur une ligne */
  gap: 1.5rem;
  padding: 1rem;
  background: var(--couleur-principale);
}

nav a {
  color: white;
  text-decoration: none;
}

table {
  width: 100%;
  border-collapse: collapse;
}

th, td {
  padding: 0.5rem;
  border-bottom: 1px solid var(--couleur-bordure);
  text-align: left;
}

form {
  display: grid;
  gap: 0.5rem;
  max-width: 400px;
}

button {
  padding: 0.5rem 1rem;
  border: none;
  border-radius: 4px;
  background: var(--couleur-principale);
  color: white;
  cursor: pointer;
}

.alerte        { padding: 0.75rem; border-radius: 4px; }
.alerte-erreur { background: #fee2e2; color: #991b1b; }
.alerte-succes { background: #dcfce7; color: #166534; }
```

Trois notions suffisent pour lire et ajuster ce fichier :

- **le modèle de boîte** : chaque élément est une boîte avec un contenu, un `padding` (espace
  intérieur), une `border` et une `margin` (espace extérieur) ;
- **`display: flex`** aligne des éléments sur une ligne (une barre de navigation), **`display:
  grid`** les empile proprement avec un espacement régulier (un formulaire) ;
- **les variables CSS** (`--couleur-principale`) évitent de répéter une couleur à dix endroits.

N'allez pas plus loin pour l'instant : pas de framework CSS, pas de JavaScript. Une page sobre,
lisible et cohérente vaut mieux qu'une page décorée.

### 2.9. Ce que le frontend ne fait pas

Pour clore cette partie, la liste de ce qu'un reviewer doit **ne pas** trouver dans le frontend,
qu'il ait été écrit par vous ou par un agent :

- une connexion à la base, un import de SQLAlchemy ou de `psycopg` ;
- une règle métier (« si l'objet est indisponible, alors… ») : le frontend affiche ce que le
  backend décide ;
- un appel `httpx` en dehors de `api_client.py` ;
- une adresse de backend en dur (`localhost:8000`) au lieu de `API_BASE_URL` ;
- un appel sans `timeout` ;
- un `|safe` sur une donnée utilisateur ;
- une action qui modifie des données déclenchée par un lien (`GET`) ;
- une stack trace ou un JSON d'erreur brut affiché à l'utilisateur.

## 3. Synthèse

- Trois services, une seule direction d'appel : frontend → backend → base. Le frontend présente,
  le backend décide, la base garantit l'intégrité.
- Chaque service reçoit sa configuration par variables d'environnement, et seulement celle dont
  il a besoin : le frontend ne connaît que `API_BASE_URL`.
- Le frontend est une application FastAPI séparée qui rend des templates Jinja2, ne parle au
  backend que par un client d'API unique, suit le schéma Post/Redirect/Get pour les formulaires et
  traduit les erreurs de l'API en messages lisibles.
- HTML décrit la structure, CSS l'apparence ; la validation du navigateur est du confort, c'est
  le backend qui protège les données.

Passez au [TP 1](../tp/tp1-frontend.md) : vous branchez un frontend sur l'API de la séance 1.
Puis revenez au [cours 2](cours2-backend-db.md) pour brancher le backend sur PostgreSQL.
