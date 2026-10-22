# W4 — Widget `subscriptions` ("Assinaturas") e tela de detalhe (Pierre v4.0.6)

Análise estática do `out.js` (Hermes v98 decompilado). **[O]** = observado no código (linha `Lnnnn` do `out.js`); **[I]** = inferido.
Referências que **não** são repetidas aqui:
- `widgets/00-base.md` — cadeia de renderização, `WidgetGlassCard`, `WidgetLoadingState`, `PrivacyText`/`PrivacyCurrencyText`, "Ocultar dados" (`useHideValues`, `HIDDEN_VALUE`), tokens dark (§13) e escala NativeWind rem = 14 (§14).
- `widgets/01-carteira-saldo-fluxo.md` — §1 `useVisionSystemWidgetData` (endpoint, queryKey, cache); §5 `BankLogo`.
- `widgets/03-limite-parcelas-cartoes-transacoes.md` — §5 `VisionTransactionRow`; §6 `MerchantAvatar` (logo local → remota → fallback, selo do banco), `matchMerchantLogo`, `getCompanyLogoUrl`.

Densidade do screenshot (1080 × 2340): **2,8125 px/dp** (384 dp de largura). Conferido pelo avatar das linhas: 135 px / 2,8125 = **48 dp** = `size 48` do código, e pelo botão voltar: 123 px → **44 dp**. Todas as medidas de screenshot abaixo estão convertidas para dp.

---

## 0. Mapa de módulos e identidade [O]

| Módulo | Linhas (`out.js`) | Conteúdo |
|---|---|---|
| M9040 | L2865490–2865800 | `SubscriptionsWidgetConnected` (L2865501) |
| M9041 | L2865770 | barrel: reexporta `SubscriptionsWidget`, `SubscriptionsWidgetFull`, `SubscriptionRow` |
| M9042 | L2865800–2867270 | `LogoItem` (L2865814), `LogoStack` (L2866089), `StateIcon` (L2866321), `FullState` (L2866383), `WideState` (L2866490), **`SubscriptionsWidget`** (L2866635), `StyleSheet` (L2867237) |
| M9043 | L2867274–2867650 | **`SubscriptionRow`** (memo, L2867327) + estilos (L2867636). Usado no full do widget **e** na tela de detalhe |
| M9044 | L2867655 | `SubscriptionsWidgetFull` (L2867690) = `<SubscriptionsWidget {...props} size="full"/>` |
| M9045 | L2867731 | `useSubscriptionsData(size)` (L2867743) |

Registro (`VISION_WIDGET_REGISTRY`, L772591):
```js
{ id: 'subscriptions', size: 'half', supportedSizes: ['half','wide','full'], label: 'Assinaturas',
  tapAction: 'navigate', navigateTo: '/(protected)/vision/subscriptions?filterKey=subscriptions' }
```
- Picker/catálogo (L775652): `description: 'Gerencie suas assinaturas recorrentes'`, `icon: 'deviceTv'`.
- Layout padrão: `{id:'subscriptions', size:'half', position: 10 | 9, visible: true}` (L2896988 / L2897010).
- Analytics de tap: `{widgetId: 'subscriptions', button: 'body'}` (L751448).
- Rota nomeada (L754707): `subscriptions: '/(protected)/vision/subscriptions'`.

---

## 1. Dados [O]

### 1.1 Hook
`useSubscriptionsData(size)` (L2867743) = `useVisionSystemWidgetData('subscriptions', size)` → `{data, isLoading, isError, isFetched}` (memoizado). Endpoint, queryKey, staleTime e cache local: **W1 §1** (`GET /api/mobile/vision/widgets/subscriptions/data?size=…`). Todo o cálculo (total, ordenação, quais assinaturas aparecem) é feito **no servidor**; o widget só formata.

### 1.2 `SubscriptionsWidgetConnected({size, onPress, emptyState, animate = false})` (L2865501)
```ts
const { data, isLoading, isError, isFetched } = useSubscriptionsData(size)
const loading = isLoading || !isFetched
const d = data ?? { logos: [], amount: 'R$ 0,00', subtitle: 'Próxima em 5 dias', totalCount: 0 }   // default memoizado
<SubscriptionsWidget logos={d.logos} amount={d.amount} subtitle={d.subtitle} totalCount={d.totalCount}
  size={size} onPress={onPress} isLoading={loading} emptyState={emptyState} hasError={isError} animate={animate}/>
```
`emptyState` e `animate` são repassados mas **não são lidos** dentro de `SubscriptionsWidget` (só desestruturados) [O].

### 1.3 Payload do widget (`data`)
| Campo | Tipo | Uso |
|---|---|---|
| `amount` | string BRL já formatada (`"R$ 116,70"`) | total do mês. half: `parseBRLToCents(amount)` → `PrivacyCurrencyText`; wide: texto direto |
| `subtitle` | string | linha inferior (half/wide) quando `logos.length !== 1` |
| `totalCount` | number | contador do "+N" na pilha de logos |
| `logos[]` | array | itens; campos lidos: `name`, `logoUrl`, `nextRenewalDate` (ISO/`YYYY-MM-DD`), e no full também `bankName`, `bankLogoUrl`, `amount` (string BRL) |

### 1.4 Regras de derivação dentro de `SubscriptionsWidget` (L2866635)
- `logos = props.logos ?? []`; `isEmpty = logos.length === 0`; `hidden = useHideValues()`; `fg = useThemeColors().foreground` (#fafafa).
- **Valor exibido** (`displayAmount`): `hidden ? HIDDEN_VALUE : isEmpty ? 'R$ 0,00' : amount`.
- **Subtítulo** (`subtitleText`):
  - se `logos.length !== 1` → `subtitle || 'Próxima em 5 dias'` (o literal é o fallback do código);
  - se exatamente 1 item e há `logos[0].nextRenewalDate`: `days = Math.ceil((createBrazilCalendarDate(next) − getTodayBrazilCalendarDate()) / 86 400 000)`:
    - `days <= 0` → **`'renovou recentemente'`**
    - `days === 1` → **`'Próxima amanhã'`**
    - senão → **`` `Próxima em ${days} dias` ``**
  - sem data → `subtitle`.
- **Linhas do full**: `logos.slice(0, 4)` (constante `4`, L2865807). **Pilha de logos**: `logos.slice(0, 3)` (constante `3`, L2865805), sobreposição `marginLeft: -14` (L2865803).

---

## 2. Widget: layout por tamanho [O]

O widget **não** usa `Widget2x2`/`WidgetWide`/`WidgetFull`: monta tudo sobre `WidgetGlassCard` (W0 §9.5) com estilos próprios (L2867237).

### 2.1 Estilos (`StyleSheet`, L2867237)
| Chave | Estilo |
|---|---|
| `halfCard` | `{height: 200, borderRadius: 24, padding: 16, overflow: 'hidden'}` |
| `wideCard` | `{minHeight: 176, borderRadius: 24, padding: 18, overflow: 'hidden'}` |
| `fullCard` | `{minHeight: 344, borderRadius: 24, padding: 20, overflow: 'hidden'}` |
| `fullStateCard` | `{minHeight: 192, borderRadius: 24, padding: 20, overflow: 'hidden'}` (full vazio ou com erro, fora do loading) |
| `cardContent` | `{flex: 1, gap: 16}` |
| `logoRow` | `{flexDirection: 'row', alignItems: 'center'}` |
| `logoCircleOuter` | `{width: 40, height: 40, borderRadius: 20}` |
| `logoCircleClip` | `{...StyleSheet.absoluteFill, borderRadius: 20, overflow: 'hidden'}` |
| `logoImage` | `{width: 52, height: 52, marginTop: -6, marginLeft: -6}` (imagem 30 % maior que o círculo e recortada: o "zoom" elimina margens brancas do PNG) |
| `logoFallback` | `{width: 40, height: 40, alignItems: 'center', justifyContent: 'center'}` |
| `stateIcon` | `{width: 44, height: 44, borderRadius: 22}` |

Escolha do estilo do card: `full && !isLoading && (hasError || isEmpty)` → `fullStateCard`; senão `full` → `fullCard`; `wide` → `wideCard`; senão `halfCard`. Fundo = `WidgetGlassCard` (glass "clear" quando há Liquid Glass; senão `bg-surface` **#18181b**). Sem borda nem sombra.

Atenção: a altura do wide é `minHeight 176` e a do half `height 200`; o slot do grid é que fixa 170 no wide (W0 §3) [I: o card estica/encolhe para o slot].

### 2.2 Casca comum
```tsx
const card = (
  <WidgetGlassCard style={cardStyle}>
    <View testID="subscriptions-widget" style={styles.cardContent /* flex 1, gap 16 */}>
      {isFull && (
        <HStack className="items-center justify-between">
          <Text allowFontScaling={false} className="text-base text-muted-foreground" numberOfLines={1}>Assinaturas esse mês</Text>
          <PierreIcon name="chevronRight" size={16} color="#F4F4F5" />
        </HStack>
      )}
      {content /* ramo abaixo, memoizado */}
    </View>
  </WidgetGlassCard>
)
return onPress
  ? <Pressable testID="subscriptions-widget-pressable" onPress={onPress} accessibilityRole="button">{card}</Pressable>
  : card
```
Notas: o chevron deste widget é **#F4F4F5** (zinc-100), diferente do `#71717a` dos componentes base. Não há háptico nem escala no press (Pressable puro) [O]. O `onPress` vem do renderer e navega para `navigateTo`.

### 2.3 Ramos do conteúdo (useMemo dentro de `SubscriptionsWidget`)
Ordem de teste: `isLoading` → `hasError` → `full` → `wide` → `half`.

| Estado | half | wide | full |
|---|---|---|---|
| carregando | `WidgetLoadingState variant="value"` | idem `"value"` | `WidgetLoadingState variant="content"` |
| erro | árvore própria (§2.4.3) | `WideState isError` | `FullState isError` |
| vazio | pilha → `StateIcon type="empty"`, valor `R$ 0,00` | pilha → `StateIcon type="empty"` | `FullState isError={false}` |
| normal | §2.4 | §2.5 | §2.6 |

### 2.4 half (200 px, padding 16)
```
VStack "flex-1 justify-between"
 ├─ HStack "justify-between"
 │   ├─ isEmpty ? StateIcon type="empty" : LogoStack logos totalCount useMerchantAvatar   (MerchantAvatar 40)
 │   └─ PierreIcon chevronRight 16 #F4F4F5
 └─ VStack "gap-1"                                          // gap 3,5
     ├─ Text "text-base text-muted-foreground" numberOfLines=1 → "Assinaturas"
     ├─ PrivacyCurrencyText valueCents={parseBRLToCents(isEmpty ? 'R$ 0,00' : amount)} hidden={hideValues}
     │      fontSize=20 fontWeight="800" color={#fafafa} disableMinWidth compactOnOverflow       (W0 §11.4)
     └─ Text "text-base text-muted-foreground" numberOfLines=1 → subtitleText (§1.4)
```
| Texto | Fonte | size / lineHeight | Cor |
|---|---|---|---|
| "Assinaturas" | Geist-Regular | 14 / 21 | #a1a1aa |
| valor | `fontWeight '800'` → ExtraBold [I: resolvido pelo `PierreAnimatedCurrencyText` nativo] | 20 / `ceil(20×1,18)` = 24 | #fafafa |
| subtítulo | Geist-Regular | 14 / 21 | #a1a1aa |

#### 2.4.1 `LogoStack({logos, totalCount, useMerchantAvatar = false})` (L2866089)
```tsx
const visible = logos.length > 3 ? logos.slice(0, 3) : logos
const overflow = (totalCount ?? logos.length) - 3
<View testID="subscriptions-logo-stack" style={styles.logoRow}>
  {visible.map((item, i) => (
    <View key={`${item.name}-${i}`} style={i !== 0 ? { marginLeft: -14, zIndex: visible.length - i } : undefined /*[I] índice 0*/}>
      {useMerchantAvatar
        ? <MerchantAvatar name={item.name} logoUrl={item.logoUrl} testID={`subscription-logo-${item.name.toLowerCase()}`} size={40} />
        : <LogoItem name={item.name} logoUrl={item.logoUrl} index={i} total={visible.length} />}
    </View>
  ))}
  {overflow > 0 && (
    <View className="items-center justify-center bg-muted" style={[styles.logoCircleOuter, { marginLeft: -14, zIndex: 0 }]}>
      <Text allowFontScaling={false} className="text-sm font-bold text-muted-foreground">+{overflow}</Text>
    </View>
  )}
</View>
```
- Círculos de **40**, sobrepostos em **14** (passo de 26 px). O primeiro fica por cima (`zIndex` decrescente). "+N": fundo `bg-muted` **#27272a**, texto Geist-Bold **16** (`text-sm`, sem lineHeight) #a1a1aa.
- O half usa `MerchantAvatar` (size 40, badgeSize default 30). Como não recebe `bankLogoUrl`, **não tem selo**. Lógica de logo: W3 §6.

#### 2.4.2 `LogoItem({name, logoUrl, index, total})` (L2865814), usado no **wide**
```tsx
const match = useMemo(() => matchMerchantLogo(name), [name])              // logo local por palavra-chave (W3 §6.4)
const remote = useMemo(() => { const u = logoUrl ?? getCompanyLogoUrl(name); return u?.startsWith('/') ? getBaseUrl() + u : u }, [logoUrl, name])
const local = match?.logoSource ?? null
const [failed, setFailed] = useState(false)
<View className="bg-muted" style={[styles.logoCircleOuter, { marginLeft: index !== 0 ? -14 : 0, zIndex: total - index }]}>
  <View style={styles.logoCircleClip}>
    {(local || remote) && !failed
      ? <Image source={local ?? { uri: remote }} style={styles.logoImage /* 52×52, −6/−6 */} resizeMode="cover"
               testID={`subscription-logo-${name}`} onError={() => setFailed(true)} />      // Image do RN, não expo-image
      : <View style={styles.logoFallback}>
          <Text allowFontScaling={false} className="text-base font-bold text-muted-foreground">{name.charAt(0).toUpperCase()}</Text>
        </View>}
  </View>
</View>
```
Diferenças em relação ao `MerchantAvatar`: imagem **ampliada para 52 px** dentro do círculo de 40 (o `MerchantAvatar` desenha a imagem no tamanho do círculo), sem selo de banco, sem ícone de categoria. Fundo `bg-muted` #27272a e inicial Geist-Bold 14/21 #a1a1aa são iguais.

#### 2.4.3 `StateIcon({type})` (L2866321) e half com erro
```tsx
<View className="items-center justify-center bg-muted" style={styles.stateIcon /* 44×44, r 22, #27272a */}>
  <PierreIcon name={type === 'error' ? 'feedbackWarning' : 'timeClock'} size={18} color={type === 'error' ? '#EF4444' : '#A1A1AA'} />
</View>
```
half com erro:
```
VStack "flex-1 justify-between"
 ├─ HStack "items-start justify-between" → StateIcon type="error" + chevron 16 #F4F4F5
 └─ VStack "gap-1"
     ├─ "Assinaturas"        text-base text-muted-foreground          (Geist-Regular 14/21 #a1a1aa)
     ├─ "R$ 0,00"            text-xl font-extrabold text-foreground   (Geist-ExtraBold 17,5/24,5 #fafafa)
     └─ "Falha ao carregar"  text-base text-destructive               (Geist-Regular 14/21 #ef4444)
```

### 2.5 wide (minHeight 176, padding 18)
```
VStack "flex-1 justify-between"
 ├─ VStack "gap-6"                                                  // gap 21
 │   ├─ HStack "items-start justify-between"
 │   │   ├─ VStack "min-w-0 flex-1"
 │   │   │   ├─ Text "text-base text-muted-foreground" numberOfLines=1 → "Assinaturas esse mês"
 │   │   │   └─ PrivacyText "text-3xl font-extrabold text-foreground" numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.72
 │   │   │                  value={displayAmount}                    // Geist-ExtraBold 26,25/31,5 #fafafa
 │   │   └─ PierreIcon chevronRight 16 #F4F4F5
 │   └─ isEmpty ? StateIcon type="empty" : LogoStack logos totalCount          // LogoItem (40, imagem 52)
 └─ Text "text-base text-muted-foreground" numberOfLines=1 → subtitleText
```
O valor do wide é texto estático (`PrivacyText`, fade de 150 ms ao mascarar), não o `PierreAnimatedCurrencyText`.

`WideState({isError})` (L2866490; no código só é chamado com `isError: true`):
```
VStack "flex-1 justify-between"
 ├─ HStack "items-start justify-between" → "Assinaturas esse mês" (text-base muted) + chevron 16 #F4F4F5
 └─ HStack "items-center gap-4"                                     // gap 14
     ├─ StateIcon type={isError ? 'error' : 'empty'}
     └─ Text className={isError ? 'text-lg text-destructive' : 'text-lg text-muted-foreground'} numberOfLines=1
            adjustsFontSizeToFit minimumFontScale=0.8 → isError ? "Falha ao carregar assinaturas" : "Não há assinaturas próximas"
```
`text-lg` = Geist-Regular 15,75/24,5, cor #ef4444 ou #a1a1aa.

### 2.6 full (minHeight 344, padding 20)
Cabeçalho da casca (§2.2: "Assinaturas esse mês" + chevron) e depois:
```
VStack "flex-1 justify-between"
 └─ VStack "gap-4"                                                  // gap 14 entre linhas
     └─ logos.slice(0, 4).map((s, i) =>
          <SubscriptionRow key={`${s.name}-${i}`} name={s.name} logoUrl={s.logoUrl} bankName={s.bankName}
             bankLogoUrl={s.bankLogoUrl} amount={s.amount} amountHidden={hideValues} nextRenewalDate={s.nextRenewalDate} />)
```
Sem `highlight` nem `subtitle`, o subtítulo sai **cinza** (`text-base text-muted-foreground`) no formato `Em N dias • 7 Out` ou `7 Out • Pago` (§3). O full **não** mostra o total nem o contador.

`FullState({isError})` (L2866383), card `fullStateCard` (minHeight 192):
```
VStack "flex-1 justify-center gap-6"
 └─ HStack "items-center gap-4"
     ├─ StateIcon type={isError ? 'error' : 'empty'}
     └─ Text className={isError ? 'text-lg text-destructive' : 'text-lg text-muted-foreground'} numberOfLines=1
            → "Falha ao carregar assinaturas" | "Não há assinaturas próximas"
```

### 2.7 Strings do widget [O]
`Assinaturas`, `Assinaturas esse mês`, `R$ 0,00`, `Próxima em 5 dias` (fallback), `renovou recentemente`, `Próxima amanhã`, `Próxima em {n} dias`, `Falha ao carregar`, `Falha ao carregar assinaturas`, `Não há assinaturas próximas`, `+{n}`, `R$ ••••`.

---

## 3. `SubscriptionRow` — a linha (widget full **e** tela de detalhe) [O]

M9043, L2867327 (memo), estilos L2867636. **Não** é o `VisionTransactionRow` (W3 §5), mas usa o **mesmo `MerchantAvatar`** (W3 §6) para logo + selo do banco. Abaixo, só o que é próprio da linha.

### 3.1 Props
`name, logoUrl, bankName, bankLogoUrl, useBankAsPrimary = false, amount (string já formatada), nextRenewalDate, subtitle, highlight, amountHidden = false, avatarIconName, avatarIconColor = '#FAFAFA', categoryEmoji, categoryName, progress, onPress`.
Constante do módulo: `AVATAR_SIZE = 48` (L2867325).

### 3.2 Estilos (L2867636)
| Chave | Estilo |
|---|---|
| `row` | `{width: '100%', minHeight: 60, paddingVertical: 8, flexDirection: 'row', alignItems: 'center', columnGap: 12}` |
| `textBlock` | `{flex: 1, minWidth: 0, height: 48, justifyContent: 'center', alignSelf: 'center'}` |
| `title` | `{lineHeight: 24}` |
| `subtitle` | `{marginTop: 2, lineHeight: 20}` |
| `progressChart` | `{marginTop: 8, height: 8}` |
| `amount` | `{minWidth: 92, textAlign: 'right', flexShrink: 0}` |

Altura efetiva da linha: `max(60, 48 + 2×8)` = **64 dp**. Conferido no screenshot: os avatares começam em y = 981, 1161 e 1341 px → passo de 180 px = **64 dp**; não há gap entre linhas.

### 3.3 Lógica
```ts
// texto de data quando o servidor NÃO manda `subtitle` (caso do widget full)
const autoSubtitle = useMemo(() => {
  if (!nextRenewalDate) return null
  const d = createBrazilCalendarDate(nextRenewalDate)          // 'YYYY-MM-DD' → Date UTC 12:00 + 3 h (L2722213)
  const days = Math.ceil((d.getTime() - getTodayBrazilCalendarDate().getTime()) / 86_400_000)
  const label = formatBrazilDayMonthShort(d)                    // Intl pt-BR {day:'numeric', month:'short'} sem '.' (L2722441)
  if (days <= 0) return `${label} • Pago`
  if (days === 1) return `Em 1 dia • ${label}`
  return `Em ${days} dias • ${label}`
}, [nextRenewalDate])
let line = subtitle ?? autoSubtitle
if (highlight) line = `${line ?? ''} • ${highlight}`.trim()
const subtitleClass = highlight ? 'text-base text-pierre' : 'text-base text-muted-foreground'

// barra de progresso opcional (não usada em assinaturas)
const progressData = typeof progress === 'number'      // no bytecode: typeof … (o decompilador mostra 'bigint')
  ? [{ id: 'filled', label: '', value: clamp(progress, 0, 1), color: '#D5FF3E' },
     { id: 'track',  label: '', value: 1 - clamp(progress, 0, 1), color: isLight ? '#FAFAFA' : '#18181B' }]
  : []
```
- `getTodayBrazilCalendarDate()` usa `Intl.DateTimeFormat('en-CA', {timeZone: 'America/Sao_Paulo', year, month: '2-digit', day: '2-digit'})` → "hoje" no fuso de São Paulo, normalizado para 12:00 UTC + 3 h. As duas datas ficam no mesmo horário, então `ceil` dá a diferença exata em dias.
- **Na tela de detalhe o servidor manda `subtitle` e `highlight`**. A linha mostra `"{subtitle} • {highlight}"` em verde-lima. O screenshot ("7 Out • Pago", lima) bate com `subtitle = "7 Out"` e `highlight = "Pago"` [I forte: é o único caminho do código que pinta o subtítulo de lima; e o `formatBrazilDayMonthShort` local geraria "7 de out", não "7 Out"]. Ou seja, **o status e o formato "7 Out" vêm prontos do servidor**.
- No widget full não há `subtitle` nem `highlight`, então o texto é local e cinza: `Em 3 dias • 10 de out` ou `7 de out • Pago` [I: saída do ICU pt-BR para `{day:'numeric', month:'short'}` é "7 de out." e o código remove o ponto].

### 3.4 Árvore
```tsx
<TouchableOpacity activeOpacity={0.85} onPress={onPress} disabled={!onPress} style={styles.row}>
  <MerchantAvatar name={name} logoUrl={logoUrl} bankName={bankName} bankLogoUrl={bankLogoUrl}
    useBankAsPrimary={useBankAsPrimary} size={48} badgeSize={30}
    fallbackLabel={name.charAt(0).toUpperCase()} avatarIconName={avatarIconName} avatarIconColor={avatarIconColor}
    categoryFallback={categoryEmoji || categoryName ? { name: categoryName ?? name, emoji: categoryEmoji ?? null } : undefined} />
  <View style={styles.textBlock}>
    <Text allowFontScaling={false} className="text-lg font-bold text-foreground capitalize" style={styles.title} numberOfLines={1}>{name}</Text>
    {line ? <Text allowFontScaling={false} className={subtitleClass} style={styles.subtitle} numberOfLines={1}
                  adjustsFontSizeToFit minimumFontScale={0.58} ellipsizeMode="tail">{line}</Text> : null}
    {progressData.length > 0 && <PierreHorizontalBarChart data={progressData} barHeight={8} segmentGap={2}
        chartBackgroundColor="transparent" expandOnPress={false} groupingEnabled={false} showNavigationControl={false}
        animated animationKey={`progress-${Math.round((progress ?? 0) * 100)}`} style={styles.progressChart} />}
  </View>
  <PrivacyText allowFontScaling={false} className="text-lg font-bold text-foreground" style={styles.amount}
    value={amountHidden ? 'R$ ••••' : amount} numberOfLines={1} />
</TouchableOpacity>
```

### 3.5 Tipografia e cores
| Elemento | Classe | Fonte | size / lineHeight | Cor |
|---|---|---|---|---|
| nome | `text-lg font-bold text-foreground capitalize` + `lineHeight 24` | Geist-Bold | 15,75 / **24** | #fafafa |
| subtítulo (tela) | `text-base text-pierre` + `marginTop 2, lineHeight 20` | Geist-Regular | 14 / **20** (encolhe até 58 %) | **#d0ff00** (`--pierre`; JPEG mede ≈ (216, 252, 76)) |
| subtítulo (widget) | `text-base text-muted-foreground` | Geist-Regular | 14 / 20 | #a1a1aa |
| valor | `text-lg font-bold text-foreground` + `minWidth 92`, à direita | Geist-Bold | 15,75 / 24,5 | #fafafa |
| barra (se `progress`) | — | — | altura 8, gap 2 | preenchido **#D5FF3E**, trilho #18181B |

`capitalize` só sobe a primeira letra de cada palavra; "YouTube Premium" continua igual.

### 3.6 Avatar e selo (diferenças em relação a W3 §6)
- `size 48` (W3 usa 45 e 44), `badgeSize 30` → selo `BankLogo` de **22 px** em `right/bottom −5`, isto é, 1 px para fora do círculo. Conferido no screenshot: selo laranja com 58–59 px ≈ **21 dp** (o anel de anti-aliasing come cerca de 1 dp), borda direita em x = 180 px contra 179 px do avatar.
- Moldura `#27272A`, sem borda, clip circular (W3 §6.3). A Netflix aparece com fundo preto porque o próprio PNG é preto; o fundo do avatar não muda.
- O selo só aparece com `bankLogoUrl != null`. No screenshot as três assinaturas trazem o Itaú, ou seja, o servidor manda o banco da conta/cartão onde a cobrança foi detectada.
- Logos do screenshot: Claro, Netflix e YouTube Premium caem em `matchMerchantLogo` (asset local em `logos/lojas-e-apps/`). A logo local **vence** o `logoUrl` do servidor (W3 §6.2). Para nomes sem asset local, entra `getCompanyLogoUrl(name)` (`/api/logo?domain=…`; a tabela `netflix → netflix.com`, `spotify → spotify.com` etc. está em `logos/mapeamento.json` → `assinaturas`).

---

## 4. Tela de detalhe `VisionSubscriptionsScreen` [O]

Rota `/(protected)/vision/subscriptions?filterKey=subscriptions`. Módulo de rota M10440 = `withVisionScreenBoundary(VisionSubscriptionsScreen, 'subscriptions')`; barrel M10441 (exporta também `VisionSubscriptionsDayScreen`); tela M10442, função em **L3324576**. A tela é genérica ("subscriptions-screen" serve também a faturas, parcelas e limites), mas aqui `filterKey` é um `useState('subscriptions')` fixo e a lista de seções é `[filterKey]` = `['subscriptions']`.

### 4.1 Dados (M10250/M10251 L3241117–3242053; API M~4076 L1646240–1646520) [O]
| Hook | queryKey | Endpoint |
|---|---|---|
| `useVisionSubscriptionsSummary()` | `['vision-subscriptions-screen','v4','summary']` | `GET /api/mobile/vision/subscriptions-screen/summary` |
| `useVisionSubscriptionsCalendar()` | `[…,'calendar']` | `GET /api/mobile/vision/subscriptions-screen/calendar` |
| `useVisionSubscriptionsSection('subscriptions')` | `[…,'section','subscriptions']` [I ordem] | `GET /api/mobile/vision/subscriptions-screen/subscriptions` |
| (`useVisionUpcomingPayments`, não usado aqui) | `[…,'upcoming-payments']` | `GET /api/mobile/vision/subscriptions-screen/upcoming-payments` |

Todos passam por `useVisionSubscriptionsSectionQuery({queryKey, queryFn})`: `useQuery({queryKey, queryFn, staleTime: 300000 (5 min), gcTime: 1800000 (30 min), refetchOnReconnect: false})` + `useRefetchOnScreenFocus(query)`; devolve `{ data: query.data?.data ?? null, isLoading, isError, refetch }`, ou seja, **o corpo da resposta é `{ data: … }`** [O].

**Summary** (`data.summaries[filterKey]`), com fallback local (L3325544):
```ts
const FALLBACK = { summaries: Object.fromEntries(['subscriptions','invoices','installments','limits'].map(k =>
  [k, { pendingAmount: 0, pendingAmountFormatted: 'R$ 0,00', settledAmount: 0, settledAmountFormatted: 'R$ 0,00' }])) }
const summary = (summaryQ.data ?? FALLBACK).summaries[filterKey] ?? FALLBACK.summaries.limits
const headerValue = (summaryQ.isLoading && !summaryQ.data) ? 'R$ 0,00' : summary.pendingAmountFormatted   // "R$ 116,70"
const headerTitle = (filterKey === 'subscriptions' || filterKey === 'installments') ? 'Compromisso Mensal' : 'Total'
```
O "Compromisso Mensal" é o **`pendingAmountFormatted`** do servidor; `settled*` não aparece nesta tela.

**Section** (`useVisionSubscriptionsSection('subscriptions').data`):
```ts
type SubscriptionsSection = {
  title: string            // "3 assinaturas"            (texto pronto do servidor)
  totalLabel: string       // "R$ 116,70 esse mês"
  items: Array<{
    id: string; name: string; logoUrl?: string | null
    bankName?: string | null; bankLogoUrl?: string | null
    amountFormatted: string                 // "R$ 59,90"
    subtitle?: string | null                // "7 Out"   [I]
    highlight?: string | null               // "Pago"    [I]  → pinta o subtítulo de lima
    avatarIconName?: string; categoryEmoji?: string; categoryName?: string
    progress?: number                       // 0..1 (barra lima; não aparece no screenshot)
    representativeTransactionId?: string    // torna a linha clicável → detalhe da transação
    limitTotalFormatted?, limitAvailableFormatted?, limitUsedPercentage?   // só faturas
  }>
  chart?: unknown          // só faturas (InvoiceHistoryChart)
}
```
**Ordenação, filtro do mês, status (Pago/Pendente/…) e o total vêm prontos do servidor**. O app não reordena nem soma nada nesta tela [O: só `items.map`]. Os únicos status escritos no app ficam no widget full ("Pago" quando a data já passou) e no calendário (§5).

Analytics: `useScreenTracking(VISION_PAGE_NAMES.SUBSCRIPTIONS)`; `trackRecurringPaymentsInteraction('add_subscription_tapped' | 'calendar_day_tapped' | …)`. Erro do calendário → toast `showError('Não foi possível carregar os pagamentos do calendário')` (uma vez por ocorrência, com `ref`).

### 4.2 Estilos da tela (`StyleSheet`, L3328127) [O]
| Chave | Estilo |
|---|---|
| `animatedCurrencyInput` | `{padding: 0, includeFontPadding: false}` |
| `pendingAmountText` | `{marginTop: 4, fontSize: 44, fontWeight: '700', lineHeight: 48}` |
| `headerValueText` | `{fontSize: 20, lineHeight: 24, fontWeight: '700'}` |
| `headerButtonLarge` | `{height: 44, width: 44, alignItems: 'center', justifyContent: 'center', borderRadius: 22}` |
| `viewToggleContainer` | `{height: 44, borderRadius: 22, paddingHorizontal: 6, flexDirection: 'row', alignItems: 'center', position: 'relative'}` |
| `headerViewToggleButton` | `{height: 38, width: 38, borderRadius: 19, alignItems: 'center', justifyContent: 'center', zIndex: 1}` |
| `headerViewToggleHighlight` | `{position: 'absolute', top: 3, width: 38, height: 38, borderRadius: 19}` |
| `sectionsContainer` | `{paddingHorizontal: 16, paddingTop: 40, gap: 40}` |
| `skeletonCircle44` | `{width: 44, height: 44, borderRadius: 22}` |
| (demais) | `inlineFilterContainer`, `filterRow`, `filterPill`, `filterOption*`, `invoice*`: usados pelas outras abas (faturas/limites), não por assinaturas |

Constantes do módulo: `TOGGLE_BUTTON = 38` (slot38), `TOGGLE_PAD = 6` (slot39), botão do olho `{44×44, r 22}` (slot37), skeleton de 3 linhas `[0,1,2]`.

### 4.3 `GlassContainer` no Android (M≈2980, L924780) [O]
Os botões redondos e o toggle usam `GlassContainer glassEffectStyle="regular"`. No Android (sem Liquid Glass) ele vira um `View` com:
- `regular` → `backgroundColor: 'rgba(38, 38, 38, 0.92)'` (light: `rgba(240,240,240,0.88)`)
- `clear` → `#18181B` (light: `#FAFAFA`)
- `tintColor` (se houver) sobrescreve o fundo.
Sobre o fundo #09090b isso dá **rgb(36,36,36) = #242424**. O screenshot mede exatamente (36, 36, 36) no voltar, no olho e no toggle ✔.

### 4.4 Árvore
```
Box "flex-1 bg-background" style={{paddingTop: insets.top}} testID="vision-subscriptions-screen"          // #09090b
 ├─ Box "relative"
 │   └─ HStack "items-center justify-between px-5 py-3"                         // px 17,5 · py 10,5
 │       ├─ HStack "items-center gap-2"                                         // gap 7
 │       │   ├─ TouchableOpacity activeOpacity=0.7 onPress={useBackOrHome()}
 │       │   │   └─ GlassContainer regular style=headerButtonLarge (44, r22, #242424)
 │       │   │       └─ PierreIcon arrowLeft 22 #FAFAFA
 │       │   └─ Animated.View pointerEvents="none" style={{opacity: headerOpacity}}   // 0 → 1 quando scrollY > 60
 │       │       └─ AnimatedCurrencyText value={headerValue} hidden style=headerValueText (20/24, 700)
 │       └─ HStack "items-center gap-2"
 │           ├─ TouchableOpacity activeOpacity=0.8 onPress={toggleHideValues}
 │           │   └─ GlassContainer regular style={44×44 r22}
 │           │       └─ PierreIcon {hidden ? 'eyeOff' : 'eyeOn'} 18 #FAFAFA
 │           └─ GlassContainer regular style=viewToggleContainer (h44, r22, ph6, row)
 │               ├─ View pointerEvents="none" style=[headerViewToggleHighlight,
 │               │     {backgroundColor: 'rgba(255, 255, 255, 0.06)', transform: [{translateX: 6 + (view==='calendar' ? 38 : 0)}]}]
 │               ├─ TouchableOpacity activeOpacity=0.82 style=headerViewToggleButton onPress={() => setView('list')}
 │               │   └─ PierreIcon contentTextTopicList 18 {view==='list' ? '#FAFAFA' : '#A1A1AA'}
 │               └─ TouchableOpacity activeOpacity=0.82 style=headerViewToggleButton onPress={() => setView('calendar')}
 │                   └─ PierreIcon dateCalendar 18 {view==='calendar' ? '#FAFAFA' : '#A1A1AA'}
 └─ ScrollView ref horizontal pagingEnabled showsHorizontalScrollIndicator={false} directionalLockEnabled
              overScrollMode="never" removeClippedSubviews={false}
              onMomentumScrollEnd={e => setView(Math.round(e.nativeEvent.contentOffset.x / width) === 1 ? 'calendar' : 'list')}
     ├─ View style={{width}}                                                   // PÁGINA 1: lista
     │   └─ ScrollView showsVerticalScrollIndicator={false} overScrollMode="never" scrollEventThrottle={16}
     │                 onScroll={y => Animated.timing(headerOpacity, {toValue: y > 60 ? 1 : 0, duration: 160, useNativeDriver: true}).start()}
     │                 contentContainerStyle={{paddingBottom: insets.bottom + 32}}
     │       ├─ VStack "px-4 pt-6"                                             // ph 14 · pt 21
     │       │   ├─ Text "text-sm text-muted-foreground" → "Compromisso Mensal"     // Geist-Regular 16, #a1a1aa
     │       │   ├─ AnimatedCurrencyText value={headerValue} hidden style=pendingAmountText   // 44/48, mt 4
     │       │   └─ filterKey !== 'invoices' &&
     │       │       HStack "mt-6"                                                    // mt 21
     │       │        └─ PierreButton variant="secondary" size="large" textClassName="text-lg"
     │       │              leftIcon={<PierreIcon calcAdd 16 #FAFAFA/>} onPress={() => { track('add_subscription_tapped'); router.push('/vision/add-subscription') }}
     │       │              accessibilityLabel="Adicionar assinatura" testID="vision-subscriptions-add-button"> Adicionar assinatura
     │       ├─ insight.showSkeleton && <SubscriptionsInsightSkeleton/>
     │       ├─ insight.data?.message && !insight.isDismissed &&
     │       │    Box "px-4 mt-8" testID="vision-subscriptions-insight" → InsightCard title description onDismiss
     │       └─ Animated.View entering={FadeIn.duration(200).easing(Easing.out(Easing.ease))} style=sectionsContainer   // ph 16 · pt 40 · gap 40
     │           └─ ['subscriptions'].map(id => <SectionBlock key={id} sectionId={id} section={q.data} hidden
     │                    isLoading={q.isLoading && !q.data} isError={q.isError} onNavigateToTransaction={openTransaction} />)
     └─ View style={{width, flex: 1}}                                          // PÁGINA 2: calendário
         └─ VisionSubscriptionsCalendarView hidden activeView={calendarMode} calendarData={calendarQ.data ?? null}
                onViewChange={v => { if (v === 'day') track('calendar_day_tapped'); setCalendarMode(v) }}  // §5
```
- Estados da tela: `filterKey = useState('subscriptions')`, `view = useState('list')` (página: `'list' | 'calendar'`) e `calendarMode = useState('month')` (`'month' | 'week' | 'day'`, repassado ao calendário como `activeView`).
- `handleViewModeChange(mode)` (L≈3324780): `setView(mode)` e `scrollRef.current?.scrollTo({x: mode === 'calendar' ? width : 0, animated: true})`. Ou seja, o toggle e o swipe horizontal fazem a mesma coisa; o realce do toggle anda 38 px (troca instantânea: `translateX` sem animação no código desta tela) [O].
- `AnimatedCurrencyText` local (L3325554): converte "R$ 116,70" → `Number(v.replace(/[^\d,.-]/g,'').replace(/\./g,'').replace(',', '.'))` (não finito → 0) e renderiza `AnimatedVisionCurrencyText` (W0 §11.1) com `durationMs 400, hidden, hiddenText 'R$ ••••', privacyFadeDurationMs 150, fontWeight '900', style [animatedCurrencyInput, style]`. O número **conta** em 400 ms. Com `fontWeight '900'`, sai em Geist-Black [I: depende do mapeamento nativo do `PierreAnimatedCurrencyText`; o screenshot mostra um peso bem pesado].
- `PierreButton` (M≈934000): base `flex-row items-center justify-center rounded-full`; `secondary` → `bg-surface` (**#18181b**) e texto `text-foreground`; `large` → `px-6 py-4` (21 / 14), texto `text-base font-medium` com `textClassName 'text-lg'` → **Geist-Medium 15,75/24,5 #fafafa**; ícone com `gap-3` (10,5). Altura 14 + 24,5 + 14 = **52,5** (screenshot: 147 px = 52,3 dp ✔; cor medida (24, 24, 26) ✔). O botão tem escala no press (`transform scale`, M≈924168) [O, valor não extraído].

### 4.5 `SectionBlock({sectionId, section, hidden, isLoading, isError, onNavigateToTransaction})` (L3327034) [O]
Ordem: `isLoading` → `SectionBlockSkeleton`; `isError` → `VStack "pt-6"` [`VisionErrorState title="Tivemos problemas para carregar as assinaturas"`, `VisionEmptyCard iconName="timeClock" label="Não há assinaturas" className="mt-10"`]; `!section` → `VisionEmptyCard timeClock "Não há assinaturas" className="mt-4"`; senão:
```
VStack
 ├─ HStack "items-center justify-between"
 │   ├─ Text "text-lg text-muted-foreground" → section.title                     // "3 assinaturas" · Geist-Regular 15,75/24,5 #a1a1aa
 │   └─ PrivacyText "text-sm text-muted-foreground" value={hidden ? 'R$ ••••' : section.totalLabel}   // "R$ 116,70 esse mês" · 16 #a1a1aa
 ├─ items.length > 0
 │    ? VStack "mt-2" → items.map(item => <View key={item.id}>
 │          <PaymentRow item hidden useBankAsPrimary={sectionId === 'invoices'}
 │             onPress={item.representativeTransactionId && onNavigateToTransaction ? () => onNavigateToTransaction(item.representativeTransactionId) : undefined}/>
 │          {/* bloco de limite só para faturas */}
 │        </View>)
 │    : VisionEmptyCard iconName="timeClock" label="Não há assinaturas" className="mt-4"
 └─ sectionId === 'invoices' && section.chart ? <InvoiceHistoryChart …/> : null
```
`PaymentRow` (L3325661) é só um adaptador: `<SubscriptionRow name logoUrl bankName bankLogoUrl useBankAsPrimary amount={item.amountFormatted} subtitle highlight amountHidden={hidden} avatarIconName categoryEmoji categoryName progress onPress />` (§3).

Componentes auxiliares:
- `VisionEmptyCard` (L3242699): container `"items-center gap-3 bg-surface px-4 py-4 " + className`, `borderRadius 16`, `PierreIcon {iconName} 20 #71717a` + `Text "text-sm text-muted-foreground"` (16, #a1a1aa) [O; direção da linha I: HStack].
- `VisionErrorState` (L3242865): `"items-center " + className`; `PierreIcon buildingsBankWarning 26 #FAFAFA`; título `"mt-6 text-[17px] font-medium text-foreground"`; descrição default `"Tente novamente em instantes"` em `"mt-2 text-base text-muted-foreground"`.
- `SectionBlockSkeleton` (L3327883): cabeçalho com barras `120×18 r9` e `110×16 r8` (`items-center justify-between`); `VStack "mt-2"` com 3 linhas `HStack "items-center gap-3 py-3"` = círculo 44 + `[146×18 r9, mt 6 → 96×14 r7]` (flex-1) + `86×18 r9`; cor `#27272A` (light `#E4E4E7`).
- `SubscriptionsInsightSkeleton` (L3327635): `Box "px-4 mt-8" testID="vision-subscriptions-insight-skeleton"`; cartão `{borderRadius 28, borderWidth 1, borderColor #27272A, backgroundColor #141417, padding 20}`; `HStack "items-center gap-4"` = `VStack "flex-1 gap-3"` [72 %×22 r11, 92 %×18 r9, 66 %×18 r9] + quadrado `84×84 r18`.

### 4.6 Medidas conferidas no screenshot (2,8125 px/dp)
| Elemento | Screenshot (px) | dp | Código |
|---|---|---|---|
| Voltar | x 49–172, y 131–253 | 44 × 44, esquerda 17,4 | 44, `px-5` = 17,5 ✔ |
| Topo do header | y 131 | 46,6 = inset 36 + `py-3` 10,5 | ✔ (inset de status bar ≈ 36 dp) |
| Olho | x 640–763 | 44 | ✔ |
| Gap olho → toggle | 763 → 783 | 7 | `gap-2` = 7 ✔ |
| Toggle | x 783–1030, y 131–253 | 88 × 44 | 6 + 38 + 38 + 6 = 88 ✔; margem direita 17,8 ≈ 17,5 ✔ |
| Realce do toggle | x 800, y 139–245 | 38 × 38, offset 6 | ✔; cor medida (49, 49, 49) = 6 % branco sobre #242424 ✔ |
| "Compromisso Mensal" | x 39, glifo y 356–395 | margem 14 | `px-4` ✔ |
| Botão adicionar | x 39–663, y 608–755 | 222 × 52,3 | altura 52,5 ✔, `rounded-full` |
| "3 assinaturas" | glifo y 887–920 | — | linha 24,5 que começa 40 dp abaixo do botão (`paddingTop 40`) ✔ |
| Avatar | x 45–179, y 981–1115 | 48, margem 16 | `size 48`, `paddingHorizontal 16` ✔ |
| Passo entre linhas | 180 | 64 | ✔ |
| Valor à direita | termina em x 1033 | margem 16 | ✔ |
| Selo itaú | 58–59 px | ≈ 21 | `BankLogo 22` ✔ |
| Fundo | (9, 9, 11) | — | #09090b ✔ |
| Texto cinza | (161, 161, 169) | — | #a1a1aa ✔ |

---

## 5. Calendário — `VisionSubscriptionsCalendarView` (página 2 do pager) [O]

Módulo M10254 (L3242993–3250770): `VisionSubscriptionsCalendarView` (memo, **L3249962**), `VisionSubscriptionsCalendarDayContent` (L3248955), `CalendarDayCell` (L3245026), `CalendarWeekCard` (L3245317), `CalendarViewSegmentControl` (L3246333), `CalendarTimelineDay` (L3247156), `CalendarTimelineEventRow`/`Preview` (L3246735/L3246962), `CalendarCollapsed*` (L3247794–3248955), `CalendarAvatar`/`CalendarEventPill`/`CalendarDayEventAvatar`/`CalendarDayEventChip`/`CalendarDayFloatingEventButton` (L3243944–3245024), `getClosestCalendarDayGroup` (L3243001), `resolveCalendarLogoUrl`, `getCalendarTimedEventsByDate`. Formatação de datas: M10255/L3257122. Grade: **`@marceloterreiro/flash-calendar`** (M7504: `Calendar`, `useCalendar`, `toDateId`, `fromDateId`, `useOptimizedDayMetadata`…).

### 5.1 Dados (`GET /api/mobile/vision/subscriptions-screen/calendar` → `data`)
```ts
type CalendarData = {
  monthLabel: string                       // título grande (ex.: "Outubro") [I conteúdo]
  events: Array<{
    id: string; dateId: string             // 'YYYY-MM-DD'
    title: string; amount: number; amountFormatted: string; subtitle?: string
    logoUrl?: string | null; bankLogoUrl?: string | null; bankName?: string | null
    accentColor?: string | null; fallbackLabel?: string
    avatarIconName?: string                // 'pixFilled' ⇒ type 'pix'; senão 'brand'
    startHour?: number; durationHours?: number   // usados só pela timeline por hora
  }>
}
```
Derivações (L3249962+):
- `itemsByDate` (grade): `{ [dateId]: Entry[] }` com `Entry = {id, type: avatarIconName === 'pixFilled' ? 'pix' : 'brand', name: title, amount: amountFormatted, subtitle, logoUrl, bankLogoUrl: ?? null, bankName: ?? null, accentColor: ?? null, fallbackLabel}`; no modo `'day'` vira `{}`.
- `dayGroups`: agrupa por `dateId`, **ordena os dias por `dateId.localeCompare`** e, dentro do dia, por `startHour` e depois `title.localeCompare`; `totalAmount = Σ event.amount`, `totalAmountFormatted = Intl.NumberFormat('pt-BR', {style:'currency', currency:'BRL'})`. Ou seja, **o total do dia é somado no app**.
- `getClosestCalendarDayGroup(groups, date)`: o grupo cuja data (UTC) está mais perto da data tocada; no empate, prefere o dia **≤** a data tocada (passado) [O L3243001].
- Mês exibido: `new Date(today.getFullYear(), today.getMonth(), 1)` (sempre o mês corrente; não há navegação entre meses) [O]. Semana: 7 dias a partir da **segunda-feira** da semana de hoje.
- `useCalendar({calendarMonthId: toDateId(firstOfMonth), calendarFirstDayOfWeek: 'monday', calendarFormatLocale: 'pt-BR', getCalendarMonthFormat: formatCalendarMonthLong, getCalendarWeekDayFormat: formatCalendarWeekdayShort})`.
- Formatadores (M10255, todos `Intl` pt-BR): `formatCalendarWeekdayShort` = `{weekday:'short'}` sem ".", ex. "seg"; `formatCalendarMonthLong` = `{month:'long'}` com inicial maiúscula ("Outubro"); `formatCalendarDayMonthShort` = `{day:'numeric', month:'short'}` ("7 de out.").

Constantes (L3243883): `DAY_HEIGHT 98`, `DAY_SPACING 4`, `WEEKNAME_HEIGHT 28`, `PILL_MAX_CHARS 18`, `40`, `4`, `CALENDAR_DAY_TIMELINE_HEIGHT 1024`, `CALENDAR_DAY_TIMELINE_TOP_MARGIN 16`, `CALENDAR_VIEW_OPTIONS = [{key:'month', label:'Mensal'}, {key:'day', label:'Diária'}]`, horas `'00:00'…'23:00'`.

### 5.2 Estilos (`StyleSheet`, L3250752) — os que aparecem em assinaturas
| Chave | Estilo |
|---|---|
| `containerContent` | `{paddingBottom: 32}` |
| `segmentControl` | `{marginTop: 32, minHeight: 58, borderRadius: 999, padding: 5, overflow: 'hidden'}` |
| `segmentTrack` | `{position: 'relative', flexDirection: 'row', alignItems: 'center', minHeight: 48}` |
| `segmentPressable` | `{flex: 1, height: 48}` |
| `segmentActivePill` | `{position: 'absolute', top: 0, left: 0, height: 48, borderRadius: 999}` |
| `segmentLabelOverlay` | `{position: 'absolute', top: 0, alignItems: 'center', justifyContent: 'center', height: 48, width: '50%'}` |
| `segmentLabelText` | `{fontSize: 17, fontWeight: '400'}` |
| `weekdayLabel` | `{color: '#A1A1AA', fontSize: 16, textAlign: 'left'}` |
| `weeksStack` | `{gap: 4}` |
| `dayCardGlass` | `{borderRadius: 10, overflow: 'hidden', minHeight: 98}` |
| `dayCard` | `{flex: 1, borderRadius: 10, paddingHorizontal: 10, paddingVertical: 8}` |
| `avatarShell` | `{marginTop: 6, alignItems: 'center', justifyContent: 'center', position: 'relative', overflow: 'visible'}` |
| `eventPill` | `{minHeight: 22, borderRadius: 999, paddingLeft: 6, paddingRight: 8, flexDirection: 'row', alignItems: 'center', marginTop: 4}` |
| `eventPillDot` | `{width: 14, height: 14, borderRadius: 7, alignItems: 'center', justifyContent: 'center', marginRight: 4}` |
| `weekGrid` | `{flexDirection: 'row', flexWrap: 'wrap', justifyContent: 'space-between', rowGap: 12}` |
| `weekDayCard` | `{width: 120, minHeight: 130, borderRadius: 16, paddingHorizontal: 8, paddingVertical: 10, alignItems: 'center', justifyContent: 'space-between'}` |
| `dailyContentLayer` / `calendarStaticRoot` | `{flex: 1}` |
| `timelineList` | `{flex: 1, gap: 32, paddingHorizontal: 16, paddingTop: 32, paddingBottom: 32}` |
| `timelineDayContainer` | `{gap: 24}` (+ `timelineDayContainerExpanded` `{flex: 1, minHeight: 0, justifyContent: 'flex-start'}`) |
| `timelineDivider` | `{height: StyleSheet.hairlineWidth, backgroundColor: '#27272A', width: '100%'}` |
| `timelineDayContent` | `{alignItems: 'flex-start', justifyContent: 'space-between'}` |
| `timelineDateColumn` | `{width: 40, alignItems: 'center', gap: 4}` |
| `timelineDateBubble` | `{width: 40, height: 40, borderRadius: 20, borderWidth: 1, borderColor: '#27272A', alignItems: 'center', justifyContent: 'center'}`; ativo `{backgroundColor: '#FAFAFA'}` |
| `timelineDateText` | `{fontSize: 16, lineHeight: 24, fontWeight: '700'}` |
| `timelineDayMain` | `{width: 309, alignItems: 'center', justifyContent: 'center', gap: 16}` |
| `timelineEventsStack` | `{width: '100%', gap: 16}` |
| `timelineEventCard` | `{width: '100%', minHeight: 64, borderRadius: 24, paddingHorizontal: 16, paddingVertical: 12, flexDirection: 'row', alignItems: 'center', overflow: 'hidden'}` |
| `timelineEventContent` | `{flex: 1, marginLeft: 12, justifyContent: 'center'}` |
| `timelineCompactRow` / `timelineCompactCard` | `{width: '100%', gap: 8, paddingTop: 28}` / `{flex: 1, minHeight: 40, borderRadius: 24, paddingHorizontal: 16, paddingVertical: 8, flexDirection: 'row', alignItems: 'center', overflow: 'hidden'}` |
| `collapsedPreviousDay` / `…DateGroup` | `{alignItems: 'center', justifyContent: 'space-between', width: '100%'}` / `{alignItems: 'center', justifyContent: 'center', gap: 4}` |
| `collapsedPreviousDateBubble` | `{width: 24, height: 24, borderRadius: 12, borderWidth: 1, borderColor: '#27272A', alignItems: 'center', justifyContent: 'center'}` |
| `collapsedPreviousDayNumber` / `…Weekday` | `{color: '#FAFAFA', fontSize: 12, lineHeight: 20, fontWeight: '400', textAlign: 'center'}` / `{color: '#FAFAFA', fontSize: 14, lineHeight: 16, …}` |
| `collapsedPreviousAvatars` | `{alignItems: 'flex-start', gap: 8}` |
| `collapsedNextDayContainer` / `…Content` | `{gap: 24}` / `{alignItems: 'flex-end', justifyContent: 'space-between'}` |
| `collapsedNextEventsRow` | `{width: 309, height: 40, gap: 8, alignItems: 'center', justifyContent: 'center'}` |
| `collapsedNextEventCard` | `{flex: 1, height: 40, borderRadius: 24, paddingHorizontal: 16, paddingVertical: 8, flexDirection: 'row', alignItems: 'center', justifyContent: 'center', gap: 12, overflow: 'hidden', backgroundColor: 'rgba(39,39,42,0.40)'}` |
| `collapsedNextEventTitle` / `…DayNumber` | `{flex: 1, color: '#FAFAFA', fontSize: 16, lineHeight: 24, fontWeight: '400'}` / `{color: '#FAFAFA', fontSize: 16, lineHeight: 24, fontWeight: '400', textAlign: 'center'}` |
| (timeline por hora, não usada no fluxo principal) | `dayTimeColumn {width 52}`, `dayTimelineRow {minHeight 40}`, `dayEventChip {minHeight 52, r14}`, `dayFloatingButton {minHeight 56, r18}`, `dayEventsOverlay {left 52}` |

O `width: 309` é fixo (não acompanha a largura da tela) [O].

### 5.3 Árvore
```
// header (sempre)
VStack "px-4 pt-6"                                                    // ph 14 · pt 21
 ├─ Text "text-3xl font-bold text-foreground" → calendarData?.monthLabel        // Geist-Bold 26,25/31,5 #fafafa
 └─ CalendarViewSegmentControl activeView={activeView === 'week' ? 'week' : 'month' /* 'day' destaca "Diária" */} onViewChange

activeView === 'day'
  ? View style=calendarStaticRoot → [header,
      Animated.View entering={FadeInDown.duration(d).easing(Easing.out(Easing.cubic))}
                    exiting={FadeOutUp.duration(d).easing(Easing.in(Easing.cubic))} style=dailyContentLayer
        └─ VisionSubscriptionsCalendarDayContent date={selectedDate} hidden calendarData dayGroups ]      // §5.6
  : ScrollView showsVerticalScrollIndicator={false} contentContainerStyle=containerContent → [header,
      Animated.View (mesmas transições) style=contentTransitionLayer
        └─ month: VStack "px-3 pt-10"                                // ph 10,5 · pt 35
              ├─ Calendar.Row.Week spacing={4}
              │    └─ weekDaysList.map(d => <Calendar.Item.WeekName height={28} theme={{content: styles.weekdayLabel}}>{Capitalizado(d)}</…>)
              │                                                       // "Seg Ter Qua Qui Sex Sáb Dom" · 16 #a1a1aa (à esquerda)
              └─ VStack "mt-3" style=weeksStack (gap 4)
                   └─ weeksList.map((week, i) => <Calendar.Row.Week key={week.id ?? `week-${i}`} spacing={0}>
                          {week.map(day => <CalendarDayCell key={day.id} metadata={day} itemsByDate eventStyle="avatars"
                                              onPress={dateId => openDay(fromDateId(dateId))}/>)}</…>)
           week: VStack "px-4 pt-10" → View style=weekGrid → weekDays.map(date =>
                   <TouchableOpacity activeOpacity={0.82} onPress={() => openDay(date)}>
                     <CalendarWeekCard date entries={itemsByDate[toDateId(date)] ?? []} isToday eventStyle="pill"
                                       showMonthLabel={date.getMonth() !== today.getMonth()}/></TouchableOpacity>) ]
```
`openDay(date)`: `group = getClosestCalendarDayGroup(dayGroups, date)`; `setSelectedDate(group?.date ?? date)`; `onViewChange('day')` (a tela registra `calendar_day_tapped`). A duração `d` dos `FadeInDown/FadeOutUp` vem de um registrador que o decompilador não materializou [lacuna].

### 5.4 `CalendarViewSegmentControl` — "Mensal | Diária" (L3246333)
```tsx
const options = CALENDAR_VIEW_OPTIONS                    // [{month,'Mensal'}, {day,'Diária'}]
const index = Math.max(0, options.findIndex(o => o.key === activeKey))
const activeProgress = useSharedValue(index); const controlWidth = useSharedValue(0)
useEffect(() => { activeProgress.value = withTiming(index, { duration: 280, easing: Easing.out(Easing.cubic) }) }, [index])
const pillStyle = useAnimatedStyle(() => {             // worklet VisionSubscriptionsCalendarViewTsx1 (fonte no bundle)
  const w = controlWidth.value > 0 ? controlWidth.value / options.length : 0
  return { width: w, transform: [{ translateX: activeProgress.value * w }] }
})
<GlassContainer style={styles.segmentControl}>                       // regular → rgba(38,38,38,0.92) ≈ #242424 no Android
  <View style={styles.segmentTrack} onLayout={e => (controlWidth.value = e.nativeEvent.layout.width)}>
    <Animated.View pointerEvents="none" style={[styles.segmentActivePill, { width: `${100 / options.length}%`, backgroundColor: '#FFFFFF' }, pillStyle]} />
    {options.map(o => <TouchableOpacity key={o.key} activeOpacity={0.82} onPress={() => onViewChange(o.key)}
                          style={styles.segmentPressable} accessibilityRole="button" accessibilityLabel={o.label} />)}
    {options.map((o, i) => (
      <View key={o.key} pointerEvents="none" style={[styles.segmentLabelOverlay, { left: `${(i * 100) / options.length}%`, width: `${100 / options.length}%` }]}>
        <Text style={[styles.segmentLabelText, { color: o.key === activeKey ? '#09090B' : '#FAFAFA' /* light: #71717A */ }]} numberOfLines={1}>{o.label}</Text>
      </View>))}
  </View>
</GlassContainer>
```
Altura 58 (padding 5 + trilho 48 + 5), pílula ativa branca de 48 que desliza em 280 ms; rótulo 17 px peso 400 (Geist-Regular), preto #09090B no ativo e branco no inativo.

### 5.5 Célula do mês — `CalendarDayCell({metadata, itemsByDate, eventStyle = 'avatars', onPress})` (L3245026)
```tsx
const day = useOptimizedDayMetadata(metadata, 'vision-subscriptions-calendar')
const entries = day.isDifferentMonth ? [] : itemsByDate[day.id] ?? []
const visible = eventStyle === 'avatars'
  ? (entries.length > 2 ? entries.slice(0, 1) : entries.slice(0, 2))   // até 2 avatares; com 3+ mostra 1 + "+N"
  : entries.slice(0, 2)
const overflow = entries.length - visible.length
<Calendar.Item.Day.Container dayHeight={98} daySpacing={4} isStartOfWeek={day.isStartOfWeek} metadata={day} shouldShowActiveDayFiller={false}>
  {day.isDifferentMonth ? null : (                       // dias de outro mês: célula vazia
    <TouchableOpacity activeOpacity={0.82} onPress={() => onPress?.(day.id)}>
      <GlassContainer style={styles.dayCardGlass}>        // r 10, minHeight 98, regular ≈ #242424
        <View style={styles.dayCard}>                     // ph 10 · pv 8
          <Text className="text-[14px] font-medium text-foreground">{day.displayLabel}</Text>   // "7" · Geist-Medium 14 #fafafa
          {entries.length > 0 && (eventStyle === 'avatars'
            ? <VStack className="mt-3 items-center">{visible.map(e => <CalendarAvatar key={e.id} entry={e}/>)}
                {overflow > 0 && <Text className="mt-1 text-sm font-medium text-foreground">+{overflow}</Text>}</VStack>
            : <VStack className="mt-3">{visible.map(e => <CalendarEventPill key={e.id} entry={e}/>)}
                {overflow > 0 && <Text className="mt-1 text-xs font-medium text-muted-foreground">+{overflow}</Text>}</VStack>)}
        </View>
      </GlassContainer>
    </TouchableOpacity>)}
</Calendar.Item.Day.Container>
```
- Grade de 7 colunas com 4 px de espaço; largura de coluna = `(W − 2×10,5 − 6×4) / 7` [I: o flash-calendar divide igualmente]. Em 384 dp, cada célula tem ≈ 48,4 × 98.
- O dia de hoje **não** tem destaque próprio na grade (o `shouldShowActiveDayFiller` é falso e não há estilo de `isToday` na célula) [O].
- **`CalendarAvatar({entry})`** (L3243944): `entry.name === 'overflow'` → `View avatarShell` + `Text "text-base font-bold text-foreground"` com **"+2" fixo**. Senão:
  `<MerchantAvatar size={24} name={useBank ? (entry.bankName ?? entry.name) : entry.name} logoUrl={entry.logoUrl} bankName bankLogoUrl useBankAsPrimary={useBank} fallbackLabel={entry.fallbackLabel} avatarIconName={entry.type === 'pix' ? 'pixFilled' : undefined} avatarIconColor="#18181B"/>`, com `useBank = !entry.logoUrl && !!entry.bankLogoUrl`. Avatar de 24 com selo de banco (badgeSize default 30 → `BankLogo` 22; W3 §6) [O; o selo fica quase do tamanho do avatar: I].
- **`CalendarEventPill({entry})`** (modo pill/semana, L3244112): `color = entry.accentColor ?? (entry.type === 'brand' ? getCompanyBrandColor(entry.name) ?? '#A855F7' : '#A855F7')`; pílula `{...eventPill, backgroundColor: color + '40'}` (25 % de alfa); bolinha `{...eventPillDot, backgroundColor: color}` com `PierreIcon` `pixFilled 10` (pix) ou `starFilled 9` (marca), cor #FAFAFA; texto `Text "flex-1 text-xs font-medium" style={{color}} numberOfLines={1}` com `name.length <= 18 ? name : name.slice(0, 17) + '…'`. É aqui que entram as **cores de marca de `logos/mapeamento.json → assinaturas.cores`** (`netflix #E50914`, `claro #DA291C`, `youtube #FF0000`…).
- `resolveCalendarLogoUrl(entry)`: `entry.logoUrl || (entry.type === 'brand' && entry.name !== 'overflow' ? getCompanyLogoUrl(entry.name) : null)`.

### 5.6 Cartão da semana — `CalendarWeekCard` (L3245317; só alcançável com `activeView === 'week'`, que o segmento não oferece) [O]
`GlassContainer style={[weekDayCard, isToday && {borderWidth: 1, borderColor: 'rgba(255,255,255,0.12)'}]}` → `Text "text-[14px] font-medium text-foreground"` (dia ou "7 de out." se outro mês) · `Text "mt-1 text-xs text-muted-foreground"` (dia da semana abreviado, capitalizado) · eventos (`VStack "w-full mt-3"` com pílulas + `"+N"` `mt-2 text-xs font-medium text-muted-foreground`) ou `"Sem lançamentos"` (`mt-7 text-xs text-muted-foreground`).

### 5.7 Modo "Diária" — `VisionSubscriptionsCalendarDayContent({date, hidden, calendarData, dayGroups})` (L3248955)
- `focusedDateId` (state, sincroniza com `date`); `selected = getClosestCalendarDayGroup(groups, focusedDate)`; `previous = groups[idx − 1]`, `next = groups[idx + 1]`.
- Sem grupo: `VStack style=timelineList → Text "text-sm text-muted-foreground" "Sem pagamentos neste dia"`.
- Com grupo:
```
GestureDetector gesture={Pan().activeOffsetY([-18, 18]).failOffsetX([-48, 48]).onEnd(worklet Tsx3)}
 └─ Animated.View style={[timelineList, {opacity: transitionOpacity, transform: [{translateY: transitionY}]}]}   // ph 16 · pt 32 · gap 32
     ├─ previous && TouchableOpacity activeOpacity=0.82 testID="calendar-day-previous-{dateId}" onPress={() => goTo(previous.dateId, -1)}
     │     └─ CalendarCollapsedPreviousDay date events          // bolha 24 com borda #27272A, número 12/20, dia da semana 14/16, avatares 25
     ├─ CalendarTimelineDay date events hidden isSelected expanded totalAmountFormatted
     └─ next && TouchableOpacity activeOpacity=0.82 testID="calendar-day-next-{dateId}" onPress={() => goTo(next.dateId, 1)}
           └─ CalendarCollapsedNextDay date events              // divisor + coluna da data + linha 309×40 com cartões rgba(39,39,42,0.40)
```
- Gesto (fonte do worklet no bundle): `shouldGoNext = translationY < -48 || velocityY < -500`; `shouldGoPrevious = translationY > 48 || velocityY > 500`. Saída: `opacity → 0 (120 ms)`, `translateY → ∓72 (160 ms, Easing.in(cubic))`; ao terminar: `setFocusedDateId`, `translateY = ±72`, depois `translateY → 0 (260 ms, out(cubic))` e `opacity → 1 (220 ms, out(cubic))`. O toque em anterior/próximo faz só a entrada (`translateY` começa em ±72) [O].
- **`CalendarTimelineDay`** (L3247156):
```
VStack style=[timelineDayContainer (gap 24), expanded && timelineDayContainerExpanded]
 ├─ View style=[timelineDivider, {backgroundColor: '#27272A'}]                       // hairline
 └─ HStack style=timelineDayContent
     ├─ VStack style=timelineDateColumn (w 40, gap 4)
     │   ├─ Text "text-base text-center text-foreground" → "Ter"                      // weekday short capitalizado
     │   └─ View style=[timelineDateBubble, isSelected && {backgroundColor: '#FAFAFA'}]   // 40, borda #27272A
     │        └─ Text style=[timelineDateText (16/24/700), {color: isSelected ? '#18181B' : '#FAFAFA'}] → "07"   // padStart(2,'0')
     └─ VStack style=timelineDayMain (w 309, gap 16)
         ├─ CalendarTimelinePreview events hidden expanded
         │    expanded → VStack timelineEventsStack (gap 16) → events.map(e => <CalendarTimelineEventRow event={e} hidden/>)
         │    vazio    → Text "pt-7 text-sm text-muted-foreground" "Sem pagamentos neste dia"
         └─ expanded && events.length > 0 &&
              HStack "items-center justify-center" → Text "text-base text-muted-foreground" "Total a ser pago no dia: "
                                                     + PrivacyText "text-base font-bold text-foreground" {hidden ? 'R$ ••••' : totalAmountFormatted}
```
- **`CalendarTimelineEventRow({event, hidden, compact})`** (L3246735): `GlassContainer style={compact ? timelineCompactCard : timelineEventCard}` (r 24, minHeight 64, ph 16 · pv 12; regular ≈ #242424) → `CalendarDayEventAvatar` (`MerchantAvatar size 36, badgeSize 20` → `BankLogo` 22 [W3 §6], `useBankAsPrimary = !logoUrl && !!bankLogoUrl`, ícone `pixFilled` para pix) + `View timelineEventContent (ml 12)` → `Text "text-base font-medium text-foreground" numberOfLines=1` (título, Geist-Medium 14/21) + `PrivacyText "text-sm text-muted-foreground"` (`hidden ? 'R$ ••••' : amountFormatted`, 16 #a1a1aa).

### 5.8 Cores do calendário (dark)
| Uso | Cor |
|---|---|
| Fundo de célula, cartões, segmento (GlassContainer regular no Android) | `rgba(38,38,38,0.92)` ≈ **#242424** |
| Pílula ativa do segmento / texto ativo | **#FFFFFF** / **#09090B** |
| Texto inativo do segmento | #FAFAFA |
| Rótulo do dia da semana | #A1A1AA, 16 |
| Número do dia | #fafafa, Geist-Medium 14 |
| Bolha de data (borda / ativa) | #27272A / fundo #FAFAFA com texto #18181B |
| Divisor | #27272A (hairline) |
| Cartão compacto do próximo dia | `rgba(39,39,42,0.40)` |
| Pílula de evento | `accentColor` ou cor da marca, fundo `+ '40'`; padrão **#A855F7** |
| Hoje (semana) | borda `rgba(255,255,255,0.12)` |

---

## 6. Tela do dia — `VisionSubscriptionsDayScreen` (rota `/(protected)/vision/subscriptions-day`) [O]

Módulo M10438, função **L3322662**; hook `useVisionSubscriptionsDayScreen` (L3323415). Existe também `./sandbox/subscriptions-day.tsx`. **Não achei nenhum `router.push` para essa rota no bundle**: no fluxo atual, tocar num dia abre o modo "Diária" dentro do próprio calendário (§5.7). Esta tela é uma timeline **por hora** (variante mais antiga ou alternativa) [O ausência de navegação; I interpretação].

```
View "flex-1" onLayout={handleListLayout}
 ├─ HStack "items-center justify-between px-5 pb-3" style={{paddingTop: insets.top + k}}
 │   └─ TouchableOpacity activeOpacity=0.7 onPress={router.back}
 │       └─ GlassContainer regular {44×44, r22} → PierreIcon arrowLeft 22 #FAFAFA
 ├─ SectionList ref sections keyExtractor stickySectionHeadersEnabled showsVerticalScrollIndicator={false}
 │      initialNumToRender={2} onScroll={handleScroll} onScrollToIndexFailed
 │      contentContainerStyle={{paddingTop: 20, paddingBottom: insets.bottom + k}}
 │      ListHeaderComponent = VStack "px-4 pt-6"
 │           ├─ Text "text-[52px] font-bold leading-[56px] text-foreground" → formatCalendarMonthLong(selectedDate)   // "Outubro"
 │           └─ HStack "mt-1 items-center gap-1"
 │                ├─ Text "text-[17px] text-muted-foreground" → "Total"
 │                └─ PrivacyText "text-[17px] font-bold text-foreground" value={hidden ? 'R$ ••••' : data?.totalAmountFormatted ?? 'R$ 0,00'}
 │      renderItem = ({item}) => <CalendarDayTimeline dateId={item.dateId} hidden timedEvents={timedEventsByDate}/>
 └─ nextFloatingEvent && View pointerEvents="box-none"
        style={{position: 'absolute', left: 72, right: 16, bottom: insets.bottom + k, transform: [{scale: 0.94}]}}
        └─ CalendarDayFloatingEventButton entry={ev.event.entry} hidden hourLabel={`${pad2(ev.event.startHour)}:00`}
                                         overflowCount={ev.overflowCount} onPress={handleScrollToFloatingEvent}
```
- Seções `{title, weekday, dateId, data: [{dateId}]}`; `scrollToLocation({sectionIndex, itemIndex: 0, animated: true, viewOffset: 20})`; erro do calendário → toast `'Não foi possível carregar os pagamentos do calendário'`.
- `CalendarDayTimeline` (L3245628) usa `getCalendarTimedEventsByDate` (eventos com `startHour` e `durationHours = max(1, durationHours)`, ordenados por hora e nome), 24 linhas `'00:00'…'23:00'` (`dayTimeColumn` 52 + `dayTimelineRow` minHeight 40, divisores de 1 px), altura total `CALENDAR_DAY_TIMELINE_HEIGHT = 1024`, margem do topo 16; cada evento vira um `CalendarDayEventChip` (`minHeight 52, r14`, glass + gradiente, avatar 36 com selo 18 em `right −3, bottom −2`).
- `k` é uma constante do módulo que o decompilador não materializou [lacuna].

---

## 7. Adicionar assinatura — `VisionAddSubscriptionScreen` (rota `/(protected)/vision/add-subscription`) [O]

Módulos M10232 (rota), M10234 (tela, **L3228300**), M10235/M10236 (busca), M10237 (`useVisionAddSubscriptionScreen`), M10238 (cópia/testIDs), M10239 (`TransactionSearchGroupRow`), M10240 (skeleton), M10241 (imagem `no-widget.png` 1024²).
Entrada: botão "Adicionar assinatura" da tela (§4.4) → `router.push('/vision/add-subscription')`.

### 7.1 Fluxo e dados
- **Não há cadastro manual de assinatura.** O usuário busca transações e marca grupos como recorrentes:
  - Busca: `useVisionTransactionSearchGroups(search)` → `GET /api/mobile/vision/transactions/search-groups?search={termo}`; queryKey `['vision-transaction-search','groups', termo]`, `retry: 1`. O termo passa por debounce [I: há uma constante 400 ms no componente].
  - Grupo: `{id, name, logoUrl, bankName, bankLogoUrl, amountFormatted, paymentCount, transactionIds[], category, categoryEmoji, iconColor}`.
  - Confirmar (`handleMarkAsSubscription`): se há seleção e nada pendente → `track('add_subscription_tapped', {payment_count: Σ paymentCount})` → `Promise.all(selected.flatMap(g => g.transactionIds).map(id => updateVisionTransaction.mutateAsync({transactionId: id, isSubscription: true})))` → **`PATCH /api/mobile/vision/transactions/detail`** com `{transactionId, isSubscription: true}` (M2164 `updateTransaction`, L755105) → `queryClient.refetchQueries({queryKey: visionSubscriptionsScreenKeys.all, type: 'active'})` → `router.back()`. Falha parcial → `console.warn('[add-subscription] some transactions failed to mark as subscription')`.
  - Fechar → `track('add_subscription_dismissed')` + `router.back()`.
- Conclusão: **a lista de assinaturas é derivada no servidor** das transações marcadas `isSubscription` (e, pelo texto do catálogo "Lista de assinaturas recorrentes detectadas", também de detecção automática) [I].

### 7.2 Textos (`ADD_SUBSCRIPTION_COPY`)
`headerTitle: 'Assinaturas'` · `pageTitle: 'Encontre uma transação recorrente para definir como assinatura'` · `searchPlaceholder: 'Buscar transação'` · `noResults: 'Não encontramos transações para o termo buscado'` · `confirmButton: 'Marcar como assinatura'` · loading `'Marcando...'` · erro `'Não foi possível buscar as transações.'` · acessibilidade `'Fechar'`, `'Limpar busca'` · linha `'1 pagamento'` / `'{n} pagamentos'`, categoria padrão `'Outros'`.
TestIDs: `vision-add-subscription-screen`, `add-subscription-search-input`, `add-subscription-clear-search-button`, `add-subscription-close-button`, `add-subscription-confirm-button`, `add-subscription-results…`, `add-subscription-group-{id}`.

### 7.3 Layout
| Estilo | Valor |
|---|---|
| `header` | `{flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between', paddingHorizontal: 18, paddingBottom: 16}` + `paddingTop: insets.top + …` |
| botão fechar | `GlassContainer regular {44×44, r22}` → `PierreIcon navigationClose 18 #FAFAFA`, `hitSlop 8`; `headerSpacer {44×44}` do outro lado |
| título do header | `Text "text-xl font-normal text-foreground"` → "Assinaturas" (Geist-Regular 17,5/24,5) |
| `pageTitle` | `{marginTop: 32}` + `Text "px-5 text-2xl font-normal text-foreground"` (Geist-Regular 21/28, ph 17,5) |
| `searchRow` | `{marginTop: 32, paddingHorizontal: 16}` |
| `searchContainer` | `{height: 50, paddingHorizontal: 16, borderWidth: hairline?, borderRadius: 25, flexDirection: 'row', alignItems: 'center', gap: 12}` + dark `{borderColor: '#3F3F46'}` (light `#D4D4D8` + fundo `rgba(255,255,255,0.84)`) |
| ícone / input | `PierreIcon formSearch 20 #A1A1AA`; `TextInput {flex: 1, fontSize: 17, padding: 0, color: '#FAFAFA'}`, placeholder `#A1A1AA`, `returnKeyType="search"`, `autoCorrect={false}`; botão limpar `navigationClose 18` (`hitSlop 12`) |
| lista | `FlashList drawDistance={900} maxItemsInRecyclePool={24} keyboardShouldPersistTaps="handled" keyboardDismissMode="on-drag"`, `contentContainerStyle {paddingTop: 16, paddingBottom: footer + 96 [I]}` |
| carregando | 8 × `TransactionSearchGroupRowSkeleton` (avatar 55 r28; barras 55 %×16 r8, 40 %×14 r7, 30 %; checkbox 20 r6; cor `rgba(255,255,255,0.08)`) |
| vazio | imagem `no-widget.png` `{144×144, marginBottom 16}` + `Text "text-center text-base text-muted-foreground"` (`paddingTop 48`) |
| erro | `paddingTop 64` + "Não foi possível buscar as transações." |
| `footer` | `{position: 'absolute', left: 0, right: 0, bottom: 0, paddingHorizontal: 20, paddingTop: 12}` + `paddingBottom: insets.bottom + 10 [I]` → `PierreButton variant="primary" size="xlarge" fullWidth disabled={!selected.length} disabledTypeBackground="solid" isLoading={isMarking} loadingText="Marcando..."` → "Marcar como assinatura" (`bg-primary` #fafafa, texto #09090b, `px-8 py-5`, `text-lg font-semibold`) |

`TransactionSearchGroupRow({group, selected, onToggle, titleFontWeight = 'normal', disabled, testID})` (L3230413):
| Estilo | Valor |
|---|---|
| `container` | `{flexDirection: 'row', alignItems: 'center', gap: 12, paddingHorizontal: 16, paddingVertical: 12, borderRadius: 24}` |
| `content` | `{flex: 1, gap: 2}` |
| `title` | `{fontSize: 16, lineHeight: 24}` + `{color: '#FAFAFA', fontWeight}` (selecionado → `'500'` [I]) |
| `subtitle` | `{fontSize: 14, fontWeight: '400', lineHeight: 18}` cor #A1A1AA → `"{amountFormatted} • {n} pagamentos"` [I composição] |
| `checkbox` | `{width: 20, height: 20, borderRadius: 6, borderWidth: 1.5, borderColor: '#52525B'}`; selecionado `{backgroundColor: '#95FF48', borderColor: '#95FF48'}` + `PierreIcon feedbackCheck 12` |
| avatar | `MerchantAvatar` (logo/banco; `categoryFallback {name: category ?? 'Outros', color: iconColor, emoji: categoryEmoji}`; W3 §6) |

---

## 8. Logos: resumo e relação com `logos/mapeamento.json` [O]

| Lugar | Componente | Tamanho | Fonte da imagem | Selo do banco |
|---|---|---|---|---|
| Widget half (pilha) | `MerchantAvatar` | 40 | W3 §6: `matchMerchantLogo(name)` (asset local) → `logoUrl` → `getCompanyLogoUrl(name)` → inicial | não (sem `bankLogoUrl`) |
| Widget wide (pilha) | `LogoItem` próprio | 40 (imagem 52, recorte −6) | mesma ordem: asset local → `logoUrl ?? getCompanyLogoUrl(name)` (prefixa `getBaseUrl()` se começa com `/`) → inicial | não |
| Widget full / tela (linhas) | `SubscriptionRow` → `MerchantAvatar` | 48 | W3 §6 | sim, `BankLogo` 22 em `−5/−5` se `bankLogoUrl` |
| Calendário (célula do mês) | `CalendarAvatar` → `MerchantAvatar` | 24 | idem; sem `logoUrl` e com `bankLogoUrl` → **logo do banco como principal** | sim (badge padrão) |
| Calendário (dia) | `CalendarDayEventAvatar` → `MerchantAvatar` | 36 | idem | `badgeSize 20` |
| Pílula (semana) | `CalendarEventPill` | bolinha 14 | sem logo: cor `accentColor` → `getCompanyBrandColor(name)` → `#A855F7` | — |

Correspondência com o `mapeamento.json`:
- `merchantPorPalavraChave` + `aliases` = `matchMerchantLogo` (Claro → `lojas-e-apps/claro.png`, Netflix → `netflix.png`, YouTube → `youtube.png`; "YouTube Premium" casa a palavra `youtube`).
- `assinaturas.dominios` (195 chaves) = tabela de `getCompanyLogoUrl` (`netflix → netflix.com` …). No app ela vira `getBaseUrl() + '/api/logo?domain=' + domínio` (M6769, L2214281), **não** o favicon do Google. A nota do JSON que fala em `google.com/s2/favicons` está errada para este fluxo: esse endereço aparece no bundle só em `siteFaviconUri` (L2347437), de outra feature.
- `assinaturas.cores` (76 chaves) = `getCompanyBrandColor`, usado **só** na pílula do calendário (§5.5).
- Ordem de prioridade nas linhas: **asset local > `logoUrl` do servidor > logo por domínio > inicial**. Se a imagem falhar (`onError`), cai direto na inicial, sem tentar a próxima fonte.

---

## 9. Snippets TSX de referência (React Native puro)

Pressupõem os helpers já descritos: `MerchantAvatar` (W3 §6), `BankLogo` (W1 §5), `PrivacyText`/`PrivacyCurrencyText`/`AnimatedVisionCurrencyText`/`HIDDEN_VALUE`/`useHideValues` (W0 §10–11), `PierreIcon`. Cores dark.

### 9.1 Linha (`SubscriptionRow`)
```tsx
import React, { memo, useMemo } from 'react'
import { Text, TouchableOpacity, View, StyleSheet } from 'react-native'

const C = { fg: '#FAFAFA', muted: '#A1A1AA', pierre: '#D0FF00' }
const fmtDayMonth = new Intl.DateTimeFormat('pt-BR', { day: 'numeric', month: 'short' })
const todayBR = () => {
  const p = new Intl.DateTimeFormat('en-CA', { timeZone: 'America/Sao_Paulo', year: 'numeric', month: '2-digit', day: '2-digit' }).format(new Date())
  return brDate(p)
}
const brDate = (id: string) => { const [y, m, d] = id.split('-').map(Number); return new Date(Date.UTC(y, m - 1, d, 12) + 3 * 3600_000) }

type RowProps = {
  name: string; logoUrl?: string | null; bankName?: string | null; bankLogoUrl?: string | null
  amount: string; amountHidden?: boolean; nextRenewalDate?: string | null
  subtitle?: string | null; highlight?: string | null; onPress?: () => void
}
export const SubscriptionRow = memo(function SubscriptionRow(p: RowProps) {
  const auto = useMemo(() => {
    if (!p.nextRenewalDate) return null
    const d = brDate(p.nextRenewalDate)
    const days = Math.ceil((d.getTime() - todayBR().getTime()) / 86_400_000)
    const label = fmtDayMonth.format(d).replace('.', '')
    if (days <= 0) return `${label} • Pago`
    return days === 1 ? `Em 1 dia • ${label}` : `Em ${days} dias • ${label}`
  }, [p.nextRenewalDate])
  let line = p.subtitle ?? auto
  if (p.highlight) line = `${line ?? ''} • ${p.highlight}`.trim()
  return (
    <TouchableOpacity activeOpacity={0.85} onPress={p.onPress} disabled={!p.onPress} style={s.row}>
      <MerchantAvatar name={p.name} logoUrl={p.logoUrl} bankName={p.bankName} bankLogoUrl={p.bankLogoUrl}
        size={48} badgeSize={30} fallbackLabel={p.name.charAt(0).toUpperCase()} />
      <View style={s.textBlock}>
        <Text allowFontScaling={false} numberOfLines={1} style={s.title}>{p.name}</Text>
        {line ? (
          <Text allowFontScaling={false} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.58} ellipsizeMode="tail"
            style={[s.subtitle, { color: p.highlight ? C.pierre : C.muted }]}>{line}</Text>
        ) : null}
      </View>
      <PrivacyText allowFontScaling={false} numberOfLines={1} style={s.amount} value={p.amountHidden ? 'R$ ••••' : p.amount} />
    </TouchableOpacity>
  )
})
const s = StyleSheet.create({
  row: { width: '100%', minHeight: 60, paddingVertical: 8, flexDirection: 'row', alignItems: 'center', columnGap: 12 },
  textBlock: { flex: 1, minWidth: 0, height: 48, justifyContent: 'center', alignSelf: 'center' },
  title: { fontFamily: 'Geist-Bold', fontSize: 15.75, lineHeight: 24, color: C.fg, textTransform: 'capitalize' },
  subtitle: { fontFamily: 'Geist-Regular', fontSize: 14, marginTop: 2, lineHeight: 20 },
  amount: { fontFamily: 'Geist-Bold', fontSize: 15.75, lineHeight: 24.5, color: C.fg, minWidth: 92, textAlign: 'right', flexShrink: 0 },
})
```

### 9.2 Widget (half / wide / full)
```tsx
type Logo = { name: string; logoUrl?: string | null; nextRenewalDate?: string | null; bankName?: string | null; bankLogoUrl?: string | null; amount?: string }
type Props = { logos?: Logo[]; amount: string; subtitle?: string; totalCount?: number; size?: 'half' | 'wide' | 'full'
               isLoading?: boolean; hasError?: boolean; onPress?: () => void }

const CHEVRON = <PierreIcon name="chevronRight" size={16} color="#F4F4F5" />
const label = { fontFamily: 'Geist-Regular', fontSize: 14, lineHeight: 21, color: '#A1A1AA' } as const

function subtitleFor(logos: Logo[], subtitle?: string) {
  if (logos.length !== 1) return subtitle || 'Próxima em 5 dias'
  const next = logos[0]?.nextRenewalDate
  if (!next) return subtitle
  const days = Math.ceil((brDate(next).getTime() - todayBR().getTime()) / 86_400_000)
  return days <= 0 ? 'renovou recentemente' : days === 1 ? 'Próxima amanhã' : `Próxima em ${days} dias`
}

function StateIcon({ type }: { type: 'empty' | 'error' }) {
  return (
    <View style={{ width: 44, height: 44, borderRadius: 22, backgroundColor: '#27272A', alignItems: 'center', justifyContent: 'center' }}>
      <PierreIcon name={type === 'error' ? 'feedbackWarning' : 'timeClock'} size={18} color={type === 'error' ? '#EF4444' : '#A1A1AA'} />
    </View>
  )
}

function LogoStack({ logos, totalCount, merchantAvatar }: { logos: Logo[]; totalCount?: number; merchantAvatar?: boolean }) {
  const visible = logos.slice(0, 3)
  const overflow = (totalCount ?? logos.length) - 3
  return (
    <View testID="subscriptions-logo-stack" style={{ flexDirection: 'row', alignItems: 'center' }}>
      {visible.map((l, i) => (
        <View key={`${l.name}-${i}`} style={i ? { marginLeft: -14, zIndex: visible.length - i } : { zIndex: visible.length }}>
          {merchantAvatar
            ? <MerchantAvatar name={l.name} logoUrl={l.logoUrl} size={40} />
            : <LogoItem name={l.name} logoUrl={l.logoUrl} />}
        </View>
      ))}
      {overflow > 0 && (
        <View style={{ width: 40, height: 40, borderRadius: 20, marginLeft: -14, zIndex: 0, backgroundColor: '#27272A', alignItems: 'center', justifyContent: 'center' }}>
          <Text allowFontScaling={false} style={{ fontFamily: 'Geist-Bold', fontSize: 16, color: '#A1A1AA' }}>+{overflow}</Text>
        </View>
      )}
    </View>
  )
}

function LogoItem({ name, logoUrl }: { name: string; logoUrl?: string | null }) {
  const [failed, setFailed] = React.useState(false)
  const local = useMemo(() => matchMerchantLogo(name)?.logoSource ?? null, [name])
  const remote = useMemo(() => { const u = logoUrl ?? getCompanyLogoUrl(name); return u?.startsWith('/') ? getBaseUrl() + u : u }, [logoUrl, name])
  return (
    <View style={{ width: 40, height: 40, borderRadius: 20, backgroundColor: '#27272A' }}>
      <View style={[StyleSheet.absoluteFill, { borderRadius: 20, overflow: 'hidden' }]}>
        {(local || remote) && !failed
          ? <Image source={local ?? { uri: remote! }} resizeMode="cover" onError={() => setFailed(true)}
                   style={{ width: 52, height: 52, marginTop: -6, marginLeft: -6 }} />
          : <View style={{ width: 40, height: 40, alignItems: 'center', justifyContent: 'center' }}>
              <Text allowFontScaling={false} style={{ fontFamily: 'Geist-Bold', fontSize: 14, lineHeight: 21, color: '#A1A1AA' }}>{name.charAt(0).toUpperCase()}</Text>
            </View>}
      </View>
    </View>
  )
}

export function SubscriptionsWidget({ logos = [], amount, subtitle, totalCount, size = 'half', isLoading, hasError, onPress }: Props) {
  const hidden = useHideValues()
  const empty = logos.length === 0
  const full = size === 'full', wide = size === 'wide'
  const shown = hidden ? HIDDEN_VALUE : empty ? 'R$ 0,00' : amount
  const sub = subtitleFor(logos, subtitle)
  const cardStyle = full && !isLoading && (hasError || empty)
    ? { minHeight: 192, borderRadius: 24, padding: 20, overflow: 'hidden' as const }
    : full ? { minHeight: 344, borderRadius: 24, padding: 20, overflow: 'hidden' as const }
    : wide ? { minHeight: 176, borderRadius: 24, padding: 18, overflow: 'hidden' as const }
    : { height: 200, borderRadius: 24, padding: 16, overflow: 'hidden' as const }

  let content: React.ReactNode
  if (isLoading) content = <WidgetLoadingState variant={full ? 'content' : 'value'} />
  else if (full && (hasError || empty)) content = (
    <View style={{ flex: 1, justifyContent: 'center', gap: 21 }}>
      <View style={{ flexDirection: 'row', alignItems: 'center', gap: 14 }}>
        <StateIcon type={hasError ? 'error' : 'empty'} />
        <Text numberOfLines={1} style={{ fontFamily: 'Geist-Regular', fontSize: 15.75, lineHeight: 24.5, color: hasError ? '#EF4444' : '#A1A1AA' }}>
          {hasError ? 'Falha ao carregar assinaturas' : 'Não há assinaturas próximas'}</Text>
      </View>
    </View>)
  else if (full) content = (
    <View style={{ flex: 1, justifyContent: 'space-between' }}>
      <View style={{ gap: 14 }}>
        {logos.slice(0, 4).map((l, i) => (
          <SubscriptionRow key={`${l.name}-${i}`} name={l.name} logoUrl={l.logoUrl} bankName={l.bankName} bankLogoUrl={l.bankLogoUrl}
            amount={l.amount ?? ''} amountHidden={hidden} nextRenewalDate={l.nextRenewalDate} />))}
      </View>
    </View>)
  else if (wide) content = hasError ? (
    <View style={{ flex: 1, justifyContent: 'space-between' }}>
      <View style={{ flexDirection: 'row', alignItems: 'flex-start', justifyContent: 'space-between' }}>
        <Text style={label} numberOfLines={1}>Assinaturas esse mês</Text>{CHEVRON}</View>
      <View style={{ flexDirection: 'row', alignItems: 'center', gap: 14 }}>
        <StateIcon type="error" />
        <Text numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.8}
          style={{ fontFamily: 'Geist-Regular', fontSize: 15.75, lineHeight: 24.5, color: '#EF4444' }}>Falha ao carregar assinaturas</Text>
      </View>
    </View>) : (
    <View style={{ flex: 1, justifyContent: 'space-between' }}>
      <View style={{ gap: 21 }}>
        <View style={{ flexDirection: 'row', alignItems: 'flex-start', justifyContent: 'space-between' }}>
          <View style={{ minWidth: 0, flex: 1 }}>
            <Text style={label} numberOfLines={1}>Assinaturas esse mês</Text>
            <PrivacyText value={shown} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.72}
              style={{ fontFamily: 'Geist-ExtraBold', fontSize: 26.25, lineHeight: 31.5, color: '#FAFAFA' }} />
          </View>{CHEVRON}
        </View>
        {empty ? <StateIcon type="empty" /> : <LogoStack logos={logos} totalCount={totalCount} />}
      </View>
      <Text style={label} numberOfLines={1}>{sub}</Text>
    </View>)
  else content = (
    <View style={{ flex: 1, justifyContent: 'space-between' }}>
      <View style={{ flexDirection: 'row', alignItems: hasError ? 'flex-start' : undefined, justifyContent: 'space-between' }}>
        {hasError ? <StateIcon type="error" /> : empty ? <StateIcon type="empty" /> : <LogoStack logos={logos} totalCount={totalCount} merchantAvatar />}
        {CHEVRON}
      </View>
      <View style={{ gap: 3.5 }}>
        <Text style={label} numberOfLines={1}>Assinaturas</Text>
        {hasError
          ? <><Text style={{ fontFamily: 'Geist-ExtraBold', fontSize: 17.5, lineHeight: 24.5, color: '#FAFAFA' }}>R$ 0,00</Text>
              <Text style={{ ...label, color: '#EF4444' }} numberOfLines={1}>Falha ao carregar</Text></>
          : <><PrivacyCurrencyText valueCents={parseBRLToCents(empty ? 'R$ 0,00' : amount)} hidden={hidden}
                 fontSize={20} fontWeight="800" color="#FAFAFA" disableMinWidth compactOnOverflow />
              <Text style={label} numberOfLines={1}>{sub}</Text></>}
      </View>
    </View>)

  const card = (
    <WidgetGlassCard style={cardStyle}>
      <View testID="subscriptions-widget" style={{ flex: 1, gap: 16 }}>
        {full && (
          <View style={{ flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between' }}>
            <Text style={label} numberOfLines={1}>Assinaturas esse mês</Text>{CHEVRON}
          </View>)}
        {content}
      </View>
    </WidgetGlassCard>)
  return onPress ? <Pressable testID="subscriptions-widget-pressable" onPress={onPress} accessibilityRole="button">{card}</Pressable> : card
}
```

### 9.3 Cabeçalho da tela (voltar, olho, toggle lista/calendário)
```tsx
const GLASS = 'rgba(38, 38, 38, 0.92)'   // GlassContainer "regular" no Android (≈ #242424 sobre #09090b)
const circle = { width: 44, height: 44, borderRadius: 22, alignItems: 'center', justifyContent: 'center', backgroundColor: GLASS } as const

function Header({ hidden, onToggleHidden, view, setView, headerValue, headerOpacity, onBack }: any) {
  return (
    <View style={{ flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between', paddingHorizontal: 17.5, paddingVertical: 10.5 }}>
      <View style={{ flexDirection: 'row', alignItems: 'center', gap: 7 }}>
        <TouchableOpacity activeOpacity={0.7} onPress={onBack}>
          <View style={circle}><PierreIcon name="arrowLeft" size={22} color="#FAFAFA" /></View>
        </TouchableOpacity>
        <Animated.View pointerEvents="none" style={{ opacity: headerOpacity }}>
          <AnimatedVisionCurrencyText value={parseBRL(headerValue)} hidden={hidden} hiddenText="R$ ••••" durationMs={400}
            privacyFadeDurationMs={150} fontWeight="900" style={{ padding: 0, includeFontPadding: false, fontSize: 20, lineHeight: 24 }} />
        </Animated.View>
      </View>
      <View style={{ flexDirection: 'row', alignItems: 'center', gap: 7 }}>
        <TouchableOpacity activeOpacity={0.8} onPress={onToggleHidden}>
          <View style={circle}><PierreIcon name={hidden ? 'eyeOff' : 'eyeOn'} size={18} color="#FAFAFA" /></View>
        </TouchableOpacity>
        <View style={{ height: 44, borderRadius: 22, paddingHorizontal: 6, flexDirection: 'row', alignItems: 'center', backgroundColor: GLASS }}>
          <View pointerEvents="none" style={{ position: 'absolute', top: 3, left: 0, width: 38, height: 38, borderRadius: 19,
            backgroundColor: 'rgba(255, 255, 255, 0.06)', transform: [{ translateX: 6 + (view === 'calendar' ? 38 : 0) }] }} />
          {(['list', 'calendar'] as const).map(v => (
            <TouchableOpacity key={v} activeOpacity={0.82} onPress={() => setView(v)}
              style={{ width: 38, height: 38, borderRadius: 19, alignItems: 'center', justifyContent: 'center', zIndex: 1 }}>
              <PierreIcon name={v === 'list' ? 'contentTextTopicList' : 'dateCalendar'} size={18} color={view === v ? '#FAFAFA' : '#A1A1AA'} />
            </TouchableOpacity>))}
        </View>
      </View>
    </View>
  )
}
// Conteúdo da página "lista":
// <View style={{paddingHorizontal: 14, paddingTop: 21}}>
//   <Text style={{fontFamily:'Geist-Regular', fontSize:16, color:'#A1A1AA'}}>Compromisso Mensal</Text>
//   <AnimatedVisionCurrencyText … style={{marginTop:4, fontSize:44, lineHeight:48}} fontWeight="900" durationMs={400}/>
//   <View style={{flexDirection:'row', marginTop:21}}>
//     <Pressable style={{flexDirection:'row', alignItems:'center', gap:10.5, borderRadius:9999, backgroundColor:'#18181B', paddingHorizontal:21, paddingVertical:14}}>
//       <PierreIcon name="calcAdd" size={16} color="#FAFAFA"/>
//       <Text style={{fontFamily:'Geist-Medium', fontSize:15.75, lineHeight:24.5, color:'#FAFAFA'}}>Adicionar assinatura</Text>
//     </Pressable></View></View>
// <View style={{paddingHorizontal:16, paddingTop:40, gap:40}}>
//   <View style={{flexDirection:'row', alignItems:'center', justifyContent:'space-between'}}>
//     <Text style={{fontFamily:'Geist-Regular', fontSize:15.75, lineHeight:24.5, color:'#A1A1AA'}}>{section.title}</Text>
//     <PrivacyText value={hidden ? 'R$ ••••' : section.totalLabel} style={{fontFamily:'Geist-Regular', fontSize:16, color:'#A1A1AA'}}/></View>
//   <View style={{marginTop:7}}>{section.items.map(it => <SubscriptionRow key={it.id} … subtitle={it.subtitle} highlight={it.highlight} amount={it.amountFormatted}/>)}</View>
// </View>
```

---

## 10. Dúvidas abertas / limites
1. **"7 Out" e "Pago" vêm do servidor** (`item.subtitle` + `item.highlight`) [I forte]. O app não tem tabela de status (Pago/Pendente/Atrasado) para assinaturas; só o widget full gera "Pago" localmente quando `days <= 0`. Outros valores possíveis de `highlight` (ex.: "Pendente", "Atrasado") e se eles têm outra cor: **não há outra cor no código**. Qualquer `highlight` pinta o subtítulo de `text-pierre` (#d0ff00).
2. O formato local `formatBrazilDayMonthShort` depende do ICU do Hermes: o esperado é "7 de out" (Intl pt-BR remove só o "."). Só dá para confirmar rodando o app.
3. Ordenação e filtro do mês da lista são do servidor. O calendário só ordena por `dateId` e, dentro do dia, por `startHour`/`title`.
4. Peso do valor grande ("R$ 116,70"): `AnimatedVisionCurrencyText fontWeight '900'` sobre um `style` com `'700'`. A família final é decidida pelo componente nativo (W0 §11.3); no screenshot o peso parece Bold/Black.
5. Durações que o decompilador não materializou: `FadeInDown/FadeOutUp` do calendário e a constante `k` de padding da tela do dia.
6. A rota `subscriptions-day` não tem `push` no bundle; pode ser legado ou só sandbox.
7. `CalendarAvatar` com `name === 'overflow'` mostra "+2" fixo. Parece resto de código, já que a célula calcula o "+N" por conta própria.
8. `WideState` com `isError=false` nunca é chamado (o wide vazio usa o `StateIcon` dentro do layout normal).
9. `emptyState` e `animate` chegam ao `SubscriptionsWidget`, mas ele não usa nenhum dos dois.
10. No screenshot não aparecem insight, skeleton nem estado vazio, então as medidas desses estados vêm só do código.
