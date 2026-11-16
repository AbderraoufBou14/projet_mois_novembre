# Contexte applicatif — SI3 / LAC → DF6 Raw Vault

> **Objet du document.** Fournir à un agent de développement le contexte applicatif utile à la conception et à la validation des contrôles automatisés de qualité du Raw Vault DF6. Ce document complète une passation technique consacrée au code de `raw_vault_tests` : il ne remplace ni les spécifications des tests, ni les JSON, ni les patterns exécutés.
>
> **Règle de fiabilité.** Les éléments ci-dessous décrivent les conventions et mécanismes documentés ou observés. Le comportement exact d'un traitement doit être vérifié dans **la configuration active, la version réellement exécutée du framework et le schéma physique**. Ne pas transformer une convention en règle universelle.

## 1. Parcours applicatif des données

```text
Applications sources (notamment SI3 ; autres provenances possibles)
       ↓
Zone LAC : tables d'entrée, parfois historisées ou incrémentales
       ↓
Patterns de chargement PySpark pilotés par JSON
       ↓
DF6 Raw Vault : HUB / LINK / SAT / MAS
       ↓
Business Vault / exposition / usages aval (selon les flux)
```

- **SI3 / LAC** : couche de données amont utilisée par de nombreux objets DF6. Certaines configurations déclarent d'autres applications sources ; ne pas supposer que toutes les sources viennent de SI3.
- **Raw Vault DF6** : couche d'intégration et d'historisation, modélisée avec les composants Data Vault.
- **Business Vault / exposition** : transformations et présentations en aval ; non nécessaires à la plupart des contrôles structurels du Raw Vault.
- **Environnements distincts** : les références Hive sont construites selon l'environnement et les paramètres de configuration. Exemples DEV : `db_dev_si3`, `db_dev_df6_brute`, `db_dev_df6_travail`.

**Ne pas confondre** la zone BRUTE amont de l'ingestion avec la base `df6_brute`, qui désigne ici le **Raw Vault**.

## 2. Objets métier, familles Vault et clés

Un **objet métier** (ex. `instrument`, `couru`) est un périmètre de configurations et de traitements. Ce n'est **pas** la même chose que sa table HUB.

| Famille | Préfixe | Rôle | Clés / particularités |
|---|---|---|---|
| HUB | `hrv_` | Identités métier | BK simple/composite et HK d'identité |
| LINK | `lrv_` | Relations entre identités HUB | HK de relation et HK des HUB participants ; CDK éventuelle |
| SAT | `srv_` | Attributs descriptifs historisés d'un parent HUB/LINK | HK de rattachement, HDIFF, dates techniques/fonctionnelles selon pattern |
| MAS | `mrv_` | Plusieurs occurrences descriptives actives pour un même parent | HK/clé de rattachement, discriminant(s), HDIFF et historisation |

- **BK** : clé métier, dont les composantes et l'ordre sont spécifiés par le mapping.
- **HK** : empreinte technique calculée selon les conventions de normalisation et l'algorithme effectifs ; ne pas présumer de l'algorithme de hash.
- **HDIFF** : empreinte des attributs descriptifs pertinents pour détecter une évolution ; différente d'une HK d'identité.
- **`ldts`** : date/heure technique de chargement ; **`dt_fonc`** : date fonctionnelle configurée ; **`rsrc`** : traçabilité de la source.
- **Discriminant MAS** : distingue des occurrences simultanées d'un même parent. Une **CDK** n'est pas automatiquement un discriminant MAS.

**Point critique pour les contrôles :** deux colonnes dont les noms commencent par `hk_` ne sont pas nécessairement comparables. Retrouver le parent, les composants de clé et la fonction de calcul dans le pattern et les JSON avant de construire une jointure.

## 3. Métadonnées JSON : source des mappings

Configurations documentées et observées :

```text
parm/raw/init/<objet_metier>/df6_raw_vault_{hub,link,sat,mas}.json
parm/raw/<objet_metier>/df6_raw_vault_{hub,link,sat,mas}.json
```

Le contenu réel varie selon l'objet : les quatre fichiers ne sont pas obligatoirement présents. Un JSON peut déclarer plusieurs tables dans `tables`.

Champs rencontrés, selon la famille et le niveau de configuration :

| Élément JSON | Utilité pour les contrôles |
|---|---|
| `appli_source`, `suffixe_base_source`, `src_table_name` | Résolution de la table source |
| `dest_table_name` | Nom de l'objet cible logique |
| `cles_src`, `src_bk_name`, `target_bk_name` | Construction et mapping des BK ; structure variable selon le pattern |
| `hubs` et ses clés associées | Identités parentes déclarées, notamment pour MAS/LINK |
| `bk_default_value`, `filtre` | Transformations / sélection à reproduire selon le traitement |
| `payload`, `champ_discriminant`, `cdk` | Description, granularité MAS et clés dépendantes |
| `dt_fonc`, `schema_partition` | Temporalité fonctionnelle et organisation des données |

**Attention :** un JSON indique ce qui est *configuré*, pas ce qui a effectivement été chargé. Vérifier également la résolution des bases, les colonnes Hive et le code des patterns de création/chargement.

## 4. INIT, récurrent et historisation

- Le dépôt distingue les configurations sous `parm/raw/init/` et sous `parm/raw/`. Le lanceur sélectionne l'une ou l'autre selon son mode d'exécution.
- Les exemples consultés utilisent parfois des sources suffixées `_delta_histo` en initialisation et `_delta` en récurrent, avec une cible Raw Vault commune. **Ces suffixes ne suffisent pas à déduire le contenu ou la fréquence réelle d'une table.**
- **CREATE** (structure) et **LOAD** (alimentation) sont deux opérations distinctes ; l'existence d'une table n'atteste pas le succès d'un chargement.
- Le Raw Vault est conçu pour conserver l'historique. Une même HK peut apparaître plusieurs fois dans un SAT ou un MAS sans constituer un doublon technique.
- Pour les **MAS v2**, la documentation décrit une comparaison de **l'état complet du groupe d'occurrences** (HK parent, discriminants, HDIFF). Une évolution du groupe peut entraîner l'insertion de toutes les occurrences de son nouvel état. Vérifier la variante réellement utilisée par chaque traitement.

## 5. Implications directes pour la qualité des données

Les contrôles sont **génériques**, mais leur applicabilité dépend des relations réellement configurées.

| Famille de contrôle | Ce qu'il faut vérifier avant de coder ou conclure |
|---|---|
| Source → HUB | BK transformées selon mapping, sources multiples, valeurs par défaut, filtre et cardinalité attendue |
| SAT → HUB | HK de rattachement au bon parent, règles de gestion des `NULL`, grain et métrique d'anomalie |
| MAS → HUB | HK parent réellement représentée dans le MAS ; cas mono-HUB, multi-HUB et CDK à distinguer |
| LINK → HUB | Colonnes HK des HUB référencés dans le LINK, et non HK de relation |
| LINK ↔ SAT / MAS | Relations effectivement déclarées ; chaque direction possède son propre sens fonctionnel |
| Source → SAT / MAS / LINK | Reproduire les transformations des patterns (HK, HDIFF, discriminants, filtres, temporalité), plutôt qu'une simple comparaison de colonnes homonymes |

**Exemple observé :** l'objet métier `instrument` possède un HUB `hrv_instrument` alimenté notamment par deux sources SI3 dont les BK d'origine sont `id` et `codevaleur`, mappées vers `id_instrument`. Des contrôles séparés ont été historisés par référence source/HUB. Une exigence de spécification évoque l'union des BK des sources : il s'agit d'un **point de conformité à arbitrer**, pas d'une correction automatiquement décidée.

**Interprétation des résultats :** `OK` = aucun écart détecté pour la règle et le périmètre effectivement exécutés ; `KO` = écart détecté ; `ERROR` = problème technique. Un job terminé avec succès ne prouve pas que tous les contrôles attendus ont été exécutés.

## 6. Consignes de vérification pour l'agent de développement

1. Identifier l'**objet métier**, la famille Vault, les sources et les relations pertinentes dans les JSON réellement sélectionnés.
2. Remonter du JSON au **pattern PySpark de chargement** et à ses fonctions de transformation partagées, dans **la version de package effectivement exécutée**. Les points d'entrée de création/chargement sont sous `src/raw/create/main.py` et `src/raw/load/main.py` ; le framework Vault peut être installé en `site-packages`.
3. Déterminer les **clés effectivement comparables**, le grain, les filtres, les règles de `NULL`, la date/partition concernée et la métrique exacte (`COUNT(*)`, clés distinctes, groupes, etc.).
4. Préparer des tests unitaires isolés **OK / KO / ERROR**, puis une vérification d'intégration DEV et une contre-vérification SQL indépendante lorsque possible.
5. Distinguer clairement **exigence de l'Excel**, **implémentation actuelle**, **comportement constaté** et **décision non encore validée**.
6. Ne pas modifier les données Raw Vault ni les tables d'audit existantes pour fabriquer un scénario KO. Les modifications de stockage et les migrations doivent préserver l'historique et faire l'objet d'une validation distincte.

> **Limite volontaire :** ce document ne fixe ni les algorithmes exacts de hash, ni la construction de la HK d'un MAS particulier, ni la fréquence des chargements, ni le périmètre contractuel de chaque contrôle. Ces détails doivent être établis à partir du code/configuration actifs et des spécifications accessibles dans le workspace.
