# Project Instructions — SonarQube Mutmut Metrics Plugin

## Objectif

Plugin SonarQube custom (Java) qui lit les résultats de mutation testing produits par **mutmut** et remonte des **mesures** (measures) dans l'onglet Measures de SonarQube, ainsi que des issues gérables pour les mutants survivants.

Cible: remplacer l'approche actuelle "generic issue import" (`sonar.externalIssuesReportPaths`) qui ne produit que des issues non gérables et aucun mutation score comme métrique.

## Périmètre

### Dans le scope
- Définir des métriques custom (mutation score, mutants total/killed/survived/timeout/skipped)
- Un Sensor qui parse les fichiers `mutants/*.meta` (JSON) laissés par `mutmut run`
- Sauver des measures au niveau projet (et optionnellement fichier)
- Créer des issues (gérables, pas external) pour les mutants survivants
- Build Maven produisant un JAR déployable
- Tests unitaires (parser + sensor)

### Hors scope
- UI custom / pages web SonarQube (trop de maintenance, non nécessaire)
- Support de MutPy, cosmic-ray ou autres outils (mutmut uniquement)
- Support de langages autres que Python (mutmut cible Python)
- Règles personnalisées complexes (un seul rule: "mutant survived")

## Stack technique

| Élément | Choix | Raison |
|---|---|---|
| Langage | Java 17 | LTS, requis par sonar-plugin-api 10.x |
| Build | Maven 3.9+ | Standard SonarQube, sonar-packaging-maven-plugin |
| API | `org.sonarsource.api.plugin:sonar-plugin-api` 10.x | API officielle, indépendante de la version SQ |
| JSON parsing | Jackson (com.fasterxml.jackson) | Déjà transitif via sonar-plugin-api, robuste |
| Tests | JUnit 5 + AssertJ + sonar-testing-harness | Standard SQ |
| Python | Aucun (plugin Java pur) | Le parsing des .meta se fait en Java |

## Structure du projet

```
mutmut-sonar-plugin/
├── pom.xml
├── src/main/java/mutmutsonar/
│   ├── MutmutPlugin.java                    # Plugin → addExtensions
│   ├── metrics/
│   │   └── MutmutMetrics.java                # Définition des métriques custom
│   ├── sensors/
│   │   └── MutmutReportSensor.java            # Lit les .meta, calcule, sauve measures + issues
│   ├── parser/
│   │   └── MutmutMetaParser.java              # Parse mutants/*.meta (JSON) → MutantResult
│   └── model/
│       └── MutantResult.java                  # Record: mutantKey, exitCode, sourceFile, status
├── src/test/java/mutmutsonar/
│   ├── parser/
│   │   └── MutmutMetaParserTest.java
│   ├── sensors/
│   │   └── MutmutReportSensorTest.java
│   └── metrics/
│       └── MutmutMetricsTest.java
└── src/main/resources/
    └── mutmutsonar/
        └── MutmutRules.json                  # Définition de la règle "mutant survived"
```

## Métriques à définir

Toutes au niveau **projet** (Metric.Domain = "Mutation testing"). ValueType: INTEGER sauf mutation_score (PERCENT).

| Key | Nom affiché | ValueType | Description |
|---|---|---|---|
| `mutmut_total_mutants` | Total mutants | INTEGER | Nombre total de mutants générés |
| `mutmut_killed_mutants` | Killed mutants | INTEGER | Mutants tués par les tests |
| `mutmut_survived_mutants` | Survived mutants | INTEGER | Mutants non tués (gap de test) |
| `mutmut_timeout_mutants` | Timeout mutants | INTEGER | Mutants en timeout |
| `mutmut_skipped_mutants` | Skipped mutants | INTEGER | Mutants ignorés |
| `mutmut_mutation_score` | Mutation score | PERCENT | killed / total * 100 |

Direction: `Metric.DIRECTION_BETTER` pour killed et score, `DIRECTION_WORST` pour survived/timeout/skipped.

## Mapping exit code → status (mutmut)

Répliquer exactement la table du script Python existant (`mutmut-to-sonar.py:36-54`):

| Exit code | Status | Reportable? |
|---|---|---|
| 1, 3 | killed | Non |
| 0 | survived | Oui |
| 5, 33 | no tests | Oui |
| 34 | skipped | Oui |
| 35 | suspicious | Oui |
| 36, 37 | caught by type check | Non |
| 24, -24, 152, 255 | timeout | Oui |
| 2 | interrupted | Non |
| None | not checked | Non |
| -11, -9 | segfault | Non |

Source de vérité: `mutmut.stats.status_by_exit_code`. Garder cette table en synchronisation manuelle (ne pas importer mutmut en Java).

## Parsing des fichiers .meta

Format d'un fichier `mutants/foo/bar.py.meta` (JSON):

```json
{
  "exit_code_by_key": {
    "app.bigapi.x_get_all__mutmut_1": 0,
    "app.bigapi.xǁMyClassǁmethod__mutmut_2": 1
  }
}
```

Logique:
1. `rglob("*.meta")` sous `mutants/`
2. Pour chaque `.meta`: chemin source = relatif sans suffixe `.meta` (ex: `mutants/app/foo.py.meta` → `app/foo.py`)
3. Pour chaque clé dans `exit_code_by_key`: extraire le nom de fonction et la classe éventuelle
4. Mapper l'exit code vers un status via la table ci-dessus
5. Calculer les agrégats (total, killed, survived, etc.) → sauver comme measures projet
6. Pour chaque mutant "reportable": créer une issue sur la ligne de définition de la fonction

### Extraction du nom de fonction depuis la clé mutmut

La clé est du type `app.bigapi.x_get_all__mutmut_1` ou `app.bigapi.xǁMyClassǁmethod__mutmut_2`.

- Strip le suffixe `__mutmut_N`
- Prend la partie après le dernier `.` → `x_get_all` ou `xǁMyClassǁmethod`
- Le séparateur de classe est `ǁ` (U+01C1)
- Si le nom commence par `x_`, c'est une fonction hors classe → strip `x_`
- Si `ǁ` est présent, extraire `class_name` entre les séparateurs et `method_name` après le dernier

Répliquer la logique de `mutmut-to-sonar.py:80-98`.

### Trouver la ligne de la fonction

Le plugin Java ne peut pas utiliser `ast` Python. Deux options:
1. **Regex sur le source Python** (recommandé, simple): chercher `def <func_name>` ou `async def <func_name>` dans le fichier source
2. Demander à mutmut d'inclure le numéro de ligne dans les `.meta` (nécessite une PR upstream)

Option 1 suffit pour un MVP. Si plusieurs `def`同名, prendre le premier. Documenter la limitation.

## Règle SonarQube custom

Une seule règle:
- Key: `mutmut:mutant-survived`
- Type: `CODE_SMELL`
- Severity: `MAJOR`
- Message: `Mutmut: mutation in <func_name> was not killed by any test (mutant: <mutant_key>)`

Pas de Quality Profile custom nécessaire (règle activée par défaut dans le plugin).

## Propriétés de configuration

Le sensor doit lire ces propriétés SonarQube (configurables via UI ou `sonar-scanner -D`):

| Propriété | Défaut | Description |
|---|---|---|
| `sonar.mutmut.reportPath` | `mutants` | Chemin vers le dossier des `.meta` |
| `sonar.mutmut.enabled` | `true` | Activer/désactiver le sensor |

## Build & test

```bash
# Build
mvn clean package
# → target/mutmut-sonar-plugin-<version>.jar

# Tests
mvn test

# Install (sur serveur SonarQube)
cp target/mutmut-sonar-plugin-*.jar $SONARQUBE_HOME/extensions/plugins/
# Redémarrer SonarQube
```

## Déploiement CI

Ajouter un job de build JAR au `.gitlab-ci.yml` existant du repo `mutmut-sonar`:
- Stage: `build`
- Image: `maven:3.9-eclipse-temurin-17`
- Script: `mvn clean package -DskipTests`
- Artifact: `target/*.jar`

## Contraintes et pièges connus

### API SonarQube
- Le `sonar-plugin-api`groupId a été déplacé: `org.sonarsource.sonarqube` (ancien) → `org.sonarsource.api.plugin` (nouveau, 10.x+). Utiliser le nouveau.
- Scope Maven: `provided` pour sonar-plugin-api (le serveur fournit l'API au runtime).
- `sonar-packaging-maven-plugin` avec `<packaging>sonar-plugin</packaging>` obligatoire.
- Le `pluginClass` dans la config Maven doit pointer vers `MutmutPlugin.java`.

### Compatibilité
- Tester contre une instance SonarQube réelle (Community ou Enterprise) avant de livrer. L'API classloader est capricieuse.
- Version SQ minimale cible: 10.x (Community Build). Documenter la compatibilité dans le README.

### Limitations du parsing
- Le parsing par regex de `def` ne gère pas les fonctions imbriquées ou les décorateurs complexes. Acceptable pour un MVP.
- Si `mutants/` n'existe pas: le sensor se termine en silence (warning dans les logs), pas d'erreur fatale.
- Le mutation score est sauvegardé comme measure projet, pas fichier. C'est volontaire (agrégat global).

## Références

- API officielle: https://docs.sonarsource.com/sonarqube-server/2025.3/extension-guide/developing-a-plugin/plugin-basics/
- Exemple officiel: https://github.com/SonarSource/sonar-custom-plugin-example (branche 10.x)
- sonar-plugin-api: https://github.com/SonarSource/sonar-plugin-api
- Script Python de référence (logique métier à répliquer): `mutmut-to-sonar.py` dans ce repo
- Generic issue format (mécanisme actuel, à remplacer): https://docs.sonarqube.org/latest/analysis/generic-issue/

## Critères d'acceptation

1. `mvn clean package` produit un JAR sans erreur.
2. Le JAR déployé sur un SonarQube 10.x affiche les 6 métriques dans Measures après une analyse.
3. Le mutation score apparaît comme measure `mutmut_mutation_score` (PERCENT).
4. Les mutants survivants apparaissent comme issues gérables (règle `mutmut:mutant-survived`), marquables "False Positive" dans l'UI.
5. `mvn test` passe (parser + sensor + metrics).
6. Le sensor gère l'absence de dossier `mutants/` sans crasher.
