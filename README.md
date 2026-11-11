# CampusFlow

**University Life Management Platform based on Microservices Architecture**

## 1. Présentation du projet

CampusFlow est une plateforme web destinée à centraliser et faciliter la gestion de la vie universitaire. Elle permet aux étudiants d'accéder à différents services depuis une seule application.

Le projet repose sur une architecture distribuée basée sur les microservices.

## 2. Fonctionnalités principales

- **Gestion des utilisateurs :** gestion des comptes et authentification.
- **Gestion des événements :** création, consultation et gestion des événements universitaires.
- **Gestion des réservations :** gestion des réservations liées aux activités universitaires.
- **Gestion des clubs :** gestion des clubs, de leurs membres et de leurs activités.
- **Covoiturage :** mise en relation des étudiants pour partager leurs déplacements.
- **Objets perdus et trouvés :** déclaration et recherche des objets perdus.
- **Notifications :** diffusion des notifications aux utilisateurs.

## 3. Architecture du projet

CampusFlow adopte une architecture microservices composée de plusieurs services indépendants :

- User Service
- Event Service
- Reservation Service
- Club Service
- Carpooling Service
- Lost & Found Service
- Notification Service
- API Gateway

## 4. Organisation du dépôt

```text
CampusFlow/
├── Backend/
│   ├── user-service/
│   ├── event-service/
│   ├── reservation-service/
│   ├── club-service/
│   ├── carpooling-service/
│   ├── lost-found-service/
│   ├── notification-service/
│   └── api-gateway/
│
├── Frontend/
│   └── campusflow-frontend/
│
├── Documentation/
│   ├── Presentation/
│   ├── Diagrams/
│   └── Technical-Report/
│
└── README.md
```

## 5. Technologies

Les technologies seront précisées et mises à jour au fur et à mesure de l'avancement du projet.

## 6. Équipe

Projet réalisé par une équipe de six étudiants dans le cadre du module Architecture Web Distribuée (AWD).
