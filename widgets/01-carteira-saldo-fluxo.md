# W1 — Widgets `wallet`, `account-balance` e `cash-flow` (Pierre v4.0.6)

> Análise estática de `_trabalho/out.js`. **[O]** = observado (`Lnnnn` = linha de out.js; `Mnnnn` = módulo Metro); **[I]** = inferido.
> Fundação (shell, Widget2x2/WidgetWide/WidgetFull, tokens dark, NativeWind rem=14, PrivacyText/PrivacyCurrencyText, formatação BRL, "Ocultar dados") está em **`w0-base.md`** e não é repetida aqui.
> Intermediários desta análise: `_trabalho/parcial/w1x/` (módulos extraídos `m<ID>.js`, índice `modidx.tsv`, script `ext.sh`).

## 0. Mapa de módulos [O]

| Módulo | Linhas | Conteúdo |
|---|---|---|
| M8989 | L2839162–2839409 | `AccountBalanceWidgetConnected` (deps 8976, 8990, 8993) |
| M8990 | L2839410–2840957 | `AccountBalanceWidget`, `BankLogosStack`, `AccountBalanceEmptyIcon`, `AccountBalanceFullEmptyContent` |
| M9002 | L2850958–2851195 | `CashFlowWidgetConnected` (deps 8976, 9003, 9004) |
| M9003 | L2851196–2852771 | `CashFlowWidget`, `MiniBar`, `WideRow` |
| M9000 | L2847697–2850898 | utilitários Pierre Card (`mergePierreCardIntoWalletWidget`, `formatReais`, `centsToReais`…) |
| M9001 | L2850899–2850957 | `formatCentsToBRL`, `formatSignedCents` |
| M9073 | L2878411–2879511 | `WalletWidget` (+ worklet `WalletWidgetTsx1`), `resolveWalletCardHeight` |
| M9074 | L2879512–2880025 | constantes `WALLET_*` |
| M9075 | L2880026–2880229 | barrel: `WalletHeaderMask, WalletLimitRing, WalletOpenFinancePane, WalletPagerDots, WalletPagerPage, WalletPagerScrollView, WalletPierreCardPane, WalletPierreHeader, WalletPierreOfferPane, WalletStatusChip, WalletVerticalDots, WalletVerticalStack, USES_GESTURE_HANDLER_PAGER` (M9076–M9087) |
| M9088 | L2886449–2886900 | `WalletWidgetConnected`, `WALLET_OPEN_FINANCE_INVOICES_ROUTE`, `WALLET_BANK_CONNECT_ROUTE` |
| M2991 | L925163–926485 | hápticos customizados (`playCustomHaptic`, `selection`…) usados pela carteira |
| M8526 | L2717532–2719412 | `useVisionSystemWidgetData` (busca comum dos widgets) |
| M9089–M9091 | L2886901–2888376 | `buildWalletPanes`, `useWalletData`, `useWalletPierreCard` |
| M3633/M3634/M3697 | L1330710, L1330882, L1342269 | `BankLogo`, `resolveBankColor`, `BANK_VISUAL_CATALOG` |

---

## 1. Busca de dados comum (M8526, L2717532) [O]

Os três widgets usam `useVisionSystemWidgetData(widgetId, size, chartVariant?)` (M8526, L2718148):
- `useAccountBalanceData(size)` (M8993, L2841787) = `useVisionSystemWidgetData('account-balance', size)` → devolve `{data, isLoading, isError, isFetched}`.
- `useCashFlowData(size)` (M9004, L2852772) = `useVisionSystemWidgetData('cash-flow', size)`.
- A carteira: ver §2 (`WalletWidgetConnected`).

| Item | Valor |
|---|---|
| Endpoint | `GET /api/mobile/vision/widgets/{widgetId}/data?size={size}&chartVariant={cv}&fresh=1` (cada parâmetro só entra se truthy; `fresh=1` só se `shouldRequestFreshVisionWidgetData()`), via `apiClient.get` [O L2717632–2717690] |
| Resposta usada | `{ data, aiContext }` |
| queryKey | `['vision-widget-item-data', widgetId, size ?? 'default', chartVariant ?? 'default']` (fábrica `item`); há também `['vision-widget-ai-context', …]` |
| Opções | `staleTime: 120000` (2 min), `gcTime: 600000` (10 min), `networkMode: 'always'`, `enabled: Boolean(userId) && …`, `retry: false` em um dos modos |
| Cache local | `getCachedWidgetItemData` / `saveWidgetItemDataCache` (`{data, updatedAt}`), chave `userId:widgetId:size…` (`'anonymous'`/`'default'` como fallback) — pinta dado salvo antes da rede [O nomes; detalhe do storage I] |
| Widgets atendidos | `wallet, account-balance, monthly-spending, spending-by-category, recent-transactions, cash-flow, subscriptions, cards, available-limit, installments, patrimony, cdb-investment, cdb-position, cdb-performance` |

---

## 2. `wallet` — "Carteira"

### 2.1 Identidade [O]
- id `wallet`, label `Carteira`, tamanhos `full | half` (default do componente: `full`), `tapAction: none`, `insertPosition: top`.
- `WalletWidgetConnected` (M9088, L2886483) → `WalletWidget` (memo, M9073, L2878485). O renderer **não** passa `onPress` (w0 §2.3): cada painel tem seus próprios toques.
- Módulos de apoio: constantes M9074; UI M9076–M9087 (barrel M9075); `buildWalletPanes` M9089; `useWalletData` M9090; `useWalletPierreCard` M9091; matemática do pager M9080.
- O widget é um **carrossel de "painéis"** (cartões): painel Pierre One (fatura do cartão próprio ou oferta para pedir o cartão) e painel Open Finance (pilha de cartões de outros bancos com total em faturas). No *full* os painéis passam na horizontal (pager com pontos); no *half* empilham na vertical (stack com gesto).

### 2.2 Constantes `WALLET_*` (M9074, L2879512) [O]
| Constante | Valor | Constante | Valor |
|---|---|---|---|
| `WALLET_CARD_HEIGHT` | **228** | `WALLET_HALF_CARD_HEIGHT` | **188** |
| `WALLET_CARD_RADIUS` | 24 | `WALLET_HEADER_INSET` | 8 |
| `WALLET_HEADER_HEIGHT` | 67 | `WALLET_HEADER_RADIUS` | 16 |
| `WALLET_HEADER_BUMP_WIDTH` | 54 | `WALLET_HEADER_BUMP_HEIGHT` | 52 |
| `WALLET_HEADER_BUMP_TOP` | 25 | `WALLET_HEADER_TOTAL_HEIGHT` | 77 |
| `WALLET_STACK_OFFSET` | 22 | `WALLET_MAX_STACKED_CARDS` | 3 |
| `WALLET_CARD_ART_ASPECT_RATIO` | 1.586 (ISO ID-1) | `WALLET_CONTENT_INSET` | 16 |
| `WALLET_LIMIT_USED_HIGH_MAX` | 30 | `WALLET_LIMIT_USED_MEDIUM_MAX` | 85 |
| `WALLET_LIMIT_RING_SIZE` | 45 | `WALLET_LIMIT_RING_STROKE` | 4 |
| `WALLET_DOT_SIZE` | 8 | `WALLET_DOT_GAP` | 4 |
| `WALLET_DOT_ACTIVE_WIDTH` | 20 | `WALLET_DOTS_MARGIN_TOP` | 20 |
| `WALLET_PAGE_MIN_SCALE` | 0.9 | `WALLET_DOTS_BOTTOM` | 8 |
| `WALLET_PANE_BOTTOM_WITH_DOTS` | 36 | `WALLET_PANE_BOTTOM_WITHOUT_DOTS` | 16 |
| `WALLET_HALF_DOT_SIZE` | 4 | `WALLET_HALF_DOT_GAP` | 4 |
| `WALLET_HALF_DOTS_RIGHT` | 6 | `WALLET_STACK_DOTS_SHOW_MS` | 150 |
| `WALLET_STACK_DOTS_HIDE_MS` | 250 | `WALLET_STACK_DOTS_HIDE_DELAY_MS` | 600 |
| `WALLET_STACK_TRAVEL` | 0.9 [O; ordem de registradores, confira] | `WALLET_STACK_SCALE_MID` | 0.85 |
| `WALLET_STACK_SCALE_OUT` | 0.8 | `WALLET_STACK_TILT_DEG` | 12 |
| `WALLET_STACK_FLICK_VELOCITY` | 400 | | |

`resolveWalletCardHeight(size) = size === 'half' ? 188 : 228` (L2878435).

`WALLET_COPY` [O]:
```ts
{ invoicePrefix:'Fatura', invoiceFallback:'Fatura', usedLimit:'Limite usado', pay:'Pagar',
  statusOpen:'Aberta', statusClosed:'Fechada', statusDueToday:'Vence Hoje', statusOverdue:'Vencida',
  offerEyebrow:'Cartão de crédito', offerProduct:'Pierre One', offerHeadline:'3% de cashback\nem compras', offerCta:'Pedir',
  openFinanceTotal:'Total em faturas', openFinanceNextDue:'Próximo vencimento:', connectBanks:'Conectar bancos',
  openPierreCard:'Abrir cartão Pierre One', openOpenFinance:'Abrir faturas dos cartões' }
```
`WALLET_TEST_IDS`: `root 'wallet-widget'`, `pager 'wallet-widget-pager'`, `singlePane 'wallet-widget-single-pane'`, `dots`, `dot(i) 'wallet-widget-dot-'+i`, `dotActive`, `dotsHost`, `stack 'wallet-widget-stack'`, `stackPane(i)`, `dotsIndicator`, `page(i) 'wallet-widget-page-'+i`, `loading`, `error`, `pierrePane 'wallet-pierre-pane'`, `pierreTap`, `pierreHeader`, `pierreMonth`, `pierreAmount`, `pierreStatus`, `pierrePay`, `pierreLimitRing`, `pierreLimitAmount`, `offerPane`, `offerTap`, `offerRequest`, `openFinancePane`, `openFinanceTap`, `openFinanceHeader`, `openFinanceStack`, `openFinanceCard(i) 'wallet-open-finance-card-'+i`, `openFinanceHero`, `openFinanceTotal`, `openFinanceDueDate`, `openFinanceConnect`.

### 2.3 Dados

#### 2.3.1 `useWalletData()` (M9090, L2887568) [O]
```ts
const primary = useVisionSystemWidgetData('wallet', 'full')               // sempre pede size=full
const fallback = primary.isError && primary.data === undefined
const cardsQ  = useVisionSystemWidgetData('cards', 'full', undefined, { enabled: fallback })
const pierre  = useVisionPierreCardInvoices()                            // { isSettled, view }
const base    = fallback ? (cardsQ.data ? walletDataFromCards(cardsQ.data) : undefined) : primary.data
const data    = pierre.isSettled ? (mergePierreCardIntoWalletWidget(base, pierre.view) ?? undefined) : undefined
const q       = fallback ? cardsQ : primary
return { data, isLoading: q.isLoading || !pierre.isSettled, isError: q.isError,
         isFetched: q.isFetched && pierre.isSettled, isFallback: fallback }
```
Payload `wallet` (= saída de `walletDataFromCards`):
```ts
type WalletData = {
  totalInvoiceAmount: string          // "R$ 3.210,55"
  nextDueDateLabel: string | null     // "15 de mai" (sem "Vence ")
  cards: { id: string; bankName: string; logoUrl: string | null; lastFourDigits: string }[]   // máx. 3
  totalCount: number
}
```
`walletDataFromCards(cardsWidget)` (L2887574): soma `parseBRLToCents(card.currentInvoiceAmount)` de **todos** os cartões → `formatCentsToBRL(total)`; `nextDueDateLabel` = `dueDateLabel` do **primeiro** cartão com valor > 0 cujo rótulo não esteja em `{'Sem vencimento','Sem fatura aberta'}`, com `/^Vence\s+/i` removido; `cards = cards.slice(0, 3).map(({id, bankName, logoUrl ?? null, lastFourDigits}))`; `totalCount = cards.length` (total antes do corte).
`mergePierreCardIntoWalletWidget` (M9000) injeta a fatura Pierre One no agregado (detalhe em M9000, L2847697) [I: o cartão Pierre entra na lista e na soma].

#### 2.3.2 `useWalletPierreCard()` (M9091, L2887928) [O]
- `useCardSync()` → `{hasActiveCard, isLoading}`; skin do cartão via `useCardStore(s => s.skin)`; `useHomeSections().enabledIds.includes('cards')`.
- `useCurrentInvoice({enabled})` e `useCreditAvailable({enabled})` (habilitados quando há cartão ativo).
- `pane`:
  - `hasActiveCard` → `buildPierreCardPane({ invoice, credit, skinBackground: skin.background })`
  - senão, se a seção `cards` está habilitada → `buildPierreOfferPane(SKINS.PIERRE_ONE_DEFAULT)`
  - senão → `null`.
- `handlePay`: `Haptics.impactAsync(Medium)` → `track(CARD_PAYMENT_STEP_EVENT, {step:'invoice', action:'pay_tapped', entry_point:'vision_wallet_widget', invoice_status: toInvoiceStatusBucket(status)})` → `router.push({ pathname: ROOT_PAYMENT_VALUE_PATH, params: {invoiceId, totalAmount, totalAmountCents, minimumPaymentCents (remaining ?? minimum), cycleEndDate ?? '', invoiceStatus} })`.
- `handleOpenCardSection`: `router.setParams({ section: 'cards' })` (troca a home para a aba de cartões).
- `handleRequestCard`: `Haptics.impactAsync(Light)` → `track(CARD_ONBOARDING_STEP_EVENT, {step:'vision_wallet_offer', action:'request_card_tapped'})` → `handleOpenCardSection()`.
- `WALLET_PIERRE_ENTRY_POINT = 'vision_wallet_widget'`.

#### 2.3.3 Painéis (M9089, L2886901) [O]
```ts
buildWalletPanes({ pierre, openFinance }) =
  !pierre                       ? [openFinance] :
  pierre.kind === 'pierre-card' ? [pierre, openFinance] :     // cartão ativo: Pierre primeiro
                                  [openFinance, pierre]       // oferta: Open Finance primeiro
openFinance = { kind:'open-finance', data, isLoading: isLoading || !isFetched, isError }
```
`buildPierreCardPane({invoice, credit, skinBackground, today = new Date()})` →
```ts
{ kind:'pierre-card', skinBackground,
  monthLabel: 'Fatura ' + getFullMonthNamePt(month(invoice.referenceMonth) ?? mêsAtual).toLocaleLowerCase('pt-BR')   // ex.: "Fatura maio"; sem nome → "Fatura"
  amountCents: invoice.totalDueCents ?? invoice.totalAmountCents ?? 0,
  status, statusLabel, showPay, limit }
status = !invoice ? 'open'
       : ['overdue','revolving'].includes(invoice.status) ? 'overdue'
       : ['closed','partially_paid'].includes(invoice.status) ? (invoice.dueDate.slice(0,10) === hojeLocal('YYYY-MM-DD') ? 'due-today' : 'closed')
       : 'open'
statusLabel = { open:'Aberta', closed:'Fechada', 'due-today':'Vence Hoje', overdue:'Vencida' }[status]
showPay = status !== 'open' && !['paid','paid_revolving','canceled','compulsory_installment_revolving'].includes(invoice.status) && amountCents > 0
limit   = showPay ? null : buildPierreLimitView(credit)
buildPierreLimitView(c) = !c ? null : (() => {
  const limit = Math.max(0, c.creditLimitCents)
  const used  = Math.max(0, Number.isFinite(c.usedCreditCents) ? c.usedCreditCents : limit - c.availableCreditCents)
  const pct   = limit > 0 ? Math.min(100, Math.max(0, used / limit * 100)) : 100
  return { usedCents: used, usedPercentage: pct, tone: pct <= 30 ? 'high' : pct <= 85 ? 'medium' : 'low' }   // tone = "saúde" do limite
})()
```
`buildPierreOfferPane(skin) = { kind:'pierre-offer', skinBackground: skin.background, skinTextColor: skin.frontTextColor, skinLabelColor: skin.labelColor, skinOverlayColor: skin.overlayColor }`.

#### 2.3.4 `WalletWidgetConnected` (L2886483) [O]
```ts
size = props.size ?? 'full'
const router = useRouter(); const { trackWidgetInteraction } = useVisionAnalytics()
const pierre = useWalletPierreCard(); const q = useWalletData()
const panes = buildWalletPanes({ pierre: pierre.pane, openFinance: { kind:'open-finance', data: q.data ?? null, isLoading: q.isLoading || !q.isFetched, isError: q.isError } })
const trackTap = (target) => trackWidgetInteraction('tapped', { widget_id:'wallet', widget_type:'system', widget_size:size, source_screen:'home', tap_target: target })
const onConnectBanks = () => { trackTap('connect_banks'); router.push('/bank-connect') }
const guard = () => { const s = useVisionWidgetOrderStore.getState(); return s.isContextMenuOpen || s.shouldSuppressWidgetTap() }
const onOpenPierreCard  = useSinglePress(() => { if (guard()) return; trackTap('pierre'); pierre.handleOpenCardSection() })
const hasCards = (q.data?.cards.length ?? 0) > 0
const onOpenOpenFinance = useSinglePress(() => { if (guard()) return; if (!hasCards) return onConnectBanks(); trackTap('open_finance'); router.push(WALLET_OPEN_FINANCE_INVOICES_ROUTE) })
return <WalletWidget panes={panes} size={size} isLoading={pierre.isLoading}
         onPayPierreInvoice={pierre.handlePay} onRequestPierreCard={pierre.handleRequestCard}
         onConnectBanks={onConnectBanks} onOpenPierreCard={onOpenPierreCard} onOpenOpenFinance={onOpenOpenFinance}/>
```
- `WALLET_BANK_CONNECT_ROUTE = '/bank-connect'`.
- `WALLET_OPEN_FINANCE_INVOICES_ROUTE = VISION_WIDGET_REGISTRY.cards?.navigateTo ?? '/(protected)/vision/card-invoices?filterKey=invoices'`.

### 2.4 `WalletWidget` (memo, M9073, L2878485) [O]
Props: `panes, size = 'full', isLoading = false, onPayPierreInvoice, onRequestPierreCard, onConnectBanks, onOpenPierreCard, onOpenOpenFinance, testID = 'wallet-widget'`.

Estado: `width` (useState 0, medido por `onLayout` → `Math.round(layout.width)`), `activeIndex` (useState 0), `activeRef` (useRef 0), `scrollX` (useSharedValue 0).
```ts
height  = size === 'half' ? 188 : 228                      // ≠ 200 dos outros half!
multi   = panes.length > 1
stack   = useWalletVerticalStack({ count: panes.length, pageHeight: height, onPageChange: setActiveIndex })
onScroll = useAnimatedScrollHandler({ onScroll: e => { scrollX.value = e.contentOffset.x } })     // worklet WalletWidgetTsx1
onMomentumScrollEnd = e => {
  const i = Math.min(Math.max(Math.round(e.nativeEvent.contentOffset.x / width), 0), Math.max(panes.length - 1, 0))
  if (i !== activeRef.current) { activeRef.current = i; haptics.selection(); setActiveIndex(i) }   // háptico de seleção (M2991)
}
reserveDotsSpace = size === 'half' ? false : multi
renderPane = pane => pane.kind === 'pierre-card'  ? <WalletPierreCardPane  pane size onPay={onPayPierreInvoice} reserveDotsSpace/>
                   : pane.kind === 'pierre-offer' ? <WalletPierreOfferPane pane size onRequest={onRequestPierreCard} reserveDotsSpace/>
                   :                                <WalletOpenFinancePane pane size onConnect={onConnectBanks} reserveDotsSpace/>
renderPaneTap = pane => {
  const onPress = pane.kind === 'open-finance' ? onOpenOpenFinance : onOpenPierreCard
  return <Pressable style={{flex:1}} onPress={onPress} disabled={!onPress} accessibilityRole={onPress ? 'button' : undefined}
           accessibilityLabel={pane.kind === 'open-finance' ? 'Abrir faturas dos cartões' : 'Abrir cartão Pierre One'}
           testID={{'pierre-card':'wallet-pierre-tap','pierre-offer':'wallet-offer-tap','open-finance':'wallet-open-finance-tap'}[pane.kind]}>
           {renderPane(pane)}</Pressable>
}
```
Árvore por caso:
```
(A) half && multi && !isLoading && width > 0  → PILHA VERTICAL
View style={[{borderRadius:24, overflow:'hidden'}, {flex:1}, {height:188}]} onLayout testID="wallet-widget" collapsable={false}
 ├─ WalletVerticalStack count width height progress gesture renderPane={i => renderPaneTap(panes[i])}
 └─ View pointerEvents="none" testID="wallet-widget-dots-host" style={{position:'absolute', top:0, bottom:0, right:6, justifyContent:'center'}}
     └─ WalletVerticalDots count progress visibility={stack.dotsVisibility} color={foreground #fafafa} activeIndex

(B) demais casos
WidgetGlassCard style={[{borderRadius:24, overflow:'hidden'}, {height}]} testID="wallet-widget-card"      // Android: VStack bg-surface #18181b
 └─ View style={[{flex:1}, {height}]} onLayout testID="wallet-widget" collapsable={false}
     └─ isLoading || width === 0
          ? View style={{flex:1, padding:16, justifyContent:'center'}} testID="wallet-widget-loading" → WidgetLoadingState variant="content"
          : multi
            ? <>
                WalletPagerScrollView horizontal pagingEnabled showsHorizontalScrollIndicator={false} onScroll scrollEventThrottle={16}
                     decelerationRate="fast" onMomentumScrollEnd directionalLockEnabled nestedScrollEnabled style={{flex:1}}
                     contentContainerStyle={{width: width * panes.length}} testID="wallet-widget-pager"
                  └─ panes.map((p, i) => <WalletPagerPage key={`${p.kind}-${i}`} index={i} pageWidth={width} scrollX>{renderPaneTap(p)}</WalletPagerPage>)
                View pointerEvents="none" testID="wallet-widget-dots-host" style={{position:'absolute', left:0, right:0, bottom:8, alignItems:'center'}}
                  └─ WalletPagerDots count activeIndex scrollX pageWidth={width}
              </>
            : View style={{width, height}} testID="wallet-widget-single-pane" collapsable={false} → renderPaneTap(panes[0])
```
Observações: na carteira não há título nem chevron de widget; o "título" visual é a arte do cartão no topo de cada painel. No (A) cada painel tem seu próprio fundo (ver §2.8).

### 2.5 Pager horizontal (full) — M9079/M9080/M9081/M9082 [O]
- `WalletPagerScrollView = Animated.createAnimatedComponent(ScrollView do react-native-gesture-handler)`; `USES_GESTURE_HANDLER_PAGER = true`.
- **WalletPagerPage** (L2882777): `Animated.View style={[{height:228, overflow:'hidden'}, {width: pageWidth}, animStyle]} testID="wallet-widget-page-{i}"`, com `animStyle = { transform: [{ scale }] }` (worklet `WalletPagerPageTsx1`).
- Funções worklet (fonte literal do bundle, `walletPagerTs*`):
```ts
const WALLET_DOT_SLOT = 8 + 4            // DOT_SIZE + DOT_GAP = 12
const WALLET_DOT_PILL_EXTRA = 20 - 8     // ACTIVE_WIDTH - DOT_SIZE = 12
clamp = (v, min, max) => Math.min(Math.max(v, min), max)
resolvePageProgress = (scrollX, index, pageWidth) => pageWidth <= 0 ? 0 : clamp((index * pageWidth - scrollX) / pageWidth, -1, 1)
resolvePageScale    = progress => 1 - (1 - 0.9) * Math.abs(progress)                   // 1 → 0,9
resolvePageTransform = (scrollX, index, pageWidth) => ({ scale: resolvePageScale(resolvePageProgress(scrollX, index, pageWidth)) })
resolveDotTranslateX = (scrollX, index, pageWidth) => { if (index <= 0 || pageWidth <= 0) return 0
  const start = (index - 1) * pageWidth; const t = clamp((scrollX - start) / pageWidth, 0, 1); return WALLET_DOT_PILL_EXTRA * (1 - t) }
resolvePillTranslateX = (scrollX, count, pageWidth) => { if (count <= 1 || pageWidth <= 0) return 0
  const page = clamp(scrollX / pageWidth, 0, count - 1); return page * WALLET_DOT_SLOT }
resolveDotsTrackWidth = count => count * 8 + Math.max(count - 1, 0) * 4 + WALLET_DOT_PILL_EXTRA
```
- **WalletPagerDots({count, activeIndex, scrollX, pageWidth})** (L2881908): trilho `View style={{height:8, position:'relative', width: resolveDotsTrackWidth(count)}} testID="wallet-widget-dots"` contendo
  - `count` × **Dot**: `Animated.View className="bg-foreground/30"` (rgba(250,250,250,0,3)) `style={[{position:'absolute', top:0, width:8, height:8, borderRadius:4}, {left: i * 12}, {transform:[{translateX: resolveDotTranslateX(scrollX, i, pageWidth)}]}]}` `accessibilityState={{selected: i === activeIndex}}` testID `wallet-widget-dot-active` (ativo) ou `wallet-widget-dot-{i}`;
  - 1 × **Pill**: `Animated.View className="bg-foreground"` (#fafafa) `pointerEvents="none"` `style={[{position:'absolute', top:0, left:0, width:20, height:8, borderRadius:4}, {transform:[{translateX: resolvePillTranslateX(scrollX, count, pageWidth)}]}]}`.
  - Efeito: a pílula ativa (20×8) desliza 12 px por página, e os pontos à direita dela abrem espaço deslocando-se 12 px; tudo acompanha o dedo (scroll contínuo), não só ao soltar.
  - Posição: centralizado, `bottom: 8` do card. Por isso os painéis com pontos usam `paddingBottom: 36` (`WALLET_PANE_BOTTOM_WITH_DOTS`) e sem pontos `16`.

### 2.6 Pilha vertical (half) — M9087 [O]
`useWalletVerticalStack({count, pageHeight, onPageChange})` (L2884782): shared values `progress = 0`, `startProgress = 0`, `dotsVisibility = 0`.
```ts
commit = useCallback((from, to) => { if (from !== to) { haptics.selection(); onPageChange?.(to) } }, [onPageChange])
gesture = Gesture.Pan()
  .activeOffsetY([-8, 8]).failOffsetX([-16, 16])
  .onStart(() => { startProgress.value = progress.value; dotsVisibility.value = withTiming(1, { duration: 150 }) })
  .onUpdate(e => { if (pageHeight <= 0) return; progress.value = startProgress.value - e.translationY / pageHeight })
  .onEnd(e => { if (pageHeight <= 0) return
     const target = resolveStackTarget(progress.value, e.velocityY)
     const from = resolvePageIndex(startProgress.value, count)
     progress.value = withSpring(target, { duration: 400, dampingRatio: 0.8, velocity: -e.velocityY / pageHeight })
     dotsVisibility.value = withDelay(600, withTiming(0, { duration: 250 }))
     runOnJS(commit)(from, resolvePageIndex(target, count)) })
```
Worklets (fonte literal, `WalletVerticalStackTsx*`):
```ts
lerpRange = (v, input, output) => { const last = input.length - 1; if (v <= input[0]) return output[0]; if (v >= input[last]) return output[last]
  let i = 0; while (i < last && v > input[i + 1]) i += 1; const span = input[i + 1] - input[i]
  const t = span === 0 ? 0 : (v - input[i]) / span; return output[i] + (output[i + 1] - output[i]) * t }
mod = (v, n) => ((v % n) + n) % n
resolvePageIndex = (progress, count) => count <= 0 ? 0 : mod(Math.round(progress), count)
wrapDistance = (index, progress, count) => { if (count <= 1) return index - progress; const raw = mod(index - progress, count); return raw > count / 2 ? raw - count : raw }
computeStackPaneMotion = (index, progress, height, reduceMotion, count = Infinity) => {
  const d = Number.isFinite(count) ? wrapDistance(index, progress, count) : index - progress; const distance = Math.abs(d)
  return { translateY: d * height * 0.9,                                                 // WALLET_STACK_TRAVEL
           scale: reduceMotion ? 1 : lerpRange(distance, [0, 0.5, 1], [1, 0.85, 0.8]),    // SCALE_MID / SCALE_OUT
           tilt: reduceMotion ? 0 : -clamp(d, -1, 1) * 12,                               // graus
           zIndex: Math.round((1 - Math.min(distance, 1)) * 10) } }
resolveStackTarget = (progress, velocityY) => velocityY < -400 ? Math.ceil(progress) : velocityY > 400 ? Math.floor(progress) : Math.round(progress)
```
- O carrossel é **circular** (`mod`/`wrapDistance`): depois do último volta ao primeiro. Arrastar para cima avança.
- **WalletVerticalStack({count, width, height, progress, gesture, renderPane})**: `GestureDetector gesture` → `View style={[{overflow:'hidden'}, {width, height}]} testID="wallet-widget-stack"` → `Array.from({length: count}, (_, i) => <WalletStackPane key={i} index={i} …reduceMotion={useReducedMotion()}>{renderPane(i)}</WalletStackPane>)`.
- **WalletStackPane**: `Animated.View style={[{position:'absolute', top:0, left:0}, {width, height}, anim]} testID="wallet-widget-stack-pane-{i}"` com `anim = { zIndex, transform: [{perspective: 800}, {translateY}, {rotateX: tilt + 'deg'}, {scale}] }`; dentro, `WidgetGlassCard style={[{borderRadius:24, overflow:'hidden'}, {width, height}]}` → painel.
- **WalletVerticalDots({count, progress, visibility, color, activeIndex})** (só se `count > 1`; `DOT_PITCH = 4 + 4 = 8`):
```
Animated.View style={[{flexDirection:'column', alignItems:'center', gap:4, width:4, overflow:'hidden'}, {opacity: visibility.value}]}
              testID="wallet-widget-dots" accessibilityRole="tablist" accessibilityValue={{now: activeIndex + 1, min: 1, max: count}}
 ├─ count × View style={[{width:4, height:4, borderRadius:2}, {opacity:0.3}, {backgroundColor: color}]} testID="wallet-widget-dot-{i}"
 ├─ Animated.View style={[dot, {position:'absolute', top:0, left:0}, {backgroundColor: color}, {transform:[{translateY: mod(progress, count) * 8}]}]} testID="wallet-widget-dots-indicator"
 └─ Animated.View (idem, translateY: (mod(progress, count) - count) * 8)  testID="wallet-widget-dots-indicator-trail"   // "rabo" para o wrap circular
```
Os pontos verticais (4×4, cor #fafafa, faixa a 6 px da borda direita) ficam **invisíveis** em repouso e aparecem só durante o arraste: fade-in de 150 ms no início e fade-out de 250 ms começando 600 ms depois de soltar.

### 2.7 Arte do cartão no topo: `WalletHeaderMask` + `WalletPierreHeader` (M9076, M9084) [O]
Todo painel começa com um "header" de 77 px desenhado em duas camadas recortadas, que formam uma faixa arredondada com uma "aba" central embaixo:
```
WalletHeaderMask({renderContent(width, part), testID})
View style={{height:77, width:'100%'}} onLayout(setWidth(round)) collapsable={false} testID
 ├─ View style={{position:'absolute', left:0, right:0, top:0, height:67, borderRadius:16, overflow:'hidden'}}     // "strip"
 │    └─ View style={{width, height:77}} → renderContent(width, 'strip')
 └─ width > 0 && View style={[{position:'absolute', top:25, width:54, height:52, borderRadius:27, overflow:'hidden'}, {left: Math.round((width - 54) / 2)}]} testID="{testID}-bump"
      └─ View style={{position:'absolute', left: -bumpLeft, top: -25, width, height:77}} → renderContent(width, 'bump')
```
O mesmo conteúdo é desenhado duas vezes e recortado por duas máscaras: retângulo 67 px de altura com raio 16 + pílula 54×52 (raio 27) centrada entre y = 25 e y = 77. O resultado é uma faixa com uma língua de 10 px saindo pela borda de baixo, no centro.
O header fica dentro de `View style={{paddingHorizontal:8, paddingTop:8}}` em todos os painéis (`WALLET_HEADER_INSET` = 8).

`WalletPierreHeader({skinBackground})`: `renderContent = w => View style={{width: w, height:77}} → expo-image Image source={skinBackground} style={[{position:'absolute', top:0, left:0}, {width: w, height: Math.ceil(w / 1.586)}]} contentFit="cover" contentPosition="top" cachePolicy="memory-disk"`. Ou seja, mostra o **topo da arte do cartão** (proporção ISO 1,586) recortado.

### 2.8 Painel Pierre One com cartão — `WalletPierreCardPane` (M9083, L2883077) [O]
```
View style={{flex:1}} testID="wallet-pierre-pane"
 ├─ View style={{paddingHorizontal:8, paddingTop:8}} → WalletPierreHeader skinBackground testID="wallet-pierre-header"
 └─ (full)
    VStack className="px-4 justify-end" style={[{flex:1}, {paddingBottom: reserveDotsSpace ? 36 : 16}]}      // px 14
     ├─ HStack className="items-start justify-between gap-3" style={{marginBottom:12}}       // infoRow / infoRowCompact (iguais)
     │   ├─ VStack className="flex-1 gap-1"                                                 // gap 3,5
     │   │   ├─ Text "text-body-small text-muted-foreground" numberOfLines=1 testID="wallet-pierre-month" → monthLabel ("Fatura maio")   // Geist-Regular 14/18 #a1a1aa
     │   │   └─ PrivacyCurrencyText valueCents={amountCents} hidden fontSize={20} fontWeight="700" color=#fafafa disableMinWidth testID="wallet-pierre-amount"
     │   └─ View style={{paddingTop:12}} → WalletStatusChip status label testID="wallet-pierre-status"
     └─ showPay
          ? PierreButton variant={status === 'overdue' ? 'destructive' : 'primary'} size="regular" fullWidth onPress={onPay} testID="wallet-pierre-pay" → 'Pagar'
          : limit
            ? HStack className="items-center gap-3" style={{height:45}}
                ├─ WalletLimitRing usedPercentage tone testID="wallet-pierre-limit-ring"
                ├─ Text "flex-1 text-body-small text-foreground" numberOfLines=1 → 'Limite usado'     // 14/18 #fafafa
                └─ PrivacyCurrencyText valueCents={limit.usedCents} hidden fontSize={14} fontWeight="500" color=#fafafa testID="wallet-pierre-limit-amount"
            : View style={{height:45}}                                                     // reserva a altura
    (half)
    VStack className="px-4 justify-end gap-1" style={[{flex:1}, {paddingBottom:16}]} → [Text monthLabel, PrivacyCurrencyText amount (20/700)]   // sem chip, sem botão, sem anel
```
O toque no painel inteiro abre a seção de cartões (`onOpenPierreCard`); o botão "Pagar" tem o próprio `onPress` (háptico Medium).

**WalletStatusChip({status, label})** (M9085): `View style={[{borderRadius:999, paddingHorizontal:8, paddingVertical:2}, {backgroundColor: bg}]}` → `Text className="font-medium" style={[{fontSize:14, lineHeight:14}, {color: text}]}` (Geist-Medium).
| status | label | fundo | texto |
|---|---|---|---|
| `open` | Aberta | `successSubtle` #243400 | `success` #cfff04 |
| `closed` | Fechada | `warningSubtle` #451a03 | `warning` #f59e0b |
| `due-today` | Vence Hoje | `warningSubtle` #451a03 | `warning` #f59e0b |
| `overdue` | Vencida | `destructiveSubtle` #450a0a | `destructive` #ff4d4d (THEME_TOKENS) |

**WalletLimitRing({usedPercentage, tone})** (M9077): rosca nativa `PierreDonutChart` (M4584) dentro de `View style={{width:45, height:45}} accessibilityRole="progressbar" accessibilityLabel={`${round(p)}% do limite usado`} pointerEvents="none"`:
```ts
p = Math.min(100, Math.max(0, usedPercentage))
color = tone === 'high' ? success (#cfff04) : tone === 'medium' ? warning (#f59e0b) : destructive (#ff4d4d)   // tone: ≤30 high, ≤85 medium, >85 low
<PierreDonutChart data={[{id:'used', label:'Usado', value:p, color}, {id:'available', label:'Disponível', value:100-p, color: muted #27272a}]}
  animated animationKey={`${Math.round(p)}-${tone}`} chartBackgroundColor="transparent" ringWidth={4}
  segmentGapDegrees={p > 0 && p < 100 ? 10 : 0} groupingEnabled={false} expandOnPress={false} showNavigationControl={false}
  loadingMode="none" style={{position:'absolute', top:0, left:0, width:45, height:45}}/>
```

### 2.9 Painel de oferta — `WalletPierreOfferPane` (M9086, L2884212) [O]
```
View style={{flex:1, overflow:'hidden'}} testID="wallet-offer-pane"
 ├─ expo-image Image source={skinBackground} style={StyleSheet.absoluteFill} contentFit="cover"
 │      contentPosition={size === 'half' ? 'right' : 'center'} cachePolicy="memory-disk" testID="wallet-offer-pane-art"
 ├─ LinearGradient colors={['transparent', skinOverlayColor, skinOverlayColor]} locations={[0.3, 0.65, 1]} style={absoluteFill} pointerEvents="none"   // vertical, de cima para baixo
 └─ (full) HStack className="items-end justify-between gap-3 px-4" style={[{flex:1}, {paddingBottom}]}
            ├─ VStack className="flex-1 gap-1"
            │   ├─ Text "text-body-small" style={{color: skinLabelColor}} numberOfLines=1 → ['Cartão de crédito', ' ', <Text "text-body-small" style={{color: skinTextColor}}>Pierre One</Text>]
            │   └─ Text "text-xl font-semibold" style={{color: skinTextColor}} numberOfLines=2 → '3% de cashback\nem compras'   // Geist-SemiBold 17,5/24,5
            └─ PierreButton variant="primary" size="small" onPress={onRequest} testID="wallet-offer-request" → 'Pedir'
    (half) VStack className="justify-end gap-2 px-4" style={[{flex:1}, {paddingBottom:16}]}
            ├─ VStack className="gap-1" → [eyebrow, Text "text-base font-semibold" (14/21) numberOfLines=2 → headline]
            └─ PierreButton primary small 'Pedir'
```
As cores (`background` = imagem, `frontTextColor`, `labelColor`, `overlayColor`) vêm de `SKINS.PIERRE_ONE_DEFAULT` (módulo de skins do cartão, fora deste escopo) [O nome; valores I].

### 2.10 Painel Open Finance — `WalletOpenFinancePane` (M9078, L2880872) [O]
```ts
cards     = pane.data?.cards ?? []
hasCards  = (pane.data?.totalCount ?? cards.length) > 0
```
```
View style={{flex:1}} testID="wallet-open-finance-pane"
 ├─ View style={{paddingHorizontal:8, paddingTop:8}}
 │   └─ hasCards
 │       ? WalletHeaderMask testID="wallet-open-finance-header" renderContent={(w, part) => <OpenFinanceStack cards width={w} withTestIds={part === 'strip'}/>}
 │       : Pressable onPress={onConnect} disabled={!onConnect} accessibilityRole="button" accessibilityLabel="Conectar bancos" testID="wallet-open-finance-connect"
 │           └─ WalletHeaderMask testID="wallet-open-finance-header" renderContent={w => <ConnectStrip width={w}/>}
 └─ VStack className="px-4 justify-end" style={[{flex:1}, {paddingBottom: reserveDotsSpace ? 36 : 16}]}
     └─ pane.isError && !data   → View style={{height:68, justifyContent:'center'}} testID="wallet-widget-error" → WidgetErrorState compact
        pane.isLoading && !data → View style={{height:68, justifyContent:'center'}} → WidgetLoadingState variant="value"
        senão → VStack className="gap-1"
                 ├─ Text "text-body-small text-muted-foreground" numberOfLines=1 → 'Total em faturas'
                 ├─ PrivacyCurrencyText valueCents={parseBRLToCents(data?.totalInvoiceAmount ?? 'R$ 0,00')} hidden fontSize={20} fontWeight="700" color=#fafafa disableMinWidth testID="wallet-open-finance-total"
                 └─ (size !== 'half' && hasCards && data.nextDueDateLabel) &&
                      Text "text-body-small text-muted-foreground" numberOfLines=1 testID="wallet-open-finance-due-date" → `Próximo vencimento:  ${nextDueDateLabel}`   // dois espaços [O]
```
**OpenFinanceStack({cards, width, withTestIds})** — a "pilha" de cartões de banco vista de lado (L2880999):
```ts
const list = cards.slice(0, 3); const single = list.length === 1
bandHeight(i) = i !== list.length - 1 ? 22 : 67 - i * 22          // resolveStackBandHeight: faixas de 22, a última ocupa o resto
logoSize(band) = band >= 40 ? 36 : 22                             // resolveStackLogoSize
```
```
View style={{width, height:77}} testID="wallet-open-finance-stack"
 └─ list.map((card, i) =>
      View key={card.id} testID="wallet-open-finance-card-{i}"
           style={[{position:'absolute', left:0, bottom:0, borderTopLeftRadius:16, borderTopRightRadius:16}, {top: i * 22, width, backgroundColor: resolveBankColor(card.bankName)}]}
       └─ single
          ? View style={{height:67, justifyContent:'center'}} testID="wallet-open-finance-hero"
              ├─ View style={{alignSelf:'flex-start'}} → BankLogo bankName imageUrl={card.logoUrl} size={68}
              └─ Text "text-body-small font-medium" style={[{color:'#FFFFFF'}, {position:'absolute', right:16, top:0, bottom:0, lineHeight:67}]} numberOfLines=1 → `•••• ${lastFourDigits}`
          : HStack className="items-center justify-between" style={[{paddingLeft:12, paddingRight:16}, {height: bandHeight(i)}]} testID="wallet-open-finance-card-{i}-row"
              ├─ BankLogo bankName imageUrl={card.logoUrl} size={logoSize(bandHeight(i))}
              └─ Text "text-body-small font-medium" style={{color:'#FFFFFF'}} numberOfLines=1 → `•••• ${lastFourDigits}`)
```
Cada cartão é um retângulo com cantos superiores de raio 16, cor sólida da marca do banco (§5.2), começando 22 px abaixo do anterior; o último fica por cima e mostra a logo maior. Exemplo com 3 cartões: faixas de 22, 22 e 23 px (67 − 44), logos 22, 22 e 22 (23 < 40); com 2: 22 e 45 → logos 22 e 36. Texto dos 4 dígitos em Geist-Medium 14/18, branco. Não há chip, bandeira (Visa/Master) nem gradiente: a "carteira" é só faixa colorida + logo + `•••• 1234`.

**ConnectStrip({width})** (sem cartões): `View className="bg-surface-hover" (#27272a) style={[{position:'absolute', left:0, bottom:0, borderTopLeftRadius:16, borderTopRightRadius:16}, {top:0, width}]}` → `HStack className="items-center justify-center gap-2" style={{height:67}}` → `[PierreIcon name="calcAdd" size=24 color=#fafafa, Text "text-body-small font-medium text-foreground" 'Conectar bancos']`.

### 2.11 Dimensões resumidas
| | full | half |
|---|---|---|
| Card | `W−32` × **228**, raio 24, bg #18181b | `(W−50)/2` × **188**, raio 24 |
| Header (arte) | inset 8; faixa 67 (raio 16) + aba 54×52 (raio 27) = 77 | idem |
| Corpo | `px-4` (14), `paddingBottom` 36 com pontos / 16 sem | 16 |
| Navegação | pager horizontal + pontos (pílula 20×8, pontos 8×8, gap 4, bottom 8) | pilha vertical (Pan) + pontos verticais 4×4 a 6 px da direita |
| Conteúdo | mês/valor + chip + Pagar/anel; total + próximo vencimento | só mês/valor ou total |

### 2.12 Strings pt-BR (carteira) [O]
`Fatura`, `Fatura {mês}`, `Limite usado`, `Pagar`, `Aberta`, `Fechada`, `Vence Hoje`, `Vencida`, `Cartão de crédito`, `Pierre One`, `3% de cashback\nem compras`, `Pedir`, `Total em faturas`, `Próximo vencimento:`, `Conectar bancos`, `Abrir cartão Pierre One`, `Abrir faturas dos cartões`, `Usado`, `Disponível`, `{n}% do limite usado`, `•••• {1234}`, `Sem vencimento` / `Sem fatura aberta` (rótulos ignorados), `R$ 0,00`.

### 2.13 Snippet TSX (núcleo da carteira: header recortado + pilha Open Finance + pager com pontos)
```tsx
import React, { useState } from 'react';
import { LayoutChangeEvent, Pressable, StyleSheet, Text, View } from 'react-native';
import Animated, { useAnimatedScrollHandler, useAnimatedStyle, useSharedValue, SharedValue } from 'react-native-reanimated';
import { ScrollView } from 'react-native-gesture-handler';
// BankLogo/resolveBankColor: §5; PrivacyCurrencyText/useHideValues: w0-base.md

const AScrollView = Animated.createAnimatedComponent(ScrollView);
const H_FULL = 228, H_HALF = 188, HDR_H = 67, HDR_TOTAL = 77, HDR_R = 16, BUMP_W = 54, BUMP_H = 52, BUMP_TOP = 25, OFFSET = 22;
const DOT = 8, GAP = 4, PILL = 20, SLOT = DOT + GAP, EXTRA = PILL - DOT;

function HeaderMask({ render }: { render: (w: number, part: 'strip' | 'bump') => React.ReactNode }) {
  const [w, setW] = useState(0);
  const left = Math.round((w - BUMP_W) / 2);
  return (
    <View style={{ height: HDR_TOTAL, width: '100%' }} onLayout={(e: LayoutChangeEvent) => setW(Math.round(e.nativeEvent.layout.width))}>
      <View style={{ position: 'absolute', left: 0, right: 0, top: 0, height: HDR_H, borderRadius: HDR_R, overflow: 'hidden' }}>
        <View style={{ width: w, height: HDR_TOTAL }}>{w > 0 && render(w, 'strip')}</View>
      </View>
      {w > 0 && (
        <View style={{ position: 'absolute', top: BUMP_TOP, left, width: BUMP_W, height: BUMP_H, borderRadius: BUMP_W / 2, overflow: 'hidden' }}>
          <View style={{ position: 'absolute', left: -left, top: -BUMP_TOP, width: w, height: HDR_TOTAL }}>{render(w, 'bump')}</View>
        </View>
      )}
    </View>
  );
}

type Card = { id: string; bankName: string; logoUrl: string | null; lastFourDigits: string };
function OpenFinanceStack({ cards, width }: { cards: Card[]; width: number }) {
  const list = cards.slice(0, 3);
  return (
    <View style={{ width, height: HDR_TOTAL }}>
      {list.map((c, i) => {
        const band = i !== list.length - 1 ? OFFSET : HDR_H - i * OFFSET;
        const digits = <Text numberOfLines={1} style={{ color: '#FFFFFF', fontFamily: 'Geist-Medium', fontSize: 14, lineHeight: 18 }}>{`•••• ${c.lastFourDigits}`}</Text>;
        return (
          <View key={c.id} style={{ position: 'absolute', left: 0, bottom: 0, top: i * OFFSET, width, borderTopLeftRadius: HDR_R, borderTopRightRadius: HDR_R, backgroundColor: resolveBankColor(c.bankName) }}>
            {list.length === 1 ? (
              <View style={{ height: HDR_H, justifyContent: 'center' }}>
                <View style={{ alignSelf: 'flex-start' }}><BankLogo bankName={c.bankName} imageUrl={c.logoUrl} size={68} /></View>
                <View style={{ position: 'absolute', right: 16, top: 0, bottom: 0, justifyContent: 'center' }}>{digits}</View>
              </View>
            ) : (
              <View style={{ height: band, paddingLeft: 12, paddingRight: 16, flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between' }}>
                <BankLogo bankName={c.bankName} imageUrl={c.logoUrl} size={band >= 40 ? 36 : 22} />
                {digits}
              </View>
            )}
          </View>
        );
      })}
    </View>
  );
}

const clamp = (v: number, a: number, b: number) => { 'worklet'; return Math.min(Math.max(v, a), b); };
function Page({ index, w, scrollX, children }: { index: number; w: number; scrollX: SharedValue<number>; children: React.ReactNode }) {
  const st = useAnimatedStyle(() => {
    const p = w <= 0 ? 0 : clamp((index * w - scrollX.value) / w, -1, 1);
    return { transform: [{ scale: 1 - 0.1 * Math.abs(p) }] };
  });
  return <Animated.View style={[{ height: H_FULL, width: w, overflow: 'hidden' }, st]}>{children}</Animated.View>;
}
function Dot({ i, w, scrollX }: { i: number; w: number; scrollX: SharedValue<number> }) {
  const st = useAnimatedStyle(() => {
    if (i <= 0 || w <= 0) return { transform: [{ translateX: 0 }] };
    const t = clamp((scrollX.value - (i - 1) * w) / w, 0, 1);
    return { transform: [{ translateX: EXTRA * (1 - t) }] };
  });
  return <Animated.View style={[{ position: 'absolute', top: 0, left: i * SLOT, width: DOT, height: DOT, borderRadius: DOT / 2, backgroundColor: 'rgba(250,250,250,0.3)' }, st]} />;
}
function Dots({ count, w, scrollX }: { count: number; w: number; scrollX: SharedValue<number> }) {
  const pill = useAnimatedStyle(() => ({ transform: [{ translateX: count <= 1 || w <= 0 ? 0 : clamp(scrollX.value / w, 0, count - 1) * SLOT }] }));
  return (
    <View style={{ height: DOT, width: count * DOT + Math.max(count - 1, 0) * GAP + EXTRA }}>
      {Array.from({ length: count }, (_, i) => <Dot key={i} i={i} w={w} scrollX={scrollX} />)}
      <Animated.View pointerEvents="none" style={[{ position: 'absolute', top: 0, left: 0, width: PILL, height: DOT, borderRadius: DOT / 2, backgroundColor: '#fafafa' }, pill]} />
    </View>
  );
}

export function WalletPager({ panes, renderPane }: { panes: any[]; renderPane: (p: any) => React.ReactNode }) {
  const [w, setW] = useState(0);
  const scrollX = useSharedValue(0);
  const onScroll = useAnimatedScrollHandler({ onScroll: (e) => { scrollX.value = e.contentOffset.x; } });
  return (
    <View style={{ height: H_FULL, borderRadius: 24, overflow: 'hidden', backgroundColor: '#18181b' }}
      onLayout={(e) => setW(Math.round(e.nativeEvent.layout.width))}>
      {w > 0 && (
        <>
          <AScrollView horizontal pagingEnabled showsHorizontalScrollIndicator={false} onScroll={onScroll} scrollEventThrottle={16}
            decelerationRate="fast" directionalLockEnabled nestedScrollEnabled contentContainerStyle={{ width: w * panes.length }}>
            {panes.map((p, i) => <Page key={`${p.kind}-${i}`} index={i} w={w} scrollX={scrollX}>{renderPane(p)}</Page>)}
          </AScrollView>
          {panes.length > 1 && (
            <View pointerEvents="none" style={{ position: 'absolute', left: 0, right: 0, bottom: 8, alignItems: 'center' }}>
              <Dots count={panes.length} w={w} scrollX={scrollX} />
            </View>
          )}
        </>
      )}
    </View>
  );
}
```

---

## 3. `account-balance` — "Saldo em contas"

### 3.1 Identidade [O]
- id `account-balance`, label `Saldo em contas`, tamanhos `half | wide | full` (default do componente: `half`).
- `AccountBalanceWidgetConnected` (M8989, L2839174) → `AccountBalanceWidget` (memo, M8990, L2839825).
- Recebe `onPress` do renderer (navegação definida no registro). testID raiz `account-balance-widget`.

### 3.2 Connected: estados de borda (L2839174) [O]
```
{data, isLoading, isError, isFetched} = useAccountBalanceData(size)
boxStyle = size==='full' ? S.full : size==='wide' ? S.wide : S.half
if (isError)            → <View testID="account-balance-error" className="rounded-3xl bg-surface" style={boxStyle}><WidgetErrorState compact/></View>
if (isFetched && !data) → <View testID="account-balance-empty" className="rounded-3xl bg-surface" style={boxStyle}><WidgetEmptyStateSlot emptyState compact/></View>
else                    → <AccountBalanceWidget data size onPress isLoading={isLoading || !isFetched} emptyState animate/>
```
`S` (StyleSheet): `half {height:200, alignItems:'center', justifyContent:'center', padding:16}`, `wide {height:170, …}`, `full {minHeight:390, …}`.
Esses contêineres usam `rounded-3xl` = **21** de raio (não 24) e padding **16** (não 14). Textos: `Falha ao sincronizar dados` / `Sem dados disponíveis` (compactos, w0 §9).

### 3.3 Payload e cálculos (L2839825+) [O]
`data` (default quando `null`: `{totalBalance:'R$ 0,00', accountCount:0, subtitle:'', accounts:[]}`):
- `totalBalance: string` já formatada pelo servidor (ex.: `"R$ 12.345,67"`). `subtitle` **não é usado**.
- `accountCount: number`
- `accounts: Array<{ name, imageUrl, balance?: string, updatedLabel?: string }>`

```ts
parseBRL = (s) => { if (!s) return 0; let t = s.replace(/[^\d,.-]/g, '');
  if (t.includes(',')) t = t.replace(/\./g, '').replace(',', '.');
  const n = Number(t); return Number.isFinite(n) ? n : 0 }
totalCents = Math.round(parseBRL(totalBalance) * 100)

// pilha de logos (half/wide)
const MAX = 3
bankLogoItems = accounts.slice(0, MAX)
extraCount    = Math.max(0, Math.max(accountCount, accounts.length) - MAX)

// linhas do full
avg  = accounts.length > 0 ? parseBRL(totalBalance) / accounts.length : 0
rows = accounts.slice(0, 3).map(a => {
  const bal = (a.balance != null && String(a.balance).trim() !== '') ? parseBRL(a.balance) : avg  // sem saldo → média
  return { name: a.name, imageUrl: a.imageUrl, balance: bal,
    formattedBalance: a.balance ?? new Intl.NumberFormat('pt-BR', {style:'currency', currency:'BRL'}).format(bal),
    updatedLabel: a.updatedLabel ?? 'Atualizado agora' }
})
sum  = rows.reduce((s, r) => s + r.balance, 0)
rows = rows.map(r => ({ ...r, ...(sum > 0 ? { percentage: Math.round(r.balance / sum * 100) } : {}) }))
subtitle = accountCount === 1 ? '1 conta conectada' : `${accountCount} contas conectadas`
```
Ordem: a do servidor (sem sort no cliente). Só as 3 primeiras contas entram nas linhas e na barra do *full*.

### 3.4 Ramo de renderização [O]
| Condição | half | wide | full |
|---|---|---|---|
| `isLoading` | `Widget2x2 visual=<WidgetLoadingState variant="visual"/> value='' subtitle=''` (**sem onPress**) | `WidgetWide visual=<WidgetLoadingState variant="visual"/> value='' onPress` | `WidgetFull children=<WidgetLoadingState variant="content"/> onPress` |
| `accountCount === 0` | `Widget2x2 visual=<AccountBalanceEmptyIcon/> value='R$ 0,00' subtitle='' onPress` | `WidgetWide visual=<AccountBalanceEmptyIcon/> value='R$ 0,00' onPress` | `WidgetFull children=<AccountBalanceFullEmptyContent/> onPress` |
| normal | `Widget2x2 visual=<BankLogosStack/> valueSlot=<PrivacyCurrencyText/> subtitle=<contas> onPress` | `WidgetWide visual=<BankLogosStack/> value={hidden?'R$ ••••':totalBalance} valueMinimumFontScale=0.75 onPress` | `WidgetFull` com conteúdo §3.6 |

Se `emptyState` vier preenchido, o vazio usa `WidgetEmptyStateSlot` (`compact` no half/wide; `message 'Nenhuma conta conectada'` no full) — nunca na home (w0 §9.2). Todos com `testID="account-balance-widget"` e título `Saldo em contas`.

### 3.5 half e wide
**Valor do half**: `PrivacyCurrencyText` (w0 §11.4) com `valueCents=totalCents, hidden=useHideValues(), fontSize=20, fontWeight='800', color=useThemeColors().foreground (#fafafa), minimumFontScale=0.75, disableMinWidth, compactOnOverflow, testID='account-balance-widget-value'`. Como `animate` não é passado (default `true`), sai o `PierreAnimatedCurrencyText` nativo com `durationMs 0` (sem contagem); no overflow vira `R$ 12.346` / `R$ 1,2 mi`.
**Subtítulo do half**: `N contas conectadas` / `1 conta conectada` (Geist-Regular 14/21 #a1a1aa).
**Wide**: valor = **string do servidor** em `text-2xl font-extrabold` (21/28, Geist-ExtraBold, #fafafa); sem subtítulo; logos embaixo.

**`BankLogosStack({items, extraCount})`** (memo, L2839539) [O]:
```
View testID="account-balance-logos" style={{flexDirection:'row', alignItems:'center'}}
 ├─ items.map((it, i) =>
 │    View collapsable={false} className="bg-muted" (#27272a) testID={`account-balance-logo-${i}`}
 │         style={[{width:40, height:40, borderRadius:20}, {marginLeft: i === 0 ? 0 : -16, zIndex: items.length - i}]}
 │      └─ BankLogo bankName={it.name} imageUrl={it.imageUrl} size={40} showLoadingPlaceholder testID={`account-balance-bank-logo-${i}`})
 └─ extraCount > 0 &&
      View className="bg-muted items-center justify-center" style={[{40×40, r20}, {marginLeft:-16, zIndex:0}]}
        └─ Text allowFontScaling={false} style={{color:'#71717a', fontSize:13, fontWeight:'600'}} → `+${extraCount}`
```
Círculos de 40 sobrepostos em 16 (passo de 24 px); o primeiro fica por cima. O "+N" usa fonte do sistema peso 600 (style, não classe). BankLogo: ver §5.

**`AccountBalanceEmptyIcon`** (L2839721): `View className="items-center justify-center bg-muted" style={{width:40, height:40, borderRadius:20}}` → `PierreIcon name="cardCredit" size=22 color="#71717a"`.

### 3.6 full (conteúdo normal) [O]
```
WidgetFull title="Saldo em contas" onPress testID="account-balance-widget"
 └─ VStack className="gap-3"                                         // 10,5
     ├─ PrivacyText allowFontScaling={false} className="text-3xl font-extrabold text-foreground"
     │      testID="account-balance-widget-value" numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.75
     │      value={hidden ? 'R$ ••••' : totalBalance}                // Geist-ExtraBold 26,25/31,5 #fafafa
     ├─ HStack className="h-2 overflow-hidden rounded-full bg-muted" // altura 7, fundo #27272a
     │    └─ rows.map((r, i) => View key={`${r.name}-${i}`} style={[{height:'100%'}, {flex: Math.max(r.percentage, 2), backgroundColor: SEG[i % 3]}]})
     └─ VStack className="gap-4 pt-3"                                // gap 14, paddingTop 10,5
          └─ rows.map((r, i) =>
               HStack key={`${r.name}-${i}`} className="items-center justify-between gap-3"
                ├─ HStack className="flex-1 items-center gap-3"
                │   ├─ BankLogo bankName imageUrl size={44} showLoadingPlaceholder
                │   └─ VStack className="flex-1"
                │       ├─ Text "text-lg font-bold text-foreground" numberOfLines=1 → r.name           // Geist-Bold 15,75/24,5 #fafafa
                │       └─ Text "text-base text-muted-foreground" numberOfLines=1 → r.updatedLabel     // Geist-Regular 14/21 #a1a1aa
                └─ VStack className="items-end"
                    ├─ PrivacyText "text-base text-foreground" numberOfLines=1 value={hidden ? 'R$ ••••' : r.formattedBalance}
                    └─ Text "text-base text-muted-foreground" → `${r.percentage}%`
```
`SEG = ['#A100FF', '#19C37D', '#FF6B00']` (roxo, verde, laranja): a cor vem da **posição**, não do banco. Segmentos com `flex = max(%, 2)` (largura mínima visível). O percentual **não** é mascarado com "Ocultar dados".

**`AccountBalanceFullEmptyContent`** (L2839765), com `accountCount === 0`:
```
VStack className="gap-4"                                             // 14
 ├─ Text "text-3xl font-extrabold text-foreground" numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.75 → 'R$ 0,00'
 ├─ View className="h-2 rounded-full bg-muted"                       // barra vazia de 7
 ├─ Text "text-base text-muted-foreground" numberOfLines=1 → 'Sem conexões bancária'   (sic)
 └─ PierreButton size="xlarge" fullWidth animated={false} → 'Conectar conta'          // sem onPress [O]
```
Como o botão não tem `onPress`, o toque cai no `Pressable` do `WidgetFull` (I).

### 3.7 Strings pt-BR [O]
`Saldo em contas`, `1 conta conectada`, `{n} contas conectadas`, `Atualizado agora`, `R$ 0,00`, `Sem conexões bancária`, `Conectar conta`, `Nenhuma conta conectada`, `+{n}`, `{p}%`.

---

## 4. `cash-flow` — "Fluxo de Caixa nas Contas"

### 4.1 Identidade [O]
- id `cash-flow`, label do registro `Fluxo de Caixa nas Contas`, tamanhos `half | wide` (default do componente: `half`).
- `CashFlowWidgetConnected` (M9002, L2850970) → `CashFlowWidget` (memo, M9003, L2851512) + `MiniBar` (memo, L2851277) + `WideRow` (memo, L2851348).
- **Não usa `Widget2x2`/`WidgetWide`**: monta o próprio card com `WidgetGlassCard` + StyleSheet. Consequências: altura do wide = `minHeight 210` (não 170), padding **16** (não 14), e o toque **não tem háptico** (o `Pressable` chama `onPress` direto).

### 4.2 Connected (L2850970) [O]
Igual ao do saldo (§3.2): `isError` → `View testID="cash-flow-error" className="rounded-3xl bg-surface" style={S}` com `WidgetErrorState compact`; `isFetched && !data` → `testID="cash-flow-empty"` com `WidgetEmptyStateSlot compact`; senão `<CashFlowWidget data size onPress isLoading={isLoading || !isFetched} emptyState animate/>`.
`S = size === 'wide' ? {height:170, alignItems:'center', justifyContent:'center', padding:16} : {height:200, …}` (aqui o wide de erro/vazio tem 170).

### 4.3 Payload e cálculos [O]
`data: { income: string, expenses: string, subtitle?: string }` (strings BRL do servidor, ex.: `"R$ 5.432,10"`; `expenses` positivo).
```ts
// parse diferente do saldo: sempre remove os pontos
parse = (s) => { if (!s) return 0; const n = Number(s.replace(/[^\d,.-]/g, '').replace(/\./g, '').replace(',', '.')); return Number.isFinite(n) ? n : 0 }
income   = parse(data?.income)
expenses = parse(data?.expenses)
net      = income - expenses
period   = data?.subtitle?.trim() || 'este mês'
max      = Math.max(income, expenses)
incomePct  = (max <= 0 || income   <= 0) ? 2 : Math.max(2, Math.min(100, income   / max * 100))
expensePct = (max <= 0 || expenses <= 0) ? 2 : Math.max(2, Math.min(100, expenses / max * 100))
isEmpty  = !isLoading && income === 0 && expenses === 0
fmt      = new Intl.NumberFormat('pt-BR', {style:'currency', currency:'BRL'}).format   // usa NBSP após "R$"
```
- A barra maior sempre vai a 100%; a outra é proporcional; mínimo de **2%**.
- `isEmpty` deixa as barras "mudas" (`muted`: 100% de largura, cor `#27272A`) e esconde a linha de saída do half.
- Estado vazio com `WidgetEmptyStateSlot` só ocorre se `emptyState` vier preenchido **e** `isEmpty` (nunca na home). Sem isso, zeros renderizam o layout normal com barras mudas.

### 4.4 Constantes e estilos (L2851196+, L2852740) [O]
| Nome | Valor |
|---|---|
| `INCOME_COLOR` | `#C6FF00` (verde-limão Pierre) |
| `EXPENSE_COLOR` | `#EF4444` |
| `MUTED_COLOR` | `#27272A` |
| `S.card` | `{borderRadius:24, padding:16, overflow:'hidden'}` |
| `S.halfCard` | `{height:200}` |
| `S.wideCard` | `{minHeight:210}` |
| `S.track` | `{height:8, width:'100%', borderRadius:999, overflow:'hidden'}` (**sem cor de fundo**: o trilho é invisível) |
| `S.activeBar` | `{height:'100%', borderRadius:999}` |
| `S.divider` | `{height: StyleSheet.hairlineWidth, backgroundColor:'#27272A'}` |

O card: `WidgetGlassCard animate style={[S.card, S.halfCard | S.wideCard]}` → no Android `VStack className="bg-surface"` (#18181b). Envolvido por:
```tsx
const wrap = (node) => onPress ? <Pressable onPress={onPress} testID="cash-flow-widget-pressable">{node}</Pressable> : node
```

**MiniBar({color, width, muted})**:
```
View style={S.track}
 └─ View style={[S.activeBar, {width: muted ? '100%' : `${width}%`, backgroundColor: muted ? '#27272A' : color}]}
```
Sem animação de largura (View comum).

### 4.5 half (200 px) [O]
```
Pressable? → WidgetGlassCard style=[card, halfCard]
 └─ VStack className="flex-1 justify-between" testID="cash-flow-widget"
     ├─ VStack
     │   ├─ Text "text-base text-muted-foreground" numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.8 → 'Fluxo de caixa'   // Geist-Regular 14/21 #a1a1aa
     │   ├─ PrivacyCurrencyText valueCents={Math.round(income*100)} hidden fontSize=20 fontWeight='800' color=#fafafa
     │   │       minimumFontScale=0.6 disableMinWidth compactOnOverflow testID="cash-flow-widget-value"
     │   └─ (!isEmpty && !hidden) && PrivacyCurrencyText valueCents={expenses > 0 ? -Math.round(expenses*100) : 0} hidden={false}
     │           fontSize=16 fontWeight='700' color='#EF4444' disableMinWidth testID="cash-flow-widget-expense-value"
     └─ VStack className="gap-2"                         // 7
         ├─ MiniBar color=#C6FF00 width=incomePct muted=isEmpty
         └─ MiniBar color=#EF4444 width=expensePct muted=isEmpty
```
- O valor grande do half é a **entrada** (não o saldo líquido); embaixo, em vermelho, a saída com sinal: `-R$ 1.234,56` (formatação nativa, w0 §11.3).
- Com "Ocultar dados" a linha vermelha some (não é mascarada) e o valor principal vira `R$ ••••`.
- Sem chevron no half.

### 4.6 wide (largura total, minHeight 210) [O]
```
Pressable? → WidgetGlassCard style=[card, wideCard]
 └─ VStack className="flex-1 justify-between gap-3" testID="cash-flow-widget"          // gap 10,5
     ├─ HStack className="items-center justify-between"
     │   ├─ Text "text-base text-muted-foreground flex-1 shrink min-w-0 mr-2" numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.65
     │   │      → `Fluxo de Caixa nas Contas • ${period}`                                  // ex.: "… • este mês"
     │   └─ Box testID="cash-flow-widget-chevron" className="p-0.5" → PierreIcon chevronRight 16 #71717a
     ├─ WideRow label='Entrada' value={fmt(Math.abs(income))} color=#C6FF00 width=incomePct hidden muted=isEmpty
     ├─ WideRow label='Saída'   value={expenses > 0 ? '-' + fmt(Math.abs(expenses)) : fmt(0)} color=#EF4444 width=expensePct
     │          hidden muted=isEmpty valueClassName='text-[#EF4444]'
     ├─ View style={S.divider}                                                              // hairline #27272A
     └─ HStack className="items-center justify-between"
         ├─ Text "text-lg font-semibold text-foreground shrink min-w-0 mr-3" numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.7 → 'Fluxo de caixa'
         └─ PrivacyText className={'text-base shrink min-w-0 ' + (net < 0 ? 'text-[#EF4444]' : 'text-foreground')}
                numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.6
                value={hidden ? 'R$ ••••' : (net < 0 ? '-' : '') + fmt(Math.abs(net))}         // positivo sem "+"
```
**WideRow({label, value, color, width, hidden, muted, valueClassName = 'text-foreground'})**:
```
VStack className="gap-1"                                         // 3,5
 ├─ HStack className="items-center justify-between"
 │   ├─ Text "text-lg font-semibold text-foreground shrink min-w-0 mr-3" numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.7 → label   // Geist-SemiBold 15,75/24,5 #fafafa
 │   └─ PrivacyText className={'text-base shrink min-w-0 ' + valueClassName} numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.6
 │          value={hidden ? 'R$ ••••' : value}                                                                                          // Geist-Regular 14/21
 └─ MiniBar color width muted
```
Altura aproximada do conteúdo: 24,5 + 10,5 + (24,5+3,5+8) ×2 + 10,5 + hairline + 10,5 + 24,5 ≈ 157 + padding 32 ≈ 189 → o `minHeight 210` domina e o `justify-between` distribui o resto [I: cálculo].

### 4.7 Carregando [O]
`WidgetGlassCard style=[card, half|wide]` → `View testID="cash-flow-widget"` → `WidgetLoadingState variant="content"` (o wrapper Pressable continua).

### 4.8 Strings pt-BR [O]
`Fluxo de caixa`, `Fluxo de Caixa nas Contas • `, `este mês`, `Entrada`, `Saída`, `R$ ••••`.

### 4.9 Snippet TSX (half + wide)
```tsx
import React from 'react';
import { Pressable, StyleSheet, Text, View } from 'react-native';
import { ChevronRight } from 'lucide-react-native';
// PrivacyText, PrivacyCurrencyText, HIDDEN_VALUE, useHideValues: ver w0-base.md §10–11, §15

const INCOME = '#C6FF00', EXPENSE = '#EF4444', MUTED = '#27272A';
const fmt = new Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' }).format;
const parse = (s?: string) => { if (!s) return 0; const n = Number(s.replace(/[^\d,.-]/g, '').replace(/\./g, '').replace(',', '.')); return Number.isFinite(n) ? n : 0; };
const pct = (v: number, max: number) => (max <= 0 || v <= 0 ? 2 : Math.max(2, Math.min(100, (v / max) * 100)));

const MiniBar = ({ color, width, muted }: { color: string; width: number; muted: boolean }) => (
  <View style={s.track}>
    <View style={[s.bar, { width: muted ? '100%' : `${width}%`, backgroundColor: muted ? MUTED : color }]} />
  </View>
);

const WideRow = ({ label, value, color, width, hidden, muted, valueColor = '#fafafa' }: any) => (
  <View style={{ rowGap: 3.5 }}>
    <View style={s.rowBetween}>
      <Text allowFontScaling={false} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.7}
        style={[s.lg, { fontFamily: 'Geist-SemiBold', color: '#fafafa', flexShrink: 1, minWidth: 0, marginRight: 10.5 }]}>{label}</Text>
      <PrivacyText value={hidden ? HIDDEN_VALUE : value} allowFontScaling={false} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.6}
        style={[s.base, { color: valueColor, flexShrink: 1, minWidth: 0 }]} />
    </View>
    <MiniBar color={color} width={width} muted={muted} />
  </View>
);

export function CashFlowWidget({ data, size = 'half', onPress, isLoading }: {
  data?: { income?: string; expenses?: string; subtitle?: string }; size?: 'half' | 'wide'; onPress?: () => void; isLoading?: boolean;
}) {
  const hidden = useHideValues();
  const income = parse(data?.income), expenses = parse(data?.expenses), net = income - expenses;
  const period = data?.subtitle?.trim() || 'este mês';
  const max = Math.max(income, expenses);
  const inc = pct(income, max), exp = pct(expenses, max);
  const empty = !isLoading && income === 0 && expenses === 0;
  const cardStyle = [s.card, size === 'wide' ? { minHeight: 210 } : { height: 200 }];
  const wrap = (n: React.ReactNode) => (onPress ? <Pressable onPress={onPress} testID="cash-flow-widget-pressable">{n}</Pressable> : n);

  if (isLoading) return wrap(<View style={cardStyle}>{/* <WidgetLoadingState variant="content"/> */}</View>);

  if (size === 'wide') return wrap(
    <View style={cardStyle}>
      <View style={{ flex: 1, justifyContent: 'space-between', rowGap: 10.5 }} testID="cash-flow-widget">
        <View style={s.rowBetween}>
          <Text allowFontScaling={false} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.65}
            style={[s.base, { fontFamily: 'Geist-Regular', color: '#a1a1aa', flex: 1, flexShrink: 1, minWidth: 0, marginRight: 7 }]}>
            {`Fluxo de Caixa nas Contas • ${period}`}
          </Text>
          <View style={{ padding: 1.75 }}><ChevronRight size={16} color="#71717a" /></View>
        </View>
        <WideRow label="Entrada" value={fmt(Math.abs(income))} color={INCOME} width={inc} hidden={hidden} muted={empty} />
        <WideRow label="Saída" value={expenses > 0 ? '-' + fmt(Math.abs(expenses)) : fmt(0)} color={EXPENSE} width={exp}
          hidden={hidden} muted={empty} valueColor={EXPENSE} />
        <View style={{ height: StyleSheet.hairlineWidth, backgroundColor: MUTED }} />
        <View style={s.rowBetween}>
          <Text allowFontScaling={false} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.7}
            style={[s.lg, { fontFamily: 'Geist-SemiBold', color: '#fafafa', flexShrink: 1, minWidth: 0, marginRight: 10.5 }]}>Fluxo de caixa</Text>
          <PrivacyText value={hidden ? HIDDEN_VALUE : (net < 0 ? '-' : '') + fmt(Math.abs(net))} allowFontScaling={false}
            numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.6}
            style={[s.base, { color: net < 0 ? EXPENSE : '#fafafa', flexShrink: 1, minWidth: 0 }]} />
        </View>
      </View>
    </View>,
  );

  return wrap(
    <View style={cardStyle}>
      <View style={{ flex: 1, justifyContent: 'space-between' }} testID="cash-flow-widget">
        <View>
          <Text allowFontScaling={false} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.8}
            style={[s.base, { fontFamily: 'Geist-Regular', color: '#a1a1aa' }]}>Fluxo de caixa</Text>
          <PrivacyCurrencyText valueCents={Math.round(income * 100)} hidden={hidden} fontSize={20} fontWeight="800" color="#fafafa"
            minimumFontScale={0.6} disableMinWidth compactOnOverflow testID="cash-flow-widget-value" />
          {!empty && !hidden && (
            <PrivacyCurrencyText valueCents={expenses > 0 ? -Math.round(expenses * 100) : 0} hidden={false} fontSize={16}
              fontWeight="700" color={EXPENSE} disableMinWidth testID="cash-flow-widget-expense-value" />
          )}
        </View>
        <View style={{ rowGap: 7 }}>
          <MiniBar color={INCOME} width={inc} muted={empty} />
          <MiniBar color={EXPENSE} width={exp} muted={empty} />
        </View>
      </View>
    </View>,
  );
}

const s = StyleSheet.create({
  card: { borderRadius: 24, padding: 16, overflow: 'hidden', backgroundColor: '#18181b' },
  track: { height: 8, width: '100%', borderRadius: 999, overflow: 'hidden' },
  bar: { height: '100%', borderRadius: 999 },
  rowBetween: { flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between' },
  base: { fontSize: 14, lineHeight: 21 }, lg: { fontSize: 15.75, lineHeight: 24.5 },
});
```

---

## 5. BankLogo e cor de banco (compartilhados)

### 5.1 `BankLogo` (M3634, L1330882; reexportado por M3633) [O]
Props: `bankName, imageUrl, transferNumber, fallbackNames, size (default 40 [I alto]), testID, showLoadingPlaceholder = false`. Raio = `size / 2` (sempre círculo).

**Ordem de resolução da imagem** (a logo **local** vence a URL do servidor):
1. `key = normalizeBankName(bankName)` → se `hasRasterBankLogo(key)` → PNG local (`getRasterBankLogo`, os `assets_images_banks_*` / `*_raster` de `logos/bancos/png/`).
2. Senão, se `hasBankSvg(bankName)` → componente SVG do `bankRegistry` (`getBankSvg`, os 50 SVG de `logos/bancos/svg/`).
3. Senão, `k = resolveBankKeyFromName(bankName, ...fallbackNames)` → raster de `k` ou `bankRegistry[k]`.
4. Senão, se houver `transferNumber`: `k = BANK_CODE_TO_REGISTRY_KEY[transferNumber.match(/^(\d{3})/)[1]]` (código COMPE: `'001'→banco-do-brasil`, `'033'→santander`, `104→caixa`, `237→bradesco`, `341→itaú`…) → raster/SVG.
5. Senão, se `imageUrl` e ainda não deu erro: termina em `.svg` → `SvgUri uri width=size height=size onLoad onError`; senão `Image source={{uri}} resizeMode="cover" style={{width:size, height:size, borderRadius:size/2}}` com `onLoadStart/onLoadEnd/onError`. Com `showLoadingPlaceholder`, enquanto carrega aparece `View pointerEvents="none" style={[absoluteFill, {backgroundColor:'#27272a'}, {width:size, height:size, borderRadius}]} testID="{testID}-loading"`.
6. Fallback final: `PierreIcon name="buildingsBank" size=16 color=mutedForeground (#a1a1aa)` sobre fundo `#27272a`.

Render (função `t9`):
```
View collapsable={false} style={[{width:size, height:size, borderRadius:size/2, justifyContent:'center', alignItems:'center'}, fallback && {backgroundColor:'#27272a'}]}
     testID accessibilityLabel={fallback ? bankName : undefined}
 └─ View collapsable={false} style={{width:size, height:size, borderRadius:size/2, overflow:'hidden', justifyContent:'center', alignItems:'center'}} testID="{testID}-clip"
     └─ raster: Image resizeMode="contain" accessible accessibilityLabel={bankName} style={{width:size, height:size, borderRadius}}
        svg:    <Svg width={size} height={size} style={{maxWidth:size, maxHeight:size, borderRadius}}/>
        url:    SvgUri | Image (cover) + placeholder
```
`resolveBankKeyFromName(...names)`: para cada nome não vazio (minúsculo), percorre `BANK_KEYWORD_TO_REGISTRY_KEY` (ex.: `'carteira pierre'→'carteira-pierre'`, `'pierre'→'cartao-pierre'`, `'nubank'→'nubank'`…) **ordenado pela palavra-chave mais longa primeiro** e aceita a primeira que aparece como palavra inteira (`matchesBankKey`: antes e depois da ocorrência não pode haver letra `[a-záàâãéèêíïóôõúüç]`). Os dados das palavras-chave já estão em `logos/mapeamento.json` e no design system.

### 5.2 `resolveBankColor(name, ...fallbacks)` (M3697, L1342727) [O]
```ts
DEFAULT_BANK_COLOR = '#27272A'; BANK_COLOR_FOREGROUND = '#FFFFFF'
normalizeBankToken = s => s.normalize('NFD').replace(/[̀-ͯ]/g, '').toLowerCase().replace(/[^a-z0-9]+/g, ' ').trim()
matchesNormalizedToken = (text, token) => ` ${text} `.includes(` ${token} `)      // palavra inteira
resolveBankVisual(name, ...fallbacks) =
  1º: para cada candidato normalizado, procura no índice de tokens do BANK_VISUAL_CATALOG (slug/nome/aliases) o primeiro que casa;
  2º: senão, resolveBankKeyFromName(name, ...fallbacks) → mapa registryKey → visual;
  senão null
resolveBankColor = (...a) => resolveBankVisual(...a)?.color ?? '#27272A'
```
`BANK_VISUAL_CATALOG` tem 58 entradas `{slug, name, color, aliases, logoFileNames, registryKeys}`; exemplos: `carteira-pierre #27272A`, `cartao-pierre #CFFF04` (alias `pierre one`), `infinite-pay #171527`, `banco-do-brasil #FDFC30` (`bb`, `ourocard`), `banco-inter #EA7100`, `bmg #FA6300`, `99-pay #FFDD00`, `bradesco #CC092F` (`bradescard`), `next #03FF60`, `itau #FF6200` (`itaucard`, `personalite`), `santander #EC0000` (`van gogh`), `nubank #820AD1` (`nu`, `ultravioleta`, `roxinho`…). A tabela completa de cores já está no design system (seção de paletas de bancos).
Na carteira, a cor do banco pinta a faixa do cartão e o texto `•••• 1234` é sempre branco (`BANK_COLOR_FOREGROUND`), sem checar contraste (com o BB amarelo #FDFC30 o branco fica com pouco contraste) [O cor; I contraste].

---

## 6. Dúvidas abertas / pontos não verificados
1. `WALLET_STACK_TRAVEL = 0.9`: o valor vem da ordem de atribuição de registradores no módulo de constantes (`slot34 = r0`, com o último `r0 = 0.9`). É coerente com o efeito (painel vizinho a 90% da altura), mas pode ser 20 se o decompilador reordenou [I médio].
2. `mergePierreCardIntoWalletWidget` (M9000): não detalhei como a fatura Pierre One entra em `cards`/`totalInvoiceAmount`. Pelo nome, ela soma o cartão Pierre ao agregado.
3. Valores de `SKINS.PIERRE_ONE_DEFAULT` (imagem da arte, `frontTextColor`, `labelColor`, `overlayColor`) e o conteúdo de `skin.background` do cartão ativo ficam no módulo de skins do cartão, que não foi aberto aqui.
4. `PierreButton` (variantes `primary/destructive`, tamanhos `small/regular/xlarge`) e `PierreIcon` (`calcAdd`, `cardCredit`, `buildingsBank`, `chevronRight`) não foram dimensionados aqui; ver design system.
5. `PierreDonutChart` é nativo (M4584, `io.cloudwalk.pierre.charts`); a duração e o easing da animação do anel não foram lidos no dex.
6. `BankLogo` size default: o literal 40 aparece no módulo, mas o registrador do default é ambíguo na decompilação [I alto].
7. No `AccountBalanceFullEmptyContent`, o botão `Conectar conta` não tem `onPress`; o toque cai no `Pressable` do `WidgetFull` (navegação do registro) [I].
8. A "carteira" **não** tem chip EMV, bandeira (Visa/Master) nem gradiente de cartão desenhado: o cartão Pierre usa a **imagem** da skin (topo recortado) e os cartões de outros bancos são faixas de cor sólida com logo. As bandeiras de `logos/bandeiras-cartao/` não são usadas por este widget [O: ausência nos módulos M9073–M9091].
