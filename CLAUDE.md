# Portal de Plano de Marketing Turístico — Amapá

> Contexto para o Claude Code. Responder sempre em **português (pt-BR)**.

## O que é
Portal web que espelha, capítulo por capítulo, um plano de marketing turístico de 181 páginas
(destino: Amapá / "Amazônia Autêntica"). Serve como **ferramenta de apresentação ao cliente**:
o cliente só visualiza; o consultor (admin) edita. Pensado para virar um sistema modular e, no futuro,
um plano em branco reutilizável para outros destinos + chatbot de IA.

## Arquitetura (regra de ouro)
- **Arquivo único**: `index.html`. Todo o conteúdo (CSS, JS, imagens em base64, e o objeto `PLAN`) vive
  dentro do arquivo. Vanilla JS, sem frameworks.
- **Conteúdo = dados; os renderers desenham.** Não escrever HTML "solto" de conteúdo: adicionar dados
  ao objeto `PLAN` e deixar o renderer correspondente montar.
- **Online via Firebase (desde a Fase 4):** o SDK do Firebase é carregado por CDN (gstatic) e o estado
  editável sincroniza em tempo real pelo **Firestore** (fonte da verdade). Por isso o portal **não abre
  mais offline via `file://`** — requer internet. O `localStorage` virou **cache**. Há um **modo local**
  de fallback automático: enquanto `FIREBASE_CONFIG` (topo do `<script>`) estiver com `"COLE_AQUI"`, o
  app roda só com `localStorage` + login demo, como antes. Setup: ver **`FIREBASE_SETUP.md`**.

## Estrutura de dados
- `const MEDIA = { ... }` — mapa de imagens base64 (injetado ANTES de `const PLAN`).
- `const PLAN = { meta, onboarding, chapters[] }`.
  - Cada `chapter`: `{ num, nome, cor, icone, resumo, sections[] }`.
  - Cada `section`: `{ titulo, blocks[] }`.
  - Cada `block`: `{ type, ...campos }` — o `type` casa com um `case` em `renderBlock()`.
- Estado dinâmico `STATE` (`pmt_state_v1` no localStorage + doc `plano/state` no Firestore):
  `{ overrides{}, statuses{}, media{}, layout{}, _addSeq, ui{} }`.
  - `overrides` = edições inline do admin (texto por chave `data-ed`).
  - `statuses` = status das ações (todo/doing/done).
  - `layout` = grades por seção do editor modular (Fase 2). Ver bloco "EDITOR MODULAR" no código.
  - `ui` = navegação atual; **fica só no localStorage** (por-dispositivo, não sincroniza).
- **Sincronização:** `saveState()` grava no localStorage e empurra (debounce) pro Firestore via
  `cloudPush()`; `cloudInit()` assina o doc com `onSnapshot` e `applyRemote()` re-renderiza ao vivo
  (com guarda p/ não estourar o cursor enquanto o admin digita). Só o admin escreve.
- **Login:** cliente = demo client-side `cliente`/`amapa` (só vê, `pmt_auth_v1`). Admin = **Firebase Auth
  e-mail/senha** (modo nuvem) ou demo `admin`/`amapa2025` (modo local de fallback). Regra do Firestore:
  só o e-mail admin escreve; leitura pública. `apiKey` no código é público por design.

## Catálogo de widgets (renderers em renderBlock)
paragraph, lead, callout, calloutGrid, manifesto, list, labelTable, chips, cards, groupList, table,
twoCol, phases, statRow, rankList, ladder, golden, swot, benchmark, persona/personas, channel,
diagnostic (8 Ps), action (com modo `collapsible` + prioridade/status), image, infographic, gallery,
audio, video, faq, signature, expandall, placeholder.

## Como adicionar um widget novo
1. **CSS**: colar o bloco de estilo logo ANTES do comentário `/* AI panel */`.
2. **Renderer**: novo `case "meuTipo":` ANTES de `case "placeholder":` em `renderBlock()`.
3. **Usar**: adicionar `{ type:"meuTipo", ... }` em algum `blocks[]` do `PLAN`.
4. Edição inline: elementos com `data-ed="chave"` viram editáveis no modo admin; usar o helper
   `txt(chave, default)` para ler override e `esc()` para escapar HTML.

## Convenções importantes
- Ações (`type:"action"`) alimentam o anel de progresso do topo (`allActions` / `updateProgress`).
  Hoje há 6 ações estratégicas (a1–a6, sempre abertas) + 23 do backlog (r1–r23, `collapsible:true`).
- Capítulos 1–12 + Onboarding já estão 100% carregados com o conteúdo real do plano.
- Manter o tom visual "Memphis/amazônico": paleta em variáveis CSS
  (--ink #241a6e, --orange #f36a21, --green #1aa64a, --yellow #ffc415, --magenta #e5167b, --cyan #19b6e6).

## Validar antes de publicar (sempre)
O JS está dentro de `<script>…</script>`. Antes de commitar, checar a sintaxe:
```bash
# extrai o <script> e valida com node
node -e "const h=require('fs').readFileSync('index.html','utf8');const m=h.match(/<script>([\s\S]*)<\/script>/);require('fs').writeFileSync('/tmp/c.js',m[1]);" && node --check /tmp/c.js && echo "JS OK"
```
Se `node` não estiver instalado nesta máquina, validar abrindo no navegador (preview) e conferindo o
console sem erros — ex.: `python3 -m http.server` na raiz e checar que `typeof render === "function"`.
Existe um `.claude/launch.json` (não versionado) que sobe esse server para o preview.

## Publicar (GitHub Pages)
Este repositório é servido pelo GitHub Pages a partir de `index.html` na raiz.
Para publicar uma alteração:
```bash
git add index.html
git commit -m "descrição da mudança"
git push
```
O site atualiza em ~1 minuto. Para preview instantâneo enquanto edita, abrir `index.html` no navegador
(file://) e recarregar — depois dar push quando estiver bom.

## Fases
- ✅ **Fase 2 — Editor modular (grid 2D):** admin arrasta/adiciona/remove/redimensiona widgets por seção
  (alça, ghost de encaixe, anti-sobreposição, modo mobile em pilha). Opt-in por seção; "Exportar p/ PLAN"
  assa o arranjo no `index.html`. Ver bloco "EDITOR MODULAR" no código.
- ✅ **Fase 4 (parcial) — Online em tempo real:** Firestore + Firebase Auth (admin e-mail/senha). Edições
  ficam online ao vivo no GitHub Pages. **Pendente:** upload de vídeos/fotos/documentos (Firebase Storage)
  e o chatbot de IA (painel lateral é stub).
- ⬜ **Fase 1:** padronizar os widgets aba a aba.
- ⬜ **Fase 3:** versão "plano em branco" reaproveitando o motor.
- ⬜ **Multi-cliente:** hoje é um doc único `plano/state`; vários clientes/planos exigiria coleção por plano.
