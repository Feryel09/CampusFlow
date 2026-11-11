# CampusFlow — Event Management Service

Microservice chargé de la **gestion des événements universitaires et des inscriptions des étudiants** dans la plateforme CampusFlow.

> **État :** conception / développement initial. Ce README décrit le périmètre et les contrats d'API prévus ; les fonctionnalités et endpoints ne sont pas nécessairement encore implémentés.

## 1. Responsabilités

- Créer et gérer des événements universitaires (titre, description, lieu, dates, capacité et catégorie).
- Consulter, rechercher et afficher les détails des événements.
- Publier, modifier et annuler un événement selon les autorisations.
- Permettre aux étudiants de s'inscrire ou d'annuler leur inscription.
- Vérifier les places disponibles et éviter les inscriptions en doublon.
- Permettre à l'organisateur de consulter les participants.
- Déclencher les notifications liées aux événements et inscriptions, selon le mécanisme de communication retenu par l'équipe.

### Hors périmètre

- **User Service :** comptes, authentification et rôles. Le service Event ne stocke que des références (`organizerId`, `studentId`).
- **Reservation Service :** réservation effective des salles et gestion de leur disponibilité. Le lieu d'un événement (`location`) ne constitue pas une réservation de salle.
- **Club Service :** gestion des clubs et de leurs membres.
- **Notification Service :** enregistrement et consultation des notifications.

## 2. Modèle de données

### `EventCategory`

| Attribut | Type | Description |
|---|---|---|
| `id` | `Long` | Identifiant de catégorie |
| `name` | `String` | Nom de catégorie |
| `description` | `String` | Description |

### `Event`

| Attribut | Type | Description |
|---|---|---|
| `id` | `Long` | Identifiant de l'événement |
| `title` | `String` | Titre |
| `description` | `String` | Description |
| `location` | `String` | Lieu indiqué de l'événement |
| `startDate` | `LocalDateTime` | Date et heure de début |
| `endDate` | `LocalDateTime` | Date et heure de fin |
| `capacity` | `Integer` | Nombre maximal de participants |
| `status` | `EventStatus` | État de l'événement |
| `organizerId` | `Long` | Référence vers User Service |
| `categoryId` | `Long` | Catégorie de l'événement |

**Opérations principales :** `createEvent()`, `updateEvent()`, `cancelEvent()`.

### `EventRegistration`

| Attribut | Type | Description |
|---|---|---|
| `id` | `Long` | Identifiant d'inscription |
| `eventId` | `Long` | Référence vers `Event` |
| `studentId` | `Long` | Référence vers User Service |
| `registrationDate` | `LocalDateTime` | Date d'inscription |
| `status` | `RegistrationStatus` | État de l'inscription |

**Opérations principales :** `register()`, `cancelRegistration()`.

### Énumérations

```java
public enum EventStatus {
    PLANNED, OPEN, CANCELLED, COMPLETED
}

public enum RegistrationStatus {
    PENDING, CONFIRMED, CANCELLED
}
```

**Relations :** `EventCategory (1) — (0..*) Event` ; `Event (1) — (0..*) EventRegistration`.

## 3. Règles métier essentielles

1. `startDate` doit être strictement antérieure à `endDate`.
2. `capacity` doit être un entier positif.
3. Seul un utilisateur autorisé (organisateur propriétaire ou administrateur, selon les droits définis) peut modifier ou annuler un événement.
4. Un étudiant ne doit pas posséder deux inscriptions actives au même événement.
5. Le nombre d'inscriptions confirmées ne doit jamais dépasser `capacity`, même en cas de demandes simultanées.
6. Une inscription n'est possible que si l'événement est ouvert (`OPEN`) et que les règles de disponibilité sont satisfaites.
7. Une annulation conserve un historique ou un statut adapté plutôt que de supprimer silencieusement les données.
8. Les références externes (`organizerId`, `studentId`) sont vérifiées via les mécanismes d'authentification/API prévus, sans accès direct à la base du User Service.

> Les règles d'approbation d'une inscription (`PENDING` → `CONFIRMED`) doivent être décidées par l'équipe. La présence de ces statuts ne signifie pas qu'une validation manuelle est obligatoire pour tous les événements.

## 4. Contrats d'API REST proposés

Préfixe commun : `/api/v1`.

| Méthode | Endpoint | Action | Réponses principales |
|---|---|---|---|
| `POST` | `/events` | Créer un événement | `201`, `400`, `401`, `403` |
| `GET` | `/events` | Lister / rechercher les événements | `200` |
| `GET` | `/events/{id}` | Consulter les détails | `200`, `404` |
| `PUT` | `/events/{id}` | Modifier un événement | `200`, `400`, `403`, `404` |
| `DELETE` | `/events/{id}` | Annuler un événement | `204`, `403`, `404` |
| `POST` | `/events/{id}/registrations` | S'inscrire à un événement | `201`, `400`, `401`, `404`, `409` |
| `GET` | `/events/{id}/registrations` | Consulter les inscrits (autorisé) | `200`, `403`, `404` |
| `DELETE` | `/events/{id}/registrations/me` | Annuler sa propre inscription | `204`, `401`, `404` |

**Codes :** `200 OK`, `201 Created`, `204 No Content`, `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict` (inscription déjà existante / capacité atteinte).

### Exemple de demande de création

```http
POST /api/v1/events
Content-Type: application/json
Authorization: Bearer <token>
```

```json
{
  "title": "Atelier Java",
  "description": "Atelier pratique pour les étudiants",
  "location": "Bâtiment A - Salle 204",
  "startDate": "2026-12-02T14:00:00",
  "endDate": "2026-12-02T16:00:00",
  "capacity": 40,
  "categoryId": 1
}
```

`organizerId` doit être obtenu à partir du contexte de sécurité validé, et non accepté aveuglément depuis le corps de la requête.

## 5. Communication avec les autres microservices

### Synchrone — REST / JSON

- **User & Authentication Service :** vérifier le compte, l'identité et les autorisations de l'organisateur ou de l'étudiant lorsque les informations disponibles dans le jeton ne suffisent pas.
- **Reservation Service (intégration éventuelle) :** si un événement exige une salle réservable, vérifier ou demander sa réservation via son API. Ne pas gérer soi-même les conflits de salles.

### Asynchrone — événements métier (proposé)

- `EventCreated` / `EventPublished`
- `EventCancelled`
- `EventRegistrationConfirmed`
- `RegistrationCancelled`

Le **Notification Service** pourrait consommer ces événements pour informer les utilisateurs. Le transport (RabbitMQ ou autre mécanisme) reste **à confirmer par l'équipe** ; il n'est pas présenté comme déjà installé.

**Exemple de message :**

```json
{
  "eventId": "message-unique-id",
  "eventType": "EventRegistrationConfirmed",
  "producer": "event-service",
  "occurredAt": "2026-12-01T10:30:00",
  "data": {
    "universityEventId": 12,
    "studentId": 25,
    "registrationId": 91
  }
}
```

La publication et la consommation doivent être conçues pour éviter les traitements en double.

## 6. Technologies prévues

| Domaine | Choix de l'équipe / statut |
|---|---|
| Backend | **Java 17**, Spring Boot *(à confirmer si effectivement retenu)* |
| Frontend | Angular / TypeScript *(application cliente gérée par le dossier Frontend)* |
| Base de données | MySQL ou MariaDB avec XAMPP en local *(choix exact à confirmer)* |
| API | REST / JSON |
| Sécurité | Spring Security / JWT *(si retenu par l'équipe)* |
| Tests d'API | Postman |
| Communication asynchrone | Solution à confirmer ; RabbitMQ seulement si décidé et mis en place |
| Gestion de versions | Git / GitHub |

Chaque microservice doit posséder sa propre base logique et ne pas effectuer de jointures SQL directes avec les tables des autres services.

## 7. Organisation du code (structure cible)

```text
Backend/
└── event-service/
    ├── README.md
    ├── pom.xml                  # Après initialisation du projet Spring Boot
    └── src/
        ├── main/
        │   ├── java/.../event/
        │   │   ├── controller/
        │   │   ├── service/
        │   │   ├── repository/
        │   │   ├── entity/
        │   │   ├── dto/
        │   │   ├── enums/
        │   │   └── config/
        │   └── resources/
        │       └── application.properties
        └── test/
            └── java/.../event/
```

*Structure indicative : les fichiers applicatifs seront créés lors du développement.*

## 8. Tests à prévoir

- Validation des dates et des capacités.
- Consultation des événements disponibles.
- Création / modification / annulation par un utilisateur autorisé.
- Refus des opérations non autorisées (`401` / `403`).
- Inscription à un événement ouvert et annulation d'inscription.
- Refus d'une inscription en doublon (`409`).
- Protection contre le dépassement de capacité en cas de requêtes simultanées.
- Vérification de la bonne gestion des événements de notification, une fois le transport asynchrone choisi.

## 9. Travail Git

Branche dédiée : `event-service`.

La branche est créée à partir de la branche de référence de l'équipe, puis les modifications sont proposées par Pull Request selon la convention Git du projet. Le README global de CampusFlow reste inchangé.

---

**CampusFlow — Event Management Service** · Document de conception du microservice, sans composant IA.
