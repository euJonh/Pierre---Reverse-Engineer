# W2 — Widgets `monthly-spending` ("Gastos do mês") e `spending-by-category` ("Gastos por categoria") (Pierre v4.0.6)

> Análise estática do bundle Hermes v98 (`_trabalho/out.js`) e, quando indicado, do dex nativo.
> **[O]** = observado no código (`Lnnnn` = linha de `out.js`; `Mnnnn` = id de módulo Metro); **[I]** = inferido.
> Unidades: px = dp. NativeWind com **rem = 14**. Fundação (Widget2x2/WidgetWide/WidgetFull, WidgetShell, estados vazio/erro/carregando,
> PrivacyText/PrivacyCurrencyText, tokens dark, tabela de classes NativeWind) está em **`w0-base.md`** (versão final: `widgets/00-base.md`), referenciada como `W0 §n`.
> Busca de dados comum (`useVisionSystemWidgetData`, `GET /api/mobile/vision/widgets/{id}/data`) está em **`w1-wallet-saldo-fluxo.md` §1** (versão final: `widgets/01-carteira-saldo-fluxo.md` §1), referenciada como `W1 §1`.

---

## 1. `monthly-spending` — identidade e camada "Connected"

### 1.1 Registro (`VISION_WIDGET_REGISTRY`, L772549) [O]
| Campo | Valor |
|---|---|
| id | `monthly-spending` |
| label | `Gastos do mês` |
| size (padrão) | `wide` |
| supportedSizes | `['half', 'wide', 'full']` |
| chartVariants | `[{id:'area', label:'Área'}, {id:'bar', label:'Barras (half)'}, {id:'blocks', label:'Bloquinhos'}]` |
| defaultChartVariant | `area` |
| tapAction / navigateTo | `navigate` → `/(protected)/vision/monthly-spending` |
| Layout padrão do usuário | `{id:'monthly-spending', size:'wide', position:7 (ou 6), visible:true}` (L2896982 / L2897004) |

`getVisionChartVariantsForSize(id, size)` (L772175) [O]:
```ts
id !== 'monthly-spending' ? [] :
  (REGISTRY['monthly-spending'].chartVariants?.map(v => v.id) ?? []).filter(v => v !== 'bar' || size === 'half')
// half → ['area','bar','blocks'];  wide/full → ['area','blocks']
```
`getVisionWidgetPreviewOptions` (L772230) usa a mesma regra para gerar as opções de prévia `{size, chartVariant}` do seletor de widgets.

Existe também um registro separado da **home do cartão** (L775648: `{id:'monthly-spending', size:'full', label:'Gasto mensal', description:'Visualize a evolução dos seus gastos no mês', icon:'arrowTrendUp'}`) e um catálogo de "card widgets" (L2985302: `name:'Gastos do mês'`, `tapAction:'navigate-expand'`, `navigateTo:'/card/spending'`). Esses não são a home Vision (fora de escopo, só citados).

### 1.2 `VisionMonthlySpendingWidgetConnected` (L2841883, módulo M8995) [O]
Props: `size = 'wide'`, `chartVariant`, `onPress`, `emptyState`, `animate = false`.

1. **Normalização da variante**: se `size === 'wide'` e `chartVariant === 'bar'` → `'area'`; caso contrário a variante passa como veio (pode ser `undefined`; o default `area` é aplicado mais abaixo — ver §2).
2. `{data, isLoading, isError, isFetched} = useMonthlySpendingData(size, chartVariant)` (ver §3).
3. Ordem das decisões:

| Condição | half | wide | full |
|---|---|---|---|
| `isError` | `Widget2x2 testID="monthly-spending-error" label="Gastos do mês" value="--" subtitle="Tente novamente" showChevron={false} visual={<WidgetErrorState compact message="Falha ao carregar"/>}` | `WidgetWide testID="monthly-spending-error" label="Gastos do mês" value="--" showChevron={false} visual={<WidgetErrorState compact message="Falha ao carregar"/>}` | `WidgetFull testID="monthly-spending-error" title="" showChevron={false}>{<WidgetErrorState/>}` |
| `isFetched && !data` (vazio) | `Widget2x2 testID="monthly-spending-empty" label="Gastos do mês" value="R$ 0,00" showChevron={false} visual={<WidgetEmptyStateSlot emptyState message="Sem gastos registrados"/>}` | `WidgetWide` idem (mesmos textos e visual) | `WidgetFull testID="monthly-spending-empty" title="" showChevron={false}>{<WidgetEmptyStateSlot emptyState/>}` (sem `message`) |
| senão | `<VisionMonthlySpendingWidget data size chartVariant onPress isLoading={isLoading \|\| !isFetched} emptyState animate/>` | idem | idem |

Os componentes de estado (`WidgetErrorState`, `WidgetEmptyStateSlot`) estão em W0 §9. Todas as saídas recebem `animate`.

---

## 2. `VisionMonthlySpendingWidget` (memo, L2843453, M8995) [O]

### 2.1 Módulo M8995: dependências e constantes
Dependências: `[978 React, 50 RN(View/StyleSheet), 239 RN, 4584 gráficos nativos (PierreAreaChart/PierreBarChart), 4428 privacidade (useHideValues/HIDDEN_VALUE), 8976 barrel de widgets, 2262 Text, 2260 HStack, 2258 VStack, 4432 PrivacyText, 8991 PrivacyCurrencyText, 8996 parseBRL, 2979 useThemeColors, 228 jsx]`.

Constantes do módulo (L2842562–2842581):
| Constante | Valor | Uso |
|---|---|---|
| `ACCENT` | `#B8FF12` | linha/área/ponto do gráfico de área; bloco ativo |
| `TRACK` | `#27272A` | eixo do gráfico de área; trilho das barras (não compacto); bloco inativo |
| `FULL_CHART_HEIGHT` | `150` | altura do gráfico de comparação (full) |
| `WIDE_CHART_HEIGHT` | `76` | altura do gráfico no wide |
| `FALLBACK_CHART_HEIGHT` | `48` | altura se o tamanho não for half/wide (inalcançável na prática) |
| `HALF_CHART_HEIGHT` | `72` | altura do gráfico no half |
| `HALF_BAR_WIDTH` | `90` | largura do gráfico de barras (half) |
| `DEFAULT_TITLE` | `'Gastos do mês'` | rótulo se `periodLabel` vier vazio |
| `EPSILON` | `0.001` | valor usado em todos os pontos quando **nenhum** ponto é > 0 (gráfico "flat") |

`StyleSheet` (L2844233–2844259):
```ts
chartContainer:     { alignSelf:'stretch', width:'100%' }
halfChartBody:      { alignItems:'flex-start', alignSelf:'stretch', width:'100%' }
halfChartContainer: { alignItems:'flex-start', alignSelf:'stretch', marginLeft:-5, width:'100%' }   // 4294967291 = -5 (uint32)
halfNativeChart:    { width:124 }
nativeChart:        { alignSelf:'flex-start' }
wideAreaNativeChart:{ alignSelf:'stretch', width:'100%' }
wideBarNativeChart: {}
block:              { width:15, height:15, borderRadius:4, backgroundColor:'#27272A' }
blockActive:        { backgroundColor:'#B8FF12' }
blockCompact:       { width:18, height:18 }
blocksGrid:         { flexDirection:'row', flexWrap:'wrap', gap:4, maxWidth:246 }
blocksGridCompact:  { maxWidth:86 }
```

### 2.2 Entradas derivadas
Props: `data, size = 'wide', chartVariant, onPress, isLoading, emptyState, animate = false`.
```ts
const hidden = useHideValues();                          // W0 §10.1
const { foreground } = useThemeColors();                 // #fafafa
const raw    = data?.data ?? [];
const finite = raw.filter(p => Number.isFinite(p.value));
const points = finite.length > 0 ? finite
  : [{value:0,label:'01'},{value:0,label:'08'},{value:0,label:'22'},{value:0,label:'31'}];   // placeholder
let variant = chartVariant ?? data?.chartVariant ?? 'area';
if (size === 'wide' && variant === 'bar') variant = 'area';
const amount      = data?.amount ?? 'R$ 0,00';           // string BRL já formatada pelo servidor/merge
const title       = data?.periodLabel ?? 'Gastos do mês';
const trendPct    = data?.trendPercentage;               // string, ex.: "12%"
```

### 2.3 half (Widget2x2, 200 px)
```
Widget2x2 testID="vision-monthly-spending-widget" onPress visualPlacement="middle" bottomClassName="gap-0" animate
  visual  = isLoading ? <WidgetLoadingState variant="content"/>
                      : <View collapsable={false} style={halfChartBody}><MonthlyChart points size="half" variant/></View>
  label   = title
  valueSlot = <PrivacyCurrencyText valueCents={parseBRLToCents(amount)} hidden fontSize={20} fontWeight="800" color={foreground}
                 disableMinWidth compactOnOverflow testID="vision-monthly-spending-widget-value"/>
  value   = hidden ? HIDDEN_VALUE : amount             (não é exibido, porque valueSlot tem prioridade)
  subtitle= (variant !== 'blocks' && trendPct) ? '↑ ' + trendPct : undefined
```
- Observação [O]: no half a seta é **sempre `↑`**, sem considerar `trend`, e o subtítulo não tem cor própria (`text-base text-muted-foreground` #a1a1aa, W0 §6). Com `blocks` não há subtítulo.
- `visualPlacement='middle'`: chevron numa linha própria em cima (`HStack justify-end`), gráfico logo abaixo (W0 §5.2). `bottomClassName='gap-0'` → label, valor e subtítulo colados.
- Valor: `PrivacyCurrencyText` (W0 §11.4), 20 px, peso 800, cor #fafafa, sem contagem (durationMs 0), compacta ("mil"/"mi") se não couber.

### 2.4 wide (WidgetWide, 170 px)
```
WidgetWide testID="vision-monthly-spending-widget" onPress animate
  label    = title                                       (text-lg muted, 15,75/24,5)
  value    = hidden ? HIDDEN_VALUE : amount              (text-2xl font-extrabold, 21/28, #fafafa; PrivacyText)
  subtitle = (variant === 'blocks' && data?.comparisonLabel) ? data.comparisonLabel : undefined
  visual   = isLoading ? <WidgetLoadingState variant="content"/> : <MonthlyChart points size="wide" variant/>
```
No wide **não há** texto de tendência; só a variante `blocks` mostra `comparisonLabel` como subtítulo.

### 2.5 full (WidgetFull, altura livre)
```
WidgetFull testID="vision-monthly-spending-widget" title={title} onPress animate
 └─ VStack className="gap-3"                                         // 10,5
     ├─ VStack className="gap-1"                                     // 3,5
     │   ├─ PrivacyText "text-3xl font-extrabold text-foreground" testID="vision-monthly-spending-widget-value"
     │   │      numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.75 allowFontScaling=false
     │   │      value = hidden ? HIDDEN_VALUE : amount               // Geist-ExtraBold 26,25/31,5 #fafafa
     │   └─ showComparison ? HStack "items-center gap-1" (3,5):
     │        ├─ Text `text-sm font-medium ${trendColor}` → `${arrow} ${trendPct}`       // Geist-Medium 16
     │        ├─ Text "text-sm text-muted-foreground" → "vs"                              // Geist-Regular 16 #a1a1aa
     │        ├─ PrivacyText "text-sm text-muted-foreground" value = hidden ? HIDDEN_VALUE : (previousAmount ?? '')
     │        └─ Text "text-sm text-muted-foreground" → data.comparisonLabel ?? 'mês anterior'
     │      : null
     └─ isLoading ? <WidgetLoadingState variant="content"/>
                  : <MonthlyComparisonChart current={points} previous={data?.previousData}/>
```
- `showComparison = Boolean(trendPct && previousAmount)`.
- `trend === 'down'` → `text-emerald-400` (**#34d399**) e seta `↓`; `trend === 'up'` → `text-[#EF4444]` e `↑`; outro valor → `text-muted-foreground` (#a1a1aa) e `↑`.
  (gasto **menor** que no mês anterior = verde.)
- Todos os `Text` com `allowFontScaling={false}`. `text-sm` = 16 sem lineHeight (padrão do RN).
- Exemplo de leitura: `↓ 12% vs R$ 3.210,00 mês anterior` [I no formato dos campos, que vêm prontos do servidor/merge].

### 2.6 `MonthlyChart` (L2842298): o seletor de gráfico
```ts
const compact = size === 'half';
const height  = compact ? 72 : size === 'wide' ? 76 : 48;
const body = variant === 'bar'    ? <MonthlyBarChart    compact height points/>
           : variant === 'blocks' ? <MonthlyBlocksChart compact points/>
           :                        <MonthlyAreaChart   compact height points/>;
return <View collapsable={false} testID={`vision-monthly-spending-${variant}-chart`}
             style={compact ? halfChartContainer : chartContainer}>{body}</View>;
```
O full **não** usa `MonthlyChart`: usa sempre `MonthlyComparisonChart` (área atual + mês anterior). A variante escolhida só afeta half e wide.


### 2.7 Os gráficos são **nativos (Kotlin)** — módulo Expo `PierreCharts`
- [O] `PierreAreaChart` / `PierreBarChart` (M4584, wrapper JS em L1832790+) são `requireNativeView('PierreCharts')` (área = view padrão) e `requireNativeView('PierreCharts','BarChart')` (L1832650–1832700). Se a view nativa não existir (ex.: iOS sem o módulo, web), o wrapper devolve um `<View {...style}/>` **vazio** (ramo `getNativePierreAreaChart() == null`).
- [O] O wrapper serializa `currentPoints`, `previousPoints` e `axisMarkers` com `JSON.stringify` (após filtrar pontos não finitos) e aplica defaults: `animated = true`, `lineStyle = 'smooth'`, `axisColor` dark `#27272A`, `labelColor` dark `#A1A1AA`, `interactionOverlayColor` dark `#B309090B` (light `#99FFFFFF`).
- [O] Desenho em `io.cloudwalk.pierre.charts.PierreAreaChartView` / `PierreBarChartView` (classes3.dex; smali em `parcial/w2/charts.smali`, utilitários em `kt.smali` = `PierreChartsModuleKt`). Tudo em `Canvas` Android, com `dp(x) = x * density`.
- Consequência para a réplica: **não há SVG no app**. Abaixo está a geometria exata do Canvas, para reproduzir com `react-native-svg` ou Skia.

Props que o widget passa a todos os gráficos [O, `b_charts.js`]:
`animated:false`, `loadingMode:'none'`, `hapticMode:'none'`, `performanceMode:'reduced'`, `animationDuration:0` (barras). Portanto **não há animação de entrada nem háptico/scrub** na home: o gráfico aparece pronto (progress = 1). As animações nativas (área: 1400 ms, revelação da esquerda para a direita por `clipRect`; barras: 920 ms com "stagger") só existem em outras telas.

### 2.8 `MonthlyAreaChart` (variante `area`, half e wide) [O b_charts.js:285]
```ts
const hasPositive = points.some(p => p.value > 0);
const currentPoints = points.map((p, i) => ({ x: i, value: hasPositive ? p.value : 0.001 }));  // EPSILON
<PierreAreaChart
  currentPoints={currentPoints} axisMarkers={[]}
  animated={false} loadingMode="none" hapticMode="none" performanceMode="reduced"
  lineColor="#B8FF12" fillColor="#B8FF12" dotColor="#B8FF12" axisColor="#27272A" labelColor="#A1A1AA"
  labelAreaHeight={0} topVisualPadding={8} bottomVisualPadding={4}
  leftInset={0} rightInset={8} endContentPadding={8}
  style={[ {alignSelf:'flex-start'}, compact ? {width:124} : {alignSelf:'stretch', width:'100%'}, {height} ]}/>
// height: half 72, wide 76.  half: container com marginLeft -5 (§2.1).
```
- O `x` é o **índice** do ponto (0..n−1), não o dia. Os dias ficam em `points[i].label`, mas a área não desenha rótulos (`axisMarkers=[]`, `labelAreaHeight=0`).
- O truque do EPSILON existe porque o nativo, quando **nenhum** valor é > 0, chama `drawEmptyState`. Sem título, esse método não desenha nada e devolve `false`. Com 0,001 o gráfico desenha uma linha reta colada à base.

**Geometria nativa (`PierreAreaChartView.onDraw`, charts.smali L1744+)** [O]:
```
chartH   = max(H − dp(labelAreaHeight), 1)                  // = H (labelArea 0)
plotW    = max(W − dp(leftInset) − dp(rightInset), 1)       // W − 8
drawW    = max(plotW − dp(endContentPadding), 1) [I*]       // W − 16  (*registrador ambíguo no smali)
minX,maxX= min/max de x entre current+previous (default 0 / 1)
yMax     = max( max(max(valores current+previous), 1) × 1.12 , 1 )   // 12% de folga no topo
yMin     = yAxisMin ?? 0 ;  yRange = max(yMax − yMin, 1)
plotH    = max(chartH − dp(top=8) − dp(bottom=4), 1)
px = dp(leftInset) + clamp((x − minX)/max(maxX − minX, 1), 0, 1) × drawW
py = dp(top) + plotH − clamp((v − yMin)/yRange, 0, 1) × plotH
```
Ordem de desenho:
1. **Eixo** (`drawAxisAndLabels`): linha horizontal `STROKE`, `strokeWidth = dp(1)`, cor `axisColor` (#27272A), na base da área do gráfico (`y = chartH`, rente à borda inferior da view), de `x = leftInset` a `leftInset + plotW` [O; a posição y vem dos registradores, então é I leve]. Os rótulos dos marcadores (15 dp, DEFAULT_BOLD, `labelColor`, 28 dp abaixo) não aparecem porque `axisMarkers` é vazio.
2. **Área** (`drawFill`): caminho da linha + `lineTo(últimoX, base)` + `lineTo(primeiroX, base)` + `close`. Preenchimento `FILL` com `LinearGradient` **vertical** de `(0,0)` (topo da view) a `(0, base)`: cor 0 = `fillColor` com **alpha 96/255 (37,6%)**, cor 1 = `0x00000000` (transparente), `CLAMP`. Recorte `clipRect(0, 0, primeiroX + (últimoX−primeiroX)×progress, base)`, com progress = 1 aqui.
3. **Linha do mês anterior** (só no full, ver §2.10): cor `comparisonColor`, que fica no default nativo `rgb(92,92,104)` = **#5C5C68** porque o JS não sobrescreve. `strokeWidth dp(1.4)`, **tracejada** com `DashPathEffect([dp(7), dp(7)], 0)`, progress 1.
4. **Linha atual**: `STROKE`, `cap ROUND`, `join ROUND` (MITER se `lineStyle='linear'`), `strokeWidth = dp(1.8)`, cor `lineColor` (#B8FF12), sem tracejado.
5. **Ponto final** (`drawDot`, se progress ≥ 0,98): círculo `FILL` na cor `dotColor` (#B8FF12), **raio dp(4.5)** (Ø 9), no último ponto.

**Curva `smooth`** (`PierreChartsModuleKt.smoothPath`, kt.smali L1405) [O]: Catmull-Rom convertida em Bézier cúbica, tensão 1/6.
```
moveTo(p0); para i em 0..n−2:
  p_prev = p[max(i−1,0)], p_i, p_next = p[i+1], p_next2 = p[min(i+2, n−1)]
  c1 = p_i    + (p_next − p_prev)/6
  c2 = p_next − (p_next2 − p_i)/6
  cubicTo(c1, c2, p_next)
```

### 2.9 `MonthlyBarChart` (variante `bar`, **só half**) [O b_charts.js:670]
```ts
const maxValue = Math.max(...points.map(p => p.value), 1);
const bars = points.slice(0, compact ? 4 : 7)           // half → 4 barras
  .map(p => ({ label: p.label ?? '', value: p.value, surfaceValue: maxValue }));
const surfaceMaxValue = Math.max(...bars.map(b => b.surfaceValue ?? b.value), 1);
<PierreBarChart bars={bars} animated={false} loadingMode="none" performanceMode="reduced"
  barGradientStartColor="#F0F30D" barGradientEndColor="#95FF48"
  labelColor="#A1A1AA" placeholderColor="#5C5C68"
  surfaceColor={compact ? 'transparent' : '#27272A'} surfaceMaxValue={surfaceMaxValue}
  placeholderSize={compact ? 14 : 18} yAxisLabelWidth={0} animationDuration={0}
  barWidth={compact ? 10 : 14} labelAreaHeight={compact ? 0 : 16}
  topVisualPadding={compact ? 4 : 8} bottomVisualPadding={compact ? 4 : 8}
  horizontalGap={compact ? 10 : 12}
  style={[{alignSelf:'flex-start'}, {width:124}, {width:90}, {height}]}/>   // 90 × 72 no half
```
(No bytecode o `Math.max` recebe `[...valores, 1]`: `arraySpread` seguido de `DefineOwnByVal(1)`.)
- Como `bar` em wide vira `area` (§1.2/§2.2), na prática só o ramo `compact` roda. Os valores não compactos ficam aqui por completude.
- **Atenção**: as barras são os **4 primeiros** buckets, não os 4 últimos. Os buckets vêm do servidor/merge (§3).

**Geometria nativa (`PierreBarChartView.onDraw`, charts.smali L3172+)** [O]:
```
chartH  = max(H − dp(labelAreaHeight), 1)       // 72
usableW = max(W − dp(yAxisLabelWidth), 1)       // 90
plotH   = max(chartH − dp(top) − dp(bottom), 1) // 72 − 4 − 4 = 64
base    = chartH − dp(bottom)                   // 68
yMax    = max( max(surfaceMaxValue ?? max(value, surfaceValue), 1) × 1.08, 1 )   // 8% de folga
bw      = max(dp(barWidth), dp(12))             // ⚠ mínimo 12 dp: o 10 do JS vira 12
gap     = dp(horizontalGap)                     // 10
content = n·bw + max(n−1,0)·gap                 // 4·12 + 3·10 = 78
totalW  = max(usableW, content + 2·gap)         // max(90, 98) = 98 → rolagem horizontal possível de 8 dp
startX  = (totalW − content)/2                  // 10
x_i     = startX + i·(bw + gap)                 // 10, 32, 54, 76
scrollX inicial = clamp(startX + sel·(bw+gap) + bw/2 − usableW/2, 0, totalW − usableW)  // sel=-1 → 0
```
Por barra:
1. **Trilho** (`surface`): `drawRoundBar(x, base, bw, clamp(surfaceValue/yMax)·plotH, surfaceColor)`. No half é `transparent` (invisível); fora do half seria `#27272A`.
2. **Valor > 0**: `drawRoundBar(x, base, bw, clamp(value/yMax)·plotH, start, end)`:
   - `h = max(altura, bw)` (altura mínima = largura, então a menor barra é um círculo);
   - `RectF(x, base − h, x + bw, base)`, `drawRoundRect` com raio `bw/2` (pílula);
   - `LinearGradient` **diagonal** de `(x, topo)` a `(x + bw, base)`: `#F0F30D` → `#95FF48`, `CLAMP`.
   - `selectedBarIndex` tem default −1, então todas as barras ficam ativas. Com seleção, as demais usariam `inactiveBarColor` (default nativo `rgb(46,46,46)` [O]).
3. **Valor ≤ 0** vira **placeholder**: oval tracejada, `STROKE dp(1.2)`, `DashPathEffect([dp(4), dp(4)])`, cor `#5C5C68`, tamanho `s = min(dp(placeholderSize=14), bw=12) = 12`, `RectF(x + (bw−s)/2, base − s, x + (bw+s)/2, base)`. Resultado: círculo tracejado de Ø 12 apoiado na base.
4. Rótulos: texto `label.take(10)`, 13 dp DEFAULT_BOLD, `labelColor`, reduzido de 0,5 em 0,5 dp até caber em `bw + 12dp` (mínimo 8 dp), centrado em `x + bw/2` e em `y = chartH + 20dp`. Com `labelAreaHeight = 0` isso fica **fora da view**, então no half os rótulos **não aparecem** [I pela geometria].
- Se todos os valores forem 0, `drawEmptyState` sem título retorna `false` (BaseChartView L75) e o gráfico desenha 4 círculos tracejados.

### 2.10 `MonthlyComparisonChart` (full) [O b_charts.js:468]
```ts
const hasPositive = current.some(p => p.value > 0) || (previous?.some(p => p.value > 0) ?? false);
const currentPoints  = current.map((p, i) => ({ x: i, value: hasPositive ? p.value : 0.001 }));
const previousPoints = previous && previous.length > 0 ? previous.map((p, i) => ({ x: i, value: p.value })) : undefined;
<PierreAreaChart currentPoints previousPoints axisMarkers={[]}  /* demais props iguais a §2.8 */
  style={[{alignSelf:'flex-start'}, {alignSelf:'stretch', width:'100%'}, {height:150}]}/>
```
- Mesma geometria de §2.8, com **altura 150** e a largura total do card. `minX/maxX` e `yMax` consideram current **e** previous juntos. Se o mês anterior tiver mais pontos, a linha atual ocupa só a parte esquerda proporcional (x é índice): é o efeito de "mês em andamento" [I].
- O mês anterior é a **linha cinza tracejada** #5C5C68 (1,4 dp, traço 7/7), sem área. O mês atual é a linha verde #B8FF12 de 1,8 dp, com área em degradê (37,6% → 0) e ponto de Ø 9 no fim.

### 2.11 `MonthlyBlocksChart` (variante `blocks`, half e wide) [O b_charts.js:960]
```ts
const count = compact ? 12 : 39;
let cells = points.slice(0, count);
if (cells.length < count) cells = [...cells, ...Array.from({length: count - cells.length}, () => ({value: 0}))];
const max = Math.max(...cells.map(c => c.value), 1);
<View style={[styles.blocksGrid, compact ? styles.blocksGridCompact : null]}>
  {cells.map((c, i) => (
    <View key={i} style={[styles.block, compact ? styles.blockCompact : null,
      c.value > 0 ? [styles.blockActive, {opacity: 0.32 + (c.value / max) * 0.68}] : null]}/>
  ))}
</View>
```
- Feito em **RN puro** (Views), não no módulo nativo.
- Grade: `flexDirection:'row', flexWrap:'wrap', gap:4`.
  - half: blocos **18×18**, raio 4, `maxWidth 86`. Cabem 4 por linha (4·18 + 3·4 = 84), em **3 linhas** (12 blocos): 84×62.
  - wide: blocos **15×15**, raio 4, `maxWidth 246`. Cabem 13 por linha (13·15 + 12·4 = 243), em **3 linhas** (39 blocos): 243×53.
- Cor: inativo (valor ≤ 0) é `#27272A` opaco; ativo é `#B8FF12` com **opacidade linear** `0.32 + 0.68·(valor/max)` (mínimo 0,32, máximo 1). Não há gradiente nem animação.
- Cada bloco é um bucket do payload; o excedente é cortado e o que falta vira bloco vazio [O]. O significado de cada bucket vem do servidor/merge (§3).
- O degradê `linear-gradient(95deg, rgb(240,243,13) 66%, rgb(149,255,72) 100%)` (`HEATMAP_BAR_GRADIENT_CSS`, L2205149, com `colors=['rgb(240, 243, 13)','rgb(149, 255, 72)']` e `locations=[0.66, 1]`) **não** é usado pelos bloquinhos. Ele pertence a outro módulo (junto de `getCategoryColor`) e é usado por `HeatmapBarGradient` (L2209735), fora dos widgets (§4.8) [O].


---

## 3. Dados de `monthly-spending`: `useMonthlySpendingData(size, chartVariant)` (M8997, c_widget.js:929) [O]

### 3.1 Duas fontes combinadas
```ts
const server  = useVisionSystemWidgetData('monthly-spending', size, chartVariant);   // W1 §1: GET /api/mobile/vision/widgets/monthly-spending/data
const periods = useMemo(() => resolvePierreWidgetPeriods(), [new Date().toDateString()]);
const pierre  = useVisionPierreCardSpending(periods);     // { current, comparison, isSettled, isEnabled }
const pending = !pierre.isSettled;
const data = useMemo(() => pending ? undefined
  : mergePierreIntoMonthlySpendingWidget(server.data, buildPierreWidgetBuckets(pierre.current, pierre.comparison),
                                         { size, chartVariant }) ?? undefined,
  [server.data, pierre.current, pierre.comparison, pending, size, chartVariant]);
return { data, isLoading: server.isLoading || pending, isError: server.isError, isFetched: server.isFetched && !pending };
```
- O **servidor** manda o payload principal (`amount`, `data`, `previousData`, `trend`, `trendPercentage`, `previousAmount`, `comparisonLabel`, `periodLabel`, `chartVariant`). O app **soma no cliente** os gastos do **cartão Pierre** (crédito próprio), que não estão no agregado do servidor.
- `resolvePierreWidgetPeriods()` (e_data2.js:329) [O]:
  - `current` = `{ startDate: 'YYYY-MM-01' do mês corrente, endDate: hoje 'YYYY-MM-DD' }`;
  - `comparison` = mês anterior inteiro, `{ 'YYYY-MM-01', 'YYYY-MM-<último dia>' }` (janeiro → dezembro do ano anterior).
- `useVisionPierreCardSpending` / `useStatementsWindow` (d_data1.js:1/184): `useQuery({ queryKey: ['vision','pierre-card-activities', …período], queryFn: getStatementsDirect({startDate, endDate}), enabled: período != null })` → `toPierreExpenses` → `filterExpensesInRange(dateKey ∈ [start, end])`.
  - `toPierreExpenses` (e_data2.js:508) mantém só `direction === 'debit'`, `!denied` e `amountCents` finito. A data é `toSaoPauloIso(transactionDate)` (fuso America/Sao_Paulo). Cada item vira `{id, dateKey, year, month, day, amount: centsToReais(amountCents), description (trim, fallback 'Compra'), category: translatePierreCardBucketToVisionGroup(mccCategory), logoUrl}`.
- `buildPierreWidgetBuckets(current, previous)` → `{ current: bucketByDay(current), previous: bucketByDay(previous) }`, em que `bucketByDay` gera `{ totals: Map<dia, soma R$>, counts: Map<dia, nº de compras> }` (d_data1.js:471).
- `cumulativeUpTo(totals, dia)` = soma de `totals` para todo dia ≤ `dia`. `round2(x) = Math.round(x*100)/100`.

### 3.2 `mergePierreIntoMonthlySpendingWidget(server, buckets, {today?, chartVariant?})` (e_data2.js:786) [O]
- Sem buckets, ou com `current.totals` **e** `previous.totals` vazios → devolve `server` como veio (`null` se não houver).
- `today = opts.today ?? new Date().getDate()`; `variant = opts.chartVariant ?? server?.chartVariant ?? 'area'`.
- `pierreNow = round2(cumulativeUpTo(current.totals, today))`; `total = round2(parseBRLToReais(server?.amount) + pierreNow)`.
- Formatação: `formatReais(n)` → "R$ 1.234,56" (BRL pt-BR; W0 §11).

**Variante `area`** (também o que o full consome):
- `data` (série **acumulada diária**):
  - com `server.data`: para cada ponto cujo `label` é numérico (dia), `value = Math.round(p.value + cumulativeUpTo(pierre.current.totals, dia))`; pontos não numéricos ficam iguais;
  - sem `server.data`: `d = 1..today` → `{ value: Math.round(acumulado Pierre até d), label: 'dd' }`.
- `previousData`: o mesmo com `previous.totals` e `server.previousData`; sem servidor, `d = 1..diasDoMêsAnterior`.
- Comparação **no mesmo dia do mês anterior**: `prevSameDay = previousData.find(p => Number(p.label) === Math.min(today, diasDoMêsAnterior))?.value ?? 0`; `prev = round2(prevSameDay)`.
- `differenceAmount = formatReais(round2(|total − prev|))`.
- `trend = prev < total ? 'up' : total < prev ? 'down' : 'flat'`.
- `trendPercentage = (prev > 0 ? Math.abs((total − prev)/prev × 100) : 0).toLocaleString('pt-BR', {minimumFractionDigits:0, maximumFractionDigits:1}) + '%'` (ex.: "12,5%", "0%").
- Saída: `{...server, amount: formatReais(total), data, previousData, previousAmount: formatReais(prev), differenceAmount, trend, trendPercentage, chartVariant:'area', periodLabel: server?.periodLabel ?? 'Gastos do mês', comparisonLabel: server?.comparisonLabel ?? 'mês anterior'}`.

**Variante `bar`**: base = `server ?? { amount: 'R$ 0,00', data: [dias 1..today com 0] }`. Saída: `{...base, amount: formatReais(total), data: base.data.map(somar acumulado Pierre nos labels numéricos), chartVariant:'bar', periodLabel}`. Não recalcula tendência: o subtítulo do half usa o `trendPercentage` do servidor. Se o servidor mandar buckets semanais com labels não numéricos (ex.: "S1"), o Pierre não é somado neles [I: formato do servidor não está no APK].

**Variante `blocks`**: base = `server ?? { amount: 'R$ 0,00', data: [] }`. Se `base.data` vier vazio, usa `Array.from({length: today}, (_, i) => ({ value: 0, label: pad2(i+1) }))`. `data = base.data.map(p => labelNumérico ? {...p, value: p.value + (pierre.current.counts.get(dia) ?? 0)} : p)`.
  - Nos **bloquinhos o valor é a QUANTIDADE de compras do dia** (`counts`), não o valor em R$ [O no merge]. A opacidade do bloco (§2.11) é proporcional ao número de transações do dia em relação ao dia de pico. Para o servidor, presume-se a mesma semântica [I].
  - Consequência: no half (12 blocos) aparecem só os **dias 1–12**; no wide (39 blocos), os dias 1–31 e 8 blocos sempre vazios no fim [O pela lógica de `slice`/padding; I no mapeamento dia↔bloco do servidor].

### 3.3 Resumo do que cada tamanho mostra
| | half | wide | full |
|---|---|---|---|
| Valor | `amount` 20 px/800 (compacta "mil") | `amount` text-2xl extrabold | `amount` text-3xl extrabold |
| Linha extra | `↑ {trendPercentage}` (sempre ↑, sem cor; oculta em `blocks`) | `comparisonLabel` só em `blocks` | `{↑/↓} {pct}` colorido + "vs" + `previousAmount` + `comparisonLabel` |
| Gráfico | área 124×72 / barras 90×72 (4 primeiros buckets) / blocos 4×3 de 18 | área 100%×76 / blocos 13×3 de 15 | comparação: área atual + linha tracejada do mês anterior, 100%×150 |
| Série | acumulado diário (área), buckets (barras), contagem/dia (blocos) | idem | acumulado diário atual vs. anterior |


---

## 4. `spending-by-category` ("Gastos por categoria")

### 4.1 Registro e identidade [O]
| Campo | Valor |
|---|---|
| id / label | `spending-by-category` / `Gastos por categoria` (L772585) |
| size padrão | `half` |
| supportedSizes | `['half','wide','full']` |
| tapAction / navigateTo | `navigate` → `/(protected)/vision/spending-by-category` |
| internalPressSizes | `['full']`. No full, o card não é um botão único: o toque é tratado por áreas internas (valor, linhas, rodapé). Helper `visionWidgetSupportsInternalPress(id, size)` (L771833) |
| Layout padrão | `{id:'spending-by-category', size:'half', position:9 (ou 8), visible:true}` (L2896986 / L2897008) |
| Deep link de volta | `/vision/spending-by-category` → `{widgetId:'spending-by-category', button:'body'}` (L751439) |

Componentes: `SpendingByCategoryWidgetConnected` (L2862447), `VisionSpendingByCategoryWidget` (L2864048) e os internos `SegmentedCategoryBar`, `SubcategoryRow`, `WidgetModeToggle` (L2863541), `EmptyLaunchesRow` e `CategoryRow`. Avatar: `CategoryAvatar` (L2723407). Cores: `resolveCategoryColor` / `resolveVisionCategoryDisplayMatch` (L2724074–2724176, módulo M8534).

**Não há gráfico de rosca, anel nem módulo nativo**: o "gráfico" é uma **barra horizontal segmentada** feita de `View`s (até 3 segmentos).

### 4.2 Dados [O]
- `useSpendingByCategoryData(size)` = `useVisionSystemWidgetData('spending-by-category', size)` → `GET /api/mobile/vision/widgets/spending-by-category/data` (W1 §1). Devolve `{data, isLoading, isError, isFetched}` sem transformação.
- Payload usado: `{ amount: string BRL, categories: [{ id?, name, value: number, amount: string BRL, percentage: number, color, iconKey?, emoji? }], subtitle: string (ex.: "8 categorias"), period: string (ex.: "este mês") }`. A ordem das categorias vem do servidor (presumivelmente decrescente por valor [I]); o app **não ordena**.
- Só no **full** e no modo `categories`: `useSpendingByCategorySubcategories(month, {enabled})`, com `month = 'YYYY-MM'` do mês corrente, calculado uma vez → `GET /api/mobile/vision/spending-by-category/subcategories?period=YYYY-MM` (L1646008). Query com `refetchOnReconnect:false` e `useRefetchOnScreenFocus`. Resultado: `data.subcategories` → `buildSubcategoryRows` (§4.7).
- Estado local: `const [widgetMode, setWidgetMode] = useState<'groups'|'categories'>('groups')`, que **não é persistido**.

### 4.3 Camada Connected (L2862447) [O]
```ts
const onPressInternal = () => size !== 'full' ? onPress?.() : onPress?.({ params: { chartMode: widgetMode } });
if (isError) return size === 'full'
  ? <WidgetFull testID="spending-by-category-error" title="" showChevron={false} animate={animate}><WidgetErrorState/></WidgetFull>
  : <View testID="spending-by-category-error" className="rounded-3xl bg-surface"
          style={size === 'half' ? {height:200, alignItems:'center', justifyContent:'center', padding:16}
                                 : {minHeight:170, alignItems:'center', justifyContent:'center', padding:16}}>
      <WidgetErrorState compact/>
    </View>;
return <VisionSpendingByCategoryWidget
  data={data ?? { amount:'R$ 0,00', categories:[], subtitle:'0 categorias', period:'este mês' }}
  size onPress={onPressInternal} widgetMode onWidgetModeChange={setWidgetMode}
  subcategoryRows={buildSubcategoryRows(subQuery.data?.subcategories)}
  subcategoriesLoading={widgetMode === 'categories' && subQuery.isLoading}
  isLoading={isLoading || !isFetched} emptyState animate/>;
```
- `rounded-3xl` = **21** (W0 §14; o raio 24 dos shells vem de `style`, não da classe); `bg-surface` = #18181B. Não existe ramo "vazio" no Connected: o vazio é tratado dentro do widget (§4.5).

### 4.4 Derivações comuns (`VisionSpendingByCategoryWidget`, L2864048) [O]
```ts
const hidden = useHideValues();
const { foreground } = useThemeColors();                  // #FAFAFA
const categories   = data.categories ?? [];
const hasPositive  = categories.some(c => Number(c.value) > 0);
const muted        = !hasPositive;
const amount       = hasPositive ? data.amount : 'R$ 0,00';
const displayAmount= hidden ? HIDDEN_VALUE : amount;
const period       = data.period?.trim() || 'este mês';
const title        = size === 'half' ? `Gastos • ${period}` : `Gastos por categoria • ${period}`;
const subtitle     = hasPositive ? data.subtitle : '0 categorias';
const extraGroups  = Math.max(0, categories.length - 3);
const extraSubs    = Math.max(0, subcategoryRows.length - 3);
```
- **Quantas categorias:** a barra e as linhas mostram no máximo **3** (`slice(0,3)`). Não existe item "Outros": o resto aparece apenas no rodapé "Ver mais N grupos" (full) e no `subtitle` vindo do servidor.
- Carregando: `<WidgetLoadingState variant={size === 'full' ? 'content' : 'value'}/>` no lugar do corpo (bytecode #73608 confirma). O cabeçalho e o card continuam.

### 4.5 `SegmentedCategoryBar` (memo) [O f_cat1.js:593]
Props `{categories, muted = false, height = 16}`.
```ts
const total = categories.reduce((s, c) => s + Math.max(0, c.value), 0);
if (muted || total <= 0)
  return <View testID="vision-spending-category-bar-empty" style={[{height:16, width:'100%', borderRadius:8, backgroundColor:'#27272A'}, {height}]}/>;
return (
  <View testID="vision-spending-category-bar" style={[{flexDirection:'row', height:16, gap:4}, {height}]}>
    {categories.slice(0, 3).map((c, i) => (
      <View key={c.name ?? `${color}-${i}`}
            style={[{borderRadius:6}, {flex: Math.max(0.05, c.value / total), backgroundColor: resolveCategoryColor({name:c.name}, c.color)}]}/>
    ))}
  </View>);
```
- `total` soma **todas** as categorias, mas só as 3 primeiras viram segmento. Como são `flex`, as 3 dividem 100% da largura **proporcionalmente entre si** (o resto não aparece como espaço vazio). Cada segmento tem peso mínimo de 0,05, e o gap entre segmentos é de 4 px.
- Cantos: raio 6 em cada segmento. Na barra de 10 px do full, o RN limita o raio a 5, o que deixa cada segmento em forma de pílula.
- Cor: `resolveCategoryColor({name}, fallback)` procura o nome normalizado em `VISION_CATEGORY_DISPLAY_MAP` (§4.8). Se não achar, usa `c.color` do servidor.
- Sem animação.

### 4.6 Layout por tamanho [O g_cat2.js]
StyleSheet (L2864800+):
```ts
halfCard: {height:200, borderRadius:24, padding:16, overflow:'hidden'}
wideCard: {minHeight:170, borderRadius:24, padding:18, overflow:'hidden'}
fullCard: {minHeight:340, borderRadius:24, padding:16, overflow:'hidden'}
bar: {flexDirection:'row', height:16, gap:4}       barSegment: {borderRadius:6}
emptyBar: {height:16, width:'100%', borderRadius:8, backgroundColor:'#27272A'}
emptyIcon: {width:44, height:44, borderRadius:22}
fullBodyTransitionContainer: {height:260}  fullAmountBlock: {height:60, justifyContent:'center'}
fullRowsBlock: {height:144, overflow:'hidden'}  fullFooterBlock: {height:32, justifyContent:'center'}
```
O card é o `WidgetGlassCard` (W0) com `animate` e o `style` acima. O widget **não usa** `Widget2x2/WidgetWide/WidgetFull`.

**half e wide**: o card inteiro é um botão.
```
Pressable testID=`vision-spending-by-category-widget-${size}` disabled={!onPress} onPress accessibilityRole={onPress ? 'button' : undefined}
 └─ WidgetGlassCard animate style={halfCard | wideCard}
     └─ VStack "flex-1 gap-4"                                             // gap 14
         ├─ HStack "items-start justify-between"
         │   ├─ half: <View/> (vazio) | wide: Text "flex-1 text-base text-muted-foreground" numberOfLines=1 → title
         │   └─ PierreIcon chevronRight 16 #F4F4F5
         └─ corpo (abaixo)
```
- **half** (corpo):
```
VStack "flex-1 justify-between"
 ├─ <View/>                                                     // espaçador vazio
 ├─ SegmentedCategoryBar categories muted                      // altura 16, até 3 segmentos
 └─ VStack "gap-1"                                              // 3,5
     ├─ Text "text-base text-muted-foreground" (1 linha) → `Gastos • ${period}`     // Geist-Regular 14/21 #A1A1AA
     ├─ PrivacyCurrencyText valueCents={parseBRLToCents(amount)} hidden fontSize=20 fontWeight="800"
     │      color=#FAFAFA disableMinWidth compactOnOverflow       // valor, compacta ("mil") se estourar
     └─ hasPositive ? Text "text-base text-muted-foreground" → subtitle  : null
```
- **wide** (corpo):
```
VStack "flex-1 justify-between"
 ├─ PrivacyText "text-3xl font-extrabold text-foreground" value=displayAmount numberOfLines=1
 │      adjustsFontSizeToFit minimumFontScale=0.72                // Geist-ExtraBold 26,25/31,5 #FAFAFA
 └─ VStack "gap-3"                                                // 10,5
     ├─ SegmentedCategoryBar categories.slice(0,3) muted           // 16 px
     └─ hasPositive ? HStack "justify-end" → Text "text-base text-muted-foreground" → subtitle
                    : <EmptyLaunchesRow/>
```
- `EmptyLaunchesRow` (memo): `HStack "items-center gap-4"` (14) → `View "items-center justify-center bg-muted"` 44×44, raio 22, #27272A, com `PierreIcon timeClock` 18 #A1A1AA; ao lado, `Text "text-lg text-muted-foreground"` 1 linha **"Não há lançamentos"** (Geist-Regular 15,75/24,5 #A1A1AA).

**full**: sem cabeçalho, sem chevron e sem Pressable externo.
```
WidgetGlassCard animate style={fullCard}
 └─ VStack "flex-1 gap-3"                                                         // 10,5
     ├─ WidgetModeToggle mode={widgetMode} onModeChange
     └─ View testID="vision-spending-category-widget-body" style={{height:260}}
         └─ Animated.View key={widgetMode} style={absoluteFill}
               entering={FadeInDown.duration(220).easing(Easing.out(Easing.cubic))}
               exiting ={FadeOutUp.duration(160).easing(Easing.out(Easing.cubic))}
             └─ VStack "gap-3"
                 ├─ Pressable (role button se onPress, disabled !onPress) testID=`vision-spending-by-category-widget-full`
                 │   └─ View pointerEvents="none" → View {height:60, justifyContent:'center'}
                 │       groups:     VStack "gap-2" [ PrivacyText text-3xl extrabold (minScale 0.72) ,
                 │                                    SegmentedCategoryBar categories.slice(0,3) muted height=10 ]
                 │       categories: VStack [ PrivacyText text-3xl extrabold ,
                 │                            Text "text-[14px] text-muted-foreground" → "Gastos esse mês" ]
                 ├─ View {height:144, overflow:'hidden'}
                 │   subcategoriesLoading → Pressable(…full) → View pn=none → <WidgetLoadingState variant="content"/>
                 │   vazio¹               → Pressable(…full) → View pn=none → <EmptyLaunchesRow/>
                 │   senão VStack "gap-3":
                 │      groups     → categories.slice(0,3).map((c,i) => <CategoryRow key={`${c.name ?? 'category'}-${i}`} category={c} hidden onPress/>)
                 │      categories → subcategoryRows.slice(0,3).map(r => <SubcategoryRow key={r.id} row={r} hidden onPress/>)
                 └─ Pressable role=button disabled={!onPress} onPress style={{height:32, justifyContent:'center'}}
                       testID = groups ? 'vision-spending-category-widget-view-more' : 'vision-spending-category-widget-view-all'
                     └─ (groups ? extraGroups : extraSubs) > 0
                          ? Text "text-center text-sm text-muted-foreground" →
                              `Ver mais ${n} ${n === 1 ? 'grupo' : 'grupos'}`  /  `Ver mais ${n} ${n === 1 ? 'categoria' : 'categorias'}`
                          : null
```
¹ vazio = `!hasPositive` em `groups`; `subcategoryRows.length === 0` em `categories`.
- Orçamento de altura: 60 + 144 + 32 + 2×10,5 = 257 ≤ 260. Três linhas de 40 + 2×10,5 = 141 ≤ 144. O card fica com 16 + toggle (≈33) + 10,5 + 260 + 16 ≈ 335,5, e o `minHeight 340` prevalece [I na altura do botão mini].
- `text-sm` = 16 (Geist-Regular). Os valores ocultos usam `HIDDEN_VALUE` (W0 §10).
- Todos os toques do full chamam `onPress({params:{chartMode: widgetMode}})`: a tela de destino abre no mesmo modo (`groups`/`categories`) [O no Connected; I no uso do parâmetro pela tela].

### 4.7 Linhas do full [O f_cat1.js]
**`CategoryRow`** (modo `groups`):
```
(onPress ? Pressable role=button testID=`vision-spending-category-row-${id ?? name}` : fragmento)
 └─ HStack "items-center justify-between gap-3"
     ├─ HStack "min-w-0 flex-1 items-center gap-3"
     │   ├─ CategoryAvatar category={c} filled size={40}
     │   └─ VStack "min-w-0 flex-1"
     │       ├─ Text "text-[16px] font-bold text-foreground" 1 linha → c.name ?? 'Categoria'     // Geist-Bold 16 #FAFAFA
     │       └─ Text "text-[14px] text-muted-foreground" 1 linha → Number.isFinite(c.percentage) ? `${c.percentage}%` : ''
     └─ PrivacyText "text-lg font-bold text-foreground" 1 linha adjustsFontSizeToFit minScale 0.75
            value = hidden ? HIDDEN_VALUE : (c.amount ?? 'R$ 0,00')                          // Geist-Bold 15,75/24,5
```
**`SubcategoryRow`** (modo `categories`):
```
(onPress ? Pressable role=button testID=`vision-spending-category-widget-sub-row-${toTestIdSlug(id)}` : fragmento)
 └─ HStack "items-center gap-3"
     ├─ View {width:40, height:40, borderRadius:20, backgroundColor: dark ? '#27272A' : '#E4E4E7', center}
     │    └─ row.emoji ? Text {fontSize:18, lineHeight:20} → emoji : PierreIcon name=row.iconKey size=20 color=row.color
     ├─ VStack "min-w-0 flex-1"
     │    ├─ Text "text-[16px] font-medium text-foreground" 1 linha → row.label                 // Geist-Medium 16
     │    └─ Text "text-[14px] text-muted-foreground" 1 linha → `${row.percentage}%`
     └─ PrivacyText "text-[14px] font-medium text-foreground" 1 linha → hidden ? HIDDEN_VALUE : row.amount
```
`toTestIdSlug(s)` = `s.normalize('NFD').replace(/[̀-ͯ]/g,'').replace(/[^a-zA-Z0-9]+/g,'-').replace(/^-+|-+$/g,'').toLowerCase()`.

**`buildSubcategoryRows(subs = [])`** (g_cat2.js:1326):
```ts
subs.map(s => {
  const own   = resolveVisionCategoryDisplayMatch({label: s.name,  name: s.name});
  const group = own ?? resolveVisionCategoryDisplayMatch({label: s.group, name: s.group});
  return { id: s.id, label: s.name,
    color:   group?.color   ?? s.groupColor ?? '#A1A1AA',
    iconKey: group?.iconKey ?? 'walletMoney',
    percentage: s.percentage, amount: s.amountFormatted, value: s.amount, groupName: s.group,
    emoji: (own == null && !isNativeCategoryName(s.name)) ? (s.emoji ?? null) : null };
});
```
(Subcategorias criadas pelo usuário mostram o emoji dele; as nativas usam o ícone do mapa.)

**`CategoryAvatar`** (L2723407) com `filled` e `size 40` [O, com detalhes de cor em I leve]:
- Resolve `{emoji, hasIcon, iconKey, backgroundColor, hasColor}` a partir de `category.iconKey` (se existir em `pierreIcons`), senão de `resolveVisionCategoryDisplayMatch` por `id/title/name/label`. Fallbacks: ícone `walletMoney`, cor `#27272A`.
- Círculo `width = height = 40`, `borderRadius = 20`, `flexShrink:0`, centralizado. Com `filled`, o fundo é a **cor da categoria** e o ícone é **#FFFFFF**; sem `filled`, o fundo é transparente/superfície (#18181B) e o ícone usa a cor da categoria ou o muted #A1A1AA.
- Ícone `PierreIcon` com `size = max(mín, round(40×0,45)) = 18`. Emoji em `Text` com `fontSize = round(40×0,46) = 18` e `lineHeight = round(40×0,5) = 20`.

### 4.8 Paleta categoria → cor/ícone (`VISION_CATEGORY_DISPLAY_MAP` / `VISION_FILTER_CATEGORY_GROUPS`, L2724176–2724366) [O]
Busca: `normalizeCategoryValue(v) = (v ?? '').normalize('NFD').replace(/[̀-ͯ]/g,'').trim().toLowerCase()`, testando em ordem `label`, `name`, `title` e `id`. Sem correspondência, vale a cor do servidor.

| Grupo (id) | Cor | Ícone do grupo | Subcategorias → ícone |
|---|---|---|---|
| Alimentação (`alimentacao`) | **#F97316** | cartShopping | Supermercado → cartShopping · Alimentos e bebidas → fastFoodDrinkCoffeeCup · Restaurantes, bares (e lanchonetes) → kitchenCookingForkKnife · Delivery de alimentos → transportationScooter |
| Compras (`compras`) | **#EC4899** | shoppingEcommerceShoppingBagPackagePack | Compras online → shoppingEcommerceBasketGlobe · Eletrônicos → computersDevicesElectronicsIphone · Pet Shops e veterinários → petsAnimalsAnimalPrint_1 · Vestuário → clothesAccessoriesTShirt · Artigos infantis → babyChildrenToysBabyChild · Livraria → interfaceEssentialBook · Artigos esportivos → sportVolleyball · Papelaria → officeWorkNotebook · Cashback → moneyDollarMoneyCurrencyBanknotesFastSend |
| Saúde (`saude`) (+ "Saúde e bem-estar", "Bem-estar") | **#10B981** | healthMedicineHeartCheckmark | Dentista → healthMedicineTeeth · Ótica → interfaceEssentialGlassesInvisible · Hospitais, clínicas e laboratórios → healthMedicineHospitalBoardSquare · Farmácia → healthMedicineHospitalCenterDrugsPill · Academia e centros de lazer → sportDumbbell · Prática de esportes → sportYogaMat |
| Moradia (`moradia`) | **#6366F1** | buildingsHousing | Aluguel → buildingsHome · Celular → deviceSmartphone · Eletricidade → utilityElectricity · Gás → utilityGas · Serviços de utilidade pública → miscBuild · TV → deviceTv · Internet / Telecomunicação → wifiOn · Água → utilityWater |
| Serviços digitais (`servicos-digitais`) | **#A855F7** | interfaceEssentialMonitorMinimalBig | Jogos e videogames → computersDevicesElectronicsJoystickXbox · Streaming de vídeo → musicAudioPlayNext · Streaming de música → musicAudioNote |
| Transporte (`transporte`) | **#F59E0B** | carServicePartsCar | Táxi e transporte privado urbano → transportationCarTaxi · Transporte público → transportationRailroadTrainStyle_2 · Aluguel de veículos → carServicePartsCar · Aluguel de bicicleta(s) → transportationBikeBicycle · Serviços automotivos → carServicePartsSettingSquare · Postos de gasolina → carServicePartsGasStation · Estacionamentos → carServicePartsParkingParkingsign · Pedágios e pagamentos de/no veículo(s) → cardPass · Taxas e impostos sobre veículo(s) → carServicePartsVehicleTaxes · Manutenção de veículos → carServicePartsBatteryEectricityGenerator · Multas de trânsito → carServicePartsTrafficFine |
| Lazer e entretenimento (`lazer`) (+ "Lazer") | **#3B82F6** | beachUmbrellaLounge | Viagens → travelAirplanePlaneTripTravel · Aeroportos e cias. aéreas → miscSchoolBag · Hospedagem → hotelsHotelSquare · Programas de milhagem / Bilhetes → travelAirlineTicket · Passagem de ônibus → travelPlaneBoardingPass · Estádios e arenas → landmarksPlacesStadium · Museus e pontos turísticos → landmarksPlacesChristTheReedemer · Cinema, Teatro e Concertos → entertainmentCasinoShowTheaterMasks |
| Educação (`educacao`) | **#06B6D4** | schoolLearningGraduateHat | Cursos online → schoolLearningBrowserOpenBook · Universidade → buildingConstructionSchoolBuilding · Escola → miscSchoolBag · Creche → babyChildrenToysToysCubes |
| Apostas (`apostas`) (+ "Apostas e jogos") | **#EF4444** | entertainmentCasinoCasinoSlotMachine | Loteria, Apostas online → idem |

- Toda subcategoria herda a **cor do grupo**. O mapa plano (`parcial/catmap.txt`, 74 chaves) inclui sinônimos ("Restaurantes, bares e lanchonetes", "Pedágios e pagamentos no veículo", "Aluguel de bicicletas", "Taxas e impostos sobre veículos" etc.).
- Categorias fora do mapa (ex.: "Transferências", "Outros", "Impostos") usam `c.color` do servidor, ou `#A1A1AA`/`walletMoney` nas subcategorias.
- Existe outra paleta, `getCategoryColor(i)` = `['#10B981','#3B82F6','#F59E0B','#EF4444','#8B5CF6','#EC4899','#06B6D4','#F97316'][i % 8]` (L2205130), usada em telas de transações, **não** neste widget. O degradê `HEATMAP_BAR_GRADIENT_*` do mesmo módulo é usado por `HeatmapBarGradient` (L2209735), também fora dos widgets.

### 4.9 `WidgetModeToggle` (memo, L2863541) [O]
```
HStack "gap-2"                                                     // 7
 ├─ PierreButton size="mini" variant={mode === 'categories' ? 'primary' : 'outline'}
 │     onPress={() => onModeChange?.('categories')} testID="vision-spending-category-widget-mode-categories" → "Categorias"
 └─ PierreButton size="mini" variant={mode === 'groups' ? 'primary' : 'outline'}
       onPress={() => onModeChange?.('groups')}     testID="vision-spending-category-widget-mode-groups"     → "Grupos"
```
- O botão "Categorias" vem **primeiro**, mas o modo inicial é `groups`. PierreButton mini: pílula com padding 10,5×7, texto `text-sm` medium (16), altura ≈33; `primary` usa o fundo da marca (#cfff04) e `outline` usa contorno de 1 px muted-foreground (design system, seção Componentes).
- Trocar o modo remonta o `Animated.View` (key = modo): o conteúdo antigo sai com FadeOutUp (160 ms) e o novo entra com FadeInDown (220 ms), ambos com `Easing.out(cubic)`. Ao entrar em `categories` dispara a query de subcategorias; enquanto ela carrega, o bloco de linhas mostra `WidgetLoadingState content`.

### 4.10 Strings pt-BR (spending-by-category)
`Gastos por categoria` · `Gastos • {período}` · `Gastos por categoria • {período}` · `este mês` (default de período) · `R$ 0,00` · `0 categorias` · `Gastos esse mês` (sic, "esse") · `Não há lançamentos` · `Categorias` · `Grupos` · `Ver mais {n} grupo|grupos` · `Ver mais {n} categoria|categorias` · `Categoria` (nome default) · `{n}%`.

---

## 5. Versão da home do **cartão** (fora da Vision, só para comparação) [O L3078245–3079300]
- `SpendingByCategoryWidget` (L3078245) é **outro componente**: `Widget2x2 testID="spending-by-category-widget" visualPlacement="middle" label="Gastos por categoria"`, `visual = <CategoryBar categories/>`, `valueSlot = PrivacyCurrencyText (20, '800', disableMinWidth, testID 'spending-by-category-widget-value')`, valor `'—'` quando falta. Vazio: `WidgetEmptyStateSlot compact`.
- `CategoryBar` (L3078530) usa **todas** as categorias (sem `slice`), `flex = value/total` **sem** o mínimo de 0,05, container `{flexDirection:'row', height:16, gap:4}` e `Segment {flex, height:16, borderRadius, backgroundColor}`. Quando o total é 0, mostra um único segmento `flex:1 #27272A`.
- `SpendingByCategoryWidgetFull` (L3078768): `WidgetFull testID="spending-by-category-widget-full" title="Gastos por categoria"`, `VStack gap-4` [text-3xl extrabold, CategoryBar, HStack justify-end → text-base muted subtitle]. Vazio: "Sem gastos por categoria".
- Há ainda um **widget de tela inicial iOS** (`@expo/ui` SwiftUI, L3063399) que replica a versão Vision: barra com até 3 segmentos de largura mínima de 5% (normalizada, com `containerRelativeFrame`), altura 16, gap 4, raio 6; avatar de 40 com ícone de 18; total com redução de fonte para 0,85 (≥ 10 caracteres) e 0,72 (≥ 12); "Ver mais N categoria(s)". Isso confirma as mesmas regras de layout.


---

## 6. Snippets de referência (React Native + `react-native-svg`) [I: reimplementação]
Reproduzem a geometria nativa de §2.8–§2.11 e o layout de §4. Usam a largura medida com `onLayout`, já que o Canvas nativo também trabalha sobre a largura real da view.

### 6.1 Área (half/wide) e comparação (full)
```tsx
import React, { useState } from 'react';
import { View, LayoutChangeEvent } from 'react-native';
import Svg, { Path, Line, Circle, Defs, LinearGradient, Stop } from 'react-native-svg';

type Pt = { x: number; value: number };
const ACCENT = '#B8FF12', TRACK = '#27272A', PREV = '#5C5C68';

function smoothPath(p: [number, number][]) {           // Catmull-Rom → Bézier (tensão 1/6), = smoothPath nativo
  if (!p.length) return '';
  let d = `M${p[0][0]},${p[0][1]}`;
  for (let i = 0; i < p.length - 1; i++) {
    const p0 = p[Math.max(i - 1, 0)], p1 = p[i], p2 = p[i + 1], p3 = p[Math.min(i + 2, p.length - 1)];
    const c1x = p1[0] + (p2[0] - p0[0]) / 6, c1y = p1[1] + (p2[1] - p0[1]) / 6;
    const c2x = p2[0] - (p3[0] - p1[0]) / 6, c2y = p2[1] - (p3[1] - p1[1]) / 6;
    d += ` C${c1x},${c1y} ${c2x},${c2y} ${p2[0]},${p2[1]}`;
  }
  return d;
}

export function PierreAreaChartSvg({ current, previous, height, width: fixedW }:
  { current: Pt[]; previous?: Pt[]; height: number; width?: number }) {
  const [w, setW] = useState(fixedW ?? 0);
  const onLayout = (e: LayoutChangeEvent) => !fixedW && setW(e.nativeEvent.layout.width);
  const top = 8, bottom = 4, left = 0, right = 8, endPad = 8;
  const all = [...current, ...(previous ?? [])];
  const minX = all.length ? Math.min(...all.map(p => p.x)) : 0;
  const maxX = all.length ? Math.max(...all.map(p => p.x)) : 1;
  const yMax = Math.max(Math.max(...all.map(p => p.value), 1) * 1.12, 1);
  const plotW = Math.max(w - left - right, 1), drawW = Math.max(plotW - endPad, 1);
  const plotH = Math.max(height - top - bottom, 1);
  const map = (pts: Pt[]) => pts.map(p => [
    left + Math.min(Math.max((p.x - minX) / Math.max(maxX - minX, 1), 0), 1) * drawW,
    top + plotH - Math.min(Math.max(p.value / yMax, 0), 1) * plotH,
  ] as [number, number]);
  const cur = map(current), prev = previous?.length ? map(previous) : [];
  const line = smoothPath(cur);
  const last = cur[cur.length - 1];
  const fill = cur.length ? `${line} L${last[0]},${height} L${cur[0][0]},${height} Z` : '';
  return (
    <View style={{ height, width: fixedW ?? '100%' }} onLayout={onLayout}>
      {w > 0 && (
        <Svg width={w} height={height}>
          <Defs>
            <LinearGradient id="pierreFill" x1="0" y1="0" x2="0" y2={height} gradientUnits="userSpaceOnUse">
              <Stop offset="0" stopColor={ACCENT} stopOpacity={96 / 255} />
              <Stop offset="1" stopColor={ACCENT} stopOpacity={0} />
            </LinearGradient>
          </Defs>
          <Line x1={left} y1={height - 0.5} x2={left + plotW} y2={height - 0.5} stroke={TRACK} strokeWidth={1} />
          {!!fill && <Path d={fill} fill="url(#pierreFill)" />}
          {prev.length > 0 && (
            <Path d={smoothPath(prev)} stroke={PREV} strokeWidth={1.4} strokeDasharray="7,7"
                  strokeLinecap="round" strokeLinejoin="round" fill="none" />
          )}
          <Path d={line} stroke={ACCENT} strokeWidth={1.8} strokeLinecap="round" strokeLinejoin="round" fill="none" />
          {last && <Circle cx={last[0]} cy={last[1]} r={4.5} fill={ACCENT} />}
        </Svg>
      )}
    </View>
  );
}

// Uso, igual ao MonthlyAreaChart/MonthlyComparisonChart:
const toPts = (pts: { value: number }[], forceEps: boolean) =>
  pts.map((p, i) => ({ x: i, value: forceEps ? 0.001 : p.value }));
// half: <View style={{ marginLeft: -5 }}><PierreAreaChartSvg current={toPts(points, !points.some(p => p.value > 0))} height={72} width={124}/></View>
// wide: <PierreAreaChartSvg current={...} height={76}/>
// full: <PierreAreaChartSvg current={...} previous={previousData.map((p, i) => ({ x: i, value: p.value }))} height={150}/>
```

### 6.2 Barras (half, 90×72)
```tsx
import React from 'react';
import Svg, { Rect, Ellipse, Defs, LinearGradient, Stop } from 'react-native-svg';

export function MonthlyBarsSvg({ points }: { points: { value: number; label?: string }[] }) {
  const W = 90, H = 72, top = 4, bottom = 4, gap = 10;
  const bw = Math.max(10, 12);                                  // mínimo nativo de 12 dp
  const maxV = Math.max(...points.map(p => p.value), 1);
  const bars = points.slice(0, 4);
  const yMax = Math.max(maxV * 1.08, 1);                        // surfaceMaxValue = maxV
  const plotH = H - top - bottom, base = H - bottom;
  const content = bars.length * bw + Math.max(bars.length - 1, 0) * gap;
  const startX = (Math.max(W, content + 2 * gap) - content) / 2;   // 10 (o nativo permite rolar 8 px; aqui é ignorado)
  return (
    <Svg width={W} height={H}>
      {bars.map((b, i) => {
        const x = startX + i * (bw + gap);
        if (b.value <= 0) {                                     // placeholder: círculo tracejado
          const s = Math.min(14, bw);
          return <Ellipse key={i} cx={x + bw / 2} cy={base - s / 2} rx={s / 2 - 0.6} ry={s / 2 - 0.6}
                          stroke="#5C5C68" strokeWidth={1.2} strokeDasharray="4,4" fill="none" />;
        }
        const h = Math.max(Math.min(b.value / yMax, 1) * plotH, bw);
        return (
          <React.Fragment key={i}>
            <Defs>
              <LinearGradient id={`g${i}`} x1={x} y1={base - h} x2={x + bw} y2={base} gradientUnits="userSpaceOnUse">
                <Stop offset="0" stopColor="#F0F30D" /><Stop offset="1" stopColor="#95FF48" />
              </LinearGradient>
            </Defs>
            <Rect x={x} y={base - h} width={bw} height={h} rx={bw / 2} fill={`url(#g${i})`} />
          </React.Fragment>
        );
      })}
    </Svg>
  );
}
```

### 6.3 Bloquinhos
```tsx
export function MonthlyBlocks({ points, compact }: { points: { value: number }[]; compact: boolean }) {
  const count = compact ? 12 : 39, size = compact ? 18 : 15;
  const cells = [...points.slice(0, count), ...Array.from({ length: Math.max(0, count - points.length) }, () => ({ value: 0 }))];
  const max = Math.max(...cells.map(c => c.value), 1);
  return (
    <View style={{ flexDirection: 'row', flexWrap: 'wrap', gap: 4, maxWidth: compact ? 86 : 246 }}>
      {cells.map((c, i) => (
        <View key={i} style={{ width: size, height: size, borderRadius: 4,
          backgroundColor: c.value > 0 ? '#B8FF12' : '#27272A',
          opacity: c.value > 0 ? 0.32 + (c.value / max) * 0.68 : 1 }} />
      ))}
    </View>
  );
}
```

### 6.4 Barra segmentada de categorias
```tsx
export function SegmentedCategoryBar({ categories, muted = false, height = 16 }:
  { categories: { name?: string; value: number; color?: string }[]; muted?: boolean; height?: number }) {
  const total = categories.reduce((s, c) => s + Math.max(0, c.value), 0);
  if (muted || total <= 0)
    return <View style={{ height, width: '100%', borderRadius: 8, backgroundColor: '#27272A' }} />;
  return (
    <View style={{ flexDirection: 'row', height, gap: 4 }}>
      {categories.slice(0, 3).map((c, i) => (
        <View key={c.name ?? i} style={{ borderRadius: 6, flex: Math.max(0.05, c.value / total),
          backgroundColor: resolveCategoryColor(c.name, c.color) }} />
      ))}
    </View>
  );
}

const GROUP_COLORS: Record<string, string> = {           // §4.8 (grupos; as subcategorias herdam a cor do grupo)
  'alimentacao': '#F97316', 'compras': '#EC4899', 'saude': '#10B981', 'moradia': '#6366F1',
  'servicos digitais': '#A855F7', 'transporte': '#F59E0B', 'lazer e entretenimento': '#3B82F6',
  'educacao': '#06B6D4', 'apostas': '#EF4444', /* + as 74 chaves de catmap.txt */
};
const norm = (v?: string) => (v ?? '').normalize('NFD').replace(/[̀-ͯ]/g, '').trim().toLowerCase();
export const resolveCategoryColor = (name?: string, fallback?: string) => GROUP_COLORS[norm(name)] ?? fallback;
```

---

## 7. Dúvidas abertas
1. **Formato do payload do servidor** (`/api/mobile/vision/widgets/{monthly-spending|spending-by-category}/data`): a estrutura foi inferida pelos campos que o app lê. O APK não diz quais buckets o servidor manda para `bar` (semanas? "S1…S4"?) nem para `blocks` (dias com contagem de transações, como no merge local, ou valores em R$?).
2. **`drawW` do gráfico de área**: o smali subtrai `endContentPadding` da largura útil, mas a alocação de registradores deixa ambíguo se o valor vai para `mapPoints` ou só para o eixo. O efeito máximo é de 8 px no x do último ponto.
3. **Posição y do eixo** da área: inferida como a borda inferior (`y = chartH`). Pode ser `chartH − bottomPadding` (4 px acima).
4. **Rótulos das barras no half**: pela geometria, ficam fora da view (`y = chartH + 20dp` com `labelAreaHeight = 0`). Não verifiquei se há `clipChildren=false` no container que os deixaria visíveis.
5. **Rolagem horizontal das barras**: com 4 barras de 12 dp e gap 10 o conteúdo mede 98 dp para 90 dp úteis, então o nativo aceita arrastar 8 dp (`onTouchEvent` com `touchSlop`). Não sei se isso é intencional.
6. **Full com variante `blocks`/`bar`**: o full sempre desenha `MonthlyComparisonChart`, mas o merge recebe a `chartVariant` salva. Se o usuário escolheu `blocks` num tamanho menor e redimensionou para full, `data` pode vir com contagens em vez do acumulado e sem `previousData`. É um caso de borda possivelmente não tratado pelo app.
7. **`CategoryAvatar` não preenchido**: as cores de fundo e de ícone quando `filled = false` foram inferidas (o widget só usa `filled`). O piso do `Math.max` no tamanho do ícone também não foi lido; para 40 px o resultado é 18 de qualquer forma.
8. **`PierreButton` mini**: a altura (≈33) vem do design system [I]. Confira o token de `primary` no tema antes de replicar.
9. **Tela de destino** (`/(protected)/vision/spending-by-category` com `params.chartMode`): não analisada aqui. Também não verifiquei como o `VisionWidgetRenderer` repassa `onPress(args)` com parâmetros para o `router` (provavelmente `router.push({pathname, params})`).
10. **Ordenação/"Outros"** em `spending-by-category`: o app não ordena nem agrupa a cauda, então depende inteiramente do servidor.
