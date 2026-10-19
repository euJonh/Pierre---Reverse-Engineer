# Widgets do Pierre: especificação para réplica

O objetivo é documentar cada widget da home "Vision" e da home do cartão com detalhe suficiente para replicá-los de forma idêntica em outro projeto React Native: cores, tipografia, layout em px, lógica de dados, logos, animações e textos. A análise é estática, feita sobre o bundle Hermes decompilado. Nos documentos, **[O]** marca o que foi observado no código (com a linha `Lnnnn` do `out.js`) e **[I]** o que foi inferido.

## Status (pausado em 19/10/2026)

| # | Arquivo | Widgets | Status |
|---|---|---|---|
| 0 | `00-base.md` | Renderer, grid, `Widget2x2`, `WidgetWide`, `WidgetFull`, estados vazio/erro/carregando, "Ocultar dados", moeda, tokens dark, config NativeWind | ✅ Completo |
| 1 | `01-carteira-saldo-fluxo.md` | `wallet`, `account-balance`, `cash-flow`, BankLogo | ✅ Completo |
| 2 | `02-gastos-PARCIAL.md` | `monthly-spending`, `spending-by-category` | ⚠️ Parcial: faltam o desenho dos 3 gráficos e todo o `spending-by-category` |
| 3 | — | `available-limit`, `installments`, `cards`, `patrimony`, `recent-transactions` (inclui a lógica de logo das transações) | ⏳ Não iniciado |
| 4 | — | `subscriptions`: widget e tela de detalhe (a do screenshot), calendário, adicionar assinatura | ⏳ Não iniciado |
| 5 | — | `cdb-investment`, `cdb-performance`, `cdb-position`, `leo-agent`, `leo-insight`, widgets de IA (`ai-<uuid>`) | ⏳ Não iniciado |
| 6 | — | Home do cartão (`WIDGET_REGISTRY`): Cashback, Fatura, Cartão, Gastos por categoria, carrossel | ⏳ Não iniciado |

## Principais achados até aqui
- **Grid:** largura útil = tela − 32 e gap 18. Half = (útil − 18)/2 × 200 px, wide = útil × 170 px, full = útil × altura livre. Raio 24.
- **Visual dos widgets:** no Android o fundo é `bg-surface` (#18181b), sem borda e sem sombra; o glass existe só no iOS. O toque não muda escala, só dispara háptico Light.
- **Valores em reais:**
  - "Ocultar dados" usa o store `pierre-privacy` e a máscara `R$ ••••`, com fade de 150 ms.
  - Os valores dos widgets não têm contagem animada (`durationMs: 0`).
  - A forma compacta só usa o sufixo "mi".
- **Dados:** vêm de `GET /api/mobile/vision/widgets/{id}/data`, com queryKey `['vision-widget-item-data', id, size, chartVariant]` e staleTime de 2 min.
- **Gastos do mês:** a variante de gráfico escolhida só vale para half e wide. O full sempre usa `MonthlyComparisonChart` (área do mês atual + mês anterior).

## Material de trabalho (fora do git)
Fica em `_trabalho/`, ignorada pelo `.gitignore`:
- `_trabalho/out.js`: bundle decompilado (145 MB). Para recriar, veja "Como o APK foi analisado" no `CLAUDE.md`.
- `_trabalho/strings.json`: tabela de strings do bundle.
- `_trabalho/parcial/`: trechos já extraídos por grupo. Destaques:
  - `w2/`: código dos gráficos e smali do módulo nativo de gráficos;
  - `catmap.txt`: categoria → cor/ícone, em cp1252;
  - `w3.js`: widgets do grupo 3;
  - `w4_screen.js`, `w4_cal.js` e `w4_widget.js`: assinaturas;
  - `w5/` e `w5_*.txt`: CDB, Leo e IA;
  - `w6/` e `w6b/`: home do cartão, com índice de módulos `mods.json`;
  - `w0tools/venv`: hermes-dec, para desmontar funções.

## Como retomar
1. Confirme que `_trabalho/out.js` existe. Se não existir, refaça a decompilação.
2. Rode um grupo por vez, com um agente seguindo `BRIEF.md`. Ajuste os caminhos do BRIEF se `_trabalho/` mudar. Passe ao agente:
   - `00-base.md` e a §1 de `01-carteira-saldo-fluxo.md` como referência, para não repetir conteúdo;
   - os intermediários do grupo em `_trabalho/parcial/`;
   - a instrução de gravar cada seção assim que terminá-la (as sessões caíram várias vezes).
3. Ordem sugerida:
   - terminar o grupo 2, que já tem 11 KB e cuja próxima etapa é `MonthlyChart` (L2842298) e os gráficos;
   - depois os grupos 3, 4, 5 e 6.
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
