# Matrice risque × impact : QuickOrder V3.1

Ce document traduit la méthode de `test-strategy.md` (section 5) en une liste de risques chiffrés. Il sert à décider où mettre l'effort de test en premier. Il est rédigé **avant** l'exécution : les scores sont des hypothèses de départ, et les deux dernières colonnes sont à remplir après la campagne.

## 1. Échelles utilisées

**Probabilité** : 1 faible, 2 moyenne, 3 élevée.
**Impact** : 1 mineur, 2 important, 3 critique.
**Score** = probabilité × impact.

| Score | Niveau |
|---|---|
| 6 à 9 | Critique : testé en premier, interface et API |
| 3 à 5 | Élevé : au moins un cas dédié |
| 1 à 2 | Modéré : couvert par un cas général ou l'exploration |

Pour fixer la probabilité, nous avons regardé trois choses : la présence d'un calcul d'argent, l'existence de deux chemins pour la même opération (interface et API), et le nombre de conditions que la règle doit gérer. Pour l'impact, nous nous sommes demandé ce qui arrive au client, à l'entreprise ou aux données si le risque se réalise.

## 2. Vue d'ensemble

Chaque cellule contient les identifiants des risques concernés.

| Probabilité ↓ / Impact → | 1 Mineur | 2 Important | 3 Critique |
|---|---|---|---|
| **3 Élevée** | R-17 | R-07, R-11, R-13 | **R-01, R-02** |
| **2 Moyenne** | R-18, R-24 | R-06, R-08, R-09, R-16, R-19, R-20, R-21, R-22, R-23 | **R-03, R-04, R-05, R-10, R-12** |
| **1 Faible** | | | R-14, R-15 |

La zone en haut à droite concentre l'essentiel de l'attention : deux risques à 9, cinq à 6 en impact critique.

## 3. Registre des risques

Les lignes sont triées par score décroissant, puis par numéro.

| ID | Risque | Parcours | P | I | Score | Niveau | Justification du score |
|---|---|---|---|---|---|---|---|
| R-01 | Les opérations de l'API ne sont pas liées à un utilisateur authentifié : un appelant muni de la seule clé peut lire, créer ou annuler des commandes d'un client | P4, P5 | 3 | 3 | **9** | Critique | La clé est affichée dans la documentation et dans Swagger. Si l'identité n'est pas vérifiée, toutes les commandes d'un compte sont exposées. Donnée personnelle et acte financier |
| R-02 | Paiement en espèces : un montant remis inférieur au total, nul ou négatif est accepté, la monnaie calculée est incohérente | P2 | 3 | 3 | **9** | Critique | Calcul d'argent, deux chemins (interface et API), champ libre saisi par l'utilisateur. Perte sèche à chaque commande concernée |
| R-03 | Une commande payée par PayPal est créée plusieurs fois ou débitée plusieurs fois | P2 | 2 | 3 | **6** | Critique | Modes de paiement traités différemment. Un doublon fait payer ou livrer deux fois |
| R-04 | Une commande est acceptée pour un restaurant fermé | P6 | 2 | 3 | **6** | Critique | L'état ouvert ou fermé est affiché, mais rien ne garantit qu'il est contrôlé à la commande. Commande impossible à honorer |
| R-05 | Une commande est acceptée alors que le stock est nul ou insuffisant, le stock peut devenir négatif | P6 | 2 | 3 | **6** | Critique | Contrôle à faire à plusieurs moments (ajout au panier, modification, paiement) et sur deux chemins. Commande impossible à honorer |
| R-10 | Le total est faux : frais de livraison, réduction ou arrondi mal combinés | P3 | 2 | 3 | **6** | Critique | Chaque commande passe par ce calcul. Une erreur d'un centime se répète sur toutes les commandes |
| R-12 | L'annulation d'une commande n'a pas les conséquences attendues : statut de paiement, remboursement, stock | P7 | 2 | 3 | **6** | Critique | Action irréversible côté client. Incohérence entre ce que voit le client et la caisse |
| R-07 | Un coupon annoncé (texte promotionnel, page restaurant) donne une autre réduction que celle affichée | P3 | 3 | 2 | **6** | Critique | Plusieurs coupons, plusieurs surfaces d'affichage. L'écart est visible pour le client et crée un litige |
| R-11 | Les changements de statut admin permettent des transitions absurdes (par exemple revenir en arrière après livraison) | P7 | 3 | 2 | **6** | Critique | Logique d'état, peu de contrôles attendus sur une action d'administration. Historique faussé |
| R-13 | Un nouvel utilisateur ne peut pas passer sa première commande proprement : adresse vide ou invalide, parcours d'inscription ambigu | P8 | 3 | 2 | **6** | Critique | Étape obligatoire, tous les nouveaux clients passent par là. Client perdu dès le premier contact |
| R-06 | Le minimum de commande affiché n'est pas appliqué | P6 | 2 | 2 | 4 | Élevé | Règle affichée dans l'interface. Perte de marge pour le restaurant |
| R-08 | L'API de validation de coupon accepte un sous-total fourni par l'appelant | P3, P4 | 2 | 2 | 4 | Élevé | Entrée contrôlée par le client. Risque de fausser le calcul de remise, impact limité si la commande recalcule le total |
| R-09 | Minimum d'achat, plafond de réduction ou expiration d'un coupon mal appliqués | P3 | 2 | 2 | 4 | Élevé | Trois conditions à combiner. Remise trop généreuse ou refusée à tort |
| R-16 | Redirection après connexion ou déconnexion non maîtrisée, session mal gérée | P5 | 2 | 2 | 4 | Élevé | Paramètres d'URL contrôlés par l'extérieur |
| R-19 | L'API accepte ou refuse mal les entrées : quantités, types, champs absents, JSON invalide | P4 | 2 | 2 | 4 | Élevé | Chaque endpoint a ses propres validations. Les codes HTTP doivent rester cohérents |
| R-20 | L'interface et l'API n'appliquent pas les mêmes règles pour la même opération | P1, P4 | 2 | 2 | 4 | Élevé | Deux chemins développés séparément |
| R-21 | Le parcours de commande est difficile à utiliser pour un nouvel utilisateur | P1 | 2 | 2 | 4 | Élevé | Mesuré par les tests utilisateurs, pas par les cas écrits |
| R-22 | Le panier se comporte mal : quantités, suppression d'un article, règle d'un seul restaurant, panier conservé | P1 | 2 | 2 | 4 | Élevé | Fonction centrale, état conservé en session |
| R-23 | Une erreur serveur expose des informations techniques ou un état interne | P4, P5 | 2 | 2 | 4 | Élevé | À vérifier en provoquant des erreurs volontairement |
| R-14 | Un client accède à la commande d'un autre depuis l'interface web | P5 | 1 | 3 | 3 | Élevé | Impact fort, mais un contrôle explicite semble présent. À confirmer |
| R-15 | Un client non admin accède à l'administration | P5 | 1 | 3 | 3 | Élevé | Même raisonnement : impact fort, probabilité faible à confirmer |
| R-17 | La recherche et les filtres renvoient des résultats incohérents avec ce que l'utilisateur demande | P1 | 3 | 1 | 3 | Élevé | Plusieurs critères combinables, mais peu de conséquence financière |
| R-18 | Avis et favoris : dépôt possible sans condition, affichage incomplet, doublons | P1 | 2 | 1 | 2 | Modéré | Fonctions secondaires |
| R-24 | « Commander à nouveau » reprend un panier incohérent avec l'état actuel (stock, restaurant) | P7 | 2 | 1 | 2 | Modéré | Fonction de confort, peu utilisée en test |

## 4. Ce que cette matrice implique pour la campagne

1. Les deux risques à 9 (R-01 et R-02) sont testés en premier, **sur l'interface et sur l'API**, avec plusieurs cas chacun.
2. Les neuf autres risques critiques sont couverts par au moins un cas de test dédié, deux quand le risque existe aussi côté API.
3. Les risques de niveau Élevé reçoivent un cas dédié ou sont couverts par l'exploration.
4. Les risques de niveau Modéré sont vérifiés pendant les sessions exploratoires.
5. Les tests utilisateurs ciblent surtout R-13, R-21 et R-22.

Les égalités à 6 ne sont pas classées entre elles : ce sont des risques de même rang, et l'ordre de test dépend de la facilité de mise en place.

## 5. Correspondance avec les cas de test

Cette colonne sera renseignée dès que les 24 cas de `test-cases.xlsx` seront rédigés, pour que les identifiants soient cohérents d'un fichier à l'autre.

| Risque | Cas de test prévus |
|---|---|
| R-01 | [À RENSEIGNER avec test-cases.xlsx] |
| R-02 | [À RENSEIGNER avec test-cases.xlsx] |
| ... | ... |

Règle de couverture à vérifier à ce moment-là : chaque risque de niveau Critique doit apparaître dans au moins un cas, et chaque cas doit citer au moins un risque.


