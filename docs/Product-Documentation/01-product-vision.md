# 01 — Vision du produit

> **Projet :** Barber Queue Management System (nom de travail)
> **Statut du document :** version 1.0 — 24/11/2026
> **Sources :** décisions D1–D12, Q1–Q15, H1–H9, R1–R3 (phase d'analyse) et diagrammes `01-use-case-diagram.puml`, `02-mcd.puml`, `03-mld.puml`.

## 1. L'idée

Une application mobile qui permet à un client de **rejoindre à distance la file d'attente d'un barbier** ou de **réserver un rendez-vous**, de suivre sa position et son temps d'attente estimé, et d'être **prévenu par notification** quand son tour approche.
Pour le salon, l'application remplace la liste papier ou la mémoire du barbier par une file numérique fiable, par barbier, qui tient compte des rendez-vous, des horaires et des pauses.

## 2. Vision

Un salon de coiffure où personne ne doit deviner l'ordre de passage : le client sait où il en est, le barbier sait qui appeler, le propriétaire garde la maîtrise de son salon.

## 3. Objectifs

### 3.1 Objectifs produit (MVP)
1. Gérer une **file d'attente par barbier**, avec ticket fixe, position, estimation du temps d'attente et statuts clairs.
2. Gérer des **rendez-vous confirmés automatiquement**, sans double réservation.
3. Faire cohabiter file et rendez-vous (le temps d'attente tient compte des rendez-vous).
4. Permettre au barbier d'**ajouter un client qui n'utilise pas encore l'application**, sans anonymat : chaque client a un compte (voir `05-product-requirements.md`).
5. Prévenir le client par **notification push** (tour proche, tour arrivé, tour dépassé).
6. Offrir des **rôles par salon** (propriétaire principal, propriétaire secondaire, barbier, staff, gestionnaire de file délégué).
7. Protéger les données personnelles (recherche de clients limitée et masquée, journal d'audit).

### 3.2 Objectifs d'apprentissage (projet étudiant)
Construire une application professionnelle tout en progressant en Java, Spring Boot, PostgreSQL, React Native, TypeScript, SOLID, tests logiciels et Clean Code. Le MVP doit rester un **monolithe modulaire**, sans sur-ingénierie.

### 3.3 Indicateurs de succès
**À définir.** Aucun objectif chiffré n'a été validé. Des indicateurs possibles (à confirmer, sans valeur cible pour l'instant) : nombre de salons pilotes, part des clients qui consultent leur position, taux d'absence (`NO_SHOW`).

## 4. Public cible

| Profil | Besoin principal | Rôle dans l'application |
|---|---|---|
| **Client** | Éviter l'attente sur place, connaître son tour | `Client` (tout compte utilisateur) |
| **Barbier** | Ordre de passage clair, moins d'interruptions | `BARBER` |
| **Réceptionniste / assistant** | Gérer la file sans couper les cheveux | `STAFF` (+ délégation `QUEUE_MANAGER`) |
| **Propriétaire de salon** | Organiser le salon, les prix, les horaires, les règles | `OWNER` (principal ou secondaire) |

**Contexte géographique :** Maroc (fuseau `Africa/Casablanca`). Un seul fuseau dans le MVP.
**Langues de l'interface :** *à confirmer* (non décidé pendant l'analyse).

## 5. Périmètre

**Dans le MVP :** comptes et vérification, salons et membres, services et prix par barbier, horaires (plusieurs périodes par jour), file d'attente, rendez-vous, notifications push, délégation de gestion de file, transfert de propriété, historique des visites, journal d'audit.

**Hors MVP (reporté, voir `05-product-requirements.md` §7) :** paiement en ligne, avis, fidélité, SMS en production (étude à faire), suivi des absences répétées, report de rendez-vous, plusieurs fuseaux.

## 6. Principes directeurs
- **Une règle claire vaut mieux qu'une règle flexible** (exemple : un seul engagement actif par client et par jour).
- **Les permissions se vérifient côté Backend**, par salon, jamais dans le seul Frontend.
- **L'historique ne se supprime pas** : annulations et corrections sont tracées.
- **Le MVP reste simple** : ce qui n'est pas validé n'est pas développé.
