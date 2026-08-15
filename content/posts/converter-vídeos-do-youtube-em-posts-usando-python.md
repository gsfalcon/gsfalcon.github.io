---
draft: false
featureimage: https://gsfalcon.com/images/uploads/youtube-to-post.png
title: Converter vídeos do YouTube em posts usando Python
date: 2026-08-15T06:53:00.000-03:00
description: Tutorial passo a passo para extrair transcrições de vídeos públicos
  do YouTube e gerar posts em Markdown automaticamente, usando Python e yt-dlp
tags:
  - Python
  - Automação
  - yt-dlp
  - YouTube
categories: []
cover:
  image: ""
  alt: Transformando vídeos do YouTube em posts de blog com Python
  caption: ""
ShowToc: true
comments: false
image: /images/uploads/youtube-to-post.png
---
Todo canal do YouTube que fala bastante — notícias, comentário, opinião — é, na prática, um arquivo enorme de conteúdo em texto que ninguém nunca vai ler, porque está preso dentro de vídeo. Neste tutorial eu mostro como transformar isso num blog: um pipeline em duas etapas que primeiro extrai a transcrição pública de todos os vídeos de um canal, e depois usa um modelo de IA rodando na sua própria máquina pra reescrever e organizar cada transcrição num post de verdade — com parágrafos, pontuação e divisão por assunto.

As duas etapas são scripts Python independentes. A primeira não depende de nada além do `yt-dlp`. A segunda usa o [Ollama](https://ollama.com), então tudo roda local, sem mandar seu conteúdo pra API de terceiro nenhuma.

## Visão geral do pipeline

1. **Descoberta e transcrição**: um script varre a página de vídeos de um canal, baixa a legenda pública (não o vídeo) de cada um, e gera um `.md` por vídeo com título, data, descrição original e a transcrição crua.
2. **Reescrita com IA local**: um segundo script lê cada `.md` gerado, manda a transcrição crua pra um modelo rodando no Ollama, e recebe de volta um texto reorganizado por tema, com pontuação e parágrafos.
   Nenhuma das duas etapas baixa vídeo. É tudo baseado em metadados e legendas públicas.

## Dependências

```bash
pip install yt-dlp requests
```

Além disso, instale o [Ollama](https://ollama.com/download) e baixe um modelo — no meu caso usei o `qwen3:8b`, que roda bem numa GPU de 8GB de VRAM:

```bash
ollama pull qwen3:8b
```

## Extraindo as transcrições do canal

### Autenticação

O YouTube passou a exigir autenticação pra descoberta em lote de vídeos, mesmo sendo tudo conteúdo público. A forma mais estável de resolver isso é exportar um `cookies.txt` do navegador (com a extensão "Get cookies.txt LOCALLY", por exemplo) em vez de deixar o script ler os cookies direto do navegador em tempo real — isso costuma falhar se o navegador estiver aberto.

Salve o `cookies.txt` na mesma pasta do script; ele é detectado automaticamente.

### Como o script funciona

Em três passos:

1. Usa `extract_flat` do yt-dlp pra listar rapidamente todos os IDs de vídeo do canal.
2. Para cada ID, busca metadados completos e tenta localizar uma faixa de legenda, com prioridade pra português (`pt-BR`, `pt`, `pt-PT`, depois qualquer outro idioma disponível).
3. Converte a legenda (formato VTT, SRV3 ou TTML) em texto simples, removendo timestamps e tags, e monta o Markdown final: front matter com título/data/descrição/link do vídeo, a transcrição, e o embed do player no final.
   Um detalhe que vale a pena configurar: o YouTube às vezes recusa entregar um "formato de vídeo" válido mesmo quando você não está baixando vídeo nenhum — isso derruba a extração de metadados sem necessidade. A opção `ignore_no_formats_error` resolve, já que o script não usa formato de vídeo pra nada.

### Rodando

```bash
python youtube_to_posts.py "https://www.youtube.com/@nome-do-canal/videos"
```

Os arquivos saem em `posts/`, um por vídeo, nomeados por slug do título + ID do vídeo — isso evita colisão de nomes e permite que o script pule vídeos já processados numa próxima execução.

## Reescrevendo com IA local

Depois da primeira etapa você tem uma pasta cheia de `.md` com transcrições cruas — sem pontuação, sem parágrafos, tudo em bloco único, exatamente como uma legenda automática entrega. É aqui que entra a segunda etapa.

### Por que local, e não uma API paga

Três motivos práticos:

* **Custo zero.** Rodando local não existe cobrança por requisição nem limite diário — só o tempo de processamento e a conta de luz.
* **Sem limite de volume.** APIs gratuitas de terceiros costumam ter cota diária de requisições, o que obriga a espaçar o processamento de um volume grande de posts ao longo de vários dias.
* **Sem camada de moderação externa.** Se o conteúdo do canal for opinativo ou tocar em temas sensíveis, uma API de terceiro pode recusar ou suavizar o texto por conta própria. Rodando local, é só o modelo e você — nenhuma aprovação externa no meio.
  A desvantagem é que um modelo de 8B rodando numa GPU de consumo não tem a mesma qualidade de escrita de um modelo grande de nuvem. Pra essa tarefa específica — reorganizar um texto que já existe, não criar do zero — isso costuma ser suficiente.

### O prompt

O núcleo do script é a instrução que vai pro modelo junto com cada transcrição. Ela pede pra:

1. Identificar os assuntos distintos abordados no vídeo, na ordem em que aparecem.
2. Corrigir erros óbvios de transcrição por semelhança sonora (legendas automáticas erram bastante por som — por exemplo, confundir uma expressão comum com um número parecido foneticamente) quando o contexto deixa claro qual era a palavra certa. Quando o modelo não tem confiança pra corrigir um nome próprio ou termo específico, ele marca o trecho de forma visível (`{{assim}}`) em vez de arriscar uma correção errada — isso facilita muito uma revisão posterior, porque dá pra buscar só pelos pontos duvidosos em vez de reler o post inteiro.
3. Reescrever com pontuação, parágrafos e subtítulos (`##`) por tema.
4. Preservar o conteúdo e o tom original do narrador, sem inserir ressalvas que não estavam no texto de origem.
   O modelo responde em JSON, o que facilita extrair só o texto reescrito de forma confiável.

### Um ajuste de desempenho importante

Alguns modelos, como o Qwen3, têm um modo de "raciocínio" (thinking) ativado por padrão, que gera um bloco de pensamento interno longo antes de cada resposta. Isso deixa o processamento bem mais lento sem ganho real pra essa tarefa. Desativar isso no payload da requisição ao Ollama acelera bastante:

```python
payload = {
    "model": OLLAMA_MODEL,
    "messages": [...],
    "format": "json",
    "think": False,
    "options": {"temperature": 0.3},
}
```

A temperatura baixa (0.3) também ajuda: deixa o modelo mais conservador, reduzindo o risco de ele "inventar" alguma coisa ao tentar corrigir um trecho.

### Rodando

```bash
python rewrite_posts.py --limit 15
```

Por padrão processa só os primeiros arquivos — vale sempre testar num lote pequeno antes de rodar em tudo, pra calibrar o prompt. Quando estiver satisfeito com o resultado:

```bash
python rewrite_posts.py --all
```

Os posts reescritos saem em `posts_reescritos/`, mantendo o mesmo nome de arquivo dos originais. O script pula automaticamente qualquer post que só tenha o aviso de transcrição indisponível (quando o vídeo não tinha legenda pública) e qualquer post que já tenha sido reescrito antes — então dá pra interromper e retomar sem perder trabalho.

## Próximos passos possíveis

Esse pipeline de duas etapas já entrega o essencial: do canal ao post organizado. A partir daqui dá pra estender de várias formas — gerar uma imagem de capa por post com um modelo de geração de imagem local, ou montar um terceiro passo que pega os trechos marcados como incertos e faz uma busca automática pra tentar resolver a dúvida antes de uma revisão manual final. Fica pra um próximo post.

## Downloads

| File | Download |
|---|---|
| YouTube to Markdown | [youtube-to-markdown.zip](/downloads/youtube-to-markdown.zip) |
| AI Rewrite Posts | [rewrite-posts.zip](/downloads/rewrite-posts.zip) |
