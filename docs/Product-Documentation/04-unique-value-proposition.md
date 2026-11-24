# 04 — Proposition de valeur

> **Règle de ce document :** nous n'affirmons pas que l'application est « unique ». L'étude (`03-...`) montre que la file numérique existe déjà ailleurs. Les éléments ci-dessous sont des **différenciateurs candidats**, classés selon leur preuve.

## 1. Promesse (proposition de valeur de travail)

- **Pour le client :** « Je sais quand c'est mon tour, et je suis prévenu à temps. »
- **Pour le barbier :** « Mon ordre de passage est clair, même avec des rendez-vous et des clients sans application. »
- **Pour le propriétaire :** « Je garde la maîtrise de mon salon, de mes règles et de mes droits d'accès. »

*Formulation à valider auprès d'utilisateurs réels avant toute communication externe.*

## 2. Avantages concrets du produit (ce que le MVP fera, d'après les décisions validées)

| Avantage | Mécanisme validé | Preuve de la différence ? |
|---|---|---|
| Une file **par barbier**, avec ticket fixe et position | D5, Q1 | Non démontrée (existe sous d'autres formes chez les concurrents) |
| File et rendez-vous **cohabitent** sans chevauchement | D1, Q9, R1 | Non démontrée (S1–S3 présentent des fonctions comparables) |
| Un client en retard garde sa **priorité** (`SKIPPED`, FIFO), la décision finale reste au barbier | Q1, Q4 | Non vérifiée chez les concurrents |
| Un barbier peut ajouter un client **sans application**, qui n'est jamais anonyme (compte `PENDING_VERIFICATION`, jamais de mot de passe connu du barbier) | Q2 | Non vérifiée |
| **Rôles par salon** : propriétaire principal / secondaires, barbier, staff, gestionnaire de file délégué par file | D6, D11, Q7, Q10, Q12 | Non vérifiée |
| **Vie privée** : recherche de client limitée (30/h), résultat masqué, journal sans contact en clair | Q11, H6 | Non vérifiée |
| **Un seul engagement actif par client et par jour** (pas de réservations fantômes dans plusieurs salons) | Q13, H5 | Choix de conception ; effet à valider |
| Délai d'annulation de rendez-vous **figé à la réservation** | Q14, H4 | Non vérifiée |
| Vérification du contact par **e-mail ou SMS** (SMS selon étude des coûts) | Q8, D12 | Moyen, pas un différenciateur en soi |

## 3. Différenciateurs candidats à tester

| # | Hypothèse | Comment la vérifier |
|---|---|---|
| DV-01 | Un focus **Maroc** (usage du téléphone comme identifiant, fuseau, langues) est un avantage face aux solutions internationales. | Entretiens ; test des concurrents sur le marché marocain ; langues de l'interface à confirmer. |
| DV-02 | La **simplicité** (pas de caisse ni de marketing dans le MVP) convient aux petits salons. | Pilotes avec 1 à 3 salons. |
| DV-03 | La règle de **priorité conservée** des clients dépassés est jugée juste par les clients et les barbiers. | Tests d'usage. |
| DV-04 | La **délégation** à un réceptionniste répond à un besoin réel. | Entretiens avec des salons de plus d'un employé. |

## 4. Ce que nous ne pouvons pas affirmer aujourd'hui
- Que l'application est « la seule » ou « la première » à proposer une file numérique : **faux d'après les sources**.
- Que les clients marocains préfèrent notre solution : **aucune donnée**.
- Des gains chiffrés (temps gagné, réduction des absences) : **aucune mesure**. Les chiffres cités par des éditeurs concurrents ne s'appliquent pas à notre produit.

## 5. Ce que le MVP ne fait pas (pour éviter toute promesse excessive)
Paiement, avis, fidélité, marketing, caisse et commissions, report de rendez-vous, SMS en production (selon étude), suivi des absences répétées.
