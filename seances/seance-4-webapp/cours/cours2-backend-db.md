# Cours 2 — Séance 4 : le backend en couches et la base de données

Après le [cours 1](cours1-frontend.md), votre application a un frontend et un backend, mais le
backend oublie tout à chaque redémarrage, et la base de la séance 3 n'a pas d'interface. Ce cours
les relie, et c'est lui qui structure réellement votre projet : l'**architecture en couches** du
backend, **SQLAlchemy**, **Alembic**, la **configuration par environnement** et un **Docker
Compose à trois services**.

C'est la partie la plus dense du module. L'authentification de la séance 5 s'appuie directement
sur ce qu'on met en place ici ; et la suite de tests créée en séance 2 change de socle : une base
PostgreSQL de test remplace les dictionnaires, et les règles métier se testent dans les services.

- [Cours 2 — Séance 4 : le backend en couches et la base de données](#cours-2--séance-4--le-backend-en-couches-et-la-base-de-données)
  - [Objectifs pédagogiques](#objectifs-pédagogiques)
  - [1. L'architecture du backend : les couches](#1-larchitecture-du-backend--les-couches)
  - [2. SQLAlchemy : l'ORM](#2-sqlalchemy--lorm)
    - [2.1. Engine et session](#21-engine-et-session)
    - [2.2. Les modèles](#22-les-modèles)
    - [2.3. Les relations](#23-les-relations)
    - [2.4. Interroger](#24-interroger)
    - [2.5. Le problème N+1](#25-le-problème-n1)
  - [3. L'injection de dépendances FastAPI](#3-linjection-de-dépendances-fastapi)
  - [4. Modèles SQLAlchemy et schémas Pydantic](#4-modèles-sqlalchemy-et-schémas-pydantic)
  - [5. Les couches en pratique](#5-les-couches-en-pratique)
    - [5.1. Repository](#51-repository)
    - [5.2. Service](#52-service)
    - [5.3. Router](#53-router)
    - [5.4. Traduire les exceptions métier en HTTP](#54-traduire-les-exceptions-métier-en-http)
  - [6. Les migrations avec Alembic](#6-les-migrations-avec-alembic)
  - [7. Configuration : le 12-factor](#7-configuration--le-12-factor)
  - [8. Docker Compose à trois services](#8-docker-compose-à-trois-services)
  - [9. Tester l'application en couches](#9-tester-lapplication-en-couches)
    - [9.1. Une base de test isolée](#91-une-base-de-test-isolée)
    - [9.2. Injecter la session de test](#92-injecter-la-session-de-test)
    - [9.3. Tester les services sans HTTP](#93-tester-les-services-sans-http)
    - [9.4. Mocker au bon niveau](#94-mocker-au-bon-niveau)
  - [10. Relire une couche complète](#10-relire-une-couche-complète)
  - [11. Synthèse](#11-synthèse)

## Objectifs pédagogiques

À la fin de ce cours, vous devez être capables de :

- expliquer le rôle de chaque couche du backend (router, service, repository, model, schema) et
  savoir où mettre un bout de code donné ;
- configurer SQLAlchemy (engine, session) et écrire des modèles avec leurs relations ;
- injecter une session de base de données dans une route FastAPI proprement ;
- distinguer modèle SQLAlchemy et schéma Pydantic, et convertir de l'un à l'autre ;
- créer, relire et appliquer une migration Alembic ;
- externaliser toute la configuration en variables d'environnement ;
- orchestrer backend, frontend et base dans un `docker-compose.yml` ;
- brancher la suite pytest sur une base PostgreSQL de test isolée et tester un service sans HTTP ;
- relire une couche d'accès aux données et y repérer les défauts classiques (N+1, injection SQL,
  `commit()` dispersés).

## 1. L'architecture du backend : les couches

Le [cours 1](cours1-frontend.md) a posé l'architecture globale : trois services, une seule direction
d'appel, un backend seul responsable des règles métier et de la base. Zoomons maintenant à
l'intérieur du backend. Comme la section 1 du cours 1, cette partie ne contient volontairement
aucun code : c'est le plan, le code vient ensuite.

Imaginez une route « créer une réservation » écrite d'un seul
bloc : elle reçoit la requête HTTP, va chercher l'objet en base, vérifie sa disponibilité,
construit une requête pour détecter les réservations qui se chevauchent, insère la nouvelle
réservation, valide la transaction et choisit le code HTTP de la réponse. Une quinzaine de lignes,
tout au même endroit.

Ça marche. Et pourtant c'est mauvais, pour quatre raisons :

1. **La règle métier est prisonnière du HTTP.** Vous voulez créer une réservation depuis un script
   d'import ou une tâche planifiée ? Vous devez dupliquer les 15 lignes.
2. **Le SQL est prisonnier de la route.** La même requête de conflit sera réécrite ailleurs.
3. **C'est intestable unitairement.** Pour tester la règle « pas de chevauchement », il faut monter
   un client HTTP et une base.
4. **Ça ne passe pas l'échelle.** Un fichier `routers/` de 900 lignes où tout est mélangé devient
   illisible : plus personne ne sait où ajouter une fonctionnalité, ni où chercher un bug.

L'architecture en couches répond à ces quatre points en donnant à chaque morceau de code **une
seule raison d'exister** :

```text
   HTTP (requête du frontend ou de tout autre client)
     │
     ▼
┌──────────────┐  routers/      Traduire HTTP ⇄ métier. Rien d'autre.
│   Router     │  + schemas/    Valide l'entrée, appelle un service, renvoie un schéma.
└──────┬───────┘
       ▼
┌──────────────┐  services/     La logique métier. Les règles, les décisions, les
│   Service    │                exceptions métier. Ne connaît ni HTTP ni SQL.
└──────┬───────┘
       ▼
┌──────────────┐  repositories/ L'accès aux données. Les requêtes. Rien de métier.
│  Repository  │
└──────┬───────┘
       ▼
┌──────────────┐  models/       Les tables, en Python.
│    Model     │
└──────┬───────┘
       ▼
  PostgreSQL
```

Cette architecture en couches n'est pas la seule possible. Vous croiserez en entreprise d'autres
architectures logicielles classiques, comme l'**architecture hexagonale** (ports et adaptateurs),
la **Clean Architecture** ou le **Domain-Driven Design** (DDD), qui poussent plus loin
l'isolation du métier vis-à-vis de l'infrastructure. Elles se justifient sur des applications
volumineuses et durables, maintenues par plusieurs équipes. Pour les besoins de notre application,
nous nous limiterons à cette architecture en couches simple : elle suffit largement, et c'est elle
que vous mettrez en place dans votre projet.

**Le router** est la porte d'entrée HTTP. Il connaît les URL, les méthodes, les codes de statut,
les en-têtes. Son travail tient en trois temps : recevoir une requête déjà validée par un schéma,
appeler le bon service, renvoyer la réponse dans le bon format avec le bon code. Il ne prend
aucune décision métier et ne construit aucune requête SQL. Un router bien écrit est presque
ennuyeux à lire : c'est bon signe.

```python
@router.post("", response_model=ReservationRead, status_code=status.HTTP_201_CREATED)
def ma_route(
    payload: ReservationCreate,
    service: Annotated[Service, Depends(...)],        # séance 5
) -> ReservationRead:
    
    try:
        response = service.creer(
            item_id=payload.item_id,
            debut=payload.date_debut,
            fin=payload.date_fin,
        )
    except ResourceDoesNotExist as exc:
        raise HTTPException(status_code=status.HTTP_404_NOT_FOUND, detail=exc.message)
    except ResourceAlreadyBooked as exc:
        raise HTTPException(status_code=status.HTTP_409_CONFLICT, detail=exc.message)
    
    return response
```


Lisez-le comme une table de traduction : le `try` appelle le service, et chaque `except`
transforme une exception métier précise en code HTTP explicite, avec un message lisible pour le
client. Notez ce qui n'y figure pas : aucune requête SQL, aucune règle métier, et pas de
`except Exception` fourre-tout.

**Les schémas** (Pydantic) décrivent le contrat de l'API : ce qui a le droit d'entrer, ce qui sort.
Ils rejettent automatiquement une date mal formée ou un champ manquant, avant même que votre code
ne s'exécute, et ils filtrent la sortie pour qu'aucun champ interne (un mot de passe hashé, par
exemple) ne quitte jamais le backend. Ce ne sont pas des tables : un même modèle de base peut avoir
plusieurs schémas (création, modification, lecture).

**Le service** est le cœur de l'application. C'est là que vivent les règles qui font de votre
application *GearShare* et pas une autre : « un objet indisponible ne se réserve pas », « deux
réservations actives ne se chevauchent pas », « seul le propriétaire modifie son objet ». Quand
une règle est violée, le service lève une **exception métier** (« objet introuvable », « créneau
déjà réservé »), sans savoir qu'elle deviendra un `404` ou un `409` : c'est le travail du router,
ou d'un gestionnaire d'exceptions global, de faire cette traduction. Parce qu'il ignore HTTP, un
service peut être appelé depuis une route, un script ou un test.

**Le repository** est le seul endroit qui sait *comment* les données sont stockées. Il répond à
des questions précises (« trouve l'objet 42 », « liste les réservations actives qui chevauchent
telle période ») et sait ajouter ou supprimer des lignes. Il ne décide de rien : il ne sait pas
si un chevauchement est un problème, il se contente de le trouver. Si demain une requête devient
lente, c'est le seul fichier à modifier.

**Le modèle** décrit une table en Python : ses colonnes, ses types, ses contraintes, ses relations
avec les autres tables. C'est la traduction directe du schéma SQL que vous avez conçu en séance 3.

**Qui valide la transaction ?** Une action métier peut toucher plusieurs tables (créer une
réservation et marquer un objet comme emprunté, par exemple) : soit tout est enregistré, soit
rien. Cette frontière transactionnelle se décide **une fois pour toute l'application** : soit
chaque requête HTTP forme une transaction (validée par la dépendance qui fournit la session de
base de données), soit chaque service valide explicitement la sienne. On compare les deux en
section 3. Ce qui est exclu, c'est un repository qui valide la transaction de son côté : il ne
sait pas si l'action métier est terminée.

**La règle de dépendance.** Chaque couche ne connaît que la couche du dessous, et ne sait rien de
celle du dessus. Le service ignore qu'il est appelé par HTTP ; le repository ignore pourquoi on
lui demande une donnée ; le modèle ignore tout du reste. Les informations remontent sous forme de
valeurs de retour ou d'exceptions, jamais par un appel vers le haut.

Reprenons cette réservation, vue de l'intérieur du backend : le router reçoit la requête et
le schéma vérifie que les dates sont bien des dates ; le router appelle le service de
réservation ; le service demande l'objet au repository des objets, vérifie sa disponibilité,
demande au repository des réservations s'il existe un chevauchement, puis lui demande d'ajouter
la nouvelle réservation ; la transaction est validée ; le router renvoie la réservation créée,
filtrée par le schéma de lecture, avec un code `201`. Si le service a levé « créneau déjà
réservé », le router (ou le gestionnaire global) le transforme en `409`.

La règle est simple et vous pouvez l'appliquer mécaniquement :

| Si le code… | il va dans… |
|---|---|
| manipule un code HTTP, un header, une réponse | `routers/` |
| exprime une règle métier (« un item indisponible ne peut pas être réservé ») | `services/` |
| construit une requête, ajoute ou supprime une ligne | `repositories/` |
| décrit une table et ses colonnes | `models/` |
| décrit ce qui entre et sort de l'API | `schemas/` |

## 2. SQLAlchemy : l'ORM

Un **ORM** (*Object-Relational Mapper*) fait correspondre des classes Python à des tables SQL.
Vous manipulez des objets, il génère le SQL.

Ce qu'il vous apporte : moins de code répétitif, une sécurité automatique contre l'injection SQL
(une attaque où une saisie utilisateur, collée telle quelle dans une requête, est exécutée comme du
SQL : `' OR 1=1 --` dans un champ de recherche, par exemple ; avec un ORM, les valeurs sont
toujours paramétrées), et un modèle de données unique en Python.

Ce qu'il vous coûte : une couche d'abstraction de plus, du SQL parfois inefficace généré dans votre
dos, et une vraie courbe d'apprentissage. **C'est pourquoi la séance 3 était en SQL pur** : vous
devez pouvoir lire ce que l'ORM produit.

### 2.1. Engine et session

Il nous faut donc un moyen de brancher le code python à la base de données PostgreSQL.
SQLAlchemy le fait en deux étapes. Attention, le code python ne sait pas (et ne dois pas savoir) quelle base est utilisée : il se contente d'une URL de connexion, qui sera fournie par variable d'environnement.

```python
# app/db/session.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from app.core.config import settings

engine = create_engine(
    settings.database_url,
    pool_pre_ping=True,   # teste la connexion avant de la réutiliser
    echo=settings.sql_echo,  # affiche le SQL généré : très utile en développement
)

SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)
```

- **L'engine** gère le pool de connexions vers PostgreSQL. Il est créé **une seule fois** au
  démarrage de l'application.
- **La session** est votre unité de travail : elle suit les objets modifiés et écrit tout au
  `commit()`. Elle est créée **par requête HTTP** et fermée à la fin.

`pool_pre_ping=True` évite une classe d'erreurs très pénible : une connexion du pool coupée
côté serveur (redémarrage de la base, timeout réseau) provoquerait sinon un plantage à la première
requête suivante.

Mettez `echo=True` en développement au moins une fois : voir le SQL réellement exécuté est le
meilleur moyen de comprendre ce que fait votre ORM — et de repérer les requêtes absurdes.

### 2.2. Les modèles

```python
# app/models/base.py
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
```

```python
# app/models/item.py
from datetime import datetime
from decimal import Decimal

from sqlalchemy import BigInteger, Boolean, CheckConstraint, DateTime, ForeignKey, Numeric, String, Text, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.models.base import Base


class Item(Base):
    __tablename__ = "items"
    __table_args__ = (
        CheckConstraint("tarif_jour > 0", name="tarif_jour_positif"),
    )

    id: Mapped[int] = mapped_column(BigInteger, primary_key=True)
    owner_id: Mapped[int] = mapped_column(
        BigInteger, ForeignKey("users.id", ondelete="CASCADE"), nullable=False, index=True
    )
    titre: Mapped[str] = mapped_column(String(120), nullable=False)
    description: Mapped[str | None] = mapped_column(Text)
    tarif_jour: Mapped[Decimal] = mapped_column(Numeric(8, 2), nullable=False)
    disponible: Mapped[bool] = mapped_column(Boolean, nullable=False, default=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), nullable=False, server_default=func.now()
    )

    owner: Mapped["User"] = relationship(back_populates="items")
    reservations: Mapped[list["Reservation"]] = relationship(
        back_populates="item", cascade="all, delete-orphan"
    )
```

Chaque ligne correspond au DDL de la séance 3 — retrouvez la correspondance, elle est directe.
Notez en particulier :

- `index=True` sur la clé étrangère : on n'oublie pas la leçon de la séance 3 ;
- `Numeric(8, 2)` pour le montant, jamais `Float` ;
- `DateTime(timezone=True)` pour `TIMESTAMPTZ` ;
- `server_default=func.now()` : la valeur par défaut est calculée **par PostgreSQL**, pas par
  Python. C'est ce que vous voulez pour un horodatage de création.

**`default` vs `server_default`** : `default` est appliqué par SQLAlchemy au moment de l'insertion
depuis Python ; `server_default` devient un `DEFAULT` dans le DDL et s'applique aussi aux
insertions faites en SQL direct (votre script de seed, par exemple). Pour `created_at`, préférez
`server_default`.

### 2.3. Les relations

`relationship()` n'est **pas** une colonne : c'est une navigation Python entre objets, construite
par-dessus la clé étrangère.

```python
item = repo.get(1)
item.owner.display_name       # navigue vers l'utilisateur
item.reservations             # la liste des réservations de cet item
user.items                    # dans l'autre sens, grâce à back_populates
```

`back_populates` déclare les deux extrémités du lien pour que SQLAlchemy les tienne synchronisées
en mémoire.

Attention à la double cascade : `ondelete="CASCADE"` (dans `ForeignKey`) est la cascade **côté
base**, appliquée par PostgreSQL ; `cascade="all, delete-orphan"` (dans `relationship`) est la
cascade **côté Python**, appliquée par la session. Elles peuvent diverger. En cas de doute,
faites confiance à la base — c'est elle qui garantit, comme on l'a vu en séance 3.

### 2.4. Interroger

SQLAlchemy 2.0 utilise la syntaxe `select()` :

```python
from sqlalchemy import select
from sqlalchemy.orm import Session

from app.models.item import Item


def list_items(db: Session, *, skip: int = 0, limit: int = 20, q: str | None = None) -> list[Item]:
    stmt = select(Item)
    if q:
        stmt = stmt.where(Item.titre.ilike(f"%{q}%"))
    stmt = stmt.order_by(Item.created_at.desc()).offset(skip).limit(limit)
    return list(db.scalars(stmt))


def get_item(db: Session, item_id: int) -> Item | None:
    return db.get(Item, item_id)     # raccourci pour une recherche par clé primaire
```

Écriture :

```python
def create_item(db: Session, *, owner_id: int, titre: str, tarif_jour: Decimal) -> Item:
    item = Item(owner_id=owner_id, titre=titre, tarif_jour=tarif_jour)
    db.add(item)
    db.flush()     # envoie l'INSERT, l'id est attribué, mais rien n'est validé
    db.refresh(item)
    return item
```

`flush()` envoie le SQL sans terminer la transaction ; `commit()` valide définitivement.
**Le `commit()` n'appartient pas au repository** : c'est la couche appelante (le service, ou une
dépendance FastAPI) qui décide de la frontière transactionnelle, parce que c'est elle qui sait si
plusieurs écritures forment une seule opération métier. Un repository qui fait un `commit()` à
chaque méthode vous interdit toute transaction multi-tables.

> Vous ne verrez jamais de f-string dans une requête SQLAlchemy correcte. Écrire
> `db.execute(text(f"SELECT * FROM items WHERE titre = '{q}'"))` est une **injection SQL**. Si vous
> croisez ce genre de code, en review ou ailleurs, refusez-le.

### 2.5. Le problème N+1

Le piège classique de tout ORM :

```python
items: list[Item] = list(db.scalars(select(Item).limit(100)))
for item in items:
    print(item.owner.display_name)   # ← une requête SQL PAR item
```

Une requête pour la liste, puis 100 requêtes pour les propriétaires : 101 requêtes au lieu de 1.
Avec `echo=True`, vous le voyez immédiatement dans les logs.

La correction : charger la relation en même temps.

```python
from sqlalchemy import Select
from sqlalchemy.orm import selectinload

stmt: Select[tuple[Item]] = select(Item).options(selectinload(Item.owner)).limit(100)
```

C'est un défaut extrêmement fréquent, parce que le code naïf est plus
lisible et fonctionne parfaitement… sur trois lignes de données de test. **Vérifiez vos logs SQL
sur une liste paginée** : c'est un des points que je regarderai dans votre rendu.

## 3. L'injection de dépendances FastAPI

Chaque requête HTTP doit obtenir une session, et la refermer même en cas d'erreur.

```python
# app/api/deps.py
from collections.abc import Generator

from sqlalchemy.orm import Session

from app.db.session import SessionLocal


def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

```python
from typing import Annotated

from fastapi import Depends

DbSession = Annotated[Session, Depends(get_db)]


@router.get("/items", response_model=list[ItemRead])
def list_items(db: DbSession, skip: int = 0, limit: int = 20) -> list[Item]:
    ...
```

FastAPI appelle `get_db()`, injecte la session, et exécute le `finally` après la réponse. L'alias
`DbSession` évite de répéter `Depends(get_db)` dans chaque signature — adoptez-le, c'est
l'idiome moderne.

Vous pouvez aller plus loin et faire de la dépendance la frontière transactionnelle :

```python
def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
        db.commit()
    except Exception:
        db.rollback()
        raise
    finally:
        db.close()
```

Chaque requête HTTP devient alors une transaction : si quoi que ce soit lève, tout est annulé.
C'est un choix d'architecture défendable et simple. L'alternative — `commit()` explicite dans le
service — est plus verbeuse mais plus lisible. **Choisissez-en un, et tenez-vous-y dans tout le
projet** ; le pire est de mélanger les deux.

Ce mécanisme de dépendances vous resservira en séance 5 : `get_current_user` sera exactement de la
même forme.

## 4. Modèles SQLAlchemy et schémas Pydantic

Ce sont **deux choses différentes**, et c'est la confusion la plus fréquente :

| | Modèle SQLAlchemy (`models/`) | Schéma Pydantic (`schemas/`) |
|---|---|---|
| Décrit | une table | un contrat d'API |
| Sert à | lire/écrire en base | valider l'entrée, filtrer la sortie |
| Contient | toutes les colonnes, `password_hash` compris | uniquement ce qui est public |
| Change quand | le schéma de base change | le contrat public change |

Ne renvoyez **jamais** un modèle SQLAlchemy directement sans `response_model`. Pour convertir :

```python
# app/schemas/item.py
from decimal import Decimal

from pydantic import BaseModel, ConfigDict


class ItemRead(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    titre: str
    description: str | None
    tarif_jour: Decimal
    disponible: bool
```

`from_attributes=True` autorise Pydantic à lire un objet par ses attributs (et non par des clés de
dictionnaire). Avec un `response_model=ItemRead` sur la route, FastAPI fait la conversion tout
seul — vous renvoyez l'objet SQLAlchemy, le client reçoit exactement les champs de `ItemRead`.

C'est ce mécanisme qui vous protège de la fuite de `password_hash`. **Il n'y a pas de champ secret
dans un schéma `Read`.**

## 5. Les couches en pratique

Déroulons la création de réservation, correctement.

### 5.1. Repository

```python
# app/repositories/reservation.py
from datetime import date

from sqlalchemy import select
from sqlalchemy.orm import Session

from app.models.reservation import Reservation


class ReservationRepository:
    def __init__(self, db: Session) -> None:
        self.db = db

    def get(self, reservation_id: int) -> Reservation | None:
        return self.db.get(Reservation, reservation_id)

    def find_chevauchements(self, item_id: int, debut: date, fin: date) -> list[Reservation]:
        stmt = select(Reservation).where(
            Reservation.item_id == item_id,
            Reservation.statut == "active",
            Reservation.date_debut < fin,
            Reservation.date_fin > debut,
        )
        return list(self.db.scalars(stmt))

    def add(self, reservation: Reservation) -> Reservation:
        self.db.add(reservation)
        self.db.flush()
        self.db.refresh(reservation)
        return reservation
```

Aucune règle métier ici : le repository sait *comment chercher*, pas *quoi décider*.

### 5.2. Service

```python
# app/services/reservation.py
from datetime import date

from app.core.exceptions import ConflitReservation, RessourceIntrouvable
from app.models.reservation import Reservation
from app.repositories.item import ItemRepository
from app.repositories.reservation import ReservationRepository


class ReservationService:
    def __init__(self, reservations: ReservationRepository, items: ItemRepository) -> None:
        self.reservations = reservations
        self.items = items

    def creer(self, *, item_id: int, borrower_id: int, debut: date, fin: date) -> Reservation:
        item = self.items.get(item_id)
        if item is None:
            raise RessourceIntrouvable(f"Item {item_id} introuvable")
        if item.owner_id == borrower_id:
            raise ConflitReservation("Vous ne pouvez pas réserver votre propre matériel")
        if not item.disponible:
            raise ConflitReservation("Ce matériel n'est pas disponible")
        if self.reservations.find_chevauchements(item_id, debut, fin):
            raise ConflitReservation("Ce créneau est déjà réservé")

        return self.reservations.add(
            Reservation(item_id=item_id, borrower_id=borrower_id,
                        date_debut=debut, date_fin=fin, statut="active")
        )
```

Ce service ne sait pas qu'il existe un protocole HTTP. Il lève des exceptions **métier**. Vous
pouvez le tester sans client HTTP (section 9), et le réutiliser depuis un script.

### 5.3. Router

```python
# app/routers/reservations.py
@router.post("", response_model=ReservationRead, status_code=201)
def create_reservation(
    payload: ReservationCreate,
    service: Annotated[ReservationService, Depends(get_reservation_service)],
    current_user: CurrentUser,          # séance 5
) -> Reservation:
    return service.creer(
        item_id=payload.item_id,
        borrower_id=current_user.id,
        debut=payload.date_debut,
        fin=payload.date_fin,
    )
```

Six lignes. Le router traduit, point.

### 5.4. Traduire les exceptions métier en HTTP

Un seul endroit fait la correspondance :

```python
# app/core/exceptions.py
class DomainError(Exception):
    def __init__(self, message: str) -> None:
        self.message = message


class RessourceIntrouvable(DomainError): ...
class ConflitReservation(DomainError): ...
class AccesRefuse(DomainError): ...
```

```python
# app/main.py
from fastapi import Request
from fastapi.responses import JSONResponse

STATUS_PAR_EXCEPTION: dict[type[DomainError], int] = {
    RessourceIntrouvable: 404,
    ConflitReservation: 409,
    AccesRefuse: 403,
}


@app.exception_handler(DomainError)
def handle_domain_error(request: Request, exc: DomainError) -> JSONResponse:
    status: int = STATUS_PAR_EXCEPTION.get(type(exc), 400)
    return JSONResponse(status_code=status, content={"detail": exc.message})
```

Résultat : vos codes HTTP sont cohérents dans toute l'API par construction, et non par discipline
répétée à chaque route.

## 6. Les migrations avec Alembic

En séance 3, vous modifiiez le schéma en rejouant un script d'init sur une base vide. Impossible en
production : il y a des données à conserver.

Une **migration** est un script versionné décrivant le passage d'une version du schéma à la
suivante. Elles sont commitées, appliquées dans l'ordre, et réversibles.

```bash
alembic init -t async alembic     # ou sans -t async pour du synchrone
```

Configurez `alembic/env.py` pour qu'il connaisse vos modèles et lise l'URL depuis l'environnement :

```python
from app.core.config import settings
from app.models.base import Base
from app.models import item, reservation, user  # noqa: F401  ← indispensable

config.set_main_option("sqlalchemy.url", settings.database_url)
target_metadata = Base.metadata
```

⚠️ **L'import des modules de modèles est indispensable.** Alembic compare `Base.metadata` au
schéma réel ; une table dont le module n'est jamais importé est invisible pour lui — et
`autogenerate` proposera de la *supprimer*. C'est la panne n°1 des débutants sur Alembic.

Générer et appliquer :

```bash
alembic revision --autogenerate -m "creation items et reservations"
alembic upgrade head        # applique jusqu'à la dernière
alembic downgrade -1        # revient d'un cran
alembic current             # où en est la base ?
alembic history             # l'historique
```

**`--autogenerate` produit un brouillon, pas une migration.** Vous devez le relire à chaque fois.
Ce qu'il détecte mal ou pas du tout :

| Alembic détecte | Alembic ne détecte pas |
|---|---|
| Tables ajoutées/supprimées | Renommage (il voit une suppression + un ajout → **perte de données**) |
| Colonnes ajoutées/supprimées | Changements de type risqués (`VARCHAR` → `INTEGER`) |
| Changements de nullabilité | La plupart des `CHECK` |
| Index et `UNIQUE` (souvent) | Les données à migrer |

Le cas qui fait perdre des données en production, et que vous devez reconnaître :

```python
# Ce que autogenerate écrit pour un renommage de colonne
op.add_column("items", sa.Column("prix_jour", sa.Numeric(8, 2)))
op.drop_column("items", "tarif_jour")

# Ce qu'il fallait écrire
op.alter_column("items", "tarif_jour", new_column_name="prix_jour")
```

Autre cas : ajouter une colonne `NOT NULL` à une table qui contient déjà des lignes échoue. Il faut
la version en trois temps — ajouter nullable, remplir, puis passer en `NOT NULL` :

```python
def upgrade() -> None:
    op.add_column("items", sa.Column("categorie", sa.String(50), nullable=True))
    op.execute("UPDATE items SET categorie = 'divers' WHERE categorie IS NULL")
    op.alter_column("items", "categorie", nullable=False)
```

Enfin, écrivez toujours le `downgrade()`. Une migration qu'on ne peut pas annuler est une migration
qu'on n'ose pas appliquer.

**Règles pour votre projet** (elles seront évaluées) :

1. Toute modification de modèle donne lieu à une migration, commitée avec le code.
2. La migration générée est relue et corrigée avant commit.
3. `alembic upgrade head` puis `alembic downgrade -1` puis `alembic upgrade head` doit fonctionner.
4. On ne modifie **jamais** une migration déjà appliquée sur une base partagée : on en ajoute une
   nouvelle.
5. `Base.metadata.create_all()` ne sert qu'aux tests. Jamais en production, jamais en parallèle
   d'Alembic — les deux mécanismes finiraient par diverger.

### 6.1. À quoi ressemble le dossier `alembic/`

Après `alembic init` et quelques migrations, voici ce que vous devez trouver dans votre backend :

```text
backend/
├── alembic.ini                  # configuration d'Alembic (l'URL y est écrasée par env.py)
├── alembic/
│   ├── env.py                   # branche Alembic sur vos modèles et sur settings.database_url
│   ├── script.py.mako           # le gabarit utilisé pour générer chaque nouvelle migration
│   └── versions/                # les migrations, une par fichier, toutes commitées
│       ├── 1a2b3c4d5e6f_creation_items_et_reservations.py
│       ├── 7c8d9e0f1a2b_ajout_categorie_items.py
│       └── 3f9a2c1b7d4e_categories_en_table.py
└── app/
    └── ...
```

Chaque fichier de `versions/` porte un identifiant (`revision`) et pointe vers la migration
précédente (`down_revision`). C'est cette chaîne, et non l'ordre alphabétique des fichiers, qui
fixe l'ordre d'application. Alembic mémorise en base, dans la table `alembic_version`, la dernière
révision appliquée.

### 6.2. Écrire une migration à la main : la migration de données

Certaines migrations ne peuvent pas être générées, parce qu'elles ne changent pas seulement la
forme des tables mais **déplacent des données existantes**. C'est le cas typique de la *data
migration*. Vous partez alors d'un fichier vide :

```bash
alembic revision -m "categories en table"     # sans --autogenerate : upgrade/downgrade vides
```

Exemple : la colonne texte `items.categorie` (ajoutée plus haut) contient des valeurs en vrac, et
vous voulez la remplacer par une vraie table `categories` référencée par une clé étrangère, **sans
perdre la catégorie d'aucun objet existant**.

```python
# alembic/versions/3f9a2c1b7d4e_categories_en_table.py
"""categories en table

Revision ID: 3f9a2c1b7d4e
Revises: 7c8d9e0f1a2b
"""
import sqlalchemy as sa
from alembic import op

revision = "3f9a2c1b7d4e"
down_revision = "7c8d9e0f1a2b"
branch_labels = None
depends_on = None


def upgrade() -> None:
    # 1. Nouvelle structure
    op.create_table(
        "categories",
        sa.Column("id", sa.Integer, primary_key=True),
        sa.Column("nom", sa.String(50), nullable=False, unique=True),
    )
    op.add_column("items", sa.Column("categorie_id", sa.Integer, nullable=True))
    op.create_foreign_key(
        "fk_items_categorie_id", "items", "categories", ["categorie_id"], ["id"]
    )

    # 2. Déplacement des données. SQL brut justifié : une migration doit rester figée dans le
    #    temps, elle ne doit pas importer les modèles de app/ qui, eux, continueront d'évoluer.
    op.execute("INSERT INTO categories (nom) SELECT DISTINCT categorie FROM items")
    op.execute(
        "UPDATE items SET categorie_id = c.id FROM categories c WHERE c.nom = items.categorie"
    )

    # 3. Verrouillage, puis suppression de l'ancienne colonne
    op.alter_column("items", "categorie_id", nullable=False)
    op.drop_column("items", "categorie")


def downgrade() -> None:
    # Chemin inverse : on recrée la colonne texte et on la remplit AVANT de supprimer la table
    op.add_column("items", sa.Column("categorie", sa.String(50), nullable=True))
    op.execute(
        "UPDATE items SET categorie = c.nom FROM categories c WHERE c.id = items.categorie_id"
    )
    op.alter_column("items", "categorie", nullable=False)

    op.drop_constraint("fk_items_categorie_id", "items", type_="foreignkey")
    op.drop_column("items", "categorie_id")
    op.drop_table("categories")
```

Trois réflexes à retenir de cet exemple :

- **L'ordre compte** : créer la nouvelle structure, copier les données, puis seulement supprimer
  l'ancienne. Inversez deux étapes et les données sont perdues.
- **Une migration n'importe jamais vos modèles** (`from app.models import ...`). Dans six mois, le
  modèle `Item` aura changé et cette vieille migration ne s'exécutera plus. On écrit le SQL, ou on
  décrit localement les tables avec `sa.table(...)`.
- **Le `downgrade()` restaure aussi les données**, pas seulement le schéma. Testez-le sur une base
  qui contient des lignes : sur une base vide, toutes les migrations de données « marchent ».

## 7. Configuration : le 12-factor

Le principe (issu de la méthodologie *[12-factor app](https://12factor.net/)*) : **la configuration vit dans
l'environnement**, pas dans le code. La même image Docker doit fonctionner en local, en test et en
production, avec des variables différentes.

```python
# app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    database_url: str
    jwt_secret: str                 # séance 5
    jwt_expire_minutes: int = 60
    sql_echo: bool = False


settings = Settings()
```

Deux bénéfices immédiats : l'application **refuse de démarrer** si une variable obligatoire manque
(mieux qu'un `None` qui explose trois heures plus tard), et les valeurs sont typées et validées.

À côté, un `.env.example` **commité** documente ce qu'il faut fournir :

```bash
# Service db (lues aussi par Compose pour la substitution ${...})
POSTGRES_USER=gearshare
POSTGRES_PASSWORD=gearshare
POSTGRES_DB=gearshare

# Service backend
DATABASE_URL=postgresql+psycopg://gearshare:gearshare@db:5432/gearshare
JWT_SECRET=changez-moi-en-production
JWT_EXPIRE_MINUTES=60
SQL_ECHO=false
```

et un `.env` **jamais commité** (ajoutez-le au `.gitignore`) contient les vraies valeurs.

Notez l'hôte `db` dans l'URL : c'est le **nom du service Compose**, résolu par le DNS interne
(séance 0). Depuis votre machine, ce serait `localhost` ; depuis le conteneur backend, c'est `db`.

## 8. Docker Compose à trois services

```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 10

  backend:
    build: ./backend
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "8000:8000"
    volumes:
      - ./backend/app:/code/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  frontend:
    build: ./frontend
    environment:
      API_BASE_URL: http://backend:8000
    depends_on:
      - backend
    ports:
      - "8080:8080"
    volumes:
      - ./frontend/app:/code/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8080 --reload

volumes:
  pgdata:
```

Points à retenir :

- **trois services** : c'est le minimum imposé pour le projet ([cours 1, section 1.1](cours1-frontend.md#11-vue-densemble--trois-services-une-seule-direction)) ;
- le frontend est lui aussi une application FastAPI, servie par `uvicorn` sur un autre port
  (`8080`) ; il a son propre `Dockerfile` dans `frontend/` ;
- le frontend ne reçoit **pas** `env_file: .env` : il n'a besoin que de `API_BASE_URL`, et
  surtout pas des secrets du backend ([cours 1, section 1.2](cours1-frontend.md#12-configurer-chaque-service--les-variables-denvironnement)) ;
- le frontend joint le backend par `http://backend:8000`, **pas** par `localhost:8000` — dans un
  conteneur, `localhost` désigne le conteneur lui-même ;
- seuls les ports que vous voulez atteindre depuis votre machine sont publiés ; le service `db`
  n'a pas besoin de `ports:` si vous ne vous y connectez qu'en `docker compose exec` ;
- `condition: service_healthy` évite le backend qui démarre avant la base.

Reste à appliquer les migrations au démarrage, avant que l'API ne serve sa première requête. Deux
approches sont acceptables.

**Option 1 — un `entrypoint.sh` dans l'image du backend.** Le conteneur applique les migrations,
puis se remplace par `uvicorn` :

```bash
#!/bin/sh
set -e
alembic upgrade head
exec uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Simple, un seul service ; mais chaque démarrage du backend relance `alembic upgrade head`, et si
vous lancez plusieurs instances du backend, elles migrent toutes en même temps.

**Option 2 — un service `migrate` dédié**, construit à partir de la même image, qui s'exécute une
fois puis s'arrête. Le backend attend qu'il ait terminé avec succès :

```yaml
  migrate:
    build: ./backend
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    command: alembic upgrade head

  backend:
    # ...
    depends_on:
      migrate:
        condition: service_completed_successfully
```

Un service de plus dans le `docker-compose.yml`, mais les responsabilités sont séparées : la
migration se lance, se relance et se lit dans les logs indépendamment de l'API.

Dans les deux cas, **ne lancez pas les migrations dans le `startup` de FastAPI** : avec plusieurs
workers ou plusieurs instances, elles s'exécuteraient en concurrence.

## 9. Tester l'application en couches

En séance 2, votre suite pytest tournait sur des dictionnaires remis à zéro par une fixture
`autouse`. Rien n'a changé dans la forme des tests — `TestClient`, Given/When/Then, fixtures dans
`conftest.py` — mais deux choses changent dans le fond : l'état vit maintenant dans PostgreSQL, et
les règles métier vivent dans des services que l'on peut tester sans HTTP.

### 9.1. Une base de test isolée

Vos tests ne doivent **jamais** toucher votre base de développement : ils créent et détruisent des
données, et une suite qui efface vos données de démo est une suite qu'on n'ose plus lancer.

Deux approches viables.

**Option A — une base PostgreSQL dédiée aux tests.** C'est celle que je recommande : vous testez
sur le même moteur qu'en production, donc les contraintes, les types et les erreurs sont les vrais.
Le service `db` de votre Compose peut héberger deux bases ; créez la seconde une fois :

```bash
docker compose exec db createdb -U gearshare gearshare_test
```

et déclarez son URL dans la configuration, à côté de `database_url` (un champ
`test_database_url: str | None = None` dans `Settings`, section 7) :

```bash
# .env.example
TEST_DATABASE_URL=postgresql+psycopg://gearshare:gearshare@db:5432/gearshare_test
```

```python
# tests/conftest.py
from collections.abc import Iterator

import pytest
from fastapi.testclient import TestClient
from sqlalchemy import Engine, create_engine
from sqlalchemy.orm import Session, sessionmaker

from app.api.deps import get_db
from app.core.config import settings
from app.main import app
from app.models import Base


@pytest.fixture(scope="session")
def engine() -> Iterator[Engine]:
    engine = create_engine(settings.test_database_url)
    Base.metadata.drop_all(engine)
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)


@pytest.fixture
def db(engine: Engine) -> Iterator[Session]:
    """Une session par test, dans une transaction annulée à la fin."""
    connection = engine.connect()
    transaction = connection.begin()
    session = sessionmaker(bind=connection, autoflush=False, autocommit=False)()

    yield session

    session.close()
    transaction.rollback()      # tout ce que le test a écrit disparaît
    connection.close()
```

Deux mécanismes à comprendre ici :

1. **La transaction annulée** : chaque test s'exécute dans une transaction dont on fait un
   `rollback` à la fin. La base est identique avant et après, sans recréer les tables. C'est
   l'équivalent exact de votre `reset_storage()` de la séance 2 — en plus rapide, et garanti par le
   moteur. Le scope `session` sur `engine` prend ici tout son sens : créer le schéma coûte cher,
   on le fait une fois.
2. **`Base.metadata.create_all`** crée le schéma directement depuis les modèles. Acceptable pour
   les tests uniquement — en production, ce sont les migrations Alembic qui font foi (section 6,
   règle 5). Si vous voulez aussi valider vos migrations, faites tourner `alembic upgrade head` sur
   la base de test dans votre automatisation de projet : c'est un excellent réflexe.

**Option B — SQLite en mémoire.** Plus simple à mettre en place, mais SQLite n'a pas les mêmes
types, ni la même gestion des contraintes (les clés étrangères sont désactivées par défaut !), ni
les fonctionnalités PostgreSQL que vous utilisez. Vous testerez du code qui passe en test et
échoue en production. **Je vous le déconseille pour ce projet.**

### 9.2. Injecter la session de test

Reste à faire en sorte que l'application, appelée par `TestClient`, utilise cette session et non
celle de `SessionLocal`. C'est le rôle de `app.dependency_overrides` :

```python
@pytest.fixture
def client(db: Session) -> Iterator[TestClient]:
    app.dependency_overrides[get_db] = lambda: db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()
```

La route demande `get_db` (section 3) ; pendant les tests, elle reçoit la session de test. Rien à
changer dans le code de l'application. C'est la raison pour laquelle l'injection de dépendances
n'est pas un luxe : elle rend l'application **substituable morceau par morceau**. Vous
réutiliserez le mécanisme en séance 5 pour `get_current_user`, et plus tard pour n'importe quel
service externe.

Une conséquence à anticiper : votre `get_db` de production fait un `commit()` en fin de requête. La
session de test, elle, ne doit jamais valider — elle vit dans la transaction du test, qui sera
annulée. Le `lambda: db` ci-dessus n'appelle pas `commit()` ; c'est voulu. Si un test a besoin de
vérifier qu'une écriture a bien eu lieu, il relit **avec la même session**.

Vos fixtures de données suivent :

```python
@pytest.fixture
def alice(db: Session) -> User:
    user = User(email="alice@esiee.fr", display_name="Alice", password_hash="x")
    db.add(user)
    db.flush()
    return user


@pytest.fixture
def item_velo(db: Session, alice: User) -> Item:
    item = Item(owner_id=alice.id, titre="Vélo de ville", tarif_jour=Decimal("8.50"))
    db.add(item)
    db.flush()
    return item
```

Le `Given` d'un test est maintenant un objet SQLAlchemy inséré par `flush()` — visible dans la
transaction du test, invisible pour tout le reste, effacé au `rollback`.

### 9.3. Tester les services sans HTTP

Votre `ReservationService` (section 5.2) ne connaît ni HTTP ni FastAPI : il prend des repositories
et lève des exceptions métier. Vous pouvez donc le tester directement, avec la vraie base de test :

```python
@pytest.fixture
def service(db: Session) -> ReservationService:
    return ReservationService(ReservationRepository(db), ItemRepository(db))


def test_reserver_un_creneau_deja_pris_leve_un_conflit(
    service: ReservationService, item_velo: Item, bob: User, carol: User
) -> None:
    # Given : Bob a déjà réservé le créneau.
    service.creer(item_id=item_velo.id, borrower_id=bob.id,
                  debut=date(2026, 9, 1), fin=date(2026, 9, 5))

    # When / Then : un second emprunteur demande un créneau qui chevauche.
    with pytest.raises(ConflitMetier, match="déjà réservé"):
        service.creer(item_id=item_velo.id, borrower_id=carol.id,
                      debut=date(2026, 9, 3), fin=date(2026, 9, 7))
```

Et la paramétrisation de la séance 2 trouve ici son vrai sujet : les cinq cas de chevauchement de
votre `docs/cas-tests-chevauchement.md`, plus les deux que tout le monde oublie.

```python
@pytest.mark.parametrize(
    ("debut_demandee", "fin_demandee", "chevauche"),
    [
        ("2026-09-06", "2026-09-08", False),  # après
        ("2026-08-28", "2026-08-31", False),  # avant
        ("2026-09-03", "2026-09-08", True),   # fin qui déborde
        ("2026-08-30", "2026-09-03", True),   # début qui déborde
        ("2026-08-30", "2026-09-10", True),   # englobante
        ("2026-09-02", "2026-09-04", True),   # incluse
        ("2026-09-05", "2026-09-08", False),  # bornes jointives
    ],
)
def test_detection_chevauchement(
    service: ReservationService,
    reservation_du_1_au_5: Reservation,
    carol: User,
    debut_demandee: str,
    fin_demandee: str,
    chevauche: bool,
) -> None:
    creer = lambda: service.creer(
        item_id=reservation_du_1_au_5.item_id, borrower_id=carol.id,
        debut=date.fromisoformat(debut_demandee), fin=date.fromisoformat(fin_demandee),
    )
    if chevauche:
        with pytest.raises(ConflitMetier):
            creer()
    else:
        creer()
```

Le dernier cas (bornes jointives : une réservation finit le jour où l'autre commence) traduit une
**décision métier** : deux réservations qui se touchent sont-elles en conflit ? La requête
`find_chevauchements` de la section 5.1 répond « non » avec ses inégalités strictes. Votre test
rend la réponse explicite — et quiconque « corrige » `<` en `<=` fera passer ce cas au rouge.

**Où mettre quoi, désormais :**

| Ce que vous testez | Niveau | Outil |
|---|---|---|
| Une règle métier avec ses cas limites | Service, sans HTTP | fixture `service`, `pytest.raises` |
| Le contrat d'une route (codes, corps, `response_model`) | Route | fixture `client` |
| La traduction exception métier → code HTTP | Route, une fois par exception | `client`, un test par ligne de `STATUS_PAR_EXCEPTION` |
| Une requête de repository non triviale | Repository, avec la base de test | fixture `db` |

Ne testez pas la même règle aux trois niveaux : la règle dans le service, la traduction en `409`
une fois dans la route, et c'est tout.

### 9.4. Mocker au bon niveau

La grille de la séance 2 (section 6.1) reste valable, et se précise avec les couches :

| Élément | Choix | Pourquoi |
|---|---|---|
| Règle métier dans un service | Réel | C'est le sujet du test |
| Schémas Pydantic, routes, `response_model` | Réel | Le contrat HTTP doit être vérifié |
| Repository et PostgreSQL de test | Réel | La requête SQL est là où sont les bugs |
| Service tiers (e-mail, paiement, stockage) | Mock | Externe, lent, payant, indisponible en test |
| Horloge et aléa | Valeur contrôlée | Reproductibilité |

Avec des services construits par injection (section 5.2), le mock d'un système tiers ne demande
plus `monkeypatch` : on passe le faux objet au constructeur.

```python
from unittest.mock import Mock


def test_confirmer_reservation_notifie_l_emprunteur(db: Session, reservation_active: Reservation) -> None:
    notifications = Mock()
    service = ReservationService(ReservationRepository(db), ItemRepository(db),
                                 notifications=notifications)

    service.confirmer(reservation_active.id)

    notifications.envoyer_confirmation.assert_called_once_with(
        email=reservation_active.borrower.email, reservation_id=reservation_active.id,
    )


def test_confirmer_reservation_propage_une_panne_de_notification(
    db: Session, reservation_active: Reservation
) -> None:
    notifications = Mock()
    notifications.envoyer_confirmation.side_effect = NotificationIndisponible()
    service = ReservationService(ReservationRepository(db), ItemRepository(db),
                                 notifications=notifications)

    with pytest.raises(NotificationIndisponible):
        service.confirmer(reservation_active.id)
```

Le service et la base restent réels ; seul le système tiers est contrôlé.

Et le piège reste le même qu'en séance 2, avec une variante propre aux couches : **ne mockez pas
votre repository pour tester votre service**. C'est tentant — le test devient « unitaire », il n'a
plus besoin de base — mais un `ItemRepository` simulé qui renvoie toujours l'item attendu ne vous
dira jamais que votre requête de chevauchement rate le cas englobant. La base de test est là,
elle est rapide grâce au `rollback` : utilisez-la.

## 10. Relire une couche complète

Quand vous ajoutez une ressource, ajoutez une **verticale complète** (modèle, schémas, repository,
service, router, migration) plutôt que des morceaux épars. Puis relisez-la avec la grille
ci-dessous, que le code soit le vôtre ou celui d'un camarade : c'est elle que j'appliquerai à
votre rendu.

**Checklist de review d'une couche :**

| Point | Ce que je cherche |
|---|---|
| Séparation | Aucun SQL dans le router, aucun `HTTPException` dans le service |
| `response_model` | Présent, et sans champ interne |
| Schémas | `Create`/`Update`/`Read` distincts, `from_attributes=True` sur `Read` |
| Types | `Numeric` pour l'argent, `DateTime(timezone=True)`, pas de `Float` |
| Index | Sur les nouvelles clés étrangères |
| Migration | Générée, **relue**, avec un `downgrade()` fonctionnel |
| Pas de N+1 | `selectinload` sur les relations chargées en liste |
| Transactions | Pas de `commit()` dispersé dans les repositories |
| Injection SQL | Aucune f-string dans une requête |
| Secrets | Rien en dur, tout dans `config.py` |
| Dépendances | Aucun paquet ajouté sans justification |

Testez ensuite le cycle complet : `alembic upgrade head`, `docker compose up`, appel de la route
depuis `/docs`, et vérification en base avec `psql`. **Un code qui n'a pas été exécuté n'est pas
du code, c'est une hypothèse.**

## 11. Synthèse

- Les couches (router → service → repository → model) ne sont pas de la décoration : elles rendent
  le métier réutilisable, testable, et le code facile à faire évoluer.
- SQLAlchemy : engine unique, session par requête, modèles `Mapped[...]`, relations
  `back_populates`, et attention au **N+1**.
- La session s'injecte par `Depends(get_db)` ; la frontière transactionnelle est un choix
  d'architecture à faire une fois pour toutes.
- Modèle SQLAlchemy ≠ schéma Pydantic. Le `response_model` est votre garde-fou contre les fuites.
- Alembic : `--autogenerate` produit un brouillon à relire, surtout sur les renommages et les
  colonnes `NOT NULL`.
- Configuration en variables d'environnement, `.env.example` commité, `.env` jamais.
- Compose à trois services, `condition: service_healthy`, communication inter-conteneurs par nom
  de service.
- Tests : base PostgreSQL de test dédiée, session par test dans une transaction annulée,
  `dependency_overrides` pour injecter la session ; les règles métier se testent dans le service,
  sans HTTP, et jamais avec un repository mocké.

Passez au [TP 2](../tp/tp2-backend-db.md) : vous branchez le backend sur PostgreSQL et vérifiez
que le frontend du TP 1 fonctionne sur la vraie base.

En séance 5, on ajoute ce qui manque encore : savoir **qui** appelle l'API, et ce qu'il a le droit
de faire.
