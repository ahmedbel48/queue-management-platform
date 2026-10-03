Décisions du projet

Ce document constitue la source de vérité concernant les décisions métier, techniques et architecturales acceptées du projet.
Une décision acceptée ne doit jamais être modifiée silencieusement.
Toute modification d'une décision existante doit être explicitement documentée selon la règle de modification définie à la fin de ce document.

1. Une file d'attente permanente par prestataire

Chaque ServiceProvider possède une seule Queue permanente.
La Queue n'est donc :

ni recréée chaque jour ;

ni créée par service.

Les Ticket appartiennent à cette Queue.
La position d'un Ticket est interprétée dans le contexte :

de sa Queue ;

de son groupe de priorité ;

de la journée opérationnelle concernée.

La position représente donc une position opérationnelle courante, et non une position historique permanente dans la Queue.

2. Activation réelle des rendez-vous au début de la journée

Un ticket SCHEDULED ne devient pas automatiquement WAITING uniquement parce que sa date correspond à la journée courante.
Une activation réelle est effectuée au début de la journée par le mécanisme prévu à cet effet.
L'activation des rendez-vous est conditionnée par la présence opérationnelle du ServiceProvider.
La confirmation de présence du prestataire permet d'activer les rendez-vous éligibles de la journée.
Les tickets WAITING correspondant aux clients déjà présents conservent leur position.
Les rendez-vous activés sont ensuite ajoutés à leur groupe de priorité respectif selon les règles définies par la décision 10.
L'activation matinale ne démarre pas automatiquement une prestation.
Le premier passage à IN_SERVICE nécessite :

une sélection selon les règles de la Queue ;

une vérification de présence physique du client ;

une action explicite du ServiceProvider.

Avant le jour du rendez-vous, le client peut consulter les informations de planification disponibles.
Les notifications opérationnelles et l'estimation dynamique sont liées au jour opérationnel concerné.

3. Séparation entre planification et statut opérationnel

La planification d'un prestataire et son statut opérationnel sont deux concepts distincts.
Le prestataire peut notamment être :

ACTIVE

ABSENT

CLOSED

L'absence manuelle n'implique pas automatiquement une heure de retour.
La fermeture peut être déclenchée automatiquement à la fin de la période de planification.
Le statut opérationnel ne remplace donc pas les informations de disponibilité planifiée.

4. Réinsertion progressive dans le même groupe de priorité

Lorsqu'un ticket est réinséré après un SKIPPED ou un DELAYED, il reste dans son groupe de priorité.
La nouvelle position dans son groupe est calculée à partir de sa position précédente et de son reinsertionCount.
Formule :
newGroupPosition = min(oldGroupPosition + reinsertionCount, groupSize)
La saturation à la dernière position disponible du groupe est acceptée.
Chaque réinsertion réelle augmente reinsertionCount.
Un changement de priorité n'est pas considéré comme une réinsertion et n'incrémente donc pas reinsertionCount.
Le compteur n'est pas réinitialisé pendant la journée opérationnelle.

5. Fin de prestation et sélection du suivant comme une seule opération métier

La fin d'une prestation et la sélection du prochain ticket sont traitées comme une opération métier cohérente lorsqu'elles sont effectuées par le cas d'utilisation correspondant.
Le ticket actuellement traité reste explicitement dans l'état IN_SERVICE jusqu'à la fin de la prestation.
La sélection du prochain ticket ne signifie pas automatiquement que celui-ci est déjà en prestation.
La vérification de présence physique du client reste nécessaire avant le passage à IN_SERVICE.

6. SUPERSEDED — Ancienne règle sur les attributs calculés de Ticket

Cette décision est remplacée par les décisions 11 à 15 concernant les durées et l'ETA.
L'ancienne décision prévoyait notamment de conserver estimatedWaitMinutes directement dans Ticket.
Cette approche est abandonnée car l'estimation du temps d'attente est une donnée calculée à partir de l'état courant de la file et des durées disponibles.
La valeur pourrait donc devenir obsolète si elle était persistée directement sur Ticket.
reinsertionCount reste en revanche un attribut persistant de Ticket, car il représente un état métier historique nécessaire au calcul de réinsertion.

7. Séparation des responsabilités des fragments de calcul

Les responsabilités métier sont séparées entre les fragments concernés.
Le calcul de réinsertion, le calcul de l'estimation et la sélection du prochain ticket ne doivent pas être implicitement imbriqués.
L'enchaînement entre ces responsabilités doit être explicite dans les cas d'utilisation concernés.
Une responsabilité ne doit pas modifier implicitement le comportement métier d'une autre responsabilité.

8. Priorité booléenne et FIFO

La priorité est représentée par un booléen :
isPriority
Les tickets prioritaires passent avant les tickets normaux.
À l'intérieur de chaque groupe, l'ordre est FIFO :

Premier entré, premier sorti.

Un client ne peut pas attribuer lui-même une priorité à son ticket.
La priorité est attribuée uniquement par le prestataire ou par un membre autorisé.
Les rendez-vous ne sont pas prioritaires par défaut lors de leur création.
Lorsqu'un ticket change de groupe de priorité :

il quitte son groupe actuel ;

il est placé à la fin du nouveau groupe ;

les positions concernées sont recalculées.

Un changement de groupe de priorité ne constitue pas une réinsertion.

9. Sélection du prochain ticket

Lors de la sélection opérationnelle du prochain ticket, les états éligibles sont :

WAITING

CONFIRMED

La sélection respecte :

le groupe prioritaire avant le groupe normal ;

la position dans la file à l'intérieur du groupe.

Un ticket DELAYED n'est pas directement sélectionné tant qu'il n'a pas été réinséré et replacé en WAITING.
Un ticket SKIPPED n'est pas sélectionnable directement.
Un ticket dont les notifications ont été définitivement épuisées selon les décisions 27 et 28 ne peut pas être réintroduit par une nouvelle notification.
Lorsqu'un ticket DELAYED après la deuxième notification est réinséré en WAITING, il redevient éligible à la sélection normale sans recevoir de troisième notification.
S'il n'existe aucun ticket éligible, le prestataire reste disponible.

10. La présence réelle prime sur l'ancienneté du rendez-vous

Lors de l'activation matinale des rendez-vous :

les tickets WAITING des clients déjà présents conservent exactement leur position actuelle ;

les rendez-vous activés sont ensuite insérés dans leur groupe de priorité respectif.

Entre les rendez-vous activés appartenant au même groupe, l'ordre est déterminé par :
createdAt ASC Ticket.id ASC en cas d'égalité
L'ancienneté de réservation ne permet donc pas à un rendez-vous de dépasser un client déjà présent dans le groupe concerné.
Le type APPOINTMENT ne constitue pas automatiquement une priorité.

11. Chaque ServiceProviderService possède une durée par défaut

Chaque ServiceProviderService possède une durée par défaut servant de valeur de référence lorsque les données historiques sont insuffisantes.
L'ancien attribut :
Service.averageDurationMinutes
ne fait plus partie du modèle retenu.
La durée par défaut est propre à la combinaison :
ServiceProvider + Service

12. Durée réelle d'une prestation

La durée réelle d'une prestation est calculée à partir de :
completedAt - inServiceAt
Elle n'est considérée comme valide que si les deux horodatages nécessaires sont présents et cohérents.
Seules les prestations réellement terminées sont prises en compte dans l'apprentissage historique des durées.

13. Apprentissage historique des durées dans le MVP

L'utilisation des durées réelles historiques fait partie du MVP.
Pour une combinaison donnée de prestataire et de service :

moins de 3 prestations réelles terminées → utilisation de la durée par défaut ;

à partir de 3 prestations réelles terminées → utilisation de la moyenne des durées réelles valides.

La moyenne est calculée uniquement à partir des prestations réellement terminées et valides.
Aucune moyenne globale entre prestataires ou services différents n'est utilisée.

14. Les durées sont propres au prestataire et au service

Les durées historiques ne sont pas mélangées entre différents prestataires ou différents services.
La référence est donc la combinaison :
ServiceProvider + Service
Un même service peut avoir une durée différente selon le prestataire.

15. Calcul de l'ETA

L'ETA est une donnée dérivée calculée dynamiquement.
Pour un ticket donné, le calcul prend en compte :

le temps restant de la prestation actuellement en IN_SERVICE ;

les durées estimées des tickets placés avant le ticket concerné dans l'ordre opérationnel.

L'estimation d'un ticket repose sur les règles de durée définies par les décisions 11 à 14.
L'ETA ne modifie jamais :

la position ;

la priorité ;

l'ordre ;

l'état du ticket ;

les notifications.

L'ETA n'est pas persistée directement dans Ticket.
Elle peut être recalculée chaque fois que l'état pertinent de la file ou les données nécessaires à l'estimation changent.
L'ETA constitue une estimation et non une garantie de temps de passage.

16. Le prestataire configure les services qu'il propose

Chaque ServiceProvider choisit les services qu'il propose et configure la durée par défaut associée à chacun.
La relation entre le prestataire et le service est représentée par :
ServiceProviderService
Cette relation garantit qu'un ticket ne peut utiliser qu'un service effectivement proposé par le prestataire concerné.

17. Customer est un concept du domaine distinct de User

Customer est un concept métier distinct de User.
User représente l'identité technique utilisée par la plateforme pour l'authentification et le compte.
Customer représente l'identité métier du client.
La relation retenue dans le MVP est :
User 1 ─── 0..1 Customer
Un User peut donc être associé à zéro ou un Customer.
Un Customer peut posséder plusieurs tickets au cours du temps.
Le Customer n'est pas directement rattaché à une seule Organization.

18. Séparation entre structuralType et Role

structuralType représente la fonction structurelle d'un membre dans l'organisation.
Role représente les droits et permissions attribués à ce membre.
Un Role ne détermine donc pas à lui seul l'existence d'un ServiceProvider.
L'existence d'un ServiceProvider dépend de la capacité opérationnelle du membre dans le domaine.
Cette capacité est représentée par la relation entre Membership et ServiceProvider, définie par la décision 30.

19. Pas de classe Notification métier dans le MVP

Le MVP ne nécessite pas de classe métier Notification.
La logique de notification reste séparée des concepts centraux du domaine.
Un service technique de notification peut exister dans l'architecture applicative, mais il ne constitue pas une entité métier persistante du MVP.
Le canal concret retenu pour le MVP est défini par la décision 37.

20. Maximum de deux tentatives de notification sans réponse

Lorsqu'un client ne répond pas à une notification, le système peut effectuer au maximum deux tentatives.
notificationAttempts représente le nombre total de tentatives d'envoi de notification effectuées pour le ticket pendant la journée opérationnelle.
Une tentative est comptabilisée lorsque le système déclenche effectivement l'envoi via le mécanisme de notification prévu.
Le compteur n'est pas réinitialisé lors d'une réinsertion.
Une réponse du client produit l'état métier approprié, notamment :

CONFIRMED

DELAYED

Aucune troisième tentative d'envoi n'est autorisée lorsque :
notificationAttempts = 2
La confirmation de l'envoi par le fournisseur externe et la confirmation de la réception par le client ne constituent pas des règles métier distinctes du Ticket dans le MVP.

21. CONFIRMED ne signifie pas présence physique

L'état CONFIRMED signifie que le client a répondu à la notification.
Il ne constitue pas une preuve de présence physique.
La présence physique est vérifiée séparément par le prestataire.
Les transitions principales sont :
WAITING → IN_SERVICE
si le client est présent.
CONFIRMED → IN_SERVICE
si le client est présent.
CONFIRMED → SKIPPED
si le client n'est pas présent.

22. Les responsabilités métier restent séparées

Les responsabilités de calcul et de traitement définies dans le modèle comportemental restent des responsabilités distinctes.
Une modification d'une responsabilité ne doit pas implicitement modifier la responsabilité d'une autre.
Les appels entre ces responsabilités doivent être explicites dans les cas d'utilisation concernés.

23. Les états terminaux ne peuvent pas être annulés

Un ticket dans un état terminal ne peut pas être annulé.
La transition suivante n'est pas autorisée :
IN_SERVICE → CANCELLED
dans le parcours métier normal.
Les états suivants sont considérés comme terminaux :

COMPLETED

CANCELLED

NO_SHOW

24. Fermeture des tickets en fin de journée

À la fin de la journée opérationnelle, les tickets encore actifs parmi les états concernés sont clôturés selon la règle métier définie.
Les états concernés sont :

WAITING

NOTIFIED

CONFIRMED

DELAYED

SKIPPED

Ils passent alors à :
NO_SHOW
selon le traitement de fin de journée.
Les tickets déjà :

COMPLETED

CANCELLED

NO_SHOW

ne sont pas concernés par ce traitement.

25. Catalogue de services structuré et association spécifique au prestataire

Le système utilise un catalogue structuré de services associé aux domaines d'activité.
Un Service représente un type de service identifiable et structuré.
Un ServiceProvider sélectionne les services qu'il propose.
La relation entre un prestataire et un service est représentée par une classe d'association :
ServiceProviderService
Cette classe permet notamment de stocker :
defaultDurationMinutes
propre à la combinaison :
ServiceProvider + Service
Ainsi, deux prestataires peuvent proposer le même service avec des durées par défaut différentes.
Exemple :
Prestataire A + Coupe de cheveux → 30 minutes Prestataire B + Coupe de cheveux → 45 minutes
Le système privilégie des services proposés dans le catalogue correspondant au domaine d'activité.
Le mécanisme précis permettant à une organisation ou à un prestataire d'ajouter un nouveau service reste à définir lorsqu'un cas d'utilisation concret le nécessitera.
Les données saisies doivent être soumises aux règles métier et aux validations techniques appropriées.
La durée doit notamment respecter les contraintes définies par le système et ne peut pas être une valeur arbitraire ou invalide.

26. Customer et contexte Organization

Le Customer représente une identité client globale.
Cependant, les données métier et les informations spécifiques d'un client sont rattachées au contexte de l'Organization concernée.
La relation Customer–Organization constitue donc un contexte propre à chaque organisation et peut porter ses propres informations.
Une organisation ne peut pas accéder automatiquement aux données spécifiques d'un Customer appartenant au contexte d'une autre organisation.
L'accès aux données est contrôlé par les mécanismes de Role et de Permission.
Principe :
Identité globale ≠ Données métier globales
Dans le MVP actuel, le contexte explicite CustomerOrganisation n'est pas implémenté tant qu'un cas d'utilisation concret ne le rend pas nécessaire.
Il reste une extension architecturale prévue pour les besoins futurs tels que :

historique client ;

données métier propres à une organisation ;

informations contextuelles propres au client dans une organisation.

27. Épuisement des tentatives de notification

Le nombre maximal de notifications envoyées à un ticket pendant la journée opérationnelle est de deux.

Première absence de réponse

NOTIFIED ↓ SKIPPED ↓ Réinsertion ↓ WAITING
La réinsertion est effectuée si les conditions normales de réinsertion sont satisfaites.

Deuxième absence de réponse

NOTIFIED ↓ SKIPPED
Aucune nouvelle réinsertion n'est effectuée.
Le ticket reste SKIPPED jusqu'au traitement de fin de journée et ne peut plus redevenir candidat opérationnel par une nouvelle notification.
Cela empêche une boucle infinie du type :
WAITING → NOTIFIED → SKIPPED → WAITING → NOTIFIED → ...
après épuisement des deux tentatives.
Si le client répond à la deuxième notification :

CONFIRMED reste éligible à la sélection ;

DELAYED suit les règles définies par la décision 28.

28. Comportement de DELAYED après épuisement des notifications

Le nombre maximal de notifications envoyées à un ticket reste fixé à deux pendant la journée opérationnelle.
Lorsque le client répond à la deuxième notification en indiquant qu'il est en retard :
NOTIFIED → DELAYED
Le ticket peut être réinséré selon les règles normales de réinsertion.
Après la réinsertion :
DELAYED → WAITING
notificationAttempts reste égal à 2 et n'est pas réinitialisé.
Lorsque ce ticket redevient candidat, aucune troisième notification n'est envoyée.
Le prestataire peut sélectionner directement le ticket selon les règles normales de sélection du prochain ticket, puis vérifier sa présence physique avant le passage à :
WAITING → IN_SERVICE
Cette règle distingue le cas DELAYED du cas d'absence totale de réponse après la deuxième notification.
Ainsi :
Deuxième notification sans réponse → SKIPPED → aucune nouvelle réinsertion
alors que :
Deuxième notification avec réponse DELAYED → DELAYED → réinsertion → WAITING → sélection directe → vérification de présence → IN_SERVICE
Aucun troisième envoi de notification n'est autorisé.
Le nombre de notifications limite les notifications envoyées, et non la possibilité pour un client ayant répondu de recevoir finalement sa prestation.

29. Règles d'annulation d'un Ticket

Un Ticket peut être annulé tant que sa prestation n'a pas commencé.
Les transitions d'annulation autorisées sont :
SCHEDULED → CANCELLED WAITING → CANCELLED NOTIFIED → CANCELLED CONFIRMED → CANCELLED DELAYED → CANCELLED SKIPPED → CANCELLED
La transition suivante n'est pas autorisée :
IN_SERVICE → CANCELLED
Les états terminaux suivants ne peuvent pas être annulés :
COMPLETED NO_SHOW CANCELLED

Autorisations

Le Customer peut annuler uniquement ses propres tickets, tant que ceux-ci ne sont pas IN_SERVICE.
Le ServiceProvider peut annuler les tickets de sa propre Queue avant le début de la prestation.
Un Receptionist ou un autre membre de l'organisation peut annuler un ticket uniquement si son Role possède la Permission correspondante.
La permission effective dépend donc du système de Role et de Permission, et non uniquement du nom ou du type structurel du membre.

Effet métier

L'annulation met immédiatement fin au parcours opérationnel du ticket.
Un ticket CANCELLED :

n'est plus candidat à la sélection ;

ne peut plus être réinséré ;

ne reçoit plus de notification ;

ne peut plus revenir à un état opérationnel.

L'annulation est donc une transition terminale.

30. Relation entre Membership et ServiceProvider

ServiceProvider représente la capacité opérationnelle d'un membre à fournir des services au sein d'une Organization.
Il ne doit pas être identifié directement uniquement par une relation :
User → ServiceProvider
car cette relation ne permet pas de représenter clairement le contexte organisationnel.
La relation retenue est :
User └── 0..* Membership ├── 1 Organization └── 0..1 ServiceProvider
Une Membership peut donc être associée à au plus un ServiceProvider.

Invariant

Un ServiceProvider doit être associé à une Membership appartenant à la même Organization.
Ainsi :

Membership représente l'appartenance du User à une Organization ;

Role et Permission représentent l'autorisation dans cette organisation ;

ServiceProvider représente la capacité opérationnelle de fournir des services ;

ServiceProvider possède sa propre Queue, ses services proposés et ses disponibilités.

Principe :
User ↓ Membership ↓ Organization ↓ ServiceProvider
Role ne détermine donc pas l'existence d'un ServiceProvider.
Un utilisateur peut avoir un rôle donné sans être ServiceProvider.
Un ServiceProvider peut également posséder un ou plusieurs rôles selon les permissions qui lui sont attribuées.

31. Extension du système par domaine

Le système est conçu pour pouvoir supporter plusieurs domaines d'activité sans modifier le noyau métier pour chaque nouveau domaine.
Domaine est un concept métier générique et non une simple énumération codée en dur.
Une Organization peut utiliser un ou plusieurs domaines à travers le concept :
OrganisationDomaine
Un même Domaine peut être utilisé par plusieurs organisations.
Les informations spécifiques à un domaine ne doivent pas être introduites directement sous forme de champs codés en dur dans les classes centrales du noyau lorsque ces informations ne sont pas nécessaires au fonctionnement général de la plateforme.
Le noyau doit rester indépendant de domaines particuliers tels que :

coiffure ;

médecine ;

dentisterie ;

garage.

L'ajout d'un nouveau domaine doit donc être principalement une opération de configuration et de données, et non une modification systématique du code métier central.

32. Extension future : CustomField / CustomFieldValue

Le système de champs personnalisés est considéré comme une extension future.
Il ne fait pas partie du noyau MVP tant qu'un besoin métier concret ne justifie pas son implémentation.
Lorsqu'il sera implémenté :

CustomField représentera la définition d'un champ ;

CustomFieldValue représentera la valeur d'un champ pour un contexte client donné ;

les types de champs seront contrôlés ;

un client pourra renseigner des valeurs mais ne pourra pas créer librement des définitions de champs ;

la définition et la modification des champs seront soumises aux permissions appropriées ;

les champs ne devront pas exécuter de logique métier arbitraire ;

les règles métier complexes d'un domaine devront être traitées par des modules métier contrôlés et non par un système de champs personnalisés.

Le stockage et le modèle physique exacts de CustomFieldValue restent à décider lorsque cette extension sera réellement nécessaire.

33. Stack technique

Le backend du MVP sera développé avec :
Java + Spring Boot
La base de données principale sera :
PostgreSQL
Le frontend restera séparé du backend et communiquera avec celui-ci via une REST API.
Ce choix vise notamment à permettre l'apprentissage et l'application concrète de :

conception backend ;

REST API ;

transactions ;

concurrence ;

sécurité ;

tests ;

persistence ;

architecture modulaire.

Le choix précis des technologies frontend sera traité séparément pendant la conception technique du frontend.

34. Style d'architecture

Le backend adoptera une architecture :
Modular Monolith
Les modules seront séparés selon les responsabilités métier.
Le projet appliquera les principes de :
Hexagonal Architecture
afin de limiter la dépendance du domaine et de la logique applicative envers :

Spring Boot ;

PostgreSQL ;

les mécanismes de persistence ;

les technologies externes.

Les Ports représentent les contrats nécessaires aux interactions avec l'extérieur.
Les Adapters implémentent ces contrats pour les technologies concrètes.
Les Microservices sont explicitement hors périmètre du MVP.
Cette décision ne signifie pas que tous les principes de l'architecture hexagonale doivent être appliqués de manière mécanique à chaque classe.
Leur utilisation doit rester proportionnée aux besoins réels du projet.

35. Stratégie de persistence

JPA/Hibernate sera utilisé comme mécanisme principal de persistence.
PostgreSQL sera utilisé comme système de gestion de base de données.
Les requêtes SQL natives pourront être utilisées ponctuellement lorsqu'un besoin technique clair le justifie, notamment pour :

certaines requêtes complexes ;

des besoins de performance ;

certains mécanismes de verrouillage ;

des opérations nécessitant un contrôle SQL précis.

Le modèle du domaine ne sera pas conçu pour satisfaire les contraintes de l'ORM.
Le choix entre ORM et SQL direct sera donc guidé par les responsabilités et les besoins techniques réels du cas d'utilisation.

36. Authentication et Authorization

L'authentification du MVP reposera principalement sur :
Phone + OTP
L'autorisation reposera sur le modèle :
User ↓ Membership ↓ Role ↓ Permission
L'API REST utilisera une authentification basée sur des tokens.
Les OTP devront être protégés notamment par :

une expiration ;

un nombre limité de tentatives ;

une limitation des demandes d'envoi ;

l'absence de stockage en clair.

Le fournisseur SMS concret utilisé pour l'OTP et les détails techniques précis du mécanisme de token restent des détails d'implémentation à préciser pendant la conception technique.

37. Canal de notification du MVP : SMS

Le canal de notification retenu pour le MVP est :
SMS
Ce choix est cohérent avec le modèle actuel basé sur le numéro de téléphone du client et ne nécessite pas la présence d'une application mobile cliente.
Le domaine et les cas d'utilisation ne doivent cependant pas dépendre directement d'un fournisseur SMS particulier.
L'architecture utilise donc un port de notification conceptuel :
Application ↓ NotificationPort ↓ SMS Adapter ↓ SMS Provider
Le fournisseur SMS concret sera choisi pendant la conception technique.
Le remplacement futur du SMS par un autre canal, tel que WhatsApp, ne doit pas nécessiter de modifier la logique métier centrale de la Queue.
Le canal de notification reste donc une responsabilité d'infrastructure/adaptation et non une responsabilité du domaine Ticket.

38. Stratégie d'identifiant de Ticket

Ticket.id sera de type logique :
Long
Il constitue la clé primaire interne du Ticket.
L'identifiant sera généré par le mécanisme de persistence retenu avec PostgreSQL/JPA.
Le Ticket.id sert notamment de critère secondaire de départage lorsque plusieurs rendez-vous possèdent le même createdAt.
Il ne remplace jamais createdAt comme critère principal d'ordre pour l'activation des rendez-vous.
Le Ticket.id n'est pas considéré comme un mécanisme de sécurité.
L'autorisation d'accès à un ticket doit toujours être contrôlée par les règles d'authentification, d'autorisation et de contexte organisationnel.
Aucun ticketNumber destiné à l'affichage utilisateur n'est ajouté au MVP sans cas d'utilisation concret le justifiant.
Un identifiant d'affichage distinct pourra être introduit ultérieurement si le produit nécessite un numéro lisible par les clients ou les prestataires.

39. Gestion des périodes d'indisponibilité conflictuelles

Une nouvelle UnavailabilityPeriod ne doit pas créer silencieusement un conflit avec des rendez-vous déjà planifiés.
Lorsqu'une période d'indisponibilité demandée entre en conflit avec des tickets planifiés existants, notamment des tickets SCHEDULED, la création ou la modification de cette période doit être refusée dans le parcours normal du MVP.
Le système ne doit pas automatiquement :

annuler les rendez-vous concernés ;

les déplacer ;

les transformer vers un nouvel état CONFLICTED.

Aucun état CONFLICTED n'est ajouté au Ticket pour résoudre ce problème dans le MVP.
Les tickets opérationnels tels que :

WAITING ;

NOTIFIED ;

CONFIRMED ;

IN_SERVICE

ne doivent pas être interrompus simplement par la création d'une nouvelle période d'indisponibilité.
La gestion d'une fermeture opérationnelle du prestataire relève du statut opérationnel et des règles correspondantes.
Cette décision permet d'éviter qu'une modification de disponibilité produise silencieusement des effets secondaires importants sur des tickets déjà existants.

40. Premier service de la journée

L'activation matinale des rendez-vous ne démarre pas automatiquement le premier service.
Après confirmation de présence du ServiceProvider et activation des rendez-vous éligibles, le premier passage à IN_SERVICE nécessite une action explicite du prestataire pour sélectionner le prochain ticket.
La sélection respecte les règles normales de priorité et de position.
La présence physique du client doit être vérifiée avant le passage à :
IN_SERVICE
Le mécanisme de début de journée est donc distinct du mécanisme de démarrage d'une prestation.

41. Transactions et concurrence

Les opérations métier qui modifient l'état du système ou des invariants critiques doivent être exécutées dans une transaction adaptée au cas d'utilisation.
Les transactions sont définies au niveau des Business Use Cases et non arbitrairement sur chaque méthode technique.

Queue opérationnelle

La Queue constitue la ressource principale de coordination pour les opérations qui modifient l'ordre opérationnel ou sélectionnent un ticket.
Les opérations critiques de Queue utilisent un contrôle de concurrence approprié, notamment un verrouillage pessimiste lorsque la cohérence de l'ordre courant l'exige.
Cela concerne notamment :

l'ajout d'un ticket à la Queue ;

la réinsertion ;

l'activation des rendez-vous dans la Queue ;

la sélection du prochain ticket ;

le démarrage d'une prestation ;

la combinaison fin de prestation + sélection/démarrage du suivant.

Le verrouillage doit être court et ne doit pas être conservé pendant un appel réseau externe.

Capacité des rendez-vous

La capacité de réservation ne repose pas sur la Queue opérationnelle.
La synchronisation nécessaire à une réservation est réalisée sur le contexte de planification pertinent, notamment :
ServiceProvider + Date
La vérification de capacité et la création de la réservation doivent être protégées contre les opérations concurrentes afin d'éviter un dépassement de capacité.
Le mécanisme physique exact de verrouillage sera défini lors de l'implémentation sans introduire inutilement une nouvelle entité métier de type DailyPlanning.

Isolation

Le niveau d'isolation par défaut retenu est :
READ COMMITTED
Il n'est pas prévu d'utiliser SERIALIZABLE globalement pour l'ensemble du système.
Un niveau d'isolation différent pourra être utilisé pour un cas d'utilisation spécifique uniquement si un besoin technique clairement identifié le justifie.

Optimistic locking

Le verrouillage optimiste peut être utilisé pour des ressources ou opérations où il est plus approprié.
Il ne constitue pas l'unique stratégie de concurrence de la Queue.

Contraintes de base de données

Les invariants critiques doivent être protégés autant que possible par des contraintes ou index de base de données en complément de la logique applicative.
Ces contraintes constituent une protection supplémentaire et ne remplacent pas les règles métier.

Appels externes

Les appels vers des services externes, notamment le fournisseur SMS, ne doivent pas maintenir un verrou de base de données ouvert pendant toute la durée de l'appel réseau.
La transaction de modification de l'état métier et l'appel externe sont donc traités comme deux responsabilités techniques distinctes.

Conflits et deadlocks

Les conflits de concurrence et deadlocks éventuels doivent être traités par la couche technique selon une stratégie contrôlée.
Un retry peut être utilisé lorsqu'il est techniquement sûr de réexécuter l'opération concernée.
Le retry ne constitue pas une règle métier.

Principe général de conception

Le projet privilégie :

un noyau métier stable ;

des responsabilités clairement séparées ;

des règles métier explicites ;

des données calculées lorsqu'elles peuvent être recalculées de manière fiable ;

des extensions génériques uniquement lorsqu'elles répondent à un besoin réel ;

l'utilisation de patterns éprouvés plutôt que l'invention de mécanismes inutilement complexes ;

la limitation du MVP aux concepts nécessaires à ses cas d'utilisation réels ;

une architecture technique proportionnée à la complexité réelle du domaine.

Toute extension future doit être évaluée selon :

son besoin métier ;

son coût de complexité ;

son impact sur le noyau existant ;

sa nécessité réelle pour le MVP ou une évolution future.

Règle de modification des décisions

Une décision acceptée ne doit jamais être modifiée silencieusement.
Si une décision doit changer :

l'ancienne décision est marquée SUPERSEDED ;

une nouvelle décision est créée ;

la raison du changement est documentée ;

les documents concernés sont synchronisés ;

les diagrammes concernés sont vérifiés ;

le code et les tests concernés sont vérifiés.

Aucune décision métier ou architecturale importante ne doit être introduite silencieusement pendant l'implémentation.

Règle générale de collaboration avec l'IA

L'IA peut proposer une solution.
L'humain valide la décision.
Le dépôt Git conserve la décision et son historique.
Principe :

L'IA peut proposer. L'humain valide. Le dépôt Git conserve la décision.

Les agents d'implémentation ne doivent pas modifier silencieusement une règle métier ou une décision architecturale validée.
Toute contradiction découverte pendant l'implémentation doit être signalée avant modification de la décision concernée.

Principe de cohérence documentaire

Les documents du projet doivent rester synchronisés.
Lorsqu'une décision modifie le comportement du système, les éléments concernés doivent être vérifiés, notamment :
DECISIONS.md CURRENT_STATE.md PROJECT_CONTEXT.md QUESTIONS.md UML Code Tests
Les diagrammes ne doivent pas contredire les décisions validées.
Le code ne doit pas introduire un comportement différent de celui documenté sans décision explicite.

État de validation

Les décisions 1 à 41 sont considérées comme acceptées pour le modèle métier et l'architecture actuels.
Les extensions suivantes restent volontairement hors du noyau MVP :

CustomerOrganisation ;

CustomField ;

CustomFieldValue ;

règles métier complexes spécifiques à un domaine ;

fonctionnalités multi-ressources avancées ;

autres extensions non justifiées par un cas d'utilisation concret.

QUESTIONS.md ne contient actuellement plus de question ouverte bloquante.
Les détails techniques nécessaires à l'implémentation, lorsqu'ils n'ont pas encore fait l'objet d'une décision, seront traités pendant la phase de conception technique et ne doivent pas être considérés comme des décisions implicites du présent document.
Fin des décisions actuellement validées.