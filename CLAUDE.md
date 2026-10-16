# Idioma
- Sempre responda em português do Brasil, independentemente do idioma do código ou dos arquivos.
- Mensagens de commit, comentários e explicações também devem ser em português do Brasil.

# Projeto
Engenharia reversa do app Android **Pierre** (`io.cloudwalk.pierre` v4.0.6, CloudWalk), um app de finanças pessoais com IA. Há dois objetivos:
1. Documentar o design system (cores, tipografia, componentes, widgets da home e a edição deles).
2. Entender os **agentes de IA** do app para reimplementá-los em outro app.

Toda a análise é **estática**: nada foi executado em aparelho. Nos documentos, **[O]** marca o que foi observado no código e **[I]** o que foi inferido.

## Arquivos
| Arquivo | Conteúdo |
|---|---|
| `pierre-design-system.html` | Design system completo: tokens dos temas dark/light, paletas de dados (categorias, 57 bancos, gráficos, gradientes), tipografia, espaçamento, raios, sombras, glass/blur, componentes, protótipo interativo da home com edição de widgets, catálogo de widgets, molas e hápticos |
| `agentes-pierre.md` | Agentes de IA: modelo de dados, API REST, logs, integração com o chat e as tools, catálogo dos 17 agentes, prompt de handoff, modo Computer e cartão agêntico, telas, além de uma proposta de implementação (SQL, worker, tool `createAgent`) |
| `Pierre.apk` | APK original (214 MB). **Fica fora do git**: passa do limite de 100 MB do GitHub e é binário de terceiros |

## Como o APK foi analisado
1. O APK é React Native/Expo. A UI fica em `assets/index.android.bundle`, compilada em **Hermes bytecode v98**.
2. Extração: `python -I -c "import zipfile; zipfile.ZipFile('Pierre.apk').extract('assets/index.android.bundle','apk')"`.
3. Decompilação com hermes-dec, instalado num venv isolado:
   `pip install git+https://github.com/P1sec/hermes-dec` e depois `hbc-decompiler apk/assets/index.android.bundle out.js`.
   O resultado tem cerca de 145 MB e 3,6 milhões de linhas. A v98 não é suportada oficialmente pela ferramenta, mas a saída é coerente. As referências `Lnnnn` dos documentos são linhas desse `out.js`.
4. Para a tabela de strings, use `HBCReader().read_whole_file()` e depois `reader.strings` (cerca de 133 mil strings).
5. Nunca leia `out.js` inteiro. Use `grep -n` para achar o trecho e `sed -n 'A,Bp' | cut -c1-2000` para ler. As funções têm o comentário `// Original name: X`, os exports usam `defineProperty(r5,'Nome',...)` e os estilos aparecem como literais de objeto.

Os artefatos gerados (`out.js`, `strings.json`, venv) ficam fora do repositório.

## Principais achados

### Stack
- Expo/React Native, com NativeWind v4 (**rem = 14**) sobre gluestack-ui v2.
- Reanimated, react-native-sortables, Zustand e react-query.
- Vercel AI SDK v5.

### Tema
- `THEME_TOKENS` com o hook `useThemeColors`. A base é zinc, a marca é `#cfff04` (dark) / `#8bb800` (light) e `pierre` = `#c7ff00`.
- `LIGHT_MODE_DISABLED = true`: o app fica sempre em dark.

### Tipografia
- Só **Geist**. As classes `font-*` trocam a família da fonte e mantêm `fontWeight` em 400.
- `text-sm` vale 16, mais que `text-base` (14/21).

### Home ("Vision")
- Grid de 2 colunas com gap 18 e padding 16. Tamanhos: half (200 px), wide (170 px) e full, todos com raio 24.
- Edição:
  - Segurar 500 ms abre o menu; mais 1500 ms entra no modo de edição.
  - No modo de edição os widgets tremem (±0.8°, 180 ms), e dá para remover, arrastar, redimensionar e usar "Preencher".
- Persistência local na chave `pierre-vision-widget-order:<userId>`; no servidor, `/api/mobile/vision/layout`.

### Agentes
- Cada agente é um registro no servidor, de tipo `scheduled` (cron) ou `condition` (com `checkInterval`).
- A criação é feita pelo chat: o LLM chama a tool `createAgent`. A API fica em `/api/agents`.
- O modelo padrão é `anthropic/claude-sonnet-4.6` (Vercel AI Gateway).
- Os prompts de sistema e as tools rodam no servidor e **não estão no APK**.

## Cuidados ao reutilizar
- Nomes, personas, textos, imagens e vídeos são da CloudWalk. Use-os como referência de arquitetura e crie identidade própria.
- No código, nunca aceite `userId` vindo do LLM. Ações de escrita e pagamentos exigem aprovação do usuário. Cartões envolvem PCI-DSS.
