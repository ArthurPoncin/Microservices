# Microservices

# TP Domain Driven Design — Plateforme d’Événements

## 0. Périmètre fonctionnel

La plateforme doit couvrir le cycle de vie complet d’un événement :
- Création, modification, suppression.
- Inscriptions, validation, annulation, listes d’attente.
- Sessions et intervenants.
- Paiement pour les événements payants.
- Génération automatique de factures et reçus.
- Envoi d’emails automatiques (confirmation, rappel, annulation).
- Notifications ciblées.
- Statistiques (participants, revenus, taux de remplissage).

---

## 1. Domaines et sous-domaines

| Domaine | Description |
|----------|--------------|
| **Gestion des événements** | Création, modification, publication, fermeture des inscriptions. |
| **Inscriptions et capacités** | Gestion des demandes, annulations, listes d’attente. |
| **Programme & Intervenants** | Sessions, créneaux, intervenants, salles. |
| **Paiement** | Traitement des paiements en ligne. |
| **Facturation & Reçus** | Génération automatique de documents comptables. |
| **Notifications & Emails** | Envois automatiques et ciblés. |
| **Statistiques** | Agrégation de données analytiques. |

---

## 2. Bounded Contexts (BC)

Chaque **Bounded Context** représente une zone métier cohérente avec ses entités, règles et API.

### A. Catalog / Événements
- **Responsabilité** : gérer les événements (métadonnées, statut, publication).
- **Entités** : `Event`, `Organizer`.
- **Règles** : transitions de statut, cohérence des dates.
- **Événements** : `EventPublished`, `EventUpdated`.

### B. Sessions & Intervenants
- **Responsabilité** : gérer le programme détaillé.
- **Entités** : `Session`, `Speaker`.
- **Règles** : pas de chevauchement, capacité respectée.
- **Événements** : `ProgramChanged`.

### C. Inscriptions & Capacités
- **Responsabilité** : demandes d’inscription, listes d’attente.
- **Entités** : `Registration`, `CapacityPolicy`.
- **Règles** : liste d’attente, promotion automatique, annulation libère une place.
- **Événements** : `RegistrationConfirmed`, `Waitlisted`, `PromotedFromWaitlist`.

### D. Paiement
- **Responsabilité** : paiement en ligne.
- **Entités** : `PaymentIntent`, `Charge`.
- **Règles** : paiement confirmé = réservation.
- **Événements** : `PaymentSucceeded`, `PaymentFailed`.

### E. Facturation & Reçus
- **Responsabilité** : génération automatique de factures et reçus.
- **Entités** : `Invoice`, `Receipt`.
- **Règles** : immutabilité, numérotation légale.
- **Événements** : `InvoiceIssued`.

### F. Notifications & Emails
- **Responsabilité** : emails et notifications ciblées.
- **Entités** : `MessageTemplate`, `Delivery`.
- **Événements consommés** : `RegistrationConfirmed`, `ProgramChanged`.
- **Règles** : idempotence, filtres ciblés.

### G. Statistiques & Reporting
- **Responsabilité** : fournir les indicateurs de performance.
- **Événements consommés** : `Registration*`, `Payment*`, `InvoiceIssued`.

---

## 3. Microservices proposés

| Microservice | Rôle principal | API principale |
|---------------|----------------|----------------|
| **Event Catalog Service** | CRUD et publication d’événements | `/events`, `/events/{id}/publish` |
| **Program Service** | Sessions, intervenants | `/events/{id}/sessions` |
| **Registration Service** | Inscriptions, files d’attente | `/registrations` |
| **Payment Service** | Paiement en ligne | `/payments/intents` |
| **Billing Service** | Factures, reçus | `/invoices`, `/receipts` |
| **Notification Service** | Emails, notifications ciblées | `/notifications` |
| **Analytics Service** | Statistiques, KPIs | `/analytics` |

---

## 4. Scénarios d’interaction

### A. Inscription payante avec facture
1. Le front envoie une demande d’inscription.
2. `Registration` crée la demande → état `pending`.
3. `Payment` traite le paiement → `PaymentSucceeded`.
4. `Registration` confirme la place → `RegistrationConfirmed`.
5. `Billing` émet une facture → `InvoiceIssued`.
6. `Notification` envoie l’email de confirmation.
7. `Analytics` met à jour les données.

### B. Liste d’attente
- Si capacité atteinte → `Waitlisted`.
- À l’annulation → promotion automatique → `PromotedFromWaitlist`.

### C. Changement de programme
- `ProgramChanged` déclenche `Notification` ciblée aux inscrits concernés.

---

## 5. Règles métier (invariants)

| Entité | Invariants |
|--------|-------------|
| **Event** | Dates cohérentes, transitions de statut valides. |
| **Session** | Pas de chevauchement salle/créneau. |
| **Registration** | 1 place confirmée = 1 siège, annulation libère. |
| **Payment** | Idempotence sur webhooks. |
| **Invoice** | Immuable, annulée via avoir. |

---

## 6. Avantages du découpage

- Alignement fort **métier ↔ code**.  
- **Évolutivité ciblée** (paiement, emails).  
- **Résilience** : chaque service isolé.  
- **Lisibilité** du modèle métier.  
- Prépare une architecture **microservices** claire et extensible.

---

## 7. Synthèse

| Élément | Détail |
|----------|--------|
| Domaines | Catalog, Program, Registration, Payment, Billing, Notifications, Analytics |
| Architecture | Un microservice par contexte borné |
| Communication | API REST + événements asynchrones |
| Données | Une base par service |
| Bénéfices | Cohérence, évolutivité, découplage, clarté |

---

© Réponse structurée — Application DDD pour plateforme d’événements.
