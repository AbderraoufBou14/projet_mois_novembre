Nous allons préparer le passage à une nouvelle session de développement.

À partir de tes deux rapports et de tes investigations dans le dépôt, crée un fichier `CONTEXTE_DEV_DF6DFM75.md` servant de document de passation technique.

Il doit contenir :
- L'état actuel du moteur `raw_vault_tests` et les principales dépendances.
- La matrice vérifiée des contrôles existants et manquants.
- Les conclusions sur les constructions HK et les particularités MAS/HUB.
- Les contraintes de compatibilité KSH, `controls.py`, packaging et CLI.
- L'architecture cible recommandée, avec modularisation raisonnable.
- La stratégie de partitionnement non destructif des tables d'audit.
- Les optimisations Spark candidates, sans présenter les gains non mesurés comme acquis.
- Les tests unitaires, TNR et validations d'intégration nécessaires.
- Les décisions prises, les points ouverts et les risques.
- Un plan de développement incrémental visant une livraison en RECETTE le 20/11/2026.

Ajoute une dernière section précisant les informations applicatives SI3/DF6 qui pourraient manquer pour réaliser correctement les contrôles de comparaison source → Raw Vault.

Le document doit être compact, structuré et exploitable dans une nouvelle session. Cite les fichiers et fonctions réels plutôt que de recopier le code.

Ne modifie aucun fichier applicatif, ne lance aucun traitement Spark et n'effectue aucun déploiement. La seule écriture autorisée est la création du document Markdown de passation.

À la fin, indique son emplacement et résume les points restant à arbitrer.