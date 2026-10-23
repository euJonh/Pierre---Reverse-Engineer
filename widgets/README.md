# Widgets do Pierre: especificação para réplica

O objetivo é documentar cada widget da home "Vision" e da home do cartão com detalhe suficiente para replicá-los de forma idêntica em outro projeto React Native: cores, tipografia, layout em px, lógica de dados, logos, animações e textos. A análise é estática, feita sobre o bundle Hermes decompilado. Nos documentos, **[O]** marca o que foi observado no código (com a linha `Lnnnn` do `out.js`) e **[I]** o que foi inferido.

## Status (atualizado em 23/10/2026)

**Próximo passo:** consolidar tudo num documento único e levar os widgets ao protótipo de `pierre-design-system.html`. Todos os grupos de widgets já estão especificados.

| # | Arquivo | Widgets | Status |
|---|---|---|---|
| 0 | `00-base.md` | Renderer, grid, `Widget2x2`, `WidgetWide`, `WidgetFull`, estados vazio/erro/carregando, "Ocultar dados", moeda, tokens dark, config NativeWind | ✅ Completo |
| 1 | `01-carteira-saldo-fluxo.md` | `wallet`, `account-balance`, `cash-flow`, BankLogo | ✅ Completo |
| 2 | `02-gastos.md` | `monthly-spending` (gráficos nativos Kotlin `PierreCharts`: área, barras, bloquinhos, comparação), `spending-by-category` (barra segmentada, toggle Categorias/Grupos) | ✅ Completo |
| 3 | `03-limite-parcelas-cartoes-transacoes.md` | `available-limit`, `installments`, `cards`, `patrimony`, `recent-transactions`, `VisionTransactionRow` e a lógica de logo (`MerchantAvatar` + selo do banco) | ✅ Completo |
| 4 | `04-assinaturas.md` | `subscriptions`: widget, tela de detalhe (a do screenshot), calendário Mensal/Diária, tela do dia, adicionar assinatura | ✅ Completo |
| 5a | `05a-cdb.md` | `cdb-investment` (wide, gráfico gifted-charts), `cdb-performance` (half/wide, retirado do picker), `cdb-position` (half/full, `PierreAreaChart`) | ✅ Completo |
| 5b | `05b-leo-ia.md` | `leo-agent`, `leo-insight`, widgets de IA (`ai-<uuid>`): `AIVisionWidget`, criação pelo `WidgetChat`, `AIWidgetRenderer` (previews e home do cartão) | ✅ Completo |
| 6a | `06a-home-cartao.md` | Home do cartão: `WIDGET_REGISTRY`, tela (`CardsScene`/`CardHomeScreen`), grade, edição e persistência, `WidgetCarousel` (do chat), dados ao vivo | ✅ Completo |
| 6b | `06b-widgets-cartao.md` | Widgets da home do cartão: `cashback`, `invoice`, `CardWidget` (arte, máscara, ações), gastos por categoria e diferenças dos demais widgets de sistema | ✅ Completo |

## Principais achados até aqui
- **Grid:** largura útil = tela − 32 e gap 18. Half = (útil − 18)/2 × 200 px, wide = útil × 170 px, full = útil × altura livre. Raio 24.
- **Visual dos widgets:** no Android o fundo é `bg-surface` (#18181b), sem borda e sem sombra; o glass existe só no iOS. O toque não muda escala, só dispara háptico Light.
- **Valores em reais:**
  - "Ocultar dados" usa o store `pierre-privacy` e a máscara `R$ ••••`, com fade de 150 ms.
  - Os valores dos widgets não têm contagem animada (`durationMs: 0`).
  - A forma compacta só usa o sufixo "mi".
- **Dados:** vêm de `GET /api/mobile/vision/widgets/{id}/data`, com queryKey `['vision-widget-item-data', id, size, chartVariant]` e staleTime de 2 min.
- **Gastos do mês:** a variante de gráfico escolhida só vale para half e wide. O full sempre usa `MonthlyComparisonChart` (área do mês atual + mês anterior).
- **Gráficos:** os de `monthly-spending` são nativos em Kotlin (módulo Expo `PierreCharts`, Canvas); para replicar, refaça em react-native-svg.
  - Área: gasto acumulado por dia, curva Catmull-Rom, linha `#B8FF12` de 1,8 dp e degradê vertical até transparente.
  - Comparação (full): linha do mês anterior tracejada 7/7, `#5C5C68`. O percentual compara com o mesmo dia do mês anterior.
  - Barras (só half): pílulas com degradê `#F0F30D` → `#95FF48`.
  - Bloquinhos: Views RN, 12 blocos de 18 px (half) ou 39 de 15 px (wide), gap 4 e raio 4; a opacidade vai de 0,32 a 1 conforme a quantidade de compras do dia.
  - Na home nenhum gráfico anima nem vibra.
- **Gastos por categoria:** barra horizontal segmentada com até 3 categorias, sem "Outros"; toggle Categorias/Grupos.
- **Transações:**
  - Linha `VisionTransactionRow` com avatar de 45 px; o valor fica `#95FF48` em crédito e `#FF5C7A` em débito.
  - Escolha da logo (`MerchantAvatar`): logo local por palavra-chave → URL `/api/logo?domain=` → ícone da categoria → inicial.
  - Selo do banco: caixa de 30 px em −5/−5 com logo de 22 px.
  - Para casar as palavras-chave do `logos/mapeamento.json`, teste da mais longa para a mais curta.
- **CDB:** visível via flag `cdbInvestmentsEnabled` de `/api/mobile/config` (`internalOnly` → só e-mails `@cloudwalk.io`). Só `cdb-investment` usa o endpoint da Vision (servidor manda `headline`, `subtitle`, `cohort`, `series`); `cdb-performance` e `cdb-position` leem `/api/cdb/*` e calculam no app, só com KYC aprovado. Nenhum usa `Widget2x2`/`WidgetWide`/`WidgetFull`.
- **Leo:** `leo-agent` (half) é estático e abre o chat; `leo-insight` (wide, altura pelo conteúdo, mín. 160) mostra o último relatório do agente via `/api/agents/{id}/logs`.
- **Home do cartão:**
  - Não tem rota própria: é a cena `CardsScene` do hub `/vision` (`?section=cards`). Grade `Sortable.Flex` com gap 12 (a Vision usa 18); o cartão fica fora da grade.
  - Há dois `WIDGET_REGISTRY` sem relação (L775589 = home do cartão, 11 ids; L2202932 = widgets `show*` do chat) e um terceiro catálogo, `WIDGET_SURFACE_CATALOG`.
  - Persistência: `pierre-widget-order:<userId>`, `pierre:cardHomeLayout:<userId>` e `GET/PATCH /api/mobile/home-layout` (PATCH ao sair da edição ou após 500 ms).
  - Dados ao vivo direto da API de crédito (`/credit/...`, `staleTime 0`) e substituem o dado do layout. Só `cashback` e `invoice` têm componente próprio; os outros sete reaproveitam os da Vision.
  - `WidgetCarousel` é do chat, não da home do cartão.
  - Cartão: número completo e CVV não são persistidos e somem da memória em 60 s; a máscara mostra só "•• " + 4 últimos dígitos.
- **Widgets de IA:** na Vision, `AIVisionWidget` só desenha `items`/`accounts` como até 3 barras, com dados de `GET /api/mobile/vision/widgets/{id}/ai-data` (refresh 3 h, cache antes da rede); sem estado de erro e sem navegação. Criação no `WidgetChat` (`/api/mobile/vision/widgets/chat`, tools `suggest_widgets`, `show_chart_options`, `generate_preview`) e `POST /api/mobile/vision/widgets`.

## Material de trabalho (fora do git)
Fica em `_trabalho/`, ignorada pelo `.gitignore`:
- `_trabalho/out.js`: bundle decompilado (145 MB). Para recriar, veja "Como o APK foi analisado" no `CLAUDE.md`.
- `_trabalho/strings.json`: tabela de strings do bundle.
- `_trabalho/parcial/`: trechos já extraídos por grupo. Destaques:
  - `w2/`: código dos gráficos e smali do módulo nativo de gráficos;
  - `catmap.txt`: categoria → cor/ícone, em cp1252;
  - `w3.js`: widgets do grupo 3;
  - `w4_screen.js`, `w4_cal.js` e `w4_widget.js`: assinaturas;
  - `w5/` e `w5_*.txt`: CDB, Leo e IA (entregues como `_trabalho/w5a-cdb.md` e `w5b-leo-ia.md`);
  - `w6a-home-cartao.md` e `w6b-widgets-cartao.md`: entregas do grupo 6;
  - `w6/` e `w6b/`: home do cartão, com índice de módulos `mods.json`;
  - `w0tools/venv`: hermes-dec, para desmontar funções.

## Como retomar
1. Confirme que `_trabalho/out.js` existe. Se não existir, refaça a decompilação.
2. Rode um grupo por vez, com um agente seguindo `BRIEF.md`. Ajuste os caminhos do BRIEF se `_trabalho/` mudar. Passe ao agente:
   - `00-base.md` e a §1 de `01-carteira-saldo-fluxo.md` como referência, para não repetir conteúdo;
   - os intermediários do grupo em `_trabalho/parcial/`;
   - a instrução de gravar cada seção assim que terminá-la (as sessões caíram várias vezes).
3. Ordem sugerida:
   - nada pendente nos grupos de widgets.
   - Atenção: o `BRIEF.md` cita a raiz `C:/REA/Pierre/`; a raiz atual é `C:/REA/Pierre---Reverse-Engineer/`.
4. Pontos de partida (linhas do `out.js`):
   - **Grupo 2:** `VisionMonthlySpendingWidget` L2843453–2850970, `SpendingByCategoryWidgetConnected` L2862447, `WidgetModeToggle` L2863541, `VisionSpendingByCategoryWidget` L2864048.
   - **Grupo 3:**
     - Limite: `AvailableLimitWidget` L2853333.
     - Parcelamentos: `InstallmentsWidget` L2855251 e `InstallmentsWidgetFull` L2856127.
     - Patrimônio: `PatrimonyWidget` L2858204.
     - Cartões: `CardsWidget` L2860269.
     - Transações: `VisionRecentTransactionsWidget` L2861750, mais `VisionTransactionRow`.
   - **Grupo 4:**
     - Widget: `SubscriptionsWidget` L2866635 e `SubscriptionsWidgetFull` L2867690.
     - Calendário: `VisionSubscriptionsCalendarView` L3249962 e `VisionSubscriptionsCalendarDayContent` L3248955.
     - Telas: `VisionSubscriptionsScreen` e `VisionAddSubscriptionScreen`.
   - **Grupo 5:**
     - CDB: `CdbInvestmentWidget` L2869001, `CdbPerformanceWidget` L2870739, `CdbPositionWidget` L2875036.
     - Leo: `LeoAgentWidget` L2876665 e `LeoInsightWidget` L2877353.
     - IA: `AIVisionWidget` L2835635, `LiveWidgetRenderer` L2982377 e `AIWidgetRenderer` L2988473.
   - **Grupo 6:**
     - Registro: `WIDGET_REGISTRY` em ~L775589 e ~L2202932.
     - Widgets: `CashbackWidget` L2985666, `InvoiceWidget` L2985949, `CardWidget` L3008528 e `WidgetCarousel` L2398930.
5. No fim, consolide tudo num documento único e, se for útil, adicione os widgets ao protótipo de `pierre-design-system.html`.

## Dúvidas abertas já registradas
- **Grupo 0:**
  - o raio 24 do `WidgetWide`;
  - se o fallback do error boundary é o `WidgetErrorState`;
  - o backend de storage do store de privacidade;
  - a fonte do número animado nativo (provavelmente Roboto).
- **Grupo 1:**
  - `WALLET_STACK_TRAVEL` (0,9?);
  - `SKINS.PIERRE_ONE_DEFAULT` e `mergePierreCardIntoWalletWidget`, não lidos;
  - a animação nativa do `PierreDonutChart`;
  - "Conectar conta" sem `onPress`.
- **Grupo 5a (CDB):**
  - como `onWidgetTap` combina `{id, from}` com o `navigateTo`;
  - se o `cdb-investment` no Android usa o destino dinâmico (`amount`/`offer`/`entry`) ou só a rota fixa;
  - textos reais do servidor e outros valores de `cohort` além de `simulated`;
  - altura 132 e padding 24 do gráfico do `cdb-investment` (inferidos).
- **Grupo 5b (Leo/IA):**
  - nenhum ponto do app navega para `vision/widget-chat`;
  - prompts das tools e formato do `config` ficam no servidor;
  - widget de IA criado como cripto ou área vira lista de barras na Vision (não validado em aparelho);
  - cor da bolha usuário × IA no chat; sentido do filtro de cripto nas sugestões;
  - botão "Conversar" do Leo provavelmente invisível no Android (mesmo fundo do card).
- **Grupo 6a (estrutura do cartão):**
  - `last-purchase` está no registro, mas não é renderizado na grade;
  - qual dos três catálogos vale na tela de adicionar widget;
  - algoritmo de `hasGaps`/`fillGaps` e gatilho de "reorganizar com IA";
  - nomes das constantes 200/20/240/52/12/120 do `JiggleWrapper`;
  - se o refetch ao voltar ao app está ligado (`focusManager`).
- **Grupo 6b (widgets do cartão):**
  - `rounded-3xl`: a config compilada dá 21 px, mas os documentos 02 e 05b dizem 24 (checar);
  - formato real das strings da API de crédito não observado;
  - sem `data` no slot, gastos por categoria pode quebrar e cashback pode mostrar "undefined pendentes";
  - `onExpandRequest` (supus 0) e momento da troca Congelar/Descongelar (supus 0,43);
  - ordenação decrescente das categorias é inferida;
  - em Parcelamentos o percentual pago é fixo em 25%; em Transações recentes os itens não trazem `type` (valor sem sinal/cor) [I].
