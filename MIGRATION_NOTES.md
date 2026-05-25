# Migration Java 21 — Qcadoo MES

## Contexte

Fork personnel de Qcadoo MES (version community 1.5-SNAPSHOT) migré de Java 8 vers Java 21.  
Réalisé à titre d'exercice technique sur la branche `migration/java-21`.

**Environnement de travail**
- Windows 10 Pro 22H2
- JDK Temurin 21.0.11
- Maven 3.9.16
- Eclipse IDE

---

## Résultat final

| Objectif | Statut |
|---|---|
| Compilation Java 21 (58/58 modules) | ✅ |
| Nettoyage APIs dépréciées | ✅ |
| Virtual Threads Tomcat | ⏸ (pas de serveur disponible) |
| Tests unitaires | ❌ (voir section Limites) |

---

## Ce qui a été modifié

### 1. `pom.xml` racine — Configuration Java 21

- Plugin AspectJ remplacé : `org.codehaus.mojo:aspectj-maven-plugin` → `dev.aspectj:aspectj-maven-plugin:1.14.1`
- Version aspectjtools : `1.9.21.2`
- Source/target/complianceLevel → `21`
- `<Xlint>ignore</Xlint>` ajouté pour supprimer les warnings AspectJ
- Ajout de `javax.annotation-api:1.3.2` (supprimé du JDK en Java 11)
- Exclusions de `xml-apis` et `stax-api` via `<dependencyManagement>` (conflits de versions)
- Plugin OpenRewrite `5.45.0` conservé (tentative abandonnée, voir section Tentatives)

### 2. Groupe 1 — BigDecimal (39 fichiers)

Constantes dépréciées remplacées par l'enum `RoundingMode` :

| Avant | Après |
|---|---|
| `BigDecimal.ROUND_HALF_UP` | `RoundingMode.HALF_UP` |
| `BigDecimal.ROUND_FLOOR` | `RoundingMode.FLOOR` |
| `BigDecimal.ROUND_DOWN` | `RoundingMode.DOWN` |
| `divide(BigDecimal, int, int)` | `divide(BigDecimal, int, RoundingMode)` |
| `setScale(int, int)` | `setScale(int, RoundingMode)` |

**Exception intentionnelle** : 6 appels `unitConversions.convertTo(x, unit, BigDecimal.ROUND_FLOOR)` conservés avec la constante `int` car l'API Qcadoo interne attend un `int`, pas un `RoundingMode` :
- `ResourceManagementServiceImpl.java` lignes 548-549
- `ProductionCountingDocumentService.java` ligne 277
- `ProductionTrackingListenerServicePFTD.java` ligne 441
- `TrackingOperationProductComponentDetailsListeners.java` ligne 212
- `ProductionTrackingXlsxImportService.java` ligne 140

### 3. Groupe 2 — Apache POI (18 fichiers)

**Catégorie A — Constantes de type cellule :**

| Avant | Après |
|---|---|
| `Cell.CELL_TYPE_NUMERIC` | `CellType.NUMERIC` |
| `Cell.CELL_TYPE_STRING` | `CellType.STRING` |
| `HSSFCell.CELL_TYPE_NUMERIC` | `CellType.NUMERIC` |
| `row.createCell(col, HSSFCell.CELL_TYPE_NUMERIC)` | `row.createCell(col)` |

Import ajouté : `import org.apache.poi.ss.usermodel.CellType;`

**Catégorie B — Couleurs HSSFColor (5 fichiers) :**

| Avant | Après |
|---|---|
| `HSSFColor.BLACK.index` | `IndexedColors.BLACK.getIndex()` |
| `HSSFColor.GREY_25_PERCENT.index` | `IndexedColors.GREY_25_PERCENT.getIndex()` |
| `HSSFColor.GREY_50_PERCENT.index` | `IndexedColors.GREY_50_PERCENT.getIndex()` |
| `HSSFColor.PALE_BLUE.index` | `IndexedColors.PALE_BLUE.getIndex()` |

Import ajouté : `import org.apache.poi.ss.usermodel.IndexedColors;`

### 4. Groupe 3 — Commons Lang (10 fichiers)

| Avant | Après |
|---|---|
| `ObjectUtils.equals(a, b)` | `Objects.equals(a, b)` |
| `ObjectUtils.hashCode(x)` | `Objects.hashCode(x)` |
| `ObjectUtils.toString(x)` | `Objects.toString(x, "")` |

Import ajouté : `import java.util.Objects;`

### 5. Groupe 4 — DateRange Qcadoo — Non traité

`com.qcadoo.commons.dateTime.DateRange` est une classe interne du framework Qcadoo dépréciée. Sans accès au code source du framework, on ne peut pas déterminer le remplacement correct sans risquer de casser la logique métier. Ce groupe est laissé intentionnellement de côté.

### 6. Groupe 5 — APIs Java diverses (5 fichiers)

| Avant | Après | Fichier |
|---|---|---|
| `new Locale("pl")` | `Locale.forLanguageTag("pl")` | `BigDecimalCellParser.java` |
| `new Locale("cn")` | `Locale.forLanguageTag("cn")` | `BigDecimalCellParser.java` |
| `new URL(string)` | `URI.create(string).toURL()` | `ExchangeRatesNbpServiceImpl.java` |
| `dateTime.toDateMidnight()` | `dateTime.withTimeAtStartOfDay()` | `ShiftsServiceImpl.java` |
| `date.getDay() == 0 ? 7 : date.getDay()` | `new LocalDate(date).getDayOfWeek()` | `PpsTimeHelper.java` |

---

## Tentatives abandonnées

### OpenRewrite
Tenté avec la recette `UpgradeToJava21`. Échec car les dépendances `qcadoo-*:1.5-SNAPSHOT` ne sont plus disponibles sur le Nexus Qcadoo (HTTP 404). Sans résolution complète des dépendances, OpenRewrite ne peut pas faire l'analyse de type nécessaire. Le plugin reste dans le `pom.xml` pour usage futur si la situation change.

### UpJavaAI
Tenté mais le packaging `qcadoo-plugin` non standard empêche l'analyse. CSV de rapport toujours vide.

---

## Limites découvertes

### Tests unitaires — Échec Mockito
Tous les tests unitaires échouent avec :
```
NoClassDefFoundError: Could not initialize class org.mockito.internal.creation.jmock.ClassImposteriser$3
```

**Cause** : Qcadoo Framework embarque Mockito 1.x avec son propre cglib. Cette version est fondamentalement incompatible avec Java 21 à cause des restrictions du module system (JPMS). Ce problème préexistait avant notre migration — il n'est pas causé par nos modifications.

**Solution** : Pour avoir des tests fonctionnels sous Java 21, il faudrait d'abord migrer **Qcadoo Framework** (`qcadoo/qcadoo`) vers Mockito 5.x et Spring 5/6. C'est un chantier plus important qui constitue un prérequis à une migration complète et stable.

### Virtual Threads Tomcat
Non configuré — nécessite un serveur Tomcat 10+ de déploiement.

---

## Conclusion

La compilation sous Java 21 est fonctionnelle (58/58 modules). La migration complète avec tests stables nécessite en premier lieu la migration de Qcadoo Framework lui-même, qui embarque des dépendances (Mockito 1.x, cglib, Spring 3.2) fondamentalement incompatibles avec Java 21.

---

*Migration réalisée le 25 mai 2026 — branche `migration/java-21` du fork `slulu90/qcadoomesJV21`*
