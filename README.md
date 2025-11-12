# TP Domain Driven Design — Plateforme d’Événements

---

## 1) Analyse des domaines et sous-domaines fonctionnels

| Domaine | Sous‑domaines | Description (langage métier) |
|---|---|---|
| **Gestion des événements** | Définition, Publication, Fermeture des inscriptions | Créer et administrer les événements (titre, dates, lieu, capacité, statut), décider de l’ouverture/fermeture des inscriptions. |
| **Inscriptions** | Demandes, Validation/Annulation, Liste d’attente, Capacité | Prendre les demandes d’inscription, confirmer/annuler, gérer la file d’attente lorsque la capacité est atteinte. |
| **Programme** | Sessions, Intervenants | Organiser le contenu d’un événement : sessions/ateliers, créneaux, salles, intervenants. |
| **Paiement** | Intention, Validation/Échec | Permettre le règlement des événements payants et refléter l’état du paiement. |
| **Facturation** | Factures, Reçus | Émettre les documents comptables liés aux paiements validés. |
| **Notifications** | Courriels, Alertes ciblées | Informer automatiquement participants et organisateurs (confirmation, rappel, annulation, changement de programme). |
| **Statistiques** | Indicateurs, Rapports | Mesurer l’activité : participation, taux de remplissage, revenus, tableaux de bord. |

---

## 2) Bounded Contexts — responsabilités, entités principales et interactions

> Objectif : rester dans l’**Ubiquitous Language** (métier), sans détails techniques.

| Contexte | Responsabilité principale | Entités clés | Interactions (métier) | Règles principales |
|---|---|---|---|---|
| **Gestion des Événements** | Gérer le cycle de vie des événements (création, modification, publication, fermeture des inscriptions). | *Événement*, *Organisateur* | Quand un événement est **publié**, il devient visible et inscriptible. Quand il est **modifié** (dates/lieu), **Inscriptions**, **Programme** et **Notifications** sont informés. En **fermeture**, plus d’inscriptions. | Publication possible seulement si dates/lieu/capacité valides. Un événement publié ne peut pas être supprimé. |
| **Inscriptions** | Prendre les demandes, gérer la capacité, confirmer/annuler, maintenir la liste d’attente. | *Inscription*, *Participant*, *Liste d’attente* | Publication d’un événement → inscriptions ouvertes. Capacité atteinte → nouvelles demandes en attente. Annulation → promotion du premier en file. Paiement validé (si payant) → inscription confirmée. | Une inscription confirmée occupe une place. Une annulation libère une place. Pas de doublon pour un même participant/événement. |
| **Programme** | Organiser sessions/ateliers et intervenants de chaque événement. | *Session*, *Intervenant* | Publication d’un événement → planification possible. Changement/annulation de session → **Notifications** ciblées aux participants concernés. | Pas de chevauchement de sessions dans une même salle. Un intervenant n’est associé qu’à des événements publiés. |
| **Paiement** | Permettre le règlement des inscriptions d’événements payants. | *Paiement*, *Montant dû* | Inscription payante créée → paiement attendu. Paiement **validé** → **Inscriptions** confirme la place et **Facturation** émet la facture. Paiement **échoué** → participant informé. | Le paiement couvre le montant dû. Un paiement validé n’est plus modifiable (politique d’annulation/remboursement dédiée). |
| **Facturation** | Produire et mettre à disposition les factures et reçus associés aux paiements. | *Facture*, *Reçu* | Paiement **validé** → émission d’une facture/du reçu et mise à disposition. **Notifications** informe le participant. **Statistiques** consolident les montants. | Facture immuable après émission ; corrections via avoir. Numérotation unique. |
| **Notifications** | Informer automatiquement (confirmation, rappel, annulation, changement de programme) et effectuer des envois ciblés. | *Notification*, *Modèle de message* | Confirmation/annulation d’inscription → message au participant. Changement de programme → message uniquement aux concernés. Facture disponible → message avec lien. | Un envoi par événement déclencheur. Modèles validés, messages clairs. |
| **Statistiques** | Fournir des indicateurs consolidés et des rapports. | *Indicateur*, *Rapport statistique* | Consomme inscriptions confirmées/annulations/paiements/factures/événements/programme pour calculer les KPIs. Rapports consultables par organisateurs. | Comptage basé sur données validées (inscriptions confirmées, paiements réussis). Cohérence temporelle des indicateurs. |

---

## 3) Découpage en microservices

> Découpage aligné sur les contexts. Chaque service **possède** ses données et expose des interfaces claires.  
> Intégration de votre proposition (`EventService`, `RegistrationService`, etc.), harmonisée en langage fonctionnel.

| Microservice (nom clair) | Responsabilité principale | Données principales (propriété) | Interfaces exposées (API) / Interactions |
|---|---|---|---|
| **EventService** | Gérer le catalogue d’événements : création, mise à jour, publication, fermeture des inscriptions. | **Événements** (id, titre, dates, lieu, capacité, statut), fenêtres d’inscription. | `POST /events` • `PUT /events/{id}` • `GET /events/{id}` • **Publie :** `event.created`, `event.updated`, `event.published`, `event.closed` |
| **RegistrationService** | Gérer inscriptions et listes d’attente ; garantir la capacité. | **Inscriptions** (id, eventId, participantId/email, statut), **ListesAttente** (rang), registre de capacité. | `POST /events/{eventId}/register` • `POST /registrations/{regId}/cancel` • `GET /events/{eventId}/attendees` • **Écoute :** `event.created|published|closed` • **Publie :** `registration.confirmed|cancelled|waitlisted|promoted` • **Appelle :** *PaymentService* (si payant) |
| **AgendaService** | Organiser sessions et intervenants pour chaque événement. | **Sessions** (id, eventId, titre, horaire, salle, capacité), **Intervenants**. | `POST /events/{eventId}/sessions` • `PUT /sessions/{sessionId}` • `GET /events/{eventId}/agenda` • **Publie :** `agenda.updated` • **Écoute :** `event.published|updated` |
| **PaymentService** | Traiter les paiements des inscriptions payantes. | **Transactions** (id, registrationId, montant, devise, statut, référence PSP). | `POST /payments` (interne) • Webhook prestataire • **Publie :** `payment.succeeded`, `payment.failed` |
| **BillingService** | Générer et archiver les documents comptables. | **Factures** (id, userId, registrationId, montant, date, numéro, lienPDF), **Reçus**. | `GET /invoices/{invoiceId}` • `GET /users/{userId}/invoices` • **Écoute :** `payment.succeeded` • **Publie :** `invoice.issued` |
| **NotificationService** | Envoyer emails et alertes ciblées (confirmation, annulation, rappels, changements). | **Templates**, **Messages envoyés** (journal). | `POST /notifications/send` (interne) • **Écoute :** `registration.confirmed|cancelled`, `event.cancelled|updated`, `agenda.updated`, `invoice.issued` |
| **AnalyticsService** | Produire indicateurs et tableaux de bord. | **EventStats** (modèle de lecture dénormalisé : participants, revenus, taux de remplissage). | `GET /analytics/events/{eventId}/stats` • `GET /analytics/dashboard` • **Écoute :** `registration.confirmed|cancelled`, `payment.succeeded`, `invoice.issued`, `event.*`, `agenda.updated` |

> Remarques d’alignement :  
> • Les noms de messages d’interaction restent indicatifs (langage domaine).  
> • Chaque service est **déployable indépendamment** et possède sa **propre base** (ownership strict).  
> • Les interfaces sont décrites côté **contrat** ; la technique (HTTP, messages) reste implicite.

---
