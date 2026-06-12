# Guia de Tarefas — Power Apps Canvas M&C TRANSPETRO

> **Como usar este guia:** cada tarefa tem pré-requisitos, passo a passo no Power Apps Studio, código pronto para colar e critério de verificação. Execute na sequência indicada — tarefas marcadas com ⚡ podem ser feitas em paralelo com a anterior.

---

## ETAPA 1 — Fundação

### T1.1 — Criar o Canvas App na solution DEV

**Pré-requisitos:** acesso de maker no ambiente DEV; solution criada.

**Passo a passo:**
1. Acesse [make.powerapps.com](https://make.powerapps.com) → selecione o ambiente DEV.
2. Abra a solution → **Novo → App → Canvas app**.
3. Nome: `AppMC_TRANSPETRO` | Formato: **Tablet**.
4. Após abrir o Studio: **Arquivo → Configurações → Tela → desmarcar "Dimensionar para ajustar"**.
5. Confirmar resolução 1366×768 em **Tamanho da tela personalizado**.
6. Salvar e fechar. A app aparece na solution.

**Verificação:** app abre em branco sem barras de escala lateral/inferior.

---

### T1.2 — Conectar fontes de dados

**Pré-requisitos:** tabelas Dataverse existem no ambiente; biblioteca SharePoint criada.

**Passo a passo:**
1. No Studio: painel esquerdo → **Dados → Adicionar dados**.
2. Buscar e adicionar uma a uma:
   - `tbl_dItensMed`, `tbl_dProjetos`, `tbl_EntregaveisProjeto`
   - `tbl_LB`, `tbl_Outlook`, `tbl_ControleMed`, `tbl_Desvios`, `tbl_Perfis`
   - `tbl_Oportunidades`
3. Adicionar conector **Office 365 Users** (buscar "Office 365 Users").
4. Adicionar conector **SharePoint** → colar a URL da biblioteca de evidências → selecionar a biblioteca.

**Verificação:** todas as fontes aparecem no painel de dados sem ícone de erro vermelho.

---

### T1.3 — Definir fórmulas nomeadas (tema e constantes)

**Pré-requisitos:** T1.1 concluído.

**Passo a passo:**
1. No Studio: menu **App** (ícone de engrenagem no painel esquerdo) → **Fórmulas**.
2. Colar o bloco abaixo em **App.Formulas**:

```
fxNavy        = ColorValue("#002D72");
fxNavy2       = ColorValue("#003A8C");
fxGreen       = ColorValue("#00A651");
fxYellow      = ColorValue("#FFD100");
fxBlue        = ColorValue("#1E6BB8");
fxOrange      = ColorValue("#E67E22");
fxRed         = ColorValue("#C0392B");
fxBg          = ColorValue("#F0F4FA");
fxSub         = ColorValue("#6B7C9A");
fxBorder      = ColorValue("#DDE4F0");
fxMesRefTexto = Text(varMesRef, "yyyy-mm");
fxFontTitle   = 18;
fxFontBody    = 13;
fxFontSmall   = 11;
fxRadius      = 8;
fxShadow      = "0 2px 8px rgba(0,45,114,0.10)"
```

3. Salvar. Os nomes ficam disponíveis em qualquer fórmula do app.

**Verificação:** digitar `fxNavy` em `Fill` de qualquer controle e ver a cor navy.

---

### T1.4 — Criar componente cmpSidebar

**Pré-requisitos:** T1.3 concluído.

**Passo a passo:**
1. Painel esquerdo → **Componentes → Novo componente** → nome: `cmpSidebar`.
2. Dimensões: Largura=220, Altura=768.
3. Adicionar propriedade de entrada:
   - Nome: `TelaAtiva` | Tipo: Text | Valor padrão: `"Home"`
4. Adicionar propriedade de saída (comportamento):
   - Nome: `OnSelectItem` | Tipo: Behavior
5. Construir o layout interno:

| Controle | Propriedade | Valor |
|---|---|---|
| Rectangle (fundo) | Fill | `fxNavy` |
| Image (logo) | Image | logo TRANSPETRO; Y=20, X=20, W=180 |
| Label (nome usuário) | Text | `cmpSidebar.NomeUsuario` (propriedade de entrada) |
| Gallery (menu) | Items | `["Home","Radar","Painel"]` |

6. Dentro da galeria, para cada item:
   - Rectangle de destaque: `Visible = ThisItem.Value = cmpSidebar.TelaAtiva`, Fill=`fxYellow`, Width=4, X=0.
   - Label: Text=`ThisItem.Value`, Color=White.
   - `OnSelect` da galeria: `cmpSidebar.OnSelectItem()`.

**Verificação:** ao mudar `TelaAtiva` na pré-visualização do componente, o destaque amarelo muda de item.

---

### T1.5 — Criar componente cmpTopbar ⚡

**Pré-requisitos:** T1.3 concluído.

**Passo a passo:**
1. Novo componente → nome: `cmpTopbar`. Largura=1146, Altura=64.
2. Propriedades de entrada:
   - `Titulo` (Text) — ex: `"Home"`
   - `Breadcrumb` (Text) — ex: `"App M&C > Home"`
3. Layout interno:

| Controle | Tipo | Configuração |
|---|---|---|
| Fundo | Rectangle | Fill=White; BorderColor=`fxBorder`; BorderThickness=1 (bottom only via Y/Height trick) |
| lblBreadcrumb | Label | Text=`cmpTopbar.Breadcrumb`; Color=`fxSub`; Size=`fxFontSmall` |
| lblTitulo | Label | Text=`cmpTopbar.Titulo`; Color=`fxNavy`; Bold=true; Size=`fxFontTitle` |
| btnMes | Button group | Três controles: `◀`, Label mês (`Text(varMesRef,"mmm yyyy")`), `▶` |
| btnRegistrar | Button | Text="+ Registrar Medição"; Fill=`fxGreen`; Color=White |
| icoSino | Icon | Icon=Bell; OnSelect=`Set(varMostrarNotif, !varMostrarNotif)` |

4. `OnSelect` do `◀`: `Set(varMesRef, DateAdd(varMesRef, -1, Months))`.
5. `OnSelect` do `▶`: `Set(varMesRef, DateAdd(varMesRef, 1, Months))`.

**Verificação:** clicar nas setas muda o label do mês corretamente.

---

### T1.6 — Criar as 5 telas com shell

**Pré-requisitos:** T1.4, T1.5 concluídos.

**Passo a passo:**
1. Criar tela → nome: `scrHome`. Repetir para: `scrDetalhe`, `scrForm`, `scrRadar`, `scrPainel`.
2. Fundo de todas: `Fill = fxBg`.
3. Em cada tela, inserir:
   - `cmpSidebar`: X=0, Y=0, propriedade `TelaAtiva` = nome da tela (ex: `"Home"`).
   - `cmpTopbar`: X=220, Y=0, W=1146, propriedades `Titulo` e `Breadcrumb` conforme a tela.
4. Área de conteúdo disponível: X=220, Y=64, W=1146, H=704.

**T1.7 — Definir tela inicial:**
- Clicar em **App** no painel → propriedade `StartScreen` = `scrHome`.

**Verificação:** ao pressionar F5, o app abre na scrHome com sidebar e topbar visíveis.

---

## ETAPA 2 — App.OnStart

### T2.1 — Implementar OnStart

**Pré-requisitos:** T1.2 (fontes conectadas), T2.3 (tbl_Perfis criada).

**Passo a passo:**
1. Clicar em **App** → propriedade `OnStart`.
2. Colar o código abaixo:

```
// 1. Identidade
Set(varEmail, Lower(User().Email));
Set(varNome,  User().FullName);
Set(varMesRef, Date(Year(Today()), Month(Today()), 1));

// 2. Coleções paralelas
Concurrent(
    ClearCollect(colItensMed,      tbl_dItensMed),
    ClearCollect(colProjetos,      tbl_dProjetos),
    ClearCollect(colMinhasEntregas,
        Filter(tbl_EntregaveisProjeto, Lower(Gestor) = varEmail)),
    ClearCollect(colMinhasMedicoes,
        Filter(tbl_ControleMed,        Lower(Gestor) = varEmail)),
    ClearCollect(colDesvios,       tbl_Desvios),
    ClearCollect(colPerfis,        tbl_Perfis)
);

// 3. Perfil
Set(varPerfil, Coalesce(LookUp(colPerfis, Lower(Email) = varEmail, Perfil), "Gestor"));

// 4. Listas estáticas
ClearCollect(colMotivosAtraso,
    Table(
        {Value: "— Sem atraso neste período —"},
        {Value: "Parada operacional programada"},
        {Value: "Aguardando aprovação do cliente"},
        {Value: "Pendência de suprimento PETROBRAS"},
        {Value: "Rescisão/substituição contratual"},
        {Value: "Problema técnico / engenharia"},
        {Value: "Outro (detalhar em Plano de Recuperação)"}
    )
);
ClearCollect(colStatus,
    Table(
        {Value: "Medido"},    {Value: "Planejado"},
        {Value: "Postergado"},{Value: "Paralisado"},
        {Value: "Cancelado"}
    )
);
```

3. Para testar: menu **App → Executar OnStart** (ícone ▷ ao lado da barra de fórmulas).

**Medir performance:**
- Abrir **Monitor** (menu Avançado → Monitor).
- Executar OnStart.
- Filtrar por `Network` → verificar que nenhuma resposta retorna 2000 linhas (seria sinal de estouro) e que o total de tempo < 8s.

**Verificação:** `varEmail`, `varPerfil`, `colItensMed` e demais coleções populadas. Inspecionar via **Monitor → App** ou colocando temporariamente um Label com `CountRows(colMinhasEntregas)`.

---

### T2.2 — Verificar campo Gestor no Dataverse

**Passo a passo:**
1. Acesse [make.powerapps.com](https://make.powerapps.com) → **Dataverse → Tabelas → tbl_EntregaveisProjeto**.
2. Abrir coluna `Gestor`: verificar se o tipo é **Pesquisa (lookup)** para a tabela `systemuser` ou se é **Texto simples**.
3. Se for texto simples contendo e-mail: o `Lower()` funciona diretamente.
4. Se for texto contendo nome: criar nova coluna `GestorEmail` (Texto) e popular via fluxo Power Automate que busca o e-mail pelo nome via Office 365 Users.
5. Repetir para `tbl_ControleMed`.

---

### T2.3 — Criar tbl_Perfis no Dataverse

**Passo a passo:**
1. [make.powerapps.com](https://make.powerapps.com) → **Dataverse → Tabelas → Nova tabela**.
2. Nome de exibição: `Perfis App MC` | Nome lógico: `tbl_Perfis`.
3. Adicionar colunas:

| Nome exibição | Nome lógico | Tipo | Configuração |
|---|---|---|---|
| Email | cr_email | Linha de texto | Obrigatório; marcar como **indexado** |
| Perfil | cr_perfil | Escolha | Opções: Gestor, Gerente, CSE |

4. Salvar a tabela.
5. Abrir a tabela → **Dados → Novo registro** → cadastrar usuários do piloto.
6. Voltar ao Power Apps Studio → **Dados → Atualizar** → `tbl_Perfis` aparece disponível.

---

## ETAPA 3 — Tela Home

### T3.1 — OnVisible da scrHome

**Pré-requisitos:** T2.1 concluído; `colMinhasEntregas` e `colMinhasMedicoes` carregadas.

**Passo a passo:**
1. Selecionar `scrHome` → propriedade `OnVisible`.
2. Colar:

```
// Pré-carregar LB e Outlook do gestor como coleções locais
// (evita N LookUps ao Dataverse dentro do AddColumns)
ClearCollect(colLBGestor,
    Filter(tbl_LB,
        GES in Distinct(colMinhasEntregas, GES)
    )
);
ClearCollect(colOutlookGestor,
    Filter(tbl_Outlook,
        GES in Distinct(colMinhasEntregas, GES)
        && Mes_Ref = fxMesRefTexto
    )
);

// Join local: enriquecer cada entrega com LB-2, Outlook do mês e status de medição
ClearCollect(colEntregasMes,
    AddColumns(
        colMinhasEntregas,
        "DataLB2",
            LookUp(colLBGestor,
                GES = ThisRecord.GES
                && ID_iTENSMED = ThisRecord.ID_iTENSMED
                && Nr_SubItem = ThisRecord.Nr_SubItem
                && Tipo_Baseline = "LB-2",
                Data_Med),
        "OutlookAtual",
            LookUp(colOutlookGestor,
                GES = ThisRecord.GES
                && ID_iTENSMED = ThisRecord.ID_iTENSMED
                && Nr_SubItem = ThisRecord.Nr_SubItem,
                Outlook),
        "StatusMes",
            Coalesce(
                LookUp(colMinhasMedicoes,
                    GES = ThisRecord.GES
                    && ID_iTENSMED = ThisRecord.ID_iTENSMED
                    && Nr_SubItem = ThisRecord.Nr_SubItem
                    && Text(Data_Med, "yyyy-mm") = fxMesRefTexto,
                    Status_Med),
                "Planejado")
    )
);

UpdateContext({ctxFiltroStatus: "Todos", ctxBusca: ""})
```

> ⚠️ `Filter` com `in Distinct(...)` **não é delegável**. Como `colMinhasEntregas` é uma coleção local, isso é aceitável — o `Filter` roda localmente sobre os dados já em memória. Nenhum round-trip ao Dataverse dentro do `AddColumns`.

---

### T3.2 — KPI Row (4 cards)

**Passo a passo:**
1. Na área de conteúdo da scrHome (X=220, Y=64), criar um **Container horizontal** (Layout → Horizontal, gap=12) com 4 cards.
2. Cada card é um Rectangle + 2 Labels. Configurar o card "Medido":

| Controle | Propriedade | Valor |
|---|---|---|
| Rectangle fundo | Fill | White; CornerRadius=`fxRadius`; BorderColor=`fxBorder` |
| Rectangle topo | Fill | `fxGreen`; Height=4; Y=0 (borda colorida superior) |
| lblNumero | Text | `Text(CountRows(Filter(colEntregasMes, StatusMes = "Medido")))` |
| lblLabel | Text | `"Medido"` |

3. Repetir para: Planejado (`fxBlue`), Em Risco (`fxOrange`), Paralisado (`fxRed`).
   - "Em Risco" = Postergado + Paralisado:
     `CountRows(Filter(colEntregasMes, StatusMes = "Postergado" || StatusMes = "Paralisado"))`

---

### T3.3 — Banner de alertas

**Passo a passo:**
1. Inserir um Rectangle com `Visible`:
   ```
   CountRows(Filter(colEntregasMes,
       DateDiff(Today(), OutlookAtual, Days) <= 5
       && DateDiff(Today(), OutlookAtual, Days) >= 0
       && StatusMes = "Planejado")) > 0
   ```
2. Fill=`fxYellow`; adicionar Label com texto dinâmico:
   ```
   "⚠ " & Text(CountRows(Filter(colEntregasMes,
       DateDiff(Today(), OutlookAtual, Days) <= 5
       && StatusMes = "Planejado")))
   & " entrega(s) com vencimento nos próximos 5 dias sem medição registrada."
   ```
3. Botão "Ver pendentes": `OnSelect = UpdateContext({ctxFiltroStatus: "Planejado"})`.

---

### T3.4 — Chips de filtro

**Passo a passo:**
1. Criar galeria horizontal (`galleryChips`) com Items:
   ```
   Table(
       {Label: "Todos",     Filtro: "Todos",     Cor: fxNavy},
       {Label: "Planejados",Filtro: "Planejado",  Cor: fxBlue},
       {Label: "Em Risco",  Filtro: "EmRisco",    Cor: fxOrange},
       {Label: "Medidos",   Filtro: "Medido",     Cor: fxGreen}
   )
   ```
2. Cada item: Rectangle (Fill branco quando inativo, cor do item quando ativo) + Label com contagem.
3. `OnSelect`: `UpdateContext({ctxFiltroStatus: ThisItem.Filtro})`.

---

### T3.5 — Galeria principal

**Passo a passo:**
1. Inserir galeria vertical (`galleryEntregas`):
   ```
   // Items:
   Filter(
       Search(colEntregasMes, ctxBusca, "NomeProjeto"),
       ctxFiltroStatus = "Todos"
       || StatusMes = ctxFiltroStatus
       || (ctxFiltroStatus = "EmRisco"
           && (StatusMes = "Postergado" || StatusMes = "Paralisado"))
   )
   ```
2. Colunas do template (usar Text labels posicionados por X):
   - NomeProjeto (X=0, W=220)
   - GES (X=230, W=80)
   - Tipo (X=320, W=80)
   - DataLB2 formatada: `Text(ThisItem.DataLB2, "dd/mm/yyyy")`
   - OutlookAtual formatada
   - Desvio em dias: `If(IsBlank(ThisItem.DataLB2), "—", Text(DateDiff(ThisItem.DataLB2, ThisItem.OutlookAtual, Days)) & "d")`
   - `cmpBadge` (ver T3.6)
   - Botão "Ver →": `OnSelect = Set(varItemSel, ThisItem); Navigate(scrDetalhe)`

---

### T3.6 — Componente cmpBadge

**Passo a passo:**
1. Novo componente → nome: `cmpBadge`. W=90, H=26.
2. Propriedade de entrada: `Status` (Text).
3. Rectangle: CornerRadius=13; Fill:
   ```
   Switch(cmpBadge.Status,
       "Medido",     fxGreen,
       "Planejado",  fxBlue,
       "Postergado", fxOrange,
       "Paralisado", fxRed,
       "Cancelado",  fxSub,
       fxSub)
   ```
4. Label: Text=`cmpBadge.Status`; Color=White; Size=`fxFontSmall`; Align=Center.

---

### T3.7 — Painel direito (Resumo do Mês)

**Passo a passo:**
1. Container no lado direito (X=1066, Y=64, W=300, H=704) com fundo White.
2. **Resumo do Mês**: galeria ou conjunto de barras de progresso por status.
   ```
   // % Medido:
   Round(CountRows(Filter(colEntregasMes, StatusMes="Medido"))
       / CountRows(colEntregasMes) * 100, 0) & "%"
   ```
3. **Financeiro**: somas formatadas:
   ```
   Text(Sum(Filter(colEntregasMes, StatusMes="Medido"), Valor_Atualizado),
       "R$ [$-pt-BR]#.##0,00")
   ```
4. **Ações Rápidas**: botão "+ Registrar Medição" → `Navigate(scrForm)`.

---

## ETAPA 4 — Tela Detalhe

### T4.1 — OnVisible da scrDetalhe

**Pré-requisitos:** `varItemSel` definido via `Set` na Home.

```
Concurrent(
    ClearCollect(colLBItem,
        Filter(tbl_LB,
            GES = varItemSel.GES
            && ID_iTENSMED = varItemSel.ID_iTENSMED
            && Nr_SubItem = varItemSel.Nr_SubItem)),
    ClearCollect(colOutlookItem,
        Filter(tbl_Outlook,
            GES = varItemSel.GES
            && ID_iTENSMED = varItemSel.ID_iTENSMED
            && Nr_SubItem = varItemSel.Nr_SubItem)),
    ClearCollect(colHistMed,
        SortByColumns(
            Filter(tbl_ControleMed,
                GES = varItemSel.GES
                && ID_iTENSMED = varItemSel.ID_iTENSMED
                && Nr_SubItem = varItemSel.Nr_SubItem),
            "Data_Med", SortOrder.Descending)),
    ClearCollect(colDesviosItem,
        Filter(colDesvios,
            GES = varItemSel.GES
            && ID_iTENSMED = varItemSel.ID_iTENSMED))
);
Set(varProjetoSel,
    LookUp(colProjetos, GES = varItemSel.GES))
```

---

### T4.2 — Hero Card navy

**Passo a passo:**
1. Rectangle (X=220, Y=64, W=760, H=120): Fill=`fxNavy`; CornerRadius=`fxRadius`.
2. Labels internos:
   - NomeProjeto: `varItemSel.NomeProjeto` — Bold, Color=White, Size=16
   - GES: `"GES: " & varItemSel.GES` — Color=White, Size=`fxFontSmall`
   - CodigoSAP: `"SAP: " & varProjetoSel.CodigoSAP`
3. `cmpBadge`: Status=`varItemSel.StatusMes`; posição top-right do card.
4. Chips (Rectangle + Label): GG, Regional, Empresa — Fill=`fxNavy2`.

---

### T4.3 — Progress Steps

**Passo a passo:**
1. Definir os 5 passos como tabela local:
   ```
   Table(
       {Passo: 1, Label: "Cadastrado",         Feito: !IsBlank(varProjetoSel)},
       {Passo: 2, Label: "LB-2 planejado",     Feito: !IsEmpty(colLBItem)},
       {Passo: 3, Label: "Outlook confirmado",  Feito: !IsEmpty(colOutlookItem)},
       {Passo: 4, Label: "Evidência enviada",   Feito: !IsBlank(Last(colHistMed).Evidencia)},
       {Passo: 5, Label: "Aprovado",            Feito: Last(colHistMed).Status_Med = "Medido"}
   )
   ```
2. Galeria horizontal: círculo (preenchido verde se `Feito`, cinza se não) + Label + linha conectora entre passos.

---

### T4.4 — Grid de Datas

**Passo a passo:**
1. Tabela HTML ou galeria 2 colunas (Label | Valor):
   ```
   Table(
       {Rotulo: "LB-2",   Valor: Text(LookUp(colLBItem, Tipo_Baseline="LB-2", Data_Med), "dd/mm/yyyy")},
       {Rotulo: "LB-9",   Valor: Text(LookUp(colLBItem, Tipo_Baseline="LB-9", Data_Med), "dd/mm/yyyy")},
       {Rotulo: "Outlook atual", Valor: Text(varItemSel.OutlookAtual, "dd/mm/yyyy")},
       {Rotulo: "Última medição",Valor: Text(First(colHistMed).Data_Med, "dd/mm/yyyy")}
   )
   ```

---

### T4.5 — Timeline de Medições

**Passo a passo:**
1. Galeria vertical sobre `colHistMed`.
2. Template: data formatada + `cmpBadge` com `Status_Med` + Label `Evidencia` (link clicável via `Launch(ThisItem.Evidencia)`).
3. Linha vertical de timeline: Rectangle W=2, Fill=`fxBorder`, conectando os itens.

---

### T4.6 — Painel Financeiro (direita)

```
// Dados de colItensMed (catálogo)
Set(varItemCatalogo, LookUp(colItensMed, ID_iTENSMED = varItemSel.ID_iTENSMED));
```

Labels:
- `"R$ " & Text(varItemCatalogo.Valor_Bruto, "#.##0,00")`
- `"R$ " & Text(varItemCatalogo.Valor_Atualizacao, "#.##0,00")`
- `"R$ " & Text(varItemSel.Valor_Atualizado, "#.##0,00")`

---

## ETAPA 5 — Formulário de Medição

### T5.1 — OnVisible da scrForm

```
UpdateContext({
    ctxStatus:      "Medido",
    ctxDataMed:     Today(),
    ctxNovoOutlook: Blank(),
    ctxMotivo:      First(colMotivosAtraso).Value,
    ctxPlanoRec:    "",
    ctxObs:         "",
    ctxAnexoOk:     false,
    ctxErros:       ""
});
// Recarregar contexto se veio direto (não pelo Detalhe)
If(IsEmpty(colLBItem) || First(colLBItem).GES <> varItemSel.GES,
    Concurrent(
        ClearCollect(colLBItem,    Filter(tbl_LB,         GES=varItemSel.GES && ID_iTENSMED=varItemSel.ID_iTENSMED && Nr_SubItem=varItemSel.Nr_SubItem)),
        ClearCollect(colOutlookItem, Filter(tbl_Outlook,  GES=varItemSel.GES && ID_iTENSMED=varItemSel.ID_iTENSMED && Nr_SubItem=varItemSel.Nr_SubItem)),
        ClearCollect(colHistMed,   SortByColumns(Filter(tbl_ControleMed, GES=varItemSel.GES && ID_iTENSMED=varItemSel.ID_iTENSMED && Nr_SubItem=varItemSel.Nr_SubItem),"Data_Med",SortOrder.Descending))
    )
)
```

---

### T5.2 — Seletor de Status

**Passo a passo:**
1. Galeria horizontal `galleryStatus` sobre `colStatus`.
2. Template: Rectangle (Fill=cor por status quando selecionado, branco quando não) + Label.
3. Seleção: `OnSelect = UpdateContext({ctxStatus: ThisItem.Value})`.
4. Destaque: `Fill = If(ThisItem.Value = ctxStatus, Switch(ThisItem.Value, "Medido", fxGreen, "Postergado", fxOrange, "Paralisado", fxRed, "Cancelado", fxSub, fxBlue), White)`.

---

### T5.3 — Campos condicionais

**Campos sempre visíveis:**
- Label + DatePicker `Data_Medição`: `DefaultDate = ctxDataMed`; `OnChange = UpdateContext({ctxDataMed: Self.SelectedDate})`.

**Visíveis apenas quando Medido:**
- Controle `Attachments` (evidência): `Visible = ctxStatus = "Medido"`.

**Visíveis quando Postergado ou Paralisado:**
- DatePicker `Novo Outlook`: `Visible = ctxStatus in ["Postergado","Paralisado"]`; `OnChange = UpdateContext({ctxNovoOutlook: Self.SelectedDate})`.
- Dropdown Motivo: `Items = colMotivosAtraso`; `Visible = ctxStatus in ["Postergado","Paralisado","Cancelado"]`.
- TextInput Plano de Recuperação: `Visible = ctxStatus in ["Postergado","Paralisado"]`.

---

### T5.4 — Barra de progresso de preenchimento

```
// % de campos obrigatórios preenchidos por status
Set(varCamposTotal,
    Switch(ctxStatus,
        "Medido",    2,   // data + evidência
        "Postergado",3,   // data + novo outlook + motivo
        "Paralisado",3,
        "Cancelado", 1,   // motivo
        1));
Set(varCamposOk,
    (If(ctxStatus = "Medido" && !IsBlank(ctxDataMed), 1, 0))
    + (If(ctxStatus = "Medido" && ctxAnexoOk, 1, 0))
    + (If(ctxStatus in ["Postergado","Paralisado"] && !IsBlank(ctxNovoOutlook), 1, 0))
    + (If(ctxStatus in ["Postergado","Paralisado","Cancelado"] && ctxMotivo <> First(colMotivosAtraso).Value, 1, 0))
);
```

Rectangle barra: `Width = (varCamposOk / varCamposTotal) * 400`; Fill=`If(varCamposOk = varCamposTotal, fxGreen, fxYellow)`.

---

### T5.5 — Criar flow RegistrarMedicao no Power Automate

**Pré-requisitos:** conexão Dataverse no Power Automate.

**Passo a passo:**
1. [make.powerautomate.com](https://make.powerautomate.com) → **Novo fluxo → Fluxo de nuvem instantâneo → Power Apps (V2)**.
2. Nome: `RegistrarMedicao`.
3. Adicionar parâmetros de entrada:
   - `GES` (Text), `ID_iTENSMED` (Text), `Nr_SubItem` (Text)
   - `Status` (Text), `DataMed` (Text), `Gestor` (Text)
   - `UrlEvidencia` (Text), `Motivo` (Text), `Obs` (Text), `PlanoRec` (Text)
4. Ação **Dataverse → Adicionar nova linha** → `tbl_ControleMed`:
   - Mapear cada campo do parâmetro para a coluna correspondente.
5. Condição: `if Status is in [Postergado, Paralisado, Cancelado]`
   - Sim → ação **Dataverse → Adicionar nova linha** → `tbl_Desvios`.
6. Resposta ao Power Apps: `{status: "ok", registro: <linha criada>}` no caminho de sucesso; `{status: "erro", erro: <mensagem>}` no catch.
7. Salvar e testar manualmente com dados de exemplo.
8. Copiar o identificador do flow para usar no app.

---

### T5.6 — Botão Submit no app

**Passo a passo:**
1. Adicionar conexão ao flow `RegistrarMedicao` em **Dados → Adicionar dados → Power Automate**.
2. Botão "Enviar para Aprovação": `DisplayMode = If(varCamposOk = varCamposTotal, DisplayMode.Edit, DisplayMode.Disabled)`.
3. `OnSelect`:

```
If(varCamposOk < varCamposTotal,
    UpdateContext({ctxErros: "Preencha todos os campos obrigatórios."}),
    // Chamar o flow
    Set(varResultado,
        RegistrarMedicao.Run(
            varItemSel.GES,
            Text(varItemSel.ID_iTENSMED),
            Text(varItemSel.Nr_SubItem),
            ctxStatus,
            Text(ctxDataMed, "yyyy-mm-dd"),
            varEmail,
            varUrlEvidencia,
            ctxMotivo,
            ctxObs,
            ctxPlanoRec
        )
    );
    If(varResultado.status = "ok",
        // Atualizar coleção local sem recarregar tudo
        Collect(colMinhasMedicoes, varResultado.registro);
        Notify("Medição registrada com sucesso ✓", NotificationType.Success);
        Navigate(scrHome),
        Notify("Erro: " & varResultado.erro, NotificationType.Error)
    )
)
```

---

### T5.7 — Upload de evidência (flow SalvarEvidencia)

**Passo a passo (Power Automate):**
1. Novo fluxo instantâneo → parâmetros: `NomeArquivo` (Text), `ConteudoBase64` (Text), `GES` (Text), `Periodo` (Text).
2. Ação **SharePoint → Criar arquivo**: Site=URL da biblioteca; Caminho=`/Evidencias/` & `GES` & `/` & `Periodo` & `/`; Nome=`NomeArquivo`; Conteúdo=`base64ToBinary(ConteudoBase64)`.
3. Ação **SharePoint → Obter propriedades do arquivo** → retornar `{url: <link do arquivo>}`.

**No app (controle Attachments):**
```
// OnAddFile do controle Attachments:
Set(varUrlEvidencia,
    SalvarEvidencia.Run(
        First(Self.Attachments).Name,
        First(Self.Attachments).Value,
        varItemSel.GES,
        fxMesRefTexto
    ).url
);
UpdateContext({ctxAnexoOk: !IsBlank(varUrlEvidencia)})
```

---

## ETAPA 6 — Radar de Oportunidades

### T6.1 — Criar tbl_Oportunidades

**Passo a passo:**
1. Dataverse → **Nova tabela** → Nome: `Oportunidades App MC` | Nome lógico: `tbl_Oportunidades`.
2. Colunas:

| Nome | Tipo | Obs |
|---|---|---|
| GES | Texto | Indexado |
| NomeProjeto | Texto | Obrigatório |
| Fase | Escolha | Avaliação / Conceitual / Básico |
| GG | Texto | |
| Regional | Texto | |
| Gestor | Texto | E-mail do responsável |
| InicioPrevisto | Data e hora | |
| PrimeiraMedicaoEst | Data e hora | |
| ValorEstimado | Número decimal | |

---

### T6.2 — OnVisible e layout da scrRadar

```
// OnVisible:
ClearCollect(colOportunidades, tbl_Oportunidades);
UpdateContext({ctxFiltroFase: "Todas", ctxMeuGG: false})
```

**KPI Cards:**
```
CountRows(Filter(colOportunidades, Fase = "Avaliação"))
CountRows(Filter(colOportunidades, Fase = "Conceitual"))
CountRows(Filter(colOportunidades, Fase = "Básico"))
Text(Sum(colOportunidades, ValorEstimado), "R$ #.##0,00")
```

**Galeria com filtros:**
```
Filter(colOportunidades,
    (ctxFiltroFase = "Todas" || Fase = ctxFiltroFase)
    && (!ctxMeuGG || GG = varGGUsuario)
)
```

---

### T6.3 — Form de Nova Oportunidade (painel/modal)

**Passo a passo:**
1. Rectangle semitransparente de fundo (overlay): X=220, Y=0, W=1146, H=768; Fill=`RGBA(0,0,8,0.5)`; `Visible = varMostrarFormOport`.
2. Container branco centralizado: W=500, H=600; CornerRadius=`fxRadius`.
3. Campos: TextInput para NomeProjeto, Dropdown para Fase, DatePicker para InicioPrevisto, etc.
4. Botão Salvar:
   ```
   Patch(tbl_Oportunidades, Defaults(tbl_Oportunidades), {
       NomeProjeto: txtNomeProjOport.Text,
       Fase: dropFase.Selected.Value,
       GES: txtGESOport.Text,
       GG: txtGGOport.Text,
       Gestor: varEmail,
       InicioPrevisto: dpInicio.SelectedDate,
       ValorEstimado: Value(txtValorOport.Text)
   });
   Collect(colOportunidades, Last(tbl_Oportunidades));
   UpdateContext({varMostrarFormOport: false})
   ```

---

## ETAPA 7 — Painel Gerencial (Power BI)

### T7.1 — Guarda de perfil e layout simples

**OnVisible da scrPainel:**
```
If(varPerfil = "Gestor", Navigate(scrHome))
```

**Layout:**
1. Label título: "Painel Gerencial M&C".
2. Label subtítulo: "A visualização consolidada está disponível no Power BI. Clique abaixo para abrir."
3. Botão "Abrir Painel BI":
   ```
   OnSelect = Launch("<COLAR_URL_DO_RELATORIO_BI>", {}, LaunchTarget.New)
   ```
   Fill=`fxNavy`; Color=White; W=280; H=48.
4. Label informativo: `"Dados atualizados em: " & Text(varUltimaAtualiz, "dd/mm/yyyy hh:mm")`.

---

## ETAPA 8 — Qualidade e publicação

### T8.1 — Checklist Power Apps Monitor

**Passo a passo:**
1. Abrir **Monitor**: menu Avançado → Monitorar → Abrir Monitor.
2. Com o monitor aberto, executar:
   - `App → Executar OnStart`: medir duração total; verificar que nenhuma resposta de rede tem `rowCount = 2000`.
   - Navegar para `scrHome`: verificar OnVisible < 2s.
   - Navegar para `scrDetalhe`: idem.
3. Filtrar por `Delegationwarning` — deve estar vazio (ou só sobre coleções locais).

---

### T8.2 — Tratamento de erros global

**Passo a passo:**
1. **App → Configurações → Próximas funcionalidades → Ativar "Gerenciamento de erro no nível de fórmula"**.
2. Envolver todos os `Patch` e `ClearCollect` de Dataverse em `IfError`:
   ```
   IfError(
       Patch(tbl_ControleMed, ...),
       Notify("Falha ao salvar. Tente novamente. " & FirstError.Message,
              NotificationType.Error)
   )
   ```

---

### T8.3 — Acessibilidade

**Passo a passo:**
1. Para cada botão/ícone sem texto visível, definir `AccessibleLabel`:
   - Ícone sino: `"Notificações"`.
   - Botões ◀▶ do mês: `"Mês anterior"` / `"Próximo mês"`.
2. Verificar contraste: badges coloridos com texto branco — usar [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/) para `fxGreen` / `fxRed` / `fxOrange`.
3. Tabindex: garantir ordem lógica de foco via propriedade `TabIndex`.

---

### T8.4 — Botão Refresh Manual

**Passo a passo:**
1. Adicionar `icoRefresh` (ícone de atualização) na topbar ou em cada tela.
2. `OnSelect`:
   ```
   Concurrent(
       ClearCollect(colMinhasEntregas,
           Filter(tbl_EntregaveisProjeto, Lower(Gestor) = varEmail)),
       ClearCollect(colMinhasMedicoes,
           Filter(tbl_ControleMed, Lower(Gestor) = varEmail))
   );
   // Re-executar OnVisible da tela atual
   Select(btnTriggerOnVisible)   // botão invisível com o código do OnVisible
   ```
   > Alternativa: criar um botão transparente `btnTriggerOnVisible` com o mesmo código do `OnVisible` e chamá-lo com `Select()`.

---

### T8.5 — Publicar na solution DEV

**Passo a passo:**
1. Studio → **Arquivo → Salvar → Publicar**.
2. [make.powerapps.com](https://make.powerapps.com) → **Soluções → AppMC_Solution → Exportar** (gerenciada para homologação).
3. Registrar no documento de versão:
   - Versão do app (ex: 1.0.0)
   - Data de publicação
   - Lista de etapas concluídas
   - Pendências conhecidas

---

## Resumo de dependências externas

| O quê | Quem faz | Antes de |
|---|---|---|
| Campo `Gestor` (Person/Group ou e-mail) em tbl_EntregaveisProjeto e tbl_ControleMed | Admin Dataverse | T2.1 |
| `tbl_Perfis` criada e populada | Admin Dataverse | T2.1 |
| `tbl_Oportunidades` criada | Admin Dataverse | T6.1 |
| Biblioteca SharePoint de evidências | Admin SharePoint | T5.4 |
| Flow `SalvarEvidencia` publicado | Dev Power Automate | T5.7 |
| Flow `RegistrarMedicao` publicado | Dev Power Automate | T5.5 |
| URL do relatório Power BI | Analista BI | T7.1 |
