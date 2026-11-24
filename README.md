# MINDWRITE TRADER // SOLANA BOT

Painel de monitoramento quantitativo em tempo real com estética *dark terminal* de alta densidade. Núcleo neural renderizado em HTML5 Canvas a 60 FPS, conectado por curvas Bézier animadas.

**Arquivo único. Zero dependências. Sem build.**

![Visão geral do painel](docs/preview.png)

> [!IMPORTANT]
> **Isto é um preview de interface com dados simulados.**
> Não executa ordens, não conecta a carteiras, RPCs ou exchanges, e não lê dados on-chain reais. Todos os números — PnL, win rate, Sharpe, latência — são gerados por `Math.random()` para demonstrar o comportamento visual da interface. **Não é recomendação de investimento.**

---

## Índice

- [O que é](#o-que-é)
- [Módulos](#módulos)
- [Como rodar](#como-rodar)
- [Arquitetura](#arquitetura)
- [Design system](#design-system)
- [Acessibilidade](#acessibilidade)
- [Responsividade](#responsividade)
- [Performance](#performance)
- [Estrutura](#estrutura)
- [Roadmap](#roadmap)
- [Licença](#licença)

---

## O que é

Um terminal de monitoramento para bot de trading na Solana, construído como peça de front-end: a interface completa, a telemetria em tempo real e a linguagem visual, sem backend.

Serve como **referência de implementação** para três problemas que aparecem juntos em qualquer painel operacional:

1. Densidade de informação sem virar ruído
2. Animação contínua que não compete com os dados que o operador precisa ler
3. Canvas a 60 FPS que não derruba o frame rate da página

---

## Módulos

| Módulo | O que mostra | Técnica |
|---|---|---|
| **The Den** | Núcleo neural: 7 canais observáveis convergindo ao centro, 6 decisões saindo | Canvas 2D, 420 partículas em 4 camadas, curvas Bézier cúbicas |
| **Exit Simulator** | Trajetória de preço entre barreiras TP/SL, com trajetórias fantasma Monte Carlo | SVG, path recalculado por tick |
| **Histograma de volume** | Distribuição lateral das saídas | Barras CSS com transição |
| **Cluster Map** | Radar on-chain: smart money, whales, Jito tips, bundlers | Canvas 2D, varredura + nós pulsantes |
| **Decision Stack** | 6 tokens rankeados com score, ação e barra de confiança | DOM, clicável, sincroniza com o radar |
| **Slot Buyer** | Execução por janela de 400 ms, segmentada em orgânico / arb / sniper | Barras empilhadas |
| **Trade Tape** | Time & sales com cabeçalho fixo | Tabela com `position: sticky` |
| **Agent Log** | Telemetria estruturada, filtrável por nível | Filtro `SIGNAL` / `ORDER` / `RISK` |
| **Core Model** | Curva de treino e contador de estados processados | SVG + count-up animado |

Controles: **PAUSAR STREAM** congela toda a telemetria e as animações. **COLAPSAR ESTADOS** contrai o núcleo numa transição de 420 ms.

![Detalhe do núcleo neural](docs/neural-core.png)

---

## Como rodar

Não há instalação, build ou servidor.

```bash
git clone https://github.com/<usuario>/<repositorio>.git
cd <repositorio>
open index.html        # macOS
# xdg-open index.html  # Linux
# start index.html     # Windows
```

### Publicar no GitHub Pages

Como o `index.html` está na raiz, funciona direto:

1. **Settings → Pages**
2. **Source:** `Deploy from a branch`
3. **Branch:** `main` / `root` → **Save**

O painel fica disponível em `https://<usuario>.github.io/<repositorio>/`.

---

## Arquitetura

Tudo vive em `index.html`: markup, CSS e JavaScript. A decisão é deliberada — o painel precisa abrir de um `file://`, de um Pages ou de um pendrive sem passo intermediário.

```
index.html
├── <style>    design tokens + layout + componentes
├── <body>     estrutura semântica (header, 4 seções)
└── <script>   IIFE única
    ├── estado          objeto central, fonte única de verdade
    ├── telemetria      setInterval 1400 ms → muta estado → re-renderiza
    ├── render*()       funções puras estado → DOM
    ├── startDen()      rAF: núcleo neural
    └── startRadar()    rAF: radar de clusters
```

**Fluxo de dados:** um tick de telemetria muta o objeto `state` e chama as funções de render afetadas. Os dois canvas leem `state` diretamente dentro do seu próprio loop `requestAnimationFrame`, desacoplados do ciclo de 1400 ms — por isso a animação roda a 60 FPS enquanto os dados atualizam a cada 1,4 s.

**Sem framework de propósito:** o gargalo aqui é taxa de preenchimento de canvas, não reconciliação de DOM. Um framework adicionaria peso sem tocar no custo real.

---

## Design system

Derivado de um sistema composto, não de um tema avulso:

| Camada | Origem | Contribuição |
|---|---|---|
| Base | Data-Dense Dashboard | Grid de 12 colunas, `--grid-gap: 8px`, `--card-pad: 12px`, tipografia 9–12 px |
| Suplemento | Real-Time Monitoring | Indicador de conexão, estados `REAL-TIME` / `CONGELADO`, token de pulso |
| Suplemento | Financial Dashboard | `Intl.NumberFormat` pt-BR, count-up, cabeçalho fixo, decimais consistentes |
| Acento | HUD / Sci-Fi FUI | Hairlines de 1 px, marcadores de canto, glow mínimo, scanlines sutis |
| Paleta | Dark Mode (OLED) | Fundo `#020617`, superfícies `#0E1223` / `#0A0E1C` |
| Tipografia | Fira Code + Fira Sans | Mono para todo dado numérico, sans para corpo |

### Tokens de cor

```css
--bg:         #020617   /* OLED */
--surface:    #0E1223   /* cards */
--surface-2:  #0A0E1C   /* painéis internos */

--fg:         #F8FAFC   /* 17,8:1 */
--fg-soft:    #CBD5E1   /* 12,5:1 — corpo e log */
--fg-muted:   #94A3B8   /*  7,3:1 — rótulos */
--fg-dim:     #64748B   /*  3,9:1 — DECORATIVO, nunca texto */

--profit:     #22C55E
--loss:       #EF4444
--warn:       #F59E0B
--info:       #38BDF8
--hud:        #22D3EE
```

### Tokens de movimento

```css
--t-micro:   150ms                            /* hover, press */
--t-state:   240ms                            /* cards, barras */
--ease:      cubic-bezier(.4, 0, .2, 1)
```

O colapso do núcleo usa `cubic-bezier(0.77, 0, 0.175, 1)` em 420 ms. Como Canvas não aceita easing de CSS, a bézier unitária é resolvida por Newton-Raphson em vez de aproximada por um polinômio parecido — e o tween é por *retargeting*, não keyframe, então um duplo-clique redireciona a partir do valor atual em vez de reiniciar.

---

## Acessibilidade

| Critério | Implementação |
|---|---|
| Contraste de texto (WCAG AA) | Toda cor de texto ≥ 4,5:1 sobre a superfície — medido, não estimado |
| Foco visível | `:focus-visible` com anel de 2 px em todo controle; nunca removido |
| `prefers-reduced-motion` | Canvas congela em quadro estático, marquee para, colapso salta sem tween |
| Cor não é indicador único | `▲` / `▼` acompanham BUY/SELL e todos os deltas |
| Conteúdo auto-rotativo | Ticker pausa no hover e no botão de stream |
| Estado de controles | `aria-pressed` em toggles, `aria-label` descritivo, `role="img"` nos canvas |
| Alvos de toque | Expandem para 44 px em `pointer: coarse` |
| Hierarquia | `h1` único, `h2` por seção, `<caption>` em tabela |

`--fg-dim` (3,91:1) está explicitamente marcado como decorativo e aparece só em separadores e na barra de rolagem — nunca em texto.

---

## Responsividade

Testado de 375 px a ultrawide, sem rolagem horizontal em nenhum ponto. O grid de 12 colunas colapsa em três estágios (`1024px`, `1280px`, `768px` para os sub-painéis) e o rodapé de métricas do Exit Simulator passa de 5 para 3 colunas abaixo de 520 px.

<img src="docs/mobile.png" alt="Painel em 375 px" width="300">

---

## Performance

Medido com Chrome headless sob *throttle* de CPU de 6×.

| Métrica | Antes | Depois |
|---|---|---|
| Draw calls por frame (2 canvas) | ~565 ¹ | **60,5** |
| `getBoundingClientRect()` por frame | 2 (≈120 reflows/s) | **0** |
| Partículas no núcleo | 260 | **420** |
| FPS em retina (DPR 2) @ 6× throttle | 12,4 | **17,5** |

¹ O laço anterior emitia um `stroke()` e um `fill()` por raio — 520 chamadas só no núcleo.

**O que mudou:**

- **Batching por camada e cor.** Um `Path2D` por lote em vez de um por partícula. A contagem de partículas virou praticamente gratuita: subir de 260 para 420 não mexeu no número de draw calls.
- **`ResizeObserver` no lugar de `getBoundingClientRect()` por frame.** O padrão anterior forçava reflow síncrono 120×/s, justamente quando o DOM estava sujo pelo tick de telemetria.
- **Gradientes cacheados.** Recriados só no resize, não a cada quadro.
- **DPR limitado a 1,5 no núcleo.** Campo de partículas com glow não desenha texto; em retina isso corta ~44% dos pixels por frame sem perda visível. O radar segue em DPR 2 porque rotula os nós.
- **Pausa por visibilidade.** `visibilitychange` suspende o timer de telemetria; os loops `rAF` param de desenhar com a aba em segundo plano.

O gargalo real não eram os draw calls — era taxa de preenchimento. Daí o ganho vir do limite de DPR, não do batching. O batching pagou a densidade visual.

---

## Estrutura

```
.
├── index.html          # aplicação completa
├── README.md
├── LICENSE
├── .gitignore
└── docs/
    ├── preview.png     # visão geral
    ├── neural-core.png # detalhe do núcleo
    └── mobile.png      # 375 px
```

---

## Roadmap

Caminhos naturais para quem quiser estender:

- [ ] Trocar o gerador simulado por WebSocket (Helius, Triton ou RPC próprio)
- [ ] Persistir preferências de filtro em `localStorage`
- [ ] Modo claro a partir dos mesmos tokens
- [ ] Exportar a fita de negócios em CSV
- [ ] `OffscreenCanvas` + Worker para tirar o núcleo da main thread

---

## Licença

MIT — veja [LICENSE](LICENSE).
