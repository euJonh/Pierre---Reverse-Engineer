# Grupo 6a: home do cartão (estrutura)

> Análise estática do bundle Hermes v98 (`_trabalho/out.js`). **[O]** = observado (`Lnnnn` = linha de `out.js`, `Mnnnn` = módulo Metro); **[I]** = inferido. Unidades em px/dp; NativeWind com rem = 14; valores de cor no tema dark.
> Base reaproveitada, não repetida aqui: `widgets/00-base.md` (componentes base, alturas, tokens, privacidade), `widgets/05b-leo-ia.md` §5 (`AIWidgetRenderer`) e §7 (`WidgetSlotRenderer`, `LiveWidgetRenderer`, `AIWidgetWrapper`, `PendingSyncOverlay`).
> Fora do escopo (agente 6b): widgets Cashback, Fatura, Cartão (`CardWithCollapse`) e Gastos por categoria.

## 1. Registro de widgets do cartão (`WIDGET_REGISTRY`)

### 1.1 Por que há duas ocorrências de `WIDGET_REGISTRY` [O]
O nome é o mesmo, mas são **dois registros sem relação**:

| Linha | Módulo | O que é | Quem usa |
|---|---|---|---|
| L775589 | **M2199** (L775571) | Registro da **home do cartão**: id → `{id, size, supportedSizes, label, description, icon?, fixed?}` | store `useWidgetOrderStore` (M2198), `SortableWidgetGrid` (M9518), `JiggleWrapper` (M9519), `PendingSyncOverlay` (M9537), `ReplaceWidgetSheet` (M9553), `useHomeLayout` (M9555), `buildPatchPayload` (M9557), adicionar widget (M9766, M9770) |
| L2202932 | **M6739** (L2202910) | Registro dos **widgets de UI generativa do chat**: nome da tool → `{component, skeletonTitle, tabTitle}` | `renderFinancialWidget` (M7082), `Widget`/`SPECIAL_TOOL_TAB_TITLES` (M7083) e `WidgetCarousel` (M7084), todos usados pela mensagem do chat (M4931) |

Ou seja: a home do cartão usa só M2199. O M6739 é do chat (ver §3) e não tem nada a ver com a grade do cartão. O registro da Vision é um terceiro objeto, `VISION_WIDGET_REGISTRY` (M2195, L771869).

### 1.2 M2199: registro da home do cartão (L775571–775700) [O]
Exports: `WIDGET_REGISTRY`, `FIXED_WIDGET_IDS`, `DEFAULT_SORTABLE_ORDER`, `DEFAULT_WIDGET_ORDER`, `isResizable`.

| # | id | label | `size` padrão | `supportedSizes` | ícone (`PierreIcon`) | description | flags |
|---|---|---|---|---|---|---|---|
| 0 | `card` | Cartão | full | full | — | Visualize seu cartão de crédito | `fixed: true` |
| 1 | `last-purchase` | Última compra | full | full | — | Veja sua compra mais recente | `fixed: true` |
| 2 | `available-limit` | Limite usado | half | half, wide | `miscGauge` | Acompanhe quanto do seu limite já foi usado | |
| 3 | `invoice` | Fatura | half | half | `receiptStraight` | Confira o valor da sua fatura atual | |
| 4 | `monthly-spending` | Gasto mensal | full | wide, full | `arrowTrendUp` | Visualize a evolução dos seus gastos no mês | |
| 5 | `subscriptions` | Assinaturas | half | half, wide, full | `deviceTv` | Gerencie suas assinaturas recorrentes | |
| 6 | `installments` | Parcelas | half | half, wide, full | `cardCredit` | Acompanhe suas compras parceladas | |
| 7 | `recent-transactions` | Transações recentes | full | half, wide, full | `contentTextList` | Veja suas últimas transações no cartão | |
| 8 | `spending-by-category` | Gastos por categoria | half | half, wide, full | `productChart` | Veja como seus gastos se dividem por categoria | |
| 9 | `cdb-position` | CDB | full | half, wide, full | `coinPiggy` | Acompanhe a posição do seu investimento em CDB | |
| 10 | `cashback` | Cashback | wide | half, wide | `cwProductsAward` | Acompanhe o cashback acumulado do seu cartão | |

Constantes:
```js
FIXED_WIDGET_IDS       = ['card', 'last-purchase']
DEFAULT_SORTABLE_ORDER = ['available-limit', 'invoice', 'cashback', 'cdb-position', 'monthly-spending',
                          'subscriptions', 'installments', 'recent-transactions', 'spending-by-category']
DEFAULT_WIDGET_ORDER   = [...FIXED_WIDGET_IDS, ...DEFAULT_SORTABLE_ORDER]
isResizable(id)        = WIDGET_REGISTRY[id].supportedSizes.length > 1   // 'invoice', 'card' e 'last-purchase' não redimensionam
```
Notas:
- Os ids `card` e `last-purchase` (`fixed`) **não** entram em `order` nem na grade. O cartão é desenhado fora da grade, pelo `CardWithCollapse` (§2.3). [O] Não há render de `last-purchase` na grade. A "última compra" virou uma **zona contextual** do topo (`HomeContextualZone`, M9171: tipo `last-purchase`, título "Compra aprovada", ícone `feedbackSuccess` `#95FF48`, prioridade 9; L2911419, L3017193) [O].
- Diferente da Vision, o registro do cartão **não** tem `tapAction`, `navigateTo` nem `chartVariants`. O tipo de toque vem do `WidgetRenderer` (L2982600, ver `05b-leo-ia.md` §7.2).
- Ids `ai-<uuid>` (widgets de IA) não estão no registro. Para eles, `resolveSize` (L772741) usa o tamanho guardado em `widgetSizes`, com fallback `'half'`.

### 1.3 Layout padrão do servidor (`DEFAULT_HOME_LAYOUT`, M9556, L2995780) [O]
Placeholder usado antes da primeira resposta (mesma forma do `GET /api/mobile/home-layout`):
```js
{ backgroundId: 'skin-ambient', generatedAt: null, version: 0, widgets: [
  { id:'available-limit',      size:'half', status:'empty', data:{ availableAmount:'R$ 0,00', totalLimit:'R$ 0,00', usedPercentage:0 } },
  { id:'invoice',              size:'half', status:'empty', data:{ monthLabel:<mês atual, toLocaleString('pt-BR',{month:'long'})>, amount:'R$ 0,00', closingDate:'', bars:[{value:0,isCurrent:true}] } },
  { id:'cashback',             size:'wide', status:'empty', data:{ amount:'R$ 0,00', pendingAmount:'R$ 0,00', availableCents:0, pendingCents:0 } },
  { id:'cdb-position',         size:'full', status:'empty', data:{ productName:'CDB mIAu Pierre', termLabel:'', totalBalance:0, investedAmount:0, totalEarnings:0, extraOverCdi:0, profitPercent:0, cdiPercent:0, positionCount:0, startLabel:'', endLabel:'', series:[…] } },
  { id:'monthly-spending',     size:'full', status:'empty', data:{ amount:'R$ 0,00', data:[{value:0,label:'01'}, …] } },
  { id:'subscriptions',        size:'half', status:'empty', data:{ logos:[], amount:'R$ 0,00', subtitle:'Nenhuma assinatura' } },
  { id:'installments',         size:'half', status:'empty', data:{ count:'0', subtitle:'Nenhuma parcela', paidPercentage:0, paidAmount:'R$ 0,00', totalAmount:'R$ 0,00', items:[] } },
  { id:'recent-transactions',  size:'full', status:'empty', data:{ transactions:[] } },
  { id:'spending-by-category', size:'half', status:'empty', data:{ amount:'R$ 0,00', categories:[], subtitle:'0 categorias' } },
]}
```
O `data` de cada item segue a forma que o `WidgetRenderer` espera (strings já formatadas em BRL pelo servidor). O detalhe por widget fica com o outro agente (6b).

### 1.4 Terceiro catálogo: `WIDGET_SURFACE_CATALOG` (M9756, L3074712) [O]
Usado pela tela de adicionar widget (`CardAddWidgetScreen`, M9751; utilitários de grade M9752: `WIDGET_GRID_SIZE_ORDER`, `getWidgetWidth`, `canResizeTo`…). Cada item tem `surfaces` (`'card'`, `'vision'` ou ambos) e `getWidgetsForSurface(surface)` filtra por superfície.

| id | label aqui | defaultSize | supportedSizes | surfaces |
|---|---|---|---|---|
| available-limit | Limite usado | half | half, wide | card, vision |
| monthly-spending | **Gastos do mês** | **wide** | half, wide, full | card, vision |
| subscriptions | Assinaturas | half | half, wide, full | card, vision |
| installments | **Parcelamentos** | half | half, wide, full | card, vision |
| recent-transactions | Transações recentes | full | half, wide, full | card, vision |
| spending-by-category | Gastos por categoria | half | half, wide, full | card, vision |
| card | Cartão | full | full | card (`fixed`) |
| last-purchase | Última compra | full | full | card (`fixed`) |
| invoice | Fatura | half | half | card |
| account-balance, cash-flow, patrimony, cards, wallet, cdb-investment\*, cdb-performance\*, cdb-position\*, leo-agent\*, leo-insight\* | … | … | … | vision (\* `onDemand`) |

Divergências com o M2199 (que manda na grade do cartão): `monthly-spending` (aqui wide/[half,wide,full]; lá full/[wide,full]), label de `installments`, e `cashback`/`cdb-position` não aparecem com `surfaces` contendo `card`. Para a réplica, use o **M2199** como fonte de verdade da grade e o catálogo só para a lista "adicionar widget" [I].

### 1.5 Widgets individuais (fora do escopo 6a)
Cashback (`cashback`), Fatura (`invoice`), Cartão (`card`/`CardWithCollapse`) e Gastos por categoria (`spending-by-category`, `SpendingByCategoryWidget`/`Full`) são cobertos pelo agente 6b. Os demais ids do cartão reaproveitam componentes já especificados em `widgets/01`–`05a` [I: confirmar no `WidgetRenderer`, L2982600].

## 2. Tela da home do cartão

### 2.1 Rota e montagem [O]
- **Não há rota própria.** A home do cartão é uma *cena* do hub da home: `HomeHubScreen` (M8777) → `CardsScene` (M8812, L2793576) → `<CardHomeScreen embedded topInset isActive scrollYShared scrollToTopSignal/>`. Rota física: `./(protected)/(drawer)/(tabs)/vision/index.tsx`. Deep link usado no código: `/vision?section=cards` (ex.: `/vision?section=cards&cardLock=toggle`).
- A cena fica num `View className="flex-1" pointerEvents={isActive ? 'auto' : 'none'}`.
- `CardHomeScreen` (memo, M9154, L2907885) também sai no barrel do cartão (M9748/M9749) [I: não achei uso fora do hub].
- Rotas abertas a partir da tela: `/card/add-widget` (adicionar widget), `/card/widget-detail?id=` (preview a partir de "adicionar"), `/card/details`, `/card/card-data`, `/card/credit-limit`, `/card/benefits`, `/card/card-settings`, `/card/add-to-wallet` e `/security/confirm-pin`.
- A lógica toda fica em `useCardHomeScreen({embedded, topInset, isActive})` (M9610, L3012427; retorno em L3015510). As animações ficam em `useCardHomeScreenAnimations(scrollYShared, scrollToTopSignal)` (M9609, L3011062).

### 2.2 Medidas globais [O]
| Item | Valor |
|---|---|
| `safeTop` | `embedded ? (topInset ?? 0) : insets.top` |
| `scrollContentStyle.paddingTop` | `safeTop + 32` |
| `scrollContentStyle.paddingBottom` | `computeCardHomeScrollBottomPadding` (M8786, L2783817): `(Android ? 80 : TAB_BAR_HEIGHT) + max(safeBottom, 12) + 16`. O 80 é `PORTED_TAB_BAR_ACCESSORY_HIDDEN_CLEARANCE` (M3627) |
| `ScrollView` | `scrollEventThrottle` 16 (caminho com `GestureDetector`) ou 32 (caminho alternativo), `showsVerticalScrollIndicator=false`, `bounces=false`, `scrollEnabled = !isContextMenuOpen` |
| Fundo da tela | `View bg-background absoluteFill` (#09090b) + `ScreenBackground {backgroundId: displayedBg.bgId, cardSkinSource: skin.background}`, com opacidade animada (`bgAnimStyle`) |
| Grade (`styles.sortableContainer`) | `{paddingHorizontal:16, paddingTop:24, paddingBottom:16}` |

Constantes de `M8786` (L2783786):
- `FAB_WIDTH = 260`;
- **`SHOW_ADD_WIDGETS_BUTTON = false`**;
- **`CARD_COLLAPSE_TO_HEADER_ENABLED = false`**;
- `CARD_SHRINK_RANGE = 160`;
- `MINI_CARD_BORDER_COLOR = '#27272A'`;
- `MINI_CARD_FRAME_STYLE = {width:64, height:40, borderRadius:6, borderWidth:1, borderColor:'#27272A'}`;
- `CARD_SNAP_COMMIT_THRESHOLD = 24`.

Com essas duas flags em `false`, o botão "Adicionar widgets" e o minicartão no header **não aparecem** nesta versão.

### 2.3 Árvore (estado normal, cartão ativo) [O L2907885–2908970]
```
View testID="card-home-content" style={flex:1}  (accessibilityElementsHidden/importantForAccessibility quando isGuarded)
├─ View bg-background absoluteFill
├─ (hasActiveCard) Animated.View absoluteFill pointerEvents=none style=bgAnimStyle → ScreenBackground
├─ IOSOnly → PullToExpandCard (só iOS)
├─ GestureDetector(composedScrollGesture) → Animated.ScrollView(contentContainerStyle=scrollContentStyle)
│   ├─ Animated.View style=cardPullStyle
│   │   ├─ VStack className="gap-4 pt-4 px-4"                         ← topo (gap 14, pt 14, px 14)
│   │   │   ├─ (contextualZones.length>0) HomeContextualZone {zones, homeState, onDismissAll, onDismissDemo, onZoneHandler, onVisibleBackgroundChange}
│   │   │   └─ Animated.View style=cardCollapseStyle onLayout=handleCardAreaLayout
│   │   │       └─ Animated.View style=cardFloatStyle
│   │   │           └─ CardWithCollapse {isCollapsed: cardExpandProgress, isLoading, lastFourDigits, skin, isVisible:true, isLocked,
│   │   │                                isLockLoading, onLock, onWallet, onBenefits, onLimits, onSettings, onOpenCardData}  ← "Cartão" (6b)
│   │   ├─ (isCardInactive) VStack pt-4 → CardInactiveState {status, onCreateNewCard}
│   │   ├─ (showWalletBanner) AddToWalletBanner {onAdd, onOpenWallet, inWallet}
│   │   ├─ View style=sortableContainer
│   │   │   ├─ (cartão ativo) CardGridSurface {widgetSlots, onReplaceWidget, onWidgetTap, scrollableRef, scrollY, jigglePaused}
│   │   │   └─ (cartão inativo) InactiveInvoiceCard
│   │   └─ (SHOW_ADD_WIDGETS_BUTTON && !isEditMode) AddWidgetsButton   ← desligado
│   │   (isFirstLoad) no lugar de tudo acima: VStack "gap-4 pt-4 px-4 pb-8" → HomeWidgetsSkeleton
│   └─ Animated.View style=collapseRunwaySpacerStyle   (altura 0 com a flag desligada)
├─ (!hasCard && !isCardLoading) NoCardEmptyState {safeBottom, isActive}
│     ou CardPreReleaseState {investedCents, hasInvestments, targetCents, canCreateCard, topInset, isActive, scrollYShared}
├─ (showCardSkeleton) View {flex:1, paddingTop: safeTop+16, paddingHorizontal:16} → HomeWidgetsSkeleton
├─ CardInvestmentUpdateSheet {isActive}
├─ (!isEditMode && !showCardSkeleton) Animated.View headerBlurWrap {absolute, top/left/right:0, zIndex:4, overflow:hidden, height: safeTop+64} + headerBlurStyle
│     └─ LinearGradient colors ['rgba(0,0,0,0.88)','rgba(0,0,0,0.88)','rgba(0,0,0,0.4)','transparent'] locations [0,0.5,0.8,1] absoluteFill
│        (no light: as mesmas paradas em rgba(255,255,255,…))
├─ (!isEditMode && !showCardSkeleton && !embedded) View fixedHeader {absolute, top/left/right:0, zIndex:5} pointerEvents=box-none → CardTopHeader
├─ (CARD_COLLAPSE_TO_HEADER_ENABLED …) minicartão {top: safeTop+8, MINI_CARD_FRAME_STYLE} → Pressable "mini-card-scroll-top"   ← desligado
├─ (isEditMode) EditModeHeader {safeTop}
├─ (!IS_ANDROID) SkiaLightOverlay {progress, startY, endY}
├─ (!IS_ANDROID) EdgeGlowLoading {visible:isAIReorganizing, onBloom, sound}
├─ DemoConfigSheet, ReplaceWidgetSheet {widgetId: replaceWidgetId}, InfoSheet de sincronização, InfoSheet de bloqueio
└─ (isGuarded) Animated.View testID="card-pin-guard" exiting=FadeOut.duration(220) {absolute, inset 0, zIndex:100, elevation:100, center}
      → View bg-background absoluteFill + Spinner size="lg"
```

Notas:
- **`CardTopHeader`** (M9579, L3002136):
  - Linha `{flexDirection:'row', alignItems:'center', justifyContent:'space-between', paddingHorizontal:16, paddingBottom:8, paddingTop: insets.top + 8}`.
  - À esquerda, `PierreAvatar size 48` (testID `card-header-avatar`, hitSlop 8).
  - À direita, o botão de privacidade (testID `card-header-privacy`): pílula `{height:36, paddingHorizontal:10, paddingVertical:8, borderRadius:9999}` com `eyeOn`/`eyeOff` 16 `#FAFAFA`.
  - Fundo da pílula: `BlurView intensity 40 tint dark` no Android, `GlassView regular` no iOS e `rgba(24, 24, 27, 0.4)` como fallback.
  - O toque chama `toggleHideValues` (store de privacidade, `00-base.md` §10).
  - **Só aparece quando a tela não está `embedded`**. No hub, quem desenha o header é o próprio hub.
- **Animações do cartão** (M9609):
  - Puxar para baixo com `scrollY ≤ 2` faz `cardPullY = −translationY × 0,5`.
  - Ao soltar, se `cardPullY ≤ −EXPAND_THRESHOLD` (**120**, M9155), alterna `cardExpandProgress` (0↔1, `withTiming 250 ms Easing.out(cubic)`) e dispara háptico. Em seguida `cardPullY` volta a 0 em 250 ms.
  - No Android, com a flag de colapso desligada, `cardFloatStyle` fica `{opacity:1, translateY:0, scale:1}`.
- `jigglePaused` fica `true` enquanto a lista rola. Ao parar, um `setTimeout` de **300 ms** libera o tremor [I: nome `scheduleScrollStop`].
- `HomeContextualZone` (M9171) mostra faixas contextuais no topo, cada uma com fundo próprio (`getStateBackground`): fatura vencida ou fechada, compra aprovada ou negada, assinatura detectada, insight. Fica fora do escopo.

### 2.4 Estados da tela [O]
| Estado | Condição | O que aparece |
|---|---|---|
| Guarda de PIN | `isGuarded` | overlay `bg-background` + `Spinner lg`, que sai com `FadeOut 220 ms`. Textos: "Confirme o PIN do app", "Para acessar seu cartão, confirme seu PIN de segurança." e "Não foi possível verificar seu PIN. Tente novamente." |
| Carregando (1ª vez) | `isFirstLoad` | `HomeWidgetsSkeleton` no lugar do conteúdo |
| Carregando cartão | `showCardSkeleton` | `HomeWidgetsSkeleton` com `paddingTop safeTop+16` e `px 16` |
| Sem cartão | `!hasCard && !isCardLoading` | `NoCardEmptyState` ou `CardPreReleaseState` (pré-lançamento: CDB investido × mínimo `DEFAULT_MIN_CDB_INVESTMENT_CENTS`) |
| Cartão inativo | `isCardInactive` | `CardInactiveState` (botão para criar um cartão novo) + `InactiveInvoiceCard` no lugar da grade |
| Sincronizando | `isSyncing \|\| isLayoutSyncing` | `InfoSheet loading centered` com "Sincronizando seus widgets" / "Seus widgets estão sendo atualizados. Isso leva apenas alguns segundos." |
| Sync concluída | após a sync | háptico `Success` + toast "Widgets sincronizados" |
| Sync falhou | `syncFailed` | toast "Estamos com problemas para sincronizar seus widgets. Fique tranquilo, logo mais isso será normalizado." |
| Widget sem dado, com erro ou pendente | por slot | §2.6 e `05b-leo-ia.md` §7 |

`HomeWidgetsSkeleton` (M9175, L2919520) usa o `Skeleton` do moti com `transition {type:'timing', duration:1200}` e `colorMode` dark:
```
VStack gap-6 testID="home-widgets-skeleton"
├─ HStack items-center justify-between: círculo 40×40 (radius 'round') | pílula 132×40 r20
├─ VStack gap-2.5: barra 100%×22 r8, barra 111×22 r8
├─ bloco 100%×232 r24              ← lugar do cartão
└─ VStack gap-2.5: barra 100%×22 r8, barra 111×22 r8
```

### 2.5 Grade de widgets [O]
`CardGridSurface` (memo, M9516/M9517, L2978457) só repassa as props para `SortableWidgetGrid` (M9518, L2978531). Diferenças em relação à Vision (`00-base.md` §3.1):

| | Vision | **Cartão** |
|---|---|---|
| GAP | 18 | **12** (constante do módulo, L2978657) |
| `fullWidth` | `W − 32` | `W − 32` |
| `halfWidth` | `(W − 32 − 18)/2` | **`(W − 32 − 12)/2`** (W = 393 → 174,5) |
| Itens fixos na grade | sim | **não** (o cartão fica fora da grade) |
| Container | `View` fora da edição, `Sortable.Flex` na edição | **sempre `Sortable.Flex`** (react-native-sortables) |
| Item | `Animated.View` com `LinearTransition.springify()` | `View {width, overflow:'visible', zIndex: isActiveResize ? 999 : 0}`, testID `sortable-item-<id>` |

Configuração do `Sortable.Flex`: `flexDirection:'row', flexWrap:'wrap', gap:12, customHandle:true, sortEnabled: activeResizeId === null, dragActivationDelay:200, activeItemScale:1.05, activeItemOpacity:1, activeItemShadowOpacity:0.4, inactiveItemOpacity:0.65, hapticsEnabled:false, autoScrollEnabled: !!scrollableRef, autoScrollDirection:'vertical', autoScrollActivationOffset:150, autoScrollMaxOverscroll:9999, autoScrollMaxVelocity:2500, autoScrollInterval:0, animateScrollTo:false`.
- Largura do item: `effectiveSize === 'half' ? halfWidth : fullWidth` (wide e full ocupam a linha inteira). As alturas vêm dos componentes base: half 200, wide 170 e full pelo conteúdo (`00-base.md` §3.3).
- `effectiveSize` = `widgetSizes[id]`. Para id de sistema sem tamanho salvo, vale `WIDGET_REGISTRY[id].size`. Para `ai-*`, vale o tamanho do widget de IA ou `'half'`.
- Slots `empty` retornam `null` e não ocupam espaço.
- Cadeia de cada item: `View` → `JiggleWrapper` (M9519) → `WidgetSlotRenderer` (M9522, `05b-leo-ia.md` §7.1).
- Ao soltar o item (`onDragEnd`): `setOrder(novaOrdem)` e o analytics `{action:'reordered', widget_id, widget_type: 'ai'|'system', from_position, to_position, source_screen:'home', page_name}`.

**`JiggleWrapper`** (M9519, L2979228) usa as mesmas constantes do `VisionJiggleWrapper` (L2979371–2979390):
- `JIGGLE_ANGLE 0.8°` e `JIGGLE_DURATION 180 ms`: o primeiro meio-ciclo dura 90 ms e depois vem `withRepeat(±0.8°, 180 ms, Easing.inOut(sin), −1, true)`.
- Long-press: **500 ms** abre o menu e **1500 ms** entra no modo de edição.
- Outras constantes do módulo: `200`, `20`, `240`, `52`, `12`, `120`.
- Pulse do widget recém-adicionado: escala 1,08 → 0,97 → 1,04 → 1 em 180/160/140/140 ms; começa 400 ms após montar e chama `clearLastAdded` após 700 ms.
- Opacidade durante o resize: `1 − min(|dx|/60, 1) × 0,5`. A prévia do resize usa opacidade `min(|dx|/15, 1) × 0,55`.
- Auto-scroll ao entrar em edição: `scrollTo({y, animated:true})` com margens 120/60 e 450 ms [I: o papel exato do 0,33].

**Menu de contexto** (`WidgetContextMenu`, M9520, L2980804):
- Cartão `{borderRadius:16, overflow:'hidden'}` dentro de `GlassContainer lightweight regular`.
- Itens `flex-row items-center gap-2 rounded-xl px-4 py-2.5 active:bg-white/10`, com ícone 16 e texto `text-[15px]` branco. A ação destrutiva usa `text-red-500`/`#FF453A`.
- Opções: "Substituir" (`arrowSwipeHorizontal`), "Ocultar dados"/"Mostrar dados" (`eyeOff`/`eyeOn`) e remover.

**`ReplaceWidgetSheet`** (M9553, L2993415) abre por `onReplaceWidget(id)`, só para ids de sistema:
- Título "Substituir" e subtítulo "Visualize este widget em modo compacto | horizontal | tela cheia" (half | wide | full).
- Botão `size xl rounded-full w-full` "Substituir" (testID `replace-widget-sheet-replace`).
- Ao confirmar, grava `widgetSizes`/`version`, faz o PATCH e mostra o toast "`<label>` alterado para `<modo>`" ou "… salvo, dados serão carregados em breve".

**`EditModeHeader`** (M9607, L3009819):
- Container `{position:absolute, top/left/right:0, paddingHorizontal:20, paddingBottom:16, zIndex:10, paddingTop: safeTop+12}` com fundo `bg-background`.
- Título "Editar widgets" (`text-2xl font-bold text-foreground`).
- Botões à direita, dentro de `GlassContainer lightweight regular` (`borderRadius 20`):
  - "Preencher" (`text-sm font-semibold text-pierre`, testID `edit-mode-fill-gaps-button`), só quando `useHasGaps()` é verdadeiro. Pílula `{flexDirection:row, gap:6, paddingHorizontal:14, paddingVertical:8, borderRadius:20}`. Chama `fillGaps()`.
  - "Concluído" (`text-base font-semibold text-pierre`, testID `edit-mode-done-button`, `{paddingHorizontal:18, paddingVertical:8, borderRadius:20}`). Chama `exitEditMode()`.

**`AddWidgetsButton`** (M9606) está desligado pela flag: `PierreButton variant secondary rounded full size xl` com `calcAdd 20` + "Adicionar widgets" (`text-xl`), container `{alignItems:'center', paddingBottom:24}`. Nesta versão, "adicionar" abre por `handleOpenSheet`: analytics `add_widget_sheet_opened` + `router.push('/card/add-widget')`.

### 2.6 Montagem dos slots (`widgetSlots`, ~L3015140) [O]
```js
const byId = new Map(layout.widgets.map(w => [w.id, w]))      // resposta de GET /api/mobile/home-layout
const pending = new Set(pendingSyncIds)
const slots = {}
for (const id of DEFAULT_SORTABLE_ORDER) {
  if (pending.has(id))       slots[id] = { kind: 'pending', widgetId: id }
  else if (byId.get(id))     slots[id] = { kind: 'system', data: byId.get(id) }
  else                       slots[id] = { kind: 'empty' }
}
for (const w of aiWidgets /* ativos */) slots['ai-' + w.id] = { kind: 'ai', widgetId: w.id, widgetType: w.widgetType }
const systemIds = DEFAULT_SORTABLE_ORDER.filter(id => slots[id]?.kind === 'system')
useWidgetLiveData(systemIds)                                    // §4
```
A ordem na tela é a do `order` do store. O mapa só resolve o conteúdo de cada id.

### 2.7 Ordem, edição e persistência [O]

#### Store local
`useWidgetOrderStore` (M2198, L772733) usa zustand + `persist` + `createJSONStorage(AsyncStorage)` e espelha o store da Vision (`05b-leo-ia.md` §4.3).
- **Chave**: **`pierre-widget-order:<userId>`** (`getWidgetOrderStorageKey`, L772781). Sem usuário, usa `'guest'`; o nome inicial é `pierre-widget-order:guest`.
- **Troca de usuário**: `setWidgetOrderStoreUser(userId)` (L773045) lê a chave do usuário e a legada `pierre-widget-order`. Se a do usuário estiver vazia e a legada existir, copia a legada e a apaga. Depois chama `persist.setOptions({name})`, reseta o estado e chama `rehydrate()`.
- **Estado persistido** (`partialize`): `{order, hiddenWidgets, widgetSizes, pendingSync, aiWidgets, version, isDirty}`.
  - Padrão: `order = DEFAULT_SORTABLE_ORDER`, demais campos vazios, `version 0`, `isDirty false`.
  - Só em memória: `isEditMode`, `isContextMenuOpen`, `previewSize`, `previewOrder`, `lastAddedWidgetId`.
- **`onRehydrateStorage`** faz três ajustes:
  - acrescenta os ids novos de `DEFAULT_SORTABLE_ORDER` que não estão em `order` nem em `hiddenWidgets`;
  - remove de `order` os ids de sistema que saíram do registro;
  - sincroniza os `ai-*` com `aiWidgets[].isActive`.
- **Ações**:
  - ordem e visibilidade: `setOrder`, `resetOrder`, `hideWidget`, `showWidget` (marca `lastAddedWidgetId`), `fillGaps`;
  - tamanho: `setWidgetSize`, `setPreviewSize`/`clearPreviewSize`;
  - sincronização: `addPendingSync`/`clearPendingSync`, `applyServerLayout(order, hidden, sizes, version)` (zera `isDirty` e `pendingSync`), `markSynced(version)`, `applyReorganizedLayout(widgets, backgroundId)`;
  - widgets de IA: `addAIWidget`/`removeAIWidget`/`hardDeleteAIWidget`/`reactivateAIWidget`/`syncAIWidgetsFromServer`;
  - interface: `enterEditMode`/`exitEditMode`, `openContextMenu`/`closeContextMenu`.
  - Toda mudança de layout faz `version + 1` e `isDirty: true`.
- **Buracos na grade**: `hasGaps(order, sizes)` / `useHasGaps()` percorre a ordem contando os `half` (2 por linha). Há buraco quando um `half` fica sozinho antes de um wide ou full [I: regra lida pela estrutura do laço]. `fillGaps()` reordena para fechar esses buracos.

#### Servidor
`homeLayoutApi` (M9554, L2994452):

| Método | Endpoint | Uso |
|---|---|---|
| GET | `/api/mobile/home-layout` | `useHomeLayout` (M9555, L2994512). queryKey **`['home-layout','layout']`** (`homeLayoutKeys`), `staleTime` **43.200.000 ms (12 h)**, `gcTime` **86.400.000 ms (24 h)**. Se `!success`, lança `Error(error.message ?? 'Failed to load home layout')` |
| PATCH | `/api/mobile/home-layout` | corpo `buildPatchPayload(state)` (M9557) |
| POST | `/api/mobile/home-layout/reorganize` | "reorganizar com IA" (§2.8) |

- **Resposta do GET**: `{backgroundId, generatedAt, version, widgets:[{id, size, status, data}]}` (§1.3).
- **Cache offline**: AsyncStorage **`pierre:cardHomeLayout:<userId>`** (`asyncStorageSchema.cardHomeLayoutCache`, descrito como "Cached Card Home layout per user — keyed by userId to prevent cross-account data leaks on shared devices"). Ao montar, a tela lê o cache e faz `setQueryData`. Cada resposta nova é salva.
- **Quando chega uma resposta** (`useEffect` em `data`):
  1. limpa `pendingSync` dos ids que voltaram;
  2. `extractArrangementFromResponse` monta `order` (ids na ordem do servidor), `hiddenWidgets` (ids não fixos do registro que faltam na resposta) e `widgetSizes` (`size` de cada item);
  3. se o store **não** estiver `isDirty`, ou se a versão do servidor for maior, aplica `applyServerLayout` mantendo os `ai-*` locais. Liga `isLayoutSyncing` e desliga após **1000 ms** (`didSyncLayout = true`);
  4. se houver ids `ai-*`, chama `aiWidgetApi.list()` e `syncAIWidgetsFromServer`.
- **`buildPatchPayload(state)`**:
```js
{ version: state.version,
  arrangement: { widgets: [
    ...state.order.map((key, i) => ({ widgetKey: key, size: state.widgetSizes[key] ?? WIDGET_REGISTRY[key]?.size ?? 'half', position: i, visible: true })),
    ...state.hiddenWidgets.map(key => ({ widgetKey: key, size: state.widgetSizes[key] ?? WIDGET_REGISTRY[key].size, position: 999, visible: false })) ] } }
```
- **Quando o PATCH sai** (`useWidgetSync`, M9614, L3017534). Sempre com a trava global `acquireLayoutSyncLock()`/`releaseLayoutSyncLock()` (M9558):
  1. ao **sair do modo de edição** com `isDirty`: PATCH imediato;
  2. fora da edição, quando `order` ou `widgetSizes` mudam: *debounce* de **500 ms**, e o PATCH só sai se o store ainda estiver `isDirty`.
  - Sucesso: `markSynced(version)` (se a versão não mudou nesse meio-tempo) + `invalidateQueries(['home-layout','layout'])` + `syncAIWidgetDeactivations()`.
  - Erro: se não for **409**, chama `captureCardError('layout_sync', err, {operation:'edit_exit', status})`. Em qualquer caso, invalida a query do layout.
- **Recuperação** (`usePendingSyncRecovery`, M9615, L3018046):
  - quando o app volta para `active` (`AppState 'change'`) com `isDirty` ou pendências, refaz o PATCH, até **3 tentativas**;
  - logs: "recovery exhausted" e "recovery patch not accepted";
  - expõe `{isSyncing, syncFailed, pendingCount}`.
- **Analytics da edição**: `edit_mode_entered {widget_count}` e `edit_mode_exited {widget_count, did_reorder}`. O `did_reorder` compara `order.join(',')` de antes e de depois.

### 2.8 "Reorganizar com IA" (dormente no cartão) [O]
`useCardHomeScreen` tem `handleAIPress/handleAIConfirm/handleBloom/aiFabHidden/showAIConfirm/aiTheme`, mas o JSX de `CardHomeScreen` **não** renderiza `AIReorganizeFAB` (M9608) nem a confirmação. Só o `EdgeGlowLoading` (iOS) depende de `isAIReorganizing`. Se o fluxo for acionado:
1. analytics `ai_reorganize_tapped`;
2. luz `withTiming(1, 1200 ms, linear)` + háptico Light;
3. `POST /api/mobile/home-layout/reorganize`;
4. sucesso: `applyReorganizedLayout(data.widgets, data.backgroundId)` + `notificationAsync(Success)` + toast "Widgets reorganizados pela AI" (`ai_reorganize_success`);
5. erro: `notificationAsync(Error)` + "Não foi possível reorganizar. Tente novamente." (`ai_reorganize_failed`);
6. o efeito termina após 1300 ms.

### 2.9 Toque nos widgets [O]
- **Widgets de sistema**: o tipo de toque (`navigate` / `navigate-expand` / `expand`) e a rota ficam no `WidgetRenderer` (L2982600; `05b-leo-ia.md` §7.2).
- **Widgets de IA**: `handleWidgetTap(id, widgetType, …, size)` só registra o analytics `WIDGET_INTERACTION {action:'tapped', widget_id:'ai-'+id, widget_type:'ai', widget_size, position: order.indexOf('ai-'+id), page_name: CARD_PAGE_NAMES.CARD_HOME}`. Quem faz a expansão é o `AIWidgetWrapper` (`05b` §7.3).


## 3. `WidgetCarousel` (M7084, L2398930)

### 3.1 Onde é usado [O]
**Não faz parte da home do cartão.** É o carrossel de abas que o **chat** usa quando uma mensagem do assistente traz vários widgets de tool:
- A mensagem (M4931, `buildPartElement` L1960484) agrupa as partes de tool elegíveis (`isCarouselEligibleWidget`, `ALL_CAROUSEL_WIDGET_TOOLS` em M7082).
- Em seguida desenha `View style=toolRow` → `WidgetCarousel {tabs}` (key `carousel-<messageId>`), com um `ToolPartRenderer` por parte (L1961667).
- O rótulo de cada aba vem de `WIDGET_REGISTRY[tool].tabTitle` (o registro do chat, M6739, §1.1) ou de `SPECIAL_TOOL_TAB_TITLES` (M7083):

```js
{ renderCustomChart:'Gráfico', generateWidget:'Gráfico', generateTransactionChart:'Gráfico', getBill:'Fatura',
  getInvestments:'Investimentos', showInvestments:'Investimentos', showStatement:'Extrato', showBankInsights:'Insights',
  getSubscriptions:'Assinaturas', getInstallments:'Parcelas', getAgentExecutionLogs:'Relatório', getBankStatement:'Extrato',
  createDocument:'Documento', updateDocument:'Documento', requestSuggestions:'Sugestões', openFinanceConnectCta:'Conectar banco',
  openPricingCta:'Planos', openSupportCta:'Suporte', createCdbInvestment:'CDB' }
```
O fallback do rótulo é `'Widget'`.

Os `tabTitle` do M6739 são estes:

| Tool | Rótulo da aba |
|---|---|
| showBalanceCard | Saldo |
| showTransactionList | Transações |
| showAccountSummary | Contas |
| showPatrimony | Patrimônio |
| showCreditCardSummary | Cartões |
| showAlertCard | Alerta |
| showExpenseBreakdown | Gastos |
| showIncomeVsExpense | Rece. vs Desp. |
| showLoanSummary | Empréstimos |
| showInvestmentSummary | Investimentos |
| showCdbMiauPierre / showCdbGatinho | CDB |
| showSubscriptionList | Assinaturas |
| showComparisonCard | Comparação |
| showMetricHighlight | Métrica |
| showSpendingTrend | Tendência |
| showCategoryDeepDive | Categoria |
| showSpendingHeatmap | Mapa de gastos |
| showInstallmentTracker | Parcelas |
| showWeeklySummary | Semana |
| showSavingsOpportunities | Economia |
| showCategoryTrend | Gastos × mês |
| showAgentReport | Relatório |
| createAgent | Agente |

O `skeletonTitle` correspondente aparece em L2202996–2203090. A função `getWidgetComponent(tool)` devolve `WIDGET_REGISTRY[tool]` ou `null`.

### 3.2 Geometria e estrutura [O L2398207–2399200]
```
WidgetCarousel({children, tabs, slideWidth?})
  children.length <= 1 → <>{children}</>                       (sem abas)
  slideWidth informado → CarouselInner
  senão → View testID="widget-carousel-root" onLayout(w = Math.round(layout.width)) → CarouselInner (só quando w > 0)

CarouselInner({count, slideWidth, tabs, children})
View
├─ View {width: slideWidth, overflow:'hidden'}
│   └─ ScrollView horizontal testID="widget-carousel-tabs" accessibilityRole="tablist" showsHorizontalScrollIndicator=false
│        contentContainerStyle={flexDirection:'row', alignItems:'center', gap:4, paddingHorizontal:12, paddingVertical:8}
│        └─ CarouselTabItem × count  (testID "widget-carousel-tab-<i>")
└─ Animated.View testID="widget-carousel-clipper" style=[{overflow:'hidden'}, heightAnimStyle]
    └─ ScrollView ref testID="widget-carousel-pager" horizontal pagingEnabled showsHorizontalScrollIndicator=false
         contentContainerStyle={flexDirection:'row', alignItems:'flex-start'} onMomentumScrollEnd
         └─ View testID="widget-carousel-slide-<i>" style={width: slideWidth, overflow:'hidden'} onLayout(h = Math.round(layout.height))
```

**Aba (`CarouselTabItem`, L2398215)**: `Pressable accessibilityRole="tab" accessibilityState={selected: isActive}`.
- Pílula `TAB_PILL = {height:34, borderRadius:17}`.
- Aba **inativa**: `GlassContainer glassEffectStyle="regular" isInteractive` com a pílula. No Android, sem efeito de vidro, sobra só o container [I].
- Aba **ativa**: `Box` com `[TAB_PILL, {backgroundColor:'rgba(255,255,255,0.85)'}]`.
- Conteúdo: `Box className="px-3.5"` (px 12,25) `style={flex:1, alignItems:'center', justifyContent:'center'}` → `Text numberOfLines=1 allowFontScaling=false`.
  - className: `"text-[14px] font-medium " + (isActive ? '' : 'text-foreground')`, ou seja, Geist-Medium 14.
  - style: `{lineHeight:18, includeFontPadding:false}`, mais `{color:'#000000'}` quando ativa.
  - Inativa usa `text-foreground` (#fafafa).

### 3.3 Paginação, snap, animação e hápticos [O]
- **Snap**: o `pagingEnabled` nativo, com página = `slideWidth` (largura do container medida). Não há `snapToInterval` nem `decelerationRate` customizados.
- **Índice ativo**: no `onMomentumScrollEnd`, `i = clamp(Math.round(contentOffset.x / slideWidth), 0, count−1)`.
- **Toque na aba**: `goTo(i)` faz `i = clamp(i, 0, count−1)`, `setActive(i)` e `pager.scrollTo({x: i × slideWidth, animated:true})`.
- **Indicadores**: as próprias abas. **Não há pontos de paginação.**
- **Altura**: cada slide mede a própria altura (`onLayout`), guardada por índice em dois estados: o valor imediato e um valor "assentado" gravado após `setTimeout` de **250 ms** (debounce por índice; os timers são limpos ao desmontar).
  - O `clipper` anima a altura até a do slide ativo: `heightAnim.value = withTiming(h, {duration:250})`. Na primeira medição, o valor é atribuído direto, sem animação.
  - Worklet: `heightAnim.value === null ? {} : {height: heightAnim.value}`.
- **Hápticos**: nenhum. O módulo não importa `expo-haptics`.

---


## 4. `LiveWidgetRenderer`: dados dos widgets de sistema (complemento a 05b)

`05b-leo-ia.md` §7.2 já descreve os estados (`empty`/`error`/`ok`). Esta seção cobre o que falta: **de onde vem o dado** e como ele substitui o dado do layout.

### 4.1 Duas camadas de dado [O]
1. **Dado do layout (base)**: cada `widgets[i].data` do `GET /api/mobile/home-layout` (§2.7). Já vem formatado pelo servidor (ex.: `'R$ 1.234,56'`) e chega como `slot.data` → `LiveWidgetRenderer {widget: {...slot.data, size}}`.
2. **Dado "ao vivo"**: `useWidgetLiveData(systemIds)` (M9620, L3019784), chamado **uma vez** por `useCardHomeScreen` com os ids `system` presentes. Ele busca direto na API de crédito, transforma os dados no formato do widget e grava em `useWidgetLiveDataStore` (M9536, L2986408: `{entries, setEntry, setEntries, clearAll}`).

`LiveWidgetRenderer` (L2982377) lê `useWidgetLiveEntry(widget.id)` e decide:
- id fora do `Set` de ids vivos: usa o dado do layout como está;
- `entry.data` presente: `{...widget, status:'ok', data: entry.data}`, ou seja, **o dado ao vivo substitui o do layout**;
- `entry.isFetched && !data && !isError`: `{...widget, status:'empty'}`;
- `entry.isError`: mantém o widget se `widget.status === 'ok'` (fallback no dado do layout), senão `{...widget, status:'error'}`;
- `isLoading` passado ao `WidgetRenderer` = `!entry || (entry.isLoading && entry.data === undefined)` [I: lido pela sequência de testes].

### 4.2 Fontes por widget (`useWidgetLiveData`) [O]
Os hooks da API de crédito ficam em M4077 (L1646529). As chamadas são **diretas** para a API de pagamentos da CloudWalk, não para o servidor do Pierre: base `getInfinitePayApiUrl()`, token de portador do cartão (M3071) e `isCorePaymentApiConfigured && creditAccountAllowsRequest` como pré-condição de todo `enabled`.

| Widget | Hook | Endpoint (M4103, L1659340+) | queryKey (`cardKeys`) | `staleTime` / `gcTime` | `enabled` | Transformação (M9621) |
|---|---|---|---|---|---|---|
| `available-limit` | `useCreditAvailable` | `GET /credit/limit?product=pierre` | `['cards', …, 'available']` | 0 / 600.000 | ids tem `available-limit` | `transformAvailableLimit` → `{availableAmount, totalLimit, usedAmount, usedPercentage}` |
| `cashback` | `useCashbackSummary` | `GET /credit/cashback/summary` | `['cards', 'cashback', 'summary']` [I] | 0 / 600.000 | ids tem `cashback` | `transformCashback` → `{amount, pendingAmount, availableCents, pendingCents}` (`formatCashbackTotal`/`formatCashbackPendingTotal`) |
| `invoice` | `useCurrentInvoice` + `useInvoices` | `GET /credit/invoices/current?product=pierre` + `GET /credit/invoices?product=pierre` | `[…, 'invoice-current']`, `[…, 'invoices']` | 0 / 600.000 e 0 / 1.800.000 | ids tem `invoice` (a atual também com `installments`) | `transformInvoice` → `{monthLabel, amount, closingDate, bars:[{value,isCurrent}]}` |
| `installments` | `useCurrentInvoice` → `useInvoiceItems(invoiceId)` | `GET /credit/invoices/items?…` | `[…, 'invoice-items', id]` | 0 / 600.000 | `installments` **e** fatura atual com `id` | `transformInstallments` → `{count, subtitle, paidPercentage, paidAmount, totalAmount, items:[{id,name,amount,installmentLabel,progressPercentage,logoUrl}]}` |
| `monthly-spending`, `recent-transactions`, `spending-by-category`, `subscriptions` | `useStatements({startDate, endDate, limit:100})`, com o mês corrente vindo de `currentCalendarMonthRange()` | `GET /credit/statements/v2?…` | `[…, 'statements', params]` | 0 / 600.000 | algum desses 4 ids presente | `transformMonthlySpending` → `{amount, data:[{value,label}]}`; `transformRecentTransactions` → `{transactions:[{name, category, amount, date, iconName:'arrowCircleUpRight', iconColor, pierreIcon, logoUrl, variant:'default'\|'cashback'\|'denied'}]}`; `transformSpendingByCategory` → `{amount, categories:[{color,value,name,amount,percentage,iconKey}], subtitle, period:'este mês'}`; `transformSubscriptions` → `{logos:[{name,logoUrl,amount}], amount, subtitle}` |
| `cdb-position` | `useVisionSystemWidgetData('cdb-position', {directFetcher: fetchCdbDailyLiquidityPositionDirect, accessGate: useCdbWidgetIdentityGate(), enabled}, 'full', CDB_POSITION_DAILY_LIQUIDITY_VARIANT)` | busca direta da posição CDB (liquidez diária); sem `directFetcher`, seria `GET /api/mobile/vision/widgets/cdb-position/data` (`01-carteira-saldo-fluxo.md` §1) | `visionWidgetDataKeys…` (M8526) | M8526: `gcTime` 600.000, `networkMode 'always'`, cache local `pierre:visionWidgetItemData` | ids tem `cdb-position` | `shapeCdbPositionData` (M9055) |

Detalhes das transformações:
- Strings fixas: "Nenhuma parcela", "Nenhuma assinatura", "Parcela ", " de ", "Parcelamento", "Em andamento", " • Não aprovada", " • Limite liberado", " categoria"/" assinatura" (+ "s" no plural), "este mês", `R$ 0,00`.
- Meses abreviados: Jan…Dez (`MONTH_NAMES`, M4856).
- Moeda: `Intl.NumberFormat('pt-BR', {style:'currency', currency:'BRL'})`.
- Paleta das categorias (`spending-by-category`): `#3B82F6 #8B5CF6 #EC4899 #F97316 #14B8A6 #F59E0B #EF4444 #84CC16 #6366F1`; a sobra vira `'outros'`.
- Transação de cashback: `iconColor` e `pierreIcon` = `#95FF48` / `arrowCircleUpRight`.

O detalhe visual de cada widget fica com o agente 6b.

Cada resultado vira uma entrada `{data: transform(query.data), isLoading, isError, isFetched}`, gravada por `useEffect` com `setEntry(id, …)` (statements: uma entrada por id, via `setEntries`).

### 4.3 Atualização [O/I]
- Todas as queries do cartão têm **`staleTime: 0`**. Elas refazem a busca a cada montagem do `useWidgetLiveData` (ex.: ao voltar para a aba do cartão) e nos gatilhos padrão do react-query (reconexão; foco do app só se o `focusManager` estiver ligado ao `AppState`) [I].
- Não há `refetchInterval` nesses hooks. Entre os módulos vistos, só o PIX da fatura usa `refetchInterval`.
- O layout (`home-layout`) é invalidado após cada PATCH bem-sucedido ou com erro (§2.7). Ele dura 12 h em cache "fresco".
- Widgets de IA: `useAIWidgetData` (`staleTime` 0 se `autoRefresh`, senão 3 h; ver `05b` §7.3).

---


## 5. Dúvidas abertas
1. **`last-purchase` no registro**: está como `fixed` em M2199 e no `WIDGET_SURFACE_CATALOG`, mas nenhum render na grade. Hoje a "última compra" é zona contextual. Pode ser resto de versão anterior.
2. **Três catálogos divergentes**: M2199 (grade), M9756 (`WIDGET_SURFACE_CATALOG`, tela de adicionar) e o catálogo do servidor. Não ficou claro qual vence na tela "adicionar widget" do cartão quanto a `monthly-spending` (full × wide) e à presença de `cashback`/`cdb-position`. Fica para quem documentar `CardAddWidgetScreen` (M9751).
3. **Regra exata de `hasGaps`/`fillGaps`** e posição de inserção de widgets novos: lida pela estrutura do laço, sem o algoritmo linha a linha.
4. **Quem aciona "reorganizar com IA" no cartão**: os handlers existem, mas nada no JSX atual chama `handleAIPress`. Talvez seja controlado por um flag remoto, ou só exista na Vision (`VisionAIReorganizeFAB`).
5. **`CardHomeScreen` fora do hub**: o barrel M9748 o exporta, mas não achei rota `./card/index`. Com `embedded=false`, o `CardTopHeader` apareceria.
6. **Foco/refresh**: se o app liga o `focusManager` do react-query ao `AppState`, as queries com `staleTime 0` refazem a busca ao voltar ao app. Não verificado.
7. **Aba inativa do `WidgetCarousel` no Android**: `GlassContainer` sem vidro. A cor de fundo efetiva da pílula inativa (provavelmente transparente) não foi confirmada.
8. **Constantes `200`, `20`, `240`, `52`, `12`, `120` do `JiggleWrapper`**: os nomes não aparecem nos worklets. Pelo paralelo com a Vision, devem ser tamanhos e limites do resize (ex.: `MAX_DRAG`).
