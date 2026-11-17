# ARD AL KHAIR — Audit initial et architecture proposée

## État des lieux au 17 novembre 2026

Sur DQ1, le notebook `notebooks/01_audit_geographique_datasets.ipynb` contient 315 cellules. Il couvre la qualité, la couverture spatio-temporelle et le rapprochement des grilles ERA5-Land et SMAP, notamment via KD-tree. Ses sorties sauvegardées comprennent des erreurs de lecture et d'exécution. Le notebook dépend de chemins propres à un Mac et des cellules écrivent des exports à côté de leurs entrées. Les sorties historiques n'ont pas été reproduites pour cet audit.

Dans le dossier Drive référencé, les noms de fichiers confirment la présence des CSV ERA5-Land 2020–2022, SMAP 2020–2025 et de dérivés de rapprochement pour 2020. Leur intégrité, leur provenance et leurs licences restent à qualifier. Les répertoires locaux de données et de modèles sont vides. PlantVillage, PlantDoc et PlantWild sont annoncés, avec emplacements, versions et contenus à confirmer. Aucun poids, pipeline d'inférence, rapport d'évaluation ou entraînement reproductible n'a été identifié sur DQ1.

Les dossiers app, src et docker sont vides. Le fichier requirements/requirements-mac.txt fige un environnement exploratoire Jupyter et scientifique. Aucun frontend Angular, service backend, schéma SQL, fournisseur d'identité ou Kafka configuré n'a été identifié. Le dépôt local n'avait ni commit ni remote configuré. Le dépôt GitHub cible est public.

## Architecture proposée

- **Angular et passerelle HTTP** — interface parcelles, demandes d'analyse, résultats et historique. Passerelle REST OpenAPI, contrôle des accès, émission d'URL de téléversement temporaires, réponses 202 avec identifiant de tâche et lecture du statut.
- **Identité** — fournisseur OIDC, par exemple Keycloak, avec Authorization Code et PKCE. Rôles exploitant, agronome et administrateur. Vérifier l'appartenance à l'exploitation et à la parcelle côté service. Identités techniques distinctes et ACL Kafka.
- **Service parcelles** — exploite PostgreSQL/PostGIS pour les géométries, cultures, sols et paramètres d'irrigation, avec contrôle des droits par exploitation.
- **Service données agricoles** — lit ERA5-Land et SMAP, vérifie qualité et unités, aligne dans le temps et l'espace, publie des instantanés horodatés et versionnés. Prévisions opérationnelles et mesures terrain entrent via des adaptateurs dédiés. Les sources historiques servent à la validation et à la préparation de variables.
- **Service maladies (Python)** — vérifie les images, applique le prétraitement versionné, infère et enregistre prédictions, scores et abstentions.
- **Service irrigation (Python)** — charge les variables d'une parcelle à un instant donné. Compare le modèle à une règle agronomique de référence et propose une lame d'eau et une fenêtre. Les contraintes de débit, volume et fraîcheur des entrées bornent le résultat. Le premier pilote requiert une validation humaine.
- **Service recommandations** — assemble les résultats indépendants, gère les tâches et conserve les décisions et leur justification. Aucun résultat de maladie ne déclenche une action sur une pompe. Un usage futur de ce résultat comme variable d'irrigation exige un contrat et une validation de sa pertinence.
- **Infrastructure** — PostgreSQL partagé en développement avec schémas, rôles et migrations par service. Stockage objet compatible S3 pour images, instantanés et poids. Registre de modèles versionné avec empreintes, provenance, métriques, environnement et statut de validation. Kafka pour les échanges asynchrones. Docker Compose pour le développement, télémétrie et métriques sur tâches, délais, erreurs et abstentions.

## Contrats de communication à valider

Les API publiques sont versionnées sous /api/v1. Kafka transporte des références et métadonnées. La passerelle gère l'autorisation des lectures et des écritures par parcelle. Le producteur conserve un journal transactionnel des événements à publier. Les consommateurs acceptent une livraison au moins une fois et dédupliquent par event_id. Les erreurs transitoires donnent lieu à des tentatives bornées et les échecs persistants à une file d'erreurs. Un rejeu conserve la clé d'idempotence et une trace de la tentative.

Enveloppe JSON Schema versionnée — event_id, event_type, schema_version, occurred_at en UTC, correlation_id, job_id, tenant_id et plot_id. Clé Kafka par exploitation et parcelle lorsque l'ordre par parcelle est nécessaire. Les changements compatibles ajoutent des champs facultatifs. Les ruptures passent par une version majeure. OpenAPI, AsyncAPI, schémas et exemples sont stockés ensemble dans le dépôt.

- **agri.features.ready.v1** — données vers irrigation. snapshot_id, object_ref, empreinte, as_of, available_at, fenêtre temporelle, version du jeu de variables, produits et versions source, méthode de correspondance, distance et indicateurs qualité. Unités documentées, dont précipitations en mm et humidité volumique en m³/m³. Des données trop anciennes, indisponibles ou hors domaine entraînent une abstention.
- **disease.requested.v1 → disease.completed.v1** — orchestration et service maladies. Entrée image_ref, empreinte, MIME et culture si connue. Sortie status, classes normalisées, scores associés, code d'abstention et versions du modèle, du prétraitement et de la taxonomie. Un score est présenté comme une probabilité uniquement après calibration validée.
- **irrigation.requested.v1 → irrigation.completed.v1** — orchestration et service irrigation. Entrée snapshot_id, as_of, horizon, paramètres de culture et de sol et contraintes hydrauliques versionnées. Sortie status, irrigation_mm, fenêtre UTC, indicateurs d'incertitude si validés, qualité des entrées, raisons d'abstention et versions du modèle et des règles.
- **Échec et abstention** — chaque tâche terminale expose succeeded, abstained ou failed, avec un code exploitable et sans détail interne sensible. Le service recommandations conserve les références aux entrées et aux résultats pour expliquer chaque décision.

## Feuille de route et critères de sortie

1. **Valider le cadrage.** Confirmer le périmètre du pilote, les rôles, les cultures et parcelles ciblées, les responsables des données et les contrats proposés.
2. **Inventorier et évaluer l'existant.** Manifeste en lecture seule avec version, licence, couverture, schéma, empreinte et emplacement par source. Retrouver le code, les poids, le prétraitement et les jeux d'évaluation d'origine. Consigner les performances déclarées et leur provenance. Avant tout réentraînement, évaluer les poids retrouvés sur des partitions indépendantes ou documenter précisément les pièces manquantes.
3. **Démarrer le socle reproductible.** Docker Compose et images Linux épinglées pour arm64 et amd64. Développement CPU sur Apple M1 et Windows x64 via Docker Desktop/WSL2. Vérifier séparément les dépendances sur Windows ARM si nécessaire. Test de démarrage sur les deux machines cibles, migrations, connexion OIDC, refus d'accès inter-exploitations et aller-retour Kafka avec déduplication sur une petite fixture synthétique. Monter les originaux en lecture seule et écrire les dérivés dans un volume distinct.
4. **Livrer le parcours maladies.** Téléverser une image depuis Angular, obtenir et tracer un résultat ou une abstention. Évaluer macro-F1, rappel par classe et latence avec séparation des doublons et évaluation indépendante sur photos de terrain. Fixer les seuils avec le référent agronome.
5. **Livrer le parcours irrigation.** Produire une recommandation à partir d'un instantané avec contrôle des dates de disponibilité et évaluation par parcelle et période. Comparer à la règle de référence sur erreur, abstentions et respect des contraintes. Vérifier l'expiration des recommandations et faire valider les résultats par un agronome.
6. **Préparer le pilote.** Documenter installation, jeux de test, supervision, restauration et retour à une version précédente des modèles. Enregistrer le code, les décisions et les rapports publiables dans le dépôt GitHub.

## Informations à récupérer

- Données — versions exactes, licences et droits de redistribution, méthodes d'extraction, unités, résolution, dates, indicateurs qualité, provenance des fichiers dérivés et emplacements PlantVillage, PlantDoc et PlantWild.
- Modèles — propriétaires, poids et empreintes, framework, plateformes supportées, hyperparamètres, taxonomie, règles de prétraitement, splits, métriques et rapports existants.
- Terrain — parcelles, cultures et stades, sols, capacité hydraulique, capteurs, observations d'humidité et volumes réellement appliqués, source de prévisions, validation agronomique et tolérance aux erreurs.
- Exploitation — cible d'hébergement et budget, exigences de disponibilité, conservation des images et coordonnées, profils utilisateurs, accès aux sources et machines de test. Les données originales, poids soumis à licence et secrets restent dans des emplacements à accès contrôlé.
