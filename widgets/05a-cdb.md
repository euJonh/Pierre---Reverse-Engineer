# W5a — Widgets CDB da home "Vision" (`cdb-investment`, `cdb-performance`, `cdb-position`)

> Análise estática do bundle Hermes v98 (`_trabalho/out.js`). **[O]** = observado (`Lnnnn` = linha de `out.js`); **[I]** = inferido.
> Base comum (shell, grid, `Widget2x2`/`WidgetWide`/`WidgetFull`, estados, privacidade, moeda, tokens): ver `widgets/00-base.md`. Busca de dados comum: §1 de `widgets/01-carteira-saldo-fluxo.md`.

## 0. Registro, visibilidade e flags

### 0.1 `VISION_WIDGET_REGISTRY` (L772595–772609) [O]
| id | label | size padrão | supportedSizes | tapAction / navigateTo | extras |
|---|---|---|---|---|---|
| `cdb-investment` | Investimentos CDB | `wide` | `['wide']` | `navigate` → `/(protected)/vision/cdb-investment-entry` (na prática o componente usa `Link` próprio, ver §1.6) | `contentOverflow:'visible'`, `insertPosition:'top'` |
| `cdb-performance` | Vencimento CDB | `half` | `['half','wide']` | `navigate` → `/(protected)/investments/cdb-investment-details` | `internalPressSizes: ['half','wide']`, `contentOverflow:'visible'`, `insertPosition:'top'` |
| `cdb-position` | Posição CDB | `full` | `['half','full']` | `navigate` → `INVESTMENTS_HOME_TAB_HREF` = **`/vision?section=investments`** (L772697) | `contentOverflow:'visible'`, `insertPosition:'top'` |

Listas do mesmo módulo [O L772620–772659]:
- `DEFAULT_VISION_SORTABLE_ORDER` = `wallet, account-balance, cash-flow, cdb-investment, available-limit, installments, cards, monthly-spending, recent-transactions, spending-by-category, subscriptions` → **`cdb-investment` está na home padrão** (4º).
- `ON_DEMAND_PICKER_WIDGET_IDS` = `cdb-position, cdb-performance, leo-agent, leo-insight` (só aparecem no picker sob demanda).
- `VISION_PICKER_RETIRED_WIDGET_IDS` = `cards, cdb-performance` → **`cdb-performance` foi aposentado do picker** (o código ainda o renderiza se já estiver no layout).
- `isVisionWidgetSystemManaged(id)` = `['cdb-investment'].includes(id)` (o sistema gerencia a presença dele).
- `isVisionWidgetHidden(id)` (L771975): `patrimony` sempre oculto; `cdb-investment` e `cdb-performance` ficam ocultos quando `!isCdbInvestmentEntryEnabled()`.
- Fora da Vision existe um 2º registro (home do cartão, L775668): `{id:'cdb-position', size:'full', label:'CDB', description:'Acompanhe a posição do seu investimento em CDB', icon:'coinPiggy'}`.
- Após uma compra de CDB, o app adiciona `cdb-position` automaticamente (eventos `widget_id:'cdb-position', source:'auto_after_buy'`, resultados `failed_to_sync | failed_to_add | sync_skipped`, L1916278–1916594) [O].

### 0.2 Flag de CDB (L1798380–1798720, L772714) [O]
`isCdbInvestmentEntryEnabled()` lê uma variável de módulo escrita por `setCdbInvestmentsEnabled(useCdbInvestmentsEnabled())` (efeito na home, L2892886/L2901087). O hook lê `useMobileConfig()` (`/api/mobile/config`):
```ts
flags = { enabled: data.cdbInvestmentsEnabled === true,
          internalOnly: metadata?.cdbInvestmentsInternalOnly !== false,   // default true
          autoOpen: metadata?.cdbOfferAutoOpenEnabled === true,
          onboardingInstability: metadata?.cdbOnboardingInstability === true,
          regularInstability: metadata?.cdbRegularInstability === true }
// sem config: {enabled:false, internalOnly:true, ...}
enabled = flags.enabled && (!flags.internalOnly || user.email.trim().toLowerCase().endsWith('@cloudwalk.io'))
```
Log: `'[CDB flags] /api/mobile/config resolved flags'`.

### 0.3 Dados
Os três passam pelo hook comum `useVisionSystemWidgetData(id, size, …)` (ver §1 de `01-carteira-saldo-fluxo.md`: queryKey, cache, `staleTime`). A origem dos dados muda:
- `cdb-investment`: endpoint da Vision `GET /api/mobile/vision/widgets/cdb-investment/data?size=wide`. O payload chega **pronto do servidor** (título, subtítulo, série).
- `cdb-performance` e `cdb-position`: `directFetcher` que chama a API bancária de CDB (`cdbBankingApiClient`: `/api/cdb/summary`, `/api/cdb/list`, …) e monta o payload **no app**. Só rodam com KYC aprovado (`accessGate`). Detalhes em §2.1 e §3.1.

## 1. `cdb-investment`: "Investimentos CDB" (só `wide`)

Módulos: `CdbInvestmentWidget`, `CdbInvestmentChart` e `CdbInvestmentWidgetPressable` (M9049, L2867861–2869670); `CdbInvestmentWidgetConnected` (M9048, L2869688); `useCdbInvestmentData` (L2870236); `useCdbWidgetEntryDestination` / `cdbWidgetEntryHref` (M9050, L2870434–2870518). Gráfico: **react-native-gifted-charts** `LineChart` (M6817). Helpers do eixo: M6870 (`formatYAxisCompact`, `computeChartLayout`, `rebaseForDomain`, L2296919).

**Não usa `WidgetWide`**: monta o próprio cartão com `WidgetGlassCard` (no Android vira `VStack bg-surface`, ver 00-base §9.5).

### 1.1 Dados [O L2870236–2870395]
- `useCdbInvestmentData()` = `useVisionSystemWidgetData('cdb-investment', 'wide')`. **Sempre pede `size=wide`**, qualquer que seja o tamanho.
- Devolve `{data, isLoading, isError, isFetched}`. Em erro, chama `logger.captureException(err, {widget_id:'cdb-investment'})`, exceto para `ApiError` com `code` em `NETWORK_ERROR | SESSION_EXPIRED | TIMEOUT`. Um erro que não é `Error` vira `new Error('[CdbInvestmentWidget] data query failed')`. Loga uma vez por objeto de erro (ref).
- Campos de `data` usados:

| Campo | Uso |
|---|---|
| `headline` | título grande (string pronta do servidor) |
| `subtitle` | linha de apoio (até 2 linhas) |
| `cohort` | `'simulated'` → modo "marcos" (pontos com valores); outro valor → modo "comparação" (2ª linha tracejada) |
| `series[]` | `{ label, cdb, comparison }` por ponto |
| `comparisonLabel` | texto da legenda da 2ª série |

### 1.2 Estados no `Connected` [O L2869688–2870170]
Props: `size='half'` (default), `onPress, onNavigationTransitionPressReady, emptyState, animate=false`.

| Condição (nesta ordem) | Render |
|---|---|
| `isError` | `View testID="cdb-investment-error" className="rounded-3xl bg-surface" style={S}` → `WidgetErrorState compact` ("Falha ao sincronizar dados") |
| `isFetched && !data` | `View testID="cdb-investment-empty" className="rounded-3xl bg-surface" style={S}` → `WidgetEmptyStateSlot {emptyState, compact: size==='half'}` ("Sem dados disponíveis") |
| senão | `CdbInvestmentWidget {data, size, onPress, onNavigationTransitionPressReady, entryHref, animate, isLoading: isLoading \|\| !isFetched}` |

`S` = `size==='wide' ? {minHeight:210, alignItems:'center', justifyContent:'center', padding:16} : {height:200, …iguais}`. `rounded-3xl` dá raio **21** (não 24); `bg-surface` = #18181b.

Analytics [O]: `getCdbExperimentVariant(user.id)` (assíncrono; default `'control'`). Depois que o variant carrega, a cada mudança de estado (`'error' | 'empty' | 'shown'`) chama `trackCdbEntryExperience({experience_type:'widget', result, experiment_key:'cdb-investment-entry', experiment_variant})`, uma vez por estado (ref).

### 1.3 Destino do toque [O L2870434–2870518, L1878350–1878500]
`useCdbWidgetEntryDestination()`:
```ts
const { hasBankConnection, hasData } = useHasBankConnection();
const { data: accounts, isLoading } = useAccountBalanceAccounts();
if (hasData && !hasBankConnection) return 'offer';
if (isLoading) return 'entry';
if (!hasIdleMoney(accounts)) return 'offer';
return selectCdbIdleMoneyPrefillCents(accounts) != null ? 'amount' : 'entry';
```
- `hasIdleMoney(accts)`: existe conta com `Number(balance) > 0` e `subtype !== 'CREDIT_CARD'`.
- `selectCdbIdleMoneyPrefillCents(accts)`: ignora `connectorName === WALLET_CONNECTOR_NAME`, pega a conta elegível de **maior saldo** e calcula `floor(Number(balance)) * 100` (reais inteiros, em centavos). Devolve `null` se o valor não for finito ou se for `< MIN_AMOUNT_CENTS` (100 = R$ 1,00 [I alto]).

`cdbWidgetEntryHref(dest)`:

| dest | href |
|---|---|
| `'amount'` | `{pathname:'/investments/cdb-amount-input', params:{entryPoint:'widget', prefillAmountSource:'idle_money'}}` |
| `'offer'` | `{pathname:'/investments/cdb-offer-modal', params:{entryPoint:'widget'}}` |
| `'entry'` (default) | `{pathname:'/vision/cdb-investment-entry', params:{entryPoint:'widget'}}` |

No Android esse `href` só é usado pelo `Link` do iOS. O toque chama o `onPress` do grid, que navega para o `navigateTo` do registro (`/(protected)/vision/cdb-investment-entry`) [I: o `onWidgetTap` da home não foi rastreado aqui].

### 1.4 Layout (`CdbInvestmentWidget`, L2869001) [O]
Props: `data, size='half', onPress, onNavigationTransitionPressReady, entryHref = CDB_INVESTMENT_ENTRY_HREF, isLoading=false, animate=false`.
`StyleSheet` (L2869660): `card {borderRadius:24, overflow:'visible'}`, `halfCard {height:240}`, `wideCard {minHeight:250}`, `innerContent {flex:1, alignItems:'center', justifyContent:'center', padding:16}`.

```
Box style={{position:'relative', overflow:'visible'}}
 ├─ WidgetGlassCard animate testID="cdb-investment-widget" style=[card, size==='wide' ? wideCard : halfCard]
 │   └─ VStack className="flex-1 px-4 pb-4 pt-4" style={{overflow:'visible', minHeight: size==='wide' ? 178 : 168}}
 │       ├─ HStack className="items-start justify-between gap-3"
 │       │   ├─ VStack className="flex-1 gap-0.5 pr-1"
 │       │   │   ├─ Text "text-2xl font-semibold text-foreground" numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.78 allowFontScaling=false → data.headline
 │       │   │   └─ Text size="sm" "text-muted-foreground" style={{fontSize:14, lineHeight:18}} numberOfLines=2 adjustsFontSizeToFit minimumFontScale=0.82 allowFontScaling=false → data.subtitle
 │       │   └─ Box testID="cdb-investment-widget-illustration-frame" pointerEvents="none" style={{width:72, height:52, position:'relative'}}   ← reserva vazia
 │       └─ Box pointerEvents="none" className="mt-2 flex-1 justify-end"
 │           └─ CdbInvestmentChart {series, comparisonLabel, showComparison: cohort!=='simulated', showMilestones: cohort==='simulated'}
 └─ Box testID="cdb-investment-widget-illustration-overflow-frame" pointerEvents="none"
        style={{position:'absolute', top:-42, right:-30, width:132, height:132, zIndex:2, overflow:'visible'}}
     └─ [só iOS com onNavigationTransitionPressReady: Link.AppleZoom]
         Box testID="cdb-investment-widget-illustration-zoom-source" collapsable={false} pointerEvents="none" style={{width:132, height:132}}
          └─ Image (gluestack) source={flying-money.gif} alt="CDB" resizeMode="contain" className="h-full w-full" testID="cdb-investment-widget-illustration"
```
Medidas (Android, wide): largura `W−32`; altura **mínima 250** (cresce com o conteúdo); raio 24; fundo #18181b; padding 14 (`px-4 pt-4 pb-4`); gap do header 10,5 (`gap-3`); gap título↔subtítulo 1,75; `pr-1` = 3,5; `mt-2` = 7.
A ilustração (132×132) **vaza** 42 px acima e 30 px à direita do cartão e fica por cima do vizinho (zIndex 2). Daí o `contentOverflow:'visible'` do registro [I].
O tamanho `half` existe no código (altura fixa 240), mas o registro só permite `wide`.

Carregando (`isLoading || !data`): `WidgetGlassCard animate testID="cdb-investment-widget" style=[card, sizeCard]` → `View style=innerContent` → `WidgetLoadingState variant="content"`, sem ilustração.

### 1.5 Tipografia

| Texto | Classe/estilo | Fonte | Size/LH | Cor |
|---|---|---|---|---|
| headline | `text-2xl font-semibold text-foreground` | Geist-SemiBold | 21 / 28 (autoshrink até 0,78) | #fafafa |
| subtitle | `Text size=sm` + `text-muted-foreground` + style | Geist-Regular | **14 / 18** (o style vence o `text-sm`=16) | #a1a1aa |
| legenda | `text-xs text-muted-foreground` | Geist-Regular | 10,5 / 14 | #a1a1aa |

### 1.6 Toque e transição [O L2869500–2869580, L2867996]
- Sem `onPress`: sem wrapper.
- **Android** (`IS_IOS` falso): `CdbInvestmentWidgetPressable` = `Pressable className="w-full" style={{overflow:'visible'}} testID="cdb-investment-widget-pressable" onPress={useWidgetPress(onPress)}` (háptico Light e depois `onPress`).
- **iOS** com `onNavigationTransitionPressReady`: `Link href={entryHref} asChild` → `Link.Trigger` → Pressable, e a ilustração vira `Link.AppleZoom` (transição de zoom do expo-router a partir do GIF). O Pressable registra no `useEffect` o callback `onNavigationTransitionPressReady(() => onPress({}))` e passa `null` no unmount.

### 1.7 Ilustração [O L2778836–2778870]
`getCdbInvestmentMoneyIllustrationSource()` devolve o asset `flying-money.gif` (720×720, hash `53a70aeb47e971a4a0b6e06e4340091e`). No APK: `res/drawable-mdpi-v4/assets_images_cdbinvestment_flyingmoney.gif` (6,9 MB). É exibido em 132×132 com `contain`. GIF animado em loop, sem controle de pausa: o `VisionHomeMediaActivity` só gerencia `cdb-position` full e os widgets Leo [O/I].

### 1.8 Gráfico `CdbInvestmentChart` (memo, L2868139) [O]
Props: `series, comparisonLabel, showComparison=true, showMilestones=false`.
Cores (`useThemeStore(s => s.mode === 'dark')`): linha CDB **`#b0ff57`** (dark) / `#95ff48` (light); comparação **`#9ca3af`** (dark) / `#6b7280` (light).

Estado: `width` (via onLayout) e `ready=false`, que vira `true` **1000 ms** depois de montar (`setTimeout`). Enquanto `!ready` ou `series.length <= 1`, renderiza `Box testID="cdb-investment-widget-chart-placeholder" style={{alignSelf:'stretch', height:156, width:'100%'}}` (vazio).

Dados:
```ts
const rows = series.map(p => ({ label: '', value: p.cdb, comparisonValue: p.comparison })); // rótulo do eixo X sempre vazio
const r = showMilestones ? null : rebaseForDomain(rows);
// rebaseForDomain (M6870):
//   all = rows.flatMap(x => [x.value, x.comparisonValue ?? 0]);
//   min = all.length ? Math.min(...all) : 0;  max = all.length ? Math.max(...all) : 1;
//   range = (max - min) || max || 1;  base = Math.max(0, min);  top = max + range * 0.2;
//   data  = rows.map(x => ({ value: x.value - base, label: x.label }));
//   data2 = rows.map(x => ({ value: (x.comparisonValue ?? 0) - base, label: x.label }));
//   maxValue = (top - base) || 1;
//   formatYLabel(v) = formatYAxisCompact(Number(v) + base)  // não é exibido (hideYAxisText)
const milestones = series.map(p => ({
  value: p.cdb, label: p.label,
  text: Math.abs(p.cdb) < 1000 ? `R$${Math.round(p.cdb)}` : `R$${Math.round(p.cdb / 1000)}k`, // sem espaço: "R$950", "R$12k"
}));
const maxMilestone = Math.max(...milestones.map(m => m.value)) * 1.2;
const chartWidth = Math.max(width - 48, 1);
const spacing = series.length > 1 ? (chartWidth - 8) / (series.length - 1) : 0;
```
`formatYAxisCompact(v)` (M6870): `|v| < 1000` → `R$${round(v)}`; senão `R$${(v/1000).toFixed(v % 1000 === 0 ? 0 : 1)}k`.

`LineChart` (só se `width > 0`):
```
data={showMilestones ? milestones : r.data}
{...(showComparison && !showMilestones ? { data2: r.data2 } : {})}
width={chartWidth} spacing={spacing} height={132} xAxisLabelsHeight={24}
maxValue={showMilestones ? maxMilestone : r.maxValue}
endSpacing={8} initialSpacing={0} curved={series.length >= 3}
color={lime} thickness={2}
[modo comparação: color2={gray} thickness2={2} strokeDashArray2={[5,5]}]
areaChart startFillColor={lime} endFillColor="transparent" startOpacity={0.4} endOpacity={0}
startFillColor2="transparent" endFillColor2="transparent" startOpacity2={0} endOpacity2={0}
[modo marcos: showDataPoints showValuesAsDataPointsText dataPointsColor={lime} dataPointsRadius={3}
              textShiftX={0} textShiftY={-8} textColor={lime} textSize={8}]
[senão: hideDataPoints]
hideDataPoints2 noOfSections={3} yAxisLabelWidth={0} yAxisThickness={0} xAxisThickness={0} hideYAxisText
rulesType="dashed" rulesColor={dark ? 'rgba(161,161,170,0.14)' : 'rgba(113,113,122,0.10)'}
rulesThickness={0.5} dashWidth={3} dashGap={3} disableScroll
```
O `height = 132` vem de uma constante de módulo; o registrador indica 132 [I alto]. Não há animação de entrada própria além do atraso de 1 s (a lib pode animar por padrão [I]).

Contêiner e legenda:
```
VStack space="xs" (gap 3,5)
 ├─ View testID="cdb-investment-widget-chart-layout" onLayout={e => setWidth(e.nativeEvent.layout.width)} style={{paddingHorizontal: 24}} → LineChart
 └─ HStack space="md" (gap 10,5)
     ├─ HStack space="xs" className="items-center" → [View {width:12, height:2, borderRadius:2, backgroundColor:lime}, Text "text-xs text-muted-foreground" "CDB Pierre"]
     └─ showComparison && HStack space="xs" className="items-center" → [View 12×2 r2 bg:gray, Text "text-xs text-muted-foreground" {comparisonLabel}]
```
O `paddingHorizontal: 24` vem do mesmo registrador da constante 24 (`xAxisLabelsHeight`) [I alto]. Ele é coerente com `chartWidth = width − 48`.
Strings fixas: `CDB Pierre`, `CDB` (alt).


## 2. `cdb-performance`: "Vencimento CDB" (`half` e `wide`)

Módulos: `CdbPerformanceWidget` (M9052, memo, L2870739; `formatMaturityDateDDMM` L2870595); `CdbPerformanceWidgetConnected` (M9053, L2871690); `useCdbPerformanceData` (M9054, L2871997); `computeCdbVencimento` / `fetchCdbPerformanceDirect` (M9055, L2872108–2872480); `useCdbWidgetIdentityGate` (L2873281).
Também **não usa** `Widget2x2`/`WidgetWide`: o cartão é um `WidgetGlassCard` próprio.
Lembrete: o widget está em `VISION_PICKER_RETIRED_WIDGET_IDS` (§0.1), então não aparece mais no picker.

### 2.1 Dados: busca direta, sem o endpoint da Vision [O]
`useCdbPerformanceData(size='half')` = `useVisionSystemWidgetData('cdb-performance', size, undefined, { directFetcher: fetchCdbPerformanceDirect, accessGate: useCdbWidgetIdentityGate() })`.
- Com `directFetcher`, o `queryFn` do hook comum (M8526, L2718478–2718760) **não chama** `/api/mobile/vision/widgets/...`: chama `directFetcher(size)` e usa `retry:false`. A queryKey, o `staleTime`, o `gcTime: 600000` e o `networkMode:'always'` continuam os mesmos.
- `enabled` também exige `accessGate.isAllowed`. O gate é `useCdbKycStatus()`: `isAllowed = data?.kycApproved === true`, `isPending = isLoading`. **Sem KYC aprovado o widget não busca nada**.

`fetchCdbPerformanceDirect` (L2872258):
```ts
const [summary, list] = await Promise.all([
  cdbBankingApiClient.get('/api/cdb/summary'),
  cdbBankingApiClient.get('/api/cdb/list'),
]);
return {
  data: { ...computeCdbVencimento(list),
          cdi_percentage: summary?.cdi_percentage ?? 0,
          exceeds_maximum_principal: summary?.exceeds_maximum_principal ?? false,
          max_principal_balance: summary?.max_principal_balance ?? 0 },
  aiContext: /* montado a partir dos mesmos dados */,
};
```
`computeCdbVencimento(list)` (L2872108) escolhe **o CDB a prazo que vence primeiro**:
```ts
const items = (list ?? []).filter(isIdentifiedTermCdbListItem);
// isIdentifiedTermCdbListItem (L1740264): is_daily_liquidity !== true && id numérico finito
//   && asset_creation_date e maturity_date strings não vazias
if (!items.length) return { id: undefined, sum_net_position: 0, sum_net_yield: 0, maturityDate: null,
  assetCreationDate: null, termMonths: 0, daysUntilMaturity: 0, monthsUntilMaturity: 0, progressPercent: 0 };
const it = [...items].sort((a, b) => a.maturity_date.localeCompare(b.maturity_date))[0];
const today = new Date(); const todayISO = `${yyyy}-${mm}-${dd}`;            // data local
const d = (s) => new Date(s.slice(0, 10) + 'T00:00:00');
let monthsLeft = (d(it.maturity_date).getFullYear() - d(todayISO).getFullYear()) * 12
               + (d(it.maturity_date).getMonth() - d(todayISO).getMonth());
if (d(it.maturity_date).getDate() < d(todayISO).getDate()) monthsLeft -= 1;
return {
  id: it.id, sum_net_position: it.net_position, sum_net_yield: it.net_yield,       // centavos
  maturityDate: it.maturity_date, assetCreationDate: it.asset_creation_date,
  termMonths: monthsBetweenDates(it.asset_creation_date, it.maturity_date),        // max(0, Δanos*12 + Δmeses), ignora o dia
  daysUntilMaturity: Math.max(0, daysUntil(it.maturity_date)),                     // round((venc − hoje 00:00)/86400000)
  monthsUntilMaturity: Math.max(0, monthsLeft),
  progressPercent: elapsedTermPercent(it.asset_creation_date, it.maturity_date, today),
};
// elapsedTermPercent (L1789745): total = round((venc − criação)/dia); se total <= 0 → 100;
//   decorrido = round((hoje − criação)/dia); clamp(decorrido/total*100, 0, 100)
```
Apesar do nome `sum_*`, os valores são **de um único CDB** (o de vencimento mais próximo) [O].

### 2.2 Estados (`Connected`) [O L2871690–2871950]
| Condição | Render |
|---|---|
| `isError` | `View testID="cdb-performance-error" className="rounded-3xl bg-surface" style={S}` → `WidgetErrorState compact` |
| `isFetched && !data` | `View testID="cdb-performance-empty" className="rounded-3xl bg-surface" style={S}` → `WidgetEmptyStateSlot {emptyState, compact: size==='half'}` |
| senão | `CdbPerformanceWidget {data, size, onPress: onPressWrapped, isLoading: !data && (isLoading \|\| !isFetched), emptyState, animate}` |

`S`: half `{height:200, alignItems:'center', justifyContent:'center', padding:16}`, wide `{minHeight:210, …}`.
`onPressWrapped` só existe se `data.id !== undefined`: `() => onPress?.({ params: { id: String(data.id), from: 'vision' } })`. Com o `navigateTo` do registro, o destino é `/(protected)/investments/cdb-investment-details?id=<id>&from=vision` [I: a junção params+rota é feita pelo `onWidgetTap` da home].

Dentro do componente [O L2870830–2871560]:
- `isLoading` → `WidgetGlassCard animate testID="cdb-performance-widget" style=[card, sizeCard]` → `View style=innerContent` → `WidgetLoadingState variant="content"`.
- `sum_net_position <= 0` → `WidgetGlassCard (mesmo)` → `WidgetEmptyStateSlot {emptyState, compact: size!=='wide'}` (sem o View interno). Isso cobre o caso "sem CDB a prazo" (o `computeCdbVencimento` devolve zeros).
- senão → conteúdo (2.4/2.5), embrulhado por `WidgetGlassCard animate testID="cdb-performance-widget" style=[card, sizeCard]`.
- Wrapper de toque (`t4`): com `onPress`, `Pressable testID="cdb-performance-widget-pressable" style={{overflow:'visible'}} onPress={useWidgetPress(onPress)}` (háptico Light). Sem `onPress`, nada.

`StyleSheet` (L2871660): `card {borderRadius:24, overflow:'visible'}`, `halfCard {height:200}`, `wideCard {minHeight:210}`, `innerContent {flex:1, alignItems:'center', justifyContent:'center', padding:16}`.

### 2.3 Textos derivados [O L2870830–2871040]
Cores de `useThemeColors()`: `pierre` (**#c7ff00**), `mutedForeground` (#a1a1aa), `foreground` (#fafafa). `hidden = useHideValues()`.
```ts
const positionText = formatCentsToBRL(sum_net_position);               // M4419→M4430 [I]: "R$ 12.345,67"
const arrow = sum_net_yield >= 0 ? '↑' : '↓';
const yieldText = `${arrow} ${formatCentsToBRL(Math.abs(sum_net_yield))}`;
const term = termMonths <= 0 ? ''
  : size !== 'wide' ? `${termMonths}m`                                 // half: "12m"
  : termMonths === 1 ? '1 mês' : `${termMonths} meses`;                // wide: "12 meses"
const title = `CDB Pierre${term ? ' ' + term : ''}`.trim();            // "CDB Pierre 12m" / "CDB Pierre 12 meses"
const remaining =
  monthsUntilMaturity >= 1
    ? (monthsUntilMaturity === 1 ? 'Falta 1 mês para o resgate' : `Faltam ${monthsUntilMaturity} meses para o resgate`)
    : daysUntilMaturity > 1 ? `Faltam ${daysUntilMaturity} dias para o resgate`
    : daysUntilMaturity === 1 ? 'Falta 1 dia para o resgate'
    : 'Resgate disponível hoje';
const dateDDMM = formatMaturityDateDDMM(maturityDate);                 // "2027-03-15" → "15/03"; falsy → ''
const pct = Math.min(Math.max(progressPercent, 0), 100);               // largura da barra: `${pct}%`
```

### 2.4 Layout `half` [O L2871104–2871235]
Cartão: `(W−50)/2 × 200`, raio 24, #18181b.
```
VStack className="flex-1 px-4 pb-4 pt-4" style={{justifyContent:'space-between', minHeight:168}}
 ├─ Text "text-base" style={{color: mutedForeground}} numberOfLines=1 allowFontScaling=false testID="cdb-performance-widget-title" → title
 ├─ VStack style={{gap:0}}
 │   ├─ PrivacyCurrencyText valueCents={sum_net_position} hidden fontSize={20} fontWeight="800" color={foreground}
 │   │      minimumFontScale={0.6} disableMinWidth compactOnOverflow testID="cdb-performance-widget-value"
 │   └─ HStack className="items-center gap-1"                                   // gap 3,5
 │       ├─ PierreIcon name={sum_net_yield >= 0 ? 'arrowUp' : 'arrowDown'} size={14} color={pierre}
 │       └─ View className="min-w-0 flex-1"
 │           └─ PrivacyCurrencyText valueCents={Math.abs(sum_net_yield)} hidden fontSize={16} color={pierre}
 │                  disableMinWidth testID="cdb-performance-widget-earnings"
 └─ VStack style={{gap:8}}
     ├─ Text "text-sm" style={{color: mutedForeground}} numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.8 allowFontScaling=false
     │       testID="cdb-performance-widget-time-pill" → remaining
     ├─ View className="bg-border" testID="cdb-performance-widget-progress-track" style={{height:12, width:'100%', borderRadius:100, overflow:'hidden'}}
     │   └─ View testID="cdb-performance-widget-progress-fill" style={{width:`${pct}%`, height:12, borderRadius:16, backgroundColor: pierre}}
     └─ Text "text-sm" (mesmas props) testID="cdb-performance-widget-date-pill" → dateDDMM
```
Observações:
- O valor e o rendimento usam `PrivacyCurrencyText` com `animate` default (`true`). Na prática saem pelo texto nativo `PierreAnimatedCurrencyText` com `durationMs:0` (sem contagem), ou seja, na fonte do sistema (ver 00-base §11.4). Se o valor não couber em `getHalfWidgetContentWidth`, o valor principal vira texto compacto (`R$ 12.346` ou `R$ 1,2 mi`) em `PrivacyText` com `lineHeight: ceil(20×1,18) = 24`.
- O rendimento oculto mostra `R$ ••••`. A seta continua visível e colorida.
- Cores: trilho `bg-border` = #27272a; preenchimento sólido #c7ff00.

| Texto | Fonte | Size/LH | Cor |
|---|---|---|---|
| título | Geist-Regular (`text-base`) | 14 / 21 | #a1a1aa |
| valor | nativo, peso 800 | 20 / 23,6 (`fontSize×1,18`) | #fafafa |
| rendimento | nativo, peso 400 | 16 / 18,88 | #c7ff00 |
| "Faltam…" / "DD/MM" | Geist-Regular (`text-sm`) | 16 / padrão do RN (autoshrink 0,8) | #a1a1aa |

### 2.5 Layout `wide` [O L2871236–2871395]
Cartão: `(W−32)`, altura mínima 210 (o conteúdo pede ≥178 + padding), raio 24, #18181b.
```
VStack className="flex-1 p-4" style={{gap:16, minHeight:178}}
 ├─ HStack className="items-start justify-between"
 │   ├─ Text "text-sm flex-1" style={{color: mutedForeground}} numberOfLines=1 allowFontScaling=false testID="cdb-performance-widget-title" → title
 │   └─ Box collapsable={false} className="rounded-xl border border-border" style={{width:40, height:40}}
 │       └─ Box collapsable={false} className="rounded-xl overflow-hidden" style={StyleSheet.absoluteFill}
 │           └─ expo-image Image source={leo.webp} style={StyleSheet.absoluteFill} contentFit="cover" testID="cdb-performance-widget-brand"
 ├─ VStack style={{gap:8}}
 │   ├─ PrivacyText "text-3xl font-bold text-foreground" numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.7 allowFontScaling=false
 │   │      testID="cdb-performance-widget-value" value={hidden ? 'R$ ••••' : positionText}
 │   └─ Text "text-sm" style={{color: pierre}} allowFontScaling=false testID="cdb-performance-widget-earnings"
 │          → hidden ? 'R$ •••• desde a aplicação' : `${yieldText} desde a aplicação`   // "↑ R$ 123,45 desde a aplicação"
 ├─ View className="bg-border" testID="cdb-performance-widget-progress-track" style={{height:18, width:'100%', borderRadius:16, overflow:'hidden'}}
 │   └─ LinearGradient colors={['#cfff04','#d6ff2b','#e1ff63']} start={{x:0,y:0}} end={{x:1,y:0}}
 │          style={{width:`${pct}%`, height:18, borderRadius:24}}
 └─ HStack className="items-center gap-2"                                       // gap 7
     ├─ Icon as={ClockIcon} size="sm" style={{color: mutedForeground}}           // gluestack; sm ≈ 14×14 com rem 14 [I]
     └─ Text "text-base font-medium" style={{color: mutedForeground}} allowFontScaling=false testID="cdb-performance-widget-time-text" → remaining
```
- No wide **não há** a data DD/MM; o gradiente horizontal substitui a cor sólida.
- O avatar "brand" é a imagem do agente **Leo**: `leo.webp` (1250×1250, hash `39c98c0eeaeda977fd47efa9b6c57497`, APK `res/drawable-mdpi-v4/src_assets_images_agents_leo.webp`), recortada em 40×40 com raio 10,5 (`rounded-xl`) e borda 1 px #27272a.

| Texto | Fonte | Size/LH | Cor |
|---|---|---|---|
| título | Geist-Regular (`text-sm`) | 16 / padrão | #a1a1aa |
| valor | Geist-Bold (`text-3xl font-bold`) | 26,25 / 31,5 (autoshrink 0,7) | #fafafa |
| rendimento | Geist-Regular (`text-sm`) | 16 / padrão | #c7ff00 |
| "Faltam…" | Geist-Medium (`text-base font-medium`) | 14 / 21 | #a1a1aa |

Strings: `CDB Pierre`, `1 mês`, ` meses`, `m`, `Resgate disponível hoje`, `Falta 1 dia para o resgate`, `Faltam N dias para o resgate`, `Falta 1 mês para o resgate`, `Faltam N meses para o resgate`, ` desde a aplicação`, `↑`, `↓`.


## 3. `cdb-position`: "Posição CDB" (`half` e `full`)

Módulos: `CdbPositionWidget` (M9058, memo, L2875036) com `ChartTouchShield` (L2873391), `buildCdbPositionAxisMarkers` (L2873432), `CdbPositionHalf` (L2873517), `CdbPositionWide` (L2873971, usado no **full**), `resolveChartYAxisMin` (L2874713) e `CdbPositionNativeAreaChart` (L2874858); `CdbPositionWidgetConnected` (L2875783); `useCdbPositionData` (M9064, L2876131); `fetchCdbPositionDirect` (M9055, L2872498); formatação dos dados em `shapeCdbPositionData` (M4575, L1829570) e `buildPortfolioEvolutionSeries` (M4574, L1829242).
Constantes do módulo [O L2874840–2874856]: `CDB_POSITION_WIDGET_RADIUS = 24`, altura half `200`, altura do gráfico half `80`, `labelAreaHeight` do full `44`, slug do vídeo `'leo-cdb'`, `CDB_POSITION_AVATAR_SIZE = 40` (de `CDB_POSITION_MEDIA_RECT = {offsetTop:16, height:40}`, L2875681).
`StyleSheet` (L2875355): `card {borderRadius:24}`, `flexOne {flex:1}`, `halfChartBody {alignSelf:'stretch', width:'100%'}`, `wideChartBody {flex:1, justifyContent:'flex-end'}`, `fullCard {minHeight:384}`, `halfSkeleton {height:200}`, `fullSkeleton {minHeight:384}`, `innerContent {flex:1, alignItems:'center', justifyContent:'center', padding:16}`.

### 3.1 Dados: busca direta e montagem no app [O]
`useCdbPositionData(size='full')` = `useVisionSystemWidgetData('cdb-position', size, undefined, { directFetcher: fetchCdbPositionDirect, accessGate: useCdbWidgetIdentityGate() })`. Valem o mesmo gate de KYC e o `retry:false` de §2.1. O `Connected` passa `size==='half' ? 'half' : 'full'`.
Erros: `captureException(err, {widget_id:'cdb-position'})`, exceto `CdbBankingApiError` com `code` em `NETWORK_ERROR | TIMEOUT | AUTH_ERROR`; erro que não é `Error` vira `'[CdbPositionWidget] data query failed'`.

`fetchCdbPositionDirect` (L2872498):
```ts
const [summary, list, monthlyBalances, movements, history] = await Promise.all([
  cdbBankingApiClient.get('/api/cdb/summary'),
  cdbBankingApiClient.get('/api/cdb/list'),
  cdbBankingApiClient.get('/api/cdb/pierre/monthly-closing-balances').then(r => r?.balances ?? null).catch(() => null),
  getCompleteDailyLiquidityMovements().then(r => r?.movements ?? null).catch(() => null),
  cdbBankingApiClient.get('/api/cdb/pierre/history/list?page=1&page_size=100').then(r => r?.cdbs ?? null).catch(() => null),
]);
const empty = !list || list.length === 0 || (summary?.sum_net_position ?? 0) <= 0;
const shaped = empty ? null : shapeCdbPositionData(summary, list, monthlyBalances);
const evo = shaped ? buildPortfolioEvolutionSeries({ list, movements, history, monthlyBalances, totalBalanceCents: shaped.totalBalance }) : null;
return { data: shaped ? (evo ? { ...shaped, ...evo } : shaped) : null, aiContext };
```

`shapeCdbPositionData(summary, list, monthlyBalances)` (L1829570) devolve (valores em centavos):

| Campo | Cálculo |
|---|---|
| `productName` | `'CDB Pierre'` (constante) |
| `totalBalance` | `summary.sum_net_position` |
| `investedAmount` | `summary.sum_net_position − summary.sum_net_yield` |
| `totalEarnings` | `list.length ? Σ(item.accrued_yield ?? item.net_yield) : summary.sum_net_yield` |
| `profitPercent` | `invested > 0 ? totalEarnings / invested × 100 : 0` (não exibido na Vision) |
| `cdiPercent` | itens a prazo (`isIdentifiedTermCdbListItem`): prazo de cada um = o mais próximo de `[1, 6, 12, 24]` meses. Se **todos** caem no mesmo prazo: `{1:102, 6:106, 12:111.11, 24:111.11}[prazo] ?? summary.cdi_percentage ?? 0`; prazos diferentes → `null`; sem itens a prazo → `summary.cdi_percentage ?? 0` |
| `extraOverCdi` | `totalEarnings <= 0 \|\| cdiPercent == null \|\| cdiPercent <= 100 ? 0 : round(totalEarnings − totalEarnings × 100 / cdiPercent)` = parte do rendimento acima de 100% do CDI |
| `termLabel` | prazo entre a menor `asset_creation_date` e a maior `maturity_date`: `''`, `'1 mês'` ou `'N meses'` (não exibido) |
| `positionCount` | `list.length` |
| `series`, `startLabel`, `endLabel` | série de saldos de fechamento mensal (`buildDailyLiquidityHistorySeries(monthlyBalances, totalBalance, invested)`) |
| `comparisonSeries` | sempre `[]` |

`buildPortfolioEvolutionSeries` (L1829242) **substitui** `series/startLabel/endLabel` quando consegue montar a evolução:
- Fluxos [O L1828819–1829000]: movimentos da liquidez diária (`DEPOSIT` → `+value`, `REDEMPTION` → `−value`, por `date_time`); títulos a prazo (`+ (net_position − net_yield)` em `asset_creation_date`); histórico liquidado (`+principal` na criação e `−net_position` em `redemption_date`).
- `buildCdbEvolutionPoints` (L1828605): fica só com datas `YYYY-MM-DD` válidas, `<= hoje` e `amount ≠ 0`; começa no primeiro fluxo positivo; acumula mês a mês (usando `monthlyBalances` quando existe); arredonda e limita em `≥ 0`; remove pontos consecutivos iguais; fecha com `{date: hoje, value: max(0, round(totalBalanceCents))}`. Exige `totalBalanceCents` finito [O parcial: os helpers internos de agregação mensal não foram lidos linha a linha].
- Rótulos: `formatCdbChartMonthLabel(date)` = nome do mês por extenso (`Janeiro … Dezembro`, via `date.slice(5,7)`); o **último ponto vira `'Hoje'`**; `startLabel = series[0].label || 'Hoje'`; `endLabel = 'Hoje'`.

### 3.2 Estados [O L2875783–2876070, L2875036–2875340]
`Connected` (props `size='full', onPress, emptyState, animate=false`; também lê `isMediaActive = useVisionHomeMediaActive()`):

| Condição | Render |
|---|---|
| `isError` | `View testID="cdb-position-error" className="rounded-3xl bg-surface" style={S}` → `WidgetErrorState compact={size==='half'}` |
| `isFetched && !data` | `View testID="cdb-position-empty" className="rounded-3xl bg-surface" style={S}` → `WidgetEmptyStateSlot {emptyState, compact: size==='half'}` |
| senão | `CdbPositionWidget {data, size, onPress, isLoading: isLoading \|\| !isFetched, emptyState, animate, isMediaActive}` |

`S`: half `{height:200, alignItems:'center', justifyContent:'center', padding:16}`, full `{minHeight:340, …}`. Analytics: `trackCdbWidgetRendered({widget_id:'cdb-position', render_state: 'error'|'empty'|'shown'})`, uma vez por estado.

`CdbPositionWidget`:
- `isLoading` → `WidgetGlassCard animate testID="cdb-position-widget-loading" style=[card, half ? halfSkeleton : fullSkeleton]` → `View style=innerContent` → `WidgetLoadingState variant={half ? 'value' : 'content'}`.
- `!data`, ou (só no full) `data.totalBalance <= 0` → `WidgetGlassCard animate testID="cdb-position-widget-empty" style=[card, skeleton]` → `WidgetEmptyStateSlot {emptyState, compact: half}`.
- half → `CdbPositionHalf {data, onPress, animate, emptyState}` (o próprio componente cuida do toque via `WidgetShell`).
- full → `CdbPositionWide {data, animate, isMediaActive}`, embrulhado em `Pressable className="w-full" testID="cdb-position-widget-pressable" onPress={useWidgetPress(onPress)}` se houver `onPress`.
- Toque: `onPress` sem parâmetros → `navigateTo` do registro = `/vision?section=investments` (aba Investimentos da própria Vision) [I: navegação feita pelo `onWidgetTap`].

### 3.3 Layout `half` (`CdbPositionHalf`) [O L2873517–2873960]
Cores: `pierre` #c7ff00, `mutedForeground` #a1a1aa, `foreground` #fafafa; `hidden = useHideValues()`.
```
WidgetShell inner useGlass(false no Android) animate glassStyle={{borderRadius:24, height:200}} widgetRadius={24}
            onPress handlePress={useWidgetPress(onPress)} testID="cdb-position-widget"
 └─ VStack className="p-4 gap-2 bg-surface" style={[{borderRadius:24, height:200}]} testID="cdb-position-widget"
     ├─ Text "text-base text-muted-foreground" numberOfLines=1 allowFontScaling=false testID="cdb-position-widget-title" → productName ("CDB Pierre")
     ├─ totalBalance <= 0
     │   ? Box "flex-1 items-center justify-center" → WidgetEmptyStateSlot {emptyState, compact:true}
     │   : ChartTouchShield (View pointerEvents="none" testID="cdb-position-chart-touch-shield") style=[{flex:1}, {alignSelf:'stretch', width:'100%'}]
     │       └─ series.length < 2
     │            ? View testID="cdb-position-chart-placeholder" style={{height:80}}
     │            : CdbPositionNativeAreaChart {series, comparisonSeries, lineColor:pierre, comparisonColor:mutedForeground,
     │                                          yAxisMin, height:80}                       // sem marcadores de eixo
     └─ totalBalance > 0 &&
         VStack className="gap-0.5"                                                       // 1,75
          ├─ PrivacyCurrencyText valueCents={totalBalance} hidden fontSize={20} fontWeight="800" color={foreground}
          │      minimumFontScale={0.6} disableMinWidth compactOnOverflow testID="cdb-position-widget-value"
          └─ HStack className="items-center gap-1"                                        // 3,5
              ├─ PierreIcon name="arrowUp" size={14} color={pierre}                       // sempre "up"
              └─ View className="min-w-0 flex-1"
                  └─ PrivacyCurrencyText valueCents={totalEarnings} hidden fontSize={16} color={pierre}
                         minimumFontScale={0.7} disableMinWidth testID="cdb-position-widget-profit"
```
`yAxisMin = resolveChartYAxisMin(data)`: `min = menor value finito da série`; se não houver → `investedAmount`; senão `max(0, min(investedAmount, min))`.
Tipografia: título Geist-Regular 14/21 #a1a1aa; valor nativo peso 800, 20 (#fafafa); lucro nativo 16 (#c7ff00). Ver §2.4 sobre o `PrivacyCurrencyText`.
O half **não tem** chevron, vídeo nem legenda.

### 3.4 Layout `full` (`CdbPositionWide`) [O L2873971–2874680]
```
WidgetGlassCard animate testID="cdb-position-widget" style=[{borderRadius:24}, {minHeight:384}]
 └─ VStack className="flex-1 p-4 gap-4"                                                  // padding 14, gap 14
     ├─ HStack className="items-start justify-between gap-3"
     │   ├─ VStack className="flex-1 gap-1"                                              // gap 3,5
     │   │   ├─ Text "text-sm text-muted-foreground" numberOfLines=1 allowFontScaling=false testID="cdb-position-widget-title" → productName
     │   │   ├─ PrivacyText "text-2xl font-bold text-foreground" numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.7
     │   │   │      testID="cdb-position-widget-balance" value={hidden ? 'R$ ••••' : formatCentsToBRL(totalBalance)}
     │   │   └─ HStack className="items-center gap-1.5" testID="cdb-position-widget-earnings"   // gap 5,25
     │   │       ├─ PierreIcon name="arrowUp" size={14} color={pierre}
     │   │       ├─ PrivacyText "text-sm" style={{color:pierre}} numberOfLines=1 value={hidden ? 'R$ ••••' : formatCentsToBRL(totalEarnings)}
     │   │       └─ extraOverCdi > 0 && Text "text-xs text-muted-foreground" numberOfLines=1 testID="cdb-position-widget-extra"
     │   │              → `${hidden ? 'R$ ••••' : formatCentsToBRL(extraOverCdi)} a mais que o CDI`
     │   └─ Box collapsable={false} className="rounded-xl border border-border" style={{width:40, height:40}}
     │       └─ Box collapsable={false} className="rounded-xl overflow-hidden" style={StyleSheet.absoluteFill}
     │           ├─ expo-image Image source={avatar estático do Leo} style={absoluteFill} contentFit="cover" testID="cdb-position-widget-avatar"
     │           └─ AgentAnimatedVideoLayer source={media.source} visible={media.visible} loop resetOnPause={false} style={absoluteFill}
     ├─ ChartTouchShield style={{flex:1, justifyContent:'flex-end'}}
     │   └─ series.length < 2
     │        ? View testID="cdb-position-chart-placeholder" style={{height:180}}
     │        : CdbPositionNativeAreaChart {series, comparisonSeries, lineColor:pierre, comparisonColor:mutedForeground, yAxisMin,
     │                                      height:180, axisMarkers: buildCdbPositionAxisMarkers(data), labelAreaHeight:44, bottomVisualPadding:16}
     └─ HStack className="items-center gap-4" testID="cdb-position-widget-legend"
         └─ HStack className="items-center gap-2"
             ├─ View style={{width:8, height:8, borderRadius:4, backgroundColor:pierre}}
             └─ Text "text-xs text-foreground" allowFontScaling=false → "Seu investimento"
```
Avatar animado [O L2874000–2874030, L2875475]:
- `src = getAgentVideoSourceBySlug('leo-cdb')`. Ele só é usado se `!isLowEndDevice() && canRenderAgentVideo(src)` (`src != null`); senão fica `undefined` e aparece só a imagem estática.
- `useVisionAnimatedMediaSource(src, isMediaActive)` → `{source: isMediaActive ? src : undefined, visible: isMediaActive && src != null}`.
- `isMediaActive` vem do `VisionHomeMediaActivityProvider`, que fica ativo quando o widget está visível no viewport: `isVisionWidgetInMediaViewport` exige `visível ≥ min(80, altura/2)` px. Só o `cdb-position` **full** registra retângulo de mídia (`CDB_POSITION_MEDIA_RECT`; o half devolve `null`, L2875425).
- Vídeo: `res/raw/src_assets_videos_agents_leo.mp4` [I alto: mesmo slug do agente Leo]. Imagem estática (sempre por baixo do vídeo): `leo.webp` (M4716, último item da lista de dependências do módulo 9058, L2873383; APK `res/drawable-mdpi-v4/src_assets_images_agents_leo.webp`) [O].

Marcadores do eixo X (`buildCdbPositionAxisMarkers`, L2873432): com `< 2` pontos → `[]`. Senão `start = startLabel || series[0].label || ''` e `end = endLabel || series[last].label || 'Vencimento'`; adiciona `{x:0, label:start}` se `start` e `{x:last, label:end}` se `end !== start`. Na prática fica "Janeiro … Hoje".

Tipografia (full):

| Texto | Fonte | Size/LH | Cor |
|---|---|---|---|
| título "CDB Pierre" | Geist-Regular (`text-sm`) | 16 / padrão | #a1a1aa |
| saldo | Geist-Bold (`text-2xl font-bold`) | 21 / 28 (autoshrink 0,7) | #fafafa |
| rendimento | Geist-Regular (`text-sm`) | 16 / padrão | #c7ff00 |
| "… a mais que o CDI" | Geist-Regular (`text-xs`) | 10,5 / 14 | #a1a1aa |
| legenda "Seu investimento" | Geist-Regular (`text-xs text-foreground`) | 10,5 / 14 | #fafafa |

`formatCentsToBRL` aqui é o de M4419 (reexporta M4430 [I]): `[-]R$ {inteiro pt-BR},{cc}` montado à mão.

### 3.5 Gráfico `CdbPositionNativeAreaChart` (memo, L2874858) [O]
Usa o componente nativo **`PierreAreaChart`** (M4584, `requireNativeView('PierreCharts')`). A geometria nativa está em `widgets/02-gastos.md`.
```tsx
<PierreAreaChart
  currentPoints={series.map((p, i) => ({ x: i, value: p.value }))}
  previousPoints={comparisonSeries?.length > 0 ? comparisonSeries.map((p, i) => ({ x: i, value: p.value })) : undefined} // na prática vazio
  axisMarkers={axisMarkers ?? []}
  animated={false} loadingMode="none" hapticMode="none" performanceMode="reduced"
  lineColor={pierre} fillColor={pierre} dotColor={pierre} comparisonColor={mutedForeground}
  axisColor="#27272A" labelColor="#A1A1AA"
  labelAreaHeight={labelAreaHeight ?? 0}            // half 0 · full 44
  topVisualPadding={4} bottomVisualPadding={bottomVisualPadding ?? 4}   // half 4 · full 16
  leftInset={0} rightInset={8} endContentPadding={8}
  yAxisMin={yAxisMin}
  style={{ height: height ?? 80, width: '100%' }}   // half 80 · full 180
/>
```
O gráfico não tem animação (`animated={false}`) nem háptico. O `ChartTouchShield` (`pointerEvents="none"`) impede o gráfico de capturar toques: o toque vai para o card.
Strings: `CDB Pierre`, `Seu investimento`, ` a mais que o CDI`, `Hoje`, `Vencimento`, meses `Janeiro … Dezembro`.

### 3.6 Variante "Pierre One" (fora da Vision) [O]
`fetchCdbDailyLiquidityPositionDirect` (L2872936), `shapeCdbDailyLiquidityPositionData` e `CDB_POSITION_DAILY_LIQUIDITY_VARIANT` servem o widget `cdb-position` da **home do cartão** (`productName 'CDB Pierre One'`, `positionCount: 1`; resumo "Widget de posição do CDB de liquidez diária que garante o limite do cartão Pierre One", L2873261). Não é usada pela Vision.


## 4. Snippet de referência: `cdb-performance` half (RN puro, caminho Android)

Usa `Widget`/`PrivacyText`/tokens de `00-base.md` §15. Os valores do PrivacyCurrencyText foram simplificados para `Text` com `formatBRLExact`.
```tsx
import React from 'react';
import { Pressable, Text, View } from 'react-native';
import * as Haptics from 'expo-haptics';
import { ArrowUp, ArrowDown } from 'lucide-react-native'; // troque pelos seus ícones

const C = { surface: '#18181b', fg: '#fafafa', muted: '#a1a1aa', pierre: '#c7ff00', border: '#27272a' };
const HIDDEN = 'R$ ••••';
const brl = (c: number) => { const s = c < 0 ? '-' : ''; const a = Math.round(Math.abs(c));
  return `${s}R$ ${Math.floor(a / 100).toLocaleString('pt-BR')},${String(a % 100).padStart(2, '0')}`; };

type Vencimento = { id?: number; sum_net_position: number; sum_net_yield: number; maturityDate: string | null;
  termMonths: number; daysUntilMaturity: number; monthsUntilMaturity: number; progressPercent: number };

export function CdbPerformanceHalf({ d, hidden, onPress }: { d: Vencimento; hidden: boolean; onPress?: () => void }) {
  const term = d.termMonths <= 0 ? '' : `${d.termMonths}m`;
  const title = `CDB Pierre${term ? ' ' + term : ''}`.trim();
  const m = d.monthsUntilMaturity, days = d.daysUntilMaturity;
  const remaining = m >= 1 ? (m === 1 ? 'Falta 1 mês para o resgate' : `Faltam ${m} meses para o resgate`)
    : days > 1 ? `Faltam ${days} dias para o resgate` : days === 1 ? 'Falta 1 dia para o resgate' : 'Resgate disponível hoje';
  const ddmm = d.maturityDate ? `${d.maturityDate.split('-')[2]}/${d.maturityDate.split('-')[1]}` : '';
  const pct = Math.min(Math.max(d.progressPercent, 0), 100);
  const Arrow = d.sum_net_yield >= 0 ? ArrowUp : ArrowDown;
  const card = (
    <View style={{ height: 200, borderRadius: 24, backgroundColor: C.surface, overflow: 'visible' }}>
      <View style={{ flex: 1, padding: 14, justifyContent: 'space-between', minHeight: 168 }}>
        <Text allowFontScaling={false} numberOfLines={1} style={{ fontFamily: 'Geist-Regular', fontSize: 14, lineHeight: 21, color: C.muted }}>{title}</Text>
        <View>
          <Text allowFontScaling={false} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.6}
            style={{ fontSize: 20, lineHeight: 24, fontWeight: '800', color: C.fg }}>{hidden ? HIDDEN : brl(d.sum_net_position)}</Text>
          <View style={{ flexDirection: 'row', alignItems: 'center', columnGap: 3.5 }}>
            <Arrow size={14} color={C.pierre} />
            <View style={{ minWidth: 0, flex: 1 }}>
              <Text allowFontScaling={false} numberOfLines={1} style={{ fontSize: 16, lineHeight: 19, color: C.pierre }}>
                {hidden ? HIDDEN : brl(Math.abs(d.sum_net_yield))}</Text>
            </View>
          </View>
        </View>
        <View style={{ rowGap: 8 }}>
          <Text allowFontScaling={false} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.8}
            style={{ fontFamily: 'Geist-Regular', fontSize: 16, color: C.muted }}>{remaining}</Text>
          <View style={{ height: 12, width: '100%', borderRadius: 100, overflow: 'hidden', backgroundColor: C.border }}>
            <View style={{ width: `${pct}%`, height: 12, borderRadius: 16, backgroundColor: C.pierre }} />
          </View>
          <Text allowFontScaling={false} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.8}
            style={{ fontFamily: 'Geist-Regular', fontSize: 16, color: C.muted }}>{ddmm}</Text>
        </View>
      </View>
    </View>
  );
  if (!onPress) return card;
  return <Pressable style={{ overflow: 'visible' }} onPress={() => { Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light); onPress(); }}>{card}</Pressable>;
}
```

## 5. Dúvidas abertas
1. **Altura do gráfico do `cdb-investment` (132)** e o `paddingHorizontal: 24` vêm de constantes de módulo atribuídas por registrador (`r0` reaproveitado). A leitura é coerente (`chartWidth = width − 48`), mas não é literal [I alto].
2. **`onWidgetTap` da home**: não confirmei como o grid junta `params` (`{id, from:'vision'}` do `cdb-performance`) com o `navigateTo` do registro, nem se o `cdb-investment` no Android usa o `navigateTo` fixo ou o `entryHref` dinâmico (`amount/offer/entry`). No código lido, o `entryHref` só entra no `Link` do iOS.
3. **`buildCdbEvolutionPoints`**: os helpers de agregação mensal (`_closure0_slot10/11/12`, M4574) não foram lidos linha a linha. A forma exata da série do `cdb-position` (um ponto por mês? interpolação com `monthlyBalances`?) é [I].
4. **`buildDailyLiquidityHistorySeries`** (série inicial de `shapeCdbPositionData`, L1829312) não foi detalhada: na Vision ela é quase sempre substituída por `buildPortfolioEvolutionSeries`.
5. **`typeof` em `isIdentifiedTermCdbListItem`**: o decompilador mostra `'bigint'`/`'boolean'` onde o código-fonte quase certamente tem `'number'`/`'string'` (tabela de `typeof` da v98 desalinhada) [I alto].
6. **GIF no Android**: o `Image` do gluestack (RN `Image`) anima GIF no Android só com o suporte do Fresco a GIF animado (o Expo costuma incluir) [I]. O arquivo tem 6,9 MB.
7. **`ClockIcon` size `sm`**: o tamanho exato depende do `Icon` do gluestack (sm = `h-4 w-4` = 14 px com rem 14) [I].
8. Payload do `cdb-investment` (`headline`, `subtitle`, `cohort`, `series[].cdb/comparison/label`, `comparisonLabel`) vem do servidor. Os textos reais e os valores de `cohort` além de `'simulated'` não estão no APK.
9. `aiContext` dos fetchers diretos não foi detalhado (serve aos agentes de IA, fora do escopo visual).
