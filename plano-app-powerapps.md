# Plano de Implementação — Power Apps Canvas (F3 detalhada)

**Referência visual:** `_revisar/visualizacao/mockup-desktop.html` (5 telas)
**Base de dados:** Dataverse — 8 tabelas do plano v3.2/v4 (`docs/plano-implementacao.md`)
**Formato do app:** Tablet / paisagem (1366×768), tema TRANSPETRO (navy `#002D72`, green `#00A651`, yellow `#FFD100`)

---

## 0. Premissas de dados e o problema da delegação

Volumes atuais (M&C 05/06):

| Tabela | Linhas | Cabe em coleção? | Estratégia |
|---|---|---|---|
| tbl_dItensMed | 47 | ✅ inteira | Coleção no **OnStart** |
| tbl_dProjetos | 617 | ✅ inteira (ou filtrada) | Coleção no **OnStart** |
| tbl_EntregaveisProjeto | 2.719 | ❌ inteira / ✅ por gestor | Filtrar por `Gestor` no **OnStart** |
| tbl_LB | 5.334 | ❌ | Filtrar por `GES` no **OnVisible** (Detalhe) |
| tbl_Outlook | 7.973 | ❌ | Filtrar por `GES` + `Mes_Ref` no **OnVisible** |
| tbl_ControleMed | 1.376 | ⚠️ por gestor | Filtrar por `Gestor` no **OnStart**; refresh no OnVisible |
| tbl_Desvios | 60 | ✅ inteira | Coleção no **OnStart** |
| tbl_Med_RM | 6.611 | ❌ | **Fora do MVP** (só visão CSE Anual — ver Etapa 7) |

**Regras de ouro adotadas no app inteiro:**

1. **Limite de linhas por consulta: subir de 500 → 2000** (Configurações → Geral → "Limite de linhas de dados"). Mesmo assim, nenhuma consulta pode *depender* de trazer mais de 2000 linhas.
2. **Galerias nunca apontam direto para Dataverse** — sempre para coleções locais. Dataverse só é tocado em `ClearCollect`/`Patch`/`LookUp` pontuais.
3. **Filtros delegáveis primeiro** (`=`, `&&`, `StartsWith` sobre colunas indexadas: `Gestor`, `GES`, `Mes_Ref`, `Status_Med`). Funções não-delegáveis (`in` com coleção, `AddColumns`, `Search`) só **depois**, sobre a coleção local.
4. **OnStart carrega o "universo do usuário"** (dimensões pequenas + dados filtrados pelo gestor logado). **OnVisible carrega o "contexto da tela"** (dados do item/projeto selecionado, agregados do mês).
5. `Concurrent()` em todos os blocos de `ClearCollect` paralelos.
6. **OnStart enxuto (< 8s)**: nada de `Navigate()`; tela inicial via `App.StartScreen`.

---

## Etapa 1 — Fundação: app, conexões, tema e shell *(F3.6.1–F3.6.3)*

- [ ] **E1.1** — Criar Canvas app tablet 1366×768 na solution DEV; desativar "Scale to fit".
- [ ] **E1.2** — Conectar as 7 tabelas Dataverse (todas menos tbl_Med_RM) + conector Office365Users + SharePoint (biblioteca de evidências).
- [ ] **E1.3** — Tema como **fórmulas nomeadas** (`App.Formulas`):
  ```
  fxNavy   = ColorValue("#002D72");
  fxNavy2  = ColorValue("#003A8C");
  fxGreen  = ColorValue("#00A651");
  fxYellow = ColorValue("#FFD100");
  fxBlue   = ColorValue("#1E6BB8");
  fxOrange = ColorValue("#E67E22");
  fxRed    = ColorValue("#C0392B");
  fxBg     = ColorValue("#F0F4FA");
  fxSub    = ColorValue("#6B7C9A");
  fxBorder = ColorValue("#DDE4F0")
  ```
- [ ] **E1.4** — **Componente cmpSidebar** (largura 220, navy): logo, avatar/nome do usuário, itens de menu com `varTelaAtiva` (custom property de entrada) e output property `OnSelectItem` → o mockup usa borda amarela à esquerda no item ativo.
- [ ] **E1.5** — **Componente cmpTopbar**: breadcrumb + título (propriedades de entrada), seletor de mês (◀ Mês ▶ alterando `varMesRef`), botão "Registrar Medição", sino de notificações.
- [ ] **E1.6** — Criar as 5 telas vazias com sidebar + topbar: `scrHome`, `scrDetalhe`, `scrForm`, `scrRadar`, `scrPainel`.
- [ ] **E1.7** — `App.StartScreen = scrHome`.

**Critério de saída:** navegação completa entre as 5 telas com visual do mockup (sem dados).

---

## Etapa 2 — App.OnStart: identidade, perfil e coleções globais *(F3.6.2)*

```
// ── 1. Identidade ──
Set(varEmail, Lower(User().Email));
Set(varNome, User().FullName);
Set(varMesRef, Date(Year(Today()), Month(Today()), 1));   // mês de competência

// ── 2. Dimensões pequenas + dados do gestor (paralelo) ──
Concurrent(
    // 47 linhas — catálogo do contrato
    ClearCollect(colItensMed, tbl_dItensMed),

    // 617 linhas — cadastro mestre (cabe; se crescer, filtrar por Gestor)
    ClearCollect(colProjetos, tbl_dProjetos),

    // Entregáveis do gestor logado (filtro delegável)
    ClearCollect(colMinhasEntregas,
        Filter(tbl_EntregaveisProjeto, Lower(Gestor) = varEmail)),

    // Medições do gestor (histórico completo dele)
    ClearCollect(colMinhasMedicoes,
        Filter(tbl_ControleMed, Lower(Gestor) = varEmail)),

    // 60 linhas — desvios (inteira)
    ClearCollect(colDesvios, tbl_Desvios)
);

// ── 3. Perfil: Gestor / Gerente / CSE ──
Set(varPerfil,
    LookUp(colPerfis, Lower(Email) = varEmail, Perfil));   // ver E2.3

// ── 4. Tabelas estáticas ──
ClearCollect(colMotivosAtraso,
    ["— Sem atraso neste período —", "Parada operacional programada",
     "Aguardando aprovação do cliente", "Pendência de suprimento PETROBRAS",
     "Rescisão/substituição contratual", "Problema técnico / engenharia",
     "Outro (detalhar em Plano de Recuperação)"]);
ClearCollect(colStatus,
    ["Medido", "Planejado", "Postergado", "Paralisado", "Cancelado"]);
```

- [ ] **E2.1** — Implementar OnStart acima; medir tempo de carga (meta < 8s).
- [ ] **E2.2** — Validar que `Gestor` em tbl_EntregaveisProjeto/tbl_ControleMed contém **e-mail** comparável a `User().Email` (a base usa `tbl_GestoresEmail`; se a coluna for nome, criar coluna de e-mail no Dataverse — pré-requisito desta etapa).
- [ ] **E2.3** — Tabela de perfis (`tbl_Perfis`: Email, Perfil ∈ {Gestor, Gerente, CSE}) ou regra fixa por grupo AD; carregar em `colPerfis` dentro do `Concurrent`.
- [ ] **E2.4** — Caso CSE/Gerente: OnStart adicional condicional (`If(varPerfil <> "Gestor", ...)`) — não carregar dados gerenciais para gestor comum.

**Critério de saída:** app abre com coleções carregadas, perfil identificado, monitor (Power Apps Monitor) sem consultas > 2000 linhas.

---

## Etapa 3 — Tela 1: Home do Gestor *(F3.1)*

**OnVisible da scrHome** — recorte do mês + enriquecimento local:

```
// Outlook e LB do mês corrente APENAS dos meus GES — N pequeno (ForAll local)
ClearCollect(colEntregasMes,
    AddColumns(
        Filter(colMinhasEntregas, /* entrega ativa */ true),
        DataLB2,    LookUp(tbl_LB, GES = ThisRecord.GES
                        && ID_iTENSMED = ThisRecord.ID_iTENSMED
                        && Nr_SubItem = ThisRecord.Nr_SubItem
                        && Tipo_Baseline = "LB-2", Data_Med),
        OutlookAtual, LookUp(tbl_Outlook, GES = ThisRecord.GES
                        && ID_iTENSMED = ThisRecord.ID_iTENSMED
                        && Nr_SubItem = ThisRecord.Nr_SubItem
                        && Mes_Ref = varMesRefTexto, Outlook),
        StatusMes,  Coalesce(LookUp(colMinhasMedicoes,
                        GES = ThisRecord.GES
                        && ID_iTENSMED = ThisRecord.ID_iTENSMED
                        && Nr_SubItem = ThisRecord.Nr_SubItem
                        && Text(Data_Med, "yyyy-mm") = Text(varMesRef, "yyyy-mm"),
                        Status_Med), "Planejado")
    )
);
UpdateContext({ctxFiltroStatus: "Todos", ctxBusca: ""})
```

> ⚠️ Os `LookUp` em tbl_LB/tbl_Outlook são pontuais e delegáveis (igualdade em colunas indexadas); como rodam dentro de `AddColumns` sobre ~8–60 entregas do gestor, o custo é aceitável. Se o gestor tiver > 100 entregas, trocar por: `ClearCollect(colLBGestor, Filter(tbl_LB, GES in <lista>))` por GES via `ForAll` e fazer o join 100% local.

- [ ] **E3.1** — KPI row (4 cards): `CountRows(Filter(colEntregasMes, StatusMes = "Medido"))` etc., borda superior colorida por status.
- [ ] **E3.2** — Banner de alerta: entregas com `OutlookAtual` nos próximos 5 dias e sem medição → texto + botão "Ver pendentes" (aplica filtro).
- [ ] **E3.3** — Galeria principal (colunas do mockup: Projeto, GES/GT, Tipo, LB-2, Outlook, Desvio, Valor, Status badge, link "Ver →"). Fonte: 
  ```
  Search(Filter(colEntregasMes,
      ctxFiltroStatus = "Todos" || StatusMes = ctxFiltroStatus),
      ctxBusca, NomeProjeto)
  ```
- [ ] **E3.4** — Chips de filtro (Todos / Planejados / Em Risco / Medidos) com contagem dinâmica.
- [ ] **E3.5** — Badge de status como componente `cmpBadge` (cores do mockup).
- [ ] **E3.6** — Painel direito: "Resumo do Mês" (barras % por status sobre colEntregasMes), "Financeiro" (somas de `Valor_Atualizado` por status), "Ações Rápidas".
- [ ] **E3.7** — `OnSelect` da linha: `Set(varItemSel, ThisItem); Navigate(scrDetalhe)`.

**Critério de saída:** Home reproduz o mockup com dados reais do gestor logado; troca de mês no topbar refaz `colEntregasMes`.

---

## Etapa 4 — Tela 2: Detalhe do Projeto *(F3.2)*

**OnVisible da scrDetalhe** — contexto do item selecionado (tudo filtrado por `varItemSel`):

```
Concurrent(
    ClearCollect(colLBItem,
        Filter(tbl_LB, GES = varItemSel.GES
            && ID_iTENSMED = varItemSel.ID_iTENSMED
            && Nr_SubItem = varItemSel.Nr_SubItem)),
    ClearCollect(colOutlookItem,
        Filter(tbl_Outlook, GES = varItemSel.GES
            && ID_iTENSMED = varItemSel.ID_iTENSMED
            && Nr_SubItem = varItemSel.Nr_SubItem)),
    ClearCollect(colHistMed,
        SortByColumns(
            Filter(tbl_ControleMed, GES = varItemSel.GES
                && ID_iTENSMED = varItemSel.ID_iTENSMED
                && Nr_SubItem = varItemSel.Nr_SubItem),
            "Data_Med", SortOrder.Descending)),
    ClearCollect(colDesviosItem,
        Filter(colDesvios, GES = varItemSel.GES
            && ID_iTENSMED = varItemSel.ID_iTENSMED))
);
Set(varProjetoSel, LookUp(colProjetos, GES = varItemSel.GES))
```

- [ ] **E4.1** — Hero card navy: NomeProjeto, GES, CodigoSAP, GT/SubPEP, badge de status, chips (GG, Regional, Empresa, Linha de Base).
- [ ] **E4.2** — Progress steps (Cadastrado → Planejado LB-2 → Outlook confirmado → Evidência enviada → Aprovado): derivado de `colLBItem`/`colHistMed`/status de aprovação.
- [ ] **E4.3** — Grid de datas: LB-2, LB-9 (de colLBItem) + Outlooks (de colOutlookItem, pivotado por `Mes_Ref`/`Versao_Origem`) + Data de Medição (colHistMed).
- [ ] **E4.4** — Card Entregável/PPU: dados de `colItensMed` (`LookUp` local por ID_iTENSMED).
- [ ] **E4.5** — Timeline "Histórico de Medições": galeria sobre `colHistMed`.
- [ ] **E4.6** — Painel direito Financeiro: Valor_Bruto, Valor_Atualizacao, Valor_Desonerado (colItensMed) + Valor_Atualizado (entrega).
- [ ] **E4.7** — Card Observações/Motivo de atraso: último registro de `colDesviosItem`.
- [ ] **E4.8** — Botão "Registrar Nova Medição" → `Navigate(scrForm)` mantendo `varItemSel`.

**Critério de saída:** detalhe abre em < 2s a partir da Home, todas as seções alimentadas pelas 4 coleções de contexto.

---

## Etapa 5 — Tela 3: Formulário de Medição *(F3.3)* — coração do app

**OnVisible da scrForm:**

```
UpdateContext({
    ctxStatus: "Medido",
    ctxDataMed: Today(),
    ctxNovoOutlook: Blank(),
    ctxMotivo: First(colMotivosAtraso).Value,
    ctxPlanoRec: "", ctxObs: "",
    ctxAnexoOk: false
});
// referência: reaproveita colLBItem/colOutlookItem se veio do Detalhe;
// se veio direto da Home, recarrega o contexto do item (mesmo bloco da Etapa 4)
If(IsEmpty(colLBItem) || First(colLBItem).GES <> varItemSel.GES,
    /* repetir Concurrent da Etapa 4 */ );
```

- [ ] **E5.1** — Seletor de status: 5 botões coloridos (galeria horizontal sobre `colStatus`), seleção em `ctxStatus`.
- [ ] **E5.2** — Campos condicionais: 
  - `Medido` → Data de Medição obrigatória + evidência obrigatória;
  - `Postergado/Paralisado` → Nova data de Outlook + Motivo + Plano de Recuperação obrigatórios;
  - `Cancelado` → Motivo obrigatório.
- [ ] **E5.3** — Barra de progresso de preenchimento (% de campos obrigatórios completos — igual ao mockup).
- [ ] **E5.4** — Upload de evidência: controle Attachments → SharePoint (Power Automate "SalvarEvidencia" retorna URL) → gravar caminho em `Evidencia`.
- [ ] **E5.5** — Painel direito: "Referência do Projeto" (colProjetos/colItensMed), "Datas de Referência" (colLBItem/colOutlookItem), card de prazo ("X dias restantes" = `DateDiff(Today(), OutlookAtual)`), "Fluxo de Aprovação" (estático no MVP).
- [ ] **E5.6** — **Submit (`Patch` duplo):**
  ```
  Patch(tbl_ControleMed, Defaults(tbl_ControleMed), {
      GES: varItemSel.GES, ID_iTENSMED: varItemSel.ID_iTENSMED,
      Nr_SubItem: varItemSel.Nr_SubItem,
      Data_Med: ctxDataMed, Status_Med: ctxStatus,
      Gestor: varEmail, Evidencia: varUrlEvidencia });
  If(ctxStatus in ["Postergado","Paralisado","Cancelado"],
      Patch(tbl_Desvios, Defaults(tbl_Desvios), {
          GES: varItemSel.GES, ID_iTENSMED: varItemSel.ID_iTENSMED,
          Nr_SubItem: varItemSel.Nr_SubItem, Data_Just: Today(),
          Categoria_Desvio: ctxMotivo, Justificativa: ctxObs,
          Plano_Recuperacao: ctxPlanoRec }));
  // sincronizar coleções locais sem reconsultar tudo:
  Collect(colMinhasMedicoes, <registro criado>);
  Notify("Medição registrada ✓", NotificationType.Success);
  Navigate(scrHome)
  ```
- [ ] **E5.7** — "Salvar Rascunho": `SaveData/LoadData` local (ou status `Rascunho` se a PO preferir persistir).
- [ ] **E5.8** — Validações com feedback visual (borda vermelha + mensagem) antes de habilitar "Enviar para Aprovação".
- [ ] **E5.9** — Disparar flow de aprovação (F4) após Patch — placeholder no MVP.

**Critério de saída:** ciclo completo Home → Detalhe → Form → Patch → Home refletindo o novo status **sem** `ClearCollect` da base inteira (atualização incremental das coleções).

---

## Etapa 6 — Tela 4: Radar de Oportunidades *(F3.4)*

Dados novos (não vêm do M&C) → tabela própria `tbl_Oportunidades` no Dataverse (GES futuro/null, NomeProjeto, Fase ∈ {Avaliação, Conceitual, Básico}, GG, Regional, Gestor, InicioPrevisto, PrimeiraMedicaoEst, ValorEstimado).

**OnVisible:** `ClearCollect(colOportunidades, tbl_Oportunidades)` — volume pequeno (~20–100), coleção inteira.

- [ ] **E6.1** — Criar `tbl_Oportunidades` no Dataverse (escrita exclusiva do app — padrão "Via APP").
- [ ] **E6.2** — KPI cards: contagens por fase + `Sum(ValorEstimado)`.
- [ ] **E6.3** — Galeria com badge de fase (cores do mockup: amarelo/verde/azul) + filtros por chips + "Meu GG".
- [ ] **E6.4** — "+ Nova Oportunidade": form em painel/modal → `Patch(tbl_Oportunidades, Defaults(...), {...})` + `Collect(colOportunidades, ...)`.
- [ ] **E6.5** — Painel direito: "Pipeline por Fase" e "Por Grupo Gerencial" (GroupBy local sobre colOportunidades).

---

## Etapa 7 — Tela 5: Painel Gerencial *(F3.5)* — onde o limite de dados morde

O painel precisa de **agregados sobre 2.719 entregáveis × 1.376 medições** — não dá para colecionar tudo. Estratégia em duas opções:

**Opção A (recomendada): tabela agregada `tbl_ResumoMensal`** alimentada por Power Automate/Dataflow (job noturno): 1 linha por `Mes_Ref × GG × Regional × Gestor` com contagens por status e somas de valor (~40 gestores × 13 meses ≈ 500 linhas). O app coleciona tudo no OnVisible. Donut, % por GG, % por Regional e Ranking viram `GroupBy`/`Sum` locais — instantâneo e sem delegação.

**Opção B (fallback sem flow):** agregações direto no Dataverse via `CountIf`-like delegável por fatia: `ForAll(colStatus, Collect(colKPIGeral, {Status: Value, Qtd: CountRows(Filter(tbl_ControleMed, Status_Med = Value && Mes_Comp = varMesRefTexto))}))` — `CountRows(Filter(...))` é delegável no Dataverse. Funciona para KPIs/donut; ranking por gestor exige ~40 consultas (lento, mas viável).

**OnVisible da scrPainel (Opção A):**
```
If(varPerfil = "Gestor", Navigate(scrHome));   // guarda de perfil
Concurrent(
    ClearCollect(colResumo, Filter(tbl_ResumoMensal, Mes_Ref = varMesRefTexto)),
    ClearCollect(colResumoHist, tbl_ResumoMensal)   // série p/ tendências (~500)
);
```

- [ ] **E7.1** — Decidir A vs B com a PO; se A, criar `tbl_ResumoMensal` + flow agregador.
- [ ] **E7.2** — KPI row (Medido / Planejado / Postergado / Paralisado, com R$).
- [ ] **E7.3** — Donut "Status Geral" (gráfico de rosca nativo ou composição de arcos como no mockup).
- [ ] **E7.4** — Cards "% Medido por GG" e "% por Regional GPP" (GroupBy de colResumo).
- [ ] **E7.5** — "Financeiro Consolidado" (somas de colResumo).
- [ ] **E7.6** — "Ranking de Aderência — Gestores": `SortByColumns(GroupBy(colResumo, ...), "PctMedido", SortOrder.Descending)` com medalhas 1/2/3.
- [ ] **E7.7** — Card "Dados PPM — Fonte": última atualização (timestamp do flow), nº projetos, nº registros.
- [ ] **E7.8** — Visão CSE Anual (tbl_Med_RM): **fora do MVP** — apontar para o relatório Power BI existente (botão `Launch()` para o report).

---

## Etapa 8 — Qualidade, performance e publicação *(F3.6.4–F3.6.5)*

- [ ] **E8.1** — Power Apps Monitor: nenhuma consulta retornando 2000 linhas (sinal de estouro silencioso); OnStart < 8s; OnVisible < 2s.
- [ ] **E8.2** — Avisos de delegação: zerar (ou justificar um a um — só são aceitáveis sobre coleções locais).
- [ ] **E8.3** — Tratamento de erros: `IfError` nos Patch + `Notify`; ativar "Formula-level error management".
- [ ] **E8.4** — Acessibilidade básica: AccessibleLabel nos botões/ícones, contraste dos badges.
- [ ] **E8.5** — Teste com 3 gestores piloto (telas Home, Detalhe, Form — prioridade do plano F3.6.4).
- [ ] **E8.6** — Botão/gesto de **Refresh manual** (re-executa o bloco do OnStart via `Select(btnRecarregar)` — OnStart não roda de novo sozinho).
- [ ] **E8.7** — Publicar em DEV dentro da solution; documentar versão e checklist de saída.

---

## Resumo: quem carrega o quê

| Momento | Coleções | Volume típico |
|---|---|---|
| **App.OnStart** | colItensMed, colProjetos, colMinhasEntregas, colMinhasMedicoes, colDesvios, colPerfis, colMotivosAtraso, colStatus | 47 + 617 + ~8–60 + ~50 + 60 |
| **scrHome.OnVisible** | colEntregasMes (join local LB/Outlook/Status do mês) | = nº entregas do gestor |
| **scrDetalhe.OnVisible** | colLBItem, colOutlookItem, colHistMed, colDesviosItem | < 30 linhas total |
| **scrForm.OnVisible** | reaproveita as do Detalhe + contexto do form | 0 consultas novas |
| **scrRadar.OnVisible** | colOportunidades | ~20–100 |
| **scrPainel.OnVisible** | colResumo, colResumoHist (tabela agregada) | ~40 + ~500 |

**Dependências externas antes de começar:** coluna `Gestor` com e-mail (E2.2) · tabela de perfis (E2.3) · `tbl_Oportunidades` (E6.1) · decisão A/B do painel (E7.1) · biblioteca SharePoint de evidências (E5.4).
