# EfficientNetB0 V3 — Audit de la partition PlantVillage

## Résultats historiques vérifiés

Les 8 146 prédictions enregistrées correspondent aux chemins et étiquettes de la partition de test `plantvillage_split_v2.npz`. Le recalcul retrouve 8 044 réponses correctes, une exactitude de 98,7479 %, un F1 macro de 98,3263 % et une matrice de confusion identique à celle archivée. Les 8 146 références de test sont présentes dans le répertoire `raw/color` du ZIP PlantVillage.

## Chevauchement entre partitions

Le rapprochement de `leaf-map.json` avec le nom des images retire le préfixe UUID et l'extension, passe en minuscules et conserve les identifiants de feuille dont la classe correspond au dossier de l'image. Il identifie 4 068 groupes de feuilles communs à l'entraînement et au test, concernant 5 985 images de test sur 8 146 (73,47 %). Parmi les autres images, 96 ont un groupe également présent en validation, 11 ont un groupe identifié absent des deux partitions et 2 054 restent sans correspondance exploitable. Les 11 images couvrent quatre classes seulement. L'exactitude historique mesure ce lot de test et ne permet pas d'estimer la généralisation à des feuilles indépendantes.

La comparaison des octets dans l'archive confirme aussi 6 images de test identiques à des images d'entraînement et 1 image de test identique à une image de validation. La comparaison a présélectionné les paires de même taille et CRC avant comparaison intégrale.

## Artéfacts

Les archives Keras V3 `best` et `latest` ont des métadonnées d'enregistrement distinctes. Les octets de leur fichier interne `model.weights.h5` ont le même SHA-256, `9977339139f27cba228eaedb1a87f558a28a5a1219b2c8ec70be73eb23b5fb62`. Le modèle utilisé pour l'essai est `v3_best.keras`, SHA-256 `020f5864e8dd7301ef09a7b4d072a79218f9b57e8374bb1c54c2ef117eb13c3f`. Il attend une image RGB de 224 × 224 et applique une remise à l'échelle interne de 1/255.

## Essai d'inférence

Le chargement sur CPU Apple Silicon a réussi avec Python 3.13.2, TensorFlow 2.21.0 et Keras 3.13.2. Un essai technique sur huit images disponibles de validation, avec décodage JPEG RGB et redimensionnement bilinéaire, a produit 38 scores finis par image et huit classes correctes. Cet essai vérifie l'exécution du modèle. Le prétraitement historique exact reste à retrouver. Le premier passage complet a rencontré une erreur de lecture d'entrée ZIP pendant l'étape de validation. Les métriques ci-dessus proviennent des prédictions historiques.

## Priorités d'évaluation

1. Constituer un jeu de test indépendant de l'entraînement V3, avec provenance et groupes par feuille, et mesurer la performance par classe et les abstentions. Pour le modèle V3 déjà entraîné, une nouvelle partition du même corpus peut contenir des images déjà vues. Réserver des photos de terrain dont l'indépendance peut être établie.
2. Pour une future version, définir les partitions par groupes avant l'entraînement, éliminer les doublons exacts et examiner les quasi-doublons. Conserver le manifeste et l'ordre de classes avec le modèle.
