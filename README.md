# CampusFlow — Reservation Management Service

> Microservice de réservation des salles et espaces universitaires — **conception prévue pour la validation intermédiaire AWD**.

## Présentation

Le **Reservation Management Service** gère les salles universitaires et les demandes de réservation dans CampusFlow. Il permet aux utilisateurs autorisés de consulter les salles et les créneaux disponibles, de demander une réservation et de l'annuler. Les responsables habilités peuvent confirmer ou refuser les demandes.

Ce microservice est **indépendant** des autres domaines CampusFlow : il possède ses propres données et communique avec les autres services via des API ou des événements métier. **Aucune fonctionnalité d'intelligence artificielle n'est incluse dans ce périmètre.**

> **État d'avancement :** ce README décrit les fonctionnalités et les contrats API **prévus**. Il ne signifie pas que les endpoints sont déjà implémentés.

## Responsabilités

- Consulter et gérer les salles universitaires.
- Vérifier la disponibilité d'une salle sur un créneau donné.
- Soumettre une demande de réservation.
- Confirmer, refuser ou annuler une réservation selon les autorisations.
- Empêcher les chevauchements de réservations confirmées.
- Publier les événements métier nécessaires aux notifications.

## Modèle de données

### `Room`

| Attribut | Type | Description |
|---|---|---|
| `id` | `Long` | Identifiant de la salle |
| `name` | `String` | Nom ou numéro de salle |
| `building` | `String` | Bâtiment |
| `capacity` | `Integer` | Capacité maximale |
| `roomType` | `String` | Type de salle |
| `status` | `RoomStatus` | Disponibilité administrative de la salle |

**Méthode principale :** `checkAvailability()`.

### `Reservation`

| Attribut | Type | Description |
|---|---|---|
| `id` | `Long` | Identifiant de réservation |
| `roomId` | `Long` | Salle concernée |
| `userId` | `Long` | Identifiant de l'utilisateur (User Service) |
| `startDateTime` | `LocalDateTime` | Début du créneau |
| `endDateTime` | `LocalDateTime` | Fin du créneau |
| `purpose` | `String` | Motif de réservation |
| `status` | `ReservationStatus` | État de la demande |

**Méthodes principales :** `createReservation()`, `confirmReservation()`, `cancelReservation()`.

### Énumérations

- `RoomStatus` : `AVAILABLE`, `UNAVAILABLE`.
- `ReservationStatus` : `PENDING`, `CONFIRMED`, `REJECTED`, `CANCELLED`.

**Relation :** une `Room` peut posséder `0..*` réservations ; chaque `Reservation` concerne exactement `1` salle.

`userId` est une référence au **User Service**, pas une clé étrangère vers sa base de données.

## Règles métier

1. Une réservation doit avoir `startDateTime < endDateTime`.
2. La salle doit exister et ne pas être administrativement indisponible.
3. Deux réservations **confirmées** ne doivent pas se chevaucher pour la même salle.
4. Une nouvelle réservation peut commencer en `PENDING`, puis être confirmée ou refusée par un rôle habilité.
5. La modification d'un statut est contrôlée par les droits de l'utilisateur.
6. La prévention des doubles réservations doit être garantie côté backend/base de données, y compris lors de demandes simultanées.

**Remarque :** `RoomStatus.AVAILABLE` ne garantit pas qu'une salle soit libre pour un créneau donné ; la disponibilité horaire est calculée à partir des réservations existantes.

## Contrats API REST proposés

Base de routes : `/api/v1`

| Méthode | Endpoint | Action | Réponses principales |
|---|---|---|---|
| `GET` | `/rooms` | Lister les salles | `200` |
| `GET` | `/rooms/{id}` | Consulter une salle | `200`, `404` |
| `GET` | `/rooms/{id}/availability?start=...&end=...` | Vérifier un créneau | `200`, `400`, `404` |
| `POST` | `/reservations` | Demander une réservation | `201`, `400`, `401`, `409` |
| `PATCH` | `/reservations/{id}/confirm` | Confirmer une demande | `200`, `403`, `404`, `409` |
| `PATCH` | `/reservations/{id}/reject` | Refuser une demande | `200`, `403`, `404` |
| `DELETE` | `/reservations/{id}` | Annuler une réservation | `204`, `403`, `404` |

**Codes HTTP :** `200` OK ; `201` Created ; `204` No Content ; `400` Bad Request ; `401` Unauthorized ; `403` Forbidden ; `404` Not Found ; `409` Conflict (par exemple, créneau déjà réservé).

### Exemple de demande de réservation

```http
POST /api/v1/reservations
Content-Type: application/json
Authorization: Bearer <JWT>
```

```json
{
  "roomId": 3,
  "startDateTime": "2026-11-18T14:00:00",
  "endDateTime": "2026-11-18T16:00:00",
  "purpose": "Réunion de club"
}
```

L'identifiant de l'utilisateur doit être déduit du contexte authentifié, et non accepté aveuglément depuis le corps de la requête.

## Communications avec les autres microservices

### Synchrone — REST

```text
Frontend -> API Gateway -> Reservation Management Service
Reservation Management Service -> User & Authentication Service
```

Le service vérifie les autorisations à partir du JWT et, **si nécessaire**, interroge le User Service pour vérifier l'existence, le statut ou les rôles de l'utilisateur. Il n'accède jamais directement à la base de données du User Service.

### Asynchrone — RabbitMQ

Le Reservation Management Service publie les événements métier suivants :

| Événement | Déclencheur |
|---|---|
| `ReservationRequested` | Nouvelle demande créée |
| `ReservationConfirmed` | Demande validée |
| `ReservationRejected` | Demande refusée |
| `ReservationCancelled` | Réservation annulée |

Le **Notification Service** peut consommer ces événements pour notifier les utilisateurs concernés.

Exemple de message :

```json
{
  "eventId": "uuid-unique",
  "eventType": "ReservationConfirmed",
  "occurredAt": "2026-11-18T10:00:00Z",
  "producer": "reservation-service",
  "data": {
    "reservationId": 15,
    "userId": 25,
    "roomId": 3
  }
}
```

Les consommateurs doivent traiter les messages de manière **idempotente** pour éviter les notifications en double.

## Technologies prévues

| Couche | Choix de l'équipe |
|---|---|
| Backend | Java 21, Spring Boot |
| Sécurité | Spring Security, JWT |
| Base de données | PostgreSQL (dédiée au microservice) |
| REST | JSON / HTTP |
| Messagerie | RabbitMQ |
| Client | React, TypeScript (Frontend de CampusFlow) |
| Architecture | Microservices, API Gateway |

## Tests à prévoir

- Création d'une réservation valide.
- Rejet d'un créneau invalide ou d'une salle inexistante.
- Refus d'une confirmation en conflit avec une réservation déjà confirmée.
- Tests de concurrence : deux confirmations simultanées sur un même créneau.
- Vérification des rôles pour confirmer/refuser.
- Publication des événements RabbitMQ sans doublons côté consommateur.
- Annulation d'une réservation et mise à jour de la disponibilité.

## Branches et contribution

Cette fonctionnalité est développée dans la branche : **`feature/reservation-service`**.

Le workflow prévu par l'équipe est : **`feature/*` → `develop` → `main`**, via Pull Request après revue.

> Le contenu de `main` est conservé au moment de la création de la branche. Ce fichier documente uniquement le Reservation Management Service.
