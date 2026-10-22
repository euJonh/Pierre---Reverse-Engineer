# W3 — Widgets `available-limit`, `installments`, `cards`, `patrimony` e `recent-transactions` (Pierre v4.0.6)

Análise estática do bundle Hermes decompilado (`_trabalho/out.js`). **[O]** = observado no código (com `Lnnnn` = linha do `out.js`; `w3:nnnn` = linha do intermediário `_trabalho/parcial/w3.js`, que é uma cópia do trecho L2852600–L2863500 aprox.); **[I]** = inferido.

Referências que **não** são repetidas aqui:
- `widgets/00-base.md`: renderer, `Widget2x2` (half, 200 px), `WidgetWide` (170 px), `WidgetFull`, `WidgetEmptyState(Slot)`, `WidgetErrorState`, `WidgetLoadingState`, tokens dark (§13), NativeWind rem=14 (§14), `HIDDEN_VALUE = 'R$ ••••'` e `useHideValues` (§10), `PrivacyCurrencyText` (§11.4), `parseBRLToCents`/`format` BRL (§12).
- `widgets/01-carteira-saldo-fluxo.md`: §1 `useVisionSystemWidgetData(id, size)` (busca comum, react-query, endpoint), §5 `BankLogo` e `resolveBankColor`.
- `widgets/02-gastos.md`: paleta de categorias e gráficos nativos.

---

## 0. Registro (`VISION_WIDGET_REGISTRY`, L772538–L772585) [O]

| id | label | size padrão | supportedSizes | tapAction / navigateTo |
|---|---|---|---|---|
| `available-limit` | Limite disponível | half | `['half','wide']` | navigate → `/(protected)/vision/card-invoices?filterKey=limits` (L772565) |
| `installments` | Parcelamentos | half | `['half','wide','full']` | navigate → `/(protected)/vision/card-invoices?filterKey=installments` (L772569) |
| `patrimony` | Patrimônio | half | `['half','full']` | **sem** tapAction/navigateTo (L772573) |
| `cards` | Cartões | half | `['half','full']` | navigate → `/(protected)/vision/card-invoices?filterKey=invoices` (L772577) |
| `recent-transactions` | Transações recentes | **full** | `['half','wide','full']` | navigate → `/(protected)/(drawer)/(tabs)/transactions` (L772581) |

Listas auxiliares logo abaixo (L772620–L772660) [O]:
- Ordem padrão/“sistema”: `['wallet','account-balance','cash-flow','cdb-investment','available-limit','installments','cards','monthly-spending','recent-transactions','spending-by-category','subscriptions']`.
- `['cards','cdb-performance']` → filtrados do **picker** (o filtro remove ids que começam com `ai-` ou estão nessa lista). Daí “cards aposentado do picker”: continua renderizável se já estiver no layout salvo, mas não aparece para adicionar.
- `['patrimony']` → lista de ids **ocultos** (nunca exibidos) [O: `_closure0_slot7`; uso exato em W0].

Os 5 widgets usam `useVisionSystemWidgetData(id, size)` (W1 §1) e depois mesclam os dados do cartão **Pierre** (cartão próprio) por funções `mergePierreCardInto*` (módulo de L2849800–L2850700).

---

## 1. `available-limit` — “Limite disponível”

### 1.1 Mapa de módulos [O]
| Módulo | Linha | Conteúdo |
|---|---|---|
| M9006 | L2852913 | `AvailableLimitWidgetConnected` + `StyleSheet` `{half:{height:200,alignItems:'center',justifyContent:'center',padding:16}, wide:{height:170, …mesmo}}` + defaults |
| M9007 | — | reexport de `AvailableLimitWidget` |
| M9008 | L2853333 | `AvailableLimitWidget` (memo) + constantes do donut + estilos da barra |
| M9010 | w3:1630 | `LIMIT_USAGE_COLORS`, `LIMIT_USAGE_HIGH_THRESHOLD_PERCENT`, `isLimitUsageHigh`, `limitUsageActiveColor` |
| M9011 | w3:1740 | `AVAILABLE_LIMIT_WIDGET_COPY`, `AVAILABLE_LIMIT_WIDGET_COLORS`, `AVAILABLE_LIMIT_WIDGET_TEST_IDS` |
| M9012 | w3:1794 | `useAvailableLimitData(size)` |
| M9013 | w3:1936 | `useVisionPierreCardInvoices()` (dados do cartão Pierre; também usado por `cards`/`installments`) |
| M9000 | L2850222 | `mergePierreCardIntoAvailableLimitWidget(server, pierreView)` |

Dependências de M9008: `react-native` (View/StyleSheet), React, `react-native-svg` (`Svg` default + `Circle`), gluestack `HStack`, store de privacidade (M4428), `Text` (gluestack), `PrivacyCurrencyText` (M8991), `useThemeColors`, base dos widgets (M8976) e `parseBRLToCents`.

### 1.2 Constantes [O]
```ts
// M9011
export const AVAILABLE_LIMIT_WIDGET_COPY = {
  label: 'Limite usado',
  highUsageLabel: 'Limite',
  highUsageAction: 'Aumentar Limite',
  availablePrefix: 'Disp:',
  totalPrefix: 'Total:',
  emptyValue: '—',            // travessão U+2014
};
export const AVAILABLE_LIMIT_WIDGET_COLORS = { track: '#27272A' };
export const AVAILABLE_LIMIT_WIDGET_TEST_IDS = {
  widget: 'available-limit-widget', value: 'available-limit-widget-value',
  donut: 'available-limit-widget-donut', wideBar: 'available-limit-widget-bar',
  wideBarFill: 'available-limit-widget-bar-fill',
  wideAvailable: 'available-limit-widget-wide-available', wideTotal: 'available-limit-widget-wide-total',
};
// M9010
export const LIMIT_USAGE_COLORS = { low: '#CFFF04', medium: '#F59E0B', high: '#EF4444' };
export const LIMIT_USAGE_HIGH_THRESHOLD_PERCENT = 90;
export const isLimitUsageHigh = (p) => Number.isFinite(p) ? p >= 90 : false;
export function limitUsageActiveColor(p) {
  if (!Number.isFinite(p)) return LIMIT_USAGE_COLORS.low;
  if (p >= 90) return LIMIT_USAGE_COLORS.high;     // vermelho
  if (p > 70)  return LIMIT_USAGE_COLORS.medium;   // âmbar
  return LIMIT_USAGE_COLORS.low;                   // lima da marca
}
// M9008 (donut)
const STROKE = 13, R = 34.5, C = 2 * Math.PI * R /* 216,77 */, SIZE = 82, CENTER = 41, OFFSET = C * 0.25 /* 54,19 */;
```
Thresholds [O]: **≤ 70 %** lima `#CFFF04`; **> 70 % e < 90 %** âmbar `#F59E0B`; **≥ 90 %** vermelho `#EF4444` e a cópia muda para o modo “alto uso”.

### 1.3 Dados [O]
`useAvailableLimitData(size)` (w3:1794):
1. `useVisionSystemWidgetData('available-limit', size)` (W1 §1). Payload do servidor (strings já formatadas em BRL): `{ availableAmount, totalLimit, usedAmount, usedPercentage }`.
2. `useVisionPierreCardInvoices()` → `{ view, isSettled }`. Enquanto `!isSettled`, o hook segura `isLoading = true`; `isFetched = query.isFetched && isSettled`.
3. Se `isSettled`: `data = mergePierreCardIntoAvailableLimitWidget(query.data, view) ?? undefined`, senão `undefined`.
4. Retorna `{ data, isLoading: query.isLoading || !isSettled, isError: query.isError, isFetched }`.

`mergePierreCardIntoAvailableLimitWidget(server, pierre)` (L2850222):
- Se não há `pierre` ou `!pierre.cardInfo.hasAvailableLimit` → devolve `server ?? null` intacto.
- Senão soma em **reais**: `total = parseBRLToReais(server?.totalLimit) + cardInfo.creditLimit`; `used = parseBRLToReais(server?.usedAmount) + cardInfo.usedAmount`; `available = parseBRLToReais(server?.availableAmount) + cardInfo.availableAmount`.
- `pct = total > 0 ? Math.round(used / total * 100) : 0`.
- Retorna `{ ...server, availableAmount: fmt(available), totalLimit: fmt(total), usedAmount: fmt(used), usedPercentage: clamp(pct, 0, 100) }` (`fmt` = formatador BRL do módulo, `Intl` pt-BR → `R$ 1.234,56`).

`AvailableLimitWidgetConnected` (L2852913):
- `size` default `'half'`, `animate` default `false`.
- Campos ausentes caem nos defaults `{availableAmount:'R$ 0,00', totalLimit:'R$ 0,00', usedAmount:'R$ 0,00', usedPercentage:0}`.
- `isLoading = isLoading || !isFetched`.
- **Erro**: `<View testID="available-limit-error" className="rounded-3xl bg-surface" style={size==='wide' ? styles.wide : styles.half}><WidgetErrorState compact={size==='half'}/></View>` (rounded-3xl = 21 px [I: 1,5rem×14]; height 200/170, padding 16, centralizado).

### 1.4 Lógica do componente `AvailableLimitWidget` (L2853333) [O]
```ts
const pct = Number.isFinite(usedPercentage) ? Math.min(100, Math.max(0, usedPercentage)) : 0;
const color = limitUsageActiveColor(pct);  const high = isLimitUsageHigh(pct);
const label   = high ? 'Limite' : 'Limite usado';
const caption = high ? 'Aumentar Limite' : `Disp: ${availableAmount}`;
const isEmpty = !isLoading && usedPercentage === 0 && (!totalLimit || totalLimit === 'R$ 0,00');
const hidden  = useHideValues();   const fg = useThemeColors().foreground; // #fafafa
```
Ordem de decisão: `isLoading` → esqueleto; `isEmpty` → vazio; senão conteúdo.

**Donut** (visual do half) — `react-native-svg`:
```tsx
<Svg width={82} height={82} testID="available-limit-widget-donut"
     accessibilityRole="progressbar" accessibilityValue={{min:0,max:100,now:Math.round(pct)}}>
  <Circle cx={41} cy={41} r={34.5} stroke="#27272A" strokeWidth={13} fill="none" />
  {pct > 0 && <Circle cx={41} cy={41} r={34.5} stroke={color} strokeWidth={13} fill="none"
     strokeLinecap="butt" strokeDasharray={`${arc} ${C - arc}`} strokeDashoffset={C * 0.25} />}
</Svg>   // arc = pct/100 * C
```
Sem rotação/transform: o `strokeDashoffset = C/4` faz o arco começar às 12 h e crescer no sentido horário [I: comportamento padrão do SVG]. Ponta reta (`butt`). Sem texto no centro. Sem animação.

**Barra** (visual do wide) — estilos (M9008, final):
```ts
footer: { gap: 12 },
track:  { height: 7, borderRadius: 4, overflow: 'hidden', backgroundColor: '#27272A' },
fill:   { height: '100%', borderRadius: 4 },   // + { width: `${pct}%`, backgroundColor: color }
```
```tsx
<View style={footer}>
  <View style={track} testID="available-limit-widget-bar">
    <View style={[fill, {width:`${pct}%`, backgroundColor: color}]} testID="available-limit-widget-bar-fill"/>
  </View>
  <HStack className="items-center justify-between">
    <Text allowFontScaling={false} className="text-base text-muted-foreground" numberOfLines={1}
          adjustsFontSizeToFit minimumFontScale={0.75} testID="…-wide-available">
      {hidden ? 'R$ ••••' : caption}</Text>
    <Text … mesmas props … testID="…-wide-total">{`Total: ${hidden ? 'R$ ••••' : totalLimit}`}</Text>
  </HStack>
</View>
```
Obs.: com valores ocultos e uso alto, o texto da esquerda vira `R$ ••••` (o código troca `caption` inteiro pelo `HIDDEN_VALUE`, inclusive “Aumentar Limite”) [O w3:946–961].

Não há **bandeira** de cartão neste widget [O: nenhum logo/ícone na árvore].

### 1.5 Layout por tamanho [O]
| Estado | half (`Widget2x2`, 200 px) | wide (`WidgetWide`, 170 px) |
|---|---|---|
| Carregando | `Widget2x2 visual={<WidgetLoadingState variant="visual"/>} label="Limite usado" value="" subtitle=""` | `WidgetWide label="Limite usado" value="" visual={<WidgetLoadingState variant="content"/>}` |
| Vazio | `Widget2x2 visual={<WidgetEmptyStateSlot emptyState={emptyState} compact/>} label="Limite usado" value="—" subtitle=""` | `WidgetWide label="Limite usado" value="—" visual={footer}` (a barra fica com 0 %, textos “Disp: R$ 0,00” / “Total: …”) |
| Conteúdo | `Widget2x2 visual={donut} label={label} valueSlot={<PrivacyCurrencyText animate={false} valueCents={parseBRLToCents(usedAmount)} hidden={hidden} fontSize={20} fontWeight="800" color="#fafafa" disableMinWidth compactOnOverflow testID="available-limit-widget-value"/>} subtitle={hidden ? '' : caption}` | `WidgetWide label={label} value={hidden ? 'R$ ••••' : usedAmount} visual={footer}` |

Todos com `testID="available-limit-widget"`, `onPress` e `animate` repassados. Tipografia dos rótulos/valores = a do `Widget2x2`/`WidgetWide` (W0 §5–7). Os textos da barra: `text-base` → Geist-Regular 14/21, `#a1a1aa`, autoshrink até 75 %.

### 1.6 Strings pt-BR [O]
`Limite usado`, `Limite`, `Aumentar Limite`, `Disp:`, `Total:`, `—`, `R$ 0,00`.

---

### 1.7 Snippet TSX de referência (`available-limit`, RN puro)
```tsx
const COLORS = { low: '#CFFF04', medium: '#F59E0B', high: '#EF4444', track: '#27272A' };
const usageColor = (p: number) => (p >= 90 ? COLORS.high : p > 70 ? COLORS.medium : COLORS.low);
const R = 34.5, C = 2 * Math.PI * R;

export function LimitDonut({ pct }: { pct: number }) {
  const arc = (pct / 100) * C;
  return (
    <Svg width={82} height={82}>
      <Circle cx={41} cy={41} r={R} stroke={COLORS.track} strokeWidth={13} fill="none" />
      {pct > 0 && (
        <Circle cx={41} cy={41} r={R} stroke={usageColor(pct)} strokeWidth={13} fill="none"
          strokeDasharray={`${arc} ${C - arc}`} strokeDashoffset={C * 0.25} />
      )}
    </Svg>
  );
}

export function LimitBar({ pct, left, right }: { pct: number; left: string; right: string }) {
  const t = { fontFamily: 'Geist-Regular', fontSize: 14, lineHeight: 21, color: '#a1a1aa' };
  return (
    <View style={{ gap: 12 }}>
      <View style={{ height: 7, borderRadius: 4, overflow: 'hidden', backgroundColor: COLORS.track }}>
        <View style={{ height: '100%', borderRadius: 4, width: `${pct}%`, backgroundColor: usageColor(pct) }} />
      </View>
      <View style={{ flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between' }}>
        <Text style={t} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.75}>{left}</Text>
        <Text style={t} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.75}>{right}</Text>
      </View>
    </View>
  );
}
// half: <Widget2x2 visual={<LimitDonut pct={pct}/>} label={label} valueSlot={<PrivacyCurrencyText … fontSize 20 / 800/>} subtitle={caption}/>
// wide: <WidgetWide label={label} value={usedAmount} visual={<LimitBar pct={pct} left={caption} right={`Total: ${totalLimit}`}/>}/>
```

### 1.8 Estados vazio/erro (resumo) [O]
- **Carregando**: esqueleto `WidgetLoadingState` (variant `visual` no half, `content` no wide), rótulo “Limite usado”.
- **Vazio**: `usedPercentage === 0` e `totalLimit` vazio ou `'R$ 0,00'` → valor `—`; o slot visual é `WidgetEmptyStateSlot` com o `emptyState` que o renderer passa (texto padrão em W0 §8).
- **Erro**: caixa `rounded-3xl bg-surface` (#18181b, raio 21) com `WidgetErrorState` (compacto no half), `testID="available-limit-error"`.

---

## 2. `installments` — “Parcelamentos”

### 2.1 Mapa de módulos [O]
| Módulo | Linha (out.js / w3) | Conteúdo |
|---|---|---|
| M9014 | — | barrel: `InstallmentsWidgetConnected`, `useInstallmentsData` |
| M9015 | L2854810 / w3:2211 | `InstallmentsWidgetConnected` + defaults + `StyleSheet` de erro |
| M9016 | — | barrel: `InstallmentsWidget`, `InstallmentsWidgetFull` |
| M9017 | L2855251 / w3:2568–3070 | `InstallmentsIcon` (memo), `InstallmentsWidget` (half/wide, memo) + estilos da barra |
| M9018 | L2856127 / w3:3073–3696 | `InstallmentsIconLarge`, `InstallmentAvatar`, `InstallmentRow`, `InstallmentsWidgetFull` |
| M9019 | w3:3710 | `useInstallmentsData(size)` |
| M9020 | w3:3850 | `useVisionPierreCardInstallments()` |
| M9021 | w3:3993–5112 | `parseInstallmentInfo`, `formatEndDateLabel`, `buildInstallmentsBarMeta`, `buildPierreInstallmentPurchases`, `mergePierreCardIntoInstallmentsChart`, `buildPierreInstallmentWidgetItem`, `mergePierreCardIntoInstallmentsWidget`, `isPierreCardInstallmentPurchase`, `PIERRE_INSTALLMENT_PURCHASE_ID_PREFIX`, `PIERRE_CARD_AVATAR_CATEGORY_FALLBACK` |
| M8531 | L2722595 | `MerchantAvatar` (avatar das linhas; ver §6) |

Dependências de M9017: compiler-runtime (M978), React, `react-native`, `PierreIcon` (M2465), `HStack` (M2260), `Text` (M2262), base dos widgets (M9009), `PrivacyText` (M4432), store de privacidade (M4428), JSX (M228).

### 2.2 Constantes [O]
```ts
// M9021 (w3:5090–5112)
PIERRE_INSTALLMENT_PURCHASE_ID_PREFIX = 'cw-' + 'inst-'           // = 'cw-inst-'
PIERRE_CARD_AVATAR_CATEGORY_FALLBACK = { name: 'Outros', color: '#A1A1AA', iconKey: 'cardCredit' };
const MIN_BARS = 6, MAX_BARS = 12, MAX_WIDGET_ITEMS = 3;
const MONTHS = ['Janeiro','Fevereiro','Março','Abril','Maio','Junho','Julho','Agosto','Setembro','Outubro','Novembro','Dezembro'];
const MONTHS_SHORT = ['JAN','FEV','MAR','ABR','MAI','JUN','JUL','AGO','SET','OUT','NOV','DEZ'];
// Constantes do cartão Pierre (L2729815–L2729831) [O]
PIERRE_CARD_ID_PREFIX = 'cw-'; PIERRE_CARD_BANK_NAME = 'Cartão Pierre'; PIERRE_CARD_DISPLAY_NAME = 'Pierre One';
PIERRE_CARD_BANK_LOGO_URL = '/images/banks/pierre.svg';
PIERRE_CARD_INVOICE_CARD_ID = 'cw-pierre-one' /* [I: PIERRE_CARD_ID_PREFIX + 'pierre-one', L2850869–L2850875] */; PIERRE_CARD_BRAND = 'MASTERCARD';
// M9017
const BADGE_ICON_COLOR = '#F97316';  // laranja (orange-500)
const iconStyles = StyleSheet.create({
  container: { width: 48, height: 48, borderRadius: 24, alignItems: 'center', justifyContent: 'center' },
  badge: { position: 'absolute', bottom: 0, right: -2, width: 20, height: 20, borderRadius: 10, alignItems: 'center', justifyContent: 'center' },
}); // right = 4294967294 no bytecode = −2 (uint32)
const barStyles = StyleSheet.create({
  wideFooter: { gap: 12 },
  track: { height: 7, borderRadius: 4, overflow: 'hidden' },            // + className 'bg-muted' = #27272a
  fill:  { height: '100%', borderRadius: 4, backgroundColor: '#C8FF00' },
});
// M9015 (erro)
StyleSheet.create({ half:{height:200,…centro,padding:16}, wide:{height:170,…}, full:{minHeight:320, alignItems:'center', justifyContent:'center', padding:16} });
const DEFAULT = { count:'0', subtitle:'Nenhuma parcela', paidPercentage:0, paidAmount:'R$ 0,00', totalAmount:'R$ 0,00', items:[] };
```

### 2.3 Dados [O]
**Payload do servidor** (`useVisionSystemWidgetData('installments', size)`, W1 §1):
```ts
{ count: string,            // ex.: '4' (string!)
  subtitle: string,         // ex.: 'Em andamento' / 'Nenhuma parcela'
  paidPercentage: number, paidAmount: string /*BRL*/, totalAmount: string /*BRL*/,
  items?: Array<{ id, name, amount /*BRL do restante*/, installmentLabel /*'Parcela 3 de 10'*/,
                  progressPercentage: number, logoUrl?, bankName?, bankLogoUrl?, avatarFallbackIconKey? }> }
```
`items` aparece no payload quando o `size` pedido é `full` [I: a query recebe `size`; o merge só monta `items` quando `size==='full'` ou o servidor já mandou `items`].

**`useInstallmentsData(size)`** (w3:3710):
1. `query = useVisionSystemWidgetData('installments', size)`.
2. `{ purchases, isSettled } = useVisionPierreCardInstallments()`.
3. `data = isSettled ? (mergePierreCardIntoInstallmentsWidget(query.data, purchases, size) ?? undefined) : undefined`.
4. `isLoading = query.isLoading || !isSettled`; `isFetched = query.isFetched && isSettled`; `isError = query.isError`.

**`useVisionPierreCardInstallments()`** (w3:3850): pega `view.currentInvoice` de `useVisionPierreCardInvoices()` (§1.1, M9013); chama `useInvoiceItems(currentInvoice?.id, { enabled: !!currentInvoice })`; `isSettled = invoices.isSettled && (!currentInvoice || items.isSuccess || items.isError)`; `purchases = currentInvoice && items.data ? buildPierreInstallmentPurchases(items.data, currentInvoice) : []`.

**`parseInstallmentInfo(s)`** (w3:3993): regex `/^\s*(\d+)\s*\/\s*(\d+)\s*$/` sobre `item.installmentInfo` (ex.: `"3/10"`); devolve `{current, total}` só se ambos > 0 e `current ≤ total`; senão `null`.

**`buildPierreInstallmentPurchases(items, invoice, now = new Date())`** (w3:4270): para cada item da fatura do cartão Pierre:
- ignora se `installmentInfo` não casa ou `amountCents ≤ 0`;
- `invoicePaid = toInvoiceStatusBucket(invoice.status) === 'paid'`;
- `paid = invoicePaid ? current : current − 1`; `remaining = total − paid`; ignora se `remaining ≤ 0`;
- `value = centsToReais(amountCents)`; `paidAmount = value × paid`; `remainingAmount = value × remaining`;
- `progressPercent = min(100, round(current / total × 100))`;
- `endMonthKey = shiftMonthKey(toMonthKey(invoice.dueDate) ?? dateToMonthKey(now), total − current)`; `endDateLabel = formatEndDateLabel(endMonthKey)` → **`Última parcela em {Mês} de {AAAA}`** (ex.: “Última parcela em Março de 2027”);
- objeto: `{ id: 'cw-inst-' + (paymentId ?? id), transactionId: paymentId != null ? 'cw-' + paymentId : null, description: description?.trim() || 'Parcelamento', companyLogo: logoUrl ?? null, accountId: PIERRE_CARD_INVOICE_CARD_ID, accountName: 'Cartão Pierre', accountLogo: '/images/banks/pierre.svg', installmentValue, installmentValueFormatted, paidInstallments: current, totalInstallments: total, remainingInstallments: remaining, paidAmount(+Formatted), remainingAmount(+Formatted), progressPercent, endDateLabel, category: mccCategory ?? null }`.
  (Obs.: `paidInstallments` recebe `current`, não `paid` [O w3:4466].)

**`mergePierreCardIntoInstallmentsWidget(server, purchases, size)`** (w3:4883):
- Sem `purchases` → `server ?? null`.
- `count = (Number(server?.count) || 0) + purchases.length`.
- `paid = parseBRLToReais(server?.paidAmount) + Σ installmentValue × paidInstallments`.
- `total = parseBRLToReais(server?.totalAmount) + Σ installmentValue × totalInstallments`.
- `paidPercentage = total > 0 ? clamp(round(paid / total × 100), 0, 100) : 0`.
- Resultado `{ ...server, count: String(count), subtitle: count > 0 ? 'Em andamento' : 'Nenhuma parcela', paidPercentage, paidAmount: formatReais(paid), totalAmount: formatReais(total) }`.
- **Itens** (se `size === 'full'` ou `server.items` existe): `pierreItems = purchases.map(buildPierreInstallmentWidgetItem)`; `serverItems = server.items.filter(i => !pierreIds.has(i.id))`; `items = [...pierreItems, ...serverItems].sort((a,b) => parseBRLToReais(b.amount) − parseBRLToReais(a.amount)).slice(0, 3)` → **ordenação: maior valor restante primeiro, no máximo 3**.

**`buildPierreInstallmentWidgetItem(p)`** (w3:4181): `{ id: p.id, name: p.description, amount: formatReais(p.remainingAmount), installmentLabel: 'Parcela ' + p.paidInstallments + ' de ' + p.totalInstallments, progressPercentage: p.totalInstallments > 0 ? round(p.paidInstallments / p.totalInstallments × 100) : 0, logoUrl: p.companyLogo, bankName: p.accountName, bankLogoUrl: p.accountLogo, avatarFallbackIconKey: 'cardCredit' }`.

Portanto o “progresso 3/10” aparece no app como **“Parcela 3 de 10”** + **“30%”** (não há string `3/10` na UI; `3/10` é só o formato de entrada `installmentInfo`).

`mergePierreCardIntoInstallmentsChart` (w3:4513) e `buildInstallmentsBarMeta` (rótulos `JAN`/`JAN/27`, título `Comprometido em {Mês}`) servem à **tela** de parcelamentos (barras de 6 a 12 meses: `min(12, max(barsDoServidor, maxRemaining, 6))`), não ao widget.

### 2.4 Conexão e estados [O]
`InstallmentsWidgetConnected({ size='half', onPress, emptyState, animate=false })` (L2854810):
- `isError` → `<View testID="installments-error" className="rounded-3xl bg-surface" style={styles[size]}><WidgetErrorState compact={size !== 'full'}/></View>` (half 200, wide 170, full `minHeight: 320`; padding 16; centralizado).
- `data ?? DEFAULT` espalhado como props; `isLoading = isLoading || !isFetched` é passado, **mas `InstallmentsWidget` e `InstallmentsWidgetFull` não leem `isLoading`**: durante o carregamento aparece o estado vazio (defaults `count:'0'`) [O].
- `size === 'full'` → `InstallmentsWidgetFull`; senão `InstallmentsWidget`.

`isEmpty = count === '0' || !count`.

### 2.5 Ícone `InstallmentsIcon` / `InstallmentsIconLarge` [O]
Idênticos (testIDs `installments-icon` / `installments-full-icon`):
```tsx
<View testID="installments-icon" className="bg-muted" style={{width:48,height:48,borderRadius:24,alignItems:'center',justifyContent:'center'}}>
  <PierreIcon name="cardCredit" size={22} color="#71717a" />
  <View className="bg-muted" style={{position:'absolute',bottom:0,right:-2,width:20,height:20,borderRadius:10,alignItems:'center',justifyContent:'center'}}>
    <PierreIcon name="timeClock" size={14} color="#F97316" />
  </View>
</View>
```
Círculo #27272a de 48 px com cartão cinza #71717a (22) e selo circular #27272a de 20 px no canto inferior direito (2 px para fora) com relógio laranja #F97316 (14). O selo não tem borda.

### 2.6 `InstallmentsWidget` — half e wide (L2855251) [O]
```ts
pct = Math.min(100, Math.max(0, paidPercentage ?? 0));   // sem arredondar
value = isEmpty ? '—' : count;
subtitleText = isEmpty ? '' : (hidden ? 'R$ ••••' : subtitle);   // sim: a máscara troca até o “Em andamento”
valueNode = <PrivacyText allowFontScaling={false} className="text-xl font-extrabold text-foreground"
                         testID="installments-widget-value" numberOfLines={1} value={value}/>;
```
| | half (`Widget2x2`) | wide (`WidgetWide`) |
|---|---|---|
| props | `testID="installments-widget" label="Parcelamentos" value={value} valueSlot={valueNode} subtitle={subtitleText} visual onPress animate` | `testID="installments-widget" label="Parcelamentos" value={value} valueMinimumFontScale={1} subtitle={subtitleText} visual onPress animate` |
| visual (conteúdo) | `<InstallmentsIcon/>` | rodapé com barra (abaixo) |
| visual (vazio) | `<WidgetEmptyStateSlot emptyState compact/>` | `<WidgetEmptyStateSlot emptyState/>` (não compacto) |

O valor é a **quantidade** de parcelamentos (ex.: `4`), não um valor em R$; no half usa `valueSlot` (17,5/24,5 Geist-ExtraBold #fafafa; ver W0 §5).

Rodapé do wide:
```tsx
<View style={{ gap: 12 }}>
  <View className="bg-muted" style={{ height: 7, borderRadius: 4, overflow: 'hidden' }}>
    <View style={[{ height: '100%', borderRadius: 4, backgroundColor: '#C8FF00' }, { width: `${pct}%` }]} />
  </View>
  <HStack className="items-center justify-between">
    <Text allowFontScaling={false} className="text-base text-muted-foreground" numberOfLines={1}>{`${pct}% pago`}</Text>
    <Text allowFontScaling={false} className="text-base text-muted-foreground" numberOfLines={1}
          adjustsFontSizeToFit minimumFontScale={0.75}>{hidden ? 'R$ ••••' : `${paidAmount} de ${totalAmount}`}</Text>
  </HStack>
</View>
```
Cor fixa da barra: `#C8FF00` (não varia por percentual, diferente do limite). Textos: Geist-Regular 14/21 #a1a1aa. O percentual “pago” **não é mascarado**.

### 2.7 `InstallmentsWidgetFull` (L2856127) [O]
```tsx
<WidgetFull testID="installments-widget-full" title="Parcelamentos" onPress animate>
  {isEmpty
    ? <WidgetEmptyStateSlot emptyState={emptyState} message="Nenhum parcelamento ativo" />
    : items.length > 0
      ? <VStack className="gap-7">{items.slice(0, 3).map((it, i) =>
          <InstallmentRow key={`${it.id}-${i}`} item={it} hidden={hidden} />)}</VStack>
      : <HStack className="items-center gap-4">
          <InstallmentsIconLarge />
          <VStack className="flex-1 gap-1">
            <Text allowFontScaling={false} className="text-3xl font-extrabold text-foreground">{count}</Text>
            <PrivacyText allowFontScaling={false} className="text-base text-muted-foreground" value={hidden ? 'R$ ••••' : subtitle} />
          </VStack>
        </HStack>}
</WidgetFull>
```
- Lista: gap vertical 24,5 px (`gap-7`), até **3** linhas (dupla garantia: merge e `slice(0,3)`).
- Resumo (sem itens): ícone 48 + gap 14 (`gap-4`); contagem `text-3xl font-extrabold` = Geist-ExtraBold 26,25/31,5 #fafafa; subtítulo 14/21 #a1a1aa; gap 3,5 (`gap-1`).
- Título do card “Parcelamentos” = estilo do `WidgetFull` (W0 §7: `text-lg text-muted-foreground`).

`InstallmentRow({ item, hidden })`:
```tsx
<HStack className="items-center justify-between gap-4">                     // gap 14
  <HStack className="min-w-0 flex-1 items-center gap-3">                    // gap 10,5
    <InstallmentAvatar item={item} />                                       // MerchantAvatar 44
    <VStack className="min-w-0 flex-1">
      <Text allowFontScaling={false} className="text-lg font-bold text-foreground" numberOfLines={1}>{item.name}</Text>
      <Text allowFontScaling={false} className="text-base text-muted-foreground" numberOfLines={1}>{item.installmentLabel}</Text>
    </VStack>
  </HStack>
  <VStack className="items-end">
    <PrivacyText allowFontScaling={false} className="text-lg text-foreground" numberOfLines={1}
                 adjustsFontSizeToFit minimumFontScale={0.75} value={hidden ? 'R$ ••••' : item.amount} />
    <Text allowFontScaling={false} className="text-base text-muted-foreground" numberOfLines={1}>{`${item.progressPercentage}%`}</Text>
  </VStack>
</HStack>
```
| Texto | Classe | Fonte | Tamanho/linha | Cor |
|---|---|---|---|---|
| nome (ex.: “Magazine Luiza”) | `text-lg font-bold text-foreground` | Geist-Bold | 15,75 / 24,5 | #fafafa |
| “Parcela 3 de 10” | `text-base text-muted-foreground` | Geist-Regular | 14 / 21 | #a1a1aa |
| valor restante (“R$ 1.234,56”) | `text-lg text-foreground` | Geist-Regular | 15,75 / 24,5 (autoshrink 75 %) | #fafafa |
| “30%” | `text-base text-muted-foreground` | Geist-Regular | 14 / 21 | #a1a1aa |

Sem barra de progresso por linha (só o texto “NN%”).

`InstallmentAvatar({ item })` (w3:3150):
```tsx
<MerchantAvatar size={44}
  name={item.name} logoUrl={item.logoUrl ?? null}
  bankName={item.bankName ?? null} bankLogoUrl={item.bankLogoUrl ?? null}
  categoryFallback={item.avatarFallbackIconKey ? { name: 'Outros', color: '#A1A1AA', iconKey: item.avatarFallbackIconKey } : undefined}
  fallbackLabel={item.name.charAt(0).toUpperCase()} />
```
Lógica de logo/selo/fallback do `MerchantAvatar`: §6.

### 2.8 Strings pt-BR [O]
`Parcelamentos`, `Nenhuma parcela`, `Em andamento`, `Nenhum parcelamento ativo`, `% pago` (sufixo), ` de ` (em “R$ X de R$ Y”), `Parcela N de M`, `Parcelamento` (descrição padrão), `Última parcela em {Mês} de {AAAA}`, `Comprometido em {Mês}`, `—`, `R$ 0,00`, meses `Janeiro…Dezembro` e `JAN…DEZ`, `Outros`.

---

## 3. `patrimony` — “Patrimônio” (oculto)

> Está na lista de ids **ocultos** do registro (§0): o renderer nunca o mostra na v4.0.6. O código está completo e funcional, então é documentado para réplica.

### 3.1 Mapa de módulos [O]
| Módulo | Linha (out.js / w3) | Conteúdo |
|---|---|---|
| M9022 | — | barrel: `PatrimonyWidget`, `PatrimonyWidgetConnected` |
| M9023 | L2858204 / w3:5160–5925 | `PatrimonyIconCircle`, `PatrimonyIconStack`, `PatrimonyRow`, `PatrimonyWidget` + `StyleSheet` dos cards |
| M9024 | L2859198 / w3:6599 | `PatrimonyWidgetConnected` + `StyleSheet` de erro/vazio |
| M9025 | w3:6849 | `usePatrimonyData(size)` = repasse puro de `useVisionSystemWidgetData('patrimony', size)` (sem mescla com o cartão Pierre) |

Dependências de M9023: compiler-runtime, React, `react-native` (View, Pressable, StyleSheet), `PierreIcon` (M2465), `Box` (M2226), `HStack` (M2260), `Text` (M2262), `VStack` (M2258), `PrivacyText` (M4432), `PrivacyCurrencyText` (M8991), `useThemeColors` (M2979), privacidade (M4428), base de widgets (M8976: `WidgetGlassCard`, `WidgetEmptyStateSlot`, `WidgetLoadingState`).

### 3.2 Dados [O]
Payload (strings BRL): `{ totalAmount, checkingAmount, investmentsAmount, manualAdditionsAmount, assetCount: number }`. Cada string ausente/vazia cai em `'R$ 0,00'`; `assetCount ?? 0`.

Linhas fixas (sempre 3, nesta ordem):
| label | campo | ícone (`PierreIcon`) | cor do ícone |
|---|---|---|---|
| Conta corrente | `checkingAmount` | `buildingsBank` | `#A1A1AA` |
| Investimentos | `investmentsAmount` | `productChart` | `#8B5CF6` (violeta) |
| Adições manuais | `manualAdditionsAmount` | `walletMoney` | `#00A66C` (verde) |

Parse local do total (sem `parseBRLToCents`): `Number(s.replace(/[^\d,.-]/g,'').replace(/\./g,'').replace(',', '.'))`, 0 se não finito. `valueCents = Math.round(total × 100)`.

**Vazio**: `!isLoading && totalReais === 0 && assetCount === 0` **e** `emptyState` fornecido.

### 3.3 Conexão e estados [O]
`PatrimonyWidgetConnected({ size='half', onPress, emptyState, animate=false })`:
- `isError` → `<View testID="patrimony-error" className="rounded-3xl bg-surface" style={size==='full' ? {minHeight:320,…} : {height:200,…}}><WidgetErrorState compact={size==='half'}/></View>` (centralizado, padding 16).
- `isFetched && !data` → `<View testID="patrimony-empty" className="rounded-3xl bg-surface" style=…><WidgetEmptyStateSlot emptyState compact={size==='half'}/></View>`.
- Senão `<PatrimonyWidget data size onPress isLoading={isLoading || !isFetched} emptyState animate/>`.

`PatrimonyWidget` **não** usa `Widget2x2`/`WidgetFull`: monta o próprio card com `WidgetGlassCard` (W0) e `style = [card, halfCard | fullCard]`:
```ts
card:     { borderRadius: 24, padding: 16, overflow: 'hidden' },
halfCard: { height: 200 },
fullCard: { minHeight: 320 },
```
Se houver `onPress`, o card inteiro é envolvido em `<Pressable testID="patrimony-widget-pressable" onPress>` (sem feedback visual próprio).
- **Carregando**: `WidgetGlassCard` > `<View testID="patrimony-widget"><WidgetLoadingState variant="content"/></View>`.
- **Vazio**: `WidgetGlassCard` > `<WidgetEmptyStateSlot emptyState compact={size==='half'}/>`.

### 3.4 Ícones [O]
`PatrimonyIconCircle({ icon, color, size = 36 })`:
```tsx
<View className="items-center justify-center bg-muted" style={{ width: size, height: size, borderRadius: size / 2 }}>
  {typeof icon === 'string'
     ? <PierreIcon name={icon} size={size * 0.48} color={color} />
     : <icon size={size * 0.48} color={color} />}   // [I: o decompilador mostra typeof 'boolean'; o ramo usa o ícone como componente]
</View>
```
Fundo #27272a; ícone = 48 % do diâmetro (36 → 17,28; 40 → 19,2).

`PatrimonyIconStack({ items })` (só no half): `<HStack testID="patrimony-icons" className="items-center">` com cada item em `<View key={label} style={{ marginLeft: -12, zIndex: items.length - i }}><PatrimonyIconCircle icon color/></View>` → 3 círculos de 36 px sobrepostos 12 px, o primeiro por cima. O `-12` vale também para o primeiro (a pilha começa 12 px à esquerda do padding) [O].

### 3.5 Layout half (200 px) [O]
```tsx
<WidgetGlassCard animate style={[card, halfCard]}>
  <VStack className="flex-1 justify-between" testID="patrimony-widget">
    <HStack className="items-start justify-between">
      <PatrimonyIconStack items={rows} />
      <Box testID="patrimony-widget-chevron" className="p-0.5"><PierreIcon name="chevronRight" size={16} color="#71717a" /></Box>
    </HStack>
    <VStack>
      <Text className="text-base text-muted-foreground" numberOfLines={1} allowFontScaling={false}>Patrimônio</Text>
      <PrivacyCurrencyText valueCents={round(total*100)} hidden={hidden} fontSize={20} fontWeight="800"
        color={foreground /* #fafafa */} disableMinWidth compactOnOverflow testID="patrimony-widget-value" />
      <Text className="text-base text-muted-foreground" numberOfLines={1} allowFontScaling={false}>
        {`${assetCount} ativo${assetCount === 1 ? '' : 's'}`}</Text>
    </VStack>
  </VStack>
</WidgetGlassCard>
```
Chevron: `Box p-0.5` (padding 1,75) com ícone 16 px #71717a. Textos 14/21 Geist-Regular #a1a1aa; valor Geist 20 px peso 800 (via `PrivacyCurrencyText`, W0 §11.4). A contagem de ativos não é mascarada.

### 3.6 Layout full (min 320 px) [O]
```tsx
<WidgetGlassCard animate style={[card, fullCard]}>
  <VStack className="flex-1 gap-6" testID="patrimony-widget">                    // gap 21
    <HStack className="items-center justify-between">
      <Text allowFontScaling={false} className="text-lg text-muted-foreground" numberOfLines={1}>Patrimônio total</Text>
      <Box testID="patrimony-widget-chevron" className="p-0.5"><PierreIcon name="chevronRight" size={16} color="#71717a"/></Box>
    </HStack>
    <PrivacyText allowFontScaling={false} className="text-3xl font-extrabold text-foreground" testID="patrimony-widget-value"
      numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.75} value={hidden ? 'R$ ••••' : totalAmount} />
    <VStack className="flex-1 justify-between">{rows.map(r => <PatrimonyRow key={r.label} item={r} hidden={hidden}/>)}</VStack>
  </VStack>
</WidgetGlassCard>
```
`PatrimonyRow`:
```tsx
<HStack className="items-center justify-between gap-4">
  <HStack className="flex-1 min-w-0 items-center gap-3">
    <PatrimonyIconCircle icon={item.icon} color={item.iconColor} size={40} />
    <Text allowFontScaling={false} className="flex-1 min-w-0 text-lg font-semibold text-foreground"
          numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.7}>{item.label}</Text>
  </HStack>
  <PrivacyText allowFontScaling={false} className="text-base shrink min-w-0 text-foreground" numberOfLines={1}
               adjustsFontSizeToFit minimumFontScale={0.6} value={hidden ? 'R$ ••••' : item.amount} />
</HStack>
```
| Texto | Classe | Fonte | Tamanho/linha | Cor |
|---|---|---|---|---|
| “Patrimônio total” | `text-lg text-muted-foreground` | Geist-Regular | 15,75 / 24,5 | #a1a1aa |
| total | `text-3xl font-extrabold` | Geist-ExtraBold | 26,25 / 31,5 (autoshrink 75 %) | #fafafa |
| rótulo da linha | `text-lg font-semibold` | Geist-SemiBold | 15,75 / 24,5 (autoshrink 70 %) | #fafafa |
| valor da linha | `text-base` | Geist-Regular | 14 / 21 (autoshrink 60 %) | #fafafa |

### 3.7 Strings pt-BR [O]
`Patrimônio`, `Patrimônio total`, `Conta corrente`, `Investimentos`, `Adições manuais`, `N ativo` / `N ativos`, `R$ 0,00`.

---

## 4. `cards` — “Cartões” (aposentado do picker)

> Filtrado do picker (§0), mas renderiza se já estiver no layout salvo do usuário.

### 4.1 Mapa de módulos [O]
| Módulo | Linha (out.js / w3) | Conteúdo |
|---|---|---|
| M9026 | — | barrel: `CardsWidget`, `CardsWidgetConnected` |
| M9027 | L2860269 / w3:6966–8484 | `resolveCardColor`, `EmptyIcon`, `UsageBar`, `CardAvatar`, `FullCardRow`, `CardsWidget` + `StyleSheet` |
| M9028 | L2861088 / w3:8489 | `CardsWidgetConnected` + defaults + estilos de erro |
| M9029 | w3:8684 | `useCardsData(size)` |
| M9000 | L2849204 / L2850166 | `buildPierreCardWidgetItem`, `mergePierreCardIntoCardsWidget` |
| M3633 (reexporta M3634/M3697) | L1330882 / L1342579 | `BankLogo` e `resolveBankVisual` (W1 §5) |
| M8727 | L2752123 | `StableCircularSurface` (círculo de cor estável) |

### 4.2 Constantes e estilos [O]
```ts
const DEFAULT_AMOUNT = 'R$ 0,00';
const CARD_FALLBACK_COLORS = ['#A100FF', '#FF6B00', '#22C55E'];   // roxo, laranja, verde
const styles = StyleSheet.create({
  card:          { borderRadius: 24, padding: 16, overflow: 'hidden' },
  halfCard:      { height: 200 },
  fullCard:      { minHeight: 200 },
  fullEmptyCard: { minHeight: 180 },
  emptyIcon:     { width: 44, height: 44, borderRadius: 22 },
  usageTrack:    { width: '100%', height: 7, borderRadius: 4, backgroundColor: '#27272A', overflow: 'hidden' },
  usageFill:     { height: '100%', borderRadius: 4 },
});
// Connected (M9028): DEFAULT = { cards: [] }; erro: half {height:200,…}, full {minHeight:320,…} (centro, padding 16)
// Cartão Pierre (M9000): id = 'cw-' + 'pierre-one' = 'cw-pierre-one'; bandeira 'MASTERCARD'
```

`resolveCardColor(bankName, index)` (w3:6978):
```ts
resolveBankVisual(bankName)?.color ?? CARD_FALLBACK_COLORS[index % 3] ?? CARD_FALLBACK_COLORS[0]
```
A cor do banco vem da tabela de bancos (W1 §5). Fallback por **posição** do cartão na lista (0 roxo, 1 laranja, 2 verde, 3 roxo…), não por hash do nome.

### 4.3 Dados [O]
Payload (`useVisionSystemWidgetData('cards', size)`):
```ts
{ cards: Array<{ id, bankName, logoUrl, lastFourDigits, currentInvoiceAmount /*BRL*/,
                 dueDateLabel /*'Vence 15 de Março'*/, usedAmount /*BRL*/, totalLimit /*BRL*/, usedPercentage: number }> }
```
`useCardsData(size)` (w3:8684) segue o padrão de `useAvailableLimitData`: `data = isSettled ? mergePierreCardIntoCardsWidget(query.data, view) ?? undefined : undefined`; `isLoading = query.isLoading || !isSettled`; `isFetched = query.isFetched && isSettled`.

`mergePierreCardIntoCardsWidget(server, pierreView)` (L2850166): sem `pierreView` → `server ?? null`; senão `{ ...server, cards: [buildPierreCardWidgetItem(pierreView), ...server.cards.filter(c => c.id !== 'cw-pierre-one')] }` → **o cartão Pierre sempre vem primeiro**.

`buildPierreCardWidgetItem({ cardInfo })` (L2849204):
```ts
{ id: cardInfo.id, bankName: cardInfo.name, logoUrl: cardInfo.logoUrl,
  lastFourDigits: cardInfo.lastFourDigits ?? '••••',
  currentInvoiceAmount: cardInfo.invoiceAmountFormatted,
  dueDateLabel: cardInfo.invoiceAmount > 0
      ? (dueDate 'AAAA-MM-DD' válido ? `Vence ${Number(dd)} de ${MESES[mm-1]}` : 'Sem vencimento')   // ex.: 'Vence 5 de Março'
      : 'Sem fatura aberta',
  usedAmount: cardInfo.usedAmountFormatted, totalLimit: cardInfo.creditLimitFormatted, usedPercentage: cardInfo.usedPercentage }
```

### 4.4 Conexão e estados [O]
`CardsWidgetConnected({ size='half', onPress, animate=false })` (não recebe `emptyState`):
- `isError` → `<View testID="cards-error" className="rounded-3xl bg-surface" style={half|full}><WidgetErrorState compact={size==='half'}/></View>`.
- Senão `<CardsWidget data={data ?? {cards:[]}} size onPress isLoading={isLoading || !isFetched} animate/>`.

`CardsWidget` monta o próprio card com `WidgetGlassCard` (como `patrimony`), envolto em `<Pressable testID="cards-widget-pressable" onPress>` quando há `onPress`.
- **Carregando** (qualquer tamanho): `WidgetGlassCard style=[card, half ? halfCard : fullCard]` > `<View testID="cards-widget"><WidgetLoadingState variant="content"/></View>`.

### 4.5 Subcomponentes [O]
```tsx
// EmptyIcon
<View className="items-center justify-center bg-muted" style={{width:44,height:44,borderRadius:22}}>
  <PierreIcon name="cardCredit" size={20} color="#A1A1AA" />
</View>

// UsageBar({ percentage, color })
<View style={usageTrack}>
  <View style={[usageFill, { width: `${Math.min(100, Math.max(0, percentage))}%`, backgroundColor: color }]} />
</View>

// CardAvatar({ card, color })
<StableCircularSurface backgroundColor={color} size={44} testID={`cards-widget-avatar-${card.id}`}>
  <BankLogo bankName={card.bankName} imageUrl={card.logoUrl} size={44} showLoadingPlaceholder testID={`cards-widget-logo-${card.id}`} />
</StableCircularSurface>
```
`CardAvatar` = círculo de 44 px na cor do banco com o `BankLogo` de 44 px por cima (sem selo nem borda). A lógica de logo do banco (SVG/PNG local por nome, URL remota, iniciais) é a do `BankLogo` (W1 §5).

`StableCircularSurface({ backgroundColor, size, testID, children })` (L2752123) [O]: `View {width:size, height:size, flexShrink:0, center}` com uma camada absoluta `{left:0, top:0, width:size, height:size, borderRadius:size/2, backgroundColor}` (`pointerEvents="none"`, testID `{testID}-surface`) e o conteúdo num `View {width, height, borderRadius:size/2, center}` (testID `{testID}-content`). É só um círculo de cor por baixo do logo.

### 4.6 Layout half (200 px) [O]
Mostra **apenas o primeiro cartão** (`cards[0]`, em geral o Pierre):
```tsx
<WidgetGlassCard animate style={[card, halfCard]}>
  <VStack className="flex-1 justify-between" testID="cards-widget">
    <HStack className="items-start justify-between">
      {first
        ? <HStack className="min-w-0 flex-1 items-center gap-2">                        // gap 7
            <CardAvatar card={first} color={resolveCardColor(first.bankName, 0)} />
            <Text className="min-w-0 flex-1 text-base font-bold text-foreground" numberOfLines={1} allowFontScaling={false}>
              {`•••• ${first.lastFourDigits}`}</Text>
          </HStack>
        : <EmptyIcon />}
      <PierreIcon name="chevronRight" size={16} color="#F4F4F5" />
    </HStack>
    <VStack className="gap-1">                                                          // gap 3,5
      <Text className="text-base text-muted-foreground" numberOfLines={1} allowFontScaling={false}>Fatura Atual</Text>
      <PrivacyCurrencyText valueCents={parseBRLToCents(first?.currentInvoiceAmount ?? 'R$ 0,00')} hidden={hidden}
        fontSize={20} fontWeight="800" color={foreground} minimumFontScale={0.75} disableMinWidth compactOnOverflow />
      <Text className="text-base text-muted-foreground" numberOfLines={1} allowFontScaling={false}>
        {first?.dueDateLabel ?? 'Sem vencimento'}</Text>
    </VStack>
  </VStack>
</WidgetGlassCard>
```
“•••• 1234”: Geist-Bold 14/21 #fafafa. Chevron #F4F4F5 (mais claro que o #71717a do patrimony). Sem cartões no half: só o `EmptyIcon`, valor `R$ 0,00`, “Sem vencimento” (não há texto de vazio dedicado).

### 4.7 Layout full [O]
```tsx
<WidgetGlassCard animate style={[card, isEmpty ? fullEmptyCard : fullCard]}>   // min 180 vazio / 200 com cartões
  <VStack className="gap-4" testID="cards-widget">                               // gap 14
    <HStack className="items-center justify-between">
      <Text allowFontScaling={false} className="text-lg text-muted-foreground" numberOfLines={1}>Seus cartões</Text>
      <PierreIcon name="chevronRight" size={16} color="#F4F4F5" />
    </HStack>
    {isEmpty
      ? <VStack className="gap-4 pt-5">                                            // gap 14, paddingTop 17,5
          <EmptyIcon />
          <VStack className="gap-1">
            <Text allowFontScaling={false} className="text-lg text-muted-foreground" numberOfLines={1}>Nenhum cartão conectado</Text>
            <PrivacyText allowFontScaling={false} className="text-2xl font-extrabold text-foreground" numberOfLines={1} value="R$ 0,00" />
          </VStack>
        </VStack>
      : <VStack className="gap-5">{cards.slice(0, 2).map((c, i) =>              // gap 17,5; no máximo 2 cartões
          <FullCardRow key={c.id} card={c} index={i} hidden={hidden} />)}</VStack>}
  </VStack>
</WidgetGlassCard>
```
`FullCardRow({ card, index, hidden })`:
```tsx
<VStack className="gap-2">                                                       // gap 7
  <HStack className="items-center justify-between gap-4">
    <HStack className="min-w-0 flex-1 items-center gap-3">
      <CardAvatar card={card} color={resolveCardColor(card.bankName, index)} />
      <VStack className="min-w-0 flex-1">
        <Text allowFontScaling={false} className="text-lg font-bold text-foreground" numberOfLines={1}>{`•••• ${card.lastFourDigits}`}</Text>
        <Text allowFontScaling={false} className="text-base text-muted-foreground" numberOfLines={1}>{card.dueDateLabel}</Text>
      </VStack>
    </HStack>
    <PrivacyText allowFontScaling={false} className="text-lg font-bold text-foreground" numberOfLines={1}
                 adjustsFontSizeToFit minimumFontScale={0.75} value={hidden ? 'R$ ••••' : card.currentInvoiceAmount} />
  </HStack>
  <UsageBar percentage={card.usedPercentage} color={resolveCardColor(card.bankName, index)} />
  <HStack className="items-center justify-between">
    <Text allowFontScaling={false} className="text-base text-muted-foreground" numberOfLines={1}>{`Usado: ${hidden ? 'R$ ••••' : card.usedAmount}`}</Text>
    <Text allowFontScaling={false} className="text-base text-muted-foreground" numberOfLines={1}>{`Total: ${hidden ? 'R$ ••••' : card.totalLimit}`}</Text>
  </HStack>
</VStack>
```
A barra de uso usa a **cor do banco** (não os thresholds do `available-limit`).

| Texto | Classe | Fonte | Tamanho/linha | Cor |
|---|---|---|---|---|
| “Seus cartões” | `text-lg text-muted-foreground` | Geist-Regular | 15,75 / 24,5 | #a1a1aa |
| “•••• 1234” (full) | `text-lg font-bold` | Geist-Bold | 15,75 / 24,5 | #fafafa |
| vencimento | `text-base text-muted-foreground` | Geist-Regular | 14 / 21 | #a1a1aa |
| valor da fatura | `text-lg font-bold` | Geist-Bold | 15,75 / 24,5 (autoshrink 75 %) | #fafafa |
| “Usado: …” / “Total: …” | `text-base text-muted-foreground` | Geist-Regular | 14 / 21 | #a1a1aa |
| “Nenhum cartão conectado” | `text-lg text-muted-foreground` | Geist-Regular | 15,75 / 24,5 | #a1a1aa |
| “R$ 0,00” (vazio) | `text-2xl font-extrabold` | Geist-ExtraBold | 21 / 28 | #fafafa |

### 4.8 Strings pt-BR [O]
`Cartões` (registro), `Seus cartões`, `Fatura Atual`, `Nenhum cartão conectado`, `Sem vencimento`, `Sem fatura aberta`, `Vence {d} de {Mês}`, `•••• ` (prefixo), `••••` (sem dígitos), `Usado: `, `Total: `, `R$ 0,00`, `Próximo vencimento` (constante do módulo M9000, usada fora do widget [I]).

---

## 5. `recent-transactions` — “Transações recentes”

### 5.1 Mapa de módulos [O]
| Módulo | Linha (out.js / w3) | Conteúdo |
|---|---|---|
| M9030 | L2861418 / w3:8819 | `VisionRecentTransactionsWidgetConnected` + `StyleSheet {half}` |
| M9031 | L2861750 / w3:9096–9732 | `TransactionsIcon`, `VisionRecentTransactionsWidget` (memo) |
| M9032 | — | reexporta `VisionTransactionRow` de M8529 |
| M9033 | w3:9764 | `useRecentTransactionsData(size)` = repasse de `useVisionSystemWidgetData('recent-transactions', size)` (sem mescla Pierre) |
| M8529 | L2720600–L2722180 | `VisionTransactionRow` (memo com `areTransactionRowsEqual`), `VISION_TRANSACTION_ROW_HEIGHT = 79` + `StyleSheet` |
| M8530 | L2722213–L2722480 | `createBrazilCalendarDate`, `getTodayBrazilCalendarDate`, `formatBrazilDayMonthShort` |
| M8531 | L2722496–L2723330 | `MerchantAvatar`, `resolveMerchantAvatarCategoryFallback` |
| M8532/M8533 | L2723407 | `CategoryAvatar` (W2 §4.7) |
| M8535 | L2724600–L2725150 | `matchMerchantLogo`, `matchMerchantLogoLenient` |
| M6769 | L2214370 | `getCompanyLogoUrl`, `getCompanyBrandColor` |

### 5.2 Dados [O]
Payload: `{ transactions: TransactionItem[], totalCount?: number }`. `count = totalCount ?? transactions.length`.

`TransactionItem` (campos lidos pela linha e por `areTransactionRowsEqual`):
```ts
{ id, name, amount: string /* 'R$ 12,90', com ou sem '+'/'-' */, type: 'CREDIT' | 'DEBIT' | string,
  date: string /* rótulo pronto do servidor */, dateValue?: 'AAAA-MM-DD',
  category?: string, iconName?: string /* chave PierreIcon */, iconColor?: string, emoji?: string,
  customLogoUrl?: string, companyLogoUrl?: string, bankName?: string, bankLogoUrl?: string,
  isBillPayment?: boolean, isHidden?: boolean, installmentNumber?: number, totalInstallments?: number,
  currencySymbol?: string }
```
Limites [O w3:9220]: `visible = transactions.slice(0, size === 'full' ? 5 : 3)`; o **wide** usa `transactions.slice(0, 1)` (1 linha). Sem ordenação local (ordem do servidor). Chave de cada linha: `` `${name}-${date}-${index}` ``.

### 5.3 Conexão e estados [O]
`VisionRecentTransactionsWidgetConnected({ size, onPress, emptyState, animate=false })` (sem default de `size`):
- `isError`: `size === 'full'` → `<WidgetFull testID="recent-transactions-error" title="" showChevron={false} animate><WidgetErrorState/></WidgetFull>`; senão `<View testID="recent-transactions-error" className="rounded-3xl bg-surface" style={{height:200, alignItems:'center', justifyContent:'center', padding:16}}><WidgetErrorState compact/></View>` (**também no wide**, com 200 px em vez de 170 [O]).
- `isFetched && !data`: igual, com `testID="recent-transactions-empty"` e `WidgetEmptyStateSlot emptyState` (compacto fora do full).
- Senão `<VisionRecentTransactionsWidget data size onPress isLoading={isLoading || !isFetched} emptyState animate/>`.

`isEmpty = transactions.length === 0`.

### 5.4 half (`Widget2x2`, 200 px) [O]
| Estado | props |
|---|---|
| carregando | `testID="vision-recent-transactions-widget" visual={<WidgetLoadingState variant="visual"/>} label="Transações" value="" subtitle="" animate` (sem `onPress`) |
| vazio | `visual={<WidgetEmptyStateSlot emptyState compact/>} label="Transações" value="—" subtitle="" onPress animate` |
| conteúdo | `visual={<TransactionsIcon/>} label="Transações" value={String(count)} subtitle={transactions[0].date} onPress animate` |

`TransactionsIcon`:
```tsx
<View testID="vision-recent-transactions-half-icon" className="bg-muted"
      style={{ width: 44, height: 44, borderRadius: 22, alignItems: 'center', justifyContent: 'center' }}>
  <PierreIcon name="contentTextList" size={20} color="#71717a" />
</View>
```
O half **não** mostra linhas: só o ícone, a contagem (ex.: “37”) e a data da transação mais recente como subtítulo. Tipografia = `Widget2x2` (W0 §5).

### 5.5 wide (`WidgetWide`, 170 px) [O]
- Carregando: `label="Transações recentes" value="" visual={<WidgetLoadingState variant="content"/>} onPress animate`.
- Normal: `label="Transações recentes" value={isEmpty ? '—' : String(count)} subtitle={isEmpty ? '' : 'últimas movimentações'} visual={…} onPress animate`, com
```tsx
visual = isEmpty
  ? <WidgetEmptyStateSlot emptyState={emptyState} />
  : <VStack className="gap-3 pt-3">{transactions.slice(0, 1).map((t, i) =>      // paddingTop 10,5
      <VisionTransactionRow key={`${t.name}-${t.date}-${i}`} item={t}
        amount={hidden ? 'R$ ••••' : t.amount} clickable={false} avatarSize={45} />)}</VStack>
```

### 5.6 full (`WidgetFull`) [O]
```tsx
<WidgetFull testID="vision-recent-transactions-widget" title="Transações recentes" onPress animate>
  {isLoading ? <WidgetLoadingState variant="content" />
   : isEmpty ? <WidgetEmptyStateSlot emptyState={emptyState} message="Sem transações recentes" />
   : <VStack className="gap-0">{transactions.slice(0, 5).map((t, i) =>
        <VisionTransactionRow key={`${t.name}-${t.date}-${i}`} item={t}
          amount={hidden ? 'R$ ••••' : t.amount} clickable={false} avatarSize={45} />)}</VStack>}
</WidgetFull>
```
Altura do conteúdo com 5 linhas ≥ 5 × 79 = 395 px (cada linha tem `minHeight: 79` e `py-3`). As linhas não são clicáveis no widget (`clickable={false}`); o toque vai para o card inteiro (rota de §0).

### 5.7 `VisionTransactionRow` (L2720681) [O]
Props e defaults: `item, testID, amount = item.amount, onPress, clickable = true, avatarSize = 55, avatarBadgeSize = 30, avatarIconSize = 22, avatarBackgroundColor, showSelectionControl = false, isSelected = false, hideBankName = false, hideBankBadge = false, dateLabel, statusLabel ({text,color}), titleFontWeight = 'bold', titleColor, showAmountPill = false, relationshipLabel, movementBadge`. O widget usa só `item`, `amount`, `clickable=false`, `avatarSize=45`.

**Lógica**:
```ts
const isLight = useThemeStore(s => s.mode === 'light');      // sempre false (app só dark)
const isBillCredit = !!item.isBillPayment && item.type === 'CREDIT';
const clean = amount.trim().replace(/^[+-]\s*/, '');
const display = isBillCredit ? clean
              : item.type === 'CREDIT' ? `+${clean}`
              : item.type === 'DEBIT'  ? `-${clean}` : amount.trim();
const masked = item.currencySymbol ? `${item.currencySymbol} ••••` : 'R$ ••••';
const amountColor = isBillCredit ? '#FAFAFA'
                  : item.type === 'CREDIT' ? '#95FF48'                 // verde
                  : item.type === 'DEBIT'  ? '#FF5C7A'                 // rosa/vermelho
                  : '#FAFAFA';                                          // (#18181B no light)
const badge = movementBadge ?? (item.isBillPayment ? 'Pagamento' : undefined);
const logoUrl = item.customLogoUrl ?? item.companyLogoUrl ?? null;
const installment = item.installmentNumber != null && item.totalInstallments != null
  ? `${item.installmentNumber}/${item.totalInstallments}` : null;        // ← o “3/10”
const isFuture = item.dateValue != null && createBrazilCalendarDate(item.dateValue) > getTodayBrazilCalendarDate();
const title = item.name.length > 25 ? item.name.slice(0, 25) + '…' : item.name;
const subtitleTail = statusLabel ? statusLabel.text
                   : (item.category.length > 25 ? item.category.slice(0, 25) + '…' : item.category);
```
`createBrazilCalendarDate('AAAA-MM-DD')` = `new Date(Date.UTC(a, m−1, d, 12, 0, 0))`; “hoje” vem de `Intl.DateTimeFormat('en-CA', {timeZone:'America/Sao_Paulo'})` (fuso de São Paulo, meio-dia UTC para evitar troca de dia).

**Árvore**:
```tsx
<HStack className="items-center gap-3 py-3"                                   // gap 10,5; py 10,5
        style={[{ minHeight: 79 }, item.isHidden && { opacity: 0.45 }]} testID={onPress == null ? testID : undefined}>
  <MerchantAvatar name={item.name} logoUrl={logoUrl} bankName={item.bankName}
     bankLogoUrl={hideBankBadge ? null : item.bankLogoUrl} size={avatarSize} badgeSize={avatarBadgeSize}
     backgroundColor={avatarBackgroundColor}
     categoryFallback={{ name: item.category, color: item.iconColor, iconSize: avatarIconSize,
                         emoji: item.emoji ?? null, iconKey: item.iconName }} />
  <VStack className="flex-1">
    <Text allowFontScaling={false} numberOfLines={1}
          className={titleFontWeight === 'bold' ? 'font-bold text-foreground' : 'text-foreground'}
          style={[{ fontSize: 16, lineHeight: 20 }, titleColor && { color: titleColor }]}>{title}</Text>
    <Text allowFontScaling={false} numberOfLines={1} className="text-muted-foreground"
          style={[{ fontSize: 12, lineHeight: 16 }, statusLabel && { color: statusLabel.color }]}>
      {dateLabel ?? item.date}{' •'}{' '}{subtitleTail}</Text>
    {(badge || relationshipLabel) && (
      <HStack className="items-center gap-2" style={{ marginTop: 5 }}>
        {badge && <View style={[{ borderRadius: 999, paddingHorizontal: 9, paddingVertical: 4 }, badgeBg(badge)]}>
          <Text allowFontScaling={false} style={[{ fontSize: 12, lineHeight: 14, fontFamily: 'Geist-SemiBold' }, badgeFg(badge)]}>{badge}</Text></View>}
        {relationshipLabel && <Text allowFontScaling={false} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.85}
          style={{ color: '#95FF48', fontSize: 12, lineHeight: 16, fontFamily: 'Geist-Medium' }}>{relationshipLabel}</Text>}
      </HStack>)}
    {!hideBankName && item.bankName != null &&
      <Text allowFontScaling={false} className="text-muted-foreground" style={{ fontSize: 12, lineHeight: 16 }}>{item.bankName}</Text>}
  </VStack>
  <HStack className="items-center gap-3">
    <VStack className="items-end justify-center">
      <PrivacyText allowFontScaling={false} className="text-base font-bold" style={{ color: amountColor }}
                   value={hidden ? masked : display} />
      {showAmountPill && isFuture &&
        <View style={[{ width: 59, height: 22, borderRadius: 9999, marginTop: 8, paddingHorizontal: 8, paddingVertical: 2,
                        alignItems: 'center', justifyContent: 'center' }, { backgroundColor: '#27272A' }]}>
          <Text allowFontScaling={false} className="text-foreground"
                style={{ fontSize: 14, lineHeight: 16, fontFamily: 'Geist-Medium', includeFontPadding: false }}>Futura</Text></View>}
      {installment && <Text allowFontScaling={false} className="text-sm text-muted-foreground" numberOfLines={1}
                            style={{ lineHeight: 18, marginTop: 1 }}>{installment}</Text>}
    </VStack>
    {showSelectionControl && <View style={[{ width: 20, height: 20, borderRadius: 6, borderWidth: 1.5, borderColor: '#52525B', center },
                                           isSelected && { borderColor: '#FAFAFA', backgroundColor: '#FAFAFA' }]}>
                               {isSelected && <PierreIcon name="feedbackCheck" size={12} />}</View>}
  </HStack>
</HStack>
// se onPress != null && clickable: envolve em <Pressable onPress testID accessibilityRole="button" style={{ borderRadius: 18 }}>
```
Selos de movimento [O estilos L2722170]:
| `badge` | fundo | texto |
|---|---|---|
| `Parcela` | `rgba(59, 130, 246, 0.18)` | `#60A5FA` |
| `Assinatura` | `rgba(168, 85, 247, 0.18)` | `#C084FC` |
| `Estorno` | `rgba(34, 197, 94, 0.18)` | `#4ADE80` |
| qualquer outro (ex.: `Pagamento`) | `rgba(20, 184, 166, 0.18)` | `#2DD4BF` |

No widget só aparece `Pagamento` (derivado de `isBillPayment`); `Parcela`/`Assinatura`/`Estorno` vêm de props de outras telas.

| Texto | Estilo | Fonte | Tamanho/linha | Cor |
|---|---|---|---|---|
| nome | `font-bold` + style | Geist-Bold | **16 / 20** (style vence a classe) | #fafafa |
| “12 out • Restaurantes” | style | Geist-Regular | 12 / 16 | #a1a1aa |
| banco (“Nubank”) | style | Geist-Regular | 12 / 16 | #a1a1aa |
| valor | `text-base font-bold` | Geist-Bold | 14 / 21 | crédito #95FF48, débito #FF5C7A, pagamento de fatura #FAFAFA |
| “3/10” | `text-sm text-muted-foreground` + style | Geist-Regular | 16 / 18 (`text-sm` = 16 neste app) | #a1a1aa |
| selo | style | Geist-SemiBold | 12 / 14 | tabela acima |

### 5.8 Strings pt-BR [O]
`Transações recentes`, `Transações`, `últimas movimentações`, `Sem transações recentes`, `—`, `Pagamento`, `Parcela`, `Assinatura`, `Estorno`, `Futura`, ` •` (separador), `…` (corte em 25 caracteres), `N/M` (parcela).

---

## 6. Lógica de logo/avatar das linhas (`MerchantAvatar`, L2722595) [O]

Usado por `VisionTransactionRow` (transações), `InstallmentAvatar` (parcelamentos) e outras listas. **Não** é usado em `cards` (que usa `CardAvatar` = `StableCircularSurface` + `BankLogo`).

### 6.1 Props e defaults
`name, logoUrl, bankName, bankLogoUrl, testID, size = 50, badgeSize = 30, backgroundColor, useBankAsPrimary = false, fallbackLabel, avatarIconName, avatarIconColor = '#FAFAFA', categoryFallback`.
Nos widgets: transações `size 45, badgeSize 30, iconSize 22`; parcelamentos `size 44` (badge 30 por default).

### 6.2 Resolução (em ordem)
1. **Categoria de fallback** (`resolveMerchantAvatarCategoryFallback`): se `bankName === 'Cartão Pierre'` e `translatePierreCardBucketToVisionName(categoryFallback.name)` acha um nome Vision, troca `{name, color, iconKey}` pelos visuais da categoria Vision (`resolvePierreCardVisualsForVisionName` → `VISION_CATEGORY_DISPLAY_MAP`) e zera `emoji`.
2. **Logo local por palavra-chave**: `match = matchMerchantLogo(name)` (M8535); `localSource = match?.logoSource ?? null` (asset PNG empacotado, `src_assets_merchantlogos_*` = `logos/lojas-e-apps/`).
3. **Logo remota**: `remote = logoUrl ?? getCompanyLogoUrl(name)`; se começa com `/`, prefixa `getBaseUrl()`.
4. `imageSource = localSource || (remote ? { uri: remote } : null)` → **a logo local vence a URL do servidor** (`customLogoUrl`/`companyLogoUrl`).
5. Estado `failedSource` (useState); `imageFailed = failedSource != null && failedSource === imageSource`. `onError` da imagem grava `failedSource = imageSource` (não tenta a próxima fonte; cai direto no fallback).
6. Conteúdo do círculo, em ordem:
   - `useBankAsPrimary && bankName` → `<BankLogo bankName imageUrl={bankLogoUrl} size={size}/>` (W1 §5);
   - `imageSource && !imageFailed` → `<Image source={imageSource} style={{width:size, height:size, borderRadius:size/2}} contentFit="cover" cachePolicy="memory-disk" transition={0} recyclingKey={match?.logoUrl ?? remote ?? name} onError testID="{testID}-image"/>` (expo-image, M3545);
   - `categoryFallback` → `<CategoryAvatar onlyIcon size={cf.iconSize ?? 22} iconSize={cf.iconSize ?? 22} category={{name, color: color ?? undefined, emoji: emoji ?? null, iconKey: iconKey ?? null}}/>`;
   - `avatarIconName` → `<PierreIcon name={avatarIconName} size={20} color={avatarIconColor}/>`;
   - senão inicial: `<Text className="text-base font-bold text-muted-foreground">{(fallbackLabel ?? name.charAt(0) ?? '?').toUpperCase()}</Text>` (Geist-Bold 14/21 #a1a1aa).
7. **Selo do banco**: se `!useBankAsPrimary && bankLogoUrl != null` (exige **URL**, não basta `bankName`):
```tsx
<Box collapsable={false} className="rounded-full items-center justify-center overflow-hidden"
     style={{ position: 'absolute', right: -5, bottom: -5, width: badgeSize, height: badgeSize }}>
  <BankLogo bankName={bankName ?? ''} imageUrl={bankLogoUrl} size={22} />
</Box>
```
   (`right/bottom = 4294967291` no bytecode = −5.) Caixa transparente de 30 × 30 sem borda nem fundo; o `BankLogo` de 22 px fica centrado nela, ou seja, o selo visível é um círculo de **22 px** cujo canto ultrapassa a borda do avatar em **1 px** à direita e embaixo (−5 + (30−22)/2 = −1) [O valores; I geometria]. Como o `BankLogo` resolve primeiro a logo local pelo `bankName` (raster/SVG), o `bankLogoUrl` só é baixado quando o banco não está no registro.

### 6.3 Moldura
```tsx
<Box collapsable={false} testID={testID} style={{ width: size, height: size, borderRadius: size / 2,
     backgroundColor: backgroundColor ?? '#27272A',            // '#E4E4E7' no light
     borderWidth: 0, borderColor: 'transparent',               // light: StyleSheet.hairlineWidth, '#D4D4D8'
     flexShrink: 0, alignItems: 'center', justifyContent: 'center' }}>
  <Box collapsable={false} testID={`${testID}-clip`}
       style={{ width: size, height: size, borderRadius: size / 2, overflow: 'hidden', alignItems: 'center', justifyContent: 'center' }}>
    {conteúdo}
  </Box>
  {selo}
</Box>
```
| | transações (45) | parcelamentos (44) |
|---|---|---|
| diâmetro / raio | 45 / 22,5 | 44 / 22 |
| fundo | #27272A | #27272A |
| borda (dark) | nenhuma | nenhuma |
| selo (caixa / logo / offset) | 30 / 22 / −5 | 30 / 22 / −5 |
| ícone de categoria | 22 | 22 |

### 6.4 `matchMerchantLogo(description)` (M8535, L2724642)
- Normalização: `normDesc(s) = s.toLowerCase().normalize('NFD').replace(/[̀-ͯ]/g, '')`.
- Padrões estritos (construídos uma vez): `Object.entries(MERCHANT_LOGO_MAP)` → descarta chaves normalizadas com < 2 caracteres → `pattern = new RegExp('(?<![a-z0-9])' + escape(chave) + '(?![a-z0-9])')` → **ordenados pelo nome do merchant mais longo primeiro** (ex.: `99food` antes de `99`). O primeiro que casa com `normDesc(desc)` vence.
- Se nenhum casar: **aliases** (10 entradas: `ifood`←`ifd`; `amazon`; `mercado pago`←`mercadopago`; `sams club`←`sams club`/`sam's club`/`samsclub`; `tokio marine`←`tokio marine*auto`/`tokio marine`; `unimed`; `boca do forno`; `gennaro`; `coris brasil turismo`←`coris brasil`/`coris turismo`; `cia do boi`), testando `desc.toLowerCase().includes(term)` (termos em minúsculas; aqui **sem** remover acentos).
- Resultado `{ merchant, logoUrl: 'merchant-logos/<arquivo>.png', logoSource: merchantLogoAssetsByFileStem[stem] ?? null }`, guardado num cache `Map` de até **512** descrições (FIFO: remove a chave mais antiga).
- `matchMerchantLogoLenient` (não usado pelo avatar): se o estrito falhar, aceita chaves com `*` de ≥ 4 caracteres por `includes`.
- É exatamente o que `logos/mapeamento.json` descreve (`merchantPorPalavraChave`, `aliases`): as 168 chaves e os 10 aliases conferem com o algoritmo acima. Ao reimplementar, lembre da **ordenação por comprimento** (o JSON não está ordenado) e do corte de 2 caracteres.

### 6.5 `getCompanyLogoUrl(name)` (M6769, L2214370) — logo remota quando não há `logoUrl`
- `key = name.toLowerCase().trim()`; se `key` existe na tabela de domínios (`netflix → netflix.com`, `spotify → spotify.com`, `ifood → ifood.com.br`, `uber → uber.com`, `99 → 99app.com`, `nubank → nubank.com.br`, `itau/itaú → itau.com.br`, …) → `buildUrl(domínio)`.
- Senão percorre as chaves **da mais longa para a mais curta**: aceita se `key.includes(chave)` e, para chaves com **≤ 3** caracteres, também se casar `(?:^|[\s.,;:!?*/\-_])chave(?:$|[\s.,;:!?*/\-_])` (case-insensitive, regex em cache).
- `buildUrl(d) = getBaseUrl() + '/api/logo?domain=' + d` → proxy de logos do backend do Pierre [I: serviço tipo favicon/brandfetch no servidor].
- Mesma tabela alimenta `getCompanyBrandColor` (`netflix #E50914`, `spotify #1DB954`, …), já em `logos/mapeamento.json` → `assinaturas`.

### 6.6 Ícone/cor de categoria (`CategoryAvatar` com `onlyIcon`, L2723407)
Complementa W2 §4.7 (lá só o modo `filled`):
- `match = resolveVisionCategoryDisplayMatch(category)` (procura `label`, `name`, `title`, `id` normalizados em `VISION_CATEGORY_DISPLAY_MAP`, W2 §4.8).
- `iconKey = match?.iconKey ?? (category.iconKey ∈ pierreIcons ? category.iconKey : 'walletMoney')` → **o mapa local vence o `iconName` do servidor**.
- `color = match?.color ?? category.color ?? '#27272A'`; `hasColor = match?.color != null || category.color != null`.
- `emoji`: `null` se algum de `id/title/name/label` é nome de categoria nativa (`isNativeCategoryName`); senão `category.emoji ?? null`.
- `size` default 40; `iconSize = iconSize ?? max(16, round(size × 0,45))`.
- Cor do ícone: `iconColor` prop ?? `#A1A1AA` (dark; `#71717A` light); com `filled` → `#FFFFFF`; sem `filled` e `hasColor` → **cor da categoria**.
- Fundo: `onlyIcon` → `transparent`; `filled` → cor da categoria; senão superfície `#18181B`.
- Emoji: `Text {fontSize: round(size × 0,46), lineHeight: round(size × 0,5)}`.

Na linha de transação, portanto: círculo #27272A de 45 px com o ícone de 22 px **colorido pela categoria** (ex.: Alimentação → `cartShopping` #F97316); emoji do usuário se a categoria for personalizada; ícone `walletMoney` cinza #A1A1AA se não houver categoria reconhecida nem cor. Nos parcelamentos o fallback é fixo: `cardCredit` #A1A1AA (`Outros` não está no mapa).

### 6.7 Snippet TSX de referência (avatar com selo)
```tsx
type CategoryFallback = { name?: string; color?: string; iconKey?: string; emoji?: string | null; iconSize?: number };

export function MerchantAvatar({ name, logoUrl, bankName, bankLogoUrl, size = 50, badgeSize = 30,
  backgroundColor = '#27272A', categoryFallback }: {
  name: string; logoUrl?: string | null; bankName?: string | null; bankLogoUrl?: string | null;
  size?: number; badgeSize?: number; backgroundColor?: string; categoryFallback?: CategoryFallback }) {
  const local = useMemo(() => matchMerchantLogo(name)?.logoSource ?? null, [name]);   // require('…png')
  const remote = useMemo(() => {
    const u = logoUrl ?? getCompanyLogoUrl(name);
    return u && u.startsWith('/') ? API_BASE + u : u;
  }, [logoUrl, name]);
  const source = local || (remote ? { uri: remote } : null);
  const [failed, setFailed] = useState<unknown>(null);
  const showImage = source && failed !== source;
  const r = size / 2;

  let content: React.ReactNode;
  if (showImage) {
    content = <Image source={source} style={{ width: size, height: size, borderRadius: r }}
                     contentFit="cover" cachePolicy="memory-disk" transition={0} onError={() => setFailed(source)} />;
  } else if (categoryFallback) {
    const vis = resolveCategoryVisual(categoryFallback);          // §6.6
    const s = categoryFallback.iconSize ?? 22;
    content = vis.emoji
      ? <Text style={{ fontSize: Math.round(s * 0.46), lineHeight: Math.round(s * 0.5) }}>{vis.emoji}</Text>
      : <PierreIcon name={vis.iconKey} size={s} color={vis.hasColor ? vis.color : '#A1A1AA'} />;
  } else {
    content = <Text style={{ fontFamily: 'Geist-Bold', fontSize: 14, lineHeight: 21, color: '#a1a1aa' }}>
                {(name.charAt(0) || '?').toUpperCase()}</Text>;
  }

  return (
    <View style={{ width: size, height: size, borderRadius: r, backgroundColor, flexShrink: 0,
                   alignItems: 'center', justifyContent: 'center' }}>
      <View style={{ width: size, height: size, borderRadius: r, overflow: 'hidden', alignItems: 'center', justifyContent: 'center' }}>
        {content}
      </View>
      {bankLogoUrl != null && (
        <View style={{ position: 'absolute', right: -5, bottom: -5, width: badgeSize, height: badgeSize,
                       borderRadius: 9999, overflow: 'hidden', alignItems: 'center', justifyContent: 'center' }}>
          <BankLogo bankName={bankName ?? ''} imageUrl={bankLogoUrl} size={22} />
        </View>
      )}
    </View>
  );
}
```

---

## 7. Dúvidas abertas
1. **`installments` sem esqueleto de carregamento**: o Connected passa `isLoading`, mas os componentes não o leem; durante o carregamento aparece o estado vazio. Pode ser intencional (cache do react-query quase sempre presente) ou bug [O código; I intenção].
2. **Payload do servidor** de `installments.items`, `patrimony`, `cards` e `recent-transactions`: estruturas inferidas pelos campos lidos. Não sei se `items` vem para `half`/`wide` (o merge aceita se vier).
3. **`PIERRE_CARD_INVOICE_CARD_ID`**: deduzido como `'cw-pierre-one'` pela concatenação vista em L2850869–L2850875; o registrador do export não foi rastreado até o fim.
4. **`PatrimonyIconCircle`**: o decompilador mostra `typeof icon === 'boolean'`; interpretei como teste de string (nome de `PierreIcon`) versus componente. Na prática os 3 ícones são strings.
5. **`recent-transactions` wide em erro/vazio** usa a caixa de 200 px (`styles.half`), não 170 px [O]; se o grid fixa a célula wide em 170, a caixa extrapola ou é cortada [I].
6. **Selo do banco**: a geometria visível (círculo de 22 px a −1 px da borda) depende do `BankLogo` ocupar só os 22 px centrados na caixa de 30; não verifiquei se o `BankLogo` com URL remota SVG respeita o mesmo tamanho.
7. **`date` das transações**: é rótulo pronto do servidor (o widget não formata). O módulo de datas tem `formatBrazilDayMonthShort` (`Intl pt-BR {day:'numeric', month:'short'}` sem ponto, ex.: “12 out”), provavelmente o mesmo formato do servidor [I].
8. **`/api/logo?domain=`**: serviço do backend do Pierre; o provedor real (Clearbit, Brandfetch, favicon) não aparece no APK.
9. **`cards` full** mostra no máximo 2 cartões e não tem “ver mais”; `fullCard.minHeight` (200) é menor que o `minHeight` do erro (320) [O].
10. **Patrimônio oculto**: está pronto, mas a lista de ocultos o esconde; não há sinal de feature flag remota para reativá-lo [I].
