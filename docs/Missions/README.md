# Missions — Barber Queue Management System

Dernière mise à jour : 24/11/2026

## Règles
1. `A-faire/` contient **uniquement** les missions non terminées. `Terminees/` contient **uniquement** les missions **implémentées et vérifiées**.
2. Une mission passe en `Terminees/` seulement si : le code ou le livrable existe, les critères d'acceptation sont vérifiés, les tests ont été exécutés et le résultat est noté. Planifier, documenter ou proposer ne suffit pas.
3. Pour déplacer une mission, utiliser `templates/mission-terminee.md` (ce qui a été réalisé, fichiers, tests, résultat, date).
4. Une mission de documentation (M00-xx) n'est **pas** une fonctionnalité développée.
5. Une mission ne démarre que si ses dépendances sont terminées.

## État actuel
- Missions terminées et vérifiées : **0**.
- Missions à faire : **29** (6 de conception + 23 de développement).

## Ordre logique et priorités

| Ordre | ID | Titre | Priorité | Dépend de |
|---|---|---|---|---|
| 1 | M00-01 | Valider et compiler les diagrammes | Critique | — |
| 2 | M00-02 | Règles métier (04) | Critique | M00-01 |
| 3 | M00-03 | State machines (05) | Critique | M00-01, M00-02 |
| 4 | M00-04 | Architecture et modules (06) | Haute | M00-01 à 03 |
| 5 | M00-05 | Points ouverts et marché | Haute | — |
| 6 | M00-06 | Journal des décisions | Haute | — |
| 7 | M01 | Init backend | Critique | M00-01, M00-04 |
| 8 | M02 | Init frontend | Haute | M00-01 |
| 9 | M03 | Contrats API | Haute | M00-02, M00-03 |
| 10 | M04 | Auth et Users | Critique | M01, M03 |
| 11 | M05 | Audit Log | Haute | M01, M04 |
| 12 | M06 | Salons, memberships, permissions | Critique | M04, M05 |
| 13 | M07 | Services | Haute | M06 |
| 14 | M08 | Horaires et disponibilité | Haute | M06 |
| 15 | M09 | Propriété du salon | Haute | M06, M05 |
| 16 | M10 | File : cœur | Critique | M06, M07, M08, M05 |
| 17 | M11 | Rendez-vous et ClientDayPolicy | Critique | M07, M08, M10 |
| 18 | M12 | ETA et conflits file / rendez-vous | Haute | M10, M11, M08 |
| 19 | M13 | File : opérations du personnel | Haute | M10, M12, M05 |
| 20 | M14 | Délégation QUEUE_MANAGER | Moyenne | M06, M13 |
| 21 | M15 | Notifications et scheduler | Haute | M04, M10, M12 |
| 22 | M16 | Historique et statistiques | Moyenne | M10, M11, M00-05 |
| 23 | M17 | Frontend : authentification | Haute | M02, M03, M04 |
| 24 | M18 | Frontend : file côté client | Haute | M17, M10, M12, M15 |
| 25 | M19 | Frontend : rendez-vous côté client | Moyenne | M17, M11 |
| 26 | M20 | Frontend : console du personnel | Haute | M17, M13, M14, M11 |
| 27 | M21 | Frontend : administration | Moyenne | M17, M06 à M09, M14 |
| 28 | M22 | Tests transversaux et sécurité | Haute | M04 à M16 |
| 29 | M23 | Fournisseur SMS (conditionnel) | Basse | M04, M00-05 |

**Parallélisme possible :** après M03, le backend (M04 à M16) et le frontend (M17 à M21) peuvent avancer en parallèle dans les deux conversations, en respectant les contrats API de M03.
