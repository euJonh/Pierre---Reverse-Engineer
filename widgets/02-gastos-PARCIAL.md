> **Status: PARCIAL.** A investigação foi interrompida em 19/10/2026. Faltam o desenho dos 3 gráficos de `monthly-spending` (área, barras, bloquinhos) e todo o widget `spending-by-category`. Veja `widgets/README.md`.

# W2 — Widgets `monthly-spending` ("Gastos do mês") e `spending-by-category` ("Gastos por categoria") (Pierre v4.0.6)

> Análise estática do bundle Hermes v98 (`_trabalho/out.js`) e, quando indicado, do dex nativo.
> **[O]** = observado no código (`Lnnnn` = linha de `out.js`; `Mnnnn` = id de módulo Metro); **[I]** = inferido.
> Unidades: px = dp. NativeWind com **rem = 14**. Fundação (Widget2x2/WidgetWide/WidgetFull, WidgetShell, estados vazio/erro/carregando,
> PrivacyText/PrivacyCurrencyText, tokens dark, tabela de classes NativeWind) está em **`w0-base.md`** — referenciada como `W0 §n`.
> Busca de dados comum (`useVisionSystemWidgetData`, `GET /api/mobile/vision/widgets/{id}/data`) está em **`w1-wallet-saldo-fluxo.md` §1** — referenciada como `W1 §1`.

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

