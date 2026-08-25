# Diagrama C4

Modelagem da arquitetura do sistema seguindo o modelo C4 (Contexto,
Containers, Componentes). Como o app não tem backend, os níveis 1 e 2
concentram a maior parte da informação relevante; o nível 3 detalha os
módulos internos do container único (o navegador).

## Nível 1 — Diagrama de Contexto

```mermaid
C4Context
    title Diagrama de Contexto — Faturamento Web App

    Person(usuario, "Usuário", "Pessoa que controla suas entradas e saídas financeiras mensais")

    System(faturamentoWeb, "Faturamento Web App", "PWA de controle financeiro pessoal, HTML/CSS/JS puro, roda 100% no navegador")

    System_Ext(faturamentoAndroid, "App Android equivalente", "Mesma paleta e mesmo formato de backup JSON; usado para interoperar via troca manual de arquivo de backup")

    System_Ext(navegador, "Navegador (Chrome/Edge/Safari)", "Fornece localStorage, Service Worker, Web Share API e instalação como PWA")

    Rel(usuario, faturamentoWeb, "Usa: registra transações, consulta resumos e análises")
    Rel(faturamentoWeb, navegador, "Persiste dados em localStorage; registra Service Worker para funcionar offline")
    Rel(usuario, faturamentoAndroid, "Usa em paralelo, no celular")
    BiRel(faturamentoWeb, faturamentoAndroid, "Troca de dados via arquivo de backup JSON (export/import manual pelo usuário)")
```

## Nível 2 — Diagrama de Containers

> Não há "containers" no sentido de servidores/serviços separados: é um site
> estático servido por qualquer servidor HTTP, sem API própria. O diagrama
> mostra os artefatos que rodam no dispositivo do usuário.

```mermaid
C4Container
    title Diagrama de Containers — Faturamento Web App

    Person(usuario, "Usuário")

    System_Boundary(dispositivo, "Dispositivo do usuário (navegador)") {
        Container(spa, "Aplicação Web (SPA)", "HTML + CSS + JavaScript vanilla (index.html)", "Renderiza a UI, contém toda a lógica de negócio; único arquivo, sem framework")
        Container(sw, "Service Worker", "JavaScript (sw.js)", "Cache-first para assets estáticos; rede-primeiro para o HTML; habilita uso offline")
        ContainerDb(storage, "localStorage", "Web Storage API", "Armazena o objeto `state` inteiro sob a chave faturamento_web_v1; é o único banco de dados do app")
        Container(manifest, "Web App Manifest", "manifest.webmanifest", "Metadados para instalação como PWA: nome, ícones, modo standalone, tema")
    }

    System_Ext(hostEstatico, "Servidor HTTP estático", "Qualquer host que sirva arquivos estáticos via HTTP(S)")
    System_Ext(sistemaOperacional, "SO / Home Screen", "Onde o PWA pode ser instalado como app")

    Rel(usuario, spa, "Interage via toque/clique", "UI HTML/CSS")
    Rel(spa, storage, "Lê/escreve state a cada save()", "localStorage API")
    Rel(spa, sw, "Registra no boot (load → bind → render → initSwipeGesture → registro do SW)")
    Rel(sw, manifest, "É referenciado junto ao manifest para instalação")
    Rel(hostEstatico, spa, "Serve index.html, sw.js, manifest.webmanifest, icon.svg via HTTP(S)")
    Rel(spa, sistemaOperacional, "Pode ser instalado como app standalone", "PWA install prompt")
```

## Nível 3 — Diagrama de Componentes (dentro da SPA)

```mermaid
C4Component
    title Diagrama de Componentes — dentro de index.html

    Container_Boundary(spa, "Aplicação Web (index.html)") {
        Component(ui, "UI / Renderer", "funções render, bind, openEditor...", "Desenha a lista de transações, overlays, resumo do mês; único ponto que manipula o DOM")
        Component(stateMgr, "State Manager", "funções load, save, normalizeState", "Carrega/persiste o objeto state completo no localStorage")
        Component(txService, "Transaction Service", "funções addNew, updateScoped, deleteScoped, ensureCoverage", "Materializa e mantém as ocorrências de transações, inclusive recorrências FIXED/INSTALLMENT")
        Component(catFlagService, "Category/Flag Service", "funções saveItem, deleteItem, catById, flagById", "CRUD de categorias e flags, nunca exclui transações associadas")
        Component(analysisService, "Analysis Service", "funções monthTotal, totalsByCategory, renderAnalysis", "Agrega totais por categoria e por mês para o overlay de análise")
        Component(backupService, "Backup/IO Service", "funções buildBackup, parseBackup, previewImport, applyImport, exportCSV", "Serializa/desserializa o estado; import em 3 etapas; exporta CSV")
        Component(themeMgr, "Theme Manager", "funções applyTheme, buildAccentSwatches", "Aplica tema claro/escuro/sistema, AMOLED, modo compacto e cor de destaque")
    }

    ContainerDb(storage, "localStorage", "Web Storage API")
    Container(sw, "Service Worker", "sw.js")

    Rel(ui, txService, "Aciona ao salvar/editar/excluir")
    Rel(ui, catFlagService, "Aciona ao gerenciar categorias/flags")
    Rel(ui, analysisService, "Aciona ao abrir análise")
    Rel(ui, backupService, "Aciona ao exportar/importar")
    Rel(ui, themeMgr, "Aciona ao trocar tema")
    Rel(ui, stateMgr, "Chama save() após qualquer mudança")

    Rel(txService, stateMgr, "Usa save()/nextId()")
    Rel(catFlagService, stateMgr, "Usa save()")
    Rel(backupService, stateMgr, "Usa save()/normalizeState()")
    Rel(analysisService, stateMgr, "Lê state, somente leitura")

    Rel(stateMgr, storage, "getItem/setItem faturamento_web_v1")
    Rel(ui, sw, "Registra no boot")
```
