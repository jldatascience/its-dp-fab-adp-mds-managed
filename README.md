# its-dp-fab-adp-mds-managed — ADP MDS

Dépôt GitHub des items Microsoft Fabric du workspace **ADP MDS**, et de leur configuration de déploiement.

Ce dépôt ne contient **aucun code de déploiement**. Il décrit *quoi* déployer et *où*. La logique (contrôles, publication, vérification, rapport) se trouve dans le dépôt socle commun `its-dp-fab-cicd-common`, appelé par le tag `v1.4.0`.

| Rubrique | Valeur |
|---|---|
| Catégorie | `adp` (Data Product de type ADP) |
| Workspaces | `FR - DP - <ENV> - FAB - ADP - MDS - MANAGED` (DEV, ITG, PPR, PRD) |
| Branches | `develop` (par défaut) · `release/*` · `main` |
| Pipelines Azure DevOps | CI : `FabricCI` · CD : à créer depuis `CD.yml` · validation de PR : à créer depuis `PR-Validation.yml` |
| Socle commun | `its-dp-fab-cicd-common`, tag `v1.4.0` |
| Statut d'inventaire | `pending` : la CD refuse toute publication tant que l'inventaire n'est pas validé sur la branche réelle |

## Dépendances

**Prérequis de déploiement unique : le Framework** (`its-dp-fab-item-framework-managed`).

- 14 règles de `parameter.yml` pointent vers des pipelines du workspace Framework **de l'environnement cible**. Elles sont résolues par leur nom au moment du déploiement, sans aucun GUID en dur.
- Avant chaque déploiement, la CD vérifie que le Framework est **disponible** sur cet environnement :
  - le workspace Framework existe ;
  - les 5 pipelines Framework utilisés sont présents (pré-vol) ;
  - le dernier déploiement du Framework a réussi (porte Azure DevOps).
- La CD **n'exige jamais de redéployer le Framework** : s'il est déjà disponible, ADP MDS se déploie immédiatement.
- **Aucune dépendance vers les BDP ni vers les autres ADP** : ordre libre, déploiement possible en parallèle.

| Pipeline Framework utilisé | Appelé depuis |
|---|---|
| `pl_get_ws_lh_ids` | `pl_master_load` |
| `log_sequence_start`, `log_sequence_ok`, `log_sequence_ko` | `pl_master_load` |
| `pl_load_object` | `pl_load_tables` |

Connexions Fabric requises, déclarées dans `deployment/dependencies.yml` et vérifiées au pré-vol :

- `MDS_<ENV>_DP_FABRIC` (source SQL Server MDS, une par environnement) ;
- `DP-Fabric-Global-APR` (connexion « Fabric Data Pipelines », nom exact à confirmer).

## Arborescence

```text
its-dp-fab-adp-mds-managed/                branches develop (défaut) · release/* · main
│
│  ── Déploiement (lu par Azure DevOps, jamais publié dans Fabric) ──
├── CI.yml                           # pipeline ADO FabricCI (template KPMG)
├── CD.yml                           # pipeline ADO de CD → cd-pipeline.yml@cicd_common
├── PR-Validation.yml                # check obligatoire des PR → validate-pipeline.yml@cicd_common
├── CHANGES.md                       # (facultatif) changements du dépôt
├── README.md                        # ce fichier
├── deployment/
│   ├── config.yml                   # workspaces « FR - DP - <ENV> - FAB - ADP - MDS - MANAGED »,
│   │                                #   repository_directory: "../fabric", types, exclusion To_delete
│   ├── parameter.yml                # 18 règles : 14 vers le Framework <ENV> (par nom),
│   │                                #   pl_load_tables, raccourci storage de lh_dso
│   └── dependencies.yml             # category adp : Framework seul prérequis (5 items, porte ADO),
│                                    #   data_dependencies, connexions MDS_<ENV>_DP_FABRIC et
│                                    #   DP-Fabric-Global-APR
│
│  ── Contenu du workspace ADP MDS (Git Integration DEV + publication fabric-cicd) ──
└── fabric/
    ├── pl_master_load.DataPipeline/          # 4 activités → Framework, 1 → pl_load_tables
    │   ├── .platform
    │   └── pipeline-content.json
    ├── pl_load_tables.DataPipeline/          # 3 activités → Framework pl_load_object
    │   ├── .platform
    │   └── pipeline-content.json
    ├── lh_dso.Lakehouse/
    │   ├── .platform
    │   ├── lakehouse.metadata.json
    │   └── shortcuts.metadata.json           # raccourci « storage » → lh_storage
    ├── lh_storage.Lakehouse/                 # .platform + lakehouse.metadata.json
    ├── nb_runtime_resources_item_diagnostic.Notebook/
    │   ├── .platform
    │   └── notebook-content.py
    ├── vl_item_config.VariableLibrary/
    │   ├── .platform
    │   ├── variables.json
    │   ├── settings.json
    │   └── valueSets/
    │       └── <value set>.json
    ├── env_dev.Environment/                  # .platform + Setting/Sparkcompute.yml
    ├── env_spark.Environment/                # idem
    ├── sm_observability.SemanticModel/       # présent mais NON déployé (exclu volontairement)
    │   ├── .platform
    │   ├── definition.pbism
    │   └── definition/
    │       └── expressions.tmdl
    └── To_delete/                            # EXCLU (ex. env-storage.Environment)
```

Les emplacements exacts des items sous `fabric/` se confirment sur la branche réelle avec `git ls-tree -r --name-only HEAD`.

## Exclusions (`deployment/config.yml`)

```yaml
item_types_in_scope: [VariableLibrary, Lakehouse, Notebook, DataPipeline, Environment]
folder_exclude_regex: "^/To_delete"
features: [enable_experimental_features, enable_exclude_folder, enable_shortcut_publish]
```

- Le type `SemanticModel` est **volontairement absent** de `item_types_in_scope` : `sm_observability` reste dans Git mais n'est jamais déployé.
- Le dossier `To_delete/` est exclu de la publication.
- `enable_shortcut_publish` est nécessaire pour publier le raccourci `storage` de `lh_dso`.
- Aucune suppression automatique : `unpublish` est désactivé sur les 4 environnements. Supprimer un item en ITG, PPR ou PRD reste une action manuelle validée.

## Fichiers par type d'item

Chaque item est un dossier `<nom>.<Type>/`. Il contient toujours `.platform` (type, nom affiché, identifiant logique) et ses fichiers de définition.

| Type | Fichiers de définition | Déployé |
|---|---|---|
| DataPipeline | `pipeline-content.json` | Oui |
| Notebook | `notebook-content.py` | Oui |
| Lakehouse | `lakehouse.metadata.json`, `shortcuts.metadata.json` (si raccourcis) | Oui |
| VariableLibrary | `variables.json`, `settings.json`, `valueSets/<value set>.json` | Oui |
| Environment | `Setting/Sparkcompute.yml`, `Libraries/…` (si bibliothèques) | Oui |
| SemanticModel | `definition.pbism`, `definition/…` (`expressions.tmdl`…) | **Non** (exclu volontairement) |

## Branches et déploiement

| Branche | Rôle | Déploiement | Réglage |
|---|---|---|---|
| `feature/*` | Développement | Aucun | PR vers `develop` |
| `develop` | Branche par défaut ; intégration des features | DEV par Git Integration ; ITG par la CD | Protégée ; workspace DEV connecté à `develop`, dossier Git `fabric` |
| `release/*` | Une branche par release (lot de features), créée depuis `develop` au besoin | PPR par la CD, après approbation | Protégée (PR obligatoire, check PR-Validation) |
| `main` | Reflet de la production | PRD par la CD, après approbation | Protégée (PR obligatoire depuis `release/*`) |

Un run de CD ne déploie qu'**un seul environnement** : celui de la branche d'où vient l'artefact CI. Toute autre branche est refusée.

## Déroulement de la chaîne

1. **Pull Request** vers `develop`, `release/*` ou `main` : `PR-Validation.yml` exécute les contrôles bloquants du socle. Ces contrôles portent sur :
   - la syntaxe YAML et le JSON des items ;
   - `config.yml` et `parameter.yml`, validés par fabric-cicd ;
   - les GUID en dur et les secrets en clair ;
   - la matrice de dépendances.

   Un check rouge empêche la fusion.
2. **Fusion** : `CI.yml` (template KPMG) calcule la version, lance SonarQube et publie l'artefact `fabric`.
3. **CD** (`CD.yml` → socle), stage `Validate_Artifact` : calcul de l'environnement cible selon la branche, contrôles rejoués sur l'artefact, empreinte SHA-256.
4. **Environment ADO** `fabric-<env>` : approbation (PPR, PRD), verrou, branche autorisée, template obligatoire.
5. **Stage `Deploy_<ENV>`** :
   - intégrité de l'artefact ;
   - secrets de l'environnement lus dans `frfrkvdevdp0001` ;
   - pré-vol Fabric (workspace, capacité, Framework, connexions, références) ;
   - porte Framework.
6. **Publication** par fabric-cicd : les items sont créés ou mis à jour dans le workspace cible, sans changer leur ID, avec les références remplacées pour l'environnement.
7. **Vérification après publication** : les items et les références déployés sont relus ; aucun GUID DEV ne doit rester.
8. **Rapport de déploiement** `deployment-report-<ENV>`, en JSON et Markdown, publié dans le run ADO, même en cas d'échec.

La CD déploie des **définitions d'items** : elle n'exécute aucun pipeline et ne charge aucune donnée.

## Contribuer

1. Travailler dans le workspace DEV, puis enregistrer sur une branche `feature/*` (Source control de Fabric).
2. Si un item appelle un nouvel item du Framework :
   - ajouter la règle correspondante dans `deployment/parameter.yml`, au format `$workspace.FR - DP - <ENV> - FAB - Framework - MANAGED.$items.<Type>.<nom>.$id` ;
   - déclarer l'item dans `framework.items` de `deployment/dependencies.yml`.
3. Ne jamais référencer le workspace d'un autre Data Product (ADP, BDP, UCS) dans `parameter.yml` : la CI le refuse (`DEPS_CROSS_WORKSPACE_FORBIDDEN`). Une lecture de données se fait à l'exécution et se déclare, pour information, dans `data_dependencies`.
4. Ouvrir une PR vers `develop` et corriger jusqu'à ce que le check PR-Validation soit vert.

Pour contrôler le dépôt en local, à partir d'un clone du socle :

```bash
python ../its-dp-fab-cicd-common/tools/deploy.py validate --repo .
```

## Ce qui ne doit pas figurer dans ce dépôt

| Élément | Pourquoi |
|---|---|
| `.github/workflows/CI.yml`, `.github/workflows/CD.yml` | Remplacés par `CI.yml`, `CD.yml` et `PR-Validation.yml` à la racine (pipelines Azure DevOps) |
| `deployment/deploy.py` | Remplacé par `tools/deploy.py` du socle (l'ancien script contenait des ID en dur) |
| `deployment/check_connection.py` (si présent) | Contrôle des connexions intégré au pré-vol du socle |
| Items à la racine du dépôt | Déplacés sous `fabric/` |

## Avant le premier déploiement

- [ ] Remplacer les marqueurs de `CD.yml` et `PR-Validation.yml` :
  - `__A_RELEVER__` : organisation et service connection GitHub ;
  - `__A_CONFIRMER_SERVICE_CONNECTION_ARM_<ENV>__` : une service connection Azure Resource Manager distincte par environnement, créée par Ops en fédération d'identité, autorisée à lire les 3 secrets de son environnement dans le coffre unique `frfrkvdevdp0001`.
- [ ] Déplacer les items sous `fabric/` et passer `repository_directory` à `"../fabric"` **dans la même PR**. Rediriger ensuite la Git Integration du workspace DEV vers le dossier `fabric`.
- [ ] Lancer `deploy.py validate` sur la branche réelle. Décisions du 22/10/2026 : le lien `pl_sequence_start` → `log_sequence_start` est confirmé ; reste à confirmer `lh_storage`, cible du raccourci.
- [x] Noms des workspaces Framework PPR et PRD confirmés le 22/10/2026 : `FR - DP - PPR - FAB - Framework - MANAGED`, `FR - DP - PRD - FAB - Framework - MANAGED`.
- [ ] Passer `inventory_status` à `validated` dans une PR revue.
- [ ] Vérifier que le Framework a été déployé au moins une fois par la nouvelle CD Framework sur l'environnement visé. Sinon la porte renvoie `GATE_NEVER_DEPLOYED`.
