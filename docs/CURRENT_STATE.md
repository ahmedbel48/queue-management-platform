CURRENT_STATE.md

État actuel du projet

1. Phase actuelle

Le projet se trouve actuellement dans la phase :
Conception technique — en cours de finalisation
La modélisation métier principale du MVP est terminée.
Les éléments suivants ont été définis et stabilisés :

le modèle du domaine ;

les organisations ;

les utilisateurs ;

les clients ;

les memberships ;

les rôles et permissions ;

les prestataires ;

les services ;

les associations prestataire-service ;

les files d'attente ;

les tickets ;

les rendez-vous ;

les disponibilités ;

les périodes d'indisponibilité ;

les états et transitions du Ticket ;

les priorités ;

la réinsertion ;

les notifications ;

la présence physique ;

l'activation des rendez-vous ;

le début et la fin d'une prestation ;

la durée réelle ;

l'estimation de durée ;

le calcul dynamique de l'ETA ;

l'annulation ;

la fin de journée ;

les domaines d'activité ;

les principales règles de concurrence ;

les principales décisions d'architecture.

La programmation de la plateforme n'a pas encore commencé.
La prochaine étape consiste à terminer la synchronisation documentaire, puis à finaliser la conception technique détaillée avant l'implémentation.

2. Sources de vérité

Le projet utilise plusieurs documents et artefacts ayant chacun un rôle précis.

PROJECT_CONTEXT.md

Contient :

le contexte général du projet ;

le périmètre ;

les objectifs ;

les règles métier générales ;

les contraintes importantes.

DECISIONS.md

Contient :

les décisions officiellement validées ;

leur justification ;

les décisions architecturales ;

les décisions métier importantes ;

les décisions techniques déjà stabilisées.

DECISIONS.md constitue la source de vérité pour les décisions.

CURRENT_STATE.md

Décrit :

l'état actuel du projet ;

ce qui est terminé ;

ce qui est stabilisé ;

ce qui reste à faire ;

l'état de l'implémentation.

Ce document ne doit pas introduire de nouvelle décision métier ou architecturale.

QUESTIONS.md

Contient :

les questions nécessitant une décision ;

l'état de résolution ;

la trace historique des questions précédemment ouvertes.

À l'état actuel, aucune question de conception bloquante ne reste ouverte.

docs/uml/

Contient les diagrammes représentant :

le domaine ;

les états du Ticket ;

les séquences principales ;

les comportements importants du système.

Les UML doivent rester cohérents avec les décisions officielles.

Code

Le code représente l'implémentation réelle des décisions validées.
Il ne constitue pas une source autonome de décisions métier.

Tests

Les tests vérifient que l'implémentation respecte les comportements attendus et les invariants définis.

3. État global

ÉlémentÉtatModélisation métierTERMINÉEDécisions métier principalesSTABILISÉESDécisions architecturales principalesSTABILISÉESDécisions techniques principalesSTABILISÉESQuestions de conception0 ouverteDiagrammes UMLCRÉÉSSynchronisation documentaire finaleEN COURSConception technique détailléeEN COURS DE FINALISATIONBackendNON COMMENCÉFrontendNON COMMENCÉBase de donnéesNON IMPLÉMENTÉETests automatisésNON COMMENCÉSDéploiementNON COMMENCÉ
Aucune implémentation importante n'a encore commencé.

4. Périmètre actuel du MVP

Le MVP actuel modélise une plateforme générique de gestion de file d'attente avec un premier cas d'utilisation centré sur les prestations de services.
Le scénario principal est celui d'une organisation possédant un ou plusieurs prestataires, proposant des services à des clients.
Le système prend notamment en charge :

les clients ;

les prestataires ;

les services ;

les rendez-vous ;

les entrées sans rendez-vous ;

les files d'attente ;

les priorités ;

les notifications ;

la présence physique ;

le suivi du cycle de vie des tickets ;

l'estimation dynamique du temps d'attente.

Le concept général est volontairement plus large que le premier domaine d'utilisation.
Les règles spécifiques à un domaine particulier ne doivent pas être codées directement dans le cœur du domaine générique sans justification.

5. Modèle actuel du domaine

Les concepts actuellement retenus dans le MVP sont :

User

Customer

Organization

Membership

Role

Permission

ServiceProvider

Service

ServiceProviderService

WeeklyAvailability

UnavailabilityPeriod

Queue

Ticket

Domaine

OrganisationDomaine

Le MVP ne contient pas actuellement :

CustomerOrganisation

CustomField

CustomFieldValue

Notification

modules métier spécifiques à un domaine particulier.

Ces éléments peuvent être introduits ultérieurement lorsqu'un besoin métier concret le justifie.

6. User et Customer

User

User représente l'identité et le compte d'authentification.
Un même utilisateur peut avoir différents rôles selon les organisations auxquelles il appartient.

Customer

Customer représente le profil métier du client.
Relation actuelle :
User 1 ─── 0..1 Customer
Un utilisateur peut donc posséder zéro ou un profil Customer.
Un client peut créer plusieurs tickets :
Customer 1 ─── 0..* Ticket
Le contexte spécifique d'un client dans une organisation n'est pas encore modélisé par CustomerOrganisation.
Cette extension reste hors du MVP actuel.

7. Organization et Membership

Une Organization représente une organisation utilisant la plateforme.
L'appartenance d'un utilisateur à une organisation est représentée par Membership.
User 1 ─── 0..* Membership Membership * ─── 1 Organization
Le Membership porte les rôles de l'utilisateur dans l'organisation.
Membership * ─── * Role Role * ─── * Permission
Le fait qu'un utilisateur possède un rôle particulier ne signifie pas automatiquement qu'il est ServiceProvider.

8. ServiceProvider

ServiceProvider représente le prestataire opérationnel dans une organisation.
Relation :
Membership 1 ─── 0..1 ServiceProvider Organization 1 ─── 0..* ServiceProvider
Invariant :
Membership.organization == ServiceProvider.organization
Le modèle n'utilise pas une hiérarchie d'héritage entre User, ServiceProvider, Receptionist, Manager, etc.
Un même utilisateur peut cumuler plusieurs responsabilités dans une organisation selon les memberships et permissions applicables.

9. Services

Service représente une prestation proposée par la plateforme.
La relation entre ServiceProvider et Service est représentée par :
ServiceProvider 1 ─── 0..* ServiceProviderService Service 1 ─── 0..* ServiceProviderService
ServiceProviderService contient notamment :
id defaultDurationMinutes
La durée par défaut appartient donc à la combinaison :
ServiceProvider + Service
et non uniquement au Service.

10. Queue

Chaque ServiceProvider possède une file permanente.
ServiceProvider 1 ─── 1 Queue Queue 1 ─── 0..* Ticket
La Queue n'est pas créée pour chaque journée.
Elle existe de manière permanente pour le prestataire.
La Queue est responsable du maintien des invariants d'ordre de la file.
Le Ticket est principalement responsable de son cycle de vie.

11. Ticket

Le Ticket représente une entrée client dans le système de file ou de rendez-vous.
Les informations actuellement retenues comprennent notamment :
id entryType status isPriority position reinsertionCount notificationAttempts createdAt scheduledAt notifiedAt confirmedAt inServiceAt completedAt
Relations principales :
Queue 1 ─── 0..* Ticket Customer 1 ─── 0..* Ticket Service 1 ─── 0..* Ticket
Un Ticket appartient à une seule Queue et possède un seul Service.

12. Identifiant du Ticket

L'identifiant technique actuel est :
Ticket.id : Long
Il constitue principalement l'identifiant interne de persistence.
Lorsque cela est nécessaire pour obtenir un ordre déterministe, Ticket.id peut être utilisé comme critère secondaire après createdAt.
Il ne remplace pas createdAt comme critère principal d'ordre.
Aucun identifiant public supplémentaire de type ticketNumber n'est actuellement prévu dans le MVP.

13. Position du Ticket

La position représente la position actuelle du Ticket dans son groupe de priorité.
Elle n'est pas une position globale unique dans toute la Queue.
Les deux groupes conceptuels sont :
Priority Normal
Les tickets prioritaires sont traités avant les tickets normaux.
À l'intérieur d'un même groupe, l'ordre dépend de la position actuelle.
Les positions actives commencent à :
1
et doivent rester cohérentes après les opérations de réorganisation.
La Queue est responsable du maintien de ces invariants.

14. Priorité

La priorité est représentée par :
isPriority : boolean
L'ordre général est :
Priority ↓ Normal
Un rendez-vous n'est pas automatiquement prioritaire.
Le type d'entrée et la priorité constituent deux concepts distincts.

15. Réinsertion

Lorsqu'un Ticket est réinséré après une sélection ignorée ou un retard, il reste dans son groupe de priorité.
Ainsi :
Priority → Priority Normal → Normal
La réinsertion n'entraîne pas automatiquement un changement de priorité.
Le système utilise :
reinsertionCount
pour suivre les réinsertions effectuées pendant la journée.
La valeur n'est pas réinitialisée pendant la journée.
La nouvelle position suit la règle définie dans les décisions officielles :
newGroupPosition = min(oldGroupPosition + reinsertionCount, groupSize)
Le compteur est partagé par les réinsertions provenant des cas SKIPPED et DELAYED.

16. États du Ticket

Les états actuellement retenus sont :
SCHEDULED WAITING NOTIFIED CONFIRMED DELAYED SKIPPED IN_SERVICE COMPLETED CANCELLED NO_SHOW
Ils sont conceptuellement regroupés en :

Planification

SCHEDULED

États opérationnels

WAITING NOTIFIED CONFIRMED DELAYED SKIPPED IN_SERVICE

États terminaux

COMPLETED CANCELLED NO_SHOW

17. Entrée sans rendez-vous

Un Walk-in entre directement dans :
WAITING
Il peut donc être candidat aux opérations normales de sélection selon les règles de la Queue.

18. Rendez-vous

Un rendez-vous commence dans :
SCHEDULED
Il ne possède pas encore de position opérationnelle dans la Queue.
Au début de la journée, un rendez-vous éligible peut être activé vers :
WAITING
L'activation ne démarre pas automatiquement une prestation.

19. Cycle de vie principal

Les transitions principales actuellement retenues sont :
SCHEDULED → WAITING WAITING → NOTIFIED NOTIFIED → CONFIRMED NOTIFIED → DELAYED NOTIFIED → SKIPPED DELAYED → WAITING SKIPPED → WAITING WAITING → IN_SERVICE CONFIRMED → IN_SERVICE IN_SERVICE → COMPLETED
Les transitions vers CANCELLED sont autorisées avant IN_SERVICE selon les règles d'autorisation.
Les états terminaux ne peuvent pas être quittés.

20. Notification

Le MVP ne possède pas d'entité métier Notification.
Les tentatives sont suivies directement sur le Ticket avec :
notificationAttempts
Le nombre maximal de tentatives est :
2
Le canal retenu est :
SMS
La logique métier reste indépendante du fournisseur SMS concret.
L'architecture prévoit :
Application ↓ NotificationPort ↓ SMS Adapter ↓ Provider externe
Le fournisseur concret reste un détail d'infrastructure.

21. Première notification

Lorsqu'un Ticket WAITING est sélectionné comme candidat à la notification :
WAITING → NOTIFIED
et :
notificationAttempts += 1
Une notification SMS est ensuite envoyée au client.
L'appel au fournisseur externe ne doit pas maintenir un verrou critique de base de données.

22. Réponse à une notification

Une confirmation du client produit :
NOTIFIED → CONFIRMED
Une indication de retard produit :
NOTIFIED → DELAYED
Le Ticket peut ensuite être réinséré selon les règles applicables :
DELAYED → WAITING
La réinsertion conserve le groupe de priorité.
Une troisième notification n'est jamais autorisée.

23. Absence de réponse

En cas d'absence de réponse dans le délai prévu :
NOTIFIED → SKIPPED
Si une nouvelle tentative est encore autorisée, le Ticket peut être réinséré dans :
WAITING
Si le nombre maximal de tentatives a été atteint, aucune troisième notification n'est autorisée.
Le traitement opérationnel ultérieur du Ticket dépend alors des règles de sélection et de fin de journée applicables.

24. Présence physique

La réponse à une notification ne constitue pas une preuve de présence physique.
Avant le passage vers :
IN_SERVICE
le prestataire doit vérifier que le client est réellement présent.
Cette vérification constitue une condition opérationnelle distincte de :
CONFIRMED

25. Début de prestation

Un Ticket passe à :
IN_SERVICE
uniquement lorsque le prestataire commence réellement la prestation.
Le début réel est enregistré dans :
inServiceAt
L'activation des rendez-vous en début de journée ne démarre pas automatiquement une prestation.
Le prestataire sélectionne explicitement le prochain candidat et vérifie sa présence avant le passage à IN_SERVICE.

26. Fin de prestation

Lorsqu'une prestation est terminée :
IN_SERVICE → COMPLETED
Le système renseigne :
completedAt
La durée réelle est alors calculable :
completedAt - inServiceAt
La fin de prestation et la recherche du prochain candidat constituent une opération métier cohérente.

27. Durée réelle

Une prestation ne participe au calcul des durées réelles que lorsqu'elle est terminée.
Les prestations sans completedAt ne sont donc pas utilisées pour calculer la moyenne.
La durée réelle est calculée à partir de :
inServiceAt completedAt

28. Durée estimée

La durée estimée est calculée séparément pour chaque combinaison :
ServiceProvider + Service
Si moins de trois prestations terminées sont disponibles :
defaultDurationMinutes
est utilisé.
À partir de trois prestations terminées :
moyenne des durées réelles terminées
est utilisée.
Il n'existe pas actuellement :

de moyenne globale ;

de moyenne mélangeant plusieurs services ;

de moyenne mélangeant plusieurs prestataires ;

de moyenne glissante.

29. ETA

L'ETA est calculé dynamiquement.
Il n'est pas persisté dans le Ticket.
Il n'existe donc pas actuellement de champ :
estimatedWaitMinutes
L'ETA dépend notamment :

du temps restant du Ticket actuellement IN_SERVICE ;

des Tickets placés avant le Ticket demandé ;

des durées estimées correspondant au couple ServiceProvider + Service.

L'ETA ne modifie pas :

la position ;

la priorité ;

le statut ;

l'ordre de passage ;

les notifications.

L'ETA constitue une estimation et non une garantie.

30. Sélection du client suivant

La sélection respecte l'ordre général :
Priority ↓ Normal
puis :
position croissante
Les Tickets terminaux ne sont pas candidats.
Un Ticket SCHEDULED n'est pas candidat tant qu'il n'a pas été activé.
Il ne doit exister qu'un seul Ticket IN_SERVICE par prestataire.
La présence physique du client doit être vérifiée avant le passage à IN_SERVICE.

31. Activation des rendez-vous

L'activation des rendez-vous dépend d'abord de la présence opérationnelle du prestataire.
Si le prestataire n'est pas présent :
SCHEDULED
reste inchangé.
Lorsque le prestataire est présent, les rendez-vous du jour éligibles peuvent être activés.
Les Walk-ins WAITING déjà présents conservent leur ordre.
Les rendez-vous activés sont ensuite insérés à la fin de leur groupe respectif.
Lorsque plusieurs rendez-vous doivent être activés, leur ordre est déterminé par :
createdAt ASC Ticket.id ASC
Ticket.id constitue le second critère lorsque createdAt est identique.
L'ancienneté d'une réservation ne permet donc pas à un rendez-vous activé de dépasser silencieusement un Walk-in déjà présent dans la file.
L'activation ne démarre jamais automatiquement une prestation.

32. Capacité des rendez-vous

La capacité de rendez-vous est calculée à partir du temps de travail disponible.
Les éléments concernés comprennent notamment :

les disponibilités hebdomadaires ;

les périodes d'indisponibilité ;

les durées des prestations déjà réservées.

La capacité est exprimée en minutes disponibles.
La capacité agrégée ne constitue pas à elle seule une garantie de placement exact dans chaque intervalle horaire.
Les contrôles de capacité doivent être protégés contre les réservations concurrentes.
Les détails algorithmiques de calcul et de verrouillage sont traités dans la conception technique.

33. Disponibilités hebdomadaires

WeeklyAvailability représente les horaires réguliers d'un prestataire.
Attributs principaux :
id dayOfWeek startTime endTime
Plusieurs intervalles peuvent exister pour une même journée.
Invariant :
startTime < endTime
Les horaires traversant minuit ne sont pas supportés dans le MVP.

34. Périodes d'indisponibilité

UnavailabilityPeriod représente une période pendant laquelle le prestataire n'est pas disponible.
Attributs principaux :
id startDateTime endDateTime reason
Une période d'indisponibilité prend priorité sur la disponibilité hebdomadaire.
Conceptuellement :
Disponibilité effective = WeeklyAvailability - UnavailabilityPeriod
La création ou modification d'une période d'indisponibilité incompatible avec des rendez-vous SCHEDULED concernés doit être refusée.
Le MVP ne prévoit pas :

d'annulation automatique ;

de déplacement automatique ;

d'état CONFLICTED.

Une indisponibilité ne doit pas interrompre silencieusement une prestation déjà en cours.

35. Annulation

Les Tickets suivants peuvent être annulés avant IN_SERVICE selon les règles d'autorisation :
SCHEDULED WAITING NOTIFIED CONFIRMED DELAYED SKIPPED
Les états suivants ne peuvent pas être annulés :
IN_SERVICE COMPLETED NO_SHOW CANCELLED
CANCELLED est terminal.
Un Ticket annulé ne peut plus :

être candidat ;

être réinséré ;

recevoir de notification ;

revenir dans la Queue opérationnelle.

Les droits d'annulation dépendent de l'acteur et de ses permissions.

36. Fin de journée

À la fin de la journée, les Tickets opérationnels encore actifs peuvent être clôturés en :
NO_SHOW
Cela concerne notamment :
WAITING NOTIFIED CONFIRMED DELAYED SKIPPED
Les Tickets déjà terminés ou annulés ne sont pas modifiés.

37. Domaines d'activité

Domaine représente un domaine d'activité générique.
Exemples :

Coiffure ;

Médecine ;

Dentisterie ;

Garage.

Le domaine n'est pas représenté par un enum codé en dur.
Une organisation peut être associée à plusieurs domaines :
Organization 1 ─── 0..* OrganisationDomaine Domaine 1 ─── 0..* OrganisationDomaine
Le système doit privilégier les données et la configuration plutôt que des conditions métier codées spécifiquement pour chaque domaine.

38. Extensions hors MVP

Les éléments suivants sont actuellement hors périmètre :

CustomerOrganisation ;

CustomField ;

CustomFieldValue ;

entité métier Notification ;

modules métier spécifiques ;

champs personnalisés ;

audit complet ;

historique avancé ;

fonctionnalités multi-prestataires avancées ;

règles métier spécifiques à chaque domaine.

Ces extensions ne doivent être ajoutées que lorsqu'un besoin concret est identifié et validé.

39. Architecture technique retenue

La stack technique retenue est :
Java Spring Boot PostgreSQL REST API
Le frontend reste séparé du backend et communique avec celui-ci via l'API REST.
L'architecture backend est :
Modular Monolith + Principes d'architecture Hexagonale
Les microservices sont hors périmètre du MVP.
Les principes hexagonaux doivent rester proportionnés au projet et ne doivent pas introduire de complexité artificielle.

40. Persistence

La persistence principale utilise :
JPA / Hibernate ↓ PostgreSQL
Le SQL natif peut être utilisé lorsqu'un besoin technique clair le justifie.
Le domaine métier ne doit pas être conçu uniquement pour satisfaire les contraintes de l'ORM.
Les entités de persistence ne doivent pas être exposées directement comme modèle d'API.
Les détails de mapping, repositories, migrations et requêtes seront définis dans la conception technique.

41. Transactions et concurrence

Les opérations métier importantes sont exécutées dans des transactions adaptées à leur cas d'utilisation.
Le niveau d'isolation PostgreSQL retenu est :
READ COMMITTED
Des verrous pessimistes peuvent être utilisés lorsque la cohérence de l'opération l'exige, notamment pour :

les opérations critiques de Queue ;

l'attribution ou la réorganisation des positions ;

la sélection du prochain candidat ;

certaines transitions critiques ;

les contrôles de capacité soumis à concurrence.

Le verrouillage optimiste peut être utilisé lorsqu'il est plus approprié.
Les contraintes de base de données constituent une protection supplémentaire des invariants.
Les transactions doivent rester courtes.
Aucun appel réseau externe ne doit être effectué en conservant un verrou critique de base de données.
Les conflits et deadlocks peuvent être traités par une stratégie de retry lorsque la réexécution est sûre.
Les détails précis d'implémentation seront définis dans TECHNICAL_DESIGN.md.

42. Authentification et autorisation

L'authentification retenue est :
Phone + OTP
L'API REST utilise une authentification basée sur un token.
L'autorisation suit :
User ↓ Membership ↓ Role ↓ Permission
Les OTP doivent respecter notamment :

une expiration ;

un nombre limité de tentatives ;

une limitation des demandes ;

l'absence de stockage en clair lorsque cela n'est pas nécessaire.

Le fournisseur SMS et les détails précis du mécanisme de token restent des détails d'implémentation à finaliser dans la conception technique.

43. Notification et services externes

Le canal retenu est :
SMS
Le domaine ne dépend pas d'un fournisseur SMS concret.
La structure cible est :
Application ↓ NotificationPort ↓ SMS Adapter ↓ Provider externe
Les effets externes ne sont pas considérés comme faisant partie de la même transaction atomique que la modification de l'état métier en base de données.

44. UML actuels

Les diagrammes actuellement retenus sont :
docs/uml/01-domain-class-diagram.puml docs/uml/02-ticket-state-diagram.puml docs/uml/03-queue-entry-sequence.puml docs/uml/04-notification-sequence.puml docs/uml/05-day-start-sequence.puml docs/uml/06-service-completion-sequence.puml docs/uml/07-eta-sequence.puml
Ils couvrent :

le modèle du domaine ;

le cycle de vie du Ticket ;

l'entrée dans la Queue ;

les notifications ;

l'activation quotidienne ;

la fin de prestation ;

le calcul de l'ETA.

Une vérification finale de cohérence avec DECISIONS.md reste nécessaire avant le début de l'implémentation.

45. Questions de conception

À l'état actuel :
Questions de conception ouvertes : 0
Les principales questions récemment résolues concernent notamment :

la stratégie de génération de Ticket.id ;

le déclenchement de la première prestation ;

le comportement d'une indisponibilité incompatible avec des rendez-vous existants ;

le canal de notification ;

la gestion générale des transactions et de la concurrence.

Les décisions correspondantes doivent rester documentées dans DECISIONS.md.
QUESTIONS.md ne doit pas être utilisé comme source de vérité des décisions finales.

46. Synchronisation documentaire restante

Avant l'implémentation, les éléments suivants doivent être vérifiés :
DECISIONS.md CURRENT_STATE.md QUESTIONS.md PROJECT_CONTEXT.md docs/uml/* TECHNICAL_DESIGN.md
La vérification doit notamment rechercher :

les contradictions ;

les règles obsolètes ;

les décisions absentes ;

les décisions présentes dans un document mais pas dans les autres ;

les UML ne correspondant plus aux décisions ;

les détails techniques non documentés lorsqu'ils sont nécessaires à l'implémentation.

Aucune décision importante ne doit être introduite silencieusement pendant cette étape.

47. Prochaine étape

L'ordre de travail prévu est :
1. Finaliser DECISIONS.md ↓ 2. Finaliser CURRENT_STATE.md ↓ 3. Vérifier QUESTIONS.md ↓ 4. Vérifier PROJECT_CONTEXT.md ↓ 5. Synchroniser les UML ↓ 6. Finaliser TECHNICAL_DESIGN.md ↓ 7. Effectuer une revue de cohérence globale ↓ 8. Geler les décisions du MVP ↓ 9. Préparer le squelette backend ↓ 10. Commencer l'implémentation
   L'implémentation ne doit pas commencer avant la vérification de cohérence documentaire prévue.

48. Règle de collaboration

Le principe de travail est :
L'IA propose. L'humain décide. Le dépôt Git conserve la décision. Le code applique la décision. Les tests vérifient le comportement.
L'IA peut intervenir comme :

Technical Lead ;

reviewer ;

conseiller en architecture ;

enseignant ;

critique des décisions ;

assistant de conception.

L'agent de programmation peut implémenter les décisions validées.
Il ne doit pas modifier silencieusement une règle métier verrouillée.
Lorsqu'une décision verrouillée doit changer :
Ancienne décision ↓ Nouvelle analyse ↓ Nouvelle décision ↓ Justification ↓ DECISIONS.md ↓ CURRENT_STATE.md ↓ UML concernés ↓ Code ↓ Tests

49. Principe de cohérence documentaire

Une même règle métier doit conserver la même signification dans :
PROJECT_CONTEXT.md DECISIONS.md CURRENT_STATE.md UML Code Tests
En cas de divergence, celle-ci doit être identifiée puis corrigée.
Aucune modification silencieuse d'une règle métier verrouillée n'est autorisée.
Le code ne doit pas devenir une source implicite de nouvelles règles métier.

50. Résumé exécutif

Projet │ ├── Modélisation métier │ ├── Domaine ✓ │ ├── Organization ✓ │ ├── User / Customer ✓ │ ├── Membership / Role / Permission ✓ │ ├── ServiceProvider ✓ │ ├── Service ✓ │ ├── ServiceProviderService ✓ │ ├── Queue ✓ │ ├── Ticket ✓ │ ├── États / transitions ✓ │ ├── Priorité ✓ │ ├── Réinsertion ✓ │ ├── Notifications ✓ │ ├── Présence physique ✓ │ ├── Rendez-vous ✓ │ ├── Activation quotidienne ✓ │ ├── Disponibilités ✓ │ ├── Durées réelles ✓ │ ├── ETA dynamique ✓ │ ├── Annulation ✓ │ ├── Fin de journée ✓ │ └── UML ✓ │ ├── Architecture │ ├── Java + Spring Boot ✓ │ ├── PostgreSQL ✓ │ ├── REST API ✓ │ ├── Modular Monolith ✓ │ ├── Principes Hexagonaux ✓ │ ├── JPA / Hibernate ✓ │ ├── SQL natif sélectif ✓ │ ├── Transactions ✓ │ ├── Concurrence ✓ │ ├── Phone + OTP ✓ │ ├── Token-based authentication ✓ │ └── SMS + NotificationPort ✓ │ ├── Questions │ └── Questions ouvertes 0 │ ├── Implémentation │ ├── Backend NON COMMENCÉ │ ├── Frontend NON COMMENCÉ │ ├── Base de données NON IMPLÉMENTÉE │ └── Tests automatisés NON COMMENCÉS │ └── Documentation ├── Synchronisation finale EN COURS └── TECHNICAL_DESIGN.md À FINALISER

51. État officiel actuel

La modélisation métier principale du MVP est stabilisée.
Les principales décisions métier et architecturales sont stabilisées.
Les principales décisions techniques sont stabilisées.
Aucune implémentation importante n'a encore commencé.
Aucune question de conception bloquante ne reste ouverte.
Le projet se trouve au point de transition entre la conception et l'implémentation.
La prochaine étape obligatoire est la finalisation de la synchronisation documentaire, suivie de la finalisation de TECHNICAL_DESIGN.md.
L'implémentation du backend ne doit commencer qu'après cette vérification de cohérence.