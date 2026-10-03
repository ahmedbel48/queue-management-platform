Technical Design

1. Objectif

Ce document décrit la conception technique détaillée de la plateforme de gestion de file d’attente.
Il traduit les décisions métier et architecturales validées dans DECISIONS.md et le contexte fonctionnel décrit dans PROJECT_CONTEXT.md en une structure technique cohérente et implémentable.
Le document couvre notamment :

les frontières des modules ;

les dépendances entre modules ;

les couches Domain / Application / Infrastructure ;

les Aggregate Roots et leurs responsabilités ;

les Entités et Value Objects ;

les Repository Ports ;

les Ports et Adapters ;

les Use Cases ;

les transactions et la concurrence ;

la persistance JPA / PostgreSQL ;

les contraintes de base de données ;

les frontières de l’API REST ;

la sécurité ;

la stratégie de tests.

Ce document ne remplace pas DECISIONS.md.
DECISIONS.md reste la source officielle des décisions validées.
PROJECT_CONTEXT.md décrit le contexte fonctionnel et architectural global.
TECHNICAL_DESIGN.md décrit comment ces décisions seront traduites techniquement.

2. Module Boundaries

2.1 Principes

Un module représente une responsabilité métier ou technique cohérente.
La création d’un module ne doit pas être justifiée uniquement par l’existence d’une classe ou d’un ensemble de classes.
Les modules doivent avoir :

une responsabilité clairement définie ;

des dépendances explicites ;

une faible connaissance des détails internes des autres modules ;

des interfaces claires pour les interactions inter-modules.

Le système reste un Modular Monolith.
Les modules ne sont pas des microservices et ne possèdent pas de déploiement indépendant dans le MVP.

2.2 Modules

Le MVP est organisé autour des responsabilités suivantes :
Identity Organization Catalog Provider Queue Scheduling
Les notifications externes sont traitées par un mécanisme de Port/Adapter et ne constituent pas un domaine métier autonome dans le MVP.

2.3 Identity

Responsabilité :

gestion de l’identité utilisateur ;

authentification par téléphone + OTP ;

gestion des credentials nécessaires à l’authentification ;

émission et validation des tokens d’API.

Concepts principaux :

User

Le module Identity ne doit pas dépendre des détails métier de Queue, Scheduling ou Catalog.

2.4 Organization

Responsabilité :

gestion des organisations ;

appartenance des utilisateurs aux organisations ;

rôles et permissions ;

autorisation liée aux memberships.

Concepts principaux :

Organization

Membership

Role

Permission

Le fait qu’un utilisateur possède un rôle de ServiceProvider ne signifie pas que le rôle lui-même crée ou possède un ServiceProvider.

2.5 Catalog

Responsabilité :

définition des services proposés ;

gestion des domaines fonctionnels d’une organisation.

Concepts principaux :

Service

Domaine

OrganisationDomaine

Le module Catalog ne possède pas la configuration opérationnelle spécifique d’un ServiceProvider pour une Service.

2.6 Provider

Responsabilité :

gestion du ServiceProvider ;

association entre un ServiceProvider et les Services qu’il fournit ;

configuration de la durée par défaut d’un service pour un provider ;

disponibilité hebdomadaire ;

périodes d’indisponibilité ;

statut opérationnel du provider.

Concepts principaux :

ServiceProvider

ServiceProviderService

WeeklyAvailability

UnavailabilityPeriod

La relation entre Membership et ServiceProvider respecte l’invariant :
Membership.organization == ServiceProvider.organization

2.7 Queue

Responsabilité :

gestion de la file opérationnelle ;

gestion du cycle de vie des tickets ;

ordre opérationnel ;

groupes Priority / Normal ;

FIFO dans chaque groupe ;

position ;

réinsertion ;

sélection du prochain candidat.

Concepts principaux :

Queue

Ticket

Queue et Ticket sont deux Aggregate Roots distincts avec des responsabilités complémentaires.
La Queue possède les invariants liés à l’ordre opérationnel.
Le Ticket possède les invariants liés à son cycle de vie.

2.8 Scheduling

Responsabilité :

gestion de la capacité de réservation ;

calcul de la capacité restante ;

gestion des appointments ;

activation quotidienne des appointments ;

interaction avec les disponibilités et indisponibilités du provider.

Dans le MVP, un appointment n’est pas représenté par une Entité Appointment séparée.
Un appointment est un Ticket avec :
entryType = APPOINTMENT status = SCHEDULED
Le module Scheduling orchestre les opérations de planification et utilise les capacités du module Queue pour les opérations liées à la file opérationnelle.

2.9 Notifications

Les notifications ne constituent pas un Aggregate ou un domaine métier autonome dans le MVP.
Le système utilise :
Application Service ↓ NotificationPort ↓ SMS Adapter ↓ External SMS Provider
Aucune Entité Notification ni NotificationRepository n’est nécessaire dans le MVP.
Les règles métier déterminant quand une notification doit être envoyée restent dans les Use Cases concernés.
Le mécanisme de notification est responsable de la manière dont le message est transmis au fournisseur externe.

2.10 Dépendances inter-modules

La règle générale est que les dépendances doivent rester explicites et éviter les dépendances circulaires.
La direction conceptuelle principale est :
Scheduling ↓ Queue
Scheduling peut demander au module Queue d’effectuer une opération liée à la file.
Queue ne doit pas dépendre de Scheduling pour déterminer les règles de disponibilité, de capacité ou de réservation.
Les interactions avec les systèmes externes passent par des Ports.
Exemple :
Application ↓ NotificationPort ↓ SMS Adapter ↓ External Provider
Les détails d’implémentation des adapters ne doivent pas remonter dans le Domain.

3. Layered Architecture

3.1 Vue générale

REST API ↓ Application ↓ Domain ↑ Infrastructure
L’architecture suit les principes de l’Hexagonal Architecture.
Les règles métier ne doivent pas dépendre directement de :

Spring Boot ;

JPA ;

Hibernate ;

PostgreSQL ;

un fournisseur SMS ;

une technologie externe d’authentification.

3.2 Domain Layer

Contient les règles métier et les modèles du domaine.

3.3 Application Layer

Orchestre les Use Cases et les interactions entre Aggregate Roots, modules et Ports.

3.4 Infrastructure Layer

Implémente les détails techniques :

JPA / Hibernate ;

PostgreSQL ;

SMS Adapter ;

sécurité technique ;

autres intégrations externes.

3.5 API Layer

Expose les Use Cases via REST.
L’API utilise des Request/Response DTOs et n’expose pas directement les Entités JPA.
3. Dependency Rules

3.1 Principe général

Chaque module doit dépendre uniquement des responsabilités dont il a réellement besoin.
Une dépendance entre modules doit être justifiée par un besoin métier ou technique explicite.
Les dépendances circulaires entre modules sont interdites.
Exemple interdit :
Module A → Module B Module B → Module A
Lorsqu’une interaction bidirectionnelle semble nécessaire, la responsabilité doit être réexaminée ou l’interaction doit être réalisée par l’intermédiaire d’un Use Case ou d’un Port approprié.

3.2 Dépendance vers le Domain d’un autre module

Un module ne doit pas accéder directement aux détails internes du Domain d’un autre module.
En particulier, un module ne doit pas :

modifier directement les Entités internes d’un autre module ;

contourner ses Aggregate Roots ;

accéder directement à ses repositories internes ;

appliquer lui-même les invariants appartenant à l’autre module.

L’interaction doit passer par une interface ou un contrat explicite.

3.3 Identity

Identity fournit les informations nécessaires à l’authentification.
Les autres modules peuvent utiliser l’identité authentifiée pour identifier l’acteur d’une opération.
Identity ne dépend pas des modules métier pour réaliser l’authentification.
Identity ↑ Modules métier
La connaissance de l’identité par les modules métier doit rester limitée au besoin réel de l’opération.

3.4 Organization

Organization fournit les informations relatives :

à l’organisation ;

aux memberships ;

aux rôles ;

aux permissions.

Les modules métier peuvent utiliser les contrats d’autorisation nécessaires à leurs opérations.
Organization ne doit pas connaître les détails internes de Queue, Scheduling ou Catalog.

3.5 Catalog → Provider

Provider utilise les Services définis par Catalog.
La configuration :
ServiceProvider + Service
est portée par ServiceProviderService.
Le module Provider peut donc référencer un Service du Catalog selon un contrat explicite.
Catalog ne dépend pas de Provider pour définir un Service.
Provider → Catalog

3.6 Provider → Scheduling

Scheduling utilise les informations fournies par Provider concernant :

la disponibilité ;

les périodes d’indisponibilité ;

la durée par défaut ;

le provider concerné.

Provider ne dépend pas de Scheduling pour définir sa configuration de disponibilité.
Scheduling → Provider

3.7 Scheduling → Queue

Scheduling peut demander à Queue d’effectuer les opérations nécessaires sur les Tickets et la file opérationnelle.
Exemples :

créer un Ticket SCHEDULED lors d’une réservation ;

activer les appointments du jour ;

insérer les Tickets activés dans l’ordre opérationnel.

Scheduling ne doit pas implémenter lui-même les règles de :

FIFO ;

priorité ;

position ;

réinsertion ;

sélection du prochain candidat.

Ces règles appartiennent au module Queue.
Scheduling → Queue

3.8 Queue → Provider

Queue a besoin de certaines informations du Provider pour effectuer ses opérations métier, notamment les informations nécessaires à l’estimation de durée.
La dépendance doit rester limitée au contrat nécessaire.
Queue ne doit pas connaître :

les mécanismes internes d’authentification ;

les détails de configuration REST ;

les détails JPA du Provider.

3.9 Queue → Catalog

Queue utilise le Service associé à un Ticket afin de déterminer les informations métier nécessaires au traitement du ticket et au calcul de l’ETA.
La dépendance doit rester limitée aux données et contrats nécessaires.
Queue ne doit pas modifier directement les Services du Catalog.

3.10 Notification

Les modules métier ne dépendent pas directement d’un fournisseur SMS concret.
Ils dépendent du :
NotificationPort
L’implémentation concrète est fournie par Infrastructure.
Business Use Case ↓ NotificationPort ↓ SMS Adapter ↓ External SMS Provider
Aucun module métier ne doit connaître le SDK ou l’API du fournisseur SMS.

3.11 Infrastructure

Infrastructure peut dépendre des modules Domain et Application afin d’implémenter leurs Ports.
Le Domain et l’Application ne doivent pas dépendre d’une implémentation Infrastructure concrète.
Application ↑ Infrastructure

3.12 REST API

L’API REST dépend de l’Application Layer.
Elle ne doit pas contenir les règles métier principales.
La direction attendue est :
REST Controller ↓ Application Use Case ↓ Domain ↑ Infrastructure
Les Controllers ne doivent pas manipuler directement les repositories ou les Entités JPA pour réaliser les Use Cases.

3.13 Règle de non-contournement

Aucun module ne doit contourner les responsabilités d’un autre module pour accéder directement à sa persistence.
Interdit :
Scheduling ↓ TicketJpaRepository
si ce repository appartient au module Queue.
Préféré :
Scheduling ↓ Queue Application Contract / Port ↓ Queue

3.14 Règle de dépendance minimale

Lorsqu’un module a seulement besoin d’une information ou d’une opération provenant d’un autre module, il ne doit pas importer toute l’implémentation de ce module.
La dépendance doit être aussi petite que possible.
Objectif :
Faible couplage + Responsabilités claires + Invariants protégés + Évolution indépendante des détails internes

3.15 Règle de réévaluation

Une nouvelle dépendance inter-module doit être considérée comme un signal d’architecture.
Avant de l’ajouter, vérifier :

Quelle responsabilité nécessite cette dépendance ?

Quel module possède réellement cette responsabilité ?

Est-ce une donnée ou une opération ?

Peut-elle être exposée par un contrat plus petit ?

Cette dépendance crée-t-elle une dépendance circulaire ?

La responsabilité est-elle placée dans le mauvais module ?

Une dépendance ne doit pas être ajoutée uniquement pour faciliter l’implémentation.
4. Aggregate Boundaries

4.1 Principes

Un Aggregate regroupe les objets métier qui doivent être cohérents dans une même frontière transactionnelle.
Chaque Aggregate possède un Aggregate Root.
Les règles suivantes s’appliquent :

les modifications métier passent par l’Aggregate Root ;

les invariants appartenant à un Aggregate sont protégés par son Root ;

un Aggregate ne doit pas charger inutilement tout le graphe du domaine ;

les relations entre Aggregates doivent rester explicites ;

un Aggregate ne doit pas être créé uniquement pour refléter une relation SQL.

Le découpage des Aggregates est déterminé par les invariants métier et les frontières transactionnelles, et non uniquement par les relations UML.

4.2 User Aggregate

Aggregate Root

User
Responsabilité :

identité utilisateur ;

données nécessaires à l’authentification ;

informations de contact liées à l’identité.

Le User Aggregate ne contient pas :

Membership ;

Organization ;

Ticket ;

Queue.

Ces concepts possèdent leurs propres responsabilités.

4.3 Customer Aggregate

Aggregate Root

Customer
Responsabilité :

profil métier du Customer.

Customer est lié à un User, mais cette relation ne signifie pas que tous les concepts Customer doivent être chargés ou modifiés à travers le User Aggregate.
Les Tickets référencés par un Customer ne font pas partie du Customer Aggregate.
Customer │ └── référence User Customer │ └── Ticket : référence externe
Le Customer ne possède donc pas la file d’attente.

4.4 Organization Aggregate

Aggregate Root

Organization
Responsabilité :

identité de l’organisation ;

invariants propres à l’organisation.

Les Memberships sont liés à l’Organization, mais leur comportement d’autorisation doit rester contrôlé par leurs propres règles.
Les détails opérationnels de Queue, Provider ou Scheduling ne font pas partie de l’Organization Aggregate.

4.5 Membership Aggregate

Aggregate Root

Membership
Responsabilité :

appartenance d’un User à une Organization ;

rôles associés ;

permissions résultantes selon le modèle d’autorisation.

Invariant important :
Membership.organization = ServiceProvider.organization
lorsqu’un Membership possède un ServiceProvider associé.
Un rôle ServiceProvider ne crée pas automatiquement un objet ServiceProvider.

4.6 ServiceProvider Aggregate

Aggregate Root

ServiceProvider
Responsabilité :

identité métier du provider ;

état opérationnel ;

configuration directement liée au provider.

Le ServiceProvider Aggregate ne possède pas les Tickets de sa Queue.
Il ne contient pas non plus les Services du Catalog eux-mêmes.

4.7 ServiceProviderService Aggregate

ServiceProviderService représente l’association opérationnelle entre un ServiceProvider et un Service.
Il possède notamment :
defaultDurationMinutes
Cette configuration est importante pour :

la réservation ;

l’estimation de durée ;

l’ETA.

Elle ne doit pas être dupliquée dans Service ou Ticket.
Selon le niveau de comportement nécessaire, ServiceProviderService peut être traité comme un Aggregate Root propre ou comme une entité gérée dans les opérations du Provider.
La décision d’implémentation détaillée sera précisée lors de la conception des Entities et Repositories.

4.8 Service Aggregate

Aggregate Root

Service
Responsabilité :

définition d’un service du Catalog ;

nom ;

description ;

informations intrinsèques du service.

Le Service Aggregate ne contient pas :

les providers qui offrent ce service ;

les Tickets utilisant ce service ;

les durées historiques réelles.

4.9 Availability Configuration

Les informations suivantes appartiennent à la configuration temporelle du Provider :
WeeklyAvailability UnavailabilityPeriod
Elles sont utilisées par Scheduling pour déterminer la disponibilité effective.
Elles ne doivent pas être considérées comme faisant partie de Queue.
La frontière transactionnelle exacte sera déterminée lors de la conception du Provider Aggregate et des Use Cases de Scheduling.

4.10 Queue Aggregate

Aggregate Root

Queue
Responsabilité principale :

protéger les invariants liés à l’ordre opérationnel de la file.

La Queue possède notamment les règles concernant :

les groupes Priority / Normal ;

l’ordre FIFO ;

les positions ;

la réinsertion ;

la sélection du prochain candidat ;

l’insertion d’un Ticket activé ;

la conservation de l’ordre existant lors de l’activation matinale.

La Queue ne possède pas les règles complètes du cycle de vie du Ticket.

4.11 Ticket Aggregate

Aggregate Root

Ticket
Responsabilité principale :

protéger le cycle de vie d’un ticket.

Le Ticket possède notamment :

status ;

entryType ;

isPriority ;

position ;

reinsertionCount ;

notificationAttempts ;

timestamps métier ;

référence au Customer ;

référence au Service ;

référence à la Queue.

Les transitions d’état doivent respecter les règles définies dans DECISIONS.md.
Le Ticket ne doit pas devenir propriétaire de la logique globale de classement de la Queue.

4.12 Queue / Ticket Boundary

Queue et Ticket sont deux Aggregate Roots distincts.
Cette séparation existe parce que leurs invariants sont différents.

Queue protège :

Priority ordering FIFO Position Reinsertion Candidate selection

Ticket protège :

State transitions Lifecycle timestamps Notification attempts Reinsertion count Entry type
La relation :
Queue 1 ─── 0..* Ticket
ne signifie donc pas que tous les Tickets doivent être chargés dans la mémoire du Queue Aggregate.
Une Queue contenant plusieurs dizaines ou centaines de Tickets ne doit pas nécessiter le chargement complet de tous les Tickets pour chaque opération.

4.13 Scheduling and Ticket

Dans le MVP, un Appointment est un Ticket :
entryType = APPOINTMENT status = SCHEDULED
Scheduling ne possède donc pas un Aggregate Appointment séparé.
Scheduling orchestre les règles de réservation et d’activation, tandis que Ticket protège son propre lifecycle.
Exemple :
BookAppointment ↓ Scheduling ↓ Create SCHEDULED Ticket ↓ Ticket
Lors de l’activation :
Scheduling ↓ Ticket SCHEDULED → WAITING ↓ Queue inserts Ticket
La modification du statut appartient au Ticket.
La modification de l’ordre opérationnel appartient à la Queue.

4.14 Cross-Aggregate Transactions

Une opération métier peut nécessiter plusieurs Aggregates.
Cela ne signifie pas que ces Aggregates doivent être fusionnés.
Exemple :
FinishAndStartNext
peut modifier :
Current Ticket Queue Next Ticket
Ces modifications peuvent être exécutées dans une même transaction applicative lorsque le Use Case l’exige.
La transaction ne transforme pas les trois objets en un seul Aggregate.

4.15 Règle de référence entre Aggregates

Les Aggregates doivent référencer d’autres Aggregates par identifiant ou contrat approprié plutôt que de créer de grands graphes d’objets métier.
Objectif :
Aggregate ↓ Reference ↓ Other Aggregate
plutôt que :
Aggregate ↓ Full object graph ↓ Multiple Aggregates ↓ Entire domain graph
Cette règle réduit :

le couplage ;

le coût de chargement ;

les risques de modifications involontaires ;

les problèmes liés à JPA.

4.16 Principe de décision

Le fait qu’une classe soit liée à une autre dans le diagramme UML ne suffit pas à justifier son inclusion dans le même Aggregate.
La question principale est :

Quels objets doivent rester cohérents ensemble pour protéger une même invariant métier dans une même frontière transactionnelle ?

Si deux concepts ont des invariants différents et peuvent évoluer séparément, ils doivent rester dans des Aggregates séparés.
4.7 ServiceProviderService

ServiceProviderService est une Entity appartenant au ServiceProvider Aggregate.
Elle représente la configuration opérationnelle d’un Service pour un ServiceProvider donné.
Elle possède notamment :
defaultDurationMinutes
Son identité métier dépend de l’association :
ServiceProvider + Service
Elle n’est pas considérée comme un Aggregate Root indépendant dans le MVP.
Le ServiceProvider Aggregate protège donc la cohérence de la configuration :
ServiceProvider ├── ServiceProviderService ├── WeeklyAvailability └── UnavailabilityPeriod
Le Service lui-même reste un Aggregate Root du module Catalog et n’est pas inclus dans le ServiceProvider Aggregate.
ServiceProviderService référence donc un Service externe au Aggregate plutôt que de le posséder.
Cette séparation permet de distinguer :

les informations intrinsèques d’un Service ;

la configuration de ce Service pour un Provider particulier.

Exemple :
Service Haircut ServiceProvider A Haircut → defaultDurationMinutes = 30 ServiceProvider B Haircut → defaultDurationMinutes = 45
La durée par défaut est donc une propriété de la relation opérationnelle entre le Provider et le Service, et non une propriété intrinsèque du Service.
5. Entities & Value Objects

5.1 Principes

Une Entity possède une identité propre et une continuité dans le temps.
Un Value Object représente une valeur définie par ses attributs et ne possède pas d’identité métier propre.
Le choix entre Entity et Value Object doit être justifié par le comportement et les invariants métier, et non uniquement par la structure des données.
Aucune classe ne doit être introduite comme Value Object sans avantage métier ou technique concret.

5.2 Entities principales

Les principales Entities du MVP sont :

Identity

User

Customer

Organization

Organization

Membership

Role

Permission

Catalog

Service

Domaine

OrganisationDomaine

Provider

ServiceProvider

ServiceProviderService

WeeklyAvailability

UnavailabilityPeriod

Queue

Queue

Ticket

Ces Entities possèdent une identité persistante et participent aux relations métier définies dans le modèle du domaine.

5.3 User

User représente l’identité technique et fonctionnelle d’un utilisateur de la plateforme.
Attributs principaux :
id phone email passwordHash
L’identité du User est indépendante de ses rôles métier dans une organisation.
Un même User peut avoir plusieurs Memberships.

5.4 Customer

Customer représente le profil métier d’un utilisateur agissant comme client.
Un Customer est lié à un User.
Cette relation ne signifie pas que Customer et User doivent être fusionnés en une seule Entity.
Un User peut exister sans être Customer.
Les Tickets d’un Customer ne font pas partie de son Aggregate.

5.5 Organization

Organization représente une organisation utilisant la plateforme.
Attributs principaux :
name
L’Organization ne possède pas les détails opérationnels des Providers ou des Queues.

5.6 Membership

Membership représente l’appartenance d’un User à une Organization.
Elle porte notamment les rôles permettant de déterminer les permissions applicables.
Un Membership appartient à une seule Organization et référence un User.
Le même User peut posséder plusieurs Memberships dans différentes Organizations.

5.7 Role et Permission

Role et Permission représentent le modèle d’autorisation.
Ils sont utilisés pour déterminer les opérations qu’un Membership peut effectuer.
Le rôle est une information d’autorisation.
Il ne constitue pas une preuve qu’un utilisateur possède automatiquement un profil métier particulier.
Par exemple :
Role = ServiceProvider
ne crée pas automatiquement un ServiceProvider.

5.8 Service

Service représente un service disponible dans le Catalog.
Attributs principaux :
name description
La durée par défaut d’un service pour un provider n’est pas stockée dans Service.
Elle est portée par ServiceProviderService.

5.9 ServiceProvider

ServiceProvider représente l’acteur métier qui fournit effectivement des services.
Il est associé à un Membership et à une Organization.
Le ServiceProvider Aggregate contient les configurations opérationnelles nécessaires au provider, conformément à la section 4.

5.10 ServiceProviderService

ServiceProviderService est une Entity appartenant au ServiceProvider Aggregate.
Elle représente la configuration d’un Service pour un Provider donné.
Attribut principal :
defaultDurationMinutes
Elle référence un Service appartenant au Catalog.
Elle ne constitue pas un Aggregate Root indépendant.

5.11 WeeklyAvailability

WeeklyAvailability représente une plage de disponibilité récurrente du Provider.
Attributs principaux :
dayOfWeek startTime endTime
Elle est rattachée au ServiceProvider Aggregate.
La disponibilité effective est calculée en tenant compte des UnavailabilityPeriod.

5.12 UnavailabilityPeriod

UnavailabilityPeriod représente une période pendant laquelle le Provider est indisponible.
Attributs principaux :
startDateTime endDateTime reason
Elle appartient au ServiceProvider Aggregate.
Les règles relatives aux périodes rétroactives sont définies dans DECISIONS.md.

5.13 Queue

Queue représente la file opérationnelle permanente d’un ServiceProvider.
Elle possède sa propre identité.
Une Queue est associée à un seul ServiceProvider dans le MVP.
Elle protège les invariants d’ordre de la file.

5.14 Ticket

Ticket représente l’entrée d’un Customer dans le système de traitement.
Attributs métier principaux :
id entryType status isPriority position reinsertionCount notificationAttempts createdAt scheduledAt notifiedAt confirmedAt inServiceAt completedAt
Le Ticket possède son propre cycle de vie.
Les règles de transition sont définies dans DECISIONS.md et représentées dans le diagramme UML d’état.

5.15 Value Objects

Le MVP doit utiliser les Value Objects uniquement lorsqu’ils apportent une vraie valeur métier ou technique.
Les candidats identifiés sont :

PhoneNumber

TimeRange

DateTimeRange

D’autres Value Objects pourront être introduits uniquement lorsqu’un besoin concret apparaîtra.

5.16 PhoneNumber

PhoneNumber représente un numéro de téléphone normalisé.
Il peut être utilisé par le modèle d’identité et pour les communications SMS.
Ses responsabilités potentielles comprennent :

validation du format ;

normalisation ;

comparaison par valeur.

Le numéro ne doit pas être traité comme une simple chaîne arbitraire lorsque les règles de normalisation deviennent nécessaires.

5.17 TimeRange

TimeRange représente une plage horaire :
startTime endTime
Il peut être utilisé pour représenter les plages de WeeklyAvailability.
Invariants :
startTime < endTime
Les règles supplémentaires concernant les plages traversant minuit doivent être définies explicitement si ce cas devient nécessaire.

5.18 DateTimeRange

DateTimeRange représente un intervalle temporel :
startDateTime endDateTime
Il peut être utilisé notamment pour les UnavailabilityPeriod.
Invariant minimal :
startDateTime < endDateTime
Il permet également de centraliser les règles de comparaison et de chevauchement des intervalles.

5.19 Enums et types contrôlés

Les valeurs possédant un ensemble fini et stable de possibilités doivent être représentées par des types contrôlés plutôt que par des chaînes arbitraires.
Exemples :
TicketStatus EntryType ProviderOperationalStatus DayOfWeek
Pour TicketStatus, les valeurs sont celles définies dans DECISIONS.md :
SCHEDULED WAITING NOTIFIED CONFIRMED DELAYED SKIPPED IN_SERVICE COMPLETED CANCELLED NO_SHOW
Aucune valeur supplémentaire ne doit être introduite sans décision métier correspondante.

5.20 Identifiants

Les identifiants persistants sont des identifiants techniques distincts de l’identité métier lorsque cela est nécessaire.
Pour Ticket :
id : Long
Le Ticket.id est généré par JPA/PostgreSQL.
Il peut être utilisé comme tie-breaker après createdAt lorsque deux Tickets doivent être départagés.
Il ne constitue pas un numéro public destiné aux utilisateurs et ne doit pas être considéré comme un mécanisme de sécurité.

5.21 Règle d’évolution

Un nouveau Value Object ou une nouvelle Entity ne doit être introduit que lorsqu’il existe au moins une justification concrète :

invariant métier propre ;

comportement propre ;

identité propre ;

réutilisation significative ;

réduction du couplage ;

simplification de la validation ;

nécessité technique clairement identifiée.

L’objectif est de conserver un modèle riche mais proportionné à la complexité réelle du MVP.
6. Repository Design

6.1 Principe

Les Repositories représentent les points d’accès à la persistance des Aggregate Roots.
Une Entity interne à un Aggregate ne possède pas automatiquement son propre Repository.
Le Repository ne doit pas contenir les règles métier principales.
Son rôle est de :

charger un Aggregate ;

sauvegarder un Aggregate ;

rechercher selon des critères nécessaires aux Use Cases ;

fournir les données nécessaires aux opérations métier.

La décision métier reste dans le Domain ou l’Application Layer selon sa nature.

6.2 Repositories des Aggregate Roots

Les Aggregate Roots identifiés dans le MVP disposent des Repositories nécessaires :
UserRepository CustomerRepository OrganizationRepository MembershipRepository ServiceRepository DomaineRepository ServiceProviderRepository QueueRepository TicketRepository
La liste pourra être ajustée si l’analyse détaillée des Use Cases montre qu’un Repository n’est pas nécessaire.

6.3 UserRepository

Responsabilité :

charger un User ;

rechercher un User à partir de son numéro de téléphone ;

sauvegarder un User.

Exemples conceptuels :
findById(userId) findByPhone(phone) save(user)
Le Repository ne réalise pas l’authentification OTP.

6.4 CustomerRepository

Responsabilité :

charger un Customer ;

rechercher un Customer associé à un User ;

sauvegarder un Customer.

Il ne contient pas les règles de gestion des Tickets.

6.5 OrganizationRepository

Responsabilité :

charger une Organization ;

sauvegarder une Organization ;

effectuer les recherches nécessaires aux Use Cases d’organisation.

Il ne gère pas directement les Queues ou les Tickets.

6.6 MembershipRepository

Responsabilité :

charger un Membership ;

rechercher l’appartenance d’un User à une Organization ;

sauvegarder les modifications du Membership.

Exemples conceptuels :
findByUserAndOrganization(userId, organizationId) findById(membershipId) save(membership)
Le Repository ne décide pas lui-même si une opération métier est autorisée.
L’autorisation est déterminée par les règles d’application et de sécurité.

6.7 ServiceRepository

Responsabilité :

charger un Service ;

rechercher les Services nécessaires au Catalog ;

sauvegarder les Services.

Le Repository ne gère pas les associations opérationnelles Provider/Service.

6.8 DomaineRepository

Responsabilité :

charger les Domaines ;

rechercher les Domaines nécessaires aux opérations du Catalog ;

sauvegarder les Domaines lorsque nécessaire.

6.9 ServiceProviderRepository

Responsabilité :

charger un ServiceProvider ;

rechercher un Provider selon les critères nécessaires ;

sauvegarder la configuration du Provider Aggregate.

Le chargement d’un ServiceProvider ne doit pas entraîner automatiquement le chargement de toute la Queue ou de tous les Tickets.

6.10 QueueRepository

QueueRepository permet de charger et sauvegarder le Queue Aggregate.
Il doit également fournir les opérations de lecture nécessaires à la gestion de l’ordre opérationnel.
Exemples conceptuels :
findById(queueId) findByServiceProvider(serviceProviderId) save(queue)
Les recherches de Tickets nécessaires à la sélection opérationnelle peuvent être réalisées par des mécanismes de persistence adaptés, sans charger inutilement toute la collection des Tickets dans le Queue Aggregate.

6.11 TicketRepository

TicketRepository permet de charger et sauvegarder le Ticket Aggregate.
Il fournit également les recherches nécessaires aux Use Cases.
Exemples conceptuels :
findById(ticketId) findActiveByQueue(queueId) findScheduledForDate(queueId, date) findTicketsAhead(queueId, ticketId)
Les méthodes exactes seront définies après la conception détaillée des Use Cases.
Les requêtes de lecture doivent être conçues selon le besoin réel du Use Case et ne doivent pas être ajoutées uniquement pour refléter les méthodes possibles du SQL.

6.12 Repository ≠ Business Service

Un Repository ne doit pas décider :
"Ce Ticket est-il prioritaire ?" "Peut-il être réinséré ?" "Qui est le prochain candidat ?" "Cette transition d'état est-elle autorisée ?"
Ces décisions appartiennent aux règles métier.
Le Repository fournit les données nécessaires à ces décisions.

6.13 Repository ≠ Controller

Un Controller REST ne doit pas appeler directement un Repository pour réaliser une opération métier.
Interdit :
REST Controller ↓ TicketRepository
Préféré :
REST Controller ↓ Application Use Case ↓ TicketRepository

6.14 Persistence Implementation

Les interfaces de Repository appartiennent au Domain/Application selon la responsabilité du contrat.
Leur implémentation technique appartient à Infrastructure.
Exemple conceptuel :
Domain/Application │ ▼ TicketRepository ▲ │ implements │ Infrastructure │ ▼ JpaTicketRepository
JPA/Hibernate ne doit donc pas imposer ses détails au modèle métier.

6.15 JPA Repository

Les interfaces Spring Data JPA peuvent être utilisées comme mécanisme technique d’implémentation.
Elles ne doivent pas être confondues avec les Repository Ports métier.
Exemple :
TicketRepository ↑ JpaTicketRepository ↓ Spring Data JPA
Le modèle métier ne dépend pas directement de Spring Data.

6.16 Native SQL

JPA/Hibernate est la stratégie principale de persistence.
Le SQL natif peut être utilisé sélectivement lorsqu’il apporte un avantage concret, notamment pour :

des requêtes de lecture complexes ;

des recherches de Tickets ordonnés ;

certaines opérations nécessitant un contrôle précis de la concurrence ;

des optimisations démontrées par le besoin réel.

L’utilisation de SQL natif ne doit pas devenir la stratégie de persistence par défaut.

6.17 Repository et concurrence

Les opérations sensibles à la concurrence doivent être conçues avec leur Use Case et leur transaction.
Exemples :

JoinQueue ;

BookAppointment ;

ActivateAppointments ;

StartService ;

FinishAndStartNext ;

CancelTicket.

Le Repository peut fournir les mécanismes de verrouillage nécessaires, mais la stratégie de concurrence est une responsabilité du Use Case et de la conception transactionnelle globale.
Les règles de concurrence détaillées sont définies dans DECISIONS.md et seront précisées dans la section dédiée aux Transactions et à la Concurrency.
7. Ports & Adapters

7.1 Principe

L’architecture utilise les principes de l’Hexagonal Architecture.
Le système distingue :

les Ports, qui représentent les contrats nécessaires à l’application ;

les Adapters, qui fournissent les implémentations techniques de ces contrats.

Le Domain ne doit pas dépendre d’une technologie externe.
L’Application Layer peut dépendre de Ports définis pour les besoins des Use Cases.
Les implémentations techniques appartiennent à Infrastructure.

7.2 Vue générale

┌─────────────────────┐ │ REST API │ │ Adapter IN │ └──────────┬──────────┘ ↓ ┌─────────────────────┐ │ Application │ │ Use Cases │ └──────────┬──────────┘ ↓ ┌─────────────────────┐ │ Domain │ └──────────┬──────────┘ ↑ ┌─────────────┴─────────────┐ │ │ ┌────────┴────────┐ ┌────────┴────────┐ │ Persistence │ │ Notification │ │ Adapter OUT │ │ Adapter OUT │ └────────┬────────┘ └────────┬────────┘ ↓ ↓ PostgreSQL SMS Provider

7.3 Inbound Ports

Les Inbound Ports représentent les opérations que l’application expose à ses clients ou à d’autres adapters entrants.
Dans le MVP, ils correspondent principalement aux Use Cases.
Exemples conceptuels :
JoinQueue BookAppointment CancelTicket StartNext FinishAndStartNext GetTicketETA
Les Controllers REST appellent les Inbound Ports plutôt que d’exécuter directement les règles métier.

7.4 Outbound Ports

Les Outbound Ports représentent les dépendances externes nécessaires aux Use Cases.
Exemples :
UserRepository CustomerRepository OrganizationRepository MembershipRepository ServiceRepository ServiceProviderRepository QueueRepository TicketRepository NotificationPort
D’autres Ports peuvent être introduits uniquement lorsqu’un besoin réel apparaît.

7.5 Persistence Adapter

L’Adapter de persistence implémente les Repository Ports.
Exemple :
TicketRepository ↑ │ JpaTicketRepository ↓ JPA / Hibernate ↓ PostgreSQL
Le modèle métier ne dépend pas de JpaRepository.
Les détails tels que :

annotations JPA ;

mappings ;

fetch strategies ;

transactions techniques ;

SQL ;

indexes ;

restent dans Infrastructure lorsque cela est approprié.

7.6 NotificationPort

Le système utilise NotificationPort pour abstraire l’envoi des SMS.
Contrat conceptuel :
NotificationPort │ └── send(...)
Le Use Case ne connaît pas :

le fournisseur SMS ;

son SDK ;

son API HTTP ;

ses credentials ;

ses détails techniques.

L’implémentation concrète appartient à Infrastructure.
NotificationPort ↑ SmsNotificationAdapter ↓ External SMS Provider

7.7 Authentication Adapter

L’authentification par téléphone + OTP implique également une séparation entre le besoin métier et les détails techniques.
Le système doit pouvoir remplacer le mécanisme concret de génération, stockage temporaire ou vérification des OTP sans modifier les règles métier principales.
Les détails techniques de tokenisation et de sécurité sont isolés dans Infrastructure / Security.
Le Domain ne doit pas dépendre directement de Spring Security.

7.8 External Service Rule

Tout service externe doit être encapsulé derrière un Port lorsqu’il constitue une dépendance de l’application.
Exemples :
SMS Authentication provider Future external services
L’objectif est de pouvoir remplacer l’Adapter sans modifier les Use Cases.

7.9 Transaction Boundary

Les transactions sont définies autour des Use Cases métier.
Un Adapter ne doit pas décider seul de la frontière transactionnelle globale.
Exemple :
FinishAndStartNext ↓ Application Use Case ↓ @Transactional ↓ Ticket + Queue + Next Ticket
La transaction doit couvrir les modifications nécessaires à l’opération métier.

7.10 External Calls and Transactions

Les appels externes ne doivent pas être effectués pendant qu’un verrou de base de données critique est maintenu.
Exemple à éviter :
BEGIN TRANSACTION ↓ LOCK Ticket ↓ Send SMS ↓ Wait for external provider ↓ COMMIT
Préféré :
Business Transaction ↓ Persist state ↓ Commit ↓ External notification
Lorsque la fiabilité entre la persistence et l’envoi externe devient un problème réel, un mécanisme plus avancé pourra être étudié.
Il n’est pas introduit dans le MVP sans besoin démontré.

7.11 Dependency Rule

La règle fondamentale est :
Domain ↑ Application ↑ Adapters
Les dépendances techniques ne doivent pas inverser cette direction.
Le Domain ne doit jamais importer directement :

Spring ;

JPA ;

Hibernate ;

PostgreSQL drivers ;

SMS SDK ;

Spring Security.

7.12 Adapter Isolation

Un Adapter peut évoluer sans modifier le contrat métier.
Exemple :
NotificationPort ↑ ├── SmsProviderAAdapter └── SmsProviderBAdapter
Le choix du fournisseur concret reste un détail d’Infrastructure.
Le même principe s’applique aux mécanismes de persistence ou aux autres intégrations externes.
8. Use Cases

8.1 Principe

Un Use Case représente une opération métier complète déclenchée par un acteur ou un mécanisme système.
Chaque Use Case doit définir clairement :

son acteur ou déclencheur ;

ses entrées ;

ses règles métier principales ;

les Aggregates concernés ;

sa frontière transactionnelle ;

son résultat.

Un Use Case ne doit pas devenir un simple wrapper technique autour d’un Repository.

8.2 Customer Use Cases

8.2.1 JoinQueue

Acteur : Customer
Objectif :
Permettre à un Customer d’entrer dans la file pour le jour courant.
Flux conceptuel :
Customer ↓ JoinQueue ↓ Validate service ↓ Create Ticket ↓ WAITING ↓ Queue assigns operational position
Règles principales :

le Customer doit être authentifié ;

le Service doit être disponible pour le ServiceProvider ;

le Ticket est créé avec entryType = WALK_IN ;

le Ticket commence avec status = WAITING ;

isPriority = false lors de la création ;

la Queue détermine la position opérationnelle ;

les règles de priorité et FIFO sont appliquées par la Queue.

La transaction doit protéger la création du Ticket et l’attribution de sa position contre les opérations concurrentes.

8.3 BookAppointment

Acteur : Customer
Objectif :
Créer un Ticket correspondant à un appointment futur.
Flux conceptuel :
Customer ↓ BookAppointment ↓ Check provider availability ↓ Check unavailability ↓ Calculate remaining capacity ↓ Create Ticket ↓ SCHEDULED
Règles principales :

l’appointment doit respecter la disponibilité effective ;

les périodes d’indisponibilité ont priorité sur la disponibilité hebdomadaire ;

la capacité restante doit être respectée ;

entryType = APPOINTMENT ;

status = SCHEDULED ;

isPriority = false lors de la création ;

aucun positionnement opérationnel n’est attribué tant que le Ticket reste SCHEDULED.

La création et la vérification de capacité doivent être protégées contre les réservations concurrentes.

8.4 ActivateAppointments

Déclencheur : Day Start Scheduler
Objectif :
Activer les appointments du jour lorsque le ServiceProvider est physiquement présent.
Flux :
Scheduler ↓ Check provider presence ↓ Find today's SCHEDULED tickets ↓ Sort createdAt ASC then Ticket.id ASC ↓ SCHEDULED → WAITING ↓ Queue inserts activated tickets
Règles principales :

si le Provider est absent, les appointments ne sont pas activés ;

les Tickets WAITING existants conservent leur position ;

les appointments activés sont ajoutés après les Tickets existants de leur groupe ;

isPriority détermine le groupe ;

l’entry type APPOINTMENT ne donne pas automatiquement la priorité ;

l’ordre des appointments est createdAt ASC, puis Ticket.id ASC.

L’activation et l’insertion opérationnelle doivent être cohérentes transactionnellement.

8.5 NotifyTicket

Déclencheur : Application Use Case
Objectif :
Notifier le Customer lorsque le Ticket atteint la phase de notification.
Flux :
Ticket WAITING ↓ Increment notificationAttempts ↓ NOTIFIED ↓ NotificationPort ↓ SMS Adapter
Règles principales :

le nombre maximal de tentatives est de 2 ;

une troisième notification est interdite ;

notificationAttempts est conservé pendant la journée ;

la notification ne décide pas de la position du Ticket ;

l’appel externe SMS ne doit pas être exécuté sous un verrou critique de base de données.

8.6 RespondToNotification

Acteur : Customer
Objectif :
Permettre au Customer de répondre à une notification.
Deux réponses métier principales sont possibles :
Confirm Delay

Confirm

NOTIFIED → CONFIRMED
Le Ticket est confirmé mais cette confirmation ne constitue pas une preuve de présence physique.

Delay

NOTIFIED → DELAYED ↓ Queue reinsertion ↓ WAITING
Règles :

reinsertionCount est incrémenté ;

la nouvelle position est calculée selon la règle de réinsertion validée ;

si la deuxième tentative a déjà été utilisée, aucune troisième notification ne doit être générée.

8.7 StartService

Acteur : ServiceProvider
Objectif :
Démarrer explicitement le service d’un Customer présent physiquement.
Transitions possibles :
WAITING → IN_SERVICE CONFIRMED → IN_SERVICE
Règles :

le Provider doit être physiquement présent ;

le Ticket doit être un candidat valide ;

la présence physique est vérifiée par le Provider ;

inServiceAt est défini au démarrage.

Pour le premier service de la journée, l’action explicite du Provider est obligatoire.
L’activation matinale des appointments ne démarre jamais automatiquement un service.

8.8 SkipTicket

Acteur : ServiceProvider ou Use Case de traitement
Objectif :
Marquer un Customer comme absent lorsque le service ne peut pas commencer.
Transitions principales :
WAITING → SKIPPED CONFIRMED → SKIPPED
Après une première notification, le Ticket peut être réinséré selon les règles définies.
Lorsque notificationAttempts = 2, aucune nouvelle notification n’est autorisée et le Ticket reste SKIPPED jusqu’à la clôture de journée si aucune autre règle applicable ne permet son traitement.

8.9 FinishService

Acteur : ServiceProvider
Objectif :
Terminer le service actuellement en cours.
Flux :
IN_SERVICE ↓ Set completedAt ↓ COMPLETED
La durée réelle du service est :
completedAt - inServiceAt
Cette durée peut être utilisée par le mécanisme d’estimation historique.

8.10 FinishAndStartNext

Acteur : ServiceProvider
Objectif :
Terminer le service courant et traiter le prochain candidat dans une seule opération métier cohérente.
Flux :
Current Ticket ↓ COMPLETED ↓ Queue selects next candidate ↓ Physical presence verification ↓ Next Ticket ↓ IN_SERVICE
Règles :

le Ticket courant doit être IN_SERVICE ;

completedAt est défini ;

le candidat suivant est sélectionné selon les règles de Queue ;

Priority est évaluée avant Normal ;

FIFO et position sont respectés ;

si le candidat est absent, il peut être SKIPPED puis réinséré lorsque les conditions le permettent ;

un Ticket ayant atteint la limite de notification ne reçoit pas de nouvelle notification.

La totalité de l’opération doit être traitée dans une transaction métier cohérente.

8.11 CancelTicket

Acteur : selon les permissions
Objectif :
Annuler un Ticket avant le début du service.
États pouvant être annulés :
SCHEDULED WAITING NOTIFIED CONFIRMED DELAYED SKIPPED
États ne pouvant pas être annulés :
IN_SERVICE COMPLETED NO_SHOW CANCELLED
Règles :

Customer peut annuler son propre Ticket avant IN_SERVICE ;

ServiceProvider peut annuler les Tickets de sa Queue avant IN_SERVICE ;

les autres acteurs dépendent de leurs permissions ;

CANCELLED est terminal ;

un Ticket annulé ne peut plus être candidat ;

un Ticket annulé ne doit pas être réinséré ;

aucune notification supplémentaire ne doit être déclenchée.

8.12 EndOfDay

Déclencheur : Scheduler
Objectif :
Clôturer les Tickets encore actifs à la fin de la journée.
Transitions principales :
WAITING → NO_SHOW NOTIFIED → NO_SHOW CONFIRMED → NO_SHOW DELAYED → NO_SHOW SKIPPED → NO_SHOW
Les états terminaux existants ne sont pas modifiés.

8.13 GetTicketETA

Acteur : Customer
Objectif :
Calculer l’ETA actuel d’un Ticket.
Flux :
Customer ↓ GetTicketETA ↓ Queue ↓ Tickets ahead in operational order ↓ Duration estimation ↓ ETA
Règles :

le Ticket cible n’est pas inclus dans les tickets devant lui ;

le temps restant du Ticket IN_SERVICE est pris en compte ;

si aucun Ticket n’est IN_SERVICE, le temps courant restant est zéro ;

les estimations sont calculées par ServiceProvider + Service ;

moins de 3 services réels terminés → defaultDurationMinutes ;

3 services réels terminés ou plus → moyenne des durées réelles ;

la durée réelle est completedAt - inServiceAt ;

l’ETA est dérivé dynamiquement ;

estimatedWaitMinutes n’est pas persisté sur Ticket.

L’ETA ne modifie :

la position ;

la priorité ;

l’ordre ;

le statut ;

les notifications.

8.14 Provider Operational Status

Les Use Cases liés au Provider doivent distinguer :
Planning vs Operational status
Les états opérationnels principaux sont :
ACTIVE ABSENT CLOSED
Les règles exactes de changement d’état sont celles définies dans DECISIONS.md.

8.15 Authorization

L’autorisation est appliquée au niveau Application / Security selon l’acteur et l’opération.
Exemples :
Customer → Cancel own Ticket ServiceProvider → Start service → Cancel own Queue Ticket → Manage Queue operations Authorized Member → Operations allowed by Role / Permission
Le Domain ne doit pas dépendre directement de Spring Security.
Les Use Cases reçoivent l’identité et le contexte d’autorisation nécessaires pour appliquer les règles correspondantes.

8.16 Use Case Naming Rule

Les Use Cases doivent représenter une intention métier claire.
Préféré :
JoinQueue BookAppointment CancelTicket StartService FinishAndStartNext GetTicketETA
À éviter :
CreateTicket UpdateTicket SaveTicket ProcessTicket
lorsqu’ils ne représentent pas une véritable intention métier.
Un Use Case doit exprimer ce que l’acteur veut accomplir, et non simplement l’opération CRUD effectuée sur une Entity.
9. Transactions & Concurrency

9.1 Principes généraux

Les opérations métier critiques sont exécutées dans des frontières transactionnelles explicites.
Une transaction doit garantir que les modifications liées à une même opération métier sont validées ou annulées ensemble.
Les transactions sont définies au niveau des Use Cases / Application Services, et non au niveau des contrôleurs REST ou des méthodes techniques isolées.
Le système utilise PostgreSQL avec le niveau d’isolation :
READ COMMITTED
qui correspond au comportement par défaut de PostgreSQL.
Le niveau d’isolation n'est pas augmenté globalement sans justification métier.

9.2 Pourquoi la concurrence est critique

Plusieurs utilisateurs ou processus peuvent agir simultanément sur une même file.
Exemples :

deux Customers rejoignent la même Queue presque simultanément ;

deux Customers réservent une capacité restante presque simultanément ;

le Scheduler active des appointments pendant qu'une autre opération modifie la Queue ;

deux actions tentent de sélectionner le même prochain Ticket ;

un Customer annule son Ticket pendant qu'un Provider tente de le sélectionner ;

plusieurs opérations modifient simultanément l'ordre d'une même Queue.

Le système doit empêcher les incohérences résultant de ces accès concurrents.

9.3 Pessimistic Locking

Le pessimistic locking est utilisé lorsque plusieurs transactions peuvent modifier simultanément une même ressource métier critique et qu'une sérialisation explicite est nécessaire.
Il est notamment applicable aux opérations :

JoinQueue ;

BookAppointment ;

ActivateAppointments ;

FinishAndStartNext ;

CancelTicket lorsque l'ordre de Queue est affecté ;

autres opérations modifiant simultanément l'ordre ou la sélection d'une Queue.

Le verrouillage doit rester limité à la durée nécessaire de l'opération métier.
Il ne doit pas être utilisé comme mécanisme global sur toutes les lectures.

9.4 JoinQueue

JoinQueue est exécuté dans une transaction.
L'opération doit garantir que deux Customers ne reçoivent pas simultanément une position incohérente dans le même groupe de priorité.
Le processus conceptuel est :

démarrer la transaction ;

verrouiller la ressource de Queue nécessaire à la modification de l'ordre ;

vérifier les règles métier ;

créer le Ticket ;

déterminer sa position opérationnelle ;

enregistrer le Ticket ;

valider la transaction.

Les opérations concurrentes sur la même Queue sont ainsi sérialisées lorsque cela est nécessaire.

9.5 BookAppointment

BookAppointment est exécuté dans une transaction.
Le calcul de capacité restante et la création du Ticket SCHEDULED doivent être protégés contre les réservations concurrentes.
Le principe est :

démarrer la transaction ;

verrouiller la ressource de planification nécessaire ;

recalculer la capacité disponible à partir de l'état actuel ;

vérifier les périodes d'indisponibilité ;

créer le Ticket SCHEDULED ;

valider la transaction.

La capacité ne doit jamais être calculée uniquement à partir d'une valeur précédemment lue par une autre transaction.

9.6 ActivateAppointments

ActivateAppointments est une opération transactionnelle.
Les Tickets SCHEDULED concernés sont traités avec une protection contre les modifications concurrentes.
Le processus doit notamment :

sélectionner les appointments concernés ;

empêcher leur activation concurrente ;

respecter l'ordre createdAt ASC, puis Ticket.id ASC ;

transformer SCHEDULED → WAITING ;

demander à la Queue leur insertion dans le groupe approprié ;

préserver les positions des Tickets WAITING déjà présents.

Les opérations de Queue et les Tickets concernés doivent utiliser une stratégie de verrouillage cohérente.

9.7 FinishAndStartNext

FinishAndStartNext constitue une seule opération métier cohérente.
La transaction doit protéger simultanément :

le Ticket actuellement IN_SERVICE ;

la sélection du prochain candidat ;

la transition du prochain Ticket vers IN_SERVICE.

Le processus conceptuel est :

verrouiller la Queue nécessaire à la sélection ;

verrouiller le Ticket courant ;

vérifier que le Ticket courant est toujours IN_SERVICE ;

le passer à COMPLETED ;

sélectionner le prochain candidat selon les règles de Queue ;

verrouiller et revérifier le candidat ;

vérifier la présence physique ;

passer le candidat à IN_SERVICE, ou appliquer la règle SKIPPED / réinsertion ;

valider la transaction.

Deux opérations concurrentes ne doivent pas pouvoir sélectionner le même Ticket comme prochain candidat.

9.8 CancelTicket

CancelTicket est exécuté dans une transaction.
Le Ticket concerné doit être relu et vérifié dans la transaction avant sa modification.
Le système doit notamment revérifier :

le statut actuel ;

l'identité de l'acteur ;

ses permissions ;

la possibilité de cancellation selon l'état courant.

Si la cancellation modifie l'ordre opérationnel de la Queue, les ressources concernées doivent être verrouillées selon la même stratégie que les autres opérations de Queue.
Une décision de cancellation basée uniquement sur un état lu avant la transaction n'est pas considérée comme suffisante.

9.9 Optimistic Locking

L'optimistic locking peut être utilisé pour les entités qui peuvent être modifiées concurrentiellement sans nécessiter la sérialisation complète de la Queue.
Il pourra notamment être utilisé sur des agrégats ou configurations dont les modifications concurrentes doivent être détectées plutôt que sérialisées.
La décision d'ajouter un champ de version, par exemple avec JPA @Version, sera prise lors de la conception détaillée des mappings.
L'optimistic locking ne remplace pas le pessimistic locking nécessaire aux opérations critiques d'ordre et de sélection de Queue.

9.10 Database Constraints

Les règles métier sont protégées principalement par le Domain et les Use Cases.
Les contraintes PostgreSQL constituent une seconde ligne de défense.
Elles peuvent notamment garantir des invariants structurels tels que :

unicité d'une Queue pour un ServiceProvider ;

unicité d'une association ServiceProviderService pour un couple Provider / Service ;

contraintes de nullité ;

contraintes d'intégrité référentielle ;

contraintes d'unicité nécessaires à la cohérence des données.

Une contrainte SQL ne remplace pas une règle métier.
Inversement, une règle métier critique ne doit pas dépendre uniquement d'un contrôle effectué dans le code sans protection contre les accès concurrents.

9.11 Ordre des Verrous

Les opérations qui verrouillent plusieurs ressources doivent suivre autant que possible un ordre cohérent.
Pour les opérations de Queue, la stratégie de référence est :
Queue → Ticket(s)
Cela réduit le risque de deadlock lorsque plusieurs transactions travaillent simultanément sur les mêmes ressources.
Toute nouvelle opération nécessitant plusieurs verrous doit vérifier son ordre de verrouillage par rapport aux opérations existantes.

9.12 Deadlocks et Retry

Un deadlock ou une erreur transactionnelle temporaire liée à la concurrence peut être retenté automatiquement lorsque l'opération est techniquement sûre à rejouer.
Le retry doit être :

limité ;

réservé aux erreurs transitoires ;

effectué au niveau transactionnel ;

séparé des erreurs métier.

Une erreur telle que :

Ticket déjà annulé ;

capacité insuffisante ;

Customer non autorisé ;

Provider absent ;

n'est pas une erreur transitoire et ne doit pas déclencher un retry automatique.
Les appels externes, notamment SMS, ne sont pas rejoués simplement parce qu'une transaction de base de données est retentée.

9.13 External Calls et Locks

Aucun appel externe ne doit être effectué pendant qu'un verrou critique de base de données est maintenu.
En particulier :
Database Lock → SMS Provider
est interdit.
Le principe recherché est :
Transaction DB → Commit → External Call
Cela évite de maintenir des verrous pendant la durée imprévisible d'un appel réseau.
Les mécanismes plus avancés comme Outbox Pattern ou Event-driven notification ne sont pas introduits dans le MVP sans justification technique supplémentaire.

9.14 ETA

GetTicketETA est une opération de lecture dérivée.
Elle ne modifie pas :

la Queue ;

le Ticket ;

la position ;

la priorité ;

le statut ;

les notifications.

Elle ne nécessite donc pas de verrouillage pessimiste de la Queue.
Une légère différence entre l'ETA calculée et l'état réel quelques instants plus tard est acceptable, puisque l'ETA est une estimation dynamique et non une réservation de position.

9.15 EndOfDay

EndOfDay est exécuté dans une transaction adaptée au périmètre du traitement.
Les Tickets encore actifs dans les états concernés sont revérifiés avant leur passage vers NO_SHOW.
Une opération concurrente ne doit pas pouvoir transformer un Ticket déjà passé dans un état terminal en un autre état incompatible.

9.16 Règle de synthèse

La stratégie de concurrence du MVP repose sur quatre niveaux :

Transaction métier au niveau des Use Cases ;

Pessimistic locking pour les opérations critiques de Queue et de capacité ;

Optimistic locking lorsque la détection de conflit est plus appropriée que la sérialisation ;

Database constraints comme défense supplémentaire.

Le système privilégie une stratégie simple et explicable avant toute optimisation prématurée.
Toute optimisation future de la concurrence devra démontrer :

le problème réel rencontré ;

la limitation de la stratégie actuelle ;

le gain attendu ;

l'absence de violation des invariants métier.
10. Persistence & JPA / PostgreSQL Mapping

10.1 Principes généraux

La persistance du système repose sur :

PostgreSQL comme base de données relationnelle ;

JPA / Hibernate comme technologie principale de persistence ;

Spring Data JPA pour l'implémentation des Repository Adapters ;

SQL natif uniquement lorsqu'il apporte une valeur technique ou fonctionnelle justifiée.

JPA est considéré comme un détail d'Infrastructure.
Le Domain et les Application Services ne doivent pas dépendre directement de JPA ou d'Hibernate.

10.2 Séparation Domain / Persistence

Les modèles métier ne doivent pas être conçus uniquement en fonction des contraintes de JPA.
Le système conserve la séparation suivante :
Domain Model ↓ Repository Port ↓ JPA Repository Adapter ↓ Hibernate ↓ PostgreSQL
Le Domain définit les besoins de persistance à travers des Ports.
L'Infrastructure fournit les implémentations concrètes.
Cette séparation permet notamment de remplacer ou modifier la technologie de persistance sans modifier les règles métier.

10.3 Mapping des Aggregate Roots

Les Aggregate Roots principaux sont persistés comme des entités JPA.
Correspondance conceptuelle :
Aggregate RootTable principaleUserusersCustomercustomersOrganizationorganizationsMembershipmembershipsServiceservicesDomainedomainesServiceProviderservice_providersQueuequeuesTickettickets
Les noms physiques définitifs des tables peuvent être adaptés aux conventions PostgreSQL/JPA lors de l'implémentation.

10.4 ServiceProvider Aggregate

Le ServiceProvider Aggregate regroupe plusieurs Entities :
ServiceProvider ├── ServiceProviderService ├── WeeklyAvailability └── UnavailabilityPeriod
Ces Entities sont persistées dans des tables séparées, tout en restant sous la responsabilité du même Aggregate Root.
La séparation physique en plusieurs tables ne transforme pas automatiquement ces Entities en Aggregate Roots indépendants.
ServiceProviderService ne possède donc pas de ServiceProviderServiceRepository.
Les modifications de ces données passent par les opérations appropriées du ServiceProvider Aggregate.

10.5 Queue et Ticket

Queue et Ticket sont deux Aggregate Roots distincts.
Ils peuvent être liés en base de données par une clé étrangère :
queues.id ↓ tickets.queue_id
Cette relation SQL ne signifie pas que Ticket devient une Entity interne du Queue Aggregate.
Chaque Aggregate conserve sa propre frontière métier.
Les opérations qui doivent modifier les deux Aggregates sont orchestrées par un Application Service dans une même transaction lorsque cela est nécessaire.

10.6 Ticket Mapping

Le Ticket contient notamment :

id

entryType

status

isPriority

position

reinsertionCount

notificationAttempts

createdAt

scheduledAt

notifiedAt

confirmedAt

inServiceAt

completedAt

Le champ id est de type Long.
Il est généré par la base de données / JPA et sert d'identifiant interne.
Ticket.id peut être utilisé comme second critère de tri après createdAt lorsque deux Tickets possèdent le même timestamp.
Il n'est pas considéré comme un numéro public destiné au Customer.

10.7 Enum Mapping

Les valeurs métier telles que :

TicketStatus

EntryType

ProviderOperationalStatus

DayOfWeek

sont des concepts explicites du modèle.
Lorsqu'elles sont persistées, leur représentation doit rester stable et lisible.
Le mapping des enums utilisera de préférence leur représentation textuelle plutôt que leur ordinal numérique afin d'éviter qu'une modification de l'ordre des valeurs dans le code modifie leur signification en base.
Exemple conceptuel :
WAITING NOTIFIED CONFIRMED ...
plutôt que :
0 1 2 ...

10.8 Date et Time

Les dates et heures métier doivent utiliser des types adaptés aux besoins réels.
Les événements liés à l'exécution du système tels que :

createdAt

notifiedAt

confirmedAt

inServiceAt

completedAt

doivent représenter correctement un instant temporel.
Les horaires récurrents de WeeklyAvailability représentent au contraire des heures locales associées à un jour de semaine.
Il faut donc distinguer :
Instant temporel ≠ Heure locale
Cette distinction doit être conservée dans le mapping JPA.

10.9 Relations et Foreign Keys

Les relations entre entités sont protégées par des Foreign Keys PostgreSQL lorsque cela correspond au modèle relationnel.
Exemples :
memberships.organization_id service_providers.organization_id service_provider_services.service_provider_id service_provider_services.service_id queues.service_provider_id tickets.queue_id tickets.customer_id tickets.service_id
Les Foreign Keys garantissent l'intégrité référentielle.
Elles ne remplacent cependant pas les invariants métier définis dans le Domain.

10.10 Unicité Structurelle

Certaines règles structurelles doivent être protégées par des contraintes UNIQUE.
Exemples conceptuels :
queues.service_provider_id
Une Queue permanente par ServiceProvider.
Et :
(service_provider_id, service_id)
pour empêcher plusieurs configurations ServiceProviderService identiques pour le même couple Provider / Service.
Ces contraintes constituent une protection supplémentaire contre les incohérences introduites par des accès concurrents ou des erreurs d'implémentation.

10.11 Lazy Loading

Les relations JPA ne doivent pas être chargées automatiquement sans nécessité.
Le système privilégie le chargement explicite des données nécessaires à un Use Case.
L'objectif est d'éviter :

le chargement massif de graphes d'objets ;

les requêtes SQL imprévues ;

les problèmes de type N+1 ;

la dépendance du Domain au comportement de Hibernate.

Les besoins de lecture complexes peuvent utiliser des projections, des requêtes dédiées ou du SQL natif lorsque cela est justifié.

10.12 Repositories

Les Repository Ports appartiennent au Domain/Application selon leur responsabilité.
Leur implémentation JPA appartient à Infrastructure.
Exemple conceptuel :
TicketRepository ↑ JpaTicketRepository ↓ Spring Data JPA ↓ PostgreSQL
Le Controller REST ne doit jamais utiliser directement JpaRepository.
Le Controller appelle un Application Service / Use Case qui utilise les Repository Ports nécessaires.

10.13 Native SQL

JPA est la solution par défaut.
Le SQL natif peut être utilisé lorsque :

une requête de Queue est difficile ou inefficace avec JPQL ;

un verrouillage PostgreSQL précis est nécessaire ;

une lecture complexe nécessite une optimisation mesurable ;

une fonctionnalité SQL apporte une valeur réelle.

Le SQL natif ne doit pas devenir le mécanisme par défaut de persistence.
Chaque utilisation importante de SQL natif doit avoir une justification technique claire.

10.14 Transactions et Persistence Context

Les transactions sont contrôlées au niveau des Application Services.
Le Persistence Context JPA reste actif pendant la transaction correspondante.
Les modifications d'Entities sont persistées dans le cadre de cette transaction et sont validées lors du commit.
Les opérations critiques de concurrence doivent toutefois revalider les états nécessaires après obtention des verrous.
Le fait qu'une Entity soit déjà présente dans le Persistence Context ne doit pas être considéré comme une garantie contre une modification concurrente en base.

10.15 API et JPA Entities

Les Entities JPA ne sont jamais exposées directement par l'API REST.
Le système utilise :
HTTP Request ↓ Request DTO ↓ Application Service ↓ Domain ↓ Response DTO ↓ HTTP Response
Cela évite notamment :

de coupler le contrat API au modèle de persistence ;

d'exposer des champs internes ;

de provoquer du lazy loading depuis la couche REST ;

de transformer automatiquement les relations JPA en graphes JSON.

10.16 Règle générale

Le choix de JPA/PostgreSQL ne doit pas modifier les règles métier.
La priorité architecturale reste :
Business Rules ↓ Domain Model ↓ Application Use Cases ↓ Persistence Ports ↓ JPA / Hibernate ↓ PostgreSQL
Toute décision de mapping qui introduit une contrainte dans le Domain uniquement pour satisfaire JPA doit être considérée comme un signal architectural et réévaluée.
11. REST API & DTOs

11.1 Principes généraux

L'API du MVP utilise une architecture REST.
La couche REST constitue un Inbound Adapter de l'architecture Hexagonale.
Son rôle est de :

recevoir les requêtes HTTP ;

valider la structure des données entrantes ;

authentifier l'utilisateur ;

transmettre la demande au Use Case approprié ;

transformer le résultat du Use Case en Response DTO ;

retourner une réponse HTTP cohérente.

La couche REST ne contient pas les règles métier principales.

11.2 Request DTOs

Les données reçues par l'API sont représentées par des Request DTOs.
Un Request DTO représente l'intention de l'appelant et non une Entity JPA.
Exemples conceptuels :
JoinQueueRequest BookAppointmentRequest CancelTicketRequest RespondToNotificationRequest StartServiceRequest FinishAndStartNextRequest
Un Request DTO ne doit pas permettre au client de modifier directement des champs qu'il ne contrôle pas.
Par exemple, lors de JoinQueue, le Customer ne peut pas envoyer :
isPriority = true position = 1 notificationAttempts = 2 status = IN_SERVICE
Ces valeurs sont déterminées par les règles métier.

11.3 Response DTOs

Les réponses HTTP utilisent des Response DTOs.
Exemples :
TicketResponse QueueResponse AppointmentResponse TicketETAResponse ProviderStatusResponse
Le Response DTO contient uniquement les informations destinées au consommateur de l'API.
Les Entities JPA ne sont jamais retournées directement.

11.4 Use Case / Endpoint

Un Use Case n'est pas obligatoirement équivalent à une simple opération CRUD.
Les endpoints REST doivent représenter les intentions métier principales.
Exemples conceptuels :
POST /queues/{queueId}/tickets POST /appointments POST /tickets/{ticketId}/cancel POST /tickets/{ticketId}/notification-response POST /provider/tickets/{ticketId}/start POST /provider/tickets/finish-and-start-next GET /tickets/{ticketId}/eta
Les URLs exactes pourront être affinées pendant l'implémentation de l'API.

11.5 JoinQueue

Le Customer appelle le Use Case JoinQueue.
Le Request DTO contient uniquement les données nécessaires à l'opération.
Conceptuellement :
JoinQueueRequest └── serviceId
Le système détermine lui-même :

le Customer authentifié ;

l'Organization concernée ;

le Queue approprié ;

entryType = WALK_IN ;

status = WAITING ;

isPriority = false ;

la position opérationnelle ;

createdAt.

Le client ne contrôle donc pas directement l'état interne du Ticket.

11.6 BookAppointment

Le Request DTO de réservation contient les informations nécessaires à la demande de réservation.
Conceptuellement :
BookAppointmentRequest ├── serviceId └── scheduledAt
Le système détermine et vérifie :

le Customer authentifié ;

le Provider concerné ;

la disponibilité effective ;

les périodes d'indisponibilité ;

la capacité restante ;

entryType = APPOINTMENT ;

status = SCHEDULED ;

isPriority = false.

Aucune position opérationnelle n'est attribuée au Ticket SCHEDULED.

11.7 Cancellation

La cancellation utilise un Use Case dédié.
Conceptuellement :
CancelTicketRequest └── reason?
Le champ reason reste optionnel et ne doit être ajouté au modèle métier que s'il existe un besoin fonctionnel réel.
Le système détermine l'identité de l'acteur à partir du contexte authentifié.
Le client ne fournit donc pas un customerId arbitraire pour annuler le Ticket d'un autre Customer.

11.8 Notification Response

La réponse à une notification est représentée par une intention métier explicite.
Conceptuellement :
RespondToNotificationRequest └── response
où response peut représenter les choix métier autorisés :
CONFIRM DELAY
Le système contrôle que le Ticket est réellement dans l'état NOTIFIED avant d'appliquer la transition correspondante.

11.9 Provider Operations

Les opérations du Provider utilisent également des DTOs spécifiques.
Exemples :
StartServiceRequest FinishAndStartNextRequest
Le système détermine le Provider à partir de l'identité authentifiée et de son contexte organisationnel.
Le client ne doit pas pouvoir sélectionner arbitrairement un autre Provider simplement en envoyant son identifiant.

11.10 Validation

La validation des Request DTOs vérifie notamment :

présence des champs obligatoires ;

format ;

longueur ;

types ;

contraintes syntaxiques.

La validation DTO ne remplace pas la validation métier.
Exemple :
scheduledAt != null
est une validation structurelle.
Alors que :
scheduledAt respecte WeeklyAvailability
est une règle métier.
Cette seconde validation appartient au Use Case / Domain.

11.11 HTTP Status Codes

L'API utilise des codes HTTP cohérents avec le résultat de l'opération.
Exemples conceptuels :
200 OK
pour une opération réussie retournant un résultat.
201 Created
pour la création réussie d'une ressource lorsque cela correspond au contrat de l'endpoint.
400 Bad Request
pour une requête structurellement invalide.
401 Unauthorized
pour une absence d'authentification valide.
403 Forbidden
pour un utilisateur authentifié mais non autorisé.
404 Not Found
lorsqu'une ressource demandée n'est pas accessible ou n'existe pas selon le contrat de l'API.
409 Conflict
lorsqu'une opération est incompatible avec l'état courant ou qu'un conflit de concurrence / d'état doit être signalé au client.
422 Unprocessable Entity
peut être utilisé pour certaines violations métier lorsque le contrat API distingue explicitement ces erreurs des conflits d'état.
Le mapping définitif des exceptions métier vers les codes HTTP sera fixé lors de l'implémentation de l'API.

11.12 Error Response

Les erreurs API doivent avoir une structure cohérente.
Conceptuellement :
{ "code": "...", "message": "...", "details": [...] }
Le code représente une erreur technique ou métier stable destinée au client.
Le message est destiné à l'affichage ou au diagnostic.
Les détails sensibles tels que :

stack traces ;

requêtes SQL ;

exceptions internes ;

informations d'infrastructure ;

ne sont jamais exposés au client.

11.13 Pagination et Queries

Les endpoints de lecture susceptibles de retourner de nombreux éléments doivent prévoir une stratégie de pagination.
Les opérations métier critiques telles que :

sélection du prochain Ticket ;

activation des appointments ;

calcul de la capacité ;

détermination des Tickets précédents ;

ne doivent cependant pas être implémentées comme de simples endpoints paginés destinés à l'interface.
Elles utilisent des requêtes adaptées au Use Case.

11.14 API ne contrôle pas le Domain

La couche REST ne doit jamais :

modifier directement un Entity ;

calculer une position de Queue ;

décider qu'un Ticket est prioritaire ;

changer directement un TicketStatus ;

calculer une capacité métier ;

appeler directement un Repository pour réaliser une opération métier.

Le flux attendu est :
HTTP Request ↓ REST Controller ↓ Request DTO ↓ Application Use Case ↓ Domain / Aggregates ↓ Repository Ports ↓ Infrastructure

11.15 Règle générale

L'API doit rester un contrat stable entre le système et ses consommateurs.
Le modèle interne peut évoluer sans obliger l'API à exposer :

la structure des Aggregate Roots ;

les relations JPA ;

les tables PostgreSQL ;

les détails d'implémentation.

Les DTOs constituent donc une frontière explicite entre l'extérieur du système et son modèle interne.
12. Security & Authentication

12.1 Principes généraux

L'authentification du MVP repose sur :

le numéro de téléphone ;

un code OTP ;

un mécanisme de Token pour authentifier les requêtes API.

La sécurité technique est une responsabilité d'Infrastructure.
Les règles métier d'autorisation restent cependant exprimées au niveau des Use Cases et des concepts métier Membership, Role et Permission.
Le Domain ne dépend pas directement de Spring Security.

12.2 Identity

L'entité User représente l'identité technique de l'utilisateur.
Elle peut être associée à :

un Customer ;

zéro ou plusieurs Membership.

Un même User peut donc posséder différents contextes métier selon ses Memberships.
Le fait qu'un User possède une Membership ne signifie pas automatiquement qu'il possède un rôle de Provider.
Le lien avec ServiceProvider est explicite :
Membership └── 0..1 ServiceProvider

12.3 Phone + OTP

Le parcours d'authentification MVP utilise le téléphone.
Flux conceptuel :
Client ↓ Request OTP ↓ OTP Service ↓ SMS ↓ Client enters OTP ↓ Verify OTP ↓ Authenticated User ↓ Token
L'OTP est un mécanisme d'authentification et ne constitue pas une règle métier de Queue.
Les détails du fournisseur SMS restent derrière un Adapter.

12.4 OTP Security

Les codes OTP doivent être protégés contre les abus.
Le système doit notamment prévoir :

expiration courte du code ;

usage unique ;

limitation du nombre de tentatives ;

limitation des demandes répétées ;

stockage sécurisé de la valeur nécessaire à la vérification ;

invalidation après vérification réussie.

Les valeurs OTP ne doivent pas être exposées dans les logs applicatifs.
Les seuils numériques exacts seront définis lors de l'implémentation selon les besoins du MVP et les contraintes du fournisseur.

12.5 Token Authentication

Après une authentification réussie, le client reçoit un Token utilisé pour les appels API protégés.
Conceptuellement :
Authorization ↓ Authentication Adapter ↓ Authenticated User ↓ Application Use Case
Le Use Case ne doit pas faire confiance à un userId envoyé librement dans le Request Body pour déterminer l'identité de l'acteur.
L'identité authentifiée provient du contexte de sécurité.

12.6 Authentication vs Authorization

L'authentification répond à :

Qui est cet utilisateur ?

L'autorisation répond à :

Cet utilisateur a-t-il le droit d'effectuer cette opération ?

Ces deux responsabilités restent distinctes.
Exemple :
Authentication ↓ User = 42 Authorization ↓ Membership + Role + Permission ↓ Can perform operation?

12.7 Membership, Role et Permission

Les droits métier sont déterminés à partir des concepts :
Membership ↓ Role ↓ Permission
Une Membership appartient à une Organization.
Un User peut avoir plusieurs Memberships dans différentes Organizations.
Le contexte organisationnel doit donc être déterminé avant d'autoriser une opération métier qui dépend d'une Organization.

12.8 Provider Authorization

Un Provider est identifié par son ServiceProvider associé à une Membership.
Avant une opération telle que :

StartService ;

FinishAndStartNext ;

gestion de sa Queue ;

modification de sa disponibilité ;

le système doit vérifier que l'utilisateur authentifié possède le contexte Provider approprié et les permissions nécessaires.
Posséder simplement un compte User ne donne pas automatiquement ces permissions.

12.9 Customer Authorization

Le Customer peut agir sur ses propres Tickets selon les règles métier.
Exemple :
CancelTicket
Le système vérifie :
Authenticated User ↓ Customer ↓ Ticket.customer ↓ Ownership ↓ Allowed?
Un Customer ne peut pas annuler arbitrairement le Ticket d'un autre Customer.
Les permissions supplémentaires des membres de l'Organization sont traitées séparément.

12.10 Organization Context

Lorsqu'une opération dépend d'une Organization, le système doit vérifier que les objets utilisés appartiennent au même contexte organisationnel.
Exemple :
Membership.organization == ServiceProvider.organization
Cet invariant est également défini dans le modèle métier.
La sécurité technique ne doit donc pas être la seule protection.

12.11 Spring Security

Spring Security peut être utilisé dans l'Infrastructure pour :

traiter le Token ;

authentifier les requêtes ;

construire le contexte de sécurité ;

appliquer les mécanismes techniques de protection HTTP.

Cependant :
Domain ✕ Spring Security
Le Domain ne doit pas contenir de dépendance directe vers Spring Security.
L'Application Layer reçoit uniquement l'information nécessaire sur l'acteur authentifié à travers une abstraction appropriée.

12.12 Secrets et Configuration

Les éléments sensibles ne doivent jamais être hardcodés dans le code source.
Cela concerne notamment :

secrets de Token ;

credentials PostgreSQL ;

clés API SMS ;

secrets OTP ;

configurations sensibles.

Ils doivent être fournis par la configuration d'environnement ou un mécanisme sécurisé approprié.
Les fichiers de configuration versionnés ne doivent pas contenir de secrets réels.

12.13 Logs et Données Sensibles

Les logs applicatifs ne doivent pas exposer :

OTP ;

Tokens ;

mots de passe ou secrets ;

credentials ;

données personnelles inutiles.

Les identifiants techniques peuvent être loggés lorsqu'ils sont nécessaires au diagnostic, tout en évitant de transformer les logs en stockage de données personnelles.

12.14 Règle générale

La séparation retenue est :
Authentication ↓ Infrastructure / Security Authorization ↓ Application + Domain rules Identity ↓ User / Membership Business permissions ↓ Role / Permission
La sécurité technique protège l'accès au système.
Les règles métier déterminent ensuite ce que l'utilisateur authentifié est réellement autorisé à faire.
Aucune couche ne doit être utilisée pour contourner les invariants du Domain.
13. Testing Strategy

13.1 Principes généraux

Les tests sont une partie du système et non une étape ajoutée après l'implémentation.
Chaque règle métier importante doit être vérifiable par un test automatisé.
La stratégie de tests suit principalement la séparation :
Domain ↓ Application ↓ Infrastructure ↓ API
Les tests doivent privilégier le comportement métier plutôt que les détails d'implémentation.

13.2 Domain Tests

Les tests du Domain vérifient les invariants et les comportements qui ne nécessitent ni HTTP ni base de données.
Ils couvrent notamment :

transitions de statut du Ticket ;

règles de priorité ;

FIFO dans chaque groupe ;

réinsertion progressive ;

reinsertionCount ;

règles de cancellation ;

règles de présence ;

calcul de durée réelle ;

règles permettant de déterminer l'estimation de durée ;

invariants des Aggregates.

Exemples :
WAITING → IN_SERVICE CONFIRMED → IN_SERVICE IN_SERVICE → COMPLETED IN_SERVICE → CANCELLED = interdit COMPLETED → CANCELLED = interdit
Ces tests doivent être rapides et déterministes.

13.3 Application Use Case Tests

Les Use Cases sont testés pour vérifier l'orchestration des règles métier et des dépendances externes.
Exemples :
JoinQueue

crée un Ticket WAITING ;

utilise WALK_IN ;

initialise isPriority = false ;

obtient une position correcte ;

protège l'opération contre les accès concurrents.

BookAppointment

vérifie la disponibilité ;

vérifie les périodes d'indisponibilité ;

vérifie la capacité restante ;

crée un Ticket SCHEDULED ;

n'attribue pas de position opérationnelle.

ActivateAppointments

refuse l'activation lorsque le Provider est absent ;

active uniquement les appointments concernés ;

respecte createdAt ASC, puis Ticket.id ASC ;

ne déplace pas les Walk-ins existants.

FinishAndStartNext

termine le Ticket courant ;

sélectionne le bon candidat ;

respecte priorité et position ;

vérifie la présence ;

applique SKIPPED et la réinsertion lorsque nécessaire ;

empêche une troisième notification.

13.4 Persistence Tests

Les tests de persistence vérifient que le mapping entre le modèle et PostgreSQL respecte les contraintes attendues.
Ils couvrent notamment :

génération de Long IDs ;

Foreign Keys ;

contraintes UNIQUE ;

mapping des enums ;

dates et heures ;

relations entre Aggregates ;

persistence des Entities internes aux Aggregates ;

comportement des requêtes Repository.

Ces tests peuvent utiliser une base PostgreSQL dédiée aux tests afin de vérifier le comportement réel du moteur SQL.

13.5 Repository Tests

Chaque Repository important doit être testé sur les requêtes qui portent une responsabilité technique significative.
Exemples :
TicketRepository.findById(...) TicketRepository.findActiveByQueue(...) TicketRepository.findScheduledForDate(...) TicketRepository.findTicketsAhead(...)
Les requêtes utilisées pour :

sélectionner le prochain candidat ;

déterminer les Tickets précédents ;

gérer la concurrence ;

calculer la capacité ;

doivent être testées avec les données représentatives correspondantes.

13.6 Concurrency Tests

Les opérations sensibles à la concurrence doivent disposer de tests spécifiques.
Cas importants :

JoinQueue

Deux Customers rejoignent simultanément la même Queue.
Résultat attendu :

aucun doublon de position ;

aucun Ticket perdu ;

ordre cohérent avec les règles métier.

BookAppointment

Deux réservations concurrentes consomment la capacité restante.
Résultat attendu :

aucune réservation au-delà de la capacité disponible ;

les décisions sont basées sur un état cohérent.

FinishAndStartNext

Deux opérations concurrentes tentent de sélectionner le prochain Ticket.
Résultat attendu :

un seul traitement obtient le candidat ;

un même Ticket ne démarre pas deux fois.

CancelTicket

Une cancellation concurrente avec une opération de sélection.
Résultat attendu :

l'état final respecte les règles métier ;

aucun Ticket annulé n'est sélectionné comme candidat.

13.7 API Tests

Les tests API vérifient le contrat HTTP.
Ils couvrent notamment :

validation des Request DTOs ;

authentification ;

autorisation ;

codes HTTP ;

structure des Response DTOs ;

erreurs métier ;

absence d'exposition des Entities JPA.

Exemple :
Customer A ↓ POST /tickets/{ticketId}/cancel ↓ Ticket belongs to Customer B ↓ 403 Forbidden
Le comportement exact du code HTTP sera aligné avec le contrat API définitif.

13.8 Security Tests

Les tests de sécurité vérifient notamment :

accès sans Token ;

Token invalide ;

Token expiré ;

Customer accédant aux données d'un autre Customer ;

Provider utilisant une ressource d'un autre Provider ;

Membership appartenant à une autre Organization ;

permissions insuffisantes.

La sécurité ne doit pas être considérée comme testée uniquement parce que Spring Security est configuré.

13.9 Integration Tests

Les tests d'intégration vérifient plusieurs couches ensemble.
Exemple :
HTTP ↓ Controller ↓ Application Service ↓ Domain ↓ Repository Adapter ↓ PostgreSQL
Ils permettent de vérifier que les différents composants collaborent réellement comme prévu.
Les Use Cases critiques du MVP doivent disposer d'un nombre raisonnable de tests d'intégration.

13.10 Tests de Scénarios Métier

Les règles complexes doivent également être vérifiées sous forme de scénarios complets.
Exemple :
1. Customer A rejoint la Queue 2. Customer B rejoint la Queue 3. Customer A est notifié 4. Customer A ne répond pas 5. Customer A est réinséré 6. Customer B devient candidat 7. Customer B démarre le service 8. Customer B termine le service 9. Customer A peut redevenir candidat selon sa position
   Ces scénarios permettent de vérifier que plusieurs règles cohérentes fonctionnent ensemble.

13.11 Tests de Régression

Une règle métier corrigée ou modifiée doit disposer d'un test empêchant sa réapparition sous une forme incorrecte.
Toute modification d'une règle verrouillée dans DECISIONS.md doit donc entraîner une réévaluation des tests concernés.
Le code ne doit pas être considéré comme la seule source de vérité.
Les tests doivent refléter les décisions métier validées.

13.12 Test Pyramid

Le projet privilégie une majorité de tests rapides et ciblés :
E2E / API ▲ │ Integration ▲ │ Application / Domain ▲ │ Unit Tests
La majorité des règles métier doit être testée sans dépendance réseau ou infrastructure inutile.
Les tests d'intégration et API sont utilisés lorsqu'ils apportent une vérification que les tests unitaires ne peuvent pas fournir.

13.13 Règle générale

Un test doit répondre à une question concrète.
Avant d'ajouter un test, il faut pouvoir identifier :

le comportement vérifié ;

la règle concernée ;

le risque évité.

Le nombre de tests n'est pas un objectif en soi.
L'objectif est de disposer d'une suite de tests permettant de modifier le système avec confiance tout en conservant les invariants métier.

14. Final Technical Design Rule

La conception technique du MVP suit les principes suivants :

le Domain porte les règles métier ;

les Application Services orchestrent les Use Cases ;

les Repository Ports isolent la persistence ;

les Adapters isolent les technologies externes ;

REST expose des contrats DTO indépendants du modèle JPA ;

PostgreSQL protège les invariants structurels ;

les transactions protègent les opérations métier ;

la concurrence est traitée explicitement là où elle est nécessaire ;

les tests protègent les comportements et invariants importants.

Toute nouvelle abstraction, dépendance ou technologie doit répondre à un besoin concret du système.
Simple, testable, cohérent et compris avant d'être optimisé.
