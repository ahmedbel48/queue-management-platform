# 02 — Problématique et solution

> **Avertissement important.** Aucune enquête de terrain n'a été réalisée pour ce projet. Les constats ci-dessous sont des **hypothèses de travail** (marquées `HP-xx`) issues de la conception du produit et de la littérature des éditeurs de logiciels (voir `03-market-and-competitor-analysis.md`). Elles doivent être confirmées par des entretiens avec des barbiers et des clients (mission `M00-05`).

## 1. Le problème

Dans un salon qui accueille des clients **sans rendez-vous**, l'ordre de passage repose souvent sur la mémoire du barbier, une liste papier ou l'ordre d'arrivée dans la salle d'attente.

| # | Hypothèse | Statut |
|---|---|---|
| HP-01 | Le client ne sait pas combien de temps il va attendre et reste sur place « au cas où ». | À valider |
| HP-02 | L'ordre de passage peut être contesté (qui est arrivé avant qui). | À valider. Un éditeur de logiciel de réservation affirme qu'une file numérique réduit ces litiges (source S13 dans le document 03, affirmation d'un fournisseur). |
| HP-03 | Les rendez-vous et les clients sans rendez-vous se gênent : le barbier arbitre à la main. | À valider |
| HP-04 | Un client absent bloque la file ou fait perdre du temps au barbier. | À valider |
| HP-05 | Dans un salon à plusieurs personnes, la gestion de l'ordre se fait de façon informelle (qui s'occupe de la file ? qui a le droit de modifier les prix ?). | À valider |

## 2. Causes possibles (hypothèses)
- Pas d'outil simple adapté au fonctionnement « file + rendez-vous » d'un barbier (à comparer avec l'étude concurrentielle).
- Durées de service variables, donc estimation difficile sans outil.
- Clients qui n'ont pas de compte ou pas d'adresse e-mail (le numéro de téléphone est souvent le seul contact : hypothèse à valider).

## 3. Difficultés côté client (hypothèses)
Attendre sans visibilité, ne pas pouvoir faire autre chose pendant l'attente, hésiter entre venir tôt ou tard, subir un ordre de passage perçu comme injuste.

## 4. Difficultés côté salon (hypothèses)
Gérer le flux sans interruption, arbitrer file et rendez-vous, gérer les absences, partager la gestion entre plusieurs personnes avec des droits différents, garder une trace des changements importants.

## 5. La solution proposée

| Problème (hypothèse) | Réponse de l'application (décisions validées) |
|---|---|
| Pas de visibilité sur l'attente (HP-01) | Ticket fixe, position et estimation d'attente (ETA) visibles par le client ; notification quand l'ETA devient inférieure ou égale au délai du salon (R3). |
| Ordre contesté (HP-02) | `ticket_number` immuable, ordre de file défini par des règles écrites (D5, Q1). Un client dépassé (`SKIPPED`) garde sa priorité. |
| File et rendez-vous en conflit (HP-03) | Un client ne peut pas rejoindre la file si son service chevauche un rendez-vous confirmé (D1). Rendez-vous confirmés automatiquement et protégés contre les conflits (Q9). |
| Absences (HP-04) | Délai de présence configurable, passage automatique en `SKIPPED` puis décision du barbier (`NO_SHOW` manuel). L'absence de réaction ne bloque pas la file (D8). |
| Gestion partagée (HP-05) | Rôles par salon, délégation `QUEUE_MANAGER`, propriétaire principal et secondaires, journal d'audit (D6, D11, Q7, Q10, Q12). |
| Clients sans l'application | Le barbier retrouve ou crée un compte (statut `PENDING_VERIFICATION`) et ajoute le client à la file (Q2). |

## 6. Limites de la solution
- Une notification push n'est pas garantie (téléphone éteint, notifications refusées). L'état réel reste consultable dans l'application (D8).
- Un client sans e-mail reste `PENDING_VERIFICATION` tant que la vérification par SMS n'est pas disponible (Q8) : il peut être ajouté à la file mais ne reçoit pas de notification push.
- Le MVP ne mesure pas encore l'impact réel : voir indicateurs à définir dans `01-product-vision.md`.
