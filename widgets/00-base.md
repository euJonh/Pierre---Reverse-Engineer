# W0 — Fundação compartilhada dos widgets da home "Vision" (Pierre v4.0.6)

> Análise estática do bundle Hermes v98 (`_trabalho/out.js`) e do dex nativo (`classes3.dex`).
> **[O]** = observado no código (`Lnnnn` = linha de `out.js`; `Mnnnn` = id de módulo Metro); **[I]** = inferido.
> Unidades: px = dp do React Native. NativeWind com **rem = 14**.

---

## 1. Visão geral da cadeia de renderização

```
VisionSortableWidgetGrid (L2798179, M8820)
 └─ VisionWidgetItem (L2797756)              ← largura calculada aqui; altura vem do widget
     └─ Animated.View(layout spring, width, zIndex)
         ├─ VisionHomeMediaActivityProvider(active)
         │   └─ VisionJiggleWrapper(id, jigglePaused, currentSize, supportedSizes, isFixed, onResize*, scroll*)
         │       └─ VisionWidgetSlotRenderer (L2833494)
         │           └─ VisionWidgetRenderer (memo, L2833731, M8973)
         │               └─ VisionWidgetErrorBoundary(widgetId)  (M9097)
         │                   └─ VisionWidgetRendererInner (memo, L2833795)
         │                       └─ <XxxWidgetConnected size chartVariant? onPress emptyState animate/>
         │                           └─ Widget2x2 | WidgetWide | WidgetFull  (M8977/M8981/M8980)
         │                               └─ WidgetShell (M8978) → [GlassContainer] → [Pressable]
         └─ (composerOpen === true) Pressable absoluteFill  (intercepta toque)
```

Barrel de exports (M8976, L2835699): `Widget2x2, WidgetValueSlot, WidgetFull, WidgetWide, WidgetEmptyState, WidgetEmptyStateSlot, WidgetErrorState, WidgetLoadingState, WidgetGlassCard` [O].

---

## 2. VisionWidgetSlotRenderer / VisionWidgetRenderer / Inner

### 2.1 VisionWidgetSlotRenderer (L2833494) [O]
Props: `slot, size, onWidgetTap, onWidgetNavigationTransitionPressReady, animate = false`.
- `slot.kind === 'system'` → monta `widget = { ...slot.data, size }` (o `size` **efetivo** vindo do grid sobrescreve o do dado) e
  `onPress = onWidgetTap ? (x) => x ? onWidgetTap(slot.data.id, x) : onWidgetTap(slot.data.id) : undefined`.
  Renderiza `<VisionWidgetRenderer widget onPress onWidgetNavigationTransitionPressReady animate/>`.
- `slot.kind === 'empty'` → `null`. Qualquer outro `kind` → `undefined` (nada).
- Não repassa `isEditMode` nem `emptyStates` (ficam `undefined` nesse caminho) [O].

### 2.2 VisionWidgetRenderer (memo, L2833731) [O]
`<VisionWidgetErrorBoundary widgetId={widget.id}>{<VisionWidgetRendererInner {...props}/>}</…>` — todas as props são espalhadas (`copyDataProperties`).

### 2.3 VisionWidgetRendererInner (memo, L2833795) [O]
Props: `widget, onPress, onWidgetNavigationTransitionPressReady, isEditMode = false, emptyStates, animate = false`.
1. `singlePress = useSinglePress(onPress)` (M3015, `DEFAULT_SINGLE_PRESS_DEBOUNCE_MS`).
2. `handlePress = (!isEditMode && onPress) ? (arg) => { const s = useVisionWidgetOrderStore.getState(); if (s.isContextMenuOpen || s.shouldSuppressWidgetTap()) return; singlePress(arg) } : undefined`
   → o toque é **suprimido** enquanto o menu de contexto está aberto ou logo após long-press/edição.
3. `emptyState = emptyStates?.[widget.id]`.
4. Se `widget.id.startsWith('ai-')`: busca no store `aiWidgets.find(w => w.id === widget.id.slice(3))`; se não achar → `null`; se achar →
   `<AIVisionWidgetConnected id title widgetType size onPress animate fallbackData={aiWidget.cachedData}/>`.
5. Senão, `switch (widget.id)` (StringSwitchImm decodificado da tabela de salto da função #52548):

| `widget.id` | Componente | Props passadas |
|---|---|---|
| `leo-insight` | `LeoInsightWidgetConnected` | `animate` |
| `leo-agent` | `LeoAgentWidgetConnected` | `onPress, animate` |
| `cdb-position` | `CdbPositionWidgetConnected` | `size, onPress, emptyState, animate` |
| `cdb-performance` | `CdbPerformanceWidgetConnected` | `size, onPress, emptyState, animate` |
| `cdb-investment` | `CdbInvestmentWidgetConnected` | `size, onPress, onNavigationTransitionPressReady = (x)=>onWidgetNavigationTransitionPressReady('cdb-investment', x), emptyState, animate` |
| `subscriptions` | `SubscriptionsWidgetConnected` | `size, onPress, emptyState, animate` |
| `spending-by-category` | `SpendingByCategoryWidgetConnected` | idem |
| `recent-transactions` | `VisionRecentTransactionsWidgetConnected` | idem |
| `cards` | `CardsWidgetConnected` | idem |
| `patrimony` | `PatrimonyWidgetConnected` | idem |
| `installments` | `InstallmentsWidgetConnected` | idem |
| `available-limit` | `AvailableLimitWidgetConnected` | idem |
| `cash-flow` | `CashFlowWidgetConnected` | idem |
| `monthly-spending` | `VisionMonthlySpendingWidgetConnected` | `size, chartVariant: widget.chartVariant, onPress, emptyState, animate` |
| `account-balance` | `AccountBalanceWidgetConnected` | `size, onPress, emptyState, animate` |
| `wallet` | `WalletWidgetConnected` | `size, emptyState, animate` (**sem onPress**) |
| outro | `null` | — |

**Como chegam `size` e `chartVariant`** [O]: `size` = tamanho efetivo calculado pelo grid (`resolveEffectiveWidgetSize(id, widgetSizes, previewSize)`) e injetado em `widget` pelo SlotRenderer; `chartVariant` vem do próprio registro do widget (`slot.data.chartVariant`) e só é lido para `monthly-spending`.

---

## 3. Grid e item: larguras/alturas reais

### 3.1 VisionSortableWidgetGrid (L2798179) [O]
- `windowWidth = useWindowDimensions().width`
- `fullWidth = windowWidth − 32` (padding horizontal de 16 da tela, aplicado pelo pai)
- `GAP = 18` (constante do módulo)
- `halfWidth = (fullWidth − 18) / 2`
- Largura passada ao item: `effectiveSize === 'half' ? halfWidth : fullWidth` (**wide e full ocupam a largura toda**).
- Container estático: `View style=[{flexDirection:'row', flexWrap:'wrap'}, {gap: 18}]`.
- Fora do modo edição: filhos = `[itens fixos, itens ordenáveis]` direto no View.
- Em modo edição: `[itens fixos (se houver), <Sortable.Flex …>]` (react-native-sortables) com:
  `flexDirection:'row', flexWrap:'wrap', gap:18, customHandle:true, sortEnabled: activeResizeId === null, dragActivationDelay: 200, activeItemScale: 1.05, activeItemOpacity: 1, activeItemShadowOpacity: 0.4, inactiveItemOpacity: 0.65, hapticsEnabled: false, autoScrollEnabled: !!scrollableRef, autoScrollDirection:'vertical', autoScrollActivationOffset: 150, autoScrollMaxOverscroll: 9999, autoScrollMaxVelocity: 2500, autoScrollInterval: 0, animateScrollTo: false`.
- Analytics de reorder: `trackWidgetInteraction('reordered', {widget_id, widget_type: id.startsWith('ai-') ? 'ai' : 'system', from_position, to_position, source_screen:'home'})`.

### 3.2 VisionWidgetItem (L2797756) [O]
- `Animated.View` com
  - `layout = animateLayout ? LinearTransition.springify().damping(22).stiffness(220).mass(0.75) : undefined`
  - `style = { width, overflow: 'visible', zIndex: isActiveResize ? 999 : 0 }`
  - `testID = 'vision-sortable-item-' + id`
- Se `composerOpen === true`: sobrepõe `Pressable style=StyleSheet.absoluteFill accessible={false} onPress={() => onWidgetTap?.(id)}`.
- `onLayout` só é ligado se o widget é de mídia animada (`isVisionHomeAnimatedMediaWidget(id, size)`) → `onMediaLayout(id, {y, height})`.

### 3.3 Alturas (definidas nos componentes base, não no grid) [O]
| Tamanho | Largura | Altura | Raio |
|---|---|---|---|
| half (`Widget2x2`) | `(W−32−18)/2` | **200** | 24 |
| wide (`WidgetWide`) | `W−32` | **170** | 24 |
| full (`WidgetFull`) | `W−32` | conteúdo (sem altura fixa) | 24 |

Ex.: W = 393 → half = 171,5 × 200; wide = 361 × 170.
Utilitário relacionado (M8992, L2841386): `getHalfWidgetContentWidth(W) = max((W − 32 − 18)/2 − 32, 0)` — largura útil do texto no half, descontando 2×16 de padding (aproximação; o padding real `p-4` é 14).

---

## 4. WidgetShell e hooks (M8978, L2836290–2836560) [O]

### 4.1 `useWidgetGlass(bg = 'glass')`
`return isLiquidGlassAvailable() && bg === 'glass'` — `isLiquidGlassAvailable` vem de `expo-glass-effect` (M1376: `GlassView, GlassContainer, isLiquidGlassAvailable, isGlassEffectAPIAvailable`).
**No Android isso é sempre `false`** [I: a API de Liquid Glass só existe no iOS 26+], então no Android todos os widgets usam o caminho **opaco** `bg-surface` (#18181b).

### 4.2 `useWidgetPress(onPress)`
Callback memoizado (cache por `onPress`):
```ts
() => { Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light); onPress?.(); }
```
O háptico **Light** dispara antes do `onPress`. É o único "press feedback" dos widgets: **não há scale, opacity nem ripple** no toque normal (o `Pressable` não recebe `style` nem `android_ripple`) [O].

### 4.3 `WidgetShell({ inner, useGlass, glassStyle, widgetRadius, containerStyle, onPress, handlePress, testID })`
```
content = useGlass
  ? <GlassContainer glassEffectStyle="clear" style={[glassStyle, {borderRadius: widgetRadius}, containerStyle]}>{inner}</GlassContainer>
  : inner
return onPress ? <Pressable onPress={handlePress} testID={`${testID}-pressable`}>{content}</Pressable> : content
```
- Sem `onPress` → não há Pressable (widget não clicável, ex.: `wallet`).
- `Pressable` = `react-native` (M50).

### 4.4 Feedback de long-press / edição (fora do shell, para referência)
O menu de contexto e o modo edição vivem no `VisionJiggleWrapper` (L2830194) — já documentados no design system. Valores observados aqui:
- "Pulse" de widget recém-adicionado (`startPulse`, 400 ms após montar): `scale = withSequence(withTiming(1.08,{180, Easing.out(ease)}), withTiming(0.97,{160, inOut(ease)}), withTiming(1.04,{140, out(ease)}), withTiming(1,{140, inOut(ease)}))`; após 700 ms chama `clearLastAdded()` [O].
- Opacidade durante resize: `1 − min(hypot(dx,dy)/60, 1) × 0.5` [O].
- Hápticos: `impactAsync(Light)` e `impactAsync(Heavy)` em pontos do gesto [O].

---

## 5. Widget2x2 — tamanho *half* (M8977, L2835896) [O]

### 5.1 Props e defaults
| Prop | Default | Uso |
|---|---|---|
| `visual` | — | nó do topo (ícone, gráfico, logo…) |
| `label` | — | texto do rótulo (rodapé) |
| `value` | — | string do valor (vai para `PrivacyText`) |
| `valueSlot` | — | nó que substitui o valor (ex.: `PrivacyCurrencyText`) |
| `subtitle` | — | linha abaixo do valor |
| `valueMinimumFontScale` | **0.6** | autoshrink do valor |
| `onPress` | — | torna clicável |
| `showChevron` | **true** | chevron no canto superior direito |
| `visualPlacement` | **'top'** | `'top'` ou `'middle'` |
| `testID` | **'widget-2x2'** | |
| `topClassName` | — | className do Box do topo |
| `bottomClassName` | — | repassado ao `WidgetValueSlot` |
| `containerStyle` | — | estilo extra do container |
| `bg` | **'glass'** | `'glass'` ou outro (ex.: `'surface'`) |

### 5.2 Árvore exata
```
WidgetShell(useGlass, glassStyle={borderRadius:24,height:200}, widgetRadius=24, containerStyle, onPress, handlePress, testID)
 └─ VStack className="justify-between p-4 gap-2 bg-surface"   (sem bg-surface se glass)
            style = glass ? {flex:1} : [{height:200}, {borderRadius:24}, containerStyle]
            testID
     ├─ TOPO (visualPlacement === 'top'):
     │   Box className={topClassName} pointerEvents="none"
     │    └─ showChevron
     │        ? HStack "items-start justify-between"
     │            ├─ Box "flex-1" → {visual}
     │            └─ Box testID="{testID}-chevron" "p-0.5" → PierreIcon chevronRight size=16 color=#71717a
     │        : {visual}
     │   TOPO (visualPlacement === 'middle'):
     │   <>
     │     {showChevron && HStack "justify-end" pointerEvents="none" → Box "p-0.5" (chevron 16 #71717a)}
     │     Box collapsable={false} pointerEvents="none" style={{alignSelf:'stretch', width:'100%'}} → {visual}
     │   </>
     └─ RODAPÉ: WidgetValueSlot {label, value, valueSlot, valueMinimumFontScale, subtitle, testID, bottomClassName}
```

### 5.3 Medidas (Android, caminho opaco)
| Item | Valor |
|---|---|
| Altura | 200 |
| Largura | `(W − 50)/2` |
| Raio | 24 (aplicado via `style`, não classe) |
| Padding | `p-4` = **14** em todos os lados |
| Gap vertical topo↔rodapé | `gap-2` = **7** (com `justify-between`: topo colado em cima, rodapé embaixo) |
| Fundo | `bg-surface` = `hsl(240 6% 10%)` = **#18181b** |
| Borda / sombra | **nenhuma** |
| Chevron | caixa `p-0.5` (1,75 de padding) → 19,5×19,5; ícone 16, cor `#71717a` (zinc-500) |

---

## 6. WidgetValueSlot (M8979, L2836601) [O]

Props: `label, labelMinimumFontScale = 0.7, value, valueSlot, valueMinimumFontScale = 0.6, subtitle, subtitleMinimumFontScale = 0.7, testID, bottomClassName = 'gap-0.5'`.

```
VStack className={bottomClassName ?? 'gap-0.5'}            // gap 1,75
 ├─ Text  "text-base text-muted-foreground"  numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.7 allowFontScaling=false → {label}
 ├─ valueSlot ?? PrivacyText "text-xl font-extrabold text-foreground" testID="{testID}-value"
 │                numberOfLines=1 adjustsFontSizeToFit minimumFontScale={valueMinimumFontScale} allowFontScaling=false value={value ?? ''}
 └─ PrivacyText "text-base text-muted-foreground" testID="{testID}-subtitle" numberOfLines=1
                adjustsFontSizeToFit minimumFontScale=0.7 allowFontScaling=false value={subtitle ?? ''}   // sempre renderizado
```
Atenção: a linha de subtítulo **sempre existe**; com `''` ela ocupa a altura de uma linha de 21. Isso afeta o alinhamento vertical [O].

| Texto | Classe | Fonte | Size/LineHeight | Cor (dark) |
|---|---|---|---|---|
| label | `text-base text-muted-foreground` | Geist-Regular | 14 / 21 | #a1a1aa |
| valor | `text-xl font-extrabold text-foreground` | Geist-ExtraBold | 17,5 / 24,5 | #fafafa |
| subtítulo | `text-base text-muted-foreground` | Geist-Regular | 14 / 21 | #a1a1aa |

---

## 7. WidgetWide — tamanho *wide* (M8981, L2837327) [O]

Props: `visual, label, value, subtitle, valueMinimumFontScale = 0.75, onPress, showChevron = true, testID = 'widget-wide', containerStyle, bg = 'glass'` (sem `valueSlot`).

```
WidgetShell(useGlass, glassStyle={borderRadius:24,height:170}, widgetRadius=24 [I], containerStyle, onPress, handlePress, testID)
 └─ VStack className="justify-between p-4 bg-surface"  style = glass ? {flex:1} : [{height:170}, {borderRadius:24}, containerStyle]  testID
     ├─ HStack "items-start justify-between" pointerEvents="none"
     │   ├─ VStack "flex-1 pr-3"                                  // paddingRight 10,5
     │   │   ├─ Text "text-lg text-muted-foreground" numberOfLines=1 adjustsFontSizeToFit minimumFontScale=0.8 allowFontScaling=false → {label}
     │   │   ├─ PrivacyText "text-2xl font-extrabold text-foreground" testID="{testID}-value" numberOfLines=1
     │   │   │              adjustsFontSizeToFit minimumFontScale={0.75} allowFontScaling=false value={value ?? ''}
     │   │   └─ subtitle ? PrivacyText "text-base text-muted-foreground" testID="{testID}-subtitle" numberOfLines=1
     │   │                 adjustsFontSizeToFit minimumFontScale=0.8 allowFontScaling=false value={subtitle} : null
     │   └─ showChevron ? Box testID="{testID}-chevron" "p-0.5" → PierreIcon chevronRight 16 #71717a : null
     └─ visual ? Box pointerEvents="none" → {visual} : null
```
Diferenças em relação ao half: valor maior (`text-2xl` 21/28), rótulo `text-lg` (15,75/24,5), **sem gap** entre label, valor e subtítulo (VStack sem gap), visual **embaixo**, subtítulo só se truthy, altura 170.
O `widgetRadius` do wide vem de uma constante do módulo que o decompilador não materializou; o `glassStyle` do mesmo módulo usa `borderRadius: 24` [I alto].

| Texto | Classe | Fonte | Size/LH | Cor |
|---|---|---|---|---|
| label | `text-lg text-muted-foreground` | Geist-Regular | 15,75 / 24,5 | #a1a1aa |
| valor | `text-2xl font-extrabold text-foreground` | Geist-ExtraBold | 21 / 28 | #fafafa |
| subtítulo | `text-base text-muted-foreground` | Geist-Regular | 14 / 21 | #a1a1aa |

---

## 8. WidgetFull — tamanho *full* (M8980, L2836921) [O]

Props: `title, children, onPress, showChevron = true, headerRight, testID = 'widget-full', bg = 'glass'`.
Não usa `WidgetShell`: reimplementa a mesma lógica inline. `LIQUID_GLASS = isLiquidGlassAvailable()` é avaliado **uma vez**, no load do módulo.

```
handlePress = () => { Haptics.impactAsync(Light); onPress?.() }
useGlass = LIQUID_GLASS && bg === 'glass'
[Pressable onPress={handlePress} testID="{testID}-pressable"]            // só se onPress
 └─ [GlassContainer glassEffectStyle="clear" style=[{borderRadius:24},{borderRadius:24}]]   // só se glass
     └─ VStack className="p-4 gap-3 bg-surface" style={glass ? undefined : {borderRadius:24}} testID
         ├─ HStack "items-center justify-between"
         │   ├─ Text "text-lg text-muted-foreground" allowFontScaling=false → {title}
         │   └─ HStack "items-center gap-2"
         │       ├─ {headerRight}
         │       └─ showChevron && Box testID="{testID}-chevron" "p-0.5" → PierreIcon chevronRight 16 #71717a
         └─ {children}
```
- Altura: **livre** (definida pelo conteúdo). Padding 14; gap entre header e conteúdo `gap-3` = **10,5**; gap no grupo direito `gap-2` = 7.
- Título: Geist-Regular 15,75/24,5 #a1a1aa (sem `numberOfLines`).
- O chevron aparece mesmo sem `onPress`, a menos que `showChevron` seja `false` (o estado de carregamento do AI widget passa `showChevron:false`).

---

## 9. Estados: vazio, erro, carregando, glass card [O]

### 9.1 WidgetEmptyState (M8982, L2837784)
Props: `message = 'Sem dados disponíveis'`, `compact = false`.
```
VStack className="items-center justify-center {compact ? 'py-1' : 'py-4'} gap-2" testID="widget-empty-state"
 ├─ PierreIcon name="wifiOff" size={compact ? 18 : 22} color="#71717a"
 └─ Text allowFontScaling=false className="{compact ? 'text-sm' : 'text-base'} text-muted-foreground text-center" → {message}
```
py-4 = 14 / py-1 = 3,5; gap 7. Texto: normal 14/21; compacto **16** (sem lineHeight, porque `text-sm` = 16 neste projeto); cor #a1a1aa.

### 9.2 WidgetEmptyStateSlot (M8983, L2837952)
`emptyState ? <>{emptyState}</> : <WidgetEmptyState message compact/>`.
Na prática `emptyState` é sempre `undefined` na home: o único chamador de `VisionWidgetRenderer` (o SlotRenderer) não passa `emptyStates` [O].

### 9.3 WidgetErrorState (M8984, L2838061)
Props: `message = 'Falha ao sincronizar dados'`, `compact = false`.
Mesmo layout do vazio, com `PierreIcon name="feedbackWarning" size={compact?18:22} color="#EF4444"` e texto `… text-destructive text-center` (#ef4444 via `--destructive`). testID `widget-error-state`.
O `VisionWidgetErrorBoundary` (M9097) renderiza um estado com `{compact: true}` quando o widget lança erro [O; pelo import do barrel M8976, é o `WidgetErrorState` — I alto]. Ele também loga `"[VisionWidgetErrorBoundary] widget render crashed: "`, envia evento com `user impact: 'widget degraded to error state; rest of the home survived'` e reseta `hasError` no `componentDidUpdate`.

### 9.4 WidgetLoadingState (M8985, L2838251): skeleton com "shimmer" de opacidade
Props: `variant = 'content' | 'value' | 'visual'` (default `'content'`).
Animação (`useShimmerStyle`): `opacity = useSharedValue(0.3)`; no mount `opacity.value = withRepeat(withTiming(0.7, {duration: 800, easing: Easing.inOut(Easing.ease)}), -1, true)`. A opacidade pulsa 0,3 ↔ 0,7 (800 ms em cada sentido). Ela é aplicada ao **container** (`Animated.View`, testID `widget-loading-state`), não a cada barra. Não há gradiente deslizante.

| Variante | Filhos (todos `backgroundColor:'#71717a'`) |
|---|---|
| `content` | `{height:44,width:'100%',borderRadius:12,marginBottom:8}` ×2 + `{height:44,width:'70%',borderRadius:12}` |
| `value` | `{height:24,width:'60%',borderRadius:12}` + `{height:14,width:'40%',borderRadius:7,marginTop:6}` |
| `visual` | `{height:64,width:64,borderRadius:32}` |

### 9.5 WidgetGlassCard (M8986, L2838586)
Props: `children, bg = 'glass', style, testID`.
Se `isLiquidGlassAvailable() && bg === 'glass'` → `GlassContainer glassEffectStyle="clear" style={style}`; senão `VStack className="bg-surface" style testID`. Não tem raio nem padding próprios (vêm do `style`).

---

## 10. "Ocultar dados": privacidade global [O]

### 10.1 Store (M4428, L1789247)
```ts
export const HIDDEN_VALUE = 'R$ ••••';               // U+2022 ×4, com espaço normal após "R$"
export const usePrivacyStore = create(persist(
  (set) => ({
    hideValues: false,
    _hasHydrated: false,
    toggleHideValues: () => set((s) => ({ hideValues: !s.hideValues })),
    _setHasHydrated: () => set({ _hasHydrated: true }),
  }),
  { name: 'pierre-privacy', storage: createJSONStorage(() => AsyncStorage /*[I]*/),
    partialize: (s) => ({ hideValues: s.hideValues }),
    onRehydrateStorage: () => (s) => s?._setHasHydrated() }
));
export const useHideValues = () => usePrivacyStore((s) => !s._hasHydrated || s.hideValues);
```
Detalhe importante: **enquanto o store não hidratou, `useHideValues()` retorna `true`**. Os valores começam mascarados e só aparecem depois da leitura do storage, o que evita um "flash" do saldo real [O].
Strings da UI relacionadas: `'Ocultar valores'`, `'Ocultar dados'`; testIDs `account-balance-privacy-toggle`, `vision-transactions-privacy-toggle`, `vision-invoice-detail-privacy-toggle` [O].

### 10.2 PrivacyText (M4432, L1790085)
`memo(({ value, className, style, testID, numberOfLines, adjustsFontSizeToFit, minimumFontScale, allowFontScaling }) => …)`
- **Não mascara sozinho**: quem chama já passa `value` mascarado (ex.: `hidden ? HIDDEN_VALUE : texto`). O componente só faz a **transição** [O].
- `displayValue` (state) começa com `value`; um `ref` guarda o último valor.
- Quando `value` muda: `opacity.value = withTiming(0, {duration: 150}, () => { runOnJS(setDisplayValue)(next); opacity.value = withTiming(1, {duration: 150}) })`. `FADE_DURATION = 150`, com o easing padrão do Reanimated [O].
- Render: `Animated.createAnimatedComponent(Text)` (Text do gluestack, M2262) com `style={[style, {opacity}]}` e `children = displayValue`.

### 10.3 Máscaras usadas
| Contexto | Máscara |
|---|---|
| Valores BRL em geral (`HIDDEN_VALUE`) | `R$ ••••` |
| `buildCurrencyHiddenText(code, symbol)` (L2720190) | `symbol.trim() + ' ••••'`; senão o símbolo de `Intl.NumberFormat('pt-BR',{style:'currency',currency:code}).formatToParts(0)` + `' ••••'`; fallback `HIDDEN_VALUE` |
| Cartões (strings no bundle) | `•••• •••• •••• 1234`, `•••• 1234` |

---

## 11. Texto de moeda animado

### 11.1 AnimatedVisionCurrencyText (M8528, L2720379) [O]
Wrapper fino sobre o componente nativo `PierreAnimatedCurrencyText` (M4584):
```ts
({ value, hidden = false, hiddenText, currencyCode, currencySymbol, ...rest }) => {
  const mode = useThemeStore(s => s.mode);
  const textColor = mode === 'light' ? '#000000' : '#FFFFFF';     // dark → branco puro
  const hiddenTxt = hiddenText ?? buildCurrencyHiddenText(currencyCode, currencySymbol);
  const num = parseVisionCurrencyValue(value);                       // aceita number ou string
  return <PierreAnimatedCurrencyText {...rest} currencyCode={currencyCode} currencySymbol={currencySymbol}
           hidden={hidden} hiddenText={hiddenTxt} textColor={textColor} value={num}/>;
}
```
`parseVisionCurrencyValue(v)`: se for string, remove tudo exceto `[\d,.-]`, remove os `.` (milhar), troca `,` por `.` e aplica `Number()`, com 0 se não for finito. Se for número, retorna ele mesmo quando finito, senão 0.

Exemplos de uso: `{durationMs:400, fontSize:16, fontWeight:'700', privacyFadeDurationMs:150}` (L2713493), `{durationMs:400, fontSize:40, fontWeight:'900', lineHeight:44}` (L2745912), `{durationMs:600, fontSize:30, fontWeight:'bold', lineHeight:30}` (L2773499).

### 11.2 PierreAnimatedCurrencyText: wrapper JS (L1836680) [O]
Defaults: `hiddenText 'R$ ••••'`, `currencyCode 'BRL'`, `localeIdentifier 'pt_BR'`, `durationMs 1800`, `privacyFadeDurationMs 150`, `minimumFontScale 0.75`, `fontSize 16`, `lineHeight = fontSize × 1.18`, `textColor '#FFFFFF'`, `textAlign 'left'`; `animateFromZero` default `true` no nativo.
O JS calcula a string formatada (mesmo algoritmo do nativo) só para reservar largura:
`minWidth = ceil(max(hiddenText.length, formatted.length) × fontSize × 0.62 + 6)` (omitido se `disableMinWidth`) e `height = lineHeight`.

### 11.3 Implementação nativa: `io.cloudwalk.pierre.charts.PierreAnimatedCurrencyTextView` (Kotlin/ExpoView, `classes3.dex`) [O]
- Um `TextView` interno: `includeFontPadding=false`, `singleLine`, sem ellipsize, cor padrão `-1` (branco).
- Auto-size uniforme: `min = max(round(clamp(minimumFontScale, 0.1, 1) × fontSize × density), 1)` px, `max = fontSize × density`, passo de 1 px.
- **Contagem**: `ValueAnimator.ofFloat(0,1)` com `duration = durationMs` e interpolador **`MoneyCounterInterpolator`**, o *smootherstep* de Perlin:
  `f(t) = t³ · (t · (6t − 15) + 10)`, com `t` limitado a [0,1].
  `display = from + (target − from) · f(t)`. `from = 0` na **primeira** animação se `animateFromZero`; depois, o valor exibido naquele momento.
  Não anima (aplica direto) se `hidden`, se `durationMs ≤ 0` ou se `value == 0`. No `onDetachedFromWindow` pula para o alvo.
- **Formatação** (`formatCurrency`): com `currencySymbol` não vazio usa `formatWithSymbol(v, symbol)`; com `currencyCode == 'BRL'` (sem diferenciar maiúsculas) usa `formatWithSymbol(v, "R$")`; senão usa `NumberFormat.getCurrencyInstance(Locale(localeIdentifier.replace('_','-')))` com `Currency(code)`.
  `formatWithSymbol`: `cents = rint(|v| × 100)`; milhar com `.`; centavos com 2 dígitos após a `,`; sinal **antes** do símbolo: `-R$ 1.234,56`. Espaço normal (não NBSP) entre símbolo e número. **Os centavos têm o mesmo tamanho** do resto (é um único TextView).
- **Privacidade**: ao alternar `hidden`, faz fade `alpha → 0` (`privacyFadeDurationMs`, `AccelerateDecelerateInterpolator`), troca o texto (para `hiddenText` ou para o valor) e volta a `alpha → 1` com a mesma duração. Ao esconder, cancela a contagem e fixa `display = target`.
- **Fonte**: `fontFamily` (opcional) via `Typeface.create(name, NORMAL)`; senão `Typeface.DEFAULT`. Peso: `'medium'→500, 'semibold'→600, 'bold'→700, 'heavy'→800, 'black'→900, '100'…'900'`, outros → 400. Em API ≥ 28 usa `Typeface.create(tf, weight, false)`; abaixo disso usa BOLD se o peso for ≥ 600. Nos call sites observados da Vision não há `fontFamily`, então o número sai na **fonte do sistema** (Roboto) com o peso dado [I alto].
- `letterSpacingPx` → `letterSpacing = px / textSize`; `lineHeightDp` → `lineSpacing extra = lineHeight×density − textSize`; `textAlign` left/center/right → gravity 19/17/21.

### 11.4 PrivacyCurrencyText (M8991, L2840958): valor BRL usado nos widgets [O]
Props: `valueCents, hidden, fontSize, fontWeight, color, minimumFontScale = 0.6, testID, style, disableMinWidth = false, compactOnOverflow = false, animate = true`.
```
compact = (compactOnOverflow && !hidden) ? resolveCompactBRL(valueCents, getHalfWidgetContentWidth(windowWidth), fontSize) : null
text    = compact ?? (animate ? null : (hidden ? HIDDEN_VALUE : formatBRLExact(valueCents)))
if (text == null)
  → <PierreAnimatedCurrencyText value={valueCents/100} hidden={hidden} hiddenText={HIDDEN_VALUE}
        animateFromZero={false} durationMs={0}            // ← sem contagem: só o fade de privacidade
        disableMinWidth fontSize fontWeight textColor={color} minimumFontScale testID style/>
else
  → <PrivacyText key=… value={text} numberOfLines={1} adjustsFontSizeToFit minimumFontScale allowFontScaling={false} testID
        style={[{fontSize, lineHeight: ceil(fontSize*1.18), color, fontWeight: map(fontWeight)}, style]}/>
```
`map(fontWeight)`: `'normal'→'normal', 'regular'→'400', 'medium'→'500', 'semibold'→'600', 'bold'→'700', 'heavy'→'800', 'black'→'900'`; qualquer outro valor passa sem mudança.

---

## 12. Utilitários de moeda/abreviação [O]

| Função | Módulo / linha | Entrada | Saída |
|---|---|---|---|
| `formatBRLExact(cents)` | M8992 L2841463 | centavos | `[-]R$ 1.234,56` (milhar `.`, decimal `,`, 2 casas, espaço normal) |
| `estimateBRLTextWidth(str, fontSize)` | M8992 | — | `Σ largura(c) × fontSize`, com `{' ':0.32, '.':0.3, ',':0.3, '-':0.45}` e **0.62** para os demais caracteres |
| `getHalfWidgetContentWidth(W)` | M8992 | largura da janela | `max((W−32−18)/2 − 32, 0)` |
| `resolveCompactBRL(cents, width, fontSize = 20)` | M8992 L2841539 | — | ver abaixo |
| `formatReais(reais)` | M9000 L2849598 | reais | `Intl.NumberFormat('pt-BR',{style:'currency',currency:'BRL',min/maxFractionDigits:2}).format` → `R$ 1.234,56` (o Intl usa NBSP U+00A0) |
| `centsToReais(c)` | M9000 | centavos | `isFinite ? round(c)/100 : 0` |
| `formatCentsToBRL(c)` | M9001 / M4430 | centavos | M9001: `(c/100).toLocaleString('pt-BR',{style:'currency',currency:'BRL'})`; M4430 monta à mão `[-]R$ {inteiro pt-BR},{cc}` |
| `formatNumberCompact(n)` | M6742 L2204544 | número | `<1000`: `round(n)`; `<999.500`: sufixo `k` (`x=n/1000`; com `x<10` usa 1 casa, senão arredonda); acima disso, sufixo `M` (`/1e6`, mesma regra). Decimal com `,`; inteiros sem casas. Ex.: `1.234→"1,2k"`, `12.345→"12k"`, `2.500.000→"2,5M"` |
| `formatBRLCompact(v)` | M6742 L2204810 | reais | `'R$ 0'` para zero; `'R$ ' + formatNumberCompact(v)`; negativo `'-R$ ' + …` |
| `formatBRL(v)` / `formatNumber(v)` / `formatCurrency(v, code='BRL')` | M6742 | — | Intl pt-BR com 2 casas |
| `formatPercentage(n)` / `formatCdiPercentage(n)` | M4430 | — | `n.toLocaleString('pt-BR',{maximumFractionDigits:2}) + '%'` / `+ '% do CDI'` |

**`resolveCompactBRL(cents, availableWidth, fontSize)`** (usado por `PrivacyCurrencyText` com `compactOnOverflow`):
1. Se `cents` ou `availableWidth` não forem finitos, ou se `availableWidth ≤ 0` → `null`.
2. `exact = formatBRLExact(cents)`; se `estimate(exact) ≤ availableWidth / 0.8` → `null` (cabe; usa o texto exato/animado).
3. `reais = |cents|/100`; se `reais < 1000` → `null`.
4. `noCents = [-]R$ {round(reais) com milhar}` (sem centavos); se couber (`≤ width/0.8`) → retorna `noCents`.
5. Se `reais < 999.500` → retorna `noCents` mesmo assim (o autoshrink resolve).
6. Senão usa milhões: `m = reais/1e6`; com `m < 100` usa `round(m·10)/10` (vírgula se fracionário), senão `round(m)` → `"[-]R$ 1,2 mi"`.

Observação: **não existe sufixo "mil"** nesse caminho; a forma compacta da Vision só usa `mi`. As strings `'mil'`, `' mil'`, `'k'`, `'M'` e `'bi'` existem no bundle para outros gráficos e eixos (ex.: `formatYAxisCompact`, `formatCompactAxis`) [O: strings; o uso por widget fica nos documentos de cada widget].

---

## 13. Tokens de cor do tema (dark)

Há **dois sistemas paralelos** de cor [O]:
1. **Variáveis CSS do NativeWind** (`vars()` do gluestack, módulo `config` M4438, L1791619 light / L1791623 dark). Elas alimentam as classes `bg-*`, `text-*` e `border-*` como `hsl(var(--x) / alpha)`. Os componentes base dos widgets usam estas.
2. **`THEME_TOKENS`** (M2979, L922395; hook `useThemeColors`) com hex prontos, usados em props JS (ícones, gráficos, `color=`).
O app é forçado em dark (`LIGHT_MODE_DISABLED`, L922727; store `pierre-theme` com `{preference:'dark', mode:'dark'}`).

### 13.1 Variáveis CSS (valor HSL do bundle → hex)
| Variável | HSL (dark) | **Dark** | Light |
|---|---|---|---|
| `--splash-bg` | 240 10% 3.9% | #09090b | #ffffff |
| `--splash-foreground` | 0 0% 98% | #fafafa | #09090b |
| `--splash-muted-foreground` | 240 5% 64.9% | #a1a1aa | #52525b |
| `--splash-primary` | 71.1 100% 50.8% | #d1ff04 | #d1ff04 |
| `--splash-border` | 240 3.7% 15.9% | #27272a | #e4e4e7 |
| `--background` | 240 10% 3.9% | #09090b | #ffffff |
| `--foreground` | 0 0% 98% | #fafafa | #09090b |
| `--foreground-disabled` | 240 5.2% 33.9% | #52525b | #a1a1aa |
| `--surface` | 240 6% 10% | #18181b | #fafafa |
| `--surface-hover` | 240 3.7% 15.9% | #27272a | #f4f4f5 |
| `--surface-glass-overlay` | 240 3.7% 15.9% | #27272a | #e4e4e7 |
| `--surface-pressed` | 240 5.3% 26.1% | #3f3f46 | #d4d4d8 |
| `--overlay` | 0 0% 0% | #000000 | #000000 |
| `--card` | 240 10% 3.9% | #09090b | #ffffff |
| `--brand-bright-green` | 95 94% 76% | #b8fb88 | #b8fb88 |
| `--card-foreground` | 0 0% 98% | #fafafa | #09090b |
| `--pierre` | 71 100% 50% | #d0ff00 | #d0ff00 |
| `--pierre-foreground` | 240 5.9% 10% | #18181b | #18181b |
| `--popover` | 240 10% 3.9% | #09090b | #ffffff |
| `--popover-foreground` | 0 0% 98% | #fafafa | #09090b |
| `--primary` | 0 0% 98% | #fafafa | #18181b |
| `--primary-foreground` | 240 10% 3.9% | #09090b | #ffffff |
| `--secondary` | 240 3.7% 15.9% | #27272a | #f4f4f5 |
| `--secondary-foreground` | 0 0% 98% | #fafafa | #18181b |
| `--muted` | 240 3.7% 15.9% | #27272a | #f4f4f5 |
| `--muted-foreground` | 240 5% 64.9% | #a1a1aa | #52525b |
| `--accent` | 0 0% 98% | #fafafa | #18181b |
| `--accent-hover` | 240 4.8% 95.9% | #f4f4f5 | #27272a |
| `--accent-foreground` | 240 10% 3.9% | #09090b | #ffffff |
| `--destructive` | 0 84.2% 60.2% | #ef4444 | #dc2626 |
| `--destructive-hover` | 0 90.6% 70.8% | #f87171 | #b91c1c |
| `--destructive-subtle` | 0 74.7% 15.5% | #450a0a | #fef2f2 |
| `--destructive-border` | 0 70% 35.3% | #991b1b | #fecaca |
| `--destructive-foreground` | 0 0% 100% | #ffffff | #ffffff |
| `--error-light` | 0 60% 70% | #e08585 | #dd3c3c |
| `--warning` | 37.7 92.1% 50.2% | #f59e0b | #d97706 |
| `--warning-hover` | 43.3 96.4% 56.3% | #fbbf24 | #b45309 |
| `--warning-subtle` | 20.9 91.7% 14.1% | #451a03 | #fffbeb |
| `--warning-border` | 22.7 82.5% 31.4% | #92400e | #fde68a |
| `--warning-foreground` | 240 10% 3.9% | #09090b | #09090b |
| `--success` | 71.5 100% 50.8% | #cfff04 | #8bb800 |
| `--success-receipt` | 74.674 100% 36.078% | #8bb800 | #8bb800 |
| `--success-hover` | 69.3 100% 65.7% | #e4ff50 | #698b00 |
| `--success-subtle` | 78.5 100% 10.2% | #243400 | #fdffe4 |
| `--success-border` | 75.3 87.9% 22.7% | #536d07 | #f2ff90 |
| `--success-foreground` | 78.5 100% 10.2% | #243400 | #09090b |
| `--border` | 240 3.7% 15.9% | #27272a | #e4e4e7 |
| `--border-prominent` | 240 5.2% 33.9% | #52525b | #d4d4d8 |
| `--border-muted` | 240 5.9% 10% | #18181b | #f4f4f5 |
| `--input` | 240 3.7% 15.9% | #27272a | #e4e4e7 |
| `--ring` | 240 4.9% 83.9% | #d4d4d8 | #18181b |
| `--chart-1` | 220 70% 50% | #2662d9 | #e76e50 |
| `--chart-2` | 160 60% 45% | #2eb88a | #2a9d90 |
| `--chart-3` | 30 80% 55% | #e88c30 | #274754 |
| `--chart-4` | 280 65% 60% | #af57db | #e8c468 |
| `--chart-5` | 340 75% 55% | #e23670 | #f4a462 |
| `--brand` | 71.5 100% 50.8% | #cfff04 | #8bb800 |
| `--category-food` | 24.6 95% 53.1% | #f97316 | #c2410c |
| `--category-shopping` | 330.4 81.2% 60.4% | #ec4899 | #be185d |
| `--category-health` | 160.1 84.1% 39.4% | #10b981 | #047857 |
| `--category-digital-services` | 270.7 91% 65.1% | #a855f7 | #7e22ce |
| `--category-transport` | 37.7 92.1% 50.2% | #f59e0b | #b45309 |
| `--category-housing` | 238.7 83.5% 66.7% | #6366f1 | #4338ca |
| `--category-entertainment` | 217.2 91.2% 59.8% | #3b82f6 | #1d4ed8 |
| `--category-education` | 188.7 94.5% 42.7% | #06b6d4 | #0e7490 |
| `--category-insurance` | 215.4 16.3% 46.9% | #64748b | #334155 |
| `--category-betting` | 0 84.2% 60.2% | #ef4444 | #b91c1c |
| `--category-donations` | 292.2 84.1% 60.6% | #d946ef | #a21caf |
| `--category-other` | 240 3.8% 46.1% | #71717a | #3f3f46 |
| `--sidebar-background` | 240 5.9% 10% | #18181b | #fafafa |
| `--sidebar-foreground` | 240 4.8% 95.9% | #f4f4f5 | #3f3f46 |
| `--sidebar-primary` | 224.3 76.3% 48% | #1d4ed8 | #18181b |
| `--sidebar-primary-foreground` | 0 0% 100% | #ffffff | #fafafa |
| `--sidebar-accent` | 240 3.7% 15.9% | #27272a | #f4f4f5 |
| `--sidebar-accent-foreground` | 240 4.8% 95.9% | #f4f4f5 | #18181b |
| `--sidebar-border` | 240 3.7% 15.9% | #27272a | #e5e7eb |
| `--sidebar-ring` | 217.2 91.2% 59.8% | #3b82f6 | #3b82f6 |
Modificadores de opacidade existem para todas as cores em passos de 5 (`/0`…`/100`), ex.: `text-muted-foreground/70` = `hsl(240 5% 64.9% / 0.7)`.

### 13.2 `THEME_TOKENS.dark` (L922395) [O]
| Token | Hex | Token | Hex |
|---|---|---|---|
| foreground | #fafafa | mutedForeground | rgb(161,161,170) = #a1a1aa |
| background | #09090b | card | #09090b |
| muted | #27272a | border | #27272a |
| borderProminent | #525259 | borderStrong | #52525b |
| borderMuted | #18181b | primary | #fafafa |
| primaryForeground | #18181b | surface | #18181b |
| surfaceHover | #27272a | surfacePressed | #3f3f46 |
| foregroundDisabled | #52525b | destructive | **#ff4d4d** (≠ CSS `--destructive` #ef4444) |
| destructiveHover | #f87171 | destructiveSubtle | #450a0a |
| destructiveBorder | #991b1b | destructiveForeground | #ffffff |
| warning | #f59e0b | warningHover | #fbbf24 |
| warningSubtle | #451a03 | warningBorder | #92400e |
| warningForeground | #09090b | success | #cfff04 |
| successHover | #e4ff50 | successSubtle | #243400 |
| successBorder | #536d07 | successForeground | #243400 |
| brand | #cfff04 | pierre | **#c7ff00** (CSS `--pierre` dá #d0ff00) |
| neonLime50 | #fdffe4 | cashbackReleased | #243400 |

Cores fixas recorrentes nos componentes base: chevron e ícone vazio `#71717a` (zinc-500), skeleton `#71717a`, ícone de erro `#EF4444`, texto animado `#FFFFFF`.

---

## 14. Configuração NativeWind compilada (tailwind do bundle)

Fonte: objeto `$compiled` injetado pelo NativeWind (extraído para `parcial/w0tools/nw.json`): `rem: 14`, `darkMode: 'class dark'`, 8.743 regras. Só existem as classes **usadas** no app (compilação JIT) [O].

### 14.1 Tamanhos de fonte
| Classe | fontSize | lineHeight |
|---|---|---|
| `text-2xs` | 10 | — |
| `text-xs` | 10,5 | 14 |
| `text-body-small` | 14 | 18 |
| `text-base` | 14 | 21 |
| `text-lg` | 15,75 | 24,5 |
| **`text-sm`** | **16** | — (sem lineHeight: usa o padrão do RN) |
| `text-xl` | 17,5 | 24,5 |
| `text-2xl` | 21 | 28 |
| `text-3xl` | 26,25 | 31,5 |
| `text-4xl` | 31,5 | 35 |
| `text-5xl` | 42 | 1em (= 42) |
| `text-6xl` | 52,5 | 1em (= 52,5) |
| arbitrários | `text-[1px]`, `[8px]`, `[9px]`, `[10px]`, `[11px]`, `[13px]` … `[20px]`, `[22px]`, `[26px]`, `[28px]`, `[30px]`, `[32px]`, `[34px]`, `[40px]`, `[52px]` | — |

### 14.2 Famílias e pesos (Geist)
Cada `font-<peso>` gera **duas** regras; a segunda vence: `fontFamily: Geist-<Peso>` + `fontWeight: '400'`.
| Classe | fontFamily |
|---|---|
| `font-thin` | Geist-Thin |
| `font-extralight` | Geist-ExtraLight |
| `font-light` | Geist-Light |
| `font-normal`, `font-sans`, `font-body`, `font-geist-regular` | Geist-Regular |
| `font-medium`, `font-geist-medium` | Geist-Medium |
| `font-semibold`, `font-geist-semibold` | Geist-SemiBold |
| `font-bold`, `font-heading`, `font-geist-bold` | Geist-Bold |
| `font-extrabold` | Geist-ExtraBold |
| `font-black` | Geist-Black |

`Text` do gluestack (M2263 `textStyle`): base `font-sans tracking-sm …` + `font-geist-regular`, `size` default `md` → `text-base`; variantes `bold/medium/semibold` → `font-geist-*`; `size: 2xs, xs, body-small, sm, md(=text-base), lg, xl, 2xl … 6xl`. `tracking-sm` **não existe** nas regras, então não tem efeito.

### 14.3 Letter-spacing e line-height
`tracking-tight −0,25`, `tracking-normal 0`, `tracking-wide 0,25`, `tracking-wider 0,5`, `tracking-widest 1`, `tracking-[-0.5px] −0,5`.
`leading-4 14`, `-5 17,5`, `-6 21`, `-7 24,5`, `-8 28`, `-9 31,5`, `-10 35`; `leading-none 1em`, `-tight 1,25em`, `-snug 1,375em`, `-normal 1,5em`, `-relaxed 1,625em`; arbitrários `[14px] [17px] [18px] [19px] [21px] [22px] [24px] [28px] [30px] [31px] [32px] [34px] [36px] [42px] [44px] [56px]`.

### 14.4 Espaçamento (padding/margin/gap/width/height)
Escala padrão do Tailwind × 3,5: `N → N × 3,5` px. Ex.: `0.5=1,75`, `1=3,5`, `1.5=5,25`, `2=7`, `2.5=8,75`, `3=10,5`, `3.5=12,25`, `4=14`, `5=17,5`, `6=21`, `7=24,5`, `8=28`, `9=31,5`, `10=35`, `11=38,5`, `12=42`, `14=49`, `16=56`, `20=70`, `24=84`, `28=98`, `32=112`, `36=126`, `40=140`, `56=196`, `64=224`, `72=252`.

### 14.5 Raios
`rounded-sm 3,5`, `rounded 3,5`, `rounded-md 5,25`, `rounded-lg 7`, `rounded-xl 10,5`, `rounded-2xl 14`, `rounded-3xl 21`, `rounded-full 9999`, `rounded-none 0`; arbitrários `[2px] [16px] [20px] [24px] [26px] [28px] [29px] [32px] [40px]`; parciais `rounded-t-2xl`, `rounded-t-3xl`, `rounded-b-3xl`, `rounded-tl-3xl`, `rounded-tr-3xl`, `rounded-l-lg`, `rounded-r-lg`.
Os widgets **não** usam classe de raio: o 24 vem de `style`.

### 14.6 Sombras e opacidade
| Classe | offset | radius | opacity | elevation |
|---|---|---|---|---|
| `shadow-sm` | 0,1 | 1 | 1 (cor `--tw-shadow-color`) | 1 |
| `shadow-xs` | 0,1 | 2 | 1 | 2 |
| `shadow` | 0,1 | 4 | 1 | 3 |
| `shadow-lg` | 0,4 | 10 | 1 | 8 |
`opacity-0, -40, -50, -60, -70, -[0.01]`.
Os componentes base de widget **não usam sombra nem borda** [O].

---

## 15. Snippets TSX de referência (React Native puro, sem NativeWind)

Reproduzem o caminho **Android** (sem Liquid Glass). Os valores vêm das seções anteriores. Fontes Geist precisam estar carregadas com estes nomes.

```tsx
// widget-base.tsx
import React, { memo, useCallback, useEffect, useRef, useState } from 'react';
import { Pressable, StyleSheet, Text, View, useWindowDimensions, type StyleProp, type ViewStyle } from 'react-native';
import * as Haptics from 'expo-haptics';
import Animated, { Easing, runOnJS, useAnimatedStyle, useSharedValue, withRepeat, withTiming } from 'react-native-reanimated';
import { ChevronRight, WifiOff, AlertTriangle } from 'lucide-react-native'; // substitua pelos seus ícones

// ---- tokens (dark) ----
export const C = {
  surface: '#18181b', foreground: '#fafafa', mutedForeground: '#a1a1aa',
  zinc500: '#71717a', destructive: '#ef4444',
};
export const RADIUS = 24, GAP = 18, H_HALF = 200, H_WIDE = 170, PAD = 14;
const T = { // text-* com rem = 14
  base: { fontSize: 14, lineHeight: 21 }, lg: { fontSize: 15.75, lineHeight: 24.5 },
  xl: { fontSize: 17.5, lineHeight: 24.5 }, x2l: { fontSize: 21, lineHeight: 28 }, sm: { fontSize: 16 },
};
const F = { regular: 'Geist-Regular', extrabold: 'Geist-ExtraBold' };

// ---- privacidade ----
export const HIDDEN_VALUE = 'R$ ••••';

export const PrivacyText = memo(function PrivacyText(
  { value, style, ...rest }: React.ComponentProps<typeof Text> & { value: string },
) {
  const [display, setDisplay] = useState(value);
  const prev = useRef(value);
  const opacity = useSharedValue(1);
  useEffect(() => {
    if (value === prev.current) return;
    prev.current = value;
    opacity.value = withTiming(0, { duration: 150 }, () => {
      runOnJS(setDisplay)(value);
      opacity.value = withTiming(1, { duration: 150 });
    });
  }, [value]);
  const a = useAnimatedStyle(() => ({ opacity: opacity.value }));
  return <Animated.Text {...rest} style={[style, a]}>{display}</Animated.Text>;
});

// ---- shell ----
export function useWidgetPress(onPress?: () => void) {
  return useCallback(() => { Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light); onPress?.(); }, [onPress]);
}

function WidgetShell({ inner, onPress, testID }: { inner: React.ReactNode; onPress?: () => void; testID: string }) {
  const handlePress = useWidgetPress(onPress);
  if (!onPress) return <>{inner}</>;
  return <Pressable onPress={handlePress} testID={`${testID}-pressable`}>{inner}</Pressable>; // sem scale/ripple
}

const Chevron = ({ testID }: { testID: string }) => (
  <View testID={`${testID}-chevron`} style={{ padding: 1.75 }}>
    <ChevronRight size={16} color={C.zinc500} />
  </View>
);

// ---- WidgetValueSlot ----
export function WidgetValueSlot({ label, value, valueSlot, subtitle, valueMinimumFontScale = 0.6, testID, gap = 1.75 }: {
  label?: string; value?: string; valueSlot?: React.ReactNode; subtitle?: string;
  valueMinimumFontScale?: number; testID: string; gap?: number;
}) {
  return (
    <View style={{ rowGap: gap }}>
      <Text allowFontScaling={false} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.7}
        style={[T.base, { fontFamily: F.regular, color: C.mutedForeground }]}>{label}</Text>
      {valueSlot ?? (
        <PrivacyText value={value ?? ''} testID={`${testID}-value`} allowFontScaling={false} numberOfLines={1}
          adjustsFontSizeToFit minimumFontScale={valueMinimumFontScale}
          style={[T.xl, { fontFamily: F.extrabold, color: C.foreground }]} />
      )}
      <PrivacyText value={subtitle ?? ''} testID={`${testID}-subtitle`} allowFontScaling={false} numberOfLines={1}
        adjustsFontSizeToFit minimumFontScale={0.7}
        style={[T.base, { fontFamily: F.regular, color: C.mutedForeground }]} />
    </View>
  );
}

// ---- Widget2x2 (half) ----
export const Widget2x2 = memo(function Widget2x2({
  visual, label, value, valueSlot, subtitle, valueMinimumFontScale = 0.6, onPress,
  showChevron = true, visualPlacement = 'top', testID = 'widget-2x2', containerStyle,
}: {
  visual?: React.ReactNode; label?: string; value?: string; valueSlot?: React.ReactNode; subtitle?: string;
  valueMinimumFontScale?: number; onPress?: () => void; showChevron?: boolean;
  visualPlacement?: 'top' | 'middle'; testID?: string; containerStyle?: StyleProp<ViewStyle>;
}) {
  const top = visualPlacement === 'middle' ? (
    <>
      {showChevron && <View pointerEvents="none" style={{ flexDirection: 'row', justifyContent: 'flex-end' }}><Chevron testID={testID} /></View>}
      <View collapsable={false} pointerEvents="none" style={{ alignSelf: 'stretch', width: '100%' }}>{visual}</View>
    </>
  ) : (
    <View pointerEvents="none">
      {showChevron ? (
        <View style={{ flexDirection: 'row', alignItems: 'flex-start', justifyContent: 'space-between' }}>
          <View style={{ flex: 1 }}>{visual}</View>
          <Chevron testID={testID} />
        </View>
      ) : visual}
    </View>
  );
  const inner = (
    <View testID={testID} style={[{ height: H_HALF, borderRadius: RADIUS, backgroundColor: C.surface,
      padding: PAD, rowGap: 7, justifyContent: 'space-between' }, containerStyle]}>
      {top}
      <WidgetValueSlot label={label} value={value} valueSlot={valueSlot} subtitle={subtitle}
        valueMinimumFontScale={valueMinimumFontScale} testID={testID} />
    </View>
  );
  return <WidgetShell inner={inner} onPress={onPress} testID={testID} />;
});

// ---- WidgetWide ----
export const WidgetWide = memo(function WidgetWide({
  visual, label, value, subtitle, valueMinimumFontScale = 0.75, onPress, showChevron = true,
  testID = 'widget-wide', containerStyle,
}: {
  visual?: React.ReactNode; label?: string; value?: string; subtitle?: string; valueMinimumFontScale?: number;
  onPress?: () => void; showChevron?: boolean; testID?: string; containerStyle?: StyleProp<ViewStyle>;
}) {
  const inner = (
    <View testID={testID} style={[{ height: H_WIDE, borderRadius: RADIUS, backgroundColor: C.surface,
      padding: PAD, justifyContent: 'space-between' }, containerStyle]}>
      <View pointerEvents="none" style={{ flexDirection: 'row', alignItems: 'flex-start', justifyContent: 'space-between' }}>
        <View style={{ flex: 1, paddingRight: 10.5 }}>
          <Text allowFontScaling={false} numberOfLines={1} adjustsFontSizeToFit minimumFontScale={0.8}
            style={[T.lg, { fontFamily: F.regular, color: C.mutedForeground }]}>{label}</Text>
          <PrivacyText value={value ?? ''} testID={`${testID}-value`} allowFontScaling={false} numberOfLines={1}
            adjustsFontSizeToFit minimumFontScale={valueMinimumFontScale}
            style={[T.x2l, { fontFamily: F.extrabold, color: C.foreground }]} />
          {subtitle ? (
            <PrivacyText value={subtitle} testID={`${testID}-subtitle`} allowFontScaling={false} numberOfLines={1}
              adjustsFontSizeToFit minimumFontScale={0.8}
              style={[T.base, { fontFamily: F.regular, color: C.mutedForeground }]} />
          ) : null}
        </View>
        {showChevron ? <Chevron testID={testID} /> : null}
      </View>
      {visual ? <View pointerEvents="none">{visual}</View> : null}
    </View>
  );
  return <WidgetShell inner={inner} onPress={onPress} testID={testID} />;
});

// ---- WidgetFull ----
export const WidgetFull = memo(function WidgetFull({
  title, children, onPress, showChevron = true, headerRight, testID = 'widget-full',
}: {
  title?: string; children?: React.ReactNode; onPress?: () => void; showChevron?: boolean;
  headerRight?: React.ReactNode; testID?: string;
}) {
  const inner = (
    <View testID={testID} style={{ borderRadius: RADIUS, backgroundColor: C.surface, padding: PAD, rowGap: 10.5 }}>
      <View style={{ flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between' }}>
        <Text allowFontScaling={false} style={[T.lg, { fontFamily: F.regular, color: C.mutedForeground }]}>{title}</Text>
        <View style={{ flexDirection: 'row', alignItems: 'center', columnGap: 7 }}>
          {headerRight}
          {showChevron && <Chevron testID={testID} />}
        </View>
      </View>
      {children}
    </View>
  );
  return <WidgetShell inner={inner} onPress={onPress} testID={testID} />;
});

// ---- estados ----
export function WidgetEmptyState({ message = 'Sem dados disponíveis', compact = false }) {
  return (
    <View testID="widget-empty-state" style={{ alignItems: 'center', justifyContent: 'center',
      paddingVertical: compact ? 3.5 : 14, rowGap: 7 }}>
      <WifiOff size={compact ? 18 : 22} color={C.zinc500} />
      <Text allowFontScaling={false} style={[compact ? T.sm : T.base,
        { fontFamily: F.regular, color: C.mutedForeground, textAlign: 'center' }]}>{message}</Text>
    </View>
  );
}

export function WidgetErrorState({ message = 'Falha ao sincronizar dados', compact = false }) {
  return (
    <View testID="widget-error-state" style={{ alignItems: 'center', justifyContent: 'center',
      paddingVertical: compact ? 3.5 : 14, rowGap: 7 }}>
      <AlertTriangle size={compact ? 18 : 22} color="#EF4444" />
      <Text allowFontScaling={false} style={[compact ? T.sm : T.base,
        { fontFamily: F.regular, color: C.destructive, textAlign: 'center' }]}>{message}</Text>
    </View>
  );
}

export function WidgetLoadingState({ variant = 'content' }: { variant?: 'content' | 'value' | 'visual' }) {
  const opacity = useSharedValue(0.3);
  useEffect(() => {
    opacity.value = withRepeat(withTiming(0.7, { duration: 800, easing: Easing.inOut(Easing.ease) }), -1, true);
  }, []);
  const a = useAnimatedStyle(() => ({ opacity: opacity.value }));
  const bar = (s: ViewStyle) => <View style={[{ backgroundColor: C.zinc500 }, s]} />;
  return (
    <Animated.View testID="widget-loading-state" style={a}>
      {variant === 'visual' && bar({ height: 64, width: 64, borderRadius: 32 })}
      {variant === 'value' && <>
        {bar({ height: 24, width: '60%', borderRadius: 12 })}
        {bar({ height: 14, width: '40%', borderRadius: 7, marginTop: 6 })}
      </>}
      {variant === 'content' && <>
        {bar({ height: 44, width: '100%', borderRadius: 12, marginBottom: 8 })}
        {bar({ height: 44, width: '100%', borderRadius: 12, marginBottom: 8 })}
        {bar({ height: 44, width: '70%', borderRadius: 12 })}
      </>}
    </Animated.View>
  );
}

// ---- grid ----
export function useVisionWidgetWidths() {
  const { width } = useWindowDimensions();
  const full = width - 32;
  return { full, half: (full - GAP) / 2 };   // wide e full usam `full`
}

// ---- contador de moeda (equivalente JS do MoneyCounterInterpolator) ----
export const smootherstep = (t: number) => { 'worklet'; const x = Math.min(Math.max(t, 0), 1); return x * x * x * (x * (6 * x - 15) + 10); };
export function formatBRLExact(cents: number) {
  const neg = cents < 0; const abs = Math.round(Math.abs(cents));
  const int = String(Math.trunc(Math.floor(abs / 100))).replace(/\B(?=(\d{3})+(?!\d))/g, '.');
  return `${neg ? '-' : ''}R$ ${int},${String(abs % 100).padStart(2, '0')}`;
}
```

Notas de fidelidade:
- No original, `VStack`, `HStack` e `Box` são do gluestack com NativeWind; o `VStack` não define `flexDirection` além de `column`. Os `gap-*` viram `rowGap`/`columnGap`.
- No iOS 26 com Liquid Glass, o original troca o `bg-surface` por `GlassContainer glassEffectStyle="clear"` (`expo-glass-effect`) com o mesmo raio e altura.
- Para o número animado idêntico ao do Android: Animated/Reanimated com `duration` do call site (400–600 ms nos usos da Vision; 1800 ms é o default) e easing `smootherstep`; primeira animação parte de 0.

---

## 16. Dúvidas abertas / limites

1. `widgetRadius` do **WidgetWide**: o decompilador perdeu a constante do módulo (`_closure0_slot9`); assumido 24 porque o `glassStyle` do mesmo módulo usa `borderRadius: 24` [I]. Também só afeta o caminho glass/iOS e o `containerStyle` opaco.
2. O fallback do `VisionWidgetErrorBoundary` chama `jsx(X, {compact:true})`; X não foi resolvido nominalmente (o import é o barrel M8976). O mais provável é `WidgetErrorState` [I alto].
3. O `storage` do `pierre-privacy` é `createJSONStorage(() => módulo.default)`; o módulo concreto (AsyncStorage ou MMKV) não foi confirmado [I].
4. `isLiquidGlassAvailable()` no Android: assumido `false` (a API é do iOS 26). Se alguma versão do `expo-glass-effect` devolver `true` no Android, o widget fica sem `bg-surface` [I].
5. Fonte do número animado nativo: os call sites da Vision não passam `fontFamily`, então sai em Roboto [I alto]. Vale validar em aparelho.
6. `PrivacyCurrencyText` usa `durationMs: 0`, então os valores BRL dos widgets que usam esse componente **não fazem contagem**, só fade de privacidade. A contagem aparece onde `AnimatedVisionCurrencyText` é usado com `durationMs` 400/600 [O]. O mapeamento widget → componente fica nos documentos de cada widget.
7. `emptyStates`/`isEditMode` do `VisionWidgetRendererInner` nunca são passados pelo único chamador observado, então `emptyState` customizado não é usado na home [O]. Pode existir em outra tela (catálogo/preview) não mapeada.
8. Os nomes de ícone (`chevronRight`, `wifiOff`, `feedbackWarning`) são do `PierreIcon` (M2465, mapa `pierreIcons`). O glyph exato (SVG) não foi extraído aqui.
