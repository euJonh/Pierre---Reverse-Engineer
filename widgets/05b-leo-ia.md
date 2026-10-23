# W5b — Widgets Leo e de IA da home "Vision" (Pierre v4.0.6)

> Análise estática do bundle Hermes v98 (`_trabalho/out.js`). **[O]** = observado (`Lnnnn` = linha de `out.js`); **[I]** = inferido.
> Base compartilhada (shell, grid, alturas, estados, moeda, privacidade, tokens): ver `widgets/00-base.md`. Busca de dados dos widgets de sistema: §1 de `widgets/01-carteira-saldo-fluxo.md`. Agentes de IA (modelo, API, logs): `agentes-pierre.md`.


---

## 0. Mapa rápido [O]

| Widget | id | Tamanhos | Padrão | Toque | Componentes (linha) |
|---|---|---|---|---|---|
| Leo | `leo-agent` | só `half` | half | registro diz `navigate` → `/(protected)/investments/agents/leo-cdb`, mas a home abre o chat com o Leo (§1.1) | `LeoAgentAvatar` L2876406, `LeoAgentWidget` L2876665 (memo), `LeoAgentWidgetConnected` L2876986 |
| Insights do Leo | `leo-insight` | só `wide` | wide | `none` (só o botão "Saiba mais") | `LeoInsightAvatar` L2877110, `LeoInsightWidget` L2877353 (memo), `LeoInsightWidgetConnected` L2877579, `useLeoInsightReport` L2877958 |
| IA (gerado) | `ai-<uuid>` | `half`, `wide`, `full` | `size` escolhido na criação | só háptico + analytics (sem navegação, §3.1) | `AIVisionWidget*` (L2835635…), `LiveWidgetRenderer` L2982377, `AIWidgetRenderer` L2988473 |

Registro `VISION_WIDGET_REGISTRY` (L772611/L772615):
```js
{ id:'leo-agent',   size:'half', supportedSizes:['half'], label:'Leo',             tapAction:'navigate', navigateTo:'/(protected)/investments/agents/leo-cdb', insertPosition:'top' }
{ id:'leo-insight', size:'wide', supportedSizes:['wide'], label:'Insights do Leo', tapAction:'none',                                                       insertPosition:'top' }
```
Catálogo do picker (L3074968/L3074974): `{id, label, supportedSizes, defaultSize, surfaces:['vision'], onDemand:true}`.

Regras do registro que afetam estes widgets (módulo L771800–772660) [O]:
- `ON_DEMAND_PICKER_WIDGET_IDS = ['cdb-position','cdb-performance','leo-agent','leo-insight']`; `isVisionWidgetOnDemand(id)` = `id.startsWith('ai-') || ON_DEMAND.includes(id)`. Ou seja: Leo, Insights e **todo widget de IA** são "sob demanda" (não entram no layout padrão `DEFAULT_VISION_SORTABLE_ORDER`; são inseridos por evento ou pelo usuário).
- `PICKER_WIDGET_IDS` = (ordem padrão + on-demand) filtrado por `!id.startsWith('ai-') && !VISION_PICKER_RETIRED_WIDGET_IDS.includes(id)` (aposentados: `['cards','cdb-performance']`). Os de IA não aparecem no picker de sistema.
- `getVisionSupportedSizes(id)`: `ai-*` → `VISION_WIDGET_SIZE_ORDER = ['half','wide','full']`; demais → `registry[id].supportedSizes ?? ['half']`.
- `isVisionResizable(id)`: `ai-*` → `true` (3 tamanhos); Leo e Insights → `false` (um tamanho só, então não aparece alça de redimensionar).
- `insertPosition:'top'` nos dois Leo: ao inserir, entram no topo do grid (`DEFAULT_VISION_INSERT_POSITION = 'bottom'` para os demais).
- No renderer (ver `00-base.md` §2.3): `leo-insight` recebe só `animate` (sem `onPress`, sem `size`); `leo-agent` recebe `onPress, animate`.

### 0.1 Mídia animada (vídeo do avatar) — compartilhada pelos dois Leo [O]
Módulo `VisionHomeMediaActivity` (M9059, L2875389):
- `getVisionHomeAnimatedMediaRect(id, size)`: `leo-agent` → `LEO_AGENT_MEDIA_RECT = {offsetTop:25, height:50}`; `leo-insight` → `LEO_INSIGHT_MEDIA_RECT = {offsetTop:16, height:40}`; `cdb-position` (só se `size==='half'`) → `{offsetTop:16, height:40}`; outros → `null`.
- `isVisionHomeAnimatedMediaWidget(id, size)` = rect ≠ null. Só esses widgets ligam `onLayout` no `VisionWidgetItem` (ver `00-base.md` §3.2) para saber a posição na tela.
- `isVisionWidgetInMediaViewport({itemTop,itemHeight,viewportTop,viewportHeight})` (worklet): `false` se altura ≤ 0; senão `overlap = max(0, min(itemTop+itemHeight, viewportTop+viewportHeight) − max(itemTop, viewportTop))` e visível se `overlap ≥ min(MIN_VISIBLE_MEDIA_PIXELS=80, itemHeight/2)`.
- `VisionHomeMediaActivityProvider({active})` = Context (default `true`); `useVisionHomeMediaActive()` lê o contexto. O grid passa `active` por item (só toca o vídeo do widget que está visível na rolagem).
- `useVisionAnimatedMediaSource(source, active)` → `{ source: active ? source : undefined, visible: active && source != null }`.

Fonte do vídeo (M6904, L2329433, `getAgentVideoSourceBySlug('leo-cdb')`): mapa `{'albert-daily','marie-monthly','galileu-weekly','leo-cdb'}` → `require` lazy do asset `src/assets/videos/agents/leo.mp4` (L2329861, hash `ac8d3f018ecf097b940c636e26765155`). Imagem estática: `src/assets/images/agents/leo.webp`, 1250×1250 (M4716, L1876080, hash `39c98c0eeaeda977fd47efa9b6c57497`). No APK Android os assets ficam em `res/drawable*/src_assets_images_agents_leo.webp` e `res/raw/src_assets_videos_agents_leo.mp4` [I: convenção do Metro/Expo].
- O vídeo **não** é usado se `isLowEndDevice()` (M2017, L677874) ou se `canRenderAgentVideo(src)` = `src != null` for falso.
- `AgentAnimatedVideoLayer` (M6924, L2330769): camada com `Animated.View pointerEvents="none"` cuja `opacity` vai de 0 → 1 (`Animated.timing`, `toValue 1`, `duration = transitionMs` default **180**, `useNativeDriver:true`) quando `visible && source`; volta a 0 ao esconder. Dentro, `AgentAnimatedVideo` (M6925, L2330244) = `VideoView` do **expo-video** (`contentFit:'cover'`, `nativeControls:false`, `allowsPictureInPicture:false`, `focusable:false`, `accessible:false`), player `useVideoPlayer` com `muted:true`, `allowsExternalPlayback:false`, `audioMixingMode:'mixWithOthers'`, `loop:true` e `resetOnPause:false` nos Leo. O vídeo fica **por cima** da imagem estática; com fade de 180 ms a imagem é "substituída" pelo vídeo.


---

## 1. `leo-agent` — "Leo" (half)

### 1.1 Identidade e dados [O]
- `LeoAgentWidgetConnected({onPress, animate=false})` (L2876986): `isMediaActive = useVisionHomeMediaActive()` → `<LeoAgentWidget onPress animate isMediaActive/>`. **Sem query, sem endpoint**: conteúdo 100% estático.
- Constantes do módulo (M9060): `LEO_AGENT_NAME = 'Leo'`, `LEO_AGENT_ROLE = 'Agente de investimentos'`, `LEO_AGENT_CTA_LABEL = 'Conversar'`, slug `'leo-cdb'`, raio 24, altura 200, `LEO_AGENT_AVATAR_SIZE = 50` (de `LEO_AGENT_MEDIA_RECT.height`).
- Toque: o widget inteiro é o `Pressable` do `WidgetShell` (háptico Light → `onPress`). O "botão" Conversar **não tem onPress próprio**: é só visual, o toque cai no shell.
- **Destino real do toque** (handler de toque da home em `useVisionHomeScreen`, L2892856; trecho L2894470): o caso `leo-agent` é tratado **antes** do `navigateTo` do registro e chama `cdbLeoChatHandoff()` **sem argumentos** (ver §2.3). Resultado: se o agente Leo já existe em `GET /api/agents`, abre o **chat** (`/chat`) com a mensagem `Oi Leo, pode me tirar uma dúvida?` e o Leo mencionado; se não existe, abre a tela do agente `/investments/agents/[agentId]` (`agentId:'leo-cdb'`, `from:'investment-success'`). O `navigateTo: '/(protected)/investments/agents/leo-cdb'` do registro **não é usado** por esse handler [O].
- Antes de navegar, a home registra `trackWidgetInteraction('tapped', {widget_id, widget_type: id.startsWith('ai-') ? 'ai' : 'system', widget_size, chart_variant (só monthly-spending), position})`.
- Sem "Ocultar dados" (não há valores), sem estados vazio/erro/carregando.

### 1.2 Árvore e medidas (Android, caminho opaco) [O L2876665–2876960]
```
WidgetShell(inner, useGlass, animate, glassStyle={borderRadius:24,height:200}, widgetRadius=24, onPress, handlePress, testID='leo-agent-widget')
 └─ VStack className="items-center justify-center gap-3 p-4 bg-surface"      // gap 10,5 · padding 14 · #18181b
           style = glass ? {flex:1} : [{borderRadius:24, height:200}]   testID='leo-agent-widget'
     ├─ LeoAgentAvatar(isMediaActive)                                      // 50×50
     │   └─ Box collapsable=false className="rounded-2xl border border-border" style={{width:50,height:50}}
     │       └─ Box collapsable=false className="rounded-2xl overflow-hidden" style=absoluteFill
     │           ├─ expo-image Image source=leo.webp style=absoluteFill contentFit="cover" testID='leo-agent-widget-avatar'
     │           └─ AgentAnimatedVideoLayer source visible loop resetOnPause={false} style=absoluteFill
     ├─ VStack className="items-center gap-0.5"                             // gap 1,75
     │   ├─ Text "text-sm font-medium text-foreground" numberOfLines=1 allowFontScaling=false testID='leo-agent-widget-name' → 'Leo'
     │   └─ Text "text-xs text-muted-foreground"     numberOfLines=1 allowFontScaling=false testID='leo-agent-widget-role' → 'Agente de investimentos'
     └─ Box className="bg-surface" style=conversarButton
         └─ HStack className="items-center justify-center gap-2" testID='leo-agent-widget-chat'   // gap 7
             ├─ PierreIcon name="chatChat" size=18 color={colors.pierre /* #c7ff00 */} strokeWidth=2.2
             └─ Text "text-sm font-medium text-foreground" numberOfLines=1 allowFontScaling=false → 'Conversar'
```
`conversarButton = StyleSheet.create({alignSelf:'stretch', height:40, borderRadius:20, alignItems:'center', justifyContent:'center', overflow:'hidden'})`.

| Item | Valor |
|---|---|
| Caixa | half: `(W−50)/2` × **200**, raio 24, `#18181b`, sem borda/sombra |
| Alinhamento | tudo centralizado vertical e horizontalmente (`items-center justify-center`), gap 10,5 entre os 3 blocos |
| Avatar | 50×50, `rounded-2xl` = raio **14**, borda 1 px `border-border` = **#27272a** (a borda fica no Box externo; o interno recorta a imagem com o mesmo raio) |
| "Botão" Conversar | largura total útil (alignSelf stretch → `(W−50)/2 − 28`), altura 40, raio 20 (pílula), fundo `bg-surface` #18181b — **igual ao fundo do widget no Android**, então visualmente só aparecem ícone + texto [O; no iOS glass o fundo surface destaca a pílula] |
| Altura do conteúdo | 50 + 10,5 + (≈19 + 1,75 + 14) + 10,5 + 40 ≈ 145,75 → sobra ≈ 54 dividida em cima/embaixo (avatar começa em y ≈ 27; o `MEDIA_RECT.offsetTop = 25` é a aproximação usada para o viewport de vídeo) [I: lineHeight de `text-sm` não definido, RN usa ≈ 1,2×16] |

### 1.3 Tipografia
| Texto | Classe | Fonte | Size / LH | Cor |
|---|---|---|---|---|
| `Leo` | `text-sm font-medium text-foreground` | Geist-Medium | 16 / padrão RN | #fafafa |
| `Agente de investimentos` | `text-xs text-muted-foreground` | Geist-Regular | 10,5 / 14 | #a1a1aa |
| `Conversar` | `text-sm font-medium text-foreground` | Geist-Medium | 16 / padrão | #fafafa |

Ícone `chatChat` (balão de conversa do pacote de ícones Pierre), 18 px, traço 2,2, cor `THEME_TOKENS.dark.pierre` = **#c7ff00**.

### 1.4 Animação
- Nenhuma animação de entrada própria; `animate` só é repassado ao `WidgetShell`.
- Vídeo `leo.mp4` em loop sobre a foto quando o widget está ≥ 80 px (ou metade) visível na rolagem e o aparelho não é "low end"; fade-in 180 ms (§0.1).

---

## 2. `leo-insight` — "Insights do Leo" (wide)

### 2.1 Dados: `useLeoInsightReport()` (M9071, L2877958) [O]
Não usa `/api/mobile/vision/widgets/{id}/data`. Lê os **agentes do usuário** e o **último relatório do Leo**:
1. `useAgents()` (M4414, L1780647) → `GET /api/agents` (queryKey `agentKeys.list` = `['agents','list',…]`; resposta `{agents, usage}`, ver `agentes-pierre.md` §1.2).
2. `leo = agents.find(a => a.defaultSlug === 'leo-cdb')`; `agentId = leo?.id ?? null`.
3. `useAgentLogs(agentId)` (L1780710) → `GET /api/agents/{agentId}/logs` (`agentKeys.logs`; desabilitada sem id).
4. `findLatestInsight(logs)` (L2877896): percorre os logs **na ordem recebida** e devolve o primeiro em que `reportLabel(log.evaluationResult)` não é `null` → `{reportId: log.id, label}`; senão `null`. [I: o servidor devolve do mais recente para o mais antigo.]
   - `parseReportSubject(str)` (L2877825): `JSON.parse(str)` → `{subject: typeof subject==='string' ? subject : null, summary: typeof summary==='string' ? summary : null}`; em exceção `{subject:null, summary:null}`.
   - `reportLabel(str)`: `subject?.trim() || summary?.trim() || null` (o assunto tem prioridade; o `content` do log é ignorado).
5. Retorno:
   - com insight: `{reportId, description: label, subject: label, isLoading, isError}`;
   - sem insight: `{reportId: null, description: LEO_INSIGHT_DEFAULT_DESCRIPTION, subject: null, isLoading, isError}`.
   - `isLoading = agentsLoading || (agentId !== null && logsLoading)`; `isError = agentsError || (agentId !== null && logsError)` [O estrutura; I a combinação exata dos `&&`].
- Texto padrão: **`Dia de salário na conta? Entenda como separar parte do seu orçamento para ele trabalhar por você`**.

### 2.2 `LeoInsightWidgetConnected({animate=false})` (L2877579) [O]
```
{reportId, description, subject, isLoading, isError} = useLeoInsightReport()
isMediaActive = useVisionHomeMediaActive()
handoff = useCdbLeoChatHandoff()
onCtaPress = () => handoff({ reportId, reportSubject: subject })
isLoading → WidgetGlassCard animate style=stateCard testID='leo-insight-widget-loading' → <WidgetLoadingState variant="content"/>
isError   → WidgetGlassCard animate style=stateCard testID='leo-insight-widget-error'   → <WidgetErrorState compact={false}/>   // 'Falha ao sincronizar dados'
senão     → <LeoInsightWidget description onCtaPress animate isMediaActive/>
```
`stateCard = {borderRadius:24, width:'100%', minHeight:160, padding:16, alignItems:'center', justifyContent:'center'}` (padding **16**, não 14). Estados de vazio não existem: sem relatório, mostra o texto padrão.

### 2.3 CTA "Saiba mais": `useCdbLeoChatHandoff()` (M4713, L1875674) [O]
```ts
const router = useRouter(); const { data } = useAgents();
return ({ reportId, reportSubject } = {}) => {
  const leo = data?.agents.find(a => a.defaultSlug === 'leo-cdb');      // CDB_LEO_AGENT_ID
  if (!leo) { router.push(getCdbLeoAgentHref()); return; }             // agente ainda não existe → tela do Leo
  const img = getAgentImageUrl({ isDefault: true, defaultSlug: 'leo-cdb', name: 'Leo' });
  router.push({ pathname: '/chat', params: {
    message: 'Oi Leo, pode me tirar uma dúvida?',
    agentMentionId: leo.id, agentMentionSource: 'agent_page',
    agentEditId: leo.id, agentEditName: 'Leo',
    ...(img && { agentEditImageUrl: img }),
    ...(reportId && { reportMentionId: reportId }),
    ...(reportSubject && { reportMentionSubject: reportSubject }),
    key: `agent-conversar-leo-cdb-${Date.now()}`,
  }});
};
```
`getCdbLeoAgentHref(videoStartAt?, agentId?)` (L1875935) → `{pathname:'/investments/agents/[agentId]', params:{agentId: agentId ?? 'leo-cdb', from:'investment-success', videoStartAt?: n.toFixed(2)}}`.
No chat, `reportMentionId` vira o part `data-agent-report-mention` (ver `agentes-pierre.md` §1.3/§1.4): a conversa abre já "mencionando" o relatório.

### 2.4 Árvore e medidas (`LeoInsightWidget`, L2877353) [O]
```
WidgetGlassCard animate style=card testID='leo-insight-widget'          // Android: VStack "bg-surface" style=card
 └─ VStack className="flex-1 p-4 gap-2"                                   // padding 14, gap 7
     ├─ HStack className="items-center gap-2"                              // gap 7
     │   ├─ LeoInsightAvatar(isMediaActive)                                // 40×40
     │   │   └─ Box collapsable=false "rounded-xl border border-border" style={{width:40,height:40}}
     │   │       └─ Box collapsable=false "rounded-xl overflow-hidden" absoluteFill
     │   │           ├─ Image leo.webp absoluteFill contentFit="cover" testID='leo-insight-widget-avatar'
     │   │           └─ AgentAnimatedVideoLayer source visible loop resetOnPause={false} absoluteFill
     │   └─ Text allowFontScaling=false "text-sm text-muted-foreground" numberOfLines=1 testID='leo-insight-widget-name' → 'Leo'
     ├─ Text allowFontScaling=false "text-base text-foreground" numberOfLines=3 testID='leo-insight-widget-description' → {description}
     └─ Box className="self-start pt-1"                                    // paddingTop 3,5
         └─ PierreButton variant="primary" size="small" onPress={onCtaPress} testID='leo-insight-widget-cta' → 'Saiba mais'
```
`card = {borderRadius:24, width:'100%', minHeight:160}`.

| Item | Valor |
|---|---|
| Caixa | largura total (`W−32`), **altura pelo conteúdo com mínimo 160** (não usa `WidgetWide`, então **não** é fixa em 170), raio 24, `#18181b`, sem borda/sombra |
| Avatar | 40×40, `rounded-xl` = raio **10,5**, borda 1 px #27272a |
| Botão "Saiba mais" | PierreButton primary/small: pílula (raio 9999), fundo `--primary` **#fafafa**, texto `--primary-foreground` **#18181b**, padding 10,5 (horizontal) × 7 (vertical), Geist-Medium 14/21 → altura ≈ 35; alinhado à esquerda (`self-start`); spring de press do design system (damping 15, stiffness 300, mass 0,5) |
| Altura típica | 14 + 40 + 7 + (3×21 = 63) + 7 + 3,5 + 35 + 14 = **183,5** com descrição de 3 linhas; com 2 linhas 162,5; o mínimo 160 só atua com 1 linha |
| Toque | o card não é clicável (`tapAction:'none'`); só o botão |

| Texto | Classe | Fonte | Size / LH | Cor |
|---|---|---|---|---|
| `Leo` | `text-sm text-muted-foreground` | Geist-Regular | 16 / padrão | #a1a1aa |
| descrição (≤ 3 linhas, reticências no fim) | `text-base text-foreground` | Geist-Regular | 14 / 21 | #fafafa |
| `Saiba mais` | (PierreButton small) | Geist-Medium | 14 / 21 | #18181b |

Estados (carregando/erro) usam `stateCard` (mesmo raio/mínimo, padding 16, conteúdo centralizado); ver `00-base.md` §9 para o skeleton e o ícone/texto de erro. Observação: dentro de `alignItems:'center'`, as barras do skeleton com `width:'100%'` dependem da largura do container animado [I: pode encolher; para réplica, dê `alignSelf:'stretch'` ao skeleton].

---

## 3. Widgets de IA na Vision: `ai-<uuid>` (`AIVisionWidget*`)

### 3.1 Cadeia de renderização [O]
1. `VisionWidgetRendererInner` (ver `00-base.md` §2.3): se `widget.id.startsWith('ai-')`, procura `aiWidgets.find(w => w.id === widget.id.slice(3))` no `useVisionWidgetOrderStore`. Não achou → `null`. Achou → `<AIVisionWidgetConnected id title widgetType size onPress animate fallbackData={aiWidget.cachedData}/>`.
2. `AIVisionWidgetConnected` (memo, L2834732, M8974):
   ```ts
   const { data, isLoading } = useAIVisionWidgetData(id, { fallbackData });
   <AIVisionWidget id title widgetType data={data} size onPress animate
                   isLoading={data == null && isLoading}/>      // só "carregando" sem nenhum dado (nem cache)
   ```
3. `AIVisionWidget` (memo, L2835635): `size === 'full' ? <AIVisionWidgetFull …/> : <AIVisionWidgetHalf …/>`. **Atenção: `wide` usa o componente Half** (`Widget2x2`, altura 200, na largura total `W−32`) [O]. `widgetType` é recebido e **ignorado** na Vision: a forma de desenhar depende só do formato de `data`.

Toque: o grid passa `onPress`, então há `Pressable` + háptico Light. Mas o handler da home (`useVisionHomeScreen`, L2894420–2894600) só registra `trackWidgetInteraction('tapped', {widget_type:'ai', …})` e depois procura `VISION_WIDGET_REGISTRY[id]`; como `ai-*` não está no registro, **não navega para lugar nenhum** [O]. A tela de detalhe `/(protected)/vision/widget-detail?…` (`VisionWidgetDetailScreen`, L3339772) é aberta só pela tela "adicionar widget" (L3232365, evento `settings_opened`).

### 3.2 Normalização dos dados (comum a Half e Full) [O L2835093–2835200]
```ts
function toRows(data) {
  if (data && 'accounts' in data && Array.isArray(data.accounts))
    return { rows: data.accounts.map(a => ({ label: a.label, formatted: a.formattedBalance, percentage: a.percentage, color: a.color })),
             total: data.totalBalance };
  if (data && 'items' in data && Array.isArray(data.items))
    return { rows: data.items.map(i => ({ label: i.label, formatted: i.formattedValue, percentage: i.percentage, color: i.color })),
             total: data.total };
  return null;
}
const period = data && 'period' in data ? data.period : undefined;
```
Ou seja, o widget de IA da Vision aceita dois formatos de payload ("comparação de contas" e "comparação de itens/categorias"). Valores já chegam **formatados pelo servidor** (`formattedBalance`/`formattedValue`, `total`/`totalBalance` são strings exibidas como estão), assim como `percentage` (0–100) e `color` (CSS).

### 3.3 `AIVisionWidgetHalf` (half **e** wide; L2835093) [O]
```
isLoading:
  Widget2x2 testID='ai-vision-widget-{id}' visual={<WidgetLoadingState variant="visual"/>} label={title} value='' subtitle='' animate
senão:
  Widget2x2 testID='ai-vision-widget-{id}'
            visual = parsed ? <ComparisonVisual rows={parsed.rows}/> : <SparklesIcon/>
            label  = title
            value  = parsed?.total ?? '—'          // travessão U+2014
            subtitle = period ?? 'Insight IA'
            onPress animate
```
Defaults do `Widget2x2` (ver `00-base.md` §5): chevron 16 #71717a no topo direito, visual no topo, rótulo `text-base` #a1a1aa, valor `text-xl font-extrabold` #fafafa (autoshrink 0,6), subtítulo `text-base` #a1a1aa. **Sem "Ocultar dados"**: o valor vai como string para `PrivacyText` sem máscara (o widget de IA ignora `hideValues`) [O: nenhum uso de `useHideValues` no módulo].

`ComparisonVisual({rows})` (memo, L2834974):
- `rows.length < 2` → `null` (sem visual; o topo fica só com o chevron).
- senão → `VStack className="gap-2 w-full" pointerEvents="none"` com as **3 primeiras** linhas (`rows.slice(0,3)`), cada uma (`_temp`, key `{label}-{index}`):
```
VStack "gap-0.5"                                                  // gap 1,75
 ├─ HStack "justify-between items-center"
 │   ├─ Text allowFontScaling=false "text-xs text-muted-foreground" numberOfLines=1 → {label}            // Geist-Regular 10,5/14 #a1a1aa
 │   └─ Text allowFontScaling=false "text-xs font-semibold text-foreground" → {formatted}                // Geist-SemiBold 10,5/14 #fafafa
 └─ View style=track {height:6, borderRadius:3, backgroundColor:'rgba(128,128,128,0.15)', overflow:'hidden'}
     └─ View style=[fill {height:6, borderRadius:3}, {width: `${percentage}%`, backgroundColor: color}]
```
Altura do visual com 3 linhas: 3 × (14 + 1,75 + 6) + 2 × 7 = **79,25** (gap-2 = 7). Barras sem animação.

`SparklesIcon` (memo, L2835042): `View className="bg-muted" testID='ai-vision-widget-icon' style={{width:44, height:44, borderRadius:22, alignItems:'center', justifyContent:'center'}}` → `PierreIcon name="jimSparkle" size=20 color="#71717a"`. Fundo `bg-muted` = **#27272a**.

### 3.4 `AIVisionWidgetFull` (L2835376) [O]
```
isLoading:
  WidgetFull testID='ai-vision-widget-{id}' title={title} showChevron={false} animate → <WidgetLoadingState/>   // variante 'content'
senão:
  WidgetFull testID='ai-vision-widget-{id}' title={title} onPress animate      // chevron padrão = true
   └─ parsed
       ? Box className="px-4 pb-4 w-full"                                        // padding-x 14, padding-bottom 14 (além do p-4 do WidgetFull)
           ├─ ComparisonVisual rows                                              // até 3 linhas; null se < 2
           └─ HStack className="justify-between items-center mt-3"               // marginTop 10,5
               ├─ Text allowFontScaling=false "text-sm text-muted-foreground" → 'Total'          // Geist-Regular 16 #a1a1aa
               └─ Text allowFontScaling=false "text-sm font-bold text-foreground" → {total}       // Geist-Bold 16 #fafafa
       : Box className="px-4 pb-4"
           └─ Text allowFontScaling=false "text-sm text-muted-foreground" → 'Carregando dados do widget...'
```
Observações:
- O conteúdo fica com recuo horizontal total de **28** (14 do `WidgetFull` + 14 do `px-4`) e 28 embaixo [O].
- No full o `period` não é exibido.
- Não existe estado de erro: com erro de rede e sem cache, `isLoading` vira `false`, `data` é `undefined` e o full mostra "Carregando dados do widget..." para sempre (o half mostra o ícone de brilho, `—` e "Insight IA") [O].

### 3.5 Dados: `useAIVisionWidgetData(id, {fallbackData, enabled=true})` (M8987, L2838703) [O]
| Item | Valor |
|---|---|
| Endpoint | `GET /api/mobile/vision/widgets/{id}/ai-data` (com `?refresh=true` se `loadData(id, true)`; o hook chama sem refresh) — `aiVisionWidgetApi.loadData` (M8988, L2839082) |
| Resposta | `{ success, data, error? }`; `!success || !data` → `throw new Error(error?.message ?? 'Failed to load vision widget data')` (classe derivada de `Error`) |
| queryKey | `aiVisionWidgetDataKeys.data(id)` = `['ai-vision-widget-data', id]` |
| staleTime | **10.800.000 ms (3 h)** |
| gcTime | **21.600.000 ms (6 h)** |
| Timeout | `AbortController` abortado após **55.000 ms**; também aborta se o `signal` do react-query disparar `abort` |
| retry | `(failureCount, err) => failureCount < 1 && err.__isApiError !== true` (1 nova tentativa só para erro que não é de API), `retryDelay: 2000` |
| placeholderData | `fallbackData` (= `aiWidget.cachedData` do store) quando truthy → pinta o cache na hora |
| Refresh | não há polling nem `refetchInterval` na Vision; atualiza ao vencer o stale (3 h) quando a query é remontada/focada, ou por invalidação. O `?refresh=true` existe na API, mas nenhum chamador da Vision o usa [O: único call site passa só o id] |

---

## 4. Ciclo de vida do widget de IA (criação, lista, remoção) [O]

### 4.1 API REST (`aiVisionWidgetApi`, M8988, L2839054–2839150)
| Função | Método e path | Body | Uso |
|---|---|---|---|
| `list()` | `GET /api/mobile/vision/widgets` | — | `{success, data: AIWidget[]}` |
| `create(body)` | `POST /api/mobile/vision/widgets` | `{prompt, widgetType, title, size, config}` | `{success, data: {id, widgetType, title, size, cachedData, …}}` |
| `loadData(id, refresh=false)` | `GET /api/mobile/vision/widgets/{id}/ai-data[?refresh=true]` | — | dados para desenhar (§3.5) |
| `remove(id)` | `DELETE /api/mobile/vision/widgets/{id}` | — | desativa (soft delete, "Remover da home") |
| `permanentDelete(id)` | `DELETE /api/mobile/vision/widgets/{id}?permanent=true` | — | "Apagar permanentemente" |
| `reactivate(id, size)` | `PATCH /api/mobile/vision/widgets/{id}` | `{isActive: true, size}` | volta à home |
| (chat) | `POST {baseUrl}/api/mobile/vision/widgets/chat` | UI messages do AI SDK | conversa de criação (§6) |

Modelo no cliente (`_temp` em L2897400 e store): `AIWidget = { id: string /*uuid*/, widgetType: string, title: string, size: 'half'|'wide'|'full', isActive: boolean, cachedData?: object }`. Na home o id vira `ai-{id}`.

Hooks (M9108, L2897450–2898150): `aiVisionWidgetKeys = { all: ['ai-vision-widgets'], list: () => ['ai-vision-widgets','list'] }`.
- `useAIVisionWidgetsList({enabled=true})`: `useQuery({queryKey: list(), queryFn, staleTime:120000, gcTime:600000})`. O `queryFn` chama `list()`; se `!success` → `Error(error?.message ?? 'Failed to list vision widgets')`; se ok, chama `useVisionWidgetOrderStore.getState().syncAIWidgetsFromServer(data.map(w => ({id, widgetType, title, size, isActive, cachedData})))` e devolve `data`.
- `useCreateAIVisionWidget()` (mutation `create` + `addAIWidget` + invalida a lista) e `useRemoveAIVisionWidget()` (mutation `remove` + `removeAIWidget`) existem, mas **não têm chamador** no bundle; a criação real está em `VisionWidgetChatScreen` (§4.2).

### 4.2 Fluxo de criação (`VisionWidgetChatScreen`, rota `/(protected)/vision/widget-chat`, M10456, L3334646)
1. Tela: `View className="flex-1 bg-background"` (#09090b) com `paddingTop = insets.top`, `paddingBottom = insets.bottom`, contendo `<WidgetChat initialPrompt={params.prompt} onBack={router.back} onWidgetCreated chatEndpoint={getBaseUrl() + '/api/mobile/vision/widgets/chat'} headerVariant="panel" subtitle="Os dados do preview são ilustrativos. Os valores reais serão usados ao criar." testID="vision-widget-chat"/>`. Analytics de tela `VISION_PAGE_NAMES.WIDGET_CHAT = 'Widget Chat'`.
2. O usuário conversa; o servidor (LLM) chama as tools `suggest_widgets` → `show_chart_options` → `generate_preview` (§6). A tool `generate_preview` devolve a **spec** `{widgetType, title, size, config, data}`.
3. Ao tocar "Adicionar" no preview, o `WidgetChat` monta `{widgetType, title, prompt: extractFirstUserText(messages), config, size}` e chama `onWidgetCreated(spec)` (háptico **Medium** antes).
4. `onWidgetCreated` (generator em L3334720–3335150):
   - `aiVisionWidgetApi.create({prompt, widgetType, title, size, config})`; se `!success || !data` → `Error(error?.message ?? 'Falha ao criar widget')`.
   - `trackWidgetInteraction('created_via_ai', {widget_id: 'ai-'+id, widget_type:'ai', ai_widget_type: widgetType, widget_size: size, source_screen:'widget_chat'})`.
   - `store.addAIWidget({id, widgetType, title, size, isActive:true, cachedData: data.cachedData})`.
   - Sincroniza o layout: `acquireVisionLayoutSyncLock()`; invalida `aiVisionWidgetKeys.all`; `visionHomeLayoutApi.patchLayout(buildVisionPatchPayload(store))` (`/api/mobile/vision/layout`); com sucesso `markSynced(version)` e invalida `visionLayoutKeys.layout`; erro `409` → `logger.warn` + invalida o layout; outros → `logger.error('[VisionWidgetChatScreen] Failed to sync AI widget layout')`; sempre `releaseVisionLayoutSyncLock()`.
   - Feedback: `successFeedback()` (háptico de sucesso), toast `showSuccess(\`${title} adicionado\`)`, fecha a tela (`router.dismiss`/`back`).
   - Erro: `ApiError` com `code === 'SESSION_EXPIRED'` é repassado (fluxo de sessão); demais → `logger.error('[VisionWidgetChatScreen] Failed to create AI widget', …)` e toast `showError('Falha ao criar widget. Tente novamente.')`.
5. **Ponto de entrada**: nenhum `router.push` para `widget-chat` foi achado no bundle (a rota existe em `app/(protected)/vision/widget-chat.tsx`, registrada como modal em L3057558). [I: entrada por flag remota, deep link ou removida nesta versão — dúvida aberta.]

### 4.3 Store da home (`useVisionWidgetOrderStore`, L768200–771500) [O]
Estado inclui `aiWidgets: AIWidget[]` (persistido junto com `order`, `widgetSizes` etc.; ver o design system para a chave `pierre-vision-widget-order:<userId>`). Ações:
- `addAIWidget(w)`: se já existe → `reactivateAIWidget(w.id, w.size)`. Senão insere `'ai-'+id` em `order` respeitando widgets fixos (`isVisionWidgetFixed`), grava `widgetSizes['ai-'+id] = w.size`, adiciona `{...w, isActive:true}` em `aiWidgets`, `version+1`, `isDirty:true`, `lastAddedWidgetId:'ai-'+id` (dispara o "pulse" de recém-adicionado, ver `00-base.md` §4.4). [I: posição exata de inserção — o laço percorre `order` contando tamanhos para achar o primeiro encaixe.]
- `removeAIWidget(id)`: marca `isActive:false`, tira `'ai-'+id` de `order` e de `widgetSizes`, `isDirty`.
- `hardDeleteAIWidget(id)`: remove de `aiWidgets`, `order` e `widgetSizes`.
- `reactivateAIWidget(id, size)`: `isActive:true`, recoloca no `order` com o tamanho.
- `syncAIWidgetsFromServer(list)`: substitui `aiWidgets` pela lista do servidor (`size ?? 'half'`); acrescenta ao `order` os ativos que faltam e remove os inativos/inexistentes (se `isDirty`, preserva a ordem local).
- `useVisionAIWidgets()` (L771445) = só ativos; `useAllVisionAIWidgets()` (L771456) = todos (tela "adicionar widget", seção "Widget IA", com opções "Remover da home", "Apagar permanentemente" e o diálogo "Apagar widget permanentemente?" / "Essa ação é irreversível. O widget será removido permanentemente e não poderá ser recuperado.").

---

## 5. `AIWidgetRenderer` (L2988473, M9541) e as primitivas da spec

### 5.1 Onde é usado [O]
`AIWidgetRenderer` **não** é usado pela home Vision (que usa `AIVisionWidget`, §3). Ele desenha:
- os **previews no chat de criação** (`WidgetCard` das sugestões, `ChartOptionCard`, `WidgetPreviewCard`, §6), inclusive no chat aberto pela Vision;
- os widgets de IA da **home do cartão** (`AIWidgetWrapper`, §7) e o `CardAIWidgetPreview` (L3080127).

Consequência para a réplica: o preview no chat pode mostrar gráfico de área, barras ou cripto, mas, depois de adicionado à Vision, o widget vira a "lista com barras" do §3 (só os formatos `items`/`accounts`).

### 5.2 Despacho por `widgetType` [O]
Props: `{widgetType, size, data, onPress, showChevron, bg, previewWidth}`. O `data` é **espalhado** como props do componente.

| `widgetType` | half / wide | full |
|---|---|---|
| `comparison`, `category-comparison`, `vision-account-comparison` | `CategoryComparison` | `CategoryComparisonFull` |
| `crypto-price` | `CryptoPrice` | `CryptoPriceFull` |
| outro | `null` | `null` |

Catálogo local de reserva (`SYSTEM_FALLBACK_CATALOG`, ~L2985270). O catálogo real vem de `GET /api/mobile/widgets/catalog` (`useWidgetCatalog`, com cache local `getCachedCatalog`/`saveCachedCatalog`):
```js
{ id:'category-comparison', name:'Comparativo de categorias', description:'Compare gastos entre 2 categorias em um período',
  icon:'BarChart3', readonly:true, tapAction:'expand', supportedSizes:['half','wide','full'], defaultSize:'half',
  supportedCharts:['area','horizontal-bar'], defaultChart:'area' }
{ id:'crypto-price', name:'Preço de criptomoeda', description:'Acompanhe o valor de uma criptomoeda em tempo real',
  icon:'TrendingUp', readonly:true, tapAction:'expand', autoRefresh:true, showChartStep:false,
  supportedSizes:['half','wide','full'], defaultSize:'half', supportedCharts:['area'], defaultChart:'area' }
```
`AI_WIDGET_ICON_MAP = {BarChart3:'productChart', TrendingUp:'arrowTrendUp'}` converte ícones Lucide em `PierreIcon` (`resolveWidgetIcon`).

### 5.3 Spec (payload) que o servidor devolve [O: campos lidos pelo cliente]
A tool `generate_preview` devolve `output = { success: true, preview: { widgetType, title, size, config, data } }`. O `create` reenvia `{prompt, widgetType, title, size, config}`, e o servidor gera o `cachedData` e o `ai-data`.

`data` por tipo:
```ts
// comparison / category-comparison / vision-account-comparison (chat e home do cartão)
{ title: string, items: { label: string, value: number, formattedValue: string, color?: string }[],
  total: string /* "R$ 1.234,56" */, period: string, chartType: 'area' | 'horizontal-bar' }
// formatos aceitos pela Vision (AIVisionWidget, §3.2)
{ items: {label, formattedValue, percentage, color}[], total: string, period?: string }
{ accounts: {label, formattedBalance, percentage, color}[], totalBalance: string, period?: string }
// crypto-price
{ symbol: string /* 'BTC' */, name: string, price: string|number, priceBRL?: string, changePercent: number,
  sparklineData: number[], lastUpdate?: string|number, period?: string }
```
O `config` é opaco no cliente: só é repassado ao `create` [O].
- A sugestão (`suggest_widgets`) traz `{widgetType, title, size, chartType, mockItems, total, periodLabel, mockData?}`.
- As opções de gráfico (`show_chart_options`) trazem `{title, widgetType, periodLabel, size, options: [{chartType, mockItems, total}]}`.

### 5.4 `CategoryComparison` (half/wide), M9542 [O]
`renderCategoryComparisonVariant(props, AreaChartHalf, HorizontalBarHalf)`: com `chartType === 'horizontal-bar'` desenha barras; senão, área. O card externo é o `CategoryComparisonCard`:
```
Widget2x2 (versão da home do cartão, M9543) testID='ai-widget-category-comparison'
  visual={gráfico} label={title} subtitle={period} onPress showChevron bg
  valueSlot = PrivacyCurrencyText valueCents={parseBRLToCents(total)} hidden={hideValues}
              fontSize=20 fontWeight='800' color={colors.foreground /* #fafafa */} disableMinWidth
              testID='ai-widget-category-comparison-value'
```
Aqui **há** "Ocultar dados" (`useHideValues` → `R$ ••••`, ver `00-base.md` §10).
- **Barras (`HorizontalBarHalf`)**: usa `items.slice(0, 2)` (2 linhas no half).
  - Cada linha é um `VStack style={gap:2}` com um `HStack {justifyContent:'space-between', alignItems:'center'}`: `label` em `text-xs text-muted-foreground` (1 linha) e `formattedValue` (ou `HIDDEN_VALUE`) em `text-xs font-semibold text-foreground`.
  - Trilho: `{height:6, borderRadius:3, backgroundColor:'rgba(255,255,255,0.08)', overflow:'hidden'}`.
  - Preenchimento: `{height:'100%', borderRadius:3, minWidth:4, width: max(value/maxValue×100, 4)+'%', backgroundColor: item.color}`.
  - O container das barras tem `{gap:6}`. A barra é relativa ao **maior** item, não à soma.
- **Área (`AreaChartHalf`)**: `CategoryAreaChart` = `LineChart` do react-native-gifted-charts (M6817).
  - Props: `curved`, `areaChart`, `thickness:2`, `endOpacity1/2: 0`, `startOpacity1 0.3`, `startOpacity2 0.2`, `isAnimated:false`.
  - Fica dentro de um `View {height:50, overflow:'hidden'}`, medido por `onLayout`.
  - Duas séries: `items[0]` (cor `color ?? '#8B5CF6'`) e `items[1]` (cor `color ?? '#3B82F6'`).
  - **Os pontos são sintéticos** (`generateCategoryPoints(value, [0.3, 0.25, 0.2])`, L2989890):
    ```
    resto = value
    para cada f: v = round(resto × f); push(v); resto −= v
    push(max(resto, 0))
    ```
    O resultado é uma curva decrescente de 4 pontos.
  - `maxValue = max(todos) × 1.2`; `spacing = (largura − endSpacing)/(n−1)`.
- Toque: o `Widget2x2` da home do cartão tem `Pressable` com escala **0,96** (`withSpring`, `damping 25`, `stiffness 800`) e háptico Light. É diferente do da Vision, que não escala.

### 5.5 `CategoryComparisonFull`, M9544 [O]
Usa o `WidgetFull` da versão do cartão (escala 0,97 no toque, mesma mola), com `testID='ai-widget-category-comparison-full'` e título = `title`.
- **Área (`AreaChartLayout`)**:
  - Legenda: `HStack className="gap-4"`; cada item é um `HStack "items-center gap-1.5"` com ponto `{width:8,height:8,borderRadius:4,backgroundColor:color}` e `label` (`text-sm text-muted-foreground`).
  - Gráfico em `View {overflow:'hidden'}`, com `startOpacity1 0.25` e `startOpacity2 0.15`.
  - Rótulos X `['01','08','15','22','31']`, com `xLabelStyle {color:'#A1A1AA' (dark) / '#71717a' (light), fontSize:10}`.
  - Pontos: `generateCategoryPoints(value, [0.25, 0.22, 0.2, 0.18], rótulos)`.
  - Rodapé: `HStack "justify-between pt-1"` com `period` (`text-sm text-muted-foreground`) e `PrivacyText` com `total` ou `R$ ••••` (`text-base font-bold text-foreground`).
- **Barras (`HorizontalBarLayout`)**: `VStack "gap-3"` com **todos** os itens.
  - Cada item: `VStack "gap-1"` com um `HStack "justify-between"` (`label` em `text-sm text-muted-foreground`; valor em `text-sm font-semibold text-foreground`).
  - Trilho: `View className="bg-muted"` (#27272a) `{height:8, borderRadius:4, overflow:'hidden'}`.
  - Preenchimento: `{height:'100%', borderRadius:4, minWidth:4, width: max(value/max×100,4)%, backgroundColor: color}`.
  - Rodapé: o mesmo de período/total.

### 5.6 `CryptoPrice` / `CryptoPriceFull` (L2991818 / L2992815) [O]
- **Cores** (`getTrendColors(changePercent)`):
  - Positivo: linha `COLOR_POSITIVE '#10B981'`; área de `rgba(16,185,129,0.3)` a `rgba(16,185,129,0.01)`.
  - Negativo: linha `'#EF4444'`; área de `rgba(239,68,68,0.3)` a `rgba(239,68,68,0.01)`.
  - A variação aparece como `▲`/`▼` + `changePercent.toFixed(2)%`, em `text-xs font-semibold` (half) ou `text-sm font-semibold` (full), na cor da tendência.
- **Logo** (`CoinLogo`): `Image {uri: 'https://cdn.jsdelivr.net/npm/cryptocurrency-icons@0.18.1/128/color/' + symbol.toLowerCase() + '.png'}`, `resizeMode:'contain'`. Tamanho 28 (half) ou 32 (full), raio = metade do tamanho.
- **`SparklineChart`**: `LineChart` do gifted-charts com `areaChart`, `thickness:2`, `startOpacity 0.9`, `endOpacity 0.1`, sem pontos, eixos ou linhas de grade. O `useSparklineChart` normaliza `sparklineData` (min/max com folga de 0,15) e calcula o `spacing` pela largura medida.
- **Half**:
  - Container: `VStack "justify-between rounded-3xl bg-surface p-4"` `{height:200}`, testID `ai-widget-crypto-price` (raio 21, padding 14).
  - Topo: `HStack "items-center justify-between"` com o símbolo (`text-base text-muted-foreground font-medium`) e `CoinLogo 28`.
  - Sparkline em `{height:60, overflow:'hidden'}`.
  - Preço: `PrivacyText text-2xl font-extrabold text-foreground` (autoshrink 0,75). `priceBRL`: `text-sm text-muted-foreground` (autoshrink 0,8).
  - Toque com escala via `withSpring`.
- **Full**:
  - `WidgetFull` `testID='ai-widget-crypto-price-full'`, título = `name`, `headerRight` = `CoinLogo 32`.
  - Preço em `text-2xl font-extrabold`; variação em `text-sm font-semibold`, entre `' ('` e `')'`.
  - Rodapé: `text-xs text-muted-foreground` com `formatTime(lastUpdate)` = `'Atualizado às ' + HH:MM` (com zero à esquerda).
- Só existe no chat e no cartão. A Vision não desenha cripto: o `AIVisionWidget` cairia no ícone de brilho com `—` [O].

---

## 6. Chat embutido de criação: `WidgetChat` (L3335653, M10459)

### 6.1 Props e transporte [O]
`WidgetChat({ initialPrompt, onBack, onWidgetCreated, chatEndpoint, alwaysSkipChartStep = false, headerTopClass = 'pt-3', subtitle, headerVariant = 'default', testID = 'widget-chat' })`.
- Usa o `useChat` do **Vercel AI SDK v5** com `new DefaultChatTransport({ api, fetch: createFetchWithAuthRetry(), headers })`:
  - `api` = `chatEndpoint` ou, por padrão, `getBaseUrl() + '/api/mobile/widgets/chat'` (cartão). A Vision passa `…/api/mobile/vision/widgets/chat`.
  - `headers` é assíncrono: `{ 'User-Agent': AI_SDK_USER_AGENT, Authorization: 'Bearer ' + await authTokenService.getToken() }`; o `Authorization` só entra se houver token.
  - Corpo e stream seguem o padrão do AI SDK (UI messages com SSE), como em `agentes-pierre.md` §1.4.
- `initialPrompt`: num `useEffect`, se houver e ainda não tiver sido enviado (ref), faz `sendMessage({text: initialPrompt})` uma vez.
- `isBusy = status === 'submitted' || status === 'streaming'`.
- Rolagem: a cada mudança de `messages.length`, `setTimeout(() => scrollRef.current?.scrollToEnd({animated:true}), …)`.

### 6.2 Protocolo de tools e comandos ("máquina de estados" no servidor) [O]
O LLM no servidor conduz três passos, e cada um é uma tool cujo `output` o cliente desenha:

| Ordem | Tool (`tool-<nome>` ou `dynamic-tool`) | `output` | UI |
|---|---|---|---|
| 1 | `suggest_widgets` | `{success, suggestions: Suggestion[]}` | `SuggestionsGrid` |
| 2 | `show_chart_options` | `{success, title, widgetType, periodLabel, size, options:[{chartType, mockItems, total}]}` | `ChartOptionsGrid` |
| 3 | `generate_preview` | `{success, preview:{widgetType, title, size, config, data}}` | `WidgetPreviewCard` + botão "Adicionar widget" |

As escolhas do usuário voltam ao servidor como **mensagens de texto de comando** (`sendMessage({text})`):
- Escolher a sugestão `i` (`onSelectSuggestion(i)`): háptico Light; `s = extractSuggestionAtIndex(messages, i)` (lê a última `suggest_widgets` com `output-available` e `success`); `size = s.size ?? 'half'`. Se `alwaysSkipChartStep` for verdadeiro ou o catálogo disser `showChartStep === false`, envia `[SKIP_CHART:{i}:{size}]`; senão, `[SELECT_WIDGET:{i}:{size}]`.
- Escolher o gráfico (`onSelectChart(chartType)`): háptico Light e `[SELECT_CHART:{chartType}]` (`'area'` = "Linhas", `'horizontal-bar'` = "Barras").
- Mensagens do usuário que começam com `[SELECT_` ou `[SKIP_CHART:` **não aparecem** na lista.
- Visibilidade das tools: um `suggest_widgets` some quando já existe `show_chart_options` ou `generate_preview` na conversa; um `show_chart_options` some quando já existe `generate_preview`. Assim, na tela fica só o passo atual.
- O `ToolCallRenderer` só desenha com `state === 'output-available'` e `output.success`. Ele também espera `InteractionManager.runAfterInteractions` antes de montar o conteúdo pesado (estado `ready`).

`extractPreviewData(messages)` (L3335289) percorre as mensagens **de trás para frente** e devolve o `preview` do último `generate_preview` com sucesso. `extractFirstUserText(messages)` (L3339376) devolve o primeiro texto digitado pelo usuário, que vira o `prompt` do `create`.

### 6.3 Layout [O]
- **Header "panel"** (o que a Vision usa):
  - Caixa `panelHeader {marginHorizontal:16, marginBottom:16, paddingTop:8, paddingHorizontal:10, paddingBottom:18, borderRadius:30, backgroundColor:'#05060A'}`.
  - Linha do botão de fechar `{width:'100%', alignItems:'flex-end'}`: `GlassContainer glassEffectStyle="regular"` de 44×44, raio 22 (no Android sem glass), com `Pressable` testID `widget-chat-back` e `Icon as={CloseIcon} size="md" className="text-foreground"`.
  - Conteúdo `{marginTop:32, gap:12}`. Título `"Criar widget"` com `{color:'#F5F7FA', fontSize:28, lineHeight:32, letterSpacing:-0.8, fontWeight:'400'}` (Geist-Regular pelo `Text` base). Subtítulo `{color:'rgba(245,247,250,0.55)', fontSize:14, lineHeight:20}` = "Os dados do preview são ilustrativos. Os valores reais serão usados ao criar."
- **Header "default"** (cartão):
  - `HStack className="items-center gap-3 px-4 {headerTopClass} pb-2"`.
  - Botão voltar: `Pressable "w-14 h-14 items-center justify-center rounded-full border border-border"` (56×56, borda #27272a) com `chevronLeft 24`.
  - Título: `Text "text-3xl font-bold text-foreground flex-1"` → `Criar widget`; subtítulo em `text-sm text-muted-foreground px-4 pb-2`.
- **Lista**: `ScrollView` com `{flex:1}`, conteúdo `{paddingHorizontal:12, paddingTop:8, paddingBottom:100}`, `keyboardShouldPersistTaps="handled"`, sem indicador.
  - **Bolha (`ChatMessage`, M10457)**: container `{paddingHorizontal:4, marginBottom:12}`, alinhado à direita (usuário) ou à esquerda (IA).
  - Bolha: `{maxWidth:'85%', borderRadius:20, paddingHorizontal:16, paddingVertical:10}`, com o canto de baixo do lado do autor em 6 (`borderBottomRightRadius` no usuário, `borderBottomLeftRadius` na IA).
  - Fundo (dark): `rgba(255,255,255,0.1)` e `rgba(255,255,255,0.06)` [O valores; I qual é do usuário: provavelmente 0,1].
  - Texto: `text-foreground`, `{fontSize:15, lineHeight:22}`. Conteúdo vazio → `null`.
  - **Pensando**: `ThinkingPlaceholder` = `Animated.View` (FadeIn/FadeOut) `{paddingHorizontal:4, paddingVertical:8, marginBottom:12}` com `PierreRiveLoading size=32 showLabel autoRotate` (animação Rive da marca), testID `thinking-placeholder`.
- **Bloco do preview** (quando `extractPreviewData` acha algo), em `Animated.View {marginBottom:20}`:
  - Bolha da IA: `Prontinho! Preparei o widget pra você. Confira o preview e toque em adicionar quando estiver satisfeito.`
  - `View {marginTop:12}` → `WidgetPreviewCard`.
- **Campo de prompt**: aparece só enquanto `messages.length === 0` [O; depois a conversa segue pelos cartões].
  - `KeyboardAvoidingView behavior="height"`, posição `{position:'absolute', bottom:-4, left:0, right:0, paddingHorizontal:12}`; linha `{flexDirection:'row', alignItems:'flex-end', gap:8}`.
  - Pílula: `GlassContainer "regular"` `{flex:1, minHeight:48, borderRadius:24, overflow:'hidden'}`, com borda absoluta `{borderRadius:24, borderWidth:1, borderColor:'rgba(255,255,255,0.1)'}`.
  - Linha interna: `{flexDirection:'row', alignItems:'center', minHeight:48, paddingHorizontal:7, paddingVertical:4, gap:2}`.
  - `TextInput`: placeholder `Envie uma mensagem` (cor `rgba(255,255,255,0.4)`), `multiline`, `maxLength=300`, `editable={!isBusy}`, estilo `{flex:1, fontSize:17, lineHeight:24, minHeight:42, maxHeight:120, paddingTop:9, paddingBottom:9, paddingLeft:12, paddingRight:4, letterSpacing:-0.15, color:'#fff'}`, testID `widget-chat-input`.
  - Botão enviar: `Pressable` `{width:34, height:34, borderRadius:17, overflow:'hidden', alignItems:'center', justifyContent:'center', flexShrink:0, backgroundColor:'rgba(255,255,255,0.88)'}`; `opacity:0.3` quando vazio ou ocupado; ícone `arrowUp` 18, cor `#0a0a0b`, traço 2,5; testID `widget-chat-send`.
  - Ao enviar: háptico Light, `sendMessage({text: input.trim()})`, limpa o campo.
  - (Valores light existem: `rgba(0,0,0,…)`/`#0a0a0b`; o app roda em dark.)

### 6.4 Cartões dos passos [O, M10457]
- **Textos de introdução** (`text-sm text-muted-foreground px-1`):
  - `suggest_widgets`: "Encontrei algumas opções que combinam com o que você quer. Escolha o design do widget:"
  - `show_chart_options`: "Boa escolha! Agora escolha o tipo de gráfico:"
- **`SuggestionsGrid`**:
  - Layout: `VStack "gap-3 mx-1"` (testID `tool-call-suggestions`) = texto + sugestões `wide`/`full` em largura total (key `wide-…`) + sugestões `half` em `HStack "gap-2"`, cada uma em `halfCard {flex:1, maxWidth:'50%'}`.
  - Filtra tipos por um `Set(['crypto-price'])` (regra de exibição de cripto) [I: o sentido exato do filtro].
  - Desabilitado (`isBusy`): `{opacity:0.5, pointerEvents:'none'}`.
- **`WidgetCard`** (cada sugestão):
  - Monta `data = s.mockData ?? {title, items: s.mockItems ?? [], total: s.total ?? '', period: s.periodLabel ?? '', chartType}`.
  - Largura de preview: `previewWidth = (half ? (W − 32 − 8)/2 : W − 32) − 32`.
  - Renderiza `AIWidgetRenderer {widgetType, size, data, showChevron: !mockData, bg:'solid', previewWidth}` dentro de um `PressableScale` (toque `scale → 0.96`, solta `→ 1`, `withSpring {damping:15, stiffness:300}`, via `GestureDetector`), testID `suggestion-card-{i}`.
- **`ChartOptionsGrid` / `ChartOptionCard`**:
  - Uma opção por `chartType`, com `data = {title, items: option.mockItems, total: option.total, period: periodLabel, chartType}`.
  - `size === 'half'`: as opções ficam lado a lado (`HStack gap-2`); senão, empilhadas (`VStack gap-3`). testID `tool-call-chart-options`.
  - Cada cartão é o `AIWidgetRenderer bg='solid'` com um selo flutuante `{position:'absolute', top:-6, left:8, zIndex:10}` → `{backgroundColor:'rgba(255,255,255,0.15)', paddingHorizontal:8, paddingVertical:2, borderRadius:6}` com o texto "Linhas"/"Barras" (`text-xs font-medium text-foreground`).
- **`WidgetPreviewCard`** (M10458):
  - `VStack "gap-3"` (testID `widget-preview-card`).
  - Preview centralizado `{alignSelf:'center', overflow:'hidden', width: half ? (W − 32 − 12)/2 : W − 32}` com `AIWidgetRenderer bg='solid'` e `pointerEvents="none"`.
  - Aviso: `text-xs text-muted-foreground text-center px-4` → "Estes dados são apenas uma prévia. Os dados reais serão carregados ao adicionar o widget."
  - Botão: container `{paddingHorizontal:16}` → gluestack `Button size="xl" rounded="full"` (`isDisabled={isAdding}`, testID `widget-preview-add-button`) → `ButtonText className="text-xl"` "Adicionar widget", ou `ButtonSpinner className="text-background"` durante `isAdding`.

### 6.5 Adicionar [O]
`onAdd`: se já está adicionando, sai. Senão: `isAdding = true`, háptico **Medium**, `await onWidgetCreated({widgetType, title, prompt: extractFirstUserText(messages), config, size})` e, no fim, `isAdding = false`. Erros → `logger.error('[WidgetChat] Failed to add widget', e)`. O que acontece depois (API, store, toast) está no §4.2.

---

## 7. `LiveWidgetRenderer` (L2982377) e `AIWidgetWrapper` (L2987909): home do cartão

Ficam aqui só para registrar o papel deles. A home do cartão é o grupo 6 (`WIDGET_REGISTRY`).

### 7.1 `WidgetSlotRenderer` (L2982177) [O]
Despacha por `slot.kind`:
- `'pending'` → `PendingSyncOverlay {widgetId, size}`;
- `'system'` → `LiveWidgetRenderer {widget: {...slot.data, size}}`;
- `'ai'` → `AIWidgetWrapper {widgetId, widgetType, size, onWidgetTap}`;
- `'empty'` → `null`.

### 7.2 `LiveWidgetRenderer` [O]
Não é IA: renderiza os widgets **de sistema** do cartão com dado "ao vivo".
- Só os ids do `Set(['available-limit','invoice','monthly-spending','recent-transactions','spending-by-category','installments','subscriptions','cdb-position','cashback'])` usam `useWidgetLiveEntry(id)` (store `setEntry`/`setEntries`/`clearAll`).
- Estado: `isFetched && !data && !isError` → `status:'empty'` (log `[LiveWidget] {id} → FETCHED but EMPTY`). `isError` → mantém `status` anterior se `'ok'`, senão `'error'` (log `… → ERROR, fallback=`). Com dado → `{status:'ok', data}`.
- Repassa para `WidgetRenderer {widget, isLoading}` (L2982600), que tem o `switch` por id do cartão e o tap `navigate` / `navigate-expand` / `expand` (com `measureInWindow` → `originX/Y/W/H`).

### 7.3 `AIWidgetWrapper` [O]
- **Dados**: `useAIWidgetData(widgetId, {autoRefresh: catalogEntry?.autoRefresh})`.
  - Endpoint: `GET /api/mobile/widgets/{id}/…` (API do cartão; `list/create/delete/permanentDelete/reactivate` em `/api/mobile/widgets`).
  - `staleTime` = **0** se `autoRefresh` (cripto), senão **10.800.000 ms (3 h)**; `gcTime` 21.600.000 (6 h).
- **Toque**:
  - `tapAction = entry?.tapAction ?? 'none'`; clicável se `!== 'none'`; `showChevron = !entry?.readonly && tapAction !== 'expand'`.
  - O toque mede o `View` (`measureInWindow`) e chama `onWidgetTap(widgetId, widgetType, size, rect…)` (expansão animada).
- **Estados**:
  - Carregando: `Animated.View entering={FadeIn.duration(300)}` (testID `ai-widget-skeleton-{id}`) com `SkeletonHalf` ou `SkeletonFull`.
    - Skeleton: `{borderRadius:24, backgroundColor:'rgba(255,255,255,0.06)', overflow:'hidden', padding:16}`, altura **200** (half) ou **180** (full).
    - Shimmer de opacidade 0,3 ↔ 0,7, `withTiming 1200 ms Easing.inOut(ease)`, em loop.
    - Barras: título `{width:'60%', height:14, borderRadius:7, rgba(255,255,255,0.08)}`, gráfico `{flex:1, borderRadius:12, rgba(255,255,255,0.05)}`, valor `{width:'40%', height:12, borderRadius:6}`.
    - O full tem também a legenda: pontos 8×8 e rótulos 48×10 `{marginRight:8}`, em linha com gap 6; e o rodapé `{width:'35%', height:12}`.
  - Erro: `Animated.View entering={FadeIn.duration(300)} exiting={FadeOut.duration(200)}` → `Pressable onPress={refetch}` (testID `ai-widget-error-{id}`) → `VStack "rounded-3xl bg-surface items-center justify-center p-4"` com altura 200 (half) ou 180 (full), contendo `PierreIcon arrowCurveRetry 24 #71717a` e "Toque para tentar novamente" (`text-sm text-muted-foreground mt-2`).
  - Ok: `View ref collapsable={false}` → `AIWidgetRenderer {widgetType, size, data, onPress, showChevron}`.
- **`PendingSyncOverlay`** (L2986694):
  - Caixa tracejada `{borderRadius:24, backgroundColor:'rgba(255, 255, 255, 0.05)', borderWidth:1, borderColor:'rgba(255, 255, 255, 0.08)', borderStyle:'dashed', alignItems:'center', justifyContent:'center', padding:16}`, com `minHeight` 140 (half) ou 120 (full).
  - Conteúdo: ícone `timeClock 20` e "Aguardando sincronização" (`text-sm font-medium text-muted-foreground text-center`).
  - O toque abre uma folha com o título `"{nome} está aguardando sincronização"` e o subtítulo "O widget será atualizado com seus dados assim que a conexão for restabelecida."

---

## 8. Snippet TSX de referência (caminho Android, dark)

Reaproveita `WidgetShell`/`useWidgetPress`, `Widget2x2`, `WidgetFull`, `PrivacyText` e os tokens `C`/`T` do snippet de `00-base.md` §15. As fontes Geist precisam estar carregadas. Troque assets, nomes e textos da CloudWalk pelos seus.

```tsx
// leo-and-ai-widgets.tsx
import React, { memo, useEffect, useRef } from 'react';
import { Animated, Pressable, StyleSheet, Text, View } from 'react-native';
import { Image } from 'expo-image';
import { VideoView, useVideoPlayer } from 'expo-video';
import { MessageCircle, Sparkles } from 'lucide-react-native';
import { C, Widget2x2, WidgetFull, useWidgetPress } from './widget-base';

// ---------- Avatar com vídeo em loop (Leo) ----------
function AgentAvatar({ size, radius, image, video, active }:
  { size: number; radius: number; image: any; video?: any; active: boolean }) {
  const visible = active && !!video;
  const opacity = useRef(new Animated.Value(visible ? 1 : 0)).current;
  const player = useVideoPlayer(video ?? null, (p) => { p.loop = true; p.muted = true; p.audioMixingMode = 'mixWithOthers'; });
  useEffect(() => {
    Animated.timing(opacity, { toValue: visible ? 1 : 0, duration: 180, useNativeDriver: true }).start();
    visible ? player.play() : player.pause();
  }, [visible]);
  return (
    <View style={{ width: size, height: size, borderRadius: radius, borderWidth: 1, borderColor: '#27272a' }}>
      <View style={[StyleSheet.absoluteFill, { borderRadius: radius, overflow: 'hidden' }]}>
        <Image source={image} style={StyleSheet.absoluteFill} contentFit="cover" />
        {video && (
          <Animated.View pointerEvents="none" style={[StyleSheet.absoluteFill, { opacity }]}>
            <VideoView player={player} style={StyleSheet.absoluteFill} contentFit="cover" nativeControls={false} />
          </Animated.View>
        )}
      </View>
    </View>
  );
}

// ---------- leo-agent (half, 200 px) ----------
export const AgentHalfWidget = memo(function AgentHalfWidget(
  { name, role, cta, image, video, active, onPress }:
  { name: string; role: string; cta: string; image: any; video?: any; active: boolean; onPress: () => void },
) {
  const press = useWidgetPress(onPress); // háptico Light → onPress (abre o chat com o agente)
  return (
    <Pressable onPress={press} testID="leo-agent-widget-pressable">
      <View style={{ height: 200, borderRadius: 24, backgroundColor: C.surface, padding: 14, rowGap: 10.5,
                     alignItems: 'center', justifyContent: 'center' }}>
        <AgentAvatar size={50} radius={14} image={image} video={video} active={active} />
        <View style={{ alignItems: 'center', rowGap: 1.75 }}>
          <Text allowFontScaling={false} numberOfLines={1} style={{ fontFamily: 'Geist-Medium', fontSize: 16, color: C.foreground }}>{name}</Text>
          <Text allowFontScaling={false} numberOfLines={1} style={{ fontFamily: 'Geist-Regular', fontSize: 10.5, lineHeight: 14, color: C.mutedForeground }}>{role}</Text>
        </View>
        <View style={{ alignSelf: 'stretch', height: 40, borderRadius: 20, overflow: 'hidden', backgroundColor: C.surface,
                       flexDirection: 'row', alignItems: 'center', justifyContent: 'center', columnGap: 7 }}>
          <MessageCircle size={18} color="#c7ff00" strokeWidth={2.2} />
          <Text allowFontScaling={false} numberOfLines={1} style={{ fontFamily: 'Geist-Medium', fontSize: 16, color: C.foreground }}>{cta}</Text>
        </View>
      </View>
    </Pressable>
  );
});

// ---------- leo-insight (wide, altura pelo conteúdo, mínimo 160) ----------
export const AgentInsightWidget = memo(function AgentInsightWidget(
  { name, description, ctaLabel, image, video, active, onCta }:
  { name: string; description: string; ctaLabel: string; image: any; video?: any; active: boolean; onCta: () => void },
) {
  return (
    <View style={{ width: '100%', minHeight: 160, borderRadius: 24, backgroundColor: C.surface }}>
      <View style={{ flex: 1, padding: 14, rowGap: 7 }}>
        <View style={{ flexDirection: 'row', alignItems: 'center', columnGap: 7 }}>
          <AgentAvatar size={40} radius={10.5} image={image} video={video} active={active} />
          <Text allowFontScaling={false} numberOfLines={1} style={{ fontFamily: 'Geist-Regular', fontSize: 16, color: C.mutedForeground }}>{name}</Text>
        </View>
        <Text allowFontScaling={false} numberOfLines={3} style={{ fontFamily: 'Geist-Regular', fontSize: 14, lineHeight: 21, color: C.foreground }}>{description}</Text>
        <View style={{ alignSelf: 'flex-start', paddingTop: 3.5 }}>
          <Pressable onPress={onCta} style={{ backgroundColor: '#fafafa', borderRadius: 9999, paddingHorizontal: 10.5, paddingVertical: 7 }}>
            <Text style={{ fontFamily: 'Geist-Medium', fontSize: 14, lineHeight: 21, color: '#18181b' }}>{ctaLabel}</Text>
          </Pressable>
        </View>
      </View>
    </View>
  );
});

// ---------- ai-<uuid> na Vision ----------
type Row = { label: string; formatted: string; percentage: number; color: string };
function toRows(d: any): { rows: Row[]; total?: string } | null {
  if (d && Array.isArray(d.accounts)) return { rows: d.accounts.map((a: any) => ({ label: a.label, formatted: a.formattedBalance, percentage: a.percentage, color: a.color })), total: d.totalBalance };
  if (d && Array.isArray(d.items)) return { rows: d.items.map((i: any) => ({ label: i.label, formatted: i.formattedValue, percentage: i.percentage, color: i.color })), total: d.total };
  return null;
}

const ComparisonVisual = ({ rows }: { rows: Row[] }) => rows.length < 2 ? null : (
  <View pointerEvents="none" style={{ rowGap: 7, width: '100%' }}>
    {rows.slice(0, 3).map((r, i) => (
      <View key={`${r.label}-${i}`} style={{ rowGap: 1.75 }}>
        <View style={{ flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center' }}>
          <Text allowFontScaling={false} numberOfLines={1} style={{ fontFamily: 'Geist-Regular', fontSize: 10.5, lineHeight: 14, color: C.mutedForeground }}>{r.label}</Text>
          <Text allowFontScaling={false} style={{ fontFamily: 'Geist-SemiBold', fontSize: 10.5, lineHeight: 14, color: C.foreground }}>{r.formatted}</Text>
        </View>
        <View style={{ height: 6, borderRadius: 3, backgroundColor: 'rgba(128,128,128,0.15)', overflow: 'hidden' }}>
          <View style={{ height: 6, borderRadius: 3, width: `${r.percentage}%`, backgroundColor: r.color }} />
        </View>
      </View>
    ))}
  </View>
);

const SparklesIcon = () => (
  <View style={{ width: 44, height: 44, borderRadius: 22, backgroundColor: '#27272a', alignItems: 'center', justifyContent: 'center' }}>
    <Sparkles size={20} color="#71717a" />
  </View>
);

export function AIVisionWidget({ id, title, size, data, isLoading, onPress }:
  { id: string; title: string; size: 'half' | 'wide' | 'full'; data?: any; isLoading: boolean; onPress?: () => void }) {
  const parsed = toRows(data);
  const testID = `ai-vision-widget-${id}`;
  if (size !== 'full') {               // half E wide usam o Widget2x2 (200 px)
    if (isLoading) return <Widget2x2 testID={testID} label={title} value="" subtitle="" visual={<View /* skeleton 64×64 */ />} />;
    return <Widget2x2 testID={testID} label={title} onPress={onPress}
      visual={parsed ? <ComparisonVisual rows={parsed.rows} /> : <SparklesIcon />}
      value={parsed?.total ?? '—'} subtitle={data?.period ?? 'Insight IA'} />;
  }
  return (
    <WidgetFull testID={testID} title={title} onPress={isLoading ? undefined : onPress} showChevron={!isLoading}>
      {parsed ? (
        <View style={{ paddingHorizontal: 14, paddingBottom: 14, width: '100%' }}>
          <ComparisonVisual rows={parsed.rows} />
          <View style={{ flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', marginTop: 10.5 }}>
            <Text allowFontScaling={false} style={{ fontFamily: 'Geist-Regular', fontSize: 16, color: C.mutedForeground }}>Total</Text>
            <Text allowFontScaling={false} style={{ fontFamily: 'Geist-Bold', fontSize: 16, color: C.foreground }}>{parsed.total}</Text>
          </View>
        </View>
      ) : (
        <View style={{ paddingHorizontal: 14, paddingBottom: 14 }}>
          <Text allowFontScaling={false} style={{ fontFamily: 'Geist-Regular', fontSize: 16, color: C.mutedForeground }}>Carregando dados do widget...</Text>
        </View>
      )}
    </WidgetFull>
  );
}

// Dados (react-query): staleTime 3 h, gcTime 6 h, placeholder = cache local, timeout de 55 s
// useQuery({ queryKey: ['ai-vision-widget-data', id], queryFn: () => api.get(`/api/mobile/vision/widgets/${id}/ai-data`),
//            staleTime: 10_800_000, gcTime: 21_600_000, retry: (n, e) => n < 1 && !e.__isApiError, retryDelay: 2000,
//            placeholderData: cachedData });
```

---

## 9. Dúvidas abertas / limites

1. **Entrada do chat de criação na Vision**: a rota `/(protected)/vision/widget-chat` existe e funciona, mas nenhum `router.push` para ela foi achado. Provavelmente é aberta por flag remota, deep link ou código fora do bundle JS.
2. **Prompts e tools no servidor**: `suggest_widgets`, `show_chart_options` e `generate_preview`, os comandos `[SELECT_WIDGET:i:size]`, `[SKIP_CHART:i:size]` e `[SELECT_CHART:type]`, e a geração do `cachedData`/`ai-data` rodam no servidor. O formato do `config` e o cálculo de `percentage`/`color` não estão no APK.
3. **Diferença preview × home**: na Vision, um widget `category-comparison` vira a lista de barras do `AIVisionWidget` (o `widgetType` é ignorado). Pelo código é o comportamento real, mas não foi validado em aparelho. `crypto-price` adicionado à Vision mostraria o ícone de brilho com `—`.
4. **Sem estado de erro nos widgets de IA da Vision**: no full, "Carregando dados do widget..." fica indefinidamente se a API falhar sem cache.
5. **Toque no widget de IA da Vision**: não navega (só háptico e analytics). A tela de detalhe `VisionWidgetDetailScreen` (L3339772) só abre pela tela "adicionar widget" e não foi detalhada aqui.
6. **Cor da bolha do chat** (0,1 × 0,06): qual valor é do usuário e qual é da IA depende do desvio de fluxo do decompilador [I].
7. **`SuggestionsGrid` e `Set(['crypto-price'])`**: o sentido exato do filtro (esconder ou só tratar cripto à parte) não foi confirmado.
8. **Constantes de módulo não materializadas** no `CategoryAreaChart` do cartão (altura/`endSpacing` exatos do gráfico de área half e full): o decompilador não mostrou os valores de closure. Valores observados: container half de 50 px; `maxValue ×1.2`.
9. **"Botão" Conversar do Leo**: no Android o fundo (`bg-surface`) é igual ao do card, então a pílula fica invisível. Só no iOS (glass) ela se destaca. Vale conferir em aparelho.
10. **lineHeight de `text-sm`** (16 px sem lineHeight no NativeWind): as alturas do Leo foram estimadas com ≈ 1,2×.
