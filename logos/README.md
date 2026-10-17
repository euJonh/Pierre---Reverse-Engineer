# Logos extraídas do Pierre.apk

Logos de bancos, lojas/apps (merchants da fatura) e bandeiras de cartão, extraídas do `Pierre.apk` (`io.cloudwalk.pierre` v4.0.6) por análise estática.

> **Atenção:** as logos são marcas registradas dos respectivos donos, e a curadoria pertence à CloudWalk. Verifique as regras de uso de marca antes de publicar.

| Pasta | Qtd | Formato | Origem no APK |
|---|---|---|---|
| `bancos/svg/` | 50 | SVG 80×80, quadrado | Componentes react-native-svg do `bankRegistry` (`getBankSvg`), reconstruídos a partir do bundle Hermes decompilado |
| `bancos/png/` | 46 | PNG 192×192 / 256×256 | `res/drawable-mdpi-v4/assets_images_banks_*` e `src_assets_banks_raster_*` (sufixo `_raster`) |
| `lojas-e-apps/` | 151 | PNG 300×300 | `res/drawable-mdpi-v4/src_assets_merchantlogos_*` |
| `bandeiras-cartao/png/` | 12 | PNG 40×28 (normal e `dark`) | `src_assets_invoicecardbrand_*` |
| `bandeiras-cartao/svg/` | 24 | SVG | Componentes `SvgIconFlag*` e `SvgMastercard*` |
| `mapeamento.json` | — | JSON | Palavras-chave da descrição → logo, aliases, domínios e cores das assinaturas |

## Observações
- **As logos não têm cantos arredondados.** O app aplica o raio e o recorte circular na renderização. O selo do banco sobre a logo do merchant (como o "itaú" no screenshot) também é montado na UI: são duas imagens sobrepostas.
- Bancos que só existem em SVG: BV (app, web e private), Crefisa, Dock, Midway, Méliuz, Necton, Player's Bank, QI SCD, Rede/Celcoin, Rico, Safra Financeira, SafraPay, Santander Cartões, Uber Conta, Unicred, Cartão Atacadão/Carrefour/Sam's Club.
- Bancos que só existem em PNG: Pague Veloz/Serasa (`paguevelozserasa_raster.png`) e Woop (`woop_raster.png`).
- `carrefou.png` tem esse nome no próprio APK.
- Os nomes dos SVGs seguem as chaves do registro do app, sem acento.
- Os SVGs foram gerados a partir dos props `jsx()` do código decompilado (atributos camelCase convertidos para kebab-case, e `style` expandido em atributos). Conferi todos renderizando no Chrome.
