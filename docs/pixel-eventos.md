# Mapeamento e Auditoria de Eventos do Meta Pixel

Este documento descreve a implementação completa do Meta Pixel no projeto **Inglês que Abre Portas**, os eventos rastreados no funil, seus parâmetros, gatilhos de disparo, deduplicação e salvaguardas de privacidade.

---

## 1. Informações Gerais

- **Meta Pixel ID:** `1471201201859881`
- **Arquivo Principal:** `index.html`
- **Modo SPA:** A aplicação é uma Single Page Application; o evento `PageView` carrega e dispara estritamente **uma única vez** no carregamento inicial da página através do código oficial do Meta Pixel no `<head>` (e `<noscript>` no `<body>`).
- **Modo de Depuração Local:** Para visualizar logs descritivos dos eventos disparados no console durante testes, utilize o parâmetro `?pixel_debug=1` na URL.

---

## 2. Tabela de Unificação e Mapeamento de Eventos (Deduplicação)

| Chamada Antiga | Evento Novo | Tipo | Gatilho / Momento | Parâmetros | Política de Deduplicação |
|---|---|---|---|---|---|
| *(N/A)* | `PageView` | Standard | Carregamento da página no `<head>` | `{ id: '1471201201859881' }` | **1 vez por carregamento** de página. Não redispara ao avançar telas da SPA. |
| `track("QuizStart")` | `ViewContent` | Standard | Início do quiz (`start()`, exibição da 1ª pergunta) | `{ content_name: 'Quiz Inglês que Abre Portas', content_category: 'Quiz' }` | **1 vez por sessão** (`sessionStorage.getItem('pixel_ViewContent')`). Substitui o antigo `QuizStart`. |
| *(N/A)* | `QuizStep` | Custom | Exibição de cada etapa do quiz (`step()`) | `{ step: <número 1-N>, step_name: <identificador> }` | **1 vez por pergunta por sessão** (`pixel_step_<n>`). Não envia respostas nem dados pessoais. |
| `track("QuizComplete_" + b)` | `Lead` | Standard | Conclusão do quiz e exibição da tela de resultado (`result()`) | `{ content_name: 'Quiz concluído', content_category: 'Resultado do Quiz' }` | **1 vez por sessão** (`sessionStorage.getItem('pixel_Lead')`). Substitui `QuizComplete_*`. |
| `track("InitiateCheckout")` | `InitiateCheckout` | Standard | Clique no botão `#buy` ou qualquer link/botão que direcione ao GGCheckout (`CFG.CHECKOUT_URL` ou domínio `ggcheckout`) | `{ content_name: 'Inglês que Abre Portas', currency: 'BRL', value: CFG.OFFER_VALUE }` | **1 vez por clique intencional** (delegação global única no `document`, trava contra cliques repetidos/duplo clique e navegação segura com atraso de 250 ms). Não dispara no `#modal-cta-buy`. |
| `track("VideoPlay")` | `VideoPlay` | Custom | Play no vídeo de depoimento (`initVideo()`) | *(sem parâmetros extras)* | Protegido por flag booleana `tPlay` (1 vez por play). |
| `track("VideoMetade")` | `VideoMetade` | Custom | 50% de reprodução do vídeo (`timeupdate`) | *(sem parâmetros extras)* | Protegido por flag booleana `tMetade` (1 vez aos 50%). |
| `track("VideoCompleto")` | `VideoCompleto` | Custom | Conclusão do vídeo (`ended`) | *(sem parâmetros extras)* | Protegido por flag booleana `tEnd` (1 vez ao finalizar). |
| `track("Objetivo_" + b)` | `Objetivo_A` / `Objetivo_B` | Custom | Escolha da ramificação do quiz em `pick()` | *(sem parâmetros extras)* | Dispara apenas na seleção da pergunta de objetivo. |
| `track("OfertaVista")` | `OfertaVista` | Custom | Exibição da seção de oferta na viewport | *(sem parâmetros extras)* | Desconectado via `IntersectionObserver.disconnect()` no primeiro disparo. |

> **Nota sobre Compra (Purchase):** O evento `Purchase` **não** é disparado no frontend do site institucional, pois a transação e confirmação de pagamento ocorrem diretamente no **GGCheckout** e são atribuídas via **webhook server-side no UTMizey** (e API de Conversões da Meta), evitando duplicações ou falsos positivos.

---

## 3. Integração com UTMizey (Rastreamento + Pixel de Otimização) e GGCheckout

- **Script de rastreamento instalado no `<head>`:** `https://utmizey.com.br/v3/utmizey.js` com `data-utmizey-id="2602fc0d-7a48-4657-bc74-94fd23eb893d"`, `data-utmizey-checkout="ggcheckout.app,ggcheckout.com"`, `data-utmizey-hotmart` e `defer`.
- **Pixel de otimização instalado antes de `</head>`:** `https://utmizey.com.br/v3/pixel.js` com `data-utmizey-pixel="dbd3139e-80cb-4d79-952f-57e21133da04"` e `defer`.
- **Por que `data-utmizey-checkout` e `data-utmizey-hotmart` são usados:** Na versão v3 do `utmizey.js` e do `pixel.js`, o domínio `ggcheckout.app` não consta na regex interna padrão de checkouts e o parâmetro `sck` só é empacotado quando `data-utmizey-hotmart` (ou domínio `cakto`) está ativo. Os dois atributos oficiais garantem que ambos os scripts reconheçam links do GGCheckout e que o `utmizey.js` gere `sck` (`u1.<click_id>.<campaign_id>.<adset_id>.<ad_id>`), `utm_*` e `utmizey_vid`.
- **Coordenação de eventos e deduplicação (`setupFbqCoordinator`):**
  - `PageView`: O `<head>` dispara imediatamente uma única vez; o `PageView` adicional emitido pelo `pixel.js` é suprimido para evitar duplicação, mantendo o `fbq('init', '1471201201859881', { external_id })` para Correspondência Avançada.
  - `InitiateCheckout`: Quando o `pixel.js` dispara `InitiateCheckout` (com `eventID` deduplicado junto à CAPI `/v1/evento`), o coordenador enriquece os parâmetros com `{ content_name: 'Inglês que Abre Portas', currency: 'BRL', value: CFG.OFFER_VALUE }` e impede disparo redundante em `handleGGCheckoutClick`. Se o `pixel.js` não disparar (ex.: bloqueador ou elemento alternativo), o fallback local dispara normalmente.
  - `Lead`: Caso o `pixel.js` dispare `Lead` pela regra configurada no painel do UTMizey (atualmente `contains: "Começar o teste grátis"`), a sessão é marcada em `pixel_Lead` para evitar um segundo `Lead` em `result()`.
  - `AddToCart`: Disparado pelo `pixel.js` conforme regra configurada no painel (`url: "ggcheckout"`).
  - Preservação de parâmetros no checkout: `handleGGCheckoutClick` aplica `applyUtmizeyParams()` (`window.__utmizey.decorate`) de forma idempotente antes de redirecionar após 250 ms, e `result()` chama `window.__utmizey.refresh()` ao montar `#buy`.

### Etapas manuais obrigatórias nos painéis (não realizadas pelo código do site)

1. **Painel do UTMizey (`utmizey.com.br`):**
   - Em **Integrações → Anúncios**, conectar a conta do Meta Ads (para cruzar gasto por campanha/conjunto/anúncio).
   - Em **Integrações → Pixel**, cadastrar o Pixel `1471201201859881` e o Token da API de Conversões da Meta (para envio server-side de `Purchase`).
   - Em **Integrações → Webhooks**, selecionar **GG Checkout** (disponível via conector de webhook do UTMizey) e copiar a **URL exclusiva de webhook** gerada.
   - Em **Integrações → UTMs**, copiar o modelo padrão de parâmetros de URL com macros do Meta (`nome|id`) e colar nos anúncios.
2. **Painel do GGCheckout:**
   - Acessar **Configurações → Webhooks**, adicionar um novo webhook com a URL gerada pelo UTMizey e marcar os eventos de pedido (como pagamento aprovado/Pix pago, recusado, reembolso e chargeback).
   - Realizar uma venda real de teste e validar no **Conector de Webhook** / **Eventos** do UTMizey se `valor`, `status`, `sck` e `utm_*` foram mapeados corretamente, conferindo no Gerenciador de Eventos da Meta que apenas uma fonte envia `Purchase` (evitando duplicidade).

---

## 4. Parâmetros e Proteção de Dados (LGPD)

- **QuizStep:** Os valores enviados em `step_name` são rótulos internos das fases técnicas (ex.: `idade`, `urg`, `dor`, `mot`, `Se apresentar`, `info`, `social`).
- **Nenhum Dado Sensível:** O script **não coleta nem envia** texto de respostas escolhidas pelo usuário, idade selecionada, nome, telefone, e-mail ou dados comportamentais sensíveis.
- **Resiliente a Bloqueadores:** As funções de redirecionamento para o checkout continuam operando normalmente mesmo se os scripts do Facebook ou do UTMizey forem bloqueados por extensões (uBlock Origin, AdGuard, Brave Shields, etc.).

---

## 5. Testes e Modo de Depuração

Para validar o rastreamento em ambiente local ou de homologação:
1. Abra a URL com o parâmetro: `http://localhost:PORT/index.html?pixel_debug=1`
2. Abra o Console das Ferramentas de Desenvolvedor (`F12` -> `Console`).
3. Observe as mensagens formatadas:
   ```text
   [Pixel Debug] Evento disparado: PageView { id: '1471201201859881' }
   [Pixel Debug] Evento disparado: ViewContent (disparo #1) { content_name: 'Quiz Inglês que Abre Portas', content_category: 'Quiz' }
   [Pixel Debug] Evento disparado: QuizStep (disparo #1) { step: 1, step_name: 'q1' }
   [Pixel Debug] Evento disparado: Lead (disparo #1) { content_name: 'Quiz concluído', content_category: 'Resultado do Quiz' }
   [Pixel Debug] Evento disparado: InitiateCheckout (disparo #1) { content_name: 'Inglês que Abre Portas', currency: 'BRL', value: 39.99 }
   ```

