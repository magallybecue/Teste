# Plano de Implementação — Power Apps Canvas (F3 detalhada)

**Referência visual:** `_revisar/visualizacao/mockup-desktop.html` (5 telas)
**Base de dados:** Dataverse — 8 tabelas do plano v3.2/v4
**Formato do app:** Tablet / paisagem (1366×768), tema TRANSPETRO (navy `#002D72`, green `#00A651`, yellow `#FFD100`)

---

## Decisões de arquitetura consolidadas

| Item | Decisão |
|---|---|
| E2.2 — Campo Gestor | Já existe em `tbl_dProjetos` como campo Person/Group do M365. Confirmar que `tbl_EntregaveisProjeto` e `tbl_ControleMed` usam o **mesmo tipo Person** (não texto livre) para garantir delegação. |
| E2.3 — Perfis | `tbl_Perfis` deve ser criada no Dataverse (Email + Choice: Gestor/Gerente/CSE). |
| E5.4 — Evidências | Biblioteca SharePoint + flow `SalvarEvidencia` devem ser criados antes do piloto do formulário. |
| E7.1 — Painel Gerencial | **Fora do MVP.** A `scrPainel` exibe apenas botão `Launch()` apontando para o relatório Power BI existente. Etapas E7.2–E7.8 canceladas. |

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
| tbl_Med_RM | 6.611 | ❌ | **Fora do MVP** (visão CSE Anual — Power BI) |

**Regras de ouro adotadas no app inteiro:**

1. **Limite de linhas por consulta: subir de 500 → 2000** (Configurações → Geral → "Limite de linhas de dados").
2. **Galerias nunca apontam direto para Dataverse** — sempre para coleções locais.
3. **Filtros delegáveis primeiro** (`=`, `&&`, `StartsWith` sobre colunas indexadas: `Gestor`, `GES`, `Mes_Ref`, `Status_Med`). Funções não-delegáveis só **depois**, sobre coleção local.
4. **OnStart carrega o "universo do usuário"**. **OnVisible carrega o "contexto da tela"**.
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
  fxBorder = ColorValue("#DDE4F0");
  fxMesRefTexto = Text(varMesRef, "yyyy-mm")
  ```
- [ ] **E1.4** — **Componente cmpSidebar** (largura 220, navy): logo, avatar/nome do usuário, itens de menu com `varTelaAtiva` e output property `OnSelectItem` → borda amarela à esquerda no item ativo.
- [ ] **E1.5** — **Componente cmpTopbar**: breadcrumb + título, seletor de mês (◀ Mês ▶ alterando `varMesRef`), botão "Registrar Medição", sino de notificações.
- [ ] **E1.6** — Criar as 5 telas vazias com sidebar + topbar: `scrHome`, `scrDetalhe`, `scrForm`, `scrRadar`, `scrPainel`.
- [ ] **E1.7** — `App.StartScreen = scrHome`.

**Critério de saída:** navegação completa entre as 5 telas com visual do mockup (sem dados).

---

## Etapa 2 — App.OnStart: identidade, perfil e coleções globais *(F3.6.2)*

```
// ── 1. Identidade ──
Set(varEmail, Lower(User().Email));
Set(varNome, User().FullName);
Set(varMesRef, Date(Year(Today()), Month(Today()), 1));

// ── 2. Dimensões pequenas + dados do gestor (paralelo) ──
Concurrent(
    ClearCollect(colItensMed, tbl_dItensMed),
    ClearCollect(colProjetos, tbl_dProjetos),
    ClearCollect(colMinhasEntregas,
        Filter(tbl_EntregaveisProjeto, Lower(Gestor) = varEmail)),
    ClearCollect(colMinhasMedicoes,
        Filter(tbl_ControleMed, Lower(Gestor) = varEmail)),
    ClearCollect(colDesvios, tbl_Desvios),
    ClearCollect(colPerfis, tbl_Perfis)
);

// ── 3. Perfil ──
Set(varPerfil, LookUp(colPerfis, Lower(Email) = varEmail, Perfil));

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
- [ ] **E2.2** — Confirmar que campo `Gestor` em `tbl_EntregaveisProjeto`/`tbl_ControleMed` é tipo **Person/Group** (vem do M365, igual ao `tbl_dProjetos`). Se for texto, criar coluna de e-mail calculada no Dataverse.
- [ ] **E2.3** — Criar `tbl_Perfis` no Dataverse (Email: Text indexado, Perfil: Choice {Gestor, Gerente, CSE}); populá-la com os usuários do piloto.
- [ ] **E2.4** — Dados gerenciais condicionais: `If(varPerfil <> "Gestor", /* carregar extras */)` — não executar para gestor comum.

**Critério de saída:** app abre com coleções carregadas, perfil identificado, Monitor sem consultas > 2000 linhas.

---

## Etapa 3 — Tela 1: Home do Gestor *(F3.1)*

**OnVisible da scrHome:**

```
ClearCollect(colLBGestor,
    Filter(tbl_LB, GES in ShowColumns(colMinhasEntregas, "GES")));
ClearCollect(colOutlookGestor,
    Filter(tbl_Outlook, GES in ShowColumns(colMinhasEntregas, "GES")
        && Mes_Ref = fxMesRefTexto));

ClearCollect(colEntregasMes,
    AddColumns(
        Filter(colMinhasEntregas, true),
        DataLB2,      LookUp(colLBGestor, GES = ThisRecord.GES
                          && ID_iTENSMED = ThisRecord.ID_iTENSMED
                          && Nr_SubItem = ThisRecord.Nr_SubItem
                          && Tipo_Baseline = "LB-2", Data_Med),
        OutlookAtual, LookUp(colOutlookGestor, GES = ThisRecord.GES
                          && ID_iTENSMED = ThisRecord.ID_iTENSMED
                          && Nr_SubItem = ThisRecord.Nr_SubItem, Outlook),
        StatusMes,    Coalesce(LookUp(colMinhasMedicoes,
                          GES = ThisRecord.GES
                          && ID_iTENSMED = ThisRecord.ID_iTENSMED
                          && Nr_SubItem = ThisRecord.Nr_SubItem
                          && Text(Data_Med, "yyyy-mm") = fxMesRefTexto,
                          Status_Med), "Planejado")
    )
);
UpdateContext({ctxFiltroStatus: "Todos", ctxBusca: ""})
```

> ⚠️ LookUps são agora sobre **coleções locais** (colLBGestor/colOutlookGestor) — sem round-trips ao Dataverse dentro do AddColumns.

- [ ] **E3.1** — KPI row (4 cards): contagens por status com borda superior colorida.
- [ ] **E3.2** — Banner de alerta: entregas com `OutlookAtual` nos próximos 5 dias sem medição.
- [ ] **E3.3** — Galeria principal (Projeto, GES/GT, Tipo, LB-2, Outlook, Desvio, Valor, Status badge, "Ver →").
- [ ] **E3.4** — Chips de filtro (Todos / Planejados / Em Risco / Medidos) com contagem dinâmica.
- [ ] **E3.5** — Componente `cmpBadge` (cores do mockup).
- [ ] **E3.6** — Painel direito: "Resumo do Mês", "Financeiro", "Ações Rápidas".
- [ ] **E3.7** — `OnSelect` da linha: `Set(varItemSel, ThisItem); Navigate(scrDetalhe)`.

**Critério de saída:** Home reproduz o mockup com dados reais; troca de mês no topbar refaz `colEntregasMes`.

---

## Etapa 4 — Tela 2: Detalhe do Projeto *(F3.2)*

**OnVisible da scrDetalhe:**

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

- [ ] **E4.1** — Hero card navy: NomeProjeto, GES, CodigoSAP, GT/SubPEP, badge de status, chips.
- [ ] **E4.2** — Progress steps (Cadastrado → Planejado LB-2 → Outlook confirmado → Evidência enviada → Aprovado).
- [ ] **E4.3** — Grid de datas: LB-2, LB-9 (colLBItem) + Outlooks (colOutlookItem) + Data de Medição (colHistMed).
- [ ] **E4.4** — Card Entregável/PPU: `LookUp(colItensMed, ID_iTENSMED = varItemSel.ID_iTENSMED)`.
- [ ] **E4.5** — Timeline "Histórico de Medições": galeria sobre `colHistMed`.
- [ ] **E4.6** — Painel direito Financeiro: Valor_Bruto, Valor_Atualizacao, Valor_Desonerado, Valor_Atualizado.
- [ ] **E4.7** — Card Observações/Motivo de atraso: `Last(colDesviosItem)`.
- [ ] **E4.8** — Botão "Registrar Nova Medição" → `Navigate(scrForm)`.

**Critério de saída:** detalhe abre em < 2s; todas as seções alimentadas pelas 4 coleções de contexto.

---

## Etapa 5 — Tela 3: Formulário de Medição *(F3.3)*

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
If(IsEmpty(colLBItem) || First(colLBItem).GES <> varItemSel.GES,
    /* repetir Concurrent da Etapa 4 */ );
```

- [ ] **E5.1** — Seletor de status: 5 botões coloridos (galeria horizontal sobre `colStatus`).
- [ ] **E5.2** — Campos condicionais por status:
  - `Medido` → Data de Medição + evidência (obrigatórios);
  - `Postergado/Paralisado` → Nova data Outlook + Motivo + Plano de Recuperação (obrigatórios);
  - `Cancelado` → Motivo (obrigatório).
- [ ] **E5.3** — Barra de progresso de preenchimento (% de campos obrigatórios completos).
- [ ] **E5.4** — Upload de evidência via controle Attachments → Power Automate `SalvarEvidencia` → URL gravada em `varUrlEvidencia`.
- [ ] **E5.5** — Painel direito: referência do projeto, datas LB/Outlook, card de prazo (`DateDiff(Today(), OutlookAtual)`).
- [ ] **E5.6** — **Submit (usar Power Automate para atomicidade):**
  ```
  // Chamar flow "RegistrarMedicao" que faz os dois Patches atomicamente:
  // 1. Patch tbl_ControleMed
  // 2. Se status in [Postergado/Paralisado/Cancelado]: Patch tbl_Desvios
  Set(varResultado,
      MedicaoFlow.Run(varItemSel.GES, varItemSel.ID_iTENSMED,
          varItemSel.Nr_SubItem, ctxStatus, Text(ctxDataMed,"yyyy-mm-dd"),
          varEmail, varUrlEvidencia, ctxMotivo, ctxObs, ctxPlanoRec));
  If(varResultado.status = "ok",
      Collect(colMinhasMedicoes, varResultado.registro);
      Notify("Medição registrada ✓", NotificationType.Success);
      Navigate(scrHome),
      Notify("Erro ao registrar: " & varResultado.erro, NotificationType.Error)
  )
  ```
- [ ] **E5.7** — "Salvar Rascunho": gravar status `Rascunho` na `tbl_ControleMed` (persistência cross-device).
- [ ] **E5.8** — Validações com feedback visual (borda vermelha + mensagem) antes de habilitar "Enviar".
- [ ] **E5.9** — Placeholder para flow de aprovação (F4) — chamar flow após submit bem-sucedido.

**Critério de saída:** ciclo completo Home → Detalhe → Form → Patch → Home refletindo o novo status sem `ClearCollect` da base inteira.

---

## Etapa 6 — Tela 4: Radar de Oportunidades *(F3.4)*

**OnVisible:** `ClearCollect(colOportunidades, tbl_Oportunidades)` — volume ~20–100 linhas.

- [ ] **E6.1** — Criar `tbl_Oportunidades` no Dataverse: GES, NomeProjeto, Fase (Choice: Avaliação/Conceitual/Básico), GG, Regional, Gestor, InicioPrevisto, PrimeiraMedicaoEst, ValorEstimado.
- [ ] **E6.2** — KPI cards: contagens por fase + `Sum(colOportunidades, ValorEstimado)`.
- [ ] **E6.3** — Galeria com badge de fase (amarelo/verde/azul) + chips de filtro + "Meu GG".
- [ ] **E6.4** — "+ Nova Oportunidade": form em painel/modal → `Patch(tbl_Oportunidades, ...)` + `Collect(colOportunidades, ...)`.
- [ ] **E6.5** — Painel direito: "Pipeline por Fase" e "Por Grupo Gerencial" (`GroupBy` local).

**Critério de saída:** CRUD completo de oportunidades funcionando offline (coleção local) com sync ao fechar o painel.

---

## Etapa 7 — Tela 5: Painel Gerencial *(simplificada — Power BI)*

> **Decisão de escopo:** agregações gerenciais ficam no Power BI existente. A tela no app é apenas um ponto de acesso.

- [ ] **E7.1** — Guarda de perfil: `If(varPerfil = "Gestor", Navigate(scrHome))`.
- [ ] **E7.2** — Layout simples: título "Painel Gerencial", subtítulo "Visualização completa disponível no Power BI", botão "Abrir Painel BI" → `Launch("<URL_REPORT_BI>", {}, LaunchTarget.New)`.
- [ ] **E7.3** — Card informativo: última atualização dos dados (buscar timestamp de `tbl_ResumoMensal` ou variável de controle).

**Critério de saída:** botão abre o relatório BI na aba correta; gestores são redirecionados para Home.

---

## Etapa 8 — Qualidade, performance e publicação *(F3.6.4–F3.6.5)*

- [ ] **E8.1** — Power Apps Monitor: nenhuma consulta retornando 2000 linhas; OnStart < 8s; OnVisible < 2s.
- [ ] **E8.2** — Avisos de delegação: zerar (ou justificar — só aceitáveis sobre coleções locais).
- [ ] **E8.3** — Tratamento de erros: `IfError` nos Patch + `Notify`; ativar "Formula-level error management".
- [ ] **E8.4** — Acessibilidade: `AccessibleLabel` nos botões/ícones, contraste dos badges.
- [ ] **E8.5** — Teste com 3 gestores piloto (telas Home, Detalhe, Form).
- [ ] **E8.6** — Botão de **Refresh manual**: re-executa o bloco do OnStart via `Select(btnRecarregar)`.
- [ ] **E8.7** — Publicar em DEV dentro da solution; documentar versão e checklist de saída.

---

## Resumo: quem carrega o quê

| Momento | Coleções | Volume típico |
|---|---|---|
| **App.OnStart** | colItensMed, colProjetos, colMinhasEntregas, colMinhasMedicoes, colDesvios, colPerfis, colMotivosAtraso, colStatus | 47 + 617 + ~8–60 + ~50 + 60 |
| **scrHome.OnVisible** | colLBGestor, colOutlookGestor, colEntregasMes | = nº GES do gestor × mês |
| **scrDetalhe.OnVisible** | colLBItem, colOutlookItem, colHistMed, colDesviosItem | < 30 linhas total |
| **scrForm.OnVisible** | reaproveita as do Detalhe + contexto do form | 0 consultas novas |
| **scrRadar.OnVisible** | colOportunidades | ~20–100 |
| **scrPainel.OnVisible** | (nenhuma — só Launch para BI) | 0 |

---

## Dependências externas antes de começar

| Dependência | Responsável | Bloqueante para |
|---|---|---|
| Campo `Gestor` tipo Person/Group em `tbl_EntregaveisProjeto` e `tbl_ControleMed` | Admin Dataverse | Etapas 2, 3, 4, 5 |
| `tbl_Perfis` criada e populada | Admin Dataverse | Etapa 2 |
| Biblioteca SharePoint de evidências criada | Admin SharePoint | Etapa 5 |
| Flow `SalvarEvidencia` criado e publicado | Dev Power Automate | Etapa 5 |
| Flow `RegistrarMedicao` (Patch atômico) criado | Dev Power Automate | Etapa 5 |
| URL do relatório Power BI | Analista BI | Etapa 7 |
| `tbl_Oportunidades` criada no Dataverse | Admin Dataverse | Etapa 6 |
