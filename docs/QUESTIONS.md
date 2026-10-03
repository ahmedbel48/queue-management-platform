Questions ouvertes du projet

1. Objectif du document

Ce document regroupe les questions de conception qui ont été ouvertes au cours du projet ainsi que leur état de résolution.
Une question présente dans la section Questions actuellement ouvertes ne doit pas être transformée en décision par hypothèse.
Avant toute implémentation dépendant d'une question encore ouverte, celle-ci doit être :

analysée ;

discutée ;

validée ;

documentée dans DECISIONS.md ;

répercutée dans les documents concernés.

Une question résolue doit être retirée de la liste des questions actuellement ouvertes.
La décision finale correspondante doit être conservée dans DECISIONS.md.

2. Questions actuellement ouvertes

Aucune question de conception métier ou architecturale précédemment identifiée ne reste actuellement ouverte.
Les questions Q1 à Q5 ont été analysées et résolues.
Les détails techniques qui restent à concevoir pendant la phase de conception technique ne constituent pas automatiquement de nouvelles questions métier ouvertes et ne doivent pas être considérés comme des décisions implicites.

3. Questions résolues

Q1 — Stratégie de génération de Ticket.id

Question

Quelle stratégie utiliser pour générer l'identifiant technique du Ticket ?
La question concernait notamment :

le type d'identifiant ;

son unicité ;

sa génération ;

son utilisation interne ;

son éventuel affichage au client ;

les contraintes liées à la concurrence ;

son comportement lors des opérations de persistence.

Résolution

La stratégie retenue est :
Ticket.id = Long
Ticket.id constitue la clé primaire interne du ticket.
Il est généré par le mécanisme de persistence utilisé avec PostgreSQL/JPA.
Le Ticket.id peut être utilisé comme critère secondaire de départage lorsque plusieurs rendez-vous possèdent le même createdAt.
Il ne remplace jamais createdAt comme critère principal d'ordre.
Aucun ticketNumber destiné à l'affichage utilisateur n'est ajouté au MVP sans cas d'utilisation concret le justifiant.

Décision officielle

Voir DECISIONS.md — Decision 38.

Q2 — Déclenchement de la première prestation de la journée

Question

Quel événement déclenche exactement le premier passage vers IN_SERVICE de la journée ?

Résolution

La présence du ServiceProvider permet l'activation opérationnelle de la journée et l'activation des rendez-vous éligibles.
Cependant, l'activation matinale ne démarre pas automatiquement une prestation.
Le premier passage à IN_SERVICE nécessite une action explicite du ServiceProvider pour sélectionner le prochain ticket.
La présence physique du client doit ensuite être vérifiée avant le passage à IN_SERVICE.
Le mécanisme de début de journée est donc distinct du mécanisme de démarrage d'une prestation.

Décision officielle

Voir DECISIONS.md — Decision 40.

Q3 — Indisponibilité ajoutée après création de rendez-vous

Question

Que se passe-t-il lorsqu'une UnavailabilityPeriod est créée alors que des tickets SCHEDULED existent déjà pendant cette période ?

Résolution

Une nouvelle UnavailabilityPeriod ne doit pas créer silencieusement un conflit avec des rendez-vous existants.
Lorsqu'un conflit avec des tickets planifiés existe, notamment avec des tickets SCHEDULED, la création ou la modification de la période d'indisponibilité est refusée dans le parcours normal du MVP.
Le système ne doit pas automatiquement :

annuler les rendez-vous ;

les déplacer ;

introduire un nouvel état CONFLICTED.

Les tickets opérationnels ne sont pas interrompus simplement par la création d'une période d'indisponibilité.
La gestion d'une fermeture opérationnelle relève du statut opérationnel du prestataire et des règles correspondantes.

Décision officielle

Voir DECISIONS.md — Decision 39.

Q4 — Canal de notification

Question

Quel canal de notification sera utilisé dans le MVP ?
Les possibilités analysées comprenaient notamment :

SMS ;

WhatsApp ;

autres canaux.

Résolution

Le canal retenu pour le MVP est :
SMS
La logique métier des notifications reste indépendante du canal concret.
L'architecture utilise un NotificationPort auquel est associé un adapter SMS.
Le fournisseur SMS concret reste un choix technique d'infrastructure et sera défini pendant la conception technique.
Le changement futur de canal ne doit pas nécessiter de modifier la logique métier centrale de la Queue.

Décision officielle

Voir DECISIONS.md — Decision 37.

Q5 — Gestion des opérations concurrentes

Question

Comment protéger les opérations sensibles lorsqu'elles sont exécutées simultanément ?
La question concernait notamment :

l'ajout simultané de tickets ;

la modification des positions ;

la sélection du prochain ticket ;

le démarrage simultané d'une prestation ;

la combinaison fin de prestation + sélection du suivant ;

les modifications concurrentes d'un groupe de priorité ;

la capacité des rendez-vous.

Résolution

Les opérations métier modifiant l'état du système ou des invariants critiques sont exécutées dans une transaction adaptée au cas d'utilisation.
Pour les opérations opérationnelles de Queue, Queue constitue la ressource principale de coordination.
Les opérations critiques utilisent un contrôle de concurrence approprié, notamment un verrouillage pessimiste lorsque la cohérence de l'ordre courant l'exige.
La capacité des rendez-vous utilise une synchronisation propre au contexte de planification pertinent, notamment :
ServiceProvider + Date
Le niveau d'isolation par défaut retenu est :
READ COMMITTED
Le verrouillage optimiste peut être utilisé lorsqu'il est plus approprié à une ressource donnée.
Les invariants critiques sont également protégés autant que possible par des contraintes ou index de base de données.
Les appels vers des services externes, notamment le fournisseur SMS, ne doivent pas maintenir un verrou de base de données pendant l'appel réseau.
Les conflits et deadlocks peuvent faire l'objet d'un retry technique lorsqu'une réexécution est sûre.

Décision officielle

Voir DECISIONS.md — Decision 41.

4. Questions précédemment résolues — décisions fondamentales

Les éléments suivants ont été résolus avant Q1–Q5 et sont conservés ici comme trace historique.
Ils ne constituent pas des questions ouvertes.

Q6 — Stack technique

Résolution

Backend → Java + Spring Boot Database → PostgreSQL API → REST API Frontend → séparé du backend
Les détails techniques du frontend restent à définir pendant la conception technique.

Décision officielle

Voir DECISIONS.md — Decision 33.

Q7 — Style d'architecture

Résolution

Modular Monolith + Principes de Hexagonal Architecture
Les modules correspondent aux responsabilités métier.
Les principes hexagonaux servent à limiter les dépendances du domaine et de la logique applicative envers les technologies externes.
Les Microservices sont hors périmètre du MVP.
La structure interne détaillée des modules est traitée pendant la conception technique.

Décision officielle

Voir DECISIONS.md — Decision 34.

Q8 — Stratégie de persistence

Résolution

JPA / Hibernate ↓ PostgreSQL
JPA/Hibernate constitue le mécanisme principal de persistence.
Les requêtes SQL natives peuvent être utilisées lorsqu'un besoin technique clair le justifie.
Le domaine ne doit pas être conçu uniquement pour satisfaire les contraintes de l'ORM.
Les mappings, repositories, migrations et détails techniques de persistence sont traités pendant la conception technique.

Décision officielle

Voir DECISIONS.md — Decision 35.

Q9 — Authentication et OTP

Résolution

Phone + OTP ↓ Authentication ↓ Token-based API authentication
L'autorisation repose sur :
User ↓ Membership ↓ Role ↓ Permission
Les OTP doivent notamment respecter :

expiration ;

nombre limité de tentatives ;

limitation des demandes ;

absence de stockage en clair.

Le fournisseur SMS et les détails précis du mécanisme de token restent des choix techniques à définir pendant la conception technique.

Décision officielle

Voir DECISIONS.md — Decision 36.

Q10 — Synchronisation UML

Résolution

La synchronisation UML n'est plus une question de conception.
Les sept diagrammes UML ont été créés et doivent être maintenus cohérents avec les décisions officielles.
Les diagrammes concernés sont :
01-domain-class-diagram.puml 02-ticket-state-diagram.puml 03-queue-entry-sequence.puml 04-notification-sequence.puml 05-day-start-sequence.puml 06-service-completion-sequence.puml 07-eta-sequence.puml
Toute modification d'une décision affectant le comportement représenté dans un diagramme doit entraîner la vérification et, si nécessaire, la mise à jour du diagramme concerné.

Nature

Il s'agit désormais d'une règle de travail et de cohérence documentaire, et non d'une question ouverte.

5. Questions métier résolues — trace historique

Customer ↔ User

Résolution

User 1 ─── 0..1 Customer
Customer représente le concept métier du client.
User représente l'identité technique utilisée par la plateforme.

Décision officielle

Voir DECISIONS.md — Decision 17.

ServiceProvider ↔ Service

Résolution

ServiceProvider │ └── 0..* ServiceProviderService │ └── Service
ServiceProviderService porte notamment :
defaultDurationMinutes
La durée par défaut appartient donc à la combinaison :
ServiceProvider + Service

Décisions officielles

Voir DECISIONS.md — Decisions 11, 16 et 25.

Customer et Organization

Résolution

Le modèle distingue :
Identité globale du Customer
et :
Contexte métier spécifique à une Organization
Le MVP ne possède pas encore CustomerOrganisation.
Cette extension reste prévue pour un besoin futur concret.
Elle ne constitue pas une question bloquante pour le MVP actuel.

Décisions officielles

Voir DECISIONS.md — Decision 26.

6. Règle de résolution des futures questions

Toute nouvelle question de conception doit suivre le processus :
Identifier le problème ↓ Examiner les contraintes ↓ Comparer les solutions ↓ Choisir une solution ↓ Expliquer pourquoi ↓ Ajouter la décision dans DECISIONS.md ↓ Mettre à jour CURRENT_STATE.md ↓ Mettre à jour PROJECT_CONTEXT.md si nécessaire ↓ Synchroniser les diagrammes UML si nécessaire ↓ Retirer la question de QUESTIONS.md
Une question ne devient jamais une décision automatiquement.
Aucune décision importante ne doit être introduite silencieusement dans le code.

7. État actuel

Questions de conception ouvertes : 0
Les questions précédemment identifiées Q1 à Q5 sont résolues.
Les questions historiques Q6 à Q10 sont également résolues et conservées uniquement pour la traçabilité du projet.
Les détails techniques nécessaires à l'implémentation qui ne constituent pas encore des décisions officielles seront traités pendant la phase de conception technique.
Ils ne doivent pas être considérés comme implicitement décidés tant qu'ils n'ont pas été analysés et validés.

8. Relation avec les autres documents

QUESTIONS.md ne constitue pas la source de vérité des décisions.
La hiérarchie documentaire est la suivante :
DECISIONS.md ↓ CURRENT_STATE.md ↓ PROJECT_CONTEXT.md ↓ TECHNICAL_DESIGN.md ↓ UML / Code / Tests
QUESTIONS.md sert à suivre les sujets qui nécessitent une décision et à conserver leur historique de résolution.
Lorsqu'une question est résolue, la décision officielle se trouve dans DECISIONS.md.
Les détails d'implémentation qui seront définis dans TECHNICAL_DESIGN.md ne doivent pas être confondus avec les décisions métier et architecturales de DECISIONS.md.