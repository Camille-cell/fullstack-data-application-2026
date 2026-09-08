# TP — Séance 0 : Docker

## Objectifs du TP

Ce TP vous fait manipuler en pratique tout le vocabulaire et les commandes vus dans le
[cours de cette séance 0](../cours/README.md). À la fin du TP, vous devez savoir :

- construire une image à partir d'un `Dockerfile` et lancer un conteneur ;
- exposer un port et vérifier qu'un service répond ;
- expliquer pourquoi deux conteneurs isolés ne peuvent pas communiquer par défaut ;
- créer un réseau Docker et y attacher des conteneurs ;
- écrire un `docker-compose.yml` qui orchestre plusieurs services et utilise le DNS interne de Compose.

Aucun rendu formel n'est attendu pour cette séance 0 : elle sert de mise à niveau avant la séance
1. Assurez-vous simplement d'être à l'aise avec les manipulations ci-dessous avant le début du
module.

## Prérequis

- Docker Desktop installé et lancé (`docker run hello-world` doit fonctionner — voir
  [INTRO.md](../../../OUTILS.md), section 5, si ce n'est pas déjà fait).
- Un terminal et un éditeur de code.

Le support de ce TP est le dossier [`demo-app/`](./demo-app), qui contient deux mini-applications
FastAPI indépendantes :

```text
demo-app/
├── api/     # renvoie {"message": "Hello from api!"}
└── front/   # affiche une page HTML qui va chercher ce message auprès de l'api
```

## Partie 1 — Construire et lancer un premier conteneur (25 min)

1. Dans `demo-app/api/`, regardez le `Dockerfile` : de quelle image de base part-il ? Que fait
   chaque instruction ?
2. Construisez l'image :

   ```bash
   cd demo-app/api
   docker build -t tp0-api .
   ```

3. Lancez un conteneur en exposant le port 8000, en arrière-plan :

   ```bash
   docker run -d --name tp0-api -p 8000:8000 tp0-api
   ```

4. Vérifiez que l'API répond, avec votre navigateur, puis avec `curl` :

   ```bash
   curl http://localhost:8000
   ```

5. Consultez les logs du conteneur, listez les conteneurs et images en cours, puis arrêtez et
   supprimez le conteneur :

   ```bash
   docker logs tp0-api
   docker ps
   docker images
   docker rm -f tp0-api
   ```

Questions à noter :

- Que se passe-t-il si vous relancez `docker run --name tp0-api ...` sans avoir supprimé le
  conteneur précédent ?
- À quoi sert exactement `-p 8000:8000` ? Que se passe-t-il si vous changez le port côté hôte
  (`-p 9000:8000`) ?

## Partie 2 — Construire sa propre image (20 min)

1. Faites la même chose avec `demo-app/front/` : construisez l'image `tp0-front`, puis lancez un
   conteneur en exposant le port 8080 :

   ```bash
   cd ../front
   docker build -t tp0-front .
   docker run -d --name tp0-front -p 8080:8080 tp0-front
   ```

2. Ouvrez `http://localhost:8080` dans votre navigateur. Vous devriez voir une page qui affiche
   **"impossible de contacter l'api"**.

Regardez le code de `demo-app/front/app/main.py` : la variable `API_URL` vaut par défaut
`http://localhost:8000`. Pourquoi cette adresse ne fonctionne-t-elle pas depuis l'intérieur du
conteneur front, alors qu'elle fonctionnait très bien pour votre navigateur à l'étape 1 ? (Indice :
`localhost` désigne toujours "cette machine-ci" — à l'intérieur du conteneur front, ce n'est pas
la même machine que votre conteneur api.)

Ne supprimez pas encore les deux conteneurs, vous allez vous en servir à la partie suivante.

## Partie 3 — Réseau Docker manuel (30 min)

Par défaut, chaque conteneur lancé avec `docker run` est isolé sur son propre réseau. Pour que
deux conteneurs se voient, il faut les attacher explicitement au même réseau.

1. Créez un réseau dédié :

   ```bash
   docker network create tp0-net
   ```

2. Attachez-y votre conteneur `tp0-api` :

   ```bash
   docker network connect tp0-net tp0-api
   ```

3. Recréez le conteneur `tp0-front` sur ce réseau, en lui passant l'URL correcte de l'api via une
   variable d'environnement (`-e`) :

   ```bash
   docker rm -f tp0-front
   docker run -d --name tp0-front -p 8080:8080 --network tp0-net -e API_URL=http://tp0-api:8000 tp0-front
   ```

4. Rafraîchissez `http://localhost:8080` : vous devriez maintenant voir **"Hello from api!"**.

Notez ce qui a changé dans l'URL : `http://tp0-api:8000` utilise le **nom du conteneur** comme
nom d'hôte, résolu par le DNS interne de Docker, à la place d'une adresse IP.

5. Ouvrez un shell dans le conteneur front et vérifiez que vous arrivez à joindre l'api depuis
   l'intérieur :

   ```bash
   docker exec -it tp0-front bash
   curl http://tp0-api:8000
   exit
   ```

6. Nettoyez :

   ```bash
   docker rm -f tp0-api tp0-front
   docker network rm tp0-net
   ```

## Partie 4 — Orchestration avec Docker Compose (40 min)

Refaire manuellement un réseau, deux `docker run` et une variable d'environnement à chaque
lancement n'est pas tenable dès que le nombre de services augmente. C'est exactement ce que
Docker Compose automatise.

1. À la racine de `demo-app/`, créez un fichier `docker-compose.yml` qui :
   - construit et lance le service `api` à partir de `./api`, en exposant le port 8000 ;
   - construit et lance le service `front` à partir de `./front`, en exposant le port 8080, avec
     la variable d'environnement `API_URL` pointant vers le service `api` (rappel : dans Compose,
     un service joint un autre par son **nom de service**, pas par un nom de conteneur ni une IP).

   Aidez-vous de l'exemple de syntaxe donné dans le [cours, section 7](../cours/README.md#7-docker-compose-et-le-dns-interne).

2. Lancez l'ensemble :

   ```bash
   docker compose up --build
   ```

3. Vérifiez que `http://localhost:8080` affiche à nouveau **"Hello from api!"** — sans avoir créé
   de réseau ni de variable d'environnement à la main : Compose s'en est chargé.

4. Modifiez le texte du template `demo-app/front/app/templates/index.html`, relancez
   `docker compose up --build` et vérifiez que le changement apparaît.

5. Ouvrez un shell dans un des deux services via Compose (remarquez la différence avec
   `docker exec` de la partie 3) :

   ```bash
   docker compose exec front bash
   ```

6. Arrêtez et nettoyez :

   ```bash
   docker compose down
   ```

Questions à noter :

- Qu'est-ce qui a changé entre la commande manuelle de la partie 3 (`-e API_URL=http://tp0-api:8000`)
  et la valeur que vous avez mise dans `docker-compose.yml` ? Pourquoi le nom change-t-il ?
- Que fait `docker compose down` par rapport à `docker rm -f` sur chaque conteneur un par un ?

## Partie 5 — Pour aller plus loin (bonus, 20 min)

Ces manipulations ne sont pas obligatoires mais vous serviront dès la séance 1.

1. **Volumes** : ajoutez un volume nommé à votre service `api` dans `docker-compose.yml`
   (`volumes: - mon_volume:/data`, à déclarer aussi au niveau racine du fichier). Relancez,
   inspectez-le avec `docker volume inspect`, puis supprimez-le avec `docker volume rm`.
2. **Multi-stage build** : réécrivez le `Dockerfile` de `demo-app/api` en deux étapes (`builder`
   puis image finale allégée), sur le modèle donné dans le
   [cours, section 8.2](../cours/README.md#82-multi-stage-builds). Vérifiez que l'image obtenue
   fonctionne toujours (`docker build`, puis `docker run`) et comparez sa taille avec
   `docker images` par rapport à la version d'origine.
3. **Registry** : si vous avez un compte Dockerhub, taguez votre image
   (`docker tag tp0-api votre_pseudo/tp0-api:v1`) et poussez-la (`docker push`). Supprimez-la
   localement (`docker rmi`), puis re-téléchargez-la (`docker pull`) pour vérifier qu'elle
   fonctionne toujours.

## Ce que vous devez retenir avant la séance 1

- La différence entre une image et un conteneur.
- Pourquoi deux conteneurs isolés ne communiquent pas par défaut, et ce qu'un réseau Docker change.
- Comment Docker Compose remplace une série de `docker run`/`docker network` manuels, et comment
  ses services se contactent par leur nom.

Ce sont exactement ces mécanismes que vous retrouverez à la séance 1, quand vous conteneuriserez
votre projet GearShare (backend, frontend, base de données) avec un unique `docker-compose.yml`.
