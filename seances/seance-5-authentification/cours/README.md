# Cours — Séance 5 : authentification et autorisation

- [Support PDF du cours](seance-5-authentification.pdf)

Votre API de la séance 4 a un problème : n'importe qui peut supprimer le matériel de n'importe qui.
Cette séance ferme la porte.

Une authentification défaillante ne se voit pas en testant le cas nominal : la connexion
fonctionne, les routes répondent, et la faille reste invisible jusqu'à ce que quelqu'un l'exploite.
C'est pourquoi cette séance insiste autant sur la relecture et sur les cas d'erreur.

- [Cours — Séance 5 : authentification et autorisation](#cours--séance-5--authentification-et-autorisation)
  - [Objectifs pédagogiques](#objectifs-pédagogiques)
  - [1. Authentification, autorisation, identification](#1-authentification-autorisation-identification)
  - [2. Stocker un mot de passe](#2-stocker-un-mot-de-passe)
    - [2.1. Ce qu'il ne faut jamais faire](#21-ce-quil-ne-faut-jamais-faire)
    - [2.2. Le hachage](#22-le-hachage)
    - [2.3. Le sel](#23-le-sel)
    - [2.4. Le coût](#24-le-coût)
    - [2.5. En pratique](#25-en-pratique)
  - [3. Sessions ou tokens ?](#3-sessions-ou-tokens-)
  - [4. Le JWT](#4-le-jwt)
    - [4.1. Anatomie](#41-anatomie)
    - [4.2. Ce que le JWT garantit — et ne garantit pas](#42-ce-que-le-jwt-garantit--et-ne-garantit-pas)
    - [4.3. Encoder et décoder](#43-encoder-et-décoder)
  - [5. Le flow complet](#5-le-flow-complet)
  - [6. Implémentation dans FastAPI](#6-implémentation-dans-fastapi)
    - [6.1. Inscription](#61-inscription)
    - [6.2. Connexion](#62-connexion)
    - [6.3. La dépendance `get_current_user`](#63-la-dépendance-get_current_user)
    - [6.4. Protéger une route](#64-protéger-une-route)
  - [7. L'autorisation](#7-lautorisation)
  - [8. Côté frontend : se connecter et rester connecté](#8-côté-frontend--se-connecter-et-rester-connecté)
    - [8.1. Le principe](#81-le-principe)
    - [8.2. Le client d'API transmet le jeton](#82-le-client-dapi-transmet-le-jeton)
    - [8.3. Connexion et déconnexion](#83-connexion-et-déconnexion)
    - [8.4. Le jeton expiré ou absent : laisser le backend décider](#84-le-jeton-expiré-ou-absent--laisser-le-backend-décider)
    - [8.5. Le cookie et le CSRF](#85-le-cookie-et-le-csrf)
  - [9. Les pièges](#9-les-pièges)
  - [10. Tester l'authentification et l'autorisation](#10-tester-lauthentification-et-lautorisation)
  - [11. Synthèse](#11-synthèse)

## Objectifs pédagogiques

À la fin de ce cours, vous devez être capables de :

- distinguer authentification et autorisation, et choisir entre `401` et `403` ;
- expliquer pourquoi un mot de passe se hache avec un sel et un coût, et jamais autrement ;
- décrire la structure d'un JWT et ce que sa signature garantit exactement ;
- implémenter inscription, connexion et une dépendance `get_current_user` dans FastAPI ;
- protéger une route et vérifier la propriété d'une ressource ;
- brancher la connexion sur le frontend : cookie, transmission du jeton au backend, retour à la
  page de connexion sur un `401`, protection contre le CSRF ;
- transformer chaque ligne de la matrice d'autorisation en test pytest ;
- identifier les failles classiques d'une authentification.

## 1. Authentification, autorisation, identification

Trois notions qu'on confond en permanence :

| Notion | Question | Exemple |
|---|---|---|
| **Identification** | Qui prétendez-vous être ? | « Je suis alice@esiee.fr » |
| **Authentification** | Pouvez-vous le prouver ? | Le mot de passe correspond |
| **Autorisation** | Avez-vous le droit de faire ça ? | Alice peut supprimer *son* vélo, pas celui de Bob |

En HTTP, cela donne deux codes qu'il vaut la peine de bien distinguer :

- **`401 Unauthorized`** — mal nommé : il veut dire « **non authentifié** ». Aucun token, token
  invalide, token expiré. La réponse doit porter un en-tête `WWW-Authenticate: Bearer`.
- **`403 Forbidden`** — « authentifié, mais pas le droit ». Alice est bien connectée, mais l'item
  appartient à Bob.

Confondre les deux n'est pas un détail cosmétique : le client s'en sert pour décider quoi faire.
Un `401` déclenche une redirection vers la connexion ; un `403` affiche « vous n'avez pas les
droits ». Une API qui renvoie `401` là où il fallait `403` déconnecte un utilisateur parfaitement
connecté ; à l'inverse, un `403` à la place d'un `401` lui affiche « vous n'avez pas les droits »
au lieu de l'inviter à se reconnecter.

> Cas subtil, et bonne pratique de sécurité : sur une ressource privée dont l'existence même est
> une information sensible, on renvoie parfois `404` au lieu de `403`, pour ne pas révéler que la
> ressource existe. Pour GearShare, le catalogue est public : `403` est le bon choix. Mais sachez
> que le compromis existe, et **documentez votre décision**.

## 2. Stocker un mot de passe

### 2.1. Ce qu'il ne faut jamais faire

```python
# ☠️ En clair
user.password = "azerty123"

# ☠️ Chiffré (réversible : si la clé fuit, tous les mots de passe fuient)
user.password = encrypt(password, key)

# ☠️ MD5 ou SHA-1 (cassés, et de toute façon trop rapides)
user.password = hashlib.md5(password.encode()).hexdigest()

# ☠️ SHA-256 nu (non cassé, mais bien trop rapide : des milliards d'essais par seconde sur GPU)
user.password = hashlib.sha256(password.encode()).hexdigest()
```

Le raisonnement à avoir : **partez du principe que votre base de données fuitera un jour.** La
question n'est pas « comment empêcher la fuite » mais « que peut faire l'attaquant avec le
contenu de la table `users` ». La réponse doit être : rien d'exploitable en un temps raisonnable.

### 2.2. Le hachage

Une fonction de hachage est **à sens unique** : de `azerty123` on obtient un condensat, du
condensat on ne peut pas revenir au mot de passe. À la connexion, on ne compare pas les mots de
passe, on compare les condensats.

Ce qui distingue une fonction de hachage **de mot de passe** d'une fonction de hachage générique
(SHA-256), ce sont deux propriétés supplémentaires : le sel et le coût.

### 2.3. Le sel

Un **sel** est une valeur aléatoire, unique par utilisateur, mélangée au mot de passe avant
hachage.

Sans sel, deux utilisateurs avec le même mot de passe ont le même condensat. Un attaquant peut
alors utiliser une **table arc-en-ciel** — un dictionnaire précalculé de millions de condensats —
et casser toute la base d'un coup.

Avec un sel unique par utilisateur, chaque mot de passe doit être attaqué séparément, et aucune
table précalculée ne sert. Le sel n'est pas secret : il est stocké à côté du condensat (les
formats modernes l'incluent dans la chaîne).

### 2.4. Le coût

Une bonne fonction de hachage de mot de passe est **délibérément lente**. Elle applique des
centaines de milliers d'itérations, avec un paramètre de coût réglable.

L'arithmétique : si vérifier un mot de passe prend 100 ms, l'utilisateur ne le remarque pas, mais
un attaquant qui teste un dictionnaire de 10 millions de mots passe de quelques secondes à près de
deux semaines par cœur de calcul. Et le paramètre s'augmente au fil des années, à mesure que le
matériel progresse.

Les algorithmes acceptables aujourd'hui :

| Algorithme | Remarque |
|---|---|
| **Argon2id** | Recommandé par l'OWASP. Résistant aux attaques GPU et ASIC |
| **bcrypt** | Éprouvé, très répandu. ⚠️ tronque silencieusement au-delà de 72 octets |
| **PBKDF2-HMAC-SHA256** | Standard, disponible dans la bibliothèque standard Python |
| **scrypt** | Correct, moins répandu |

Pour votre projet, nous utiliserons **PBKDF2-HMAC-SHA256**, implémenté avec la bibliothèque standard.
Ce n'est pas le meilleur algorithme de la table : Argon2id l'est, parce qu'il consomme aussi de
la mémoire, ce qui rend les attaques sur GPU coûteuses, alors que PBKDF2 ne coûte que du calcul.
Mais PBKDF2 est un standard (NIST, FIPS 140), l'OWASP l'accepte toujours avec **au moins 600 000
itérations** en SHA-256, et surtout il tient en vingt lignes sans aucune dépendance : vous voyez
le sel, le coût et la comparaison dans votre propre code, au lieu de les confier à une boîte
noire. En entreprise, vous utiliserez probablement Argon2id via une bibliothèque — vous saurez
alors exactement ce qu'elle fait pour vous.

### 2.5. En pratique

```python
# app/core/security.py
import hashlib
import hmac
import secrets

ALGORITHM = "pbkdf2_sha256"
ITERATIONS = 600_000
SALT_BYTES = 16


def hash_password(password: str) -> str:
    salt = secrets.token_bytes(SALT_BYTES)
    dk = hashlib.pbkdf2_hmac("sha256", password.encode(), salt, ITERATIONS)
    return f"{ALGORITHM}${ITERATIONS}${salt.hex()}${dk.hex()}"


def verify_password(plain: str, stored: str) -> bool:
    try:
        algorithm, iterations, salt_hex, hash_hex = stored.split("$")
        salt, expected = bytes.fromhex(salt_hex), bytes.fromhex(hash_hex)
        iterations = int(iterations)
    except ValueError:
        return False  # condensat malformé : on refuse la connexion, sans lever d'erreur 500
    if algorithm != ALGORITHM:
        return False
    dk = hashlib.pbkdf2_hmac("sha256", plain.encode(), salt, iterations)
    return hmac.compare_digest(dk, expected)
```

Le condensat stocké en base contient l'algorithme, le coût, le sel et le condensat lui-même :

<!-- schema: format-condensat -->
```text
pbkdf2_sha256$600000$1c4e9a07b2f3d8e6...$8d0f3a91c27e5b44...
└─────┬─────┘ └──┬─┘ └────────┬───────┘ └────────┬────────┘
  algorithme    coût          sel             condensat
```

C'est ce qui rend le format **auto-descriptif** : `verify_password` relit le nombre d'itérations
dans la chaîne stockée, pas dans la constante. Le jour où vous passez `ITERATIONS` à 1 000 000,
les comptes existants continuent de fonctionner ; vous pouvez alors re-hacher leur mot de passe
au prochain login réussi, puisque c'est le seul moment où vous l'avez en clair.

Quatre détails, souvent ratés :

- **`secrets`, pas `random`.** Le module `random` est un générateur pseudo-aléatoire prévisible,
  conçu pour les simulations. Pour un sel, un jeton ou un secret, c'est `secrets` (ou
  `os.urandom`).
- **`hmac.compare_digest`, pas `==`.** Une comparaison classique s'arrête au premier octet
  différent, et sa durée dépend donc du nombre d'octets corrects : c'est une **attaque
  temporelle**. `compare_digest` compare en temps constant. C'est le détail le plus souvent
  oublié.
- **Un condensat malformé ne fait pas planter la connexion.** L'utilisateur de démonstration de la
  séance 4 a `password_hash='x'` : `verify_password` doit renvoyer `False`, pas lever une
  exception qui finit en `500`.
- **Le coût se mesure.** 600 000 itérations, c'est de l'ordre de 100 ms par vérification sur un
  portable récent : imperceptible au login, rédhibitoire pour un attaquant. Inutile de baisser cette
  valeur pour accélérer vos tests : les fixtures `alice` et `bob` n'ont pas besoin de vrai
  hachage, seuls les tests de `/auth/register` et `/auth/login` le paient.

Dernier point, essentiel : **`password_hash` ne sort jamais de l'API.** C'est précisément le
rôle du schéma `UserRead` de la séance 4 — vérifiez qu'il ne contient pas ce champ.

## 3. Sessions ou tokens ?

Deux façons de garder l'utilisateur connecté entre deux requêtes.

**Sessions serveur** : le serveur crée une session en mémoire ou en base, envoie un identifiant de
session dans un cookie. À chaque requête, il retrouve la session.

- ✅ Révocation immédiate (on supprime la session).
- ✅ L'identifiant ne contient aucune information.
- ❌ Le serveur doit stocker un état, partagé entre toutes ses instances.
- ❌ Mal adapté aux clients non-navigateurs.

**Tokens (JWT)** : le serveur signe un jeton contenant l'identité, et ne stocke rien. Le client le
renvoie à chaque requête ; le serveur vérifie la signature.

- ✅ **Sans état** : n'importe quelle instance peut valider le jeton.
- ✅ Fonctionne avec tout type de client (mobile, script, autre service).
- ❌ **Révocation difficile** : un jeton signé reste valide jusqu'à son expiration.
- ❌ Le contenu est lisible par tout le monde (signé ≠ chiffré).

Pour une API REST sans état, **le JWT est le choix standard**, et c'est celui que nous retenons pour
votre backend : il est appelé par votre frontend, mais aussi par Swagger, par vos tests et par
`curl`, qui savent tous joindre un en-tête `Authorization`.

Le navigateur, lui, ne sait pas le faire : une page HTML et ses formulaires ne peuvent pas ajouter
un en-tête à une requête. La seule chose qu'un navigateur renvoie automatiquement à chaque
requête, c'est un **cookie**. Votre application combine donc les deux mécanismes :

- entre le navigateur et le frontend, un **cookie**, que le navigateur renvoie tout seul ;
- entre le frontend et le backend, un **jeton `Bearer`**, que le frontend joint à chaque appel.

Le contenu du cookie, c'est le JWT lui-même. La section 8 détaille ce raccord.

## 4. Le JWT

### 4.1. Anatomie

Un JWT (*JSON Web Token*) est une chaîne en trois parties séparées par des points :

<!-- schema: anatomie-jwt -->
```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsImlhdCI6MTc2ODc0MTYwMCwiZXhwIjoxNzY4NzQ1MjAwfQ.xfTWkc3DZRPbwcWLckYJhx9YC82rfpvsWPrVA78qSO0
└───────────── header ─────────────┘ └───────────────────────── payload ──────────────────────────┘ └─────────────── signature ───────────────┘
```

Le header et le payload sont du JSON encodé en **base64url** ; la signature, des octets bruts
encodés de la même façon.

Le **header** décrit l'algorithme :

```json
{"alg": "HS256", "typ": "JWT"}
```

Le **payload** contient les *claims* :

```json
{
  "sub": "42",
  "iat": 1768741600,
  "exp": 1768745200
}
```

Quelques claims standardisés utiles : `sub` (le sujet, votre identifiant utilisateur), `exp`
(expiration, timestamp Unix), `iat` (émis à), `iss` (émetteur), `jti` (identifiant unique du
jeton).

La **signature** est calculée par le serveur avec sa clé secrète :

```text
HMAC-SHA256(base64url(header) + "." + base64url(payload), SECRET)
```

### 4.2. Ce que le JWT garantit — et ne garantit pas

C'est le point le plus important de la séance, et le plus mal compris.

**Le payload n'est PAS chiffré. Il est encodé.** N'importe qui peut le lire :

```bash
python3 -c "import base64; p='eyJzdWIiOiI0MiIsImlhdCI6MTc2ODc0MTYwMCwiZXhwIjoxNzY4NzQ1MjAwfQ'; print(base64.urlsafe_b64decode(p + '=' * (-len(p) % 4)))"
# b'{"sub":"42","iat":1768741600,"exp":1768745200}'
```

(Un simple `base64 -d` ne suffit pas toujours : le base64url remplace `+` et `/` par `-` et `_`, et
supprime le padding `=` final. Selon les systèmes, `base64 -d` refuse la chaîne ou tronque
silencieusement la fin.)

Allez sur [jwt.io](https://jwt.io), collez un jeton, et regardez : le contenu s'affiche sans
aucune clé.

Conséquence directe : **évitez de placer dans un JWT** un mot de passe, un numéro de carte, un
identifiant de session sensible, ou toute donnée personnelle que vous ne diffuseriez pas.

Ce que la signature garantit, c'est l'**intégrité** : personne ne peut modifier le payload sans
invalider la signature, parce que la recalculer exige le secret. Si un attaquant change `"sub":
"42"` en `"sub": "1"`, la vérification échoue.

Donc :

| Le JWT garantit | Le JWT ne garantit pas |
|---|---|
| Que le contenu n'a pas été modifié | Que le contenu est secret |
| Qu'il a été émis par le détenteur du secret | Qu'il n'a pas été volé |
| Qu'il n'a pas expiré (si vous vérifiez `exp`) | Qu'il n'a pas été révoqué |

### 4.3. Encoder et décoder

```python
# app/core/security.py
from datetime import datetime, timedelta, timezone

import jwt   # PyJWT

from app.core.config import settings


def create_access_token(user_id: int) -> str:
    now = datetime.now(timezone.utc)
    payload = {
        "sub": str(user_id),
        "iat": now,
        "exp": now + timedelta(minutes=settings.jwt_expire_minutes),
    }
    return jwt.encode(payload, settings.jwt_secret, algorithm="HS256")


def decode_access_token(token: str) -> int:
    payload = jwt.decode(token, settings.jwt_secret, algorithms=["HS256"])
    return int(payload["sub"])
```

Trois détails qui comptent :

1. **`algorithms=["HS256"]` au décodage, en liste explicite.** Une bibliothèque qui fait confiance
   au champ `alg` du header laisse un attaquant forger un jeton avec `"alg": "none"` (pas de
   signature) ou basculer d'un algorithme asymétrique vers HMAC. PyJWT 2.x rend ce paramètre
   obligatoire, mais d'anciennes versions et d'autres bibliothèques ne le font pas : la règle à
   retenir est de toujours fixer vous-mêmes la liste des algorithmes acceptés.
2. **`sub` doit être une chaîne.** La spécification l'exige, et les versions récentes de PyJWT
   refusent un entier. D'où le `str(user_id)` / `int(...)`.
3. **`exp` est vérifiée automatiquement** par `jwt.decode`, qui lève `ExpiredSignatureError`. Ne
   réimplémentez pas la comparaison à la main.

Sur la durée de vie : 15 à 60 minutes est raisonnable pour un jeton d'accès. Un jeton valable
30 jours, c'est un mot de passe permanent volable. La solution industrielle est le couple
*access token* court + *refresh token* long et révocable ; pour votre projet, un jeton court
suffit, mais **documentez le compromis dans votre README**.

## 5. Le flow complet

<!-- schema: flux-complet -->
```text
  ┌────────┐                                    ┌────────┐         ┌──────┐
  │ Client │                                    │  API   │         │  DB  │
  └───┬────┘                                    └───┬────┘         └──┬───┘
      │ POST /auth/register {email, password}       │                 │
      │────────────────────────────────────────────►│  hash(password) │
      │                                             │  INSERT user    │
      │                                             │────────────────►│
      │◄──────────────────────── 201 {id, email}    │                 │
      │                                             │                 │
      │ POST /auth/login {email, password}          │                 │
      │────────────────────────────────────────────►│  SELECT user    │
      │                                             │────────────────►│
      │                                             │  verify(hash)   │
      │                                             │  create JWT     │
      │◄──── 200 {access_token, token_type}         │                 │
      │                                             │                 │
      │ DELETE /items/42                            │                 │
      │ Authorization: Bearer <jwt>                 │                 │
      │────────────────────────────────────────────►│ decode JWT      │
      │                                             │ SELECT user     │
      │                                             │ SELECT item     │
      │                                             │────────────────►│
      │◄──── 204   ou 401   ou 403   ou 404         │                 │
```

L'en-tête à chaque requête protégée :

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

## 6. Implémentation dans FastAPI

### 6.1. Inscription

```python
@router.post("/register", response_model=UserRead, status_code=201)
def register(payload: UserCreate, service: Annotated[AuthService, Depends(get_auth_service)]) -> User:
    return service.register(email=payload.email, password=payload.password,
                            display_name=payload.display_name)
```

```python
class AuthService:
    def register(self, *, email: str, password: str, display_name: str) -> User:
        if self.users.get_by_email(email):
            raise EmailDejaUtilise("Un compte existe déjà avec cet email")
        return self.users.add(
            User(email=email, display_name=display_name,
                 password_hash=hash_password(password))
        )
```

`EmailDejaUtilise` est une nouvelle exception métier. Ajoutez-la à côté de celles de la séance 4,
**et** dans la table de correspondance du gestionnaire d'exceptions : sans cela, elle tombe dans
le cas par défaut et devient un `400` au lieu d'un `409`.

```python
# app/core/exceptions.py
class EmailDejaUtilise(DomainError): ...
```

```python
# app/main.py
STATUS_PAR_EXCEPTION: dict[type[DomainError], int] = {
    RessourceIntrouvable: 404,
    ConflitReservation: 409,
    EmailDejaUtilise: 409,
    AccesRefuse: 403,
}
```

Le schéma d'entrée impose une politique de mot de passe minimale :

```python
class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=12, max_length=128)
    display_name: str = Field(min_length=2, max_length=80)
```

`EmailStr` demande le paquet `pydantic[email]`. Sur la longueur minimale : les recommandations
actuelles de l'OWASP privilégient la longueur (12 caractères minimum) plutôt que les règles de
composition (« une majuscule, un chiffre, un caractère spécial »), qui poussent surtout à
`Password1!`. Le `max_length` n'est pas décoratif non plus : une entrée sans borne est une entrée
qu'un attaquant peut gonfler à volonté (taille des requêtes, mémoire, logs). Avec PBKDF2 tel que
l'implémente `hashlib`, un mot de passe long ne coûte pas plus cher à hacher ; mais d'autres
implémentations, et bcrypt avec sa limite de 72 octets, ne se comportent pas ainsi. Borner
explicitement vos entrées vous évite d'avoir à le savoir.

Et le schéma de sortie :

```python
class UserRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    email: EmailStr
    display_name: str
    # pas de password_hash ici
```

### 6.2. Connexion

```python
from fastapi.security import OAuth2PasswordRequestForm


@router.post("/login", response_model=TokenRead)
def login(
    form: Annotated[OAuth2PasswordRequestForm, Depends()],
    service: Annotated[AuthService, Depends(get_auth_service)],
) -> TokenRead:
    user = service.authenticate(email=form.username, password=form.password)
    if user is None:
        raise HTTPException(
            status_code=401,
            detail="Email ou mot de passe incorrect",
            headers={"WWW-Authenticate": "Bearer"},
        )
    return TokenRead(
        access_token=create_access_token(user.id),
        token_type="bearer",
        expires_in=settings.jwt_expire_minutes * 60,
    )
```

```python
class TokenRead(BaseModel):
    access_token: str
    token_type: str
    expires_in: int   # durée de validité, en secondes
```

`expires_in` est un champ standard de la réponse OAuth2 : il dit au client combien de temps le
jeton reste valable, sans qu'il ait à décoder le JWT. Votre frontend s'en servira pour régler la
durée de vie de son cookie (section 8).

`OAuth2PasswordRequestForm` attend un formulaire `application/x-www-form-urlencoded` avec les
champs `username` et `password` — c'est la convention OAuth2, et c'est ce qui fait fonctionner le
bouton « Authorize » de Swagger UI. FastAPI s'appuie sur le paquet `python-multipart` pour lire ce
format : ajoutez-le aux dépendances **du backend** (en séance 4, seul le frontend en avait besoin),
sinon l'application refuse de démarrer. Le champ s'appelle `username` parce que la convention l'impose,
mais il transporte **l'email** de l'utilisateur : c'est l'email qui identifie un compte GearShare
depuis la séance 4. Vous pouvez aussi accepter un JSON avec votre propre schéma ;
dans ce cas vous perdez l'intégration Swagger. Je vous conseille la forme standard.

Le message d'erreur mérite attention : **« Email ou mot de passe incorrect »**, jamais « cet email
n'existe pas ». Sinon vous offrez un oracle qui permet d'énumérer les comptes de votre plateforme.

```python
class AuthService:
    def authenticate(self, *, email: str, password: str) -> User | None:
        user = self.users.get_by_email(email)
        if user is None or not verify_password(password, user.password_hash):
            return None
        return user
```

> Pour être rigoureux, ce code laisse encore fuir une information par le temps de réponse : si
> l'utilisateur n'existe pas, on ne fait pas de hachage, donc la réponse est plus rapide. La parade
> est de hacher un mot de passe factice dans ce cas. Ce n'est pas exigé pour le projet, mais je
> serai content si vous en parlez.

### 6.3. La dépendance `get_current_user`

C'est la pièce centrale, et c'est exactement le mécanisme d'injection de la séance 4 :

```python
# app/api/deps.py
from typing import Annotated

import jwt
from fastapi import Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer

from app.core.security import decode_access_token
from app.models.user import User
from app.repositories.user import UserRepository

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="auth/login")

CREDENTIALS_EXCEPTION: HTTPException = HTTPException(
    status_code=401,
    detail="Identifiants invalides",
    headers={"WWW-Authenticate": "Bearer"},
)


def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    db: DbSession,
) -> User:
    try:
        user_id = decode_access_token(token)
    except (jwt.PyJWTError, KeyError, ValueError):
        raise CREDENTIALS_EXCEPTION

    user = UserRepository(db).get(user_id)
    if user is None:
        raise CREDENTIALS_EXCEPTION
    return user


CurrentUser = Annotated[User, Depends(get_current_user)]
```

La dépendance réutilise `decode_access_token` (section 4.3) : l'algorithme et le secret ne sont
définis qu'à un seul endroit. Elle passe aussi par le repository, comme le reste de votre code
depuis la séance 4 : une dépendance FastAPI n'est pas une excuse pour court-circuiter les couches.

Deux points souvent ratés :

- **On relit l'utilisateur en base.** Le jeton dit « utilisateur 42 » ; encore faut-il que 42
  existe toujours. Sans cette vérification, le compte supprimé hier continue d'utiliser l'API
  jusqu'à l'expiration de son jeton.
- **On renvoie toujours le même message d'erreur**, quelle que soit la cause (jeton absent,
  malformé, expiré, utilisateur disparu). Un message détaillé aide surtout l'attaquant.

### 6.4. Protéger une route

```python
@router.post("", response_model=ItemRead, status_code=201)
def create_item(payload: ItemCreate, current_user: CurrentUser, service: ItemServiceDep) -> Item:
    return service.creer(payload, owner_id=current_user.id)
```

Ajouter `current_user: CurrentUser` à la signature suffit : FastAPI exécute la dépendance avant la
fonction, et la route devient inaccessible sans jeton valide. Bonus : Swagger affiche un cadenas
sur la route.

Notez que `owner_id` vient du **jeton**, pas du corps de la requête. Si vous laissez le client
envoyer `owner_id`, il peut publier du matériel au nom de quelqu'un d'autre. **Toute donnée
d'identité vient du jeton, jamais du payload.** C'est une des erreurs les plus fréquentes.

Pour protéger tout un router d'un coup :

```python
router = APIRouter(prefix="/reservations", dependencies=[Depends(get_current_user)])
```

Et pour une route où l'authentification est optionnelle (afficher le catalogue enrichi si connecté,
sinon la version publique), on écrit une variante `get_current_user_optional` qui renvoie `None` au
lieu de lever.

## 7. L'autorisation

Être authentifié ne suffit pas. Alice ne doit pas pouvoir supprimer le vélo de Bob.

```python
class ItemService:
    def supprimer(self, item_id: int, *, current_user_id: int) -> None:
        item = self.items.get(item_id)
        if item is None:
            raise RessourceIntrouvable(f"Item {item_id} introuvable")
        if item.owner_id != current_user_id:
            raise AccesRefuse("Vous n'êtes pas le propriétaire de ce matériel")
        self.items.delete(item)
```

**La vérification est dans le service, pas dans le router.** C'est une règle métier : « seul le
propriétaire modifie son matériel ». Elle doit s'appliquer quel que soit le point d'entrée.

Grâce au gestionnaire d'exceptions de la séance 4, `AccesRefuse` devient automatiquement un `403`
et `RessourceIntrouvable` un `404`. Vous n'écrivez la correspondance qu'une fois.

Pour GearShare, je vous propose d'écrire explicitement vos règles d'autorisation dans votre
documentation, par exemple :

| Action | Qui a le droit |
|---|---|
| Lister/consulter le matériel | Tout le monde (public) |
| Publier du matériel | Utilisateur authentifié |
| Modifier/supprimer un matériel | Son propriétaire uniquement |
| Réserver un matériel | Utilisateur authentifié, **sauf** le propriétaire lui-même |
| Annuler une réservation | L'emprunteur ou le propriétaire du matériel |
| Consulter l'historique d'un utilisateur | Cet utilisateur uniquement |

Si vos besoins deviennent plus riches, la marche suivante est le **RBAC** (*Role-Based Access
Control*) : un champ `role` sur l'utilisateur, une dépendance `require_role("admin")`. Pour votre
projet, l'ownership suffit en général ; ajoutez des rôles seulement si un besoin réel apparaît.

## 8. Côté frontend : se connecter et rester connecté

Votre backend sait maintenant émettre et vérifier un jeton. Reste à le faire vivre dans
l'application que vos utilisateurs voient réellement : le frontend de la séance 4.

### 8.1. Le principe

<!-- schema: architecture -->

<!-- schema: flux-frontend -->
```text
Navigateur                      frontend (:8080)                  backend (:8000)
    │ POST /connexion                 │                                 │
    │ email, password                 │                                 │
    │ ──────────────────────────────► │  POST /auth/login               │
    │                                 │ ──────────────────────────────► │
    │                                 │ ◄────────────────────────────── │
    │                                 │  200 {access_token, expires_in} │
    │ ◄────────────────────────────── │                                 │
    │  303 + Set-Cookie: access_token │                                 │
    │                                 │                                 │
    │ POST /items/3/reservations      │                                 │
    │ Cookie: access_token=<jwt>      │                                 │
    │ ──────────────────────────────► │  POST /reservations             │
    │                                 │  Authorization: Bearer <jwt>    │
    │                                 │ ──────────────────────────────► │
    │                                 │ ◄────────────────────────────── │
    │ ◄────────────────────────────── │  201, 401, 403, 409…            │
```

Trois règles en découlent :

- **Le frontend ne vérifie jamais le jeton.** Il n'a pas `JWT_SECRET`, et ne doit pas l'avoir
  (séance 4, section 1.2) : le service le plus exposé des trois serait alors capable de forger des
  jetons. Pour lui, le JWT est une chaîne opaque, qu'il range dans un cookie et recopie dans un
  en-tête. **Seul le backend juge de sa validité.**
- **Le frontend ne décide toujours rien.** Il ne sait pas si l'utilisateur a le droit de réserver,
  ni même si son jeton est encore valable : il transmet, et il traduit la réponse du backend.
- **Le navigateur ne voit jamais l'en-tête `Authorization`.** Il ne connaît que son cookie ; c'est
  le frontend qui fait la conversion.

### 8.2. Le client d'API transmet le jeton

La séance 4 l'avait annoncé : `ApiClient` est le seul endroit à modifier pour ajouter le jeton.
La tentation est de le ranger dans l'instance, une fois pour toutes :

```python
# ☠️ à éviter
def set_token(self, token: str) -> None:
    self._token = token
```

Rappelez-vous la dernière ligne de `api_client.py` : `api_client` est une **instance unique,
partagée par toutes les requêtes de tous les utilisateurs**. Si Alice se connecte et que son jeton
est rangé dans l'instance, la requête suivante — celle de Bob, ou d'un visiteur anonyme — part avec
le jeton d'Alice. Ce bug ne se voit pas quand vous testez seul sur votre machine ; il apparaît dès
que deux personnes utilisent l'application en même temps.

La règle : **le jeton appartient à la requête du navigateur, pas au client d'API.** Il est passé en
paramètre à chaque appel qui en a besoin :

```python
# app/api_client.py (extraits modifiés)
class ApiClient:
    def _send(self, method: str, url: str, *, token: str | None = None, **kwargs: Any) -> Any:
        headers = {"Authorization": f"Bearer {token}"} if token else {}
        try:
            response = self._http.request(method, url, headers=headers, **kwargs)
        except httpx.RequestError as exc:
            raise ApiError(503, "Le service est momentanément indisponible.") from exc
        ...  # traitement des erreurs inchangé

    def login(self, email: str, password: str) -> dict[str, Any]:
        # formulaire OAuth2 : data= (encodé comme un formulaire), pas json=
        return self._send("POST", "/auth/login", data={"username": email, "password": password})

    def register(self, email: str, password: str, display_name: str) -> dict[str, Any]:
        return self._send(
            "POST",
            "/auth/register",
            json={"email": email, "password": password, "display_name": display_name},
        )

    def create_reservation(
        self, token: str | None, item_id: int, date_debut: date, date_fin: date
    ) -> dict[str, Any]:
        return self._send(
            "POST",
            "/reservations",
            token=token,
            json={"item_id": item_id, "date_debut": date_debut.isoformat(), "date_fin": date_fin.isoformat()},
        )
```

Notez que `create_reservation` accepte un `token` à `None` : un visiteur non connecté qui soumet
le formulaire déclenche un appel sans jeton, et c'est le backend qui répond `401`. Le frontend n'a
pas à le deviner (section 8.4).

Côté routes, une petite dépendance lit le cookie :

```python
# app/auth.py
import os
from typing import Annotated

from fastapi import Depends, Request

COOKIE_JETON = "access_token"
COOKIE_SECURE = os.environ.get("COOKIE_SECURE", "true").lower() == "true"


def get_token(request: Request) -> str | None:
    return request.cookies.get(COOKIE_JETON)


Token = Annotated[str | None, Depends(get_token)]
```

```python
# app/routers/items.py (extrait modifié)
@router.post("/items/{item_id}/reservations")
def reserver(
    request: Request,
    item_id: int,
    token: Token,
    date_debut: Annotated[date, Form()],
    date_fin: Annotated[date, Form()],
) -> Response:
    try:
        api_client.create_reservation(token, item_id, date_debut, date_fin)
    ...  # suite inchangée
```

`COOKIE_SECURE` est une nouvelle variable d'environnement du frontend, non secrète : elle vaut
`false` en local, où vous travaillez en HTTP, et `true` dès que l'application est servie en HTTPS.
La valeur par défaut est `true` : un oubli de configuration doit produire un cookie trop protégé,
pas l'inverse.

### 8.3. Connexion et déconnexion

La connexion est un formulaire comme les autres, et suit le schéma Post/Redirect/Get de la
séance 4 :

```python
# app/routers/auth.py
from typing import Annotated

from fastapi import APIRouter, Form, Request
from fastapi.responses import HTMLResponse, RedirectResponse, Response

from app.api_client import ApiError, api_client
from app.auth import COOKIE_JETON, COOKIE_SECURE
from app.templating import templates

router = APIRouter()


def chemin_local(cible: str | None) -> str:
    """N'autorise qu'une redirection vers une page de ce site."""
    if cible and cible.startswith("/") and not cible.startswith(("//", "/\\")):
        return cible
    return "/items"


@router.get("/connexion", response_class=HTMLResponse)
def connexion_page(request: Request, suivant: str | None = None) -> HTMLResponse:
    return templates.TemplateResponse(request, "auth/connexion.html", {"suivant": suivant})


@router.post("/connexion")
def connexion(
    request: Request,
    email: Annotated[str, Form()],
    password: Annotated[str, Form()],
    suivant: Annotated[str | None, Form()] = None,
) -> Response:
    try:
        jeton = api_client.login(email, password)
    except ApiError as exc:
        if exc.status_code != 401:
            raise
        return templates.TemplateResponse(
            request,
            "auth/connexion.html",
            {"erreur": "Email ou mot de passe incorrect.", "saisie": {"email": email}, "suivant": suivant},
            status_code=401,
        )
    response = RedirectResponse(chemin_local(suivant), status_code=303)
    response.set_cookie(
        COOKIE_JETON,
        jeton["access_token"],
        max_age=jeton["expires_in"],
        httponly=True,
        secure=COOKIE_SECURE,
        samesite="lax",
    )
    return response


@router.post("/deconnexion")
def deconnexion() -> Response:
    response = RedirectResponse("/items", status_code=303)
    response.delete_cookie(COOKIE_JETON)
    return response
```

```html
{# app/templates/auth/connexion.html (extrait) #}
<form method="post" action="/connexion">
  <input type="hidden" name="suivant" value="{{ suivant or '' }}">
  <label for="email">Email</label>
  <input type="email" id="email" name="email" autocomplete="username"
         value="{{ saisie.email if saisie else '' }}" required>
  <label for="password">Mot de passe</label>
  <input type="password" id="password" name="password" autocomplete="current-password" required>
  <button type="submit">Se connecter</button>
</form>
```

Cinq remarques :

- **Le message d'erreur reste celui du backend** : « Email ou mot de passe incorrect », sans
  distinguer les deux cas. Ne réintroduisez pas côté frontend l'énumération de comptes que le
  backend a fermée.
- **On réaffiche l'email saisi, jamais le mot de passe.** Un mot de passe ne repasse jamais du
  serveur vers le navigateur, même celui que l'utilisateur vient de taper.
- **`max_age=expires_in`** : le cookie disparaît du navigateur en même temps que le jeton expire
  côté backend. Un cookie qui survivrait au jeton ne servirait qu'à provoquer des `401`.
- **La déconnexion est un `POST`**, déclenché par un bouton de formulaire, jamais par un lien. C'est
  la règle de la séance 4 (« aucune action qui modifie quelque chose déclenchée par un `GET` ») ; la
  section 8.5 montre qu'elle est aussi une protection.
- **La déconnexion supprime le cookie, pas le jeton.** Un JWT ne se révoque pas (section 3) : s'il
  a été copié avant la déconnexion, il reste valable jusqu'à son `exp`. C'est l'une des raisons de
  garder une durée de vie courte.

`chemin_local` mérite qu'on s'y arrête. Le paramètre `suivant` permet de revenir, après la
connexion, sur la page que l'utilisateur voulait voir. Mais il vient de l'URL, donc de n'importe
qui : sans vérification, `https://gearshare.fr/connexion?suivant=https://gearshare-login.fr`
renvoie votre utilisateur, fraîchement connecté et en confiance, sur un site qui imite le vôtre.
C'est une **redirection ouverte**, un classique du phishing. `chemin_local` n'accepte qu'un chemin
de ce site : il commence par `/`, mais pas par `//` ni `/\`, que les navigateurs interprètent
comme l'adresse d'un autre site.

L'inscription suit exactement le schéma de la réservation de la séance 4 : `409` → « Un compte
existe déjà avec cet email », `422` → message sur le formulaire, succès → redirection vers
`/connexion?inscription=ok`.

Enfin, le gabarit commun affiche le bon lien selon que le cookie est présent ou non :

```html
{# extrait de app/templates/base.html #}
{% if request.cookies.get("access_token") %}
  <form method="post" action="/deconnexion">
    <button type="submit">Se déconnecter</button>
  </form>
{% else %}
  <a href="/connexion">Se connecter</a>
{% endif %}
```

C'est de l'affichage, pas de la sécurité : un cookie présent ne prouve pas que le jeton est
valide. Si le jeton a expiré, la barre affiche « Se déconnecter » jusqu'au prochain appel au
backend, qui répondra `401`.

### 8.4. Le jeton expiré ou absent : laisser le backend décider

Le frontend ne sait pas si un jeton est valide ; le backend, si. Le seul endroit où traiter
l'absence ou l'expiration du jeton est donc la **réponse `401` du backend**, et le gestionnaire
d'exceptions global de la séance 4 est déjà là pour ça :

```python
# app/main.py (gestionnaire complété)
from urllib.parse import urlencode

from fastapi.responses import RedirectResponse, Response

from app.auth import COOKIE_JETON


@app.exception_handler(ApiError)
def api_error_page(request: Request, exc: ApiError) -> Response:
    if exc.status_code == 401:
        cible = "/connexion"
        if request.method == "GET":
            cible += "?" + urlencode({"suivant": request.url.path})
        response = RedirectResponse(cible, status_code=303)
        response.delete_cookie(COOKIE_JETON)
        return response
    ...  # pages d'erreur de la séance 4, inchangées
```

Un seul mécanisme couvre ainsi tous les cas : pas de cookie, cookie trafiqué, jeton expiré,
compte supprimé. Dans chacun, le backend répond `401`, et l'utilisateur se retrouve sur la page
de connexion avec un cookie nettoyé. Pour une page consultée en `GET`, il y revient ensuite
directement ; pour un formulaire soumis, il devra le soumettre à nouveau.

Le `403`, lui, ne déconnecte personne : l'utilisateur est bien authentifié, il n'a simplement pas
le droit. Il garde la page d'erreur « Vous n'avez pas le droit de faire cette action » de la
séance 4.

### 8.5. Le cookie et le CSRF

Un cookie a une propriété qui fait sa commodité et son danger : **le navigateur l'envoie tout seul**
à chaque requête vers votre site — y compris quand cette requête est déclenchée par un autre site.

C'est le principe du **CSRF** (*Cross-Site Request Forgery*). Alice est connectée à GearShare. Elle
visite un site piégé, qui contient un formulaire invisible soumis automatiquement :

```html
<form method="post" action="https://gearshare.fr/items/3/reservations">
  <input type="hidden" name="date_debut" value="2026-12-01">
  <input type="hidden" name="date_fin" value="2026-12-31">
</form>
<script>document.forms[0].submit()</script>
```

Si le navigateur joint le cookie d'Alice, votre frontend reçoit une requête parfaitement valide,
avec un jeton parfaitement valide : la réservation est faite au nom d'Alice, sans qu'elle l'ait
voulue. Ni le backend ni le JWT n'y peuvent rien — le jeton n'a pas été volé, il a été *utilisé*
à l'insu de sa propriétaire.

Pour votre projet, la défense retenue tient en deux règles :

1. **`samesite="lax"` sur le cookie.** Le navigateur n'envoie alors pas le cookie sur un `POST`
   venu d'un autre site : le formulaire piégé arrive sans cookie, le backend répond `401`.
2. **Aucune modification par `GET`.** `SameSite=Lax` laisse passer le cookie sur une navigation
   `GET` ordinaire (un lien cliqué depuis un autre site), sans quoi l'utilisateur arriverait
   déconnecté sur votre site à chaque lien entrant. Une route `GET` qui modifie des données — une
   déconnexion par lien, une annulation par lien — reste donc exploitable. D'où la règle de la
   séance 4, qui devient ici une règle de sécurité.

Les autres attributs du cookie, que vous avez vus dans `set_cookie` :

| Attribut | Ce qu'il empêche |
|---|---|
| `HttpOnly` | La lecture du cookie par un script. Votre frontend n'a pas de JavaScript, mais une faille XSS (un filtre Jinja `safe` mal placé sur une donnée utilisateur, séance 4) permettrait à un attaquant d'en injecter un : le jeton resterait hors de sa portée. |
| `Secure` | L'envoi du cookie en HTTP non chiffré, où n'importe qui sur le réseau pourrait le lire. |
| `SameSite=Lax` | L'envoi du cookie sur les requêtes `POST` venues d'un autre site (CSRF). |
| `Max-Age` | La survie du cookie au-delà de la validité du jeton. |

`SameSite=Lax` a des limites, qu'il est utile de connaître : pour le navigateur, deux sous-domaines
d'un même domaine (`app.gearshare.fr` et `blog.gearshare.fr`) sont le même « site », et de très
anciens navigateurs ignorent l'attribut. La défense complète, recommandée par l'OWASP, ajoute un
**jeton CSRF** : une valeur aléatoire placée dans chaque formulaire et vérifiée par le serveur, qu'un
site tiers ne peut pas deviner. Ce n'est pas attendu pour votre projet ; si vous l'ajoutez, pensez à
documenter votre choix.

## 9. Les pièges

Ceux que je vois chaque année :

| Piège | Conséquence | Parade |
|---|---|---|
| Secret JWT en dur dans le code | Le dépôt public donne la clé : n'importe qui forge un jeton admin | `settings.jwt_secret`, `.env` non commité |
| Secret faible (`"secret"`, `"changeme"`) | Cassable en quelques secondes | 32+ octets aléatoires : `openssl rand -hex 32` |
| Pas d'`exp` sur le jeton | Jeton éternel | `exp` obligatoire, 15–60 min |
| `algorithms` non spécifié au décodage | Attaque `alg: none` ou confusion d'algorithme | Liste explicite `["HS256"]` |
| Données sensibles dans le payload | Lisibles par tous | Ne mettez que `sub`, `exp`, `iat` |
| `owner_id` pris dans le corps de la requête | Usurpation d'identité | Toujours depuis `current_user` |
| Autorisation oubliée sur `PUT`/`DELETE` | N'importe qui modifie les données d'autrui | Vérification d'ownership dans le service |
| Message de login trop précis | Énumération des comptes | « Email ou mot de passe incorrect » |
| `==` pour comparer des condensats | Attaque temporelle | `hmac.compare_digest` |
| Mot de passe sans `max_length` | Entrée que l'attaquant peut gonfler à volonté | `Field(max_length=128)` |
| Mot de passe loggé | Fuite dans les logs, souvent conservés longtemps | Ne loguez jamais le corps d'une requête d'auth |
| `JWT_SECRET` donné au frontend | Le service le plus exposé peut forger des jetons | Le frontend ne vérifie jamais le jeton |
| Jeton rangé dans l'instance partagée `ApiClient` | Chaque utilisateur hérite de l'identité du précédent | Jeton passé en paramètre à chaque appel |
| Cookie sans `HttpOnly` ni `SameSite` | Jeton lisible par un script injecté ; formulaires forgés depuis un autre site (CSRF) | `httponly=True`, `samesite="lax"` |
| Action qui modifie déclenchée par un `GET` | CSRF possible malgré `SameSite=Lax` | Toujours un formulaire en `POST`, déconnexion comprise |
| Paramètre `suivant` non vérifié | Redirection ouverte vers un site de phishing | N'accepter qu'un chemin local |
| Pas de HTTPS | Le jeton circule en clair | Obligatoire en production, cookie `Secure` |

Deux d'entre eux méritent une insistance particulière, parce qu'ils sont invisibles au test :

**Le secret en dur.** Votre application fonctionnera parfaitement. Elle sera juste totalement
compromise dès que le dépôt sera lu. Faites `git grep -i "secret\|password\|token" -- '*.py'` sur
votre projet avant chaque rendu.

**L'autorisation oubliée.** Tester `DELETE /items/42` connecté en tant que propriétaire passe. Le
test qui manque, c'est `DELETE /items/42` connecté en tant que *quelqu'un d'autre* — et il doit
renvoyer `403`. C'est exactement pour ça que la séance 2 insiste autant sur les cas d'erreur.

## 10. Tester l'authentification et l'autorisation

Votre suite de tests dispose depuis la séance 4 d'une fixture `client` branchée sur la base de
test, et d'une fixture `alice`. Il vous faut un second utilisateur, et des clients qui portent une
identité :

```python
# tests/conftest.py
@pytest.fixture
def bob(db: Session) -> User:
    user = User(email="bob@esiee.fr", display_name="Bob", password_hash="x")
    db.add(user)
    db.flush()
    return user


def _bearer(user: User) -> dict[str, str]:
    return {"Authorization": f"Bearer {create_access_token(user.id)}"}


@pytest.fixture
def client_alice(client: TestClient, alice: User) -> TestClient:
    return TestClient(app, headers=_bearer(alice))


@pytest.fixture
def client_bob(client: TestClient, bob: User) -> TestClient:
    return TestClient(app, headers=_bearer(bob))
```

Trois choix à comprendre :

- **On fabrique le jeton directement** avec `create_access_token` (section 4.3) plutôt que de
  passer par `POST /auth/login` : le login a ses propres tests, et les cinquante autres tests n'ont
  pas à le rejouer. `get_current_user` relit l'utilisateur en base ; comme `alice` et `bob` sont
  insérés dans la transaction du test, ils sont trouvés.
- **`client_alice` dépend de `client`** sans s'en servir : c'est la fixture `client` qui installe
  l'override de `get_db`. Sans elle, le client d'Alice taperait dans la base de développement.
- **Chaque identité a sa propre instance de `TestClient`.** Si `client_alice` se contentait
  d'ajouter un en-tête à `client`, les trois fixtures partageraient le même objet : un test qui
  demande `client` et `client_alice` n'aurait plus aucun client anonyme, et un test qui demande
  `client_alice` et `client_bob` serait authentifié deux fois en tant que Bob. Ce genre de test
  passe au vert tout en ne testant pas ce que son nom annonce.

L'item de test est la fixture `item_velo` de la séance 4, qui appartient à Alice :

```python
def test_creer_item_authentifie(client_alice: TestClient) -> None:
    response = client_alice.post("/items", json={"titre": "Vélo", "tarif_jour": 8.5})

    assert response.status_code == 201
    assert response.json()["titre"] == "Vélo"


def test_anonyme_ne_peut_pas_creer_item(client: TestClient) -> None:
    response = client.post("/items", json={"titre": "Vélo", "tarif_jour": 8.5})

    assert response.status_code == 401
    assert response.headers["WWW-Authenticate"] == "Bearer"


def test_bob_ne_peut_pas_supprimer_item_alice(client_bob: TestClient, item_velo: Item) -> None:
    response = client_bob.delete(f"/items/{item_velo.id}")

    assert response.status_code == 403
```

Avec `client`, `client_alice` et `client_bob`, la matrice d'autorisation de la section 7 se lit
ligne par ligne : **chaque ligne, trois tests** — anonyme, propriétaire, autre utilisateur. Ce sont
les tests au meilleur retour sur investissement de toute votre suite : ils attrapent les
régressions les plus graves (une faille d'accès) et ce sont les plus faciles à casser sans s'en
rendre compte — retirer `current_user` d'une signature de route lors d'un refactoring ne fait
échouer aucun test nominal.

La règle « propriétaire uniquement » vit dans le service (section 7) ; testez-la aussi là, sans
HTTP, sur le modèle de la fixture `service` de la séance 4 :

```python
@pytest.fixture
def item_service(db: Session) -> ItemService:
    return ItemService(ItemRepository(db))


def test_supprimer_l_item_d_un_autre_leve_acces_refuse(
    item_service: ItemService, item_velo: Item, bob: User
) -> None:
    with pytest.raises(AccesRefuse):
        item_service.supprimer(item_velo.id, current_user_id=bob.id)
```

Et l'identité tirée du jeton, jamais du corps (section 9) : un test envoie `owner_id` de Bob dans
le corps avec le jeton d'Alice, et vérifie que l'item créé appartient à Alice.

Enfin, testez le login lui-même, et surtout ses échecs : mauvais mot de passe et email inexistant
doivent renvoyer le **même** `401` avec le **même** message. Un test paramétré sur ces deux cas,
qui compare les deux corps de réponse, protège votre défense contre l'énumération de comptes.

## 11. Synthèse

- Authentification = prouver qui vous êtes (`401`). Autorisation = avoir le droit (`403`).
- Un mot de passe se stocke **haché, salé, avec un coût** : pour vous, PBKDF2-HMAC-SHA256 avec au
  moins 600 000 itérations. Jamais en clair, jamais en SHA nu.
- Un JWT est **signé, pas chiffré** : intégrité garantie, confidentialité nulle. Rien de sensible
  dans le payload.
- `exp` obligatoire, `algorithms` explicite au décodage, secret en variable d'environnement.
- `get_current_user` est une dépendance FastAPI qui décode le jeton **et relit l'utilisateur en
  base**.
- L'identité vient toujours du jeton, jamais du corps de la requête.
- L'ownership se vérifie dans le **service**, et se traduit en `403`.
- Côté frontend, le JWT vit dans un cookie `HttpOnly`, `SameSite=Lax`, `Secure` en production ; le
  frontend le recopie en `Bearer` à chaque appel, **en paramètre**, jamais dans l'instance
  partagée du client d'API. Il ne le vérifie jamais : un `401` du backend renvoie à la page de
  connexion.
- Aucune modification par `GET`, déconnexion comprise : c'est ce qui rend `SameSite=Lax` efficace
  contre le CSRF.
- Le cas nominal qui passe ne prouve rien : ce sont les tests d'erreur (anonyme, autre
  utilisateur, jeton invalide) qui prouvent que la porte est fermée.

Passez au [TP](../tp/README.md).
