# EfficientNetB0 V3 — Audit de la partition PlantVillage

## Résultats historiques vérifiés

Les 8 146 prédictions enregistrées correspondent aux chemins et étiquettes de la partition de test `plantvillage_split_v2.npz`. Le recalcul retrouve 8 044 réponses correctes, une exactitude de 98,7479 %, un F1 macro de 98,3263 % et une matrice de confusion identique à celle archivée. Les 8 146 références de test sont présentes dans le répertoire `raw/color` du ZIP PlantVillage.

## Chevauchement entre partitions

Le rapprochement de `leaf-map.json` avec le nom des images retire le préfixe UUID et l'extension, passe en minuscules et conserve les identifiants de feuille dont la classe correspond au dossier de l'image. Il identifie 4 068 groupes de feuilles communs à l'entraînement et au test, concernant 5 985 images de test sur 8 146 (73,47 %). Parmi les autres images, 96 ont un groupe également présent en validation, 11 ont un groupe identifié absent des deux partitions et 2 054 restent sans correspondance exploitable. Les 11 images couvrent quatre classes seulement. L'exactitude historique mesure ce lot de test et ne permet pas d'estimer la généralisation à des feuilles indépendantes.

La comparaison des octets dans l'archive confirme aussi 6 images de test identiques à des images d'entraînement et 1 image de test identique à une image de validation. La comparaison a présélectionné les paires de même taille et CRC avant comparaison intégrale.

## Artéfacts

Les archives Keras V3 `best` et `latest` ont des métadonnées d'enregistrement distinctes. Les octets de leur fichier interne `model.weights.h5` ont le même SHA-256, `9977339139f27cba228eaedb1a87f558a28a5a1219b2c8ec70be73eb23b5fb62`. Le modèle utilisé pour l'essai est `v3_best.keras`, SHA-256 `020f5864e8dd7301ef09a7b4d072a79218f9b57e8374bb1c54c2ef117eb13c3f`. Il attend une image RGB de 224 × 224 et applique une remise à l'échelle interne de 1/255.

## Inférence complète sur CPU

Le test a exécuté `v3_best` sur les 8 146 images de la partition d'origine sur CPU Apple Silicon avec Python 3.13.2, TensorFlow 2.21.0, Keras 3.13.2 et NumPy 2.5.5. Les JPEG ont été décodés en RGB et redimensionnés à 224 × 224, en conservant l'échelle 0–255 attendue par le modèle. Le choix du redimensionnement a été fait sur 160 images de validation tirées sans remise avec `numpy.random.default_rng(42)`. Résultats sur cet échantillon de validation, bilinéaire 159/160, plus proche voisin 160/160, bicubique 159/160. Le plus proche voisin a ensuite été appliqué à tout le test. Les images ont été lues dans l'ordre des entrées du ZIP et les prédictions remises dans l'ordre de la partition avant comparaison.

- Test complet — 8 006 prédictions correctes sur 8 146, soit **98,2814 % d'exactitude** et **97,7865 % de F1 macro**.
- Comparaison avec l'historique — 8 052 classes identiques et **94 divergences**. L'écart absolu moyen entre les scores maximaux est de 0,01311.
- Le prétraitement historique exact reste introuvable. L'écart observé ne permet pas d'en isoler la cause. La partition conserve le chevauchement par feuille décrit ci-dessus et un test sur des images indépendantes reste nécessaire.

Empreintes d'entrée SHA-256, hors modèle déjà documenté :

- `plantvillage_split_v2.npz` — `c5dc6ee7703cfec34b1997dd660fa9afd61772841a057dea43eae76da188bbb4`
- `test_predictions_v3.csv` — `9fb4279e39c2094200e20492d2af04d0cde814ab95fb7b751723cf6582653800`
- `class_names_v2.json` — `16925c0cb74cd5be219eb9e8cf87f0fa4f5c255f49b8db7c865c55050cb7d307`


## Priorités d'évaluation

1. Constituer un jeu de test indépendant de l'entraînement V3, avec provenance et groupes par feuille, et mesurer la performance par classe et les abstentions. Pour le modèle V3 déjà entraîné, une nouvelle partition du même corpus peut contenir des images déjà vues. Réserver des photos de terrain dont l'indépendance peut être établie.
2. Pour une future version, définir les partitions par groupes avant l'entraînement, éliminer les doublons exacts et examiner les quasi-doublons. Conserver le manifeste et l'ordre de classes avec le modèle.
