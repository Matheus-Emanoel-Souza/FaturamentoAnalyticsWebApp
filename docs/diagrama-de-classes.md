# Diagrama de Classes

> O app é escrito em JavaScript vanilla, procedural, sem classes ES6 reais.
> Este diagrama documenta o **modelo de dados** (as estruturas persistidas em
> `state`, salvas em `localStorage`) e os **módulos funcionais** do único
> arquivo `index.html`, tratados como classes lógicas para fins de
> documentação de arquitetura.

## Modelo de dados e serviços

```mermaid
classDiagram
    class AppState {
        +number seq
        +Transaction[] transactions
        +number[] paidExcludedMonths
        +Category[] categories
        +Flag[] flagsConfig
        +Prefs prefs
    }

    class Transaction {
        +number id
        +string uuid
        +string seriesId
        +string name
        +number amount
        +TransactionType type
        +number monthIndex
        +number dayOfMonth
        +RecurrenceType recurrence
        +number installmentNumber
        +number installmentTotal
        +boolean settled
        +boolean isOverride
        +string categoryId
        +string[] flags
    }

    class Category {
        +string id
        +string name
        +string icon
    }

    class Flag {
        +string id
        +string name
        +string icon
    }

    class Prefs {
        +string theme
        +string accent
        +boolean amoled
        +boolean compact
        +string sortField
        +boolean sortAscending
    }

    class TransactionType {
        <<enumeration>>
        INCOME
        EXPENSE
    }

    class RecurrenceType {
        <<enumeration>>
        NONE
        FIXED
        INSTALLMENT
    }

    class EditScope {
        <<enumeration>>
        SINGLE
        THIS_AND_FOLLOWING
    }

    AppState "1" *-- "0..*" Transaction
    AppState "1" *-- "0..*" Category
    AppState "1" *-- "0..*" Flag
    AppState "1" *-- "1" Prefs
    Transaction --> TransactionType
    Transaction --> RecurrenceType
    Transaction "0..*" --> "0..1" Category : categoryId
    Transaction "0..*" --> "0..*" Flag : flags[]

    class StateManager {
        +load() void
        +save() void
        +normalizeState() void
        +nextId() number
    }

    class TransactionService {
        +addNew(input) void
        +pushTx(t) void
        +ensureCoverage(upto) void
        +updateScoped(edited, scope EditScope) void
        +deleteScoped(tx, scope EditScope) void
        +txById(id) Transaction
        +sortTx(arr) Transaction[]
    }

    class CategoryFlagService {
        +listOf(kind) Array
        +catById(id) Category
        +flagById(id) Flag
        +saveItem() void
        +deleteItem(kind, id) void
    }

    class AnalysisService {
        +monthTotal(mi, type) number
        +totalsByCategory(mi, type) Map
        +countsInAnalysis(t, mi, type) boolean
        +renderAnalysis() void
    }

    class BackupService {
        +buildBackup() object
        +exportBackup() void
        +exportCSV() void
        +parseBackup(text) object
        +migrateV3ToV4(data) object
        +previewImport(text) object
        +applyImport(mode) void
        +shareOrDownload(fname, mime, text) void
    }

    class UIRenderer {
        +render() void
        +applyTheme() void
        +animateMoney(el, endValue) void
        +changeMonthAnimated(direction) void
        +openEditor(tx) void
        +openSettings() void
        +openAnalysis() void
    }

    class ThemeManager {
        +applyTheme() void
        +buildAccentSwatches() void
    }

    StateManager ..> AppState : gerencia
    TransactionService ..> AppState : lê/escreve
    TransactionService ..> StateManager : chama save()
    CategoryFlagService ..> AppState
    AnalysisService ..> AppState : somente leitura
    BackupService ..> AppState : serializa/restaura
    UIRenderer ..> AppState : lê para desenhar
    UIRenderer ..> TransactionService
    UIRenderer ..> AnalysisService
    UIRenderer ..> ThemeManager
```

## Notas

- **`Category` e `Flag`** são conceitos distintos: uma `Category` por
  `Transaction` (classificação única), N `Flag`s por `Transaction`
  (marcadores livres).
- **`seriesId`** liga as ocorrências materializadas de uma mesma recorrência
  (`FIXED` ou `INSTALLMENT`); `EditScope` define se uma edição/exclusão afeta
  só a ocorrência (`SINGLE`) ou ela e as futuras da mesma série
  (`THIS_AND_FOLLOWING`).
- **`uuid`** é a chave de deduplicação usada por `BackupService` ao importar
  (`MERGE`); **`id`** é apenas o identificador interno incremental.
