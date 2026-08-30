# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este repositório

Um único arquivo: [trailer.html](trailer.html). Página autocontida (HTML + CSS + JS inline, sem build, sem dependências instaladas, sem `package.json`) que exibe o trailer de *Kill Bill: Vol. 1* dentro de um player próprio construído sobre o embed do YouTube com `controls=0`. Todo o CSS fica em um `<style>` no `<head>`; todo o JS em uma IIFE `"use strict"` no fim do `<body>`.

## Rodar

Precisa ser servido por HTTP — aberto via `file://` o YouTube devolve erro 153 e a página cai para a tela de contingência (comportamento intencional e já tratado no código, não é bug a corrigir).

```bash
python -m http.server 8000   # depois abrir http://localhost:8000/trailer.html
```

Não há build, lint nem suíte de testes. Verificação é manual no navegador; os caminhos que importam são os descritos em "Matriz de verificação" abaixo.

## Arquitetura

### A ideia central: o YouTube reproduz, mas nunca aparece

O iframe do YouTube fica em `.video__stage`, que tem `pointer-events: none`; por cima dele há `#trailer-shield`, um retângulo transparente que captura todo clique sobre o quadro. A interface do YouTube nunca é acionada. Os controles próprios (`.dialog__bar` no topo, `.ctrl` embaixo) flutuam em `z-index: 6` sobre o vídeo com gradientes que escondem o cabeçalho e o rodapé de sugestões que o YouTube desenha quando pausado. O `<iframe>` recebe `scale(1.06)` para o mesmo fim.

Consequência prática: **qualquer camada nova sobre o vídeo precisa declarar seu z-index dentro dessa pilha** — capa `1`, máscara/escudo `2`, endcard e fallback `5`, barras `6`.

### Máquina de estados por classe no `.dialog`

O JS quase nunca mexe em `style`; ele liga e desliga classes e o CSS decide o resto:

- `is-cover` — estado inicial (já vem no HTML). Mostra só a miniatura e o botão grande; o resto da barra de controles fica oculto.
- `is-starting` — entre o toque e o `onReady`: a capa continua sob um véu translúcido para o quadro não piscar preto.
- `is-idle` — barras recolhidas por inatividade.
- `is-fs` — contingência de tela cheia por CSS quando a Fullscreen API não é concedida.
- `is-rotated` — giro de 90° por CSS quando `screen.orientation.lock` falha.

Além disso, `#trailer-controls[data-disabled]` desabilita a barra inteira (opacidade + `pointer-events`), e o bloqueio precisa ser reafirmado nos filhos porque eles têm `pointer-events: auto`.

### Facade: nada do YouTube antes do primeiro toque

A página carrega apenas a miniatura (`i.ytimg.com/vi/<ID>/maxresdefault.jpg`, com `src` literal no HTML para o download começar no parse). O script `iframe_api` só é baixado em `startFromCover()`. `paintCover()` degrada para `hqdefault` medindo `naturalWidth === 120 && naturalHeight === 90` — o CDN responde **200 com um cinza 120×90** quando `maxres` não existe, então `onerror` sozinho não detectaria.

### O botão único

`#trailer-toggle` é o mesmo elemento na capa e nos controles; `paintToggleSkin()` só troca a classe (`cover__play` ↔ `ctrl__btn`) e revela/oculta o texto. Não duplicar esse botão.

### Ciclo de vida do player

`mountTrailer()` → `loadApi()` (promise única, memoizada em `apiPromise`) → `createTrailerPlayer()` → `onPlayerReady()`. Um `watchdog` de 6500 ms e um timeout de 8000 ms no carregamento da API levam a `showFallback()`. `destroyTrailerPlayer()` destrói o player e recria o `<div id="trailer-player">` do zero, porque a API do YouTube substitui esse nó pelo iframe.

**A capa só é escondida no primeiro `PLAYING`, nunca em `onPlayerReady()`** — pronto não é o mesmo que tocando. Ver "Detalhes fáceis de quebrar".

`isHttp` (protocolo `http`/`https`) governa duas escolhas: `playerVars.origin` só é enviado sob HTTP, e o host é `youtube-nocookie.com` sob HTTP / `youtube.com` fora dele.

### Vedação: a interface do YouTube nunca é visível nem alcançável

A única interação com o YouTube que o usuário pode alcançar é play/pause. Três mecanismos independentes sustentam isso, e cada um cobre o que os outros não cobrem:

- **Ponteiro** — `pointer-events: none` no `.video__stage` tira o iframe do hit-testing; `#trailer-shield` por cima é a segunda barreira e é quem trata o play/pause. Efeito colateral desejável: o clique direito abre o menu do documento pai, nunca o "Copiar URL do vídeo" do YouTube.
- **Foco** — `onPlayerReady()` marca o iframe com `tabindex="-1"` e `aria-hidden="true"`. `pointer-events` não barra foco: sem isso o `Tab` entra no documento do YouTube e, a partir dali, os atalhos `f`/`Escape` daqui deixam de receber tecla.
- **Pixel** — a barra de título é preta sólida (`--curtain`), porque a faixa de fade do gradiente antigo ainda deixava passar o título e a marca que o YouTube desenha no alto; a barra de controles mantém o gradiente. Somam-se o `scale(1.06)` no iframe cortando as bordas, `IDLE_MS` recolhendo as barras só depois que o cabeçalho do YouTube já saiu, e `finishTrailer()` encerrando antes da tela de sugestões do fim.
- **Estado** — a capa cobre o quadro até haver reprodução de fato, o que fecha o único estado em que o embed mostra a interface inteira (ver abaixo).

**Não existe painel cobrindo o quadro na pausa, e não deve ser adicionado.** Foi testado: o overlay de pausa do YouTube (cabeçalho, marca, grade de sugestões) depende de evento de mouse nascido **dentro** do iframe, e `pointer-events: none` garante que nenhum chegue lá — a pausa via `postMessage` não conta como interação. Medição em Chrome e WebKit, 25 s pausado: capturas byte a byte idênticas, quadro limpo. Ou seja, a barreira de ponteiro já cobre esse vetor de graça. Um painel opaco ali só produz tela preta ao pausar, escondendo o frame do filme sem esconder nada do YouTube. `rel: 0` apenas restringe a grade ao mesmo canal e `modestbranding` foi descontinuado em 2023 — não conte com nenhum dos dois.

### O iPhone já teve um caminho próprio; não reintroduzir

Houve uma exceção só para iPhone: escudo removido e `playsinline: 0`, para que o toque nascesse **dentro** do iframe e o iOS concedesse o fullscreen nativo (`playVideo()` por `postMessage` não atravessa a fronteira de origem). Foi **removida de propósito** — era o único ponto em que a interface do YouTube ficava tocável. O iPhone agora segue a rota comum e cai na contingência que já existia: `requestFullscreen` no `.dialog` falha no WebKit → `.is-fs` cobre a viewport por CSS → `screen.orientation.lock` é negado → `is-rotated` gira 90°.

Para tela cheia em geral há três rotas encadeadas: `requestFullscreen` no `.dialog` (nunca no iframe) → `screen.orientation.lock("landscape")` em aparelho de mão → giro por CSS (`is-rotated`) quando o lock é negado. `isHandheld()` decide por capacidades (ponteiro grosso + toque + menor lado ≤ 820px), não por sniffing.

## Convenções deste arquivo

- **JS em ES5 com `var`**, funções nomeadas e `function(){}` em callbacks. É o estilo consistente do arquivo inteiro (única exceção: um `const` em `paintSeek()`). Isso diverge da preferência global de `const`/`let`; ao editar, seguir o estilo local a menos que o usuário peça a migração.
- **Sem `innerHTML` com dado dinâmico** — só `textContent` e `createElement`. O único `innerHTML` é `stage.innerHTML = ""` (limpeza).
- **Tokens CSS** em `:root` (`--ink`, `--panel`, `--line`, `--bone`, `--smoke`, `--sun`, mais as três famílias de fonte). Cores novas viram token.
- **Comentários em português explicam o *porquê*, não o *o quê*** — quase todo bloco não óbvio tem um comentário justificando a escolha (por que `svh` e não `dvh`, por que `pointerover` e não `pointerenter`, por que a margem negativa no `::-webkit-slider-thumb`). Mantenha esse padrão: ao alterar uma dessas decisões, atualize o comentário junto.
- Unidades: `rem`/`clamp()` no que é novo; ainda há `px` em regras antigas.
- Ícones vêm do Material Symbols Outlined trocando `textContent` do `<span>`.

## Detalhes fáceis de quebrar

- **`paintSeek()`**: a barra de posição pinta o trecho percorrido com um gradiente de duas paradas no mesmo ponto (`--fill`), porque WebKit/Blink não têm pseudo-elemento de progresso. **Toda escrita em `seek.value` precisa chamar `paintSeek()`**, senão o preenchimento congela.
- **A capa só sai no primeiro `PLAYING`, nunca em `onPlayerReady()`.** Pronto não é tocando: se o navegador recusar o `playVideo()` — política de autoplay, o caso comum em aparelho de mão e janela estreita —, esconder a capa em `onReady` deixa o quadro exposto com a tela inicial do YouTube: título, nome do canal, botão vermelho no meio e a marca no rodapé. O botão central e a marca não estão ao alcance das barras, que só cobrem topo e base; nenhum ajuste de opacidade nelas resolve isso. Com a capa no lugar, quem não conseguiu autoplay simplesmente toca no play dos controles próprios.
- **`finishTrailer()`** pausa e volta ao início ~1,1 s antes do fim para a tela de sugestões do YouTube nunca aparecer. Não substituir por confiar apenas no evento `ENDED`.
- **`IDLE_MS = 4000`** (3000 do overlay do YouTube + 1000 de folga): as barras próprias só recuam depois que o cabeçalho do YouTube já saiu. Encurtar esse valor expõe a interface do YouTube.
- **`is-rotated` usa `svh`/`svw`, nunca `dvh`/`dvw`** — a unidade dinâmica oscila enquanto a barra do navegador encolhe e vira tremor de layout no overlay girado.
- Trocar o vídeo é trocar `VIDEO_ID` no JS **e** o `src` literal da miniatura no HTML (mais os títulos em `#dialog-name` e `#cover-name`).

## Matriz de verificação manual

Ao mexer no player, conferir: desktop (mouse, teclado `f`/`Escape`, e `Tab` percorrendo os controles sem o foco entrar no iframe), pausa no meio do vídeo (frame congelado à vista, sem nada do YouTube e sem tela preta, e o clique no quadro retoma), aparelho de mão Android (lock de orientação real), iPhone (tela cheia por `.is-fs` + `is-rotated`, já que não há fullscreen nativo), e a página aberta por `file://` (deve mostrar a contingência do erro 153, com "Tentar novamente" funcionando).

Boa parte disso é automatizável com o `playwright-cli`, **desde que em `--headed`**: em headless o iframe do YouTube não é decodificado e o quadro sai preto em toda captura, o que faz qualquer verificação visual passar por engano.
