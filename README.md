# Flow Visualizer [BETA] 0.0.1

Este projeto é uma página **HTML única** (`index.html`) que renderiza visualmente um **fluxo do Blip (JSON)** como um **grafo**:

- **Nós (cards)** = estados do fluxo
- **Arestas (linhas com seta)** = transições entre estados

Funcionalidades principais:
- Upload de JSON do fluxo (arquivo local)
- Carregar `fluxo.json` via `fetch` (quando disponível)
- Zoom in / Zoom out (mouse wheel + botões)
- Pan (arrastar o fundo)
- Drag (arrastar cards)
- Busca por texto (título/id/mensagens)
- Seleção de nó (destaca conexões ligadas)
- Seleção de aresta (destaca os dois nós conectados)
- InfoBar inferior com detalhes do item selecionado

---

## 📁 Estrutura do Projeto

O projeto é composto por **um único arquivo**:

- `index.html`
  - HTML (estrutura)
  - CSS (estilos)
  - JavaScript (lógica completa)

---

## 🧱 HTML — Estrutura da Página

### `<head>`
Contém:
- Charset UTF-8
- Viewport para responsividade
- `<title>` da aba
- `<style>` com todos os estilos do projeto

### `<body>`

#### `<header>`
Barra fixa no topo com:
- Título e instruções de uso
- Upload de JSON
- Botões:
  - Ver fluxo inteiro
  - Zoom +
  - Zoom -
  - Resetar visualização
- Campo de busca
- Indicador de zoom em `%`

#### `.viewport#viewport`
Área principal que:
- captura **pan** (arrastar fundo)
- captura **zoom** (scroll do mouse)
- contém todo o conteúdo visual do grafo

#### `#board`
Container que recebe transformações:
- `translate(x,y)` para pan
- `scale(s)` para zoom

Ou seja: **zoom/pan são aplicados no board**.

#### `svg#wires`
SVG onde as **linhas** (edges) são desenhadas.
- `<defs>` define duas setas:
  - `#arrow`: seta normal
  - `#arrowBlue`: seta quando a linha está selecionada

#### `#canvas`
Container onde os **cards** (`div.node`) são criados via JavaScript.

#### `#infoBar`
Barra inferior que exibe:
- informações do nó selecionado
- informações da ligação selecionada

---

## 🎨 CSS — Estilos (Visão Geral)

### Variáveis em `:root`
Define valores globais usados no layout, por exemplo:
- cores: `--bg`, `--stroke`, `--blue`
- dimensões: `--cardW`, `--pad`
- espaçamentos: `--levelGapY`, `--nodeGapX`, `--treeGapX`, `--islandGapX`
- outros: `--shadow`, `--radius`

### Layout
- `body`: fundo escuro, sem scroll (`overflow: hidden`)
- `header`: fixo no topo com blur
- `.viewport`: área “arrastável”
- `#board`: transform-origin 0,0 para zoom/pan

### Cards `.node`
- bloco absoluto com sombra e borda lateral
- variações via `data-kind`:
  - `decision` (várias saídas)
  - `redirect`
  - `error`

### Linhas `path.edge`
- desenhadas no SVG
- clicáveis (`pointer-events: stroke`)
- usam seta via `marker-end`

### Estados visuais
- `.dim` / `.highlight`: usados na busca
- `.selected`: nó selecionado
- `.edge-selected`: linha selecionada (cor azul + espessura maior + seta azul)

---

## 🧠 JavaScript — Organização Geral

O script está organizado em blocos lógicos:

1. Helpers
2. Detecção de conteúdo (mensagens / JS / HTTP)
3. Construção do grafo
4. Organização em árvores (forest)
5. Cálculo de posições
6. Renderização dos nós
7. Espaçamento dinâmico por nível
8. Desenho das arestas
9. Seleção
10. Busca
11. Zoom e Pan
12. Drag de nós
13. Carregamento do JSON

---

## 1) Helpers

### Referências do DOM
Elementos principais da página são obtidos via `getElementById`:
- viewport, board, canvas, svg
- zoomLabel
- infoBar

### `safeStr(x)`
Converte valores em string:
- `null` / `undefined` → `""`
- outros → `.toString()`

### `escapeHtml(str)`
Escapa caracteres especiais para evitar quebra de layout e injeção de HTML:
- `& < > " '`

### `cssSafeId(id)`
Transforma um id qualquer em um id válido para DOM:
- caracteres inválidos são substituídos por `_`

### `showInfo(html)` / `hideInfo()`
Controlam a barra inferior de informações:
- `showInfo`: exibe e injeta HTML
- `hideInfo`: esconde e limpa

---

## 2) Detecção de Conteúdo

### `extractMessages(state)`
Extrai textos enviados ao usuário:
- percorre `$contentActions`
- ignora chatstate
- trata:
  - `text/plain`
  - `select+json` (texto + opções)
- retorna `string[]`

### `hasJS(state)`
Heurística para identificar uso de JavaScript:
- busca por palavras-chave no `action.type`
- ou por trechos comuns de script no JSON

### `hasHTTP(state)`
Heurística para identificar chamadas HTTP:
- presença de URLs
- endpoint `msging.net/commands`
- presença de `"method"` e `"uri"`

### `classifyKind(state)`
Classifica o card para estilização:
- `decision`: múltiplas saídas
- `redirect`: tag Redirect
- `error`: id ou título indica erro
- `normal`: padrão

---

## 3) Construção do Grafo

### `buildGraph(blipJson)`
Converte o JSON do Blip em estrutura de grafo:
- cria mapa de nós
- mapeia saídas
- conta entradas
- gera lista deduplicada de arestas

Retorno:
```
{ nodes, out, inc, edges }
```

---

## 4) Organização em Árvores (Forest)

### `computeForest(graph)`
Organiza o grafo em múltiplas árvores:
- identifica nós isolados
- define raízes explícitas ou naturais
- executa BFS para calcular:
  - parent
  - depth
  - children
- ciclos viram novas raízes

Retorno:
```
{ roots, isolated, parent, depth, children }
```

---

## 5) Cálculo de Posições

### `computePositions(graph, forest)`
Calcula coordenadas `{x,y}` dos cards:
- usa ordenação por folhas (DFS)
- centraliza pais entre filhos
- separa árvores horizontalmente
- posiciona isolados em linha inferior

Usa variáveis CSS para espaçamento.

---

## 6) Renderização

### `render(blipJson)`
Função principal:
1. monta grafo
2. calcula forest
3. calcula posições
4. limpa canvas e SVG
5. cria cards e badges
6. adiciona eventos de clique e drag
7. ajusta layout final e desenha arestas

Atualiza estado global:
```
CURRENT = { graph, forest, pos }
```

---

## 7) Espaçamento Dinâmico

### `applyDynamicLevelSpacing()`
Evita sobreposição vertical:
- mede altura real dos cards por nível
- recalcula posição Y
- aplica novos valores

---

## 8) Desenho das Arestas

### `drawAllEdges()`
Para cada ligação:
- calcula pontos de saída/entrada
- escolhe formato do path:
  - loop
  - descendo
  - voltando
  - lateral
- cria `<path>` no SVG
- adiciona evento de clique

### `scheduleEdgesRedraw()`
Redesenha arestas usando `requestAnimationFrame` para suavidade.

---

## 9) Seleção

### `toggleNodeSelection(id)`
Seleciona/deseleciona nó:
- limpa seleção de edge
- atualiza infoBar
- destaca conexões

### `toggleEdgeSelection(key)`
Seleciona/deseleciona linha:
- limpa seleção de nó
- atualiza infoBar
- destaca nós conectados

### `updateHighlights()`
Aplica classes visuais conforme seleção atual.

---

## 10) Busca

### `applySearch(q)`
Filtra visualmente os cards:
- se não contém texto → `.dim`
- se contém → `.highlight`

Busca em tempo real.

---

## 11) Zoom e Pan

### Estado da View
```
view = { x, y, s }
```

### `applyView()`
Aplica `translate` e `scale` no board.

### `fitToViewport()`
Centraliza e ajusta escala para caber tudo na tela.

### `zoomAt(x,y,factor)`
Faz zoom mantendo ponto do mouse fixo no mundo.

Eventos:
- scroll → zoom
- drag no fundo → pan
- botões → zoom/reset

---

## 12) Drag de Nós

Durante drag:
- converte mouse para coordenada do mundo
- atualiza `left/top` do card
- redesenha arestas em tempo real

Ao soltar:
- reajusta board
- redesenha arestas

Clique é ignorado se houve movimento.

---

## 13) Carregamento do JSON

### `loadFromUrl()`
Busca `fluxo.json` via `fetch` e renderiza.

### `loadFromFile(file)`
Lê arquivo local:
- `file.text()`
- `JSON.parse()`
- `render(json)`

---

## ▶️ Fluxo de Execução

1. Usuário carrega JSON
2. `render()` monta layout
3. Usuário interage:
   - zoom
   - pan
   - drag
   - busca
   - seleção

Tudo ocorre no client-side, sem backend.

---

## 📌 Observações

- Projeto totalmente client-side
- Nenhuma dependência externa
- Funciona em qualquer servidor estático
- Compatível com Firebase Hosting, Vercel, Netlify, GitHub Pages
