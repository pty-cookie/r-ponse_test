# Stratégie de test : audit qualité de QuickOrder V3.1

| | |
|---|---|
| Équipe | chérine el abassi |
| Version testée | QuickOrder 3.1.0, commit `v1` |
| Période de test | 12/10/2026 |
| Objet | Dire si l'application peut passer en production |

## 1. Contexte et objectif

QuickOrder est une plateforme de livraison de repas. Un client cherche un restaurant, remplit un panier, applique un coupon, choisit une adresse et un moyen de paiement, puis suit sa commande. Un administrateur fait avancer les statuts. Une API REST permet de faire une partie de ces opérations sans passer par l'interface.

Nous n'avons pas écrit cette application et nous ne la corrigeons pas. Notre rôle est celui d'une équipe QA juste avant la mise en production : trouver ce qui ne marche pas, le prouver, mesurer ce que cela coûterait à un client ou à l'entreprise, puis recommander **Go** ou **No-Go**.

Deux règles guident toute la campagne :

- **Aucune modification du code source** pendant l'audit. Les corrections éventuelles seront proposées plus tard, dans une branche séparée.
- **Une anomalie n'existe que si elle est prouvée** : capture, requête et réponse, ou sortie console. Une intuition ne suffit pas.

## 2. Périmètre

### 2.1 Dans le périmètre

| Zone | Ce que nous testons |
|---|---|
| Compte | Inscription, connexion, déconnexion, option « se souvenir de moi », profil, adresses (ajout, défaut, suppression) |
| Catalogue | Liste des restaurants, recherche, filtre par catégorie, filtre végétarien, filtre de frais de livraison, trois tris (note, prix de livraison, délai), page restaurant, état ouvert ou fermé |
| Panier | Ajout, modification de quantité, suppression, vidage, règle « un seul restaurant par panier » |
| Coupons | Validité, expiration, minimum de commande, plafond de réduction, cohérence avec l'offre affichée |
| Commande | Checkout, choix de l'adresse, note de livraison, total, confirmation |
| Paiement simulé | Carte, PayPal, espèces (montant remis, monnaie) |
| Suivi | Liste et détail des commandes, filtre par statut, annulation, « commander à nouveau » |
| Avis et favoris | Dépôt d'un avis, favoris |
| Administration | Tableau de bord, changement de statut d'une commande, contrôle d'accès |
| API REST | Les 11 endpoints décrits dans `openapi.yaml`, clé API, codes HTTP, contenu JSON, règles métier côté API |
| Sécurité de base | Contrôle d'accès, identité derrière les appels API, accès entre comptes, redirections, sessions |
| Utilisabilité | Trois scénarios réalisés par au moins deux personnes extérieures à l'équipe |

### 2.2 Hors périmètre

| Exclu | Pourquoi |
|---|---|
| Test de charge et de performance | Base SQLite locale, serveur de développement : les mesures ne diraient rien de la production |
| Test d'intrusion approfondi | Nous faisons des vérifications de sécurité de base, pas un pentest |
| Vrais paiements | Le paiement est simulé, aucun prestataire n'est branché |
| Compatibilité large navigateurs et mobiles | Un navigateur principal, un second en vérification ponctuelle |
| Accessibilité complète (RGAA, WCAG) | Seules les évidences sont notées en passant |
| Infrastructure Docker et déploiement | Nous nous servons de Docker pour lancer l'application, pas pour l'auditer |
| Emails, notifications, suivi de livreur | Fonctions absentes de l'application |
| Revue de code | L'audit est une boîte noire : comportement observé uniquement |

## 3. Parcours critiques

Un parcours est critique quand son échec fait perdre de l'argent, expose des données ou empêche de commander.

| ID | Parcours | Pourquoi il est critique |
|---|---|---|
| P1 | Commande complète via l'interface : recherche, menu, panier, coupon, checkout, paiement, confirmation | C'est la raison d'être du produit |
| P2 | Paiement : carte, PayPal, espèces | Erreur de montant ou double débit : perte d'argent et de confiance |
| P3 | Coupons et calcul du total | Chaque commande passe par ce calcul |
| P4 | Commande via l'API | Second point d'entrée, doit appliquer les mêmes règles que l'interface |
| P5 | Identité et accès : qui peut voir, créer ou annuler quelle commande | Données personnelles et responsabilité légale |
| P6 | Disponibilité : restaurant ouvert, produit en stock, minimum de commande | Une commande acceptée mais impossible à honorer coûte cher |
| P7 | Cycle de vie d'une commande : statuts, annulation, conséquences | Cohérence entre ce que voit le client, le restaurant et la caisse |
| P8 | Création de compte et première commande d'un nouvel utilisateur | Premier contact, l'échec ici perd un client pour toujours |

Les parcours secondaires (avis, favoris, profil, recherche et filtres, administration) sont testés, mais avec moins de profondeur.

## 4. Approche de test

### 4.1 Types de tests

| Type | Description | Outils |
|---|---|---|
| Fonctionnel UI | Parcours complets et fonctions isolées dans le navigateur | Navigateur, DevTools (Réseau, Console, Application) |
| Fonctionnel API | Chaque endpoint : cas nominal, validations, codes HTTP, structure JSON | Swagger UI, curl, Postman (collection fournie) |
| Règles métier | Coupons, minimum de commande, stock, statuts, calcul des montants, vérifiés à la main sur un calcul indépendant | Tableur ou calculatrice |
| Limites | Valeurs aux bornes : quantité 0, 1, 20, 21, 99, minimum juste atteint, coupon à un centime du seuil | Navigateur, Swagger |
| Négatifs et erreurs | Champs vides, formats invalides, identifiants inexistants, clé absente ou fausse, doubles envois | Navigateur, curl |
| Sécurité de base | Identité derrière l'API, accès à la commande d'un autre compte, accès admin, redirections | curl sans session, deux comptes, navigation privée |
| Exploratoire | Sessions de 30 minutes avec une mission écrite, pour trouver ce que les cas écrits n'ont pas prévu | Navigateur |
| Tests utilisateurs | 3 scénarios, au moins 2 testeurs, observation et chronométrage | Grille de `user-feedback.md` |
| Non-régression de base | Suite `pytest` fournie, lancée telle quelle | `pytest -q` |

### 4.2 Répartition des 24 cas de test

La consigne impose 24 cas répartis en quatre familles de 6. Nous les répartissons pour couvrir en priorité les parcours critiques.

| Famille | Nombre | Rôle |
|---|---|---|
| Nominaux | 6 | Vérifier que le chemin normal fonctionne |
| Limites | 6 | Vérifier le comportement aux bornes |
| Erreurs | 6 | Vérifier le refus propre des entrées invalides |
| Règles métier | 6 | Vérifier que les règles annoncées sont appliquées |

Chaque cas porte un ID `TC-01` à `TC-24`, un parcours (P1 à P8), un risque (voir `risk-matrix.md`) et, si une anomalie est trouvée, un identifiant `BUG-XX`. Le détail est dans `test-cases.xlsx`.

Il y aura des cas à interface web et des cas à interface API. Chaque parcours critique P1 à P6 est couvert par au moins un cas, et P2, P3, P5 et P6 par au moins deux, sur l'interface comme sur l'API.

### 4.3 Données et comptes

| Élément | Valeur |
|---|---|
| Clients | `alice@example.com` et `bob@example.com`, mot de passe `Password123!` |
| Administrateur | `admin@quickorder.local`, mot de passe `Admin123!` |
| Clé API | `quickorder-demo-key` |
| Restaurants | Cinq restaurants ouverts et un restaurant marqué fermé, à utiliser pour les cas de disponibilité |
| Produits | Certains produits sont à stock nul ou faible : à repérer pendant l'exploration |
| Coupons | `WELCOME10`, `TOKYO30`, `ATLAS15`, plus des codes à identifier pendant l'exploration (un coupon expiré par exemple) |

Avant chaque série de tests, la base est remise à l'état initial (`python seed.py`, ou `docker compose down -v` puis relance). Cela garantit que le stock et l'historique des commandes sont identiques d'un testeur à l'autre. Les comptes de test supplémentaires sont créés par l'inscription et notés dans `execution-report.md`.

## 5. Priorisation risque × impact

Le détail complet est dans `risk-matrix.md`. Voici la méthode et le résultat attendu.

**Probabilité** (que l'anomalie existe ou se produise) :
1. Faible : fonctionnalité simple et peu modifiée
2. Moyenne : règle métier avec plusieurs conditions
3. Élevée : calcul d'argent, doubles chemins (interface et API), logique d'état

**Impact** (conséquence si elle se produit) :
1. Mineur : gêne, contournement évident
2. Important : mauvaise expérience, erreur de montant ponctuelle, support sollicité
3. Critique : perte financière, fuite ou usage abusif de données personnelles, commande impossible à honorer

**Score = probabilité × impact**, de 1 à 9 :

| Score | Niveau | Conséquence pour les tests |
|---|---|---|
| 6 à 9 | Critique | Testé en premier, plusieurs cas, interface et API |
| 3 à 5 | Élevé | Au moins un cas dédié |
| 1 à 2 | Modéré | Couvert par un cas général ou par l'exploration |

Risques attendus en tête de classement, avant exécution :

1. Identité derrière l'API : les opérations sont-elles liées à un utilisateur authentifié ?
2. Paiement : montant espèces, double commande, cohérence du total
3. Disponibilité : restaurant fermé, stock, minimum de commande
4. Coupons : écart entre l'offre annoncée et le calcul, confiance accordée au client
5. Statuts de commande : transitions autorisées, conséquences d'une annulation
6. Première commande d'un nouvel utilisateur

Cette liste est une hypothèse de départ. L'exécution peut la modifier, et `risk-matrix.md` aura une colonne pour le constat réel.

## 6. Classification des anomalies

### Sévérité (effet de l'anomalie)

| Niveau | Définition |
|---|---|
| Critique | Perte financière, accès non autorisé à des données ou à des actions, commande impossible à honorer |
| Majeure | Fonction centrale incorrecte, avec contournement difficile ou erreur de montant |
| Mineure | Fonction secondaire incorrecte, ou incohérence d'affichage sans conséquence financière |
| Cosmétique | Libellé, alignement, détail visuel |

### Priorité de correction (ordre de traitement)

| Niveau | Définition |
|---|---|
| P1 | À corriger avant toute mise en production |
| P2 | À corriger dans la première version suivante |
| P3 | À planifier |
| P4 | À traiter si le temps le permet |

La sévérité décrit ce qui se passe. La priorité décrit quand le corriger. Les deux peuvent différer : une anomalie mineure sur une page très vue peut être plus urgente qu'une majeure sur un cas rare.

### Statuts d'un test

| Statut | Sens |
|---|---|
| Pass | Résultat conforme à l'attendu |
| Fail | Résultat différent de l'attendu, anomalie rédigée |
| Blocked | Test impossible à cause d'un autre défaut ou d'un problème d'environnement, cause notée |
| Skipped | Test volontairement non exécuté, raison notée |

## 7. Critères d'entrée, de sortie et de décision

### Critères d'entrée

- L'application démarre et la vérification rapide d'`INSTALLATION.md` (partie D) est passée.
- Les trois comptes de démonstration se connectent.
- La base est dans son état initial.
- La stratégie, la matrice de risques et les 24 cas sont rédigés.

### Critères de sortie

- Les 24 cas sont exécutés, ou leur non-exécution est justifiée (Blocked ou Skipped avec motif).
- Chaque Fail est rattaché à un bug reproduit trois fois, avec preuve.
- Les 3 scénarios utilisateurs sont passés par au moins 2 personnes.
- Les totaux Pass, Fail, Blocked et Skipped sont identiques dans `test-cases.xlsx`, `execution-report.md` et `bug-reports/README.md`.

### Règle de décision Go / No-Go

La recommandation suit une règle écrite à l'avance, pour qu'elle ne dépende pas de l'humeur du moment.

| Condition constatée | Décision |
|---|---|
| Au moins une anomalie **critique** ouverte | **No-Go** |
| Aucune critique, mais une anomalie **majeure** sur un parcours P1 à P4 | **No-Go**, sauf contournement documenté et accepté |
| Aucune critique, majeures limitées aux parcours secondaires, mineures nombreuses | **Go conditionnel**, avec liste de correctifs datés |
| Aucune critique ni majeure | **Go** |

La décision et son argumentaire figurent dans `execution-report.md`. Nous ne la fixons pas ici : elle dépend des résultats.

## 8. Environnement

| Élément | Valeur |
|---|---|
| Lancement | [Docker Compose ou Python venv] |
| Système | [Windows 11] |
| Navigateur principal | [Opera] |
| Navigateur secondaire | [Chrome] |
| URL application | `http://127.0.0.1:5000` |
| URL Swagger | `http://127.0.0.1:5000/docs/` |
| Outils | DevTools, Swagger UI, curl, Postman ou Bruno, pytest |

## 9. Organisation

| Rôle | Responsable | Tâches |
|---|---|---|
| Pilotage et rapport d'exécution | [chérine] | Planning, consolidation des chiffres, Go / No-Go |
| Tests UI | [chérine] | Cas à interface web, captures |
| Tests API et sécurité | [chérine] | Cas API, requêtes brutes |
| Tests utilisateurs | [chérine] | Recrutement, protocole, observation |

Calendrier prévu :

| Étape | Date |
|---|---|
| Exploration libre | [12/10/2026] |
| Exécution des cas | [12/10/2026] |
| Tests utilisateurs | [12/10/2026] |
| Rapport et décision | [12/10/2026] |

## 10. Preuves et traçabilité

- Les preuves sont dans `evidence/`, nommées selon le cas ou le bug : `TC-07_ui_01.png`, `BUG-03_curl.txt`, `pytest_baseline.txt`, `UT-A1_S2.png`.
- Une capture doit montrer l'URL, le message ou le montant concerné.
- Une requête API est sauvegardée en entier : méthode, URL, en-têtes utiles, corps envoyé, code HTTP, corps reçu.
- Chaque bug cite le ou les cas de test qui l'ont révélé, et chaque cas en échec cite son bug.

## 11. Risques du projet de test et hypothèses

| Risque ou hypothèse | Réponse prévue |
|---|---|
| L'état de la base change au fil des tests (stock, commandes) | Remise à zéro avant chaque série, état noté dans le rapport |
| Un bug en bloque d'autres | Statut Blocked avec le bug en cause, contournement tenté |
| Peu de testeurs disponibles | Minimum de 2 personnes, profils différents (habitué des applis de livraison, utilisateur occasionnel) |
| Les cas écrits ne trouvent pas tout | Sessions exploratoires chronométrées, anomalies rattachées à un nouveau cas ou ajoutées en note |
| Les résultats d'un seul passage sont trompeurs | Chaque bug est rejoué trois fois |
| Le comportement attendu n'est pas toujours documenté | Quand la documentation se tait, nous nous référons à la pratique courante des plateformes de livraison, et nous le signalons dans le cas concerné |
