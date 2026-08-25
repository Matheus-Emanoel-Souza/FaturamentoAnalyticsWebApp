# Diagrama de Implementação de Pacotes

> O projeto não tem build nem módulos ES (`import`/`export`) — tudo vive num
> único `<script>` dentro de `index.html`. Os "pacotes" abaixo são
> **agrupamentos lógicos de responsabilidade** dentro desse arquivo, mais os
> artefatos físicos do repositório (`sw.js`, `manifest.webmanifest`,
> `icon.svg`). O diagrama mostra as dependências reais entre esses
> agrupamentos.

## Pacotes lógicos (dentro de `index.html`)

```mermaid
flowchart TB
    subgraph PKG_UI["📦 ui (render, overlays, eventos)"]
        UI1[render / bind]
        UI2[openEditor / openSettings / openAnalysis]
        UI3[changeMonthAnimated / initSwipeGesture]
        UI4[showOverlay / hideOverlay]
    end

    subgraph PKG_STATE["📦 state (persistência)"]
        ST1[load / save]
        ST2[normalizeState]
        ST3[nextId / txById]
    end

    subgraph PKG_TX["📦 transactions (domínio principal)"]
        TX1[addNew / pushTx]
        TX2[ensureCoverage]
        TX3[updateScoped / deleteScoped]
        TX4[sortTx]
    end

    subgraph PKG_CATFLAG["📦 categories-flags"]
        CF1[listOf / catById / flagById]
        CF2[saveItem / deleteItem]
        CF3[buildCategorySelect / buildFlagPicker]
    end

    subgraph PKG_ANALYSIS["📦 analysis"]
        AN1[monthTotal / totalsByCategory]
        AN2[countsInAnalysis]
        AN3[renderAnalysis]
    end

    subgraph PKG_BACKUP["📦 backup-io (import/export)"]
        BK1[buildBackup / exportBackup]
        BK2[exportCSV]
        BK3[parseBackup / migrateV3ToV4]
        BK4[previewImport / applyImport]
        BK5[shareOrDownload]
    end

    subgraph PKG_THEME["📦 theme"]
        TH1[applyTheme]
        TH2[buildAccentSwatches / ACCENTS]
    end

    subgraph PKG_UTIL["📦 util (formatação/datas)"]
        UT1[money / formatDate / parseAmount]
        UT2[monthLabel / ymOf / daysInMonth]
        UT3[uuid / newUuid / makeSlugId]
    end

    subgraph PKG_PWA["📦 pwa (fora do index.html)"]
        PW1[sw.js — service worker]
        PW2[manifest.webmanifest]
        PW3[icon.svg]
    end

    PKG_UI --> PKG_TX
    PKG_UI --> PKG_CATFLAG
    PKG_UI --> PKG_ANALYSIS
    PKG_UI --> PKG_BACKUP
    PKG_UI --> PKG_THEME
    PKG_UI --> PKG_STATE
    PKG_UI --> PKG_UTIL

    PKG_TX --> PKG_STATE
    PKG_TX --> PKG_UTIL
    PKG_CATFLAG --> PKG_STATE
    PKG_ANALYSIS --> PKG_STATE
    PKG_ANALYSIS --> PKG_UTIL
    PKG_BACKUP --> PKG_STATE
    PKG_BACKUP --> PKG_TX
    PKG_BACKUP --> PKG_CATFLAG
    PKG_THEME --> PKG_STATE

    PKG_UI -. registra .-> PKG_PWA
```

## Pacotes físicos (repositório)

```mermaid
flowchart LR
    subgraph ROOT["Raiz do repositório"]
        A[index.html\napp inteiro: markup + CSS + JS]
        B[sw.js\nservice worker]
        C[manifest.webmanifest\nmetadados PWA]
        D[icon.svg]
        E[README.md]
        F[docs/]
    end

    A -- registra e é cacheado por --> B
    B -- lista para cache --> A
    B -- lista para cache --> C
    B -- lista para cache --> D
    C -- referencia --> D
```

## Regras de dependência

- **`ui`** é o único pacote que conhece todos os outros — orquestra tudo a
  partir de `render()` e dos handlers registrados em `bind()`.
- **`analysis`**, **`categories-flags`** e **`backup-io`** só leem/escrevem
  `state` diretamente, nunca manipulam o DOM sozinhos (quem desenha é sempre
  `ui`).
- **`util`** não depende de nenhum outro pacote — é a camada mais baixa
  (formatação de moeda, datas, geração de ids).
- **`pwa`** é independente do domínio de negócio: só faz cache de arquivos
  estáticos, não conhece `Transaction`, `Category` etc.
- Não há dependência de pacotes externos (sem `package.json`, sem CDN) — é
  100% vanilla.
