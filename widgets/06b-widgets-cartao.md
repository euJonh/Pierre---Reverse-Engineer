# W6b — Widgets de sistema da home do cartão (Pierre v4.0.6)

> Análise estática de `_trabalho/out.js`. **[O]** = observado (`Lnnnn` = linha do out.js, `Mnnnn` = módulo Metro); **[I]** = inferido.
> Base compartilhada (não repetida aqui): `widgets/00-base.md` (`Widget2x2` §5, `WidgetValueSlot` §6, `WidgetWide` §7, `WidgetFull` §8, estados §9, "Ocultar dados" §10, `PrivacyCurrencyText` §11.4, utilitários de moeda §12, tokens §13, NativeWind §14).
> IA na home do cartão: `widgets/05b-leo-ia.md` §5 (`AIWidgetRenderer`) e §7 (`WidgetSlotRenderer`, `LiveWidgetRenderer`, `AIWidgetWrapper`, `PendingSyncOverlay`).
> Registro (`WIDGET_REGISTRY`), tela, grid e `WidgetCarousel` ficam com outro agente: aqui só aparecem quando mudam o comportamento de um widget.
> Unidades: px = dp. Tudo em dark (`LIGHT_MODE_DISABLED = true`).

---

## 0. Mapa rápido [O]

O `WidgetRenderer` do cartão (M9524, L2982681) despacha **por id**. Só **dois** widgets de sistema são exclusivos da home do cartão; os outros sete **reaproveitam os componentes da Vision**, mas com dados montados no app a partir da API de crédito (não do endpoint `/api/mobile/vision/widgets/{id}/data`).

| id | Componente | Módulo / linha | Tamanhos que o componente trata | Documentado em |
|---|---|---|---|---|
| `cashback` | `CashbackWidget` (memo) | M9532, L2985666 | half (`Widget2x2`) / wide (`WidgetWide`) | **§2** |
| `invoice` | `InvoiceWidget` (memo) | M9534, L2985949 | só `Widget2x2` (ignora `size`) | **§3** |
| `available-limit` | `AvailableLimitWidget` | M9007, L2853333 | half/wide | `03-…` §1 + §6.1 aqui |
| `monthly-spending` | `VisionMonthlySpendingWidget` | M8995, L2843453 | half/wide/full | `02-gastos.md` §2 + §6.2 aqui |
| `recent-transactions` | `VisionRecentTransactionsWidget` | M9031, L2861750 | half/wide/full | `03-…` + §6.3 aqui |
| `spending-by-category` | `VisionSpendingByCategoryWidget` (o **da Vision**, sem a camada Connected) | M9035, L2864048 | half/wide/full | `02-gastos.md` §4 + **§5** aqui |
| `installments` | `InstallmentsWidget` / `InstallmentsWidgetFull` | M9016, L2855251 / L2856127 | half/wide → `InstallmentsWidget`; full → `…Full` | `03-…` + §6.4 aqui |
| `subscriptions` | `SubscriptionsWidget` / `SubscriptionsWidgetFull` (via M9535 → M9041) | L2866635 / L2867690 | full → `…Full`; resto → `SubscriptionsWidget` | `04-assinaturas.md` + §6.5 aqui |
| `cdb-position` | `CdbPositionWidget` | M9058, L2873385 | `size === 'half' ? 'half' : 'full'` | `05a-cdb.md` §3 + §6.6 aqui |
| (fora do grid) | `CardWidget` (arte do cartão + ações) | M9604, L3008528 | — | **§4** |
| (código morto) | `SpendingByCategoryWidget` / `…Full` / `CategoryBar` do cartão | M9763/M9765/M9764, L3078245–3079300 | — | §5.4 |

Ids fora dessa lista (ex.: `ai-<uuid>`) vão pelo ramo `'ai'` do `WidgetSlotRenderer` (05b §7). Um id de sistema desconhecido resulta em `null` [O L2982681, ramo final `r10 = null`].

---

## 1. O que o `WidgetRenderer` do cartão injeta em todo widget [O L2982681–2983800]

Só o necessário para entender os widgets; o resto é do agente da tela/grid.

- Entradas: `{widget: {id, size, status, data}, isLoading, emptyStates}`.
- `onPress` passado ao widget = **`undefined` em modo de edição** (`useWidgetEditMode()`), senão o handler de toque. Em edição, portanto, nenhum widget é clicável e o chevron continua visível (o chevron do `Widget2x2` depende de `showChevron`, não de `onPress`).
- `emptyState` = `emptyStates?.[widget.id]` (pode ser `undefined` → `WidgetEmptyStateSlot` cai no padrão "Sem dados disponíveis", 00-base §9.1/9.2).
- Toque (`t3`): lê `catalogEntry = useWidgetCatalog().systemCatalog.get(id)`; `tapAction ∈ {'navigate','navigate-expand','expand','none'}`.
  - `navigate` → `navigateToWidgetDestination(router, resolveCardWidgetTapHref(id, catalogEntry))`.
  - `navigate-expand`/`expand` → `ref.measureInWindow((x,y,w,h) => router.push({pathname: entry.navigateTo, params: {...entry.navigateParams, originX, originY, originW, originH}}))`, cada origem como `String(Math.round(v))`.
- `resolveCardWidgetTapHref(id, entry)` (M9529, L2985345) [O]:

| id | Destino |
|---|---|
| `cdb-position` | `{pathname: CDB_INVESTMENT_DETAILS_ROUTE, params: {from:'card_home', productId:'pierre-one'}}` (rota `/(protected)/investments/cdb-investment-details`, ver 05a) |
| `cashback` | `{pathname:'/card/cashback-history', params:{from:'card_home'}}` |
| `invoice` | `{pathname:'/card/invoice-stack', params:{focus:'first-open'}}` (`INVOICE_FOCUS_PARAM='focus'`, `INVOICE_FOCUS_FIRST_OPEN='first-open'`, M8691 L2733087) |
| outro | `entry?.navigateTo ?? null` |

- **Estados vindos do `LiveWidgetRenderer`** (05b §7.2): `status === 'error'` ou `'empty'` **substituem o widget inteiro** (o componente nem é montado):
  - full: `WidgetFull testID="{id}-error|{id}-empty" title="" showChevron={false}` → `WidgetErrorState` / `WidgetEmptyStateSlot {emptyState}`.
  - half/wide: `View className="rounded-3xl bg-surface" testID="{id}-error|{id}-empty" style={{height:200, alignItems:'center', justifyContent:'center', padding:16}}` → `WidgetErrorState compact` / `WidgetEmptyStateSlot {emptyState, compact:true}`. **Atenção:** a altura é 200 também no wide (o wide normal tem 170) [O: um único estilo `halfContainer`]. `rounded-3xl` = **21** px pela config compilada (00-base §14.5), não 24 como nos cards normais; `bg-surface` = #18181B.
  - Textos: "Falha ao sincronizar dados" (ícone `feedbackWarning` #EF4444) e "Sem dados disponíveis" (ícone `wifiOff` #71717A).
- Resultado embrulhado em `View ref={measureRef}` (para o `measureInWindow`).

### 1.1 Fonte de dados comum: `useWidgetLiveData(ids)` (M9620, `parcial/w6/livedata.js`) [O]
Cada widget presente no layout liga só as queries de que precisa (`enabled` = `ids.has(id)`), transforma o resultado e grava no store `useWidgetLiveDataStore` com `setEntry(id, {data, isLoading, isError, isFetched})` (lote: `setEntries`). Transformações em M9621 (L≈2984000; `livedata.js` 1124–2130).

| Query (M4077) | Endpoint (`creditRead`, GET) | queryKey (`cardKeys`, L1649649) | staleTime / gcTime | Widgets |
|---|---|---|---|---|
| `useCashbackSummary` (L1647310) | `/credit/cashback/summary` + header `x-product: pierre` | `cashbackSummary` | 0 / 600 000 | cashback |
| `useCurrentInvoice` (L1647923) | `/credit/invoices/current?product=pierre` | `invoiceCurrent` | 0 / 600 000 | invoice, installments |
| `useInvoices` (L1647808) | `/credit/invoices?product=pierre` | `invoices` | 0 / 1 800 000 | invoice |
| `useCreditAvailable` | `/credit/limit` [I pela lista de rotas L1659268] | `available` | — | available-limit |
| `useStatements({startDate,endDate,limit:100})` | `/credit/statements/v2` [I idem] | `statements` | — | monthly-spending, recent-transactions, spending-by-category, subscriptions |
| `useInvoiceItems(currentInvoice.id)` | `/credit/invoices/items` [I idem] | `invoiceItems` | — | installments |
| `useVisionSystemWidgetData('cdb-position','full', CDB_POSITION_DAILY_LIQUIDITY_VARIANT, {directFetcher: fetchCdbDailyLiquidityPositionDirect, accessGate: useCdbWidgetIdentityGate()})` | API de CDB (05a) | — | — | cdb-position |

- Todas as queries de crédito exigem `isCorePaymentApiConfigured() && creditAccountAllowsRequest(...)` além do `enabled` [O L1647310].
- O período de `useStatements` é `currentCalendarMonthRange()` (mês civil corrente) [O].
- `getCurrentInvoiceDirect` aceita resposta objeto ou array; escolhe a fatura com `status === 'open'`, senão a primeira; sem nada → `null` [O L1661845–1661990].

---

## 2. `cashback` — "Cashback" (`CashbackWidget`, M9532, L2985666)

### 2.1 Identidade [O]
- id `cashback`; tamanhos tratados: **half** e **wide** (qualquer valor ≠ `'wide'` cai no half; default `'half'`).
- Tap: `/card/cashback-history?from=card_home` (§1). O renderer **não** passa `emptyState` para este widget, e o componente não tem ramo vazio: com cashback zerado mostra "R$ 0,00".
- testID: `cashback-widget` (+ `-value`, `-subtitle`, `-chevron` herdados do Widget2x2/WidgetWide) e `cashback-widget-chip`.

### 2.2 Dados e lógica [O]
Fonte: `GET /credit/cashback/summary` → `{availableCents?, pendingCents?, projectedCents?}` (campos lidos pelo app).
```ts
// M9622 (L≈2984060) — null/undefined contam como 0
pendingTotal = (s.pendingCents ?? 0) + (s.projectedCents ?? 0)
total        = (s.availableCents ?? 0) + pendingTotal
// transformCashback (M9621)
data = {
  amount:        formatCentsToBRL(total),         // M9001: (c/100).toLocaleString('pt-BR',{style:'currency',currency:'BRL'}) → "R$ 1.234,56"
  pendingAmount: formatCentsToBRL(pendingTotal),
  availableCents: s.availableCents, pendingCents: s.pendingCents,
}
```
- Ou seja: o **valor grande é o total** (disponível + pendente + projetado), e o subtítulo mostra o pendente somado ao projetado.
- No widget:
```ts
const hidden   = useHideValues();
const value    = hidden ? HIDDEN_VALUE : amount;          // 'R$ ••••'
const subtitle = hidden ? '' : `${pendingAmount} pendentes`;
```
  Com "Ocultar dados", o subtítulo **some** (string vazia; no half a linha continua ocupando altura, no wide não renderiza porque `subtitle` é falsy). O valor é texto pronto (`value`), **não** `PrivacyCurrencyText`: não há contagem nem compactação; o fade de 150 ms vem do `PrivacyText` (00-base §10.2).
- `availableCents`/`pendingCents` chegam como props, mas o widget não os usa [O].

### 2.3 Layout [O]
Visual comum, `CashbackChip`:
```
Box className="h-10 w-10 items-center justify-center rounded-full" style={{backgroundColor:'#243400'}} testID="cashback-widget-chip"
 └─ PierreIcon name="cwProductsAward" size={24} color="#CFFF04"
```
- `h-10 w-10` = **35×35 px** (spacing = n × 3,5, 00-base §14.4); círculo (`rounded-full`); fundo `#243400` (= token `successForeground`, verde-oliva escuro); ícone 24 px `#CFFF04` (marca), centralizado.

| Tamanho | Árvore |
|---|---|
| **half** | `Widget2x2 testID="cashback-widget" visual={<CashbackChip/>} label="Cashback" value={value} subtitle={subtitle} onPress` (visualPlacement padrão `'top'`, chevron padrão `true`, bg padrão → caminho opaco no Android: `bg-surface` #18181B, 200 px, raio 24, p-4) |
| **wide** | `WidgetWide testID="cashback-widget" label="Cashback" value subtitle visual={<CashbackChip/>} onPress` (170 px; textos em cima, chip **embaixo** à esquerda) |

### 2.4 Tipografia (herdada, 00-base §6/§7)
| Texto | half | wide |
|---|---|---|
| "Cashback" | `text-base` Geist-Regular 14/21 #A1A1AA | `text-lg` Geist-Regular 15,75/24,5 #A1A1AA |
| valor "R$ 12,34" | `text-xl font-extrabold` Geist-ExtraBold 17,5/24,5 #FAFAFA (autoshrink até 0,6) | `text-2xl font-extrabold` 21/28 #FAFAFA (até 0,75) |
| "R$ 3,21 pendentes" | `text-base` 14/21 #A1A1AA | idem |

### 2.5 Carregando [O]
- half: `Widget2x2 testID="cashback-widget" label="Cashback" value="" subtitle="" visual={<WidgetLoadingState variant="visual"/>}` (círculo 64×64 #71717A pulsando 0,3↔0,7), **sem** `onPress`.
- wide: `WidgetWide testID="cashback-widget" label="Cashback" value="" visual={<WidgetLoadingState variant="content"/>}` (três barras 44 px).

### 2.6 Animações / hápticos
Nenhuma própria. Toque = háptico Light do `WidgetShell` (00-base §4.2).

### 2.7 Strings
`Cashback` · `{valor} pendentes`. Na tela de destino (M9530, `CASHBACK_HISTORY_COPY`): `Cashback Total`, `Disponível:`, `Pendente:`, `Usar cashback na fatura`, `Sobre o cashback pendente`, `Seu cashback pendente é liberado no pagamento da sua fatura em aber…` (tela fora do escopo).

---

## 3. `invoice` — "Fatura de {mês}" (`InvoiceWidget`, M9534, L2985949)

### 3.1 Identidade [O]
- id `invoice`. O componente **ignora `size`**: sempre `Widget2x2` (200 px). Se o registro permitir wide/full, o card fica com o layout half esticado na largura [O: não há leitura de `size`].
- Tap: `/card/invoice-stack?focus=first-open` (§1).
- testIDs: `invoice-widget`, `invoice-widget-value`, `invoice-slot-empty-{n}`.

### 3.2 Dados: `transformInvoice(current, invoices)` (M9621) [O]
Entrada: `current` = fatura aberta (`useCurrentInvoice`), `invoices` = lista (`useInvoices`, `[]` se null). Só roda quando **as duas** respostas existem; `isLoading = a||b`, `isError = a||b`, `isFetched = a&&b`.
```ts
const MONTHS = ['Jan','Fev','Mar','Abr','Mai','Jun','Jul','Ago','Set','Out','Nov','Dez'];
const last5 = invoices.slice().sort((a,b) => a.referenceMonth.localeCompare(b.referenceMonth)).slice(-5);
const bars = last5.length > 0
  ? last5.map((inv, i) => ({ value: inv.grossTotalCents ?? inv.totalAmountCents, isCurrent: i === last5.length - 1 }))
  : [{ value: 0, isCurrent: true }];
if (!current) return { monthLabel: MONTHS[new Date().getMonth()] ?? 'Jan', amount: 'R$ 0,00', closingDate: '', bars };
const m = current.referenceMonth ? Number(current.referenceMonth.slice(5,7)) - 1 : new Date().getMonth();
return {
  monthLabel: MONTHS[m] ?? 'Jan',                         // "Out"
  amount: current.totalAmount,                            // string BRL pronta do servidor [I: comparada com 'R$ 0,00' e passada a parseBRLToCents]
  closingDate: current.cycleEndDate ? formatDayMonth(current.cycleEndDate) : '',   // 'YYYY-MM-DD' → 'DD/MM'
  bars,
};
```
- "Barra atual" = a **última** fatura por `referenceMonth` (YYYY-MM), não necessariamente a `current`.

### 3.3 Lógica no widget [O]
```ts
const hidden = useHideValues();
const { foreground } = useThemeColors();                       // #FAFAFA
const bars0 = bars?.length ? bars : [{ value: 0, isCurrent: true }];
const isEmpty = !isLoading && bars0.every(b => b.value === 0) && (!amount || amount === 'R$ 0,00');
const max = Math.max(...bars0.map(b => b.value), 1);          // mínimo 1 → evita divisão por 0
// slots: sempre 5
const recent = bars0.slice(-5);
const slots = [
  ...recent.map(b => b.value > 0 ? { kind:'bar', value:b.value, isCurrent:b.isCurrent } : { kind:'empty' }),
  ...Array.from({ length: 5 - recent.length }, () => ({ kind:'empty' })),   // vazios completam À DIREITA
];
heightPct(b) = Math.max(b.value / max * 100, 5) + '%';         // mínimo 5%
marginLeft(i) = i > 0 ? 6 : 0;
```

### 3.4 Layout e geometria do gráfico [O, StyleSheet L2986330]
```
Widget2x2 testID="invoice-widget"
  label   = `Fatura de ${monthLabel}`                          // "Fatura de Out"
  visual  = <View style={row}>{slots}</View>
  valueSlot = <PrivacyCurrencyText animate={false} valueCents={parseBRLToCents(amount)} hidden={hidden}
               fontSize={20} fontWeight="800" color={foreground /* #FAFAFA */} disableMinWidth testID="invoice-widget-value"/>
  subtitle = closingDate ? `Fecha em ${closingDate}` : 'Sem data de fechamento'
  onPress
```
- `visualPlacement` = `'top'` (padrão): o gráfico fica no topo à esquerda, ao lado do chevron.
- Estilos:

| Chave | Estilo | Uso |
|---|---|---|
| `row` | `{flexDirection:'row', justifyContent:'flex-start', alignItems:'flex-end', height:80}` | contêiner das 5 posições (largura = soma: 5×10 + 4×6 = **74 px**, se nenhuma for o ponto de 12) |
| `bar` (não atual) | `{width:10, borderRadius:5}` + `{height:'{pct}%', backgroundColor:'#27272A', marginLeft}` | barra cinza (zinc-800) |
| `track` (atual) | `{width:10, height:'100%', borderRadius:5, justifyContent:'flex-end', overflow:'hidden'}` + `{backgroundColor:'#27272A', marginLeft}` | trilho de 80 px |
| `fill` (atual) | `{width:'100%', borderRadius:5}` + `{height:'{pct}%', backgroundColor:'#CFFF04'}` | preenchimento verde-limão de baixo para cima |
| `emptySlot` | `{width:12, height:12, borderRadius:6, backgroundColor:'#27272A'}` + `{marginLeft}` | **ponto** de 12 px para mês sem valor / posição faltante |

- Resultado visual: até 5 colunas alinhadas pela base, 80 px de altura; meses passados em `#27272A`; o mês "atual" tem trilho cinza cheio com preenchimento `#CFFF04` na altura proporcional; posições sem valor viram bolinhas de 12 px.
- Sem animação de altura (Views estáticas).

### 3.5 Tipografia
| Texto | Classe/estilo | Fonte | Size/LH | Cor |
|---|---|---|---|---|
| "Fatura de Out" | `text-base text-muted-foreground` (label do Widget2x2) | Geist-Regular | 14/21 | #A1A1AA |
| valor | `PrivacyCurrencyText` 20 / '800' | Geist-ExtraBold | 20 | #FAFAFA |
| "Fecha em 10/11" / "Sem data de fechamento" | `text-base text-muted-foreground` | Geist-Regular | 14/21 | #A1A1AA |

"Ocultar dados": o valor vira `R$ ••••` (via `hidden`, 00-base §11.4); label, subtítulo e barras continuam visíveis [O].

### 3.6 Estados [O]
- **Carregando** (`isLoading`): `Widget2x2 testID="invoice-widget" label={`Fatura de ${monthLabel ?? ''}`} value="" subtitle="" visual={<WidgetLoadingState variant="visual"/>}` (sem `onPress`).
- **Vazio** (`isEmpty`): `Widget2x2 testID="invoice-widget" label={`Fatura de ${monthLabel ?? ''}`} value="—" subtitle="" visual={<WidgetEmptyStateSlot emptyState compact/>} onPress` → ícone `wifiOff` 18 + "Sem dados disponíveis" (16 px) no topo, valor "—" (travessão U+2014). Continua clicável.
- **Erro**: tratado antes, pelo renderer (§1).

### 3.7 Strings
`Fatura de {Mmm}` · meses `Jan Fev Mar Abr Mai Jun Jul Ago Set Out Nov Dez` · `Fecha em {DD/MM}` · `Sem data de fechamento` · `—` · `R$ 0,00`.

---

## 4. `CardWidget` — arte do cartão e ações (M9604, L3008528)

### 4.1 Onde fica [O]
- **Não** é um widget do grid: não está no `WidgetRenderer` nem no `WIDGET_REGISTRY`. É o cabeçalho fixo da home do cartão, montado por `CardWithCollapse` (M9602, L3008204) a partir da tela (L2908156).
- Props que a tela passa: `{isCollapsed (SharedValue 0..1), isLoading, lastFourDigits, skin, isVisible: true, isLocked, isLockLoading, onLock, onWallet, onBenefits, onLimits, onSettings, onDetails}`.
- `CardWithCollapse`: `collapsed = isCollapsed.value < 0.5` (via `useAnimatedReaction` + `runOnJS(setCollapsed)`); `onExpandRequest = () => isCollapsed.value = withTiming(0 [I: constante não materializada], {duration: 250})`; `initialCollapsed` = valor da 1ª renderização (ref).
- `CardWidget` separa `{onDetails, onLock, onWallet, onBenefits, onLimits, onSettings, isLocked=false, isLockLoading=false}` e repassa **o resto** (`lastFourDigits`, `skin`, `collapsed`, `initialCollapsed`, `isVisible`, `onExpandRequest`, `isLoading`) para `CreditCard`.

```
VStack className="gap-4"                                   // 14
 ├─ CreditCard {...rest, isLocked, freezeProgress}           // §4.2
 └─ ScrollView horizontal showsHorizontalScrollIndicator={false} testID="card-actions-scroll"
        style={{marginHorizontal:-16, marginVertical:-12}}         // "bleed" (uint32 4294967280/4294967284 no bytecode)
        contentContainerStyle={{paddingHorizontal:16, paddingVertical:12}}
     └─ CardActionsRow
          Android/sem liquid glass: View testID="card-actions-row" style={{flexDirection:'row', alignItems:'center', gap:10}}
          iOS 26 (useNativeLiquidGlass): GlassContainer spacing={8} testID="card-actions-glass-group" style=actionsRow
          └─ 6 × CardActionButton (ordem em §4.3)
```
Constantes exportadas: `CARD_ACTIONS_GAP = 10`, `CARD_ACTIONS_GLASS_GROUP_SPACING = 8`, `CARD_ACTIONS_VERTICAL_BLEED = 12` [O].

### 4.2 `CreditCard` (M3725, L1348812) [O]
Props: `lastFourDigits, skin, initialCollapsed = true, collapsed, isVisible = true, onExpandRequest, disableTilt, disableInteraction = false, isLocked = false, isLoading = false, freezeProgress`.

**Dimensões**
- `cardWidth = Math.min(windowWidth − 32)`: o `Math.min` recebe um só argumento no bytecode, então é só `W − 32` [O, desmontagem da função #33218, offsets 0x0cc–0x0dc].
- `cardHeight = cardWidth / 1.586` (`CARD_ASPECT_RATIO`, proporção ISO ID-1). Ex.: tela de 393 → 361 × 227,6.
- `STRIP_HEIGHT = 70` (altura recolhida).

**Árvore**
```
Animated.View style=[{width: cardWidth, alignSelf:'center'}, aEntrada]
 └─ Animated.View testID="credit-card-frame" collapsable={false} style=[{width: cardWidth}, aAltura]
     ├─ Animated.View testID="credit-card-surface" pointerEvents="none" style=[absoluteFill, {backgroundColor:'#09090B'}, aRaio]
     ├─ Animated.View testID="credit-card-clip" style=[absoluteFill, {overflow:'hidden'}, aRaio]
     │   └─ Pressable testID={collapsed ? 'collapsed-card' : 'expanded-card'} style={{flex:1}}
     │        onPress = disableInteraction ? undefined : (collapsed ? expandir : (skin.purrOnPress ? catPurr : undefined))
     │       ├─ Animated.Image source={skin.background} resizeMode="cover"
     │       │      style=[{width: cardWidth, height: cardHeight, position:'absolute'}, aDeslocaImagem]
     │       ├─ Animated.View className="absolute inset-0 p-5 justify-end" pointerEvents={collapsed ? 'none' : 'auto'} style=aOpacFrente
     │       │   └─ HStack "items-end justify-between"
     │       │       └─ Text "text-xl font-extrabold tracking-widest" style={{color: skin.frontTextColor}} → "•• {lastFourDigits}"
     │       └─ Animated.View className="absolute inset-0 flex-row items-center" pointerEvents={collapsed ? 'auto' : 'none'} style=aOpacFaixa
     │           ├─ Box "z-[1] h-full items-center justify-center px-4"
     │           │   └─ Text "text-xl font-extrabold tracking-widest" style={{color: skin.collapsed?.frontTextColor ?? skin.frontTextColor}} → "•• {lastFourDigits}"
     │           └─ Box "flex-1"
     ├─ freezeProgress.isFrostMounted && Animated.View testID="credit-card-locked-overlay" pointerEvents="none" style=[absoluteFill, {overflow:'hidden'}, aRaio]
     │   └─ CardFrostOverlay {progress, width: cardWidth, height: cardHeight, imageStyle: aDeslocaImagem, reducedMotion}   // §4.4
     ├─ isLoading && Animated.View testID="credit-card-loading-overlay" pointerEvents="none" style=[absoluteFill, {overflow:'hidden'}, {zIndex:10}, aRaio]
     │   └─ BlurView intensity={20} tint="dark" blurMethod="dimezisBlurView" style={{flex:1, alignItems:'center', justifyContent:'center'}}
     │       └─ ActivityIndicator size="small" color="rgba(255,255,255,0.8)"
     └─ Animated.View testID="credit-card-hairline" pointerEvents="none" style=[absoluteFill, {borderWidth:1, borderColor:'rgba(255, 255, 255, 0.12)'}, aRaio]
```
- Medidas: `p-5` = 17,5; `px-4` = 14. No cartão aberto o número fica no canto **inferior esquerdo** (HStack `items-end` com filho único). Na faixa recolhida ele fica à esquerda, centralizado na vertical.
- Tipografia do número: `text-xl font-extrabold tracking-widest` → Geist-ExtraBold, 17,5 / 24,5, letterSpacing 1, cor `#FFFFFF` nas duas skins.
- **O componente não desenha bandeira.** A skin tem `brand: SvgMastercardBrand` (círculos `#EB001B` e `#F79E1B`, interseção `#FF5F00`, viewBox `0 0 131.39 86.92`, M3214 L1021864), mas o `CreditCard` não a usa; a marca já está **dentro da imagem de fundo** [I: o PNG é a arte inteira]. `brand`, `textColor`, `labelColor` (`rgba(255,255,255,0.4)`) e `overlayColor` (`rgba(0,0,0,0.7)`) servem a outras telas (dados do cartão, fora do escopo).

**Máscara dos dados do cartão** [O]
- A home mostra **só** `"•• " + lastFourDigits` (dois bullets U+2022, espaço e os 4 dígitos). Nunca o número completo.
- Estado inicial do store `useCardStore` (zustand persist, `name:'pierre-card'`, L1021482): `lastFourDigits:'••••'`, `cardNumber:'•••• •••• •••• ••••'`, `holderName:''`, `expiryDate:'••/••'`, `cvv:'•••'`, `skinId:'PIERRE_ONE_DEFAULT'`, `backgroundId:'skin-ambient'`, `homeState:'default'`, `isLocked:false`. Antes de os dados chegarem, o cartão mostra "•• ••••".
- `partialize` **exclui** da persistência `lastFourDigits, cardNumber, holderName, expiryDate, cvv` (e as funções): os dados do cartão não vão para o disco.
- `setCardData`: se vier `cardNumber` ou `cvv`, agenda `setTimeout` de **60 000 ms** que zera `{cardNumber:'', cvv:''}`. O número completo e o CVV somem da memória depois de 1 minuto.

**Skins** (`SKINS`, M3213 L1021806) [O]. `skin = SKINS[skinId.toUpperCase()] ?? SKINS['PIERRE_ONE_DEFAULT']`:

| skinId | Fundo (asset → drawable Android) | frontTextColor | purrOnPress |
|---|---|---|---|
| `PIERRE_ONE_DEFAULT` | `assets/images/card/pierre-one-default.png` 1536×969 → `res/drawable-mdpi-v4/assets_images_card_pierreonedefault.png` | `#FFFFFF` | — |
| `PIERRE_ONE_001_EDITION` | `assets/images/card/pierre-one-001-edition.png` 1536×969 → `assets_images_card_pierreone001edition.png` | `#FFFFFF` | `true`: o toque no cartão aberto chama `catPurr()`, um padrão háptico de "ronronar" |

**Animações (Reanimated)** [O, código dos worklets]
- Entrada (`aEntrada`): `opacity` e `translateY` começam em `isVisible ? 1 : 0` e `isVisible ? 0 : -20`; quando `isVisible` muda → `withTiming(…, {duration: 300})`. No iOS há também `perspective: 1200`.
- Recolher (`progress`): `withSpring(collapsed ? 1 : 0, {damping: 15, stiffness: 180, mass: 1})`.
  - `aAltura.height = interpolate(p, [0,1], [cardHeight, 70])`; tilt `rotateX/rotateY` só no iOS (no Android `transform: []`).
  - `aRaio.borderRadius = interpolate(p, [0,1], [24, 35])`: recolhido, vira pílula.
  - `aDeslocaImagem.translateY = interpolate(p, [0,1], [0, −(cardHeight − 70)/2])`: a faixa mostra o **meio** da arte.
  - `aOpacFrente = interpolate(p, [0, 0.4], [1, 0])`; `aOpacFaixa = interpolate(p, [0.6, 1], [0, 1])`.
- Tilt por giroscópio: `useGyroscopeTilt({maxAngle: 16, updateInterval: 100, …})`, aplicado só no iOS.
- Toque no cartão recolhido: `onExpandRequest ? onExpandRequest() : setCollapsed(false)`.

### 4.3 Ações (`CardActionButton`, M9605 L3008899) [O]
Ordem: **Dados · Congelar/Descongelar · Configurações · Limites · Benefícios · Carteira**.

| label | ícone (`PierreIcon`) | testID | handler |
|---|---|---|---|
| `Dados` | `cardCredit` | `card-action-details` | `onDetails` |
| `Congelar` / `Descongelar` | `miscSnowflake` / `miscFire` | `card-action-lock` (+ `loadingTestID="card-lock-loading"`) | `onLock`; `active` = `useFreezeActionState(isLocked, reducedMotion)`; `disabled` = `loading` = `isLockLoading` |
| `Configurações` | `navigationSettings` | `card-action-settings` | `onSettings` |
| `Limites` | `formFilter` | `card-action-limits` | `onLimits` |
| `Benefícios` | `cwProductsAward` | `card-action-benefits` | `onBenefits` |
| `Carteira` | `walletSimple` | `card-action-wallet` | `onWallet` |

- O rótulo e o ícone do botão de congelar trocam junto com `active` [I: provavelmente quando a animação passa de `CARD_FREEZE_ACTION_SWAP_PROGRESS = 0.43`].
- **Android** (`useNativeLiquidGlass()` falso) → `FallbackCardActionButton`:
```
Button variant={active ? 'default' : 'secondary'} rounded="full" size="default" disabled testID onPress
 ├─ loading ? ActivityIndicator size="small" testID={loadingTestID}
 │          : PierreIcon name size={20} className={active ? 'text-primary-foreground' : 'text-foreground'}
 └─ ButtonText → label
```
  O `Button` (gluestack, M3614 L1324800) tem base `flex-row items-center justify-center gap-2` (gap 7), `size default` = `h-10 px-4 py-2` (35 de altura, padding 14 × 7) e `rounded-full`. Cores: `secondary` = `bg-secondary` **#27272A**, `default` = `bg-primary` **#FAFAFA**; desabilitado fica com `opacity-60`. `ButtonText` = `text-sm font-medium` → Geist-Medium **16**, cor #FAFAFA (secondary) ou #09090B (default). Ícone de 20 px, #FAFAFA ou #09090B.
- **iOS 26** (liquid glass) → `GlassCardActionButton`, só para referência:
  - Contêiner: `GlassContainer glassEffectStyle="regular" isInteractive`, pílula `{height:35, borderRadius:17.5}`.
  - `Pressable` `{flex:1, flexDirection:'row', alignItems:'center', justifyContent:'center', gap:7, paddingHorizontal:14}`.
  - Overlay ativo absoluto (raio 17,5, `backgroundColor: primary`) com `opacity = activeValue`, animada por `withTiming(active ? 1 : 0, {duration: 220, easing: Easing.out(Easing.quad)})`.
  - Texto `text-sm font-medium` com cor interpolada `foreground → primaryForeground`; desabilitado `opacity 0.6`.

### 4.4 Congelar o cartão: `CardFrostOverlay` (M4142, L1674836) [O]
Montado quando `freezeProgress.isFrostMounted`. Constantes (M4143, L1675300):

| Constante | Valor |
|---|---|
| `CARD_FREEZE_START_DELAY_MS` | 400 |
| `CARD_FREEZE_DURATION_MS` / `CARD_THAW_DURATION_MS` | 1150 / 750 |
| `CARD_FREEZE_BEZIER` | `[0.3, 0, 0.25, 1]` |
| `CARD_FREEZE_ICE_STAGES` | 4 quadros de gelo, com faixas `0.04–0.33`, `0.20–0.54`, `0.36–0.75` e `0.52–0.96` |
| `CARD_FREEZE_COLD_WASH_RANGE` / `…_MAX_OPACITY` / `…_COLOR` | `0.04–0.82` / 0,42 / `#D6E6F7` |
| `CARD_FREEZE_MOTES_RANGE` | `0.12–0.70` |
| `CARD_FREEZE_ACTION_SWAP_PROGRESS` | 0,43 |
| `CARD_FROZEN_INSIGHT_DELAY_MS` | 800 |

- Gelo: 4 imagens `assets/images/card/freeze/ice1..4` (722×456; drawables `assets_images_card_freeze_ice1..4.webp`). Cada uma tem `opacity = interpolate(progress, [stage.start, stage.end], [0,1], CLAMP)` e usa o mesmo `imageStyle` do fundo.
- "Cold wash": camada `#D6E6F7` (testID `credit-card-frost-wash`) com opacidade de 0 até 0,42.
- Partículas: 17 ícones `miscSnowflake` (testID `credit-card-frost-motes`).
  - Tamanho 7–9 px, cores `#D8CEEF` ou `#CCECFC`, opacidade 0,30–0,38.
  - Ciclos de 4 600 a 6 800 ms, deslocamento `travelX/Y` de até ±50 px.
  - `fadeIn = min(1, t/0.2)` e `fadeOut = max(0, (1−t)/0.35)`.
  - Tabela completa em L1674374–1674404.
- `useCardFreezeProgress(isLocked)` (M4148) devolve `{progress, isFrostMounted, reducedMotion}`. Com "reduzir movimento" ativo, o estado muda sem animação [I pelo nome].

### 4.5 Strings
`Dados` · `Congelar` · `Descongelar` · `Configurações` · `Limites` · `Benefícios` · `Carteira` · `•• {4 dígitos}`.

---

## 5. Gastos por categoria na home do cartão

### 5.1 Componente e o que muda em relação à Vision [O L2982992]
O `WidgetRenderer` chama **diretamente** `VisionSpendingByCategoryWidget` (M9035), o mesmo da Vision (`02-gastos.md` §4.4–4.10), **sem** a camada `SpendingByCategoryWidgetConnected`. Props passadas: `{data: widget.data, size: widget.size, onPress, isLoading, emptyState}`. Layout, tipografia, barra segmentada, linhas e strings são **idênticos** aos do 02-gastos; as diferenças são estas:

| Aspecto | Vision (02-gastos §4.3) | Home do cartão |
|---|---|---|
| Dados | `GET /api/mobile/vision/widgets/spending-by-category/data` | `transformSpendingByCategory(statements)`, calculado no app a partir de `/credit/statements/v2` do mês civil (§5.2) |
| Erro | ramo próprio do Connected (`testID spending-by-category-error`, wide com `minHeight 170`) | substituído pelo placeholder do renderer (§1): `{id}-error`, **altura 200 também no wide** |
| `widgetMode` | estado local `groups`/`categories` | não é passado → default `'groups'` fixo |
| Toggle Categorias/Grupos (full) | troca o modo e busca subcategorias | **aparece, mas não faz nada**: `onWidgetModeChange` é `undefined` (o botão chama `onModeChange?.(…)`) [O defaults L2864048] |
| Subcategorias | `useSpendingByCategorySubcategories` | nunca carregadas (`subcategoryRows = []`, `subcategoriesLoading = false`) |
| Toque no full | `onPress({params:{chartMode}})` | `onPress` do renderer, sem parâmetros; destino resolvido por `entry.navigateTo` + `tapAction` (§1) |
| `animate` | repassado pela home | não passado → `false` |
| Default de `data` | `{amount:'R$ 0,00', categories:[], subtitle:'0 categorias', period:'este mês'}` quando `data` é nulo | **sem default**: o componente lê `data.categories` diretamente [O]. Se a entrada não tiver `data` (nem live nem do slot), a renderização quebra [I: risco; ver Dúvidas] |

### 5.2 Dados: `transformSpendingByCategory(statements)` (M9621, `livedata.js` 1453) [O]
```ts
const debits = statements.filter(t => t.direction === 'debit' && !t.denied);
const byKey = new Map<string, number>();
for (const t of debits) {
  const key = (t.mccCategory ?? t.category ?? 'outros').toLowerCase().trim();
  byKey.set(key, (byKey.get(key) ?? 0) + t.amountCents);
}
const total = debits.reduce((s, t) => s + t.amountCents, 0);
const top = [...byKey.entries()].sort((a, b) => b[1] - a[1]).slice(0, 10);   // [I: ordem decrescente pelo padrão do comparador]
const PALETTE = ['#95FF48','#3B82F6','#F59E0B','#EF4444','#8B5CF6','#EC4899','#14B8A6','#F97316','#6366F1','#84CC16'];
const categories = top.map(([key, cents], i) => {
  const d = getCategoryDisplay(key);                     // M8702 L2734603
  return {
    color: d.color ?? PALETTE[i % PALETTE.length],       // na prática d.color sempre existe
    value: cents,
    name: d.label,
    amount: (cents / 100).toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' }),
    percentage: total > 0 ? Math.round(cents / total * 100) : 0,
    iconKey: key,
  };
});
return {
  amount: formatBRL(total),                               // soma de TODOS os débitos do mês, não só do top 10
  categories,
  subtitle: `${categories.length} categoria${categories.length !== 1 ? 's' : ''}`,   // no máximo "10 categorias"
  period: 'este mês',
};
```
- Fonte: `useStatements({startDate, endDate, limit: 100})` com o mês civil corrente. Só as **100** primeiras transações do mês entram na conta [O `limit: 100`].
- Agrupamento pela chave crua (minúscula): chaves desconhecidas diferentes viram **várias** categorias chamadas "Outros" [O: cada chave é uma entrada do Map; `getCategoryDisplay` devolve o item `outros` para qualquer chave fora do mapa].

### 5.3 Mapa de categorias do cartão (`getCategoryDisplay`, M8702) vs. cor final exibida [O]
Chaves aceitas (minúsculas, com e sem acento, e em inglês): `alimentacao|alimentação|food`, `restaurantes|restaurants`, `transporte|transport`, `viagem|travel`, `combustivel|combustível|fuel`, `saude|saúde|health`, `educacao|educação|education`, `entretenimento|entertainment`, `compras|shopping`, `casa|home`, `servicos|serviços|services`, `governo|government`, `outros|others`. Qualquer outra → `outros`.

O widget da Vision **recolore** pelo nome (`resolveCategoryColor({name}, c.color)`, 02-gastos §4.5/§4.8). O nome que chega é o `label` do mapa do cartão, então a cor final é:

| label (cartão) | Cor do mapa do cartão | Ícone do mapa do cartão | Casa com o mapa da Vision? | **Cor exibida** |
|---|---|---|---|---|
| Alimentação | #F97316 | cartShopping | sim | **#F97316** |
| Restaurantes | #F97316 | miscForkKnive | não (a Vision só tem "Restaurantes, bares…") | **#F97316** |
| Transporte | #FBBF24 | transportTrain | sim | **#F59E0B** (Vision vence) |
| Viagem | #3B82F6 | transportPlane | não ("Viagens") | **#3B82F6** |
| Combustível | #FBBF24 | miscFire | não | **#FBBF24** |
| Saúde | #10B981 | heartHealth | sim | **#10B981** |
| Educação | #22D3EE | miscBook | sim | **#06B6D4** (Vision vence) |
| Entretenimento | #3B82F6 | miscMusic | não | **#3B82F6** |
| Compras | #EC4899 | productBag | sim | **#EC4899** |
| Casa | #6366F1 | buildingsHome | não ("Moradia") | **#6366F1** |
| Serviços | #A1A1AA | buildingsEnterprise | não | **#A1A1AA** |
| Governo | #A1A1AA | miscLegal | não | **#A1A1AA** |
| Outros | #A1A1AA | navigationCategories | não | **#A1A1AA** |

- Avatares das linhas do full (`CategoryAvatar`, 02-gastos §4.7): o `iconKey` enviado é a **chave crua** (`alimentacao`, `outros`…), que não é nome de `PierreIcon`; o avatar cai então na busca por nome no mapa da Vision e, sem correspondência, no ícone `walletMoney` [I: segue a regra de `03-…` §6 (`iconKey` só vale se existir em `pierreIcons`)]. Os `iconName` do mapa do cartão **não** chegam ao widget.

### 5.4 O `SpendingByCategoryWidget` "do cartão" (M9763) é código morto [O]
`SpendingByCategoryWidget` (L3078245), `SpendingByCategoryWidgetFull` (L3078768) e `CategoryBar` (L3078530), descritos em 02-gastos §5, só são reexportados pelo barrel M9762 → M9752 (usado por `CardAddWidgetScreen` e `CardWidgetDetailScreen`). Nenhum código lê `.SpendingByCategoryWidget` desse barrel: as telas de adicionar e de detalhe usam `SharedAddWidgetScreen`/`SharedWidgetDetailScreen`/`CardAIWidgetPreview`/`AIWidgetRenderer` [O: grep L3072693–3080323]. Para a réplica, ignore-os; o widget real é o da Vision com os dados de §5.2.

---

## 6. Outros widgets de sistema do cartão (diferenças em relação à Vision)

Todos usam o componente **de apresentação** da Vision (nunca o `…Connected`), então valem integralmente o layout, a tipografia, as cores e as strings dos documentos citados. O que muda é a **origem e o formato dos dados**: o app monta os dados a partir da API de crédito (M9621), e não do endpoint `/vision/widgets/{id}/data`. Onde o formato não bate com o que o componente espera, o visual muda; isso vai marcado abaixo.

### 6.1 `available-limit` → `AvailableLimitWidget` (03-… §1) [O]
```ts
// transformAvailableLimit(GET /credit/limit)
{ availableAmount: r.availableCredit, totalLimit: r.creditLimit, usedAmount: r.usedCredit, usedPercentage: r.utilizationPercentage }
```
- Sem a mescla com cartões do Open Finance que a Vision faz em `useAvailableLimitData` (03-… §1): só o cartão Pierre.
- Os campos vão crus. O componente compara `totalLimit === 'R$ 0,00'` e mostra `Disp: ${availableAmount}`, então espera **strings BRL** [I: a API de crédito devolve os valores já formatados; não verificado].

### 6.2 `monthly-spending` → `VisionMonthlySpendingWidget` (02-gastos §2) [O]
```ts
// transformMonthlySpending(statements, startDate, endDate)
{ amount: formatBRL(getSpendingTotalCents(statements)), data: buildMonthlyCumulativeTrend(statements, startDate, endDate) }
```
- O renderer passa só `{data, size, onPress, isLoading}`: **sem `chartVariant`** e sem `emptyState`. Vale então a variante padrão do componente (02-gastos §2.6) [I: default `area`].
- Os gráficos nativos (`PierreCharts`) são os mesmos. A série diária vem das transações do cartão do mês (até 100), e não dos buckets do servidor [O].

### 6.3 `recent-transactions` → `VisionRecentTransactionsWidget` (03-… §5) [O]
O renderer passa `data = {transactions: widget.data.transactions}`. Itens gerados por `transformRecentTransactions(statements)`:
```ts
statements.sort((a,b) => b.transactionDate.localeCompare(a.transactionDate)).slice(0, 5).map(t => {
  const d = getCategoryDisplay(t.mccCategory ?? t.category);
  const iso = toSaoPauloIso(t.transactionDate);
  const hhmm = formatTime(iso);                 // iso.slice(11,16) → "14:32"
  const date = formatDayMonth(iso);             // "DD/MM"
  if (t.denied)                return { name: t.description, category: `${hhmm} • Não aprovada`, amount: t.amount, date,
                                        iconName:'arrowCircleUpRight', iconColor: d.color, pierreIcon: d.iconName, logoUrl: t.logoUrl, variant:'denied' };
  if (t.direction === 'credit') return { name: t.description, category: `${hhmm} • Limite liberado`, amount: t.amount, date,
                                        iconName:'arrowCircleUpRight', iconColor:'#95FF48', pierreIcon:'arrowCircleUpRight', logoUrl: undefined, variant:'cashback' };
  return                               { name: t.description, category: `${hhmm} • ${d.label}`, amount: t.amount, date,
                                        iconName:'arrowCircleUpRight', iconColor: d.color, pierreIcon: d.iconName, logoUrl: t.logoUrl, variant:'default' };
});
```
- Incompatibilidade de formato [I, comparando com os campos lidos pela `VisionTransactionRow`, 03-…]: o item **não tem `type`**, então o valor sai sem sinal e na cor neutra #FAFAFA (nada de verde ou rosa). Também não tem `customLogoUrl`/`companyLogoUrl`: a linha ignora `logoUrl`, `pierreIcon` e `variant`, e o avatar cai no `iconName` (`arrowCircleUpRight`) com `iconColor`.
- Limite: o transform corta em 5; o componente mostra 3 (half), 1 (wide) ou 5 (full).

### 6.4 `installments` → `InstallmentsWidget` / `InstallmentsWidgetFull` (03-… §2) [O]
Fonte: itens da fatura atual (`useInvoiceItems(currentInvoice.id)`). `transformInstallments(items)`:
```ts
const inst = items.filter(i => i.installmentInfo != null);          // ex.: "3/10"
const total = inst.reduce((s, i) => s + i.amountCents, 0);
const rows = inst.slice(0, 3).map(i => {
  const [a, b] = (i.installmentInfo ?? '').split('/');
  const cur = Number(a) || 1, tot = Number(b) || cur;
  return { id: String(i.id), name: i.description || 'Parcelamento', amount: i.amount,
           installmentLabel: `Parcela ${cur} de ${tot}`, progressPercentage: tot > 0 ? Math.round(cur / tot * 100) : 0,
           logoUrl: i.logoUrl ?? null };
});
const n = inst.length;
return { count: String(n), subtitle: n === 0 ? 'Nenhuma parcela' : 'Em andamento',
         paidPercentage: n === 0 ? 0 : 25,                                         // ← constante!
         paidAmount: n === 0 ? (0).toLocaleString('pt-BR',{style:'currency',currency:'BRL'}) : formatBRL(Math.round(total * 0.25)),
         totalAmount: formatBRL(total), items: rows };
```
- **Atenção:** o progresso pago é **fixo em 25 %**, e o "pago" é 25 % do total. É um valor de marcação no código, não um cálculo real [O]. Para replicar o comportamento idêntico, copie; para um produto real, calcule.
- O registro manda `size: 'full'` para `InstallmentsWidgetFull` e o tamanho real para `InstallmentsWidget`.

### 6.5 `subscriptions` → `SubscriptionsWidget` / `SubscriptionsWidgetFull` (04-assinaturas) [O]
Fonte: transações do mês (até 100). `transformSubscriptions(statements)`:
```ts
const groups = new Map();                                // chave = description (exata)
for (const t of statements.filter(t => t.direction === 'debit' && !t.denied)) {
  const g = groups.get(t.description);
  const isSub = t.movementType === 'subscription';
  if (!g) groups.set(t.description, { count: 1, totalCents: t.amountCents, isSubscription: isSub });
  else { g.count++; g.totalCents += t.amountCents; if (isSub) g.isSubscription = true; }
}
const subs = [...groups].filter(([, g]) => g.isSubscription || g.count >= 3)
  .map(([name, g]) => { const avg = Math.round(g.totalCents / g.count); return { name, amount: formatBRL(avg), avgCents: avg }; });
return { logos: subs.map(s => ({ name: s.name, logoUrl: undefined, amount: s.amount })),
         amount: formatBRL(subs.reduce((s, x) => s + x.avgCents, 0)),
         subtitle: subs.length === 0 ? 'Nenhuma assinatura' : `${subs.length} assinatura${subs.length > 1 ? 's' : ''}` };
```
- Heurística: é assinatura o que vier marcado como `movementType: 'subscription'` **ou** a mesma descrição que aparece 3 ou mais vezes no mês.
- Sem `logoUrl`, `nextRenewalDate`, `bankName` e `totalCount`: as logos caem no fallback de iniciais (04-assinaturas) e o subtítulo é sempre o do transform (a regra "Próxima em N dias" exige `nextRenewalDate`) [I pela regra de 04 §1].
- O renderer não passa `totalCount` ao componente [O].

### 6.6 `cdb-position` → `CdbPositionWidget` (05a-cdb §3) [O]
- Dados: `useVisionSystemWidgetData('cdb-position', 'full', CDB_POSITION_DAILY_LIQUIDITY_VARIANT, {directFetcher: fetchCdbDailyLiquidityPositionDirect, accessGate: useCdbWidgetIdentityGate(), enabled})`. É sempre a variante de **liquidez diária** e sempre pedida como `full`, qualquer que seja o tamanho do slot.
- Tamanho entregue ao componente: `size === 'half' ? 'half' : 'full'` (o wide vira full).
- Toque: `CARD_CDB_POSITION_TAP_HREF` → detalhes do CDB com `{from:'card_home', productId:'pierre-one'}` (§1).

---

## 7. Snippet TSX de referência (caminho Android, dark)

Reaproveita `Widget2x2`, `WidgetWide`, `WidgetLoadingState`, `WidgetEmptyStateSlot`, `PrivacyCurrencyText`, `useHideValues`, `HIDDEN_VALUE` e `parseBRLToCents` do snippet de `00-base.md` §15. Os ícones (`AwardIcon`) são seus; não use os da CloudWalk.

```tsx
// card-home-widgets.tsx
import React, { memo, useMemo } from 'react';
import { View } from 'react-native';
import { Widget2x2, WidgetWide, WidgetLoadingState, WidgetEmptyStateSlot,
         PrivacyCurrencyText, useHideValues, HIDDEN_VALUE, parseBRLToCents } from './widget-base';
import { AwardIcon } from './icons';

const brl = (cents: number) => (cents / 100).toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' });

// ---------- Cashback ----------
export function toCashback(s?: { availableCents?: number; pendingCents?: number; projectedCents?: number }) {
  const pending = (s?.pendingCents ?? 0) + (s?.projectedCents ?? 0);
  return { amount: brl((s?.availableCents ?? 0) + pending), pendingAmount: brl(pending) };
}

const CashbackChip = () => (
  <View style={{ width: 35, height: 35, borderRadius: 9999, backgroundColor: '#243400',
                 alignItems: 'center', justifyContent: 'center' }}>
    <AwardIcon size={24} color="#CFFF04" />
  </View>
);

export const CashbackWidget = memo(function CashbackWidget(
  { amount, pendingAmount, size = 'half', onPress, isLoading }:
  { amount: string; pendingAmount: string; size?: 'half' | 'wide'; onPress?: () => void; isLoading?: boolean }) {
  const hidden = useHideValues();
  const value = hidden ? HIDDEN_VALUE : amount;
  const subtitle = hidden ? '' : `${pendingAmount} pendentes`;
  if (isLoading) return size === 'wide'
    ? <WidgetWide testID="cashback-widget" label="Cashback" value="" visual={<WidgetLoadingState variant="content" />} />
    : <Widget2x2 testID="cashback-widget" label="Cashback" value="" subtitle="" visual={<WidgetLoadingState variant="visual" />} />;
  return size === 'wide'
    ? <WidgetWide testID="cashback-widget" label="Cashback" value={value} subtitle={subtitle} visual={<CashbackChip />} onPress={onPress} />
    : <Widget2x2 testID="cashback-widget" label="Cashback" value={value} subtitle={subtitle} visual={<CashbackChip />} onPress={onPress} />;
});

// ---------- Fatura ----------
const MONTHS = ['Jan','Fev','Mar','Abr','Mai','Jun','Jul','Ago','Set','Out','Nov','Dez'];
type Bar = { value: number; isCurrent: boolean };

export function toInvoice(current: any | null, invoices: any[]) {
  const last5 = [...invoices].sort((a, b) => a.referenceMonth.localeCompare(b.referenceMonth)).slice(-5);
  const bars: Bar[] = last5.length
    ? last5.map((inv, i) => ({ value: inv.grossTotalCents ?? inv.totalAmountCents, isCurrent: i === last5.length - 1 }))
    : [{ value: 0, isCurrent: true }];
  if (!current) return { monthLabel: MONTHS[new Date().getMonth()] ?? 'Jan', amount: 'R$ 0,00', closingDate: '', bars };
  const m = current.referenceMonth ? Number(current.referenceMonth.slice(5, 7)) - 1 : new Date().getMonth();
  const end: string | undefined = current.cycleEndDate;
  return { monthLabel: MONTHS[m] ?? 'Jan', amount: current.totalAmount,
           closingDate: end ? `${end.slice(8, 10)}/${end.slice(5, 7)}` : '', bars };
}

const InvoiceBars = ({ bars }: { bars: Bar[] }) => {
  const max = Math.max(...bars.map(b => b.value), 1);
  const recent = bars.slice(-5);
  const slots = [...recent.map(b => (b.value > 0 ? b : null)), ...Array(5 - recent.length).fill(null)];
  return (
    <View style={{ flexDirection: 'row', justifyContent: 'flex-start', alignItems: 'flex-end', height: 80 }}>
      {slots.map((b, i) => {
        const ml = i > 0 ? 6 : 0;
        if (!b) return <View key={`empty-${i}`} style={{ width: 12, height: 12, borderRadius: 6, backgroundColor: '#27272A', marginLeft: ml }} />;
        const h = `${Math.max((b.value / max) * 100, 5)}%` as const;
        return b.isCurrent
          ? <View key={`bar-${i}`} style={{ width: 10, height: '100%', borderRadius: 5, justifyContent: 'flex-end', overflow: 'hidden', backgroundColor: '#27272A', marginLeft: ml }}>
              <View style={{ width: '100%', borderRadius: 5, height: h, backgroundColor: '#CFFF04' }} />
            </View>
          : <View key={`bar-${i}`} style={{ width: 10, borderRadius: 5, height: h, backgroundColor: '#27272A', marginLeft: ml }} />;
      })}
    </View>
  );
};

export const InvoiceWidget = memo(function InvoiceWidget(
  { monthLabel, amount, closingDate, bars, onPress, isLoading, emptyState }:
  { monthLabel?: string; amount?: string; closingDate?: string; bars?: Bar[]; onPress?: () => void; isLoading?: boolean; emptyState?: React.ReactNode }) {
  const hidden = useHideValues();
  const b = bars?.length ? bars : [{ value: 0, isCurrent: true }];
  const label = `Fatura de ${monthLabel ?? ''}`;
  const isEmpty = !isLoading && b.every(x => x.value === 0) && (!amount || amount === 'R$ 0,00');
  if (isLoading) return <Widget2x2 testID="invoice-widget" label={label} value="" subtitle="" visual={<WidgetLoadingState variant="visual" />} />;
  if (isEmpty) return <Widget2x2 testID="invoice-widget" label={label} value="—" subtitle=""
                                 visual={<WidgetEmptyStateSlot emptyState={emptyState} compact />} onPress={onPress} />;
  return (
    <Widget2x2 testID="invoice-widget" label={label} visual={<InvoiceBars bars={b} />} onPress={onPress}
      valueSlot={<PrivacyCurrencyText animate={false} valueCents={parseBRLToCents(amount!)} hidden={hidden}
                   fontSize={20} fontWeight="800" color="#FAFAFA" disableMinWidth testID="invoice-widget-value" />}
      subtitle={closingDate ? `Fecha em ${closingDate}` : 'Sem data de fechamento'} />
  );
});

// ---------- Máscara do cartão (só os 4 últimos) ----------
export const maskedCardNumber = (lastFour = '••••') => `•• ${lastFour}`;
```

---

## 8. Dúvidas abertas
1. **`rounded-3xl` = 21 ou 24?** A config compilada (00-base §14.5) dá 21. Os placeholders de vazio/erro do renderer do cartão usam essa classe; já os docs 02-gastos e 05b dizem 24. Para os cards normais o raio 24 vem de `style`, então não há conflito.
2. **Formato dos campos da API de crédito.** `totalAmount` (fatura), `availableCredit`/`creditLimit`/`usedCredit` (limite), `amount` (transações e itens da fatura): o app trata todos como strings BRL prontas (compara com `'R$ 0,00'`, faz `parseBRLToCents`), mas a resposta não foi vista [I].
3. **Transações recentes no cartão.** O transform gera `{iconName, iconColor, pierreIcon, logoUrl, variant}` sem `type`, enquanto a `VisionTransactionRow` lê `type`/`customLogoUrl`/`companyLogoUrl`. Conclusão (valor neutro e sem logo) por comparação de campos, sem acompanhar a linha passo a passo [I].
4. **`data` ausente.** Quando não há entrada "live", o `LiveWidgetRenderer` repassa o `widget` do slot como veio. Se o slot não trouxer `data` (ex.: primeira abertura sem cache), o `VisionSpendingByCategoryWidget` (`data.categories`) e o `CashbackWidget` (template com `pendingAmount` indefinido → "undefined pendentes") podem quebrar ou mostrar lixo. Não verifiquei se o slot sempre traz `data` (é do agente da tela/grid).
5. **`onExpandRequest`** do `CardWithCollapse`: o alvo do `withTiming(…, 250)` não foi materializado pelo decompilador. Supus 0 (expandido).
6. **`useFreezeActionState`** e **`useCardFreezeProgress`** não foram lidos linha a linha: não está confirmado o momento exato em que o botão troca "Congelar" ↔ "Descongelar" (supus o `CARD_FREEZE_ACTION_SWAP_PROGRESS = 0.43`) nem o comportamento com "reduzir movimento".
7. **Ordenação do `transformSpendingByCategory`.** O comparador desmontado subtrai os valores das duas entradas; entendi como `b − a` (decrescente), coerente com o uso. Confirmar pela desmontagem da função, se precisar de certeza.
8. **Tamanhos aceitos pelo registro do cartão** (`supportedSizes` de `cashback` e `invoice`) são do agente do registro. O `InvoiceWidget` ignora `size`. Se o registro permitir wide, o layout half é esticado na largura (170 vs 200 não se aplica: a altura continua 200).
9. **Ícones `PierreIcon`** (`cwProductsAward`, `cardCredit`, `miscSnowflake`, `miscFire`, `navigationSettings`, `formFilter`, `walletSimple`): os SVGs não foram extraídos aqui.
