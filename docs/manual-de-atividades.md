# Manual de Atividades (com Diagrama)

Guia de uso do app descrevendo os principais fluxos de atividade do usuário,
cada um acompanhado do diagrama de atividades correspondente (UML,
representado em Mermaid).

---

## 1. Inicialização do app

Ao abrir `index.html`, o app carrega o estado salvo, prepara a tela e
registra o service worker para funcionar offline.

```mermaid
flowchart TD
    Start([Início]) --> Load[load]
    Load --> Norm[normalizeState]
    Norm --> Bind[bind: registra eventos da UI]
    Bind --> Render[render]
    Render --> Coverage[ensureCoverage: materializa recorrências\naté 12 meses à frente]
    Coverage --> Theme[applyTheme]
    Theme --> Filter[Filtra transações do currentMonth]
    Filter --> Totals[Calcula totais\ncom animateMoney]
    Totals --> List[Reconstrói lista de cards]
    List --> Swipe[initSwipeGesture]
    Swipe --> SW[Registra Service Worker]
    SW --> End([App pronto para uso])
```

---

## 2. Adicionar ou editar uma transação

```mermaid
flowchart TD
    Start([Usuário toca no FAB + ou num card existente]) --> Open[openEditor]
    Open --> IsNew{Transação nova?}
    IsNew -- Sim --> ShowRec[Mostra bloco de recorrência\nNONE / FIXED / INSTALLMENT]
    IsNew -- Não --> HideRec[Esconde bloco de recorrência\nrecorrência só se define na criação]
    ShowRec --> Fill[Usuário preenche nome, valor,\ncategoria, flags, data]
    HideRec --> Fill
    Fill --> Save{Usuário confirma?}
    Save -- Cancelar --> Close1[hideOverlay]
    Save -- Salvar --> Collect[collectEditorInput]
    Collect --> IsExisting{Editando transação\nde uma série existente?}
    IsExisting -- Sim --> Scope[Abre scopeOverlay:\nSINGLE ou THIS_AND_FOLLOWING]
    Scope --> Apply[updateScoped]
    IsExisting -- Não --> AddNew[addNew:\ngera 36 linhas se FIXED,\nN linhas se INSTALLMENT]
    Apply --> PersistSave[save]
    AddNew --> PersistSave
    PersistSave --> Rerender[render]
    Rerender --> End([Lista atualizada])
    Close1 --> End
```

---

## 3. Excluir uma transação

```mermaid
flowchart TD
    Start([Usuário aciona excluir num card]) --> Check{Faz parte de uma\nsérie recorrente?}
    Check -- Não --> Del[deleteScoped: remove só essa linha]
    Check -- Sim --> Scope[Abre scopeOverlay]
    Scope --> Single[SINGLE:\nremove só a ocorrência]
    Scope --> Following[THIS_AND_FOLLOWING:\nremove todas da mesma seriesId\ncom monthIndex >= atual]
    Single --> Save[save]
    Following --> Save
    Del --> Save
    Save --> Render[render]
    Render --> End([Lista atualizada])
```

---

## 4. Alternar "descontar pagos" (botão R$) e navegar entre meses

```mermaid
flowchart TD
    Start([Usuário toca no botão R$]) --> Toggle[Alterna mês atual em\npaidExcludedMonths]
    Toggle --> Save[save]
    Save --> Render1[render]
    Render1 --> Recalc{Botão ligado?}
    Recalc -- Sim --> Exclude[Saídas já settled\nnão entram no total\nnem no saldo]
    Recalc -- Não --> Include[Todas as saídas contam\nno total normalmente]
    Exclude --> End1([Resumo atualizado])
    Include --> End1

    Start2([Usuário toca ‹ / › ou faz swipe]) --> ChangeMonth[changeMonthAnimated]
    ChangeMonth --> Slide[Anima slide de transição]
    Slide --> Coverage[ensureCoverage]
    Coverage --> Render2[render com novo currentMonth]
    Render2 --> End2([Novo mês exibido])
```

---

## 5. Importar backup (3 etapas)

```mermaid
flowchart TD
    Start([Usuário escolhe arquivo de backup]) --> Preview[previewImport: parseBackup]
    Preview --> Version{version < 4?}
    Version -- Sim --> Migrate[migrateV3ToV4:\ngera uuid, flags: [],\ncategoryId: null]
    Version -- Não --> Summary
    Migrate --> Summary[Mostra resumo:\nversão, nº transações,\nperíodo, categorias, flags,\ntotais, quantas seriam novas]
    Summary --> Choice{Usuário escolhe modo}
    Choice -- REPLACE --> Replace[Apaga transações atuais\ne restaura o backup]
    Choice -- MERGE --> Merge[Adiciona sem apagar:\ndeduplica por uuid,\nmescla categorias/flags por id\nlocal prevalece,\npushTx reatribui id]
    Choice -- Cancelar --> Cancel[Nada é alterado]
    Replace --> Apply[applyImport]
    Merge --> Apply
    Apply --> Save[save]
    Save --> Render[render]
    Render --> End([Dados importados])
    Cancel --> End2([Overlay fechado, estado intacto])
```

---

## 6. Exportar dados (backup JSON v4 ou CSV)

```mermaid
flowchart TD
    Start([Usuário abre Configurações]) --> Choice{Tipo de exportação}
    Choice -- Backup completo --> Build[buildBackup:\nformat, version 4, exportedAt,\ncategories, flags, transactions,\nsettings, tasks]
    Choice -- CSV --> CSV[exportCSV:\ncolunas data;descricao;tipo;\ncategoria;valor;pago;flags\nseparador ; decimal com vírgula\nBOM UTF-8]
    Build --> Share[shareOrDownload]
    CSV --> Share
    Share --> Available{Web Share API\ndisponível?}
    Available -- Sim --> WebShare[Abre menu de compartilhamento\nnativo do dispositivo]
    Available -- Não --> Download[Faz download do arquivo]
    WebShare --> End([Arquivo compartilhado])
    Download --> End
```
