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

## 2. Définition des Bounded Contexts — Vue d’ensemble (Langage Métier)

| **Contexte** | **Responsabilité principale** | **Entités clés** | **Interactions (métier, non techniques)** | **Règles principales** |
|---------------|-------------------------------|------------------|-------------------------------------------|------------------------|
| **Gestion des Événements** | Gérer le cycle de vie des événements : création, modification, publication, fermeture des inscriptions. | Événement, Organisateur | • Lorsqu’un événement est publié, il devient visible et les inscriptions peuvent s’ouvrir.<br>• Lorsqu’il est modifié (date, lieu, description), les inscriptions, le programme et les notifications doivent être mis à jour.<br>• Lorsqu’il est fermé, plus aucune inscription n’est possible. | • Un événement ne peut être publié que s’il possède une date, un lieu et une capacité valides.<br>• Un événement publié ne peut être supprimé. |
| **Inscriptions** | Gérer les demandes d’inscription, confirmations, annulations et listes d’attente. | Inscription, Participant, Liste d’attente | • Lorsqu’un événement est publié, les inscriptions peuvent s’ouvrir.<br>• Quand la capacité est atteinte, les nouvelles demandes sont placées sur liste d’attente.<br>• Lorsqu’un participant annule, le premier de la liste obtient une place.<br>• Lorsqu’un paiement est confirmé, l’inscription devient « confirmée ». | • Une inscription confirmée occupe une place.<br>• Une annulation libère une place.<br>• Une même personne ne peut pas s’inscrire deux fois au même événement. |
| **Paiement** | Gérer les paiements pour les événements payants. | Paiement, Montant dû | • Lorsqu’une inscription payante est créée, un paiement est attendu.<br>• Lorsqu’un paiement est validé, l’inscription correspondante est confirmée et la facturation est déclenchée.<br>• Lorsqu’un paiement échoue, le participant en est informé. | • Le paiement doit couvrir le montant exact dû.<br>• Un paiement validé ne peut pas être modifié, seulement annulé selon les règles de remboursement. |
| **Facturation** | Produire les documents comptables liés aux paiements (factures, reçus). | Facture, Reçu | • Lorsqu’un paiement est validé, une facture est automatiquement générée.<br>• Les participants reçoivent une copie de leur facture ou un reçu.<br>• Les factures alimentent les statistiques financières. | • Une facture émise ne peut pas être modifiée.<br>• En cas d’erreur, un avoir doit être généré.<br>• Chaque facture possède un numéro unique. |
| **Programme & Intervenants** | Organiser les sessions, ateliers et intervenants associés aux événements. | Session, Intervenant | • Lorsqu’un événement est publié, les sessions peuvent être planifiées.<br>• Lorsqu’une session est modifiée ou annulée, les participants concernés doivent être informés.<br>• Les inscriptions peuvent être liées à des sessions spécifiques. | • Un intervenant ne peut être associé qu’à des événements publiés.<br>• Une session ne peut pas se chevaucher avec une autre dans la même salle. |
| **Notifications** | Informer automatiquement les participants et organisateurs selon les actions de la plateforme. | Notification, Modèle de message | • Lorsqu’une inscription est confirmée ou annulée, une notification est envoyée.<br>• Lorsqu’un programme change, seuls les participants concernés sont avertis.<br>• Lorsqu’une facture est disponible, le participant reçoit une notification avec le lien. | • Chaque notification doit être envoyée une seule fois.<br>• Les messages doivent être clairs et validés avant envoi. |
| **Statistiques** | Fournir une vision d’ensemble de l’activité : participation, taux de remplissage, revenus. | Indicateur, Rapport statistique | • Les inscriptions, paiements et annulations alimentent les indicateurs.<br>• Les organisateurs peuvent consulter les statistiques de leurs événements.<br>• Les données de facturation permettent de calculer le chiffre d’affaires. | • Les statistiques reposent uniquement sur des inscriptions confirmées et paiements validés.<br>• Les rapports doivent être cohérents et actualisés. |

---

> Cette vue d’ensemble exprime les **frontières fonctionnelles** et les **relations métier** entre les différents contextes, sans mention technique (API, messages, bases de données, etc.).  
> Elle sert de base pour comprendre le modèle conceptuel du domaine et établir un **langage commun** entre développeurs et experts métier.

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
