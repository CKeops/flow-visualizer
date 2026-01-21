<<<<<<< HEAD
# Flow Visualizer [BETA] 0.0.4 — Explicação do Código
=======
# Flow Visualizer [BETA] 0.0.3
>>>>>>> 22bf2bdcd85e207c453843ae523a4439aeec496d

Este projeto renderiza um fluxo exportado do Blip (JSON) como um grafo visual:
- Cada **state** vira um **card** (“node”).
- Cada transição vira uma **linha com seta** (“edge”).
- Possui: pan/zoom, seleção, busca, drag de nós e export para PNG.

---

## Estrutura do arquivo

### `<!doctype html>` e `<html lang="pt-BR">`
Define HTML5 e idioma pt-BR.

---

## `<head>`

### Meta tags
- `charset="utf-8"`: suporta acentuação.
- `viewport`: responsividade.

### Favicon
- `media\fav.gif`: ícone da aba.

### `<title>`
Nome do documento.

---

## CSS (estilos)

### `:root` (variáveis CSS)
Define variáveis globais do tema (cores, tamanhos, gaps, etc).
Exemplos:
- `--bg`: cor do fundo.
- `--cardW`: largura base dos cards.
- `--levelGapY`: distância vertical entre níveis do grafo.
- `--nodeGapX`: distância horizontal entre nós do mesmo nível.

### `body.export-mode`
Quando ativado (no export), aumenta:
- largura dos cards
- fontes
- espaçamentos
para melhorar legibilidade da imagem exportada.

### Layout principal
- `header`: barra fixa no topo com controles.
- `.viewport`: área que recebe pan/zoom.
- `#board`: “mundo” do grafo (é nele que aplicamos translate/scale).
- `svg#wires`: onde desenhamos as linhas.
- `#canvas`: onde ficam os cards (divs dos nós).

### Cards (Nodes)
`.node` cria cartões com:
- borda colorida por tipo (normal/decision/redirect/error)
- sombra
- texto e mensagens

### Arestas (Edges)
`path.edge` define o estilo das linhas:
- stroke (cor)
- arrow marker
- clique na linha é permitido (`pointer-events: stroke`)

### Seleção
- `.selected`: destaque em um nó
- `.edge-selected`: destaque em uma linha

### Busca
- `.dim`: desbota nós que não batem
- `.highlight`: destaca nós que batem

### Info bar
`#infoBar`: painel inferior com info do nó/aresta selecionado.

---

## JS — Núcleo do sistema

### Seletores principais
Você pega referências DOM:
- `viewport`, `board`, `canvas`, `svg`, etc.

### Helpers
- `safeStr`: evita `null/undefined`.
- `escapeHtml`: evita XSS ao inserir texto do JSON no HTML.
- `cssSafeId`: transforma id em string segura para usar em `id=""`.

### Extração de mensagens do state
`extractMessages(state)`:
- percorre `$contentActions`
- procura `SendMessage`
- suporta `text/plain` e `select`
- retorna array de mensagens para exibir no card.

### Heurística de JS e HTTP
`hasJS(state)`:
- identifica se o state parece ter script (function run, JSON.parse, etc).
`hasHTTP(state)`:
- identifica URLs/indicadores de request.

### Classificação do tipo do nó
`classifyKind(state)`:
- `decision` se tem múltiplas saídas
- `redirect` se tiver tag redirect
- `error` se id/título indicarem erro
- caso contrário: normal

---

## Construção do grafo

### `buildGraph(blipJson)`
- coleta `states` em `blipJson.flow`
- cria `nodes` (Map id -> state)
- cria `out` (adjacência: id -> lista de destinos)
- cria `inc` (indegree: id -> quantas entradas)
- cria `edges` deduplicadas (lista final para desenhar linhas)

---

## Forest / raízes

### `computeForest(graph)`
Objetivo: detectar raízes e componentes.

- `isolated`: nó sem entrada e sem saída.
- `explicitRoots`: states com `root: true`
- `zeroIn`: nós com `indegree = 0`

Escolhe roots por prioridade:
1. `explicitRoots` se existirem
2. senão `zeroIn`

Depois faz BFS para montar:
- `parent`
- `depth`
- `children`

E ainda trata componentes cíclicos (sem indegree0) como novas roots.

---

## Posicionamento (layout)

### `computePositions(graph, forest)`
Cria `pos` (Map id -> {x,y}):

- Cada árvore (root) recebe um “bloco” horizontal.
- Usa DFS para medir “folhas” e distribuir nós.
- `treeGapX` separa árvores diferentes.
- `isolated` vão para uma faixa lá embaixo.

---

## Renderização

### `render(blipJson)`
1. Monta grafo (`buildGraph`)
2. Monta forest (`computeForest`)
3. Calcula posições (`computePositions`)
4. Limpa o canvas
5. Cria `div.node` para cada state:
   - título
   - badges (root, JS, HTTP)
   - id e número de saídas
   - mensagens extraídas

Cada nó recebe:
- `click`: seleciona/deseleciona nó
- `mousedown`: inicia drag do nó

Após criar tudo:
- `applyDynamicLevelSpacing()`: corrige espaçamento vertical baseado na altura real
- `fitToViewport()`: enquadra tudo
- `drawAllEdges()`: desenha linhas

---

## Ajuste vertical dinâmico

### `applyDynamicLevelSpacing()`
Mede o maior card em cada depth e recalcula o `top` de cada nó para não sobrepor.

---

## Desenho das arestas

### `drawAllEdges()`
Cria `<path>` SVG para cada edge.

Ele calcula pontos de saída/entrada (ports):
- `portX` e `portY` distribuem conexões ao longo do card.

Cria rotas diferentes:
- A -> A (loop)
- descendo (depth aumenta): bottom->top
- voltando/subindo: corredor lateral
- lateral: right->left

Cada linha:
- tem `data-key`, `data-from`, `data-to`
- recebe `click` para seleção da linha

---

## Seleção e destaques

### `toggleNodeSelection(id)`
Marca o nó como selecionado e destaca as arestas conectadas.

### `toggleEdgeSelection(key)`
Seleciona uma aresta específica e destaca também os nós envolvidos.

### `updateHighlights()`
Aplica classes CSS corretas conforme `selectedNodeId` ou `selectedEdgeKey`.

---

## Busca

### `applySearch(q)`
Desbota nós que não batem e destaca os que batem no texto.

---

## Pan/Zoom

### `view = {x,y,s}`
- `x,y`: deslocamento do board
- `s`: scale

### `applyView()`
Aplica `translate + scale` em `#board`.

### `fitToViewport()`
Calcula escala e posição para caber todo conteúdo na viewport.

### `zoomAt()`
Da zoom centrado no ponteiro do mouse.

Eventos:
- `mousedown` no fundo: inicia pan
- `mousemove`: move pan ou arrasta nó
- `wheel`: zoom
- botões: zoomIn/zoomOut/reset/fit

---

## Load JSON
- `loadFromUrl()` busca `fluxo.json`
- `loadFromFile(file)` lê upload e faz `render(json)`

---

## Export (Baixar Imagem)

Ao clicar em "Baixar Imagem do Fluxo":
1. Entra em `export-mode` (card maior + fonte maior)
2. Zera pan/zoom
3. Ajusta viewport para “não recortar”
4. Usa `html2canvas` para capturar em PNG
5. Faz download via Blob
6. Restaura tudo ao final

Inclui um cálculo de `captureScale` para evitar canvas gigante:
- limita lado máximo (`MAX_SIDE`)
- limita área máxima (`MAX_AREA`)

---

## Dependências
Carregadas via CDN:
- html2canvas
- jspdf (está importado mas no momento você está baixando PNG, não PDF)

---
