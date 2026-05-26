# Enrichment Runbook — imagens + features reais dos mocks

Objetivo: transformar os ~495 mocks de template (placeholder) em páginas com
**foto real da fachada** e **features reais** do estabelecimento (dono, horário,
serviços, especialidade, reviews citando nomes, Instagram). Sem inventar dados.

> Sincronização entre workspaces: só `mocks/`, `index.html` e arquivos `*.md`
> são versionados (git). `scripts/`, `data/`, `enriched/`, `outreach/` são
> locais de cada workspace. Portanto: fotos (`mocks/<slug>/photos/`) e este
> runbook sincronizam; o conteúdo de `research.md` (em `enriched/`, gitignored)
> **não** sincroniza — cole os achados no card do Paperclip e/ou aplique na sua
> própria cópia antes de commitar o `mocks/<slug>/index.html` resultante.

## Quem faz o quê

- **Research agent** → coleta features reais e atualiza `enriched/<slug>/research.md`
  (formato abaixo), depois roda `apply_research_to_mocks.py` para gravar no HTML.
- **Agente de fotos** (com Chrome) → captura a foto de capa via Google Maps.
- **Operador / orquestrador** → consolida, aplica, commita e dá push.

## Fila de trabalho

`scripts/build_photo_queue.py` gera, para cada mock SEM foto:
`scripts/_photo_queue2_<categoria>.json` = lista de `[slug, "query p/ Maps"]`.
Total atual: 495 (salão 131, barbearia 99, pet 94, lanchonete 87, oficina 84).

## A) Captura de foto (Google Maps) — precisa de Chrome

Há 2 Chrome conectados; escolha o instance "Chrome" pessoal (não o Workstation IDR).
Cuidado com contenção: **um agente de foto por vez** no browser.

Por item da fila `[slug, query]`:

1. `navigate` para `https://www.google.com/maps/search/<query url-encoded>`
2. Esperar carregar; extrair a URL da foto de capa via `javascript_tool`:
   ```js
   (() => {
     const img = document.querySelector('img[src*="gps-cs-s"], img[src*="googleusercontent.com/gps"], img[src*="streetviewpixels"]');
     return img ? img.src : 'None';
   })()
   ```
   Se a busca cair direto na ficha do lugar, a capa costuma ser o primeiro
   `img[src*="gps-cs-s"]`. Se cair na lista, clicar no 1º resultado que casa o
   endereço antes de extrair.
3. Acumular linhas `slug<TAB>url` num arquivo, depois baixar:
   ```
   py scripts/_dl_photos.py < scripts/_photo_urls.txt
   ```
   O `_dl_photos.py` reescreve o sufixo para `=w800-h600-k-no` (upscale) e grava
   em `mocks/<slug>/photos/photo1.jpg`. O template do mock já usa `photos/photo1.jpg`
   automaticamente quando existe.
4. Validar (HTTP 200 / tamanho > 10KB). Se não achar foto no Maps, registrar o
   slug em `scripts/_no_photo.txt` (fallback fica o placeholder).

## B) Features reais → research.md → HTML

Para cada slug, pesquisar (Google, Instagram, Facebook, iFood, Booksy/Trinks/Avec)
e adicionar ao topo de `enriched/<slug>/research.md` um bloco:

```
## Features reais (verificado AAAA-MM-DD)
- Tagline: <frase curta e específica da casa>
- Dono/profissionais: <Nome1>, <Nome2>   (nomes citados em reviews)
- Fundação: <ano> (se houver)
- Especialidade/diferencial: <ex: corte navalhado; buffet caseiro; banho low-stress>
- Horário: <ex: Seg-Sáb 9h-19h>
- Serviços: item1; item2; item3
- Instagram: @handle
- Facebook: facebook.com/...
- Reviews reais (curtos, entre aspas): "<trecho1>"; "<trecho2>"
- Dor latente: <D1 sem presença / D3 plataforma alheia / etc>
```

Depois aplicar ao HTML (reusa a lógica testada de `upgrade_enriched_mocks.py`):

```
py scripts/apply_research_to_mocks.py --slug <slug>
# ou em lote:
py scripts/apply_research_to_mocks.py --slugs-file scripts/_hot_new_leads.csv
py scripts/apply_research_to_mocks.py --all
```

O patcher atualiza: subtítulo do hero (tagline), seção "Sobre" (tradição/dor),
bloco de reviews (números + nomes + link Maps), chips de Instagram/Facebook e a
meta description. É idempotente.

## C) Commit & push (para convergir workspaces)

```
rtk git add mocks/ index.html ENRICHMENT-RUNBOOK.md
rtk git commit -m "feat(enrich): fotos + features reais lote <N>"
rtk git push
```

(`enriched/` não entra no git; cole o resumo das features no card do Paperclip.)

## Prioridade

1. **67 hot leads** (`scripts/_hot_new_leads.csv`) — celular + alto reviews, alvo do outreach de amanhã.
2. Demais mocks por nº de reviews decrescente, por categoria.

## Regras

- Nunca inventar foto, telefone, nome ou review. Sem dado → deixar placeholder/vazio.
- Foto deve ser do estabelecimento certo (conferir endereço antes de extrair).
- pt-BR, UTF-8, sem em-dash em qualquer texto que vá pro cliente.
