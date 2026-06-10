# Portal de Plano de Marketing Turístico — Amapá

> Contexto para o Claude Code. Responder sempre em **português (pt-BR)**.

## O que é
Portal web que espelha, capítulo por capítulo, um plano de marketing turístico de 181 páginas
(destino: Amapá / "Amazônia Autêntica"). Serve como **ferramenta de apresentação ao cliente**:
o cliente só visualiza; o consultor (admin) edita. Pensado para virar um sistema modular e, no futuro,
um plano em branco reutilizável para outros destinos + chatbot de IA.

## Arquitetura (regra de ouro)
- **Arquivo único, autossuficiente, offline**: `index.html`. **Sem CDN, sem fetch externo em runtime.**
  Tudo (CSS, JS, imagens em base64) vive dentro do arquivo. Tem que abrir via `file://` também.
- Vanilla JS, sem frameworks. Estado persistido em `localStorage`.
- **Conteúdo = dados; os renderers desenham.** Não escrever HTML "solto" de conteúdo: adicionar dados
  ao objeto `PLAN` e deixar o renderer correspondente montar.

## Estrutura de dados
- `const MEDIA = { ... }` — mapa de imagens base64 (injetado ANTES de `const PLAN`).
- `const PLAN = { meta, onboarding, chapters[] }`.
  - Cada `chapter`: `{ num, nome, cor, icone, resumo, sections[] }`.
  - Cada `section`: `{ titulo, blocks[] }`.
  - Cada `block`: `{ type, ...campos }` — o `type` casa com um `case` em `renderBlock()`.
- Estado dinâmico em `localStorage` (`pmt_state_v1`): `{ overrides{}, statuses{}, media{}, ui{} }`.
  - `overrides` = edições inline do admin (texto por chave `data-ed`).
  - `statuses` = status das ações (todo/doing/done).
- Login (demo, client-side, `pmt_auth_v1`): `cliente`/`amapa` (só vê) e `admin`/`amapa2025` (modo edição).

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

## Próximas fases (combinadas, ainda não feitas)
1. Padronizar os widgets aba a aba.
2. Editor modular: arrastar/adicionar/remover widgets por tela (como o dashboard) — admin monta, cliente vê.
3. Versão "plano em branco" reaproveitando o motor.
4. Com HTTPS (este hosting): login real multi-cliente, chatbot de IA (interface lateral já existe) e
   upload de vídeos/fotos/documentos.
