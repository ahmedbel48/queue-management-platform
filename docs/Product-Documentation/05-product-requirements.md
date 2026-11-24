# 05 — Exigences produit (vue fonctionnelle)

> **Rôle de ce document :** présenter le produit (rôles, fonctionnalités, règles principales). Il **ne duplique pas** les règles détaillées ni les états : ceux-ci seront dans `04-business-rules.md` et `05-state-machines.md` (non encore rédigés, missions `M00-02` et `M00-03`, voir `02-Missions/A-faire/`).
> **Sources :** décisions validées (D1–D12, Q1–Q15, H1–H9, R1–R3) et diagrammes de la Livraison 1 (**en attente de validation**).
> **Aucune fonctionnalité n'est ajoutée ici** : tout ce qui suit provient des décisions ou des diagrammes. Les points non décidés sont marqués « à confirmer ».

## 1. Rôles utilisateurs

| Rôle | Portée | Description |
|---|---|---|
| **User / Client** | Compte | Tout compte peut rejoindre une file, réserver, consulter son historique. |
| **BARBER** | Salon | Fournit des services, gère **sa** file, ses périodes de travail et ses congés. |
| **STAFF** | Salon | Membre sans file propre. Ne gère une file que par délégation `QUEUE_MANAGER`. |
| **OWNER — PRIMARY_OWNER** | Salon | Un seul par salon. Seul à pouvoir nommer, retirer ou rétrograder un propriétaire secondaire, transférer la propriété et fermer le salon. |
| **OWNER — SECONDARY_OWNER** | Salon | Gestion quotidienne : membres BARBER/STAFF, services et prix, files et rendez-vous, horaires, règles de file, délégations. Mêmes droits pour tous les secondaires (Q12). |
| **QUEUE_MANAGER** | Délégation | Permission complémentaire donnée par un propriétaire à un BARBER ou STAFF, limitée aux files choisies. |
| **Système (Scheduler)** | Technique | Envoie les notifications de tour proche, passe en `SKIPPED` après le délai de présence, fait expirer les transferts de propriété. |

Les rôles sont **par salon** : un utilisateur peut être propriétaire dans un salon et simple barbier dans un autre (D11). Les droits sont vérifiés côté Backend à chaque requête.

## 2. Fonctionnalités du MVP

Les identifiants `UC` renvoient à `01-use-case-diagram.puml`.

### 2.1 Comptes et authentification — UC01 à UC08
- FR-01 Créer un compte avec e-mail ou téléphone (au moins un identifiant, uniques).
- FR-02 Vérifier le contact par code (e-mail en premier ; SMS selon étude).
- FR-03 Définir son mot de passe **après** vérification. Un compte non vérifié ne peut pas se connecter.
- FR-04 Se connecter par e-mail + mot de passe ou téléphone + mot de passe.
- FR-05 Session : jeton d'accès court et jeton de rafraîchissement stocké haché, avec rotation et révocation.
- FR-06 Gérer son profil, enregistrer son appareil pour les notifications.

### 2.2 Découverte — UC09 à UC12
- FR-07 Parcourir les salons, consulter détails, barbiers, services et prix.
- FR-08 Consulter son historique de visites (vue basée sur les visites terminées) et ses notifications.

### 2.3 File d'attente — UC20 à UC44
- FR-09 Rejoindre la file d'un barbier en choisissant un ou plusieurs services (durée et prix totaux).
- FR-10 Voir sa position et son temps d'attente estimé ; annuler son entrée avant le début du service.
- FR-11 Barbier / gestionnaire / propriétaire secondaire : voir la file, appeler le client suivant, dépasser un client (`SKIPPED`), rappeler un client dépassé, prolonger le délai de présence, démarrer et terminer le service, marquer `NO_SHOW`, annuler une entrée avec motif, réorganiser.
- FR-12 Ajouter un client : rechercher son compte (résultat masqué, 30 recherches par heure) ou en créer un en `PENDING_VERIFICATION`, puis l'ajouter à la file.
- FR-13 Le système notifie (tour proche, tour arrivé, tour dépassé, annulation) et recalcule ordre et estimations après chaque événement.

### 2.4 Rendez-vous — UC50 à UC60
- FR-14 Consulter les disponibilités et réserver : le rendez-vous est directement `CONFIRMED`.
- FR-15 Annuler avant la limite d'annulation (15 minutes par défaut, configurable, figée à la réservation).
- FR-16 Barbier / propriétaire secondaire : voir l'agenda, annuler à titre exceptionnel avec motif, démarrer, terminer, marquer `NO_SHOW`.

### 2.5 Administration du salon — UC70 à UC83
- FR-17 Créer un salon (le créateur devient `PRIMARY_OWNER`).
- FR-18 Gérer informations, catalogue de services du salon, prix et durées par barbier.
- FR-19 Ajouter, modifier, désactiver des membres BARBER/STAFF.
- FR-20 Gérer périodes de travail (plusieurs par jour), congés, fermetures du salon.
- FR-21 Configurer les règles du salon (délais de rappel et de présence, taille de file, annulation de rendez-vous, pas des créneaux...).
- FR-22 Attribuer ou révoquer `QUEUE_MANAGER`.
- FR-23 Consulter l'activité et des statistiques de base — **contenu des statistiques à confirmer** (UC82, jamais détaillé pendant l'analyse).
- FR-24 Consulter le journal d'audit.

### 2.6 Propriété — UC90 à UC99
- FR-25 Nommer, retirer ou rétrograder un propriétaire secondaire (propriétaire principal).
- FR-26 Proposer, accepter, refuser, annuler ou laisser expirer un transfert de propriété (ré-authentification par mot de passe requise).
- FR-27 Fermer définitivement un salon (refusé s'il reste des entrées actives ou des rendez-vous futurs confirmés).

## 3. Règles métier principales (résumé)

Le détail, avec identifiants `BR-xx`, sera dans `04-business-rules.md`.

| Domaine | Règle | Décision |
|---|---|---|
| File | `ticket_number` fixe ; ordre = (1) `CALLED`/`IN_SERVICE`, (2) `SKIPPED` par premier dépassement, (3) `WAITING`/`NOTIFIED` par position. | D5, Q1 |
| File | Un seul client `CALLED` ou `IN_SERVICE` par barbier (409 sinon). | R1 |
| File | `NO_SHOW` est définitif ; un `IN_SERVICE` ne s'annule pas, il se termine en `COMPLETED`. | R2 |
| File | Annulable : `WAITING`, `NOTIFIED`, `CALLED`, `SKIPPED`. Par le personnel : motif obligatoire et audit. | Q15, H7 |
| Notifications | `UPCOMING_TURN` quand l'ETA ≤ délai de rappel du salon ; pas de doublon ; l'absence de réaction ne bloque pas la file. | D8, R3 |
| Rendez-vous | Statuts `CONFIRMED`, `CANCELLED`, `COMPLETED`, `NO_SHOW` ; aucun chevauchement par barbier. | Q9 |
| Rendez-vous | Date limite d'annulation calculée à la réservation, non modifiée ensuite. | Q14, H4 |
| Clients | Pas de client anonyme ; compte unique par e-mail/téléphone ; le barbier ne connaît jamais le mot de passe. | Q2 |
| Clients | **Un seul engagement actif par client et par jour local**, tous salons confondus (file ou rendez-vous). | D7, Q13, Q13-bis, H5 |
| Services | Un client peut choisir plusieurs services ; prix et durées figés à la confirmation. | D4, D10 |
| Horaires | Plusieurs périodes par jour ; pauses = intervalles ; pas de chevauchement ; pas de période au-delà de minuit. | D9, Q5 |
| Permissions | Rôles par salon, vérifiés en base ; jeton d'accès = identité seulement. | D11, Q7, Q12 |
| Propriété | Exactement un `PRIMARY_OWNER` actif par salon ; changement par transfert accepté par le destinataire. | Q3, Q10, Q10-bis |
| Vie privée | Recherche de client : prénom + initiale, contact masqué, 30 recherches/h par utilisateur, journal sans contact en clair. | Q11, H6, H8, H9 |
| Historique | Visites terminées conservées ; l'historique est une vue sur les visites. | Q6 |
| Authentification | E-mail ou téléphone, vérifiés avant usage ; session avec rotation. | D12, Q8 |

## 4. Exigences non fonctionnelles

| Domaine | Exigence | Statut |
|---|---|---|
| Sécurité | Mots de passe et codes hachés ; permissions vérifiées côté Backend ; limitation des envois de codes et des recherches. | Décidé (principes) ; valeurs détaillées à écrire en `BR` |
| Fiabilité | Garanties de concurrence : contraintes PostgreSQL + transactions + verrou. | Décidé |
| Temps | Instants en `TIMESTAMPTZ`, fuseau du salon `Africa/Casablanca`, horloge du serveur uniquement. | Décidé |
| Traçabilité | Journal d'audit en ajout seul. | Décidé |
| Performance | **Aucun objectif chiffré** décidé. | À confirmer |
| Disponibilité, hébergement, sauvegardes | Non traités pendant l'analyse. | À confirmer |
| Langues de l'interface | Non décidées. | À confirmer |
| Accessibilité | Non traitée. | À confirmer |

## 5. Architecture (rappel)
Monolithe modulaire : Auth, Users, Salons, Memberships, Barbers, Services, Queue, Appointments, Notifications, Working Hours, Audit Log. Backend Java / Spring Boot / PostgreSQL ; Frontend React Native / Expo / TypeScript ; communication REST. Le détail des dépendances entre modules sera dans `06-architecture-review.md` (mission `M00-04`).

## 6. Contraintes assumées du MVP
Un seul service par visite n'est **plus** une limite (D10). Limites actuelles : un seul fuseau, un engagement par jour, pas de report de rendez-vous, pas de restauration d'un `NO_SHOW`, pas de permissions personnalisées pour les secondaires, pas d'invitation avec acceptation pour les nouveaux membres.

## 7. Hors périmètre / V2
Fournisseur SMS en production (selon étude), suivi des absences répétées, limite de rendez-vous futurs, report de rendez-vous, restauration d'un `NO_SHOW`, plusieurs fuseaux, vue matérialisée de l'historique, archivage, alerte anti-énumération, confirmation à deux pour les transferts, permissions personnalisées, paiement, avis, fidélité.

## 8. Points à confirmer
Voir `02-Missions/A-faire/M00-05-clarifier-points-ouverts.md`.
