# Catálogo dos widgets do Pierre (consolidado)

Visão única dos documentos `00` a `06b`. Aqui ficam o mapa, as regras que valem para todos os widgets, uma tabela por superfície (home Vision e home do cartão), os endpoints, as chaves de storage, as inconsistências já resolvidas e o índice das dúvidas abertas. Para réplica de um widget, leia a seção indicada na coluna **Doc**: este arquivo não repete cores, layout em px nem snippets.

Análise estática do bundle Hermes v98 (`out.js`). **[O]** = observado no código; **[I]** = inferido. Linhas `Lnnnn` e módulos `Mnnnn` referem-se ao `out.js`.

## 1. Mapa dos documentos

| Doc | Conteúdo |
|---|---|
| `00-base.md` | Cadeia de renderização, grid da Vision, `Widget2x2`/`WidgetWide`/`WidgetFull`, estados, "Ocultar dados", moeda, tokens dark, config NativeWind, snippets |
| `01-carteira-saldo-fluxo.md` | `wallet`, `account-balance`, `cash-flow`; §1 = busca de dados comum (`useVisionSystemWidgetData`); `BankLogo` |
| `02-gastos.md` | `monthly-spending` (gráficos nativos Kotlin `PierreCharts`), `spending-by-category` |
| `03-limite-parcelas-cartoes-transacoes.md` | `available-limit`, `installments`, `patrimony`, `cards`, `recent-transactions`, `MerchantAvatar` |
| `04-assinaturas.md` | `subscriptions`, tela de detalhe, calendário, tela do dia, adicionar assinatura |
| `05a-cdb.md` | `cdb-investment`, `cdb-performance`, `cdb-position`, flag `cdbInvestmentsEnabled` |
| `05b-leo-ia.md` | `leo-agent`, `leo-insight`, widgets de IA `ai-<uuid>`, `WidgetChat`, `AIWidgetRenderer`, `LiveWidgetRenderer` |
| `06a-home-cartao.md` | Registro, tela, grade, edição e persistência da home do cartão; `WidgetCarousel` (do chat) |
| `06b-widgets-cartao.md` | `cashback`, `invoice`, `CardWidget`, e as diferenças dos widgets reaproveitados pelo cartão |
| `BRIEF.md` | Briefing usado para gerar os documentos (cita a raiz antiga `C:/REA/Pierre/`) |

## 2. Duas superfícies, dois registros

| | Home Vision | Home do cartão |
|---|---|---|
| Onde | Hub `/vision` | Cena `CardsScene` do mesmo hub (`/vision?section=cards`), sem rota própria |
| Registro | `VISION_WIDGET_REGISTRY` (M2195, L771869) | `WIDGET_REGISTRY` (M2199, L775589) |
| Gap da grade | **18** | **12** |
| Grade | `VisionSortableWidgetGrid` | `Sortable.Flex`; o cartão fica **fora** da grade |
| Dados | `GET /api/mobile/vision/widgets/{id}/data` | API de crédito da CloudWalk (`/credit/...`), transformada no app |
| Layout no servidor | `/api/mobile/vision/layout` | `GET/PATCH /api/mobile/home-layout` |
| Ordem local | `pierre-vision-widget-order:<userId>` | `pierre-widget-order:<userId>` (+ cache `pierre:cardHomeLayout:<userId>`) |

O `WIDGET_REGISTRY` de L2202932 (M6739) é outra coisa: widgets de UI generativa do **chat**. E `WIDGET_SURFACE_CATALOG` (M9756) alimenta só a tela de adicionar widget (06a §1.4). Na dúvida, a fonte de verdade da grade do cartão é o M2199.

## 3. Regras comuns

- **Tamanhos [O]** (00 §3): half = `(W−32−18)/2` × 200; wide = `W−32` × 170; full = `W−32` com altura do conteúdo. Raio 24 vem de `style` (`borderRadius: 24`), não de classe.
- **Visual no Android [O]:** fundo `bg-surface` (`#18181b`), sem borda e sem sombra. Glass só no iOS. O toque não muda escala; dispara só háptico Light.
- **Alturas que fogem da regra:**

| Widget | Altura |
|---|---|
| `leo-insight` | conteúdo, mín. 160 |
| `cdb-investment` (wide) | mín. 250 |
| `cdb-performance` | half 200; wide mín. 210 |
| `cdb-position` | half 200; full mín. 384 |
| `installments` full | mín. 320 |
| `ai-<uuid>` half e wide | 200 |
| estados de erro/vazio do cartão | 200, inclusive no wide |

- **Privacidade [O]:** "Ocultar dados" usa o store `pierre-privacy` e a máscara `R$ ••••`, com fade de 150 ms (00 §10).
- **Números:** sem contagem animada nos widgets (`durationMs: 0`); a forma compacta só usa o sufixo "mi" (00 §11–12).
- **Tipografia:** só Geist. `font-*` troca a família e mantém `fontWeight` 400. `text-sm` = 16 e `text-base` = 14/21 (00 §14).
- **Escala NativeWind:** rem = 14, então `p-4` = 14. `rounded-3xl` = 21 (ver §8).
- **Estados [O]:** erro "Falha ao sincronizar dados" (ícone `feedbackWarning` `#EF4444`) e vazio "Sem dados disponíveis" (ícone `wifiOff` `#71717A`), em 00 §9.

## 4. Home Vision

Ordem padrão `DEFAULT_VISION_SORTABLE_ORDER` [O]: `wallet`, `account-balance`, `cash-flow`, `cdb-investment`, `available-limit`, `installments`, `cards`, `monthly-spending`, `recent-transactions`, `spending-by-category`, `subscriptions`.

| id | Label | Padrão | Tamanhos | Toque | No picker | Doc |
|---|---|---|---|---|---|---|
| `wallet` | Carteira | ver 01 | ver 01 | ver 01 §2 | sim | 01 §2 |
| `account-balance` | Saldo em contas | ver 01 | ver 01 | ver 01 §3 | sim | 01 §3 |
| `cash-flow` | Fluxo de Caixa nas Contas | ver 01 | ver 01 | ver 01 §4 | sim | 01 §4 |
| `monthly-spending` | Gastos do mês | wide | half, wide, full | `/vision/monthly-spending` | sim | 02 §1–3 |
| `spending-by-category` | Gastos por categoria | half | half, wide, full | `/vision/spending-by-category` (full: toque interno) | sim | 02 §4 |
| `available-limit` | Limite disponível | half | half, wide | `card-invoices?filterKey=limits` | sim | 03 §1 |
| `installments` | Parcelamentos | half | half, wide, full | `card-invoices?filterKey=installments` | sim | 03 §2 |
| `patrimony` | Patrimônio | half | half, full | nenhum | **oculto sempre** | 03 §3 |
| `cards` | Cartões | half | half, full | `card-invoices?filterKey=invoices` | aposentado | 03 §4 |
| `recent-transactions` | Transações recentes | full | half, wide, full | aba `transactions` | sim | 03 §5 |
| `subscriptions` | Assinaturas | half | half, wide, full | `/vision/subscriptions?filterKey=subscriptions` | sim | 04 |
| `cdb-investment` | Investimentos CDB | wide | wide | `/vision/cdb-investment-entry` (o componente usa `Link` próprio) | gerenciado pelo sistema | 05a §1 |
| `cdb-performance` | Vencimento CDB | half | half, wide | `/investments/cdb-investment-details` | aposentado | 05a §2 |
| `cdb-position` | Posição CDB | full | half, full | `/vision?section=investments` | sob demanda; entra sozinho após comprar | 05a §3 |
| `leo-agent` | Leo | half | half | registro aponta para `leo-cdb`, mas a home abre o chat | sob demanda | 05b §1 |
| `leo-insight` | Insights do Leo | wide | wide | nenhum (só o botão "Saiba mais") | sob demanda | 05b §2 |
| `ai-<uuid>` | (título gerado) | escolhido na criação | half, wide, full | nenhum | fora do picker; criado pelo chat | 05b §3–6 |

Regras do registro [O, 05a §0 e 05b §0]:
- `ON_DEMAND_PICKER_WIDGET_IDS` = `cdb-position`, `cdb-performance`, `leo-agent`, `leo-insight`. Todo `ai-*` também é sob demanda.
- `VISION_PICKER_RETIRED_WIDGET_IDS` = `cards`, `cdb-performance`: ainda renderizam se estiverem no layout salvo.
- Os três widgets CDB ficam ocultos quando `isCdbInvestmentEntryEnabled()` é falso. Isso exige `cdbInvestmentsEnabled` em `/api/mobile/config` e, com `internalOnly` (padrão), e-mail `@cloudwalk.io`.
- `insertPosition: 'top'` em CDB e Leo; os demais entram no fim.

## 5. Home do cartão

Registro M2199 [O, 06a §1.2]. Ordem padrão `DEFAULT_SORTABLE_ORDER`: `available-limit`, `invoice`, `cashback`, `cdb-position`, `monthly-spending`, `subscriptions`, `installments`, `recent-transactions`, `spending-by-category`.

| id | Label | Padrão | Tamanhos | Componente | Doc |
|---|---|---|---|---|---|
| `card` | Cartão | full (fixo) | full | `CardWidget` / `CardWithCollapse`, fora da grade | 06b §4 |
| `last-purchase` | Última compra | full (fixo) | full | não renderiza na grade; virou zona contextual "Compra aprovada" (`HomeContextualZone`, M9171) | 06a §1.2 |
| `available-limit` | Limite usado | half | half, wide | da Vision | 03 §1, 06b §6.1 |
| `invoice` | Fatura | half | half (não redimensiona) | **próprio** (`InvoiceWidget`, M9534) | 06b §3 |
| `cashback` | Cashback | wide | half, wide | **próprio** (`CashbackWidget`, M9532) | 06b §2 |
| `monthly-spending` | Gasto mensal | full | wide, full | da Vision | 02, 06b §6.2 |
| `subscriptions` | Assinaturas | half | half, wide, full | da Vision | 04, 06b §6.5 |
| `installments` | Parcelas | half | half, wide, full | da Vision | 03 §2, 06b §6.4 |
| `recent-transactions` | Transações recentes | full | half, wide, full | da Vision | 03 §5, 06b §6.3 |
| `spending-by-category` | Gastos por categoria | half | half, wide, full | da Vision, sem a camada `Connected` | 02 §4, 06b §5 |
| `cdb-position` | CDB | full | half, wide, full (o componente só trata half e full) | da Vision | 05a §3, 06b §6.6 |

Toque no cartão [O, 06b §1]: `cdb-position` → `/investments/cdb-investment-details` (`from=card_home`, `productId=pierre-one`); `cashback` → `/card/cashback-history`; `invoice` → `/card/invoice-stack?focus=first-open`.

## 6. Endpoints

| Uso | Endpoint |
|---|---|
| Dados dos widgets Vision | `GET /api/mobile/vision/widgets/{id}/data?size=...` (queryKey `['vision-widget-item-data', id, size, chartVariant]`, staleTime 2 min) |
| Widgets de IA (dados) | `GET /api/mobile/vision/widgets/{id}/ai-data` (refresh a cada 3 h, cache local antes da rede) |
| Criar widget de IA | chat em `/api/mobile/vision/widgets/chat` (tools `suggest_widgets`, `show_chart_options`, `generate_preview`) e `POST /api/mobile/vision/widgets` com `{prompt, widgetType, title, size, config}` |
| Subcategorias (full) | `GET /api/mobile/vision/spending-by-category/subcategories?period=YYYY-MM` |
| Layout Vision | `/api/mobile/vision/layout` |
| Layout do cartão | `GET/PATCH /api/mobile/home-layout` (stale 12 h, gc 24 h; PATCH ao sair da edição ou após 500 ms; `POST /reorganize` existe mas nada o aciona) |
| Flags | `/api/mobile/config` (`cdbInvestmentsEnabled` etc.) |
| CDB | `/api/cdb/summary`, `/api/cdb/list`, `/pierre/monthly-closing-balances`, `/pierre/history/list` (só com KYC aprovado) |
| Leo | `GET /api/agents` e `/api/agents/{id}/logs` |
| Crédito (cartão) | `/credit/limit`, `/credit/cashback/summary`, `/credit/invoices`, `/credit/invoices/current`, `/credit/invoices/items`, `/credit/statements/v2`, todos `GET` com `staleTime 0` |
| Logo de empresa | `/api/logo?domain=` (03 §6) |

## 7. Chaves de storage

| Chave | Uso |
|---|---|
| `pierre-vision-widget-order:<userId>` | ordem local da Vision |
| `pierre-widget-order:<userId>` | ordem local do cartão (migra da chave legada) |
| `pierre:cardHomeLayout:<userId>` | cache do layout do cartão |
| `pierre-privacy` | "Ocultar dados" |

## 8. Inconsistências resolvidas entre os documentos

- **Raio 21 × 24.** Os shells (`Widget2x2`, `WidgetWide`, `WidgetFull`) usam `style={{borderRadius: 24}}`. A classe `rounded-3xl` compila para **21** (00 §14). Portanto as caixas de erro/vazio feitas com `className="rounded-3xl bg-surface"` têm raio 21 (01 §3.2, 03 §1 e §2 já estavam certos). A frase de `02-gastos.md` §4 que dizia `rounded-3xl` = 24 foi corrigida.
- **Altura do wide.** 170 é do `WidgetWide`. `leo-insight` (mín. 160, conteúdo), `cdb-*` e o estado de erro do cartão (200) fogem dessa altura.
- **Labels diferem entre catálogos.** "Gastos do mês" (Vision) = "Gasto mensal" (cartão); "Parcelamentos" = "Parcelas"; "Limite disponível" = "Limite usado". Cada superfície usa o seu.
- **`monthly-spending` no cartão.** O M2199 manda (full, `[wide, full]`); o catálogo de superfície diz wide e `[half, wide, full]`.

## 9. Dúvidas abertas (índice)

Cada lista completa está no fim do documento indicado e em `widgets/README.md`.

- **00 e 01:** raio do `WidgetWide` (constante não materializada); `WALLET_STACK_TRAVEL`; `mergePierreCardIntoWalletWidget`; animação nativa do `PierreDonutChart`; "Conectar conta" sem `onPress`.
- **05a:** como `onWidgetTap` combina `{id, from}` com `navigateTo`; destino dinâmico do `cdb-investment` no Android; textos e `cohort` do servidor.
- **05b:** nada navega para `vision/widget-chat`; prompts das tools e formato de `config` ficam no servidor; widget de IA de cripto ou área vira lista de barras na Vision (não validado); botão "Conversar" do Leo provavelmente invisível no Android.
- **06a:** `last-purchase` não renderiza na grade; qual catálogo vale na tela de adicionar; algoritmo de `hasGaps`/`fillGaps`; gatilho de "reorganizar com IA".
- **06b:** formato real das strings da API de crédito; slot sem `data` pode quebrar gastos por categoria e mostrar "undefined pendentes" no cashback; alvo de `onExpandRequest` e momento da troca Congelar/Descongelar (supostos).

## 10. Defeitos do app que a réplica pode corrigir

- `installments` (cartão): o percentual pago é fixo em 25% no código.
- `recent-transactions` (cartão): os itens não trazem `type`, então o valor aparece sem sinal e sem cor [I].
- `spending-by-category` (cartão): o toggle Categorias/Grupos aparece mas não faz nada (falta a camada `Connected`).
- Widget de IA (Vision): sem estado de erro e sem navegação ao tocar.

## 11. Ordem sugerida para reimplementar

1. Base: tokens, `Widget2x2`/`WidgetWide`/`WidgetFull`, estados e privacidade (`00`).
2. Grade ordenável e persistência (`00` §3, `06a` §2).
3. Widgets só de dados e texto (`03`, `04`, `01` §3–4).
4. Gráficos em `react-native-svg` (`02`, `05a` §3).
5. Cartão e crédito (`06b`), com a máscara e o descarte de número/CVV em 60 s.
6. Leo e IA por último (`05b`), pois dependem de agentes e do chat no servidor (`agentes-pierre.md`).

Nomes, textos, imagens e vídeos são da CloudWalk: use este material como referência de arquitetura e crie identidade própria.
