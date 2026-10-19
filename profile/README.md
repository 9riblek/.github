# 9riblek — Plateforme de Mise en Relation Clients & Workers

Bienvenue sur l'organisation **9riblek**. Ce dépôt de présentation décrit la vision du projet, l'architecture technique simplifiée pour le MVP (Minimum Viable Product) et le fonctionnement global de la plateforme.

---

## 1. Vision et Objectif du Projet

**9riblek** est une plateforme facilitant la mise en relation directe entre des clients ayant besoin de travaux/services et des travailleurs (*workers*) qualifiés.

Le MVP a pour objectif principal de valider rapidement le cœur de la plateforme :
* Permettre aux clients de publier des annonces de missions (*jobs*).
* Permettre aux workers de parcourir les offres et de postuler.
* Permettre aux clients d'accepter ou de refuser des candidatures.
* Offrir une architecture microservices propre, simple et prête pour le déploiement sur AWS ECS Fargate.

---

## 2. Architecture Globale

L'architecture est construite selon un modèle **microservices** découplé communiquant via des requêtes HTTP REST.

```text
               +-------------------+
               |  Application Web  |
               |    (Frontend)     |
               +---------+---------+
                         |
                         v
               +-------------------+
               |    API Gateway    |
               +----+----+----+----+
                    |    |    |
       +------------+    |    +------------+
       |                 |                 |
       v                 v                 v
+--------------+  +--------------+  +---------------------+
| user-service |  | job-service  |  | application-service |
+--------------+  +--------------+  +---------------------+
```

---

## 3. Description des Microservices

### A. `user-service`
Gère les utilisateurs, l'authentification et les profils.
* **Responsabilités :** Authentification, gestion des comptes, gestion des rôles (CLIENT, WORKER), profil des workers.
* **Endpoints principaux :**
  * `POST /auth/register`
  * `POST /auth/login`
  * `GET /auth/me`
  * `GET /users/{id}` | `PUT /users/{id}` | `DELETE /users/{id}`
  * `GET /workers/{id}` | `PUT /workers/profile`

### B. `job-service`
Gère la création, la publication et le cycle de vie des missions.
* **Responsabilités :** Publication de jobs, mise à jour, recherche et filtre, gestion des statuts (`OPEN`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED`).
* **Endpoints principaux :**
  * `POST /jobs`
  * `GET /jobs` | `GET /jobs/{id}`
  * `PUT /jobs/{id}` | `DELETE /jobs/{id}`
  * `GET /jobs/user/{userId}`
  * `PUT /jobs/{id}/status`

### C. `application-service`
Gère les candidatures envoyées par les workers pour les jobs.
* **Responsabilités :** Postuler à un job, accepter ou refuser une candidature, consulter l'historique des candidatures.
* **Endpoints principaux :**
  * `POST /applications`
  * `GET /applications/job/{jobId}`
  * `GET /applications/user/{userId}`
  * `PUT /applications/{id}/accept`
  * `PUT /applications/{id}/reject`
  * `DELETE /applications/{id}`

---

## 4. Communications Inter-services (REST HTTP)

1. **`job-service` ↔ `user-service`**
   * Vérification de l'existence du client lors de la création d'un job.
   * Récupération des informations du client pour l'affichage d'un job.

2. **`application-service` ↔ `job-service`**
   * Vérification que le job concerné est bien au statut `OPEN` avant l'envoi d'une candidature.
   * Changement automatique du statut du job vers `IN_PROGRESS` lorsqu'une candidature est acceptée.

3. **`application-service` ↔ `user-service`**
   * Validation de l'existence et du rôle du worker candidat.
   * Récupération du profil worker lors de la consultation des candidatures par le client.

---

## 5. Modèle de Données

### Table `users`
| Champ | Type | Description |
| :--- | :--- | :--- |
| `id` | UUID | Identifiant unique |
| `first_name` | VARCHAR | Prénom |
| `last_name` | VARCHAR | Nom |
| `email` | VARCHAR | Adresse email (unique) |
| `password` | VARCHAR | Mot de passe (haché) |
| `phone` | VARCHAR | Numéro de téléphone |
| `role` | ENUM | `CLIENT` ou `WORKER` |
| `city` | VARCHAR | Ville de résidence |
| `profile_image`| TEXT | URL de la photo de profil |
| `bio` | TEXT | Description/biographie |
| `created_at` | TIMESTAMP | Date de création |

### Table `jobs`
| Champ | Type | Description |
| :--- | :--- | :--- |
| `id` | UUID | Identifiant unique |
| `user_id` | UUID | Clé étrangère du client |
| `title` | VARCHAR | Titre de la mission |
| `description` | TEXT | Détails de la mission |
| `category` | VARCHAR | Catégorie de service |
| `budget` | DECIMAL | Budget proposé |
| `city` | VARCHAR | Ville d'exécution |
| `address` | TEXT | Adresse précise |
| `status` | VARCHAR | `OPEN`, `IN_PROGRESS`, `COMPLETED`, `CANCELLED` |
| `created_at` | TIMESTAMP | Date de création |

### Table `applications`
| Champ | Type | Description |
| :--- | :--- | :--- |
| `id` | UUID | Identifiant unique |
| `job_id` | UUID | Clé étrangère du job |
| `worker_id` | UUID | Clé étrangère du worker candidat |
| `proposed_price`| DECIMAL | Offre financière du worker |
| `message` | TEXT | Lettre/Message de motivation |
| `status` | VARCHAR | `PENDING`, `ACCEPTED`, `REJECTED` |
| `created_at` | TIMESTAMP | Date de soumission |

---

## 6. Structure du Frontend & Parcours Utilisateur

* **Pages Publiques :** Landing Page, Inscription (*Register*), Connexion (*Login*).
* **Espace Client :**
  * Création et gestion des jobs.
  * Consultation des candidatures reçues et sélection des candidats.
* **Espace Worker :**
  * Consultation des offres disponibles.
  * Envoi et suivi des candidatures.
  * Gestion du profil professionnel.

---

## 7. Infrastructure et Déploiement

* **Conteneurisation :** Docker
* **Déploiement cible :** AWS ECS Fargate
* **Réseau :** API Gateway pour le routage des requêtes externes
