---
draft: false
featureimage: https://gsfalcon.com/images/uploads/youtube-to-post.png
title: Transformando vídeos do YouTube em posts de blog com Python
date: 2026-08-09T18:59:00.000-03:00
description: Tutorial passo a passo para extrair transcrições de vídeos públicos
  do YouTube e gerar posts em Markdown automaticamente, usando Python e yt-dlp
tags:
  - Python
  - Automação
  - yt-dlp
  - YouTube
categories:
  - Python
  - Automação
  - yt-dlp
  - YouTube
cover:
  image: /images/uploads/youtube-to-post.png
  alt: Transformando vídeos do YouTube em posts de blog com Python
  caption: ""
ShowToc: true
comments: true
image: /images/uploads/youtube-to-post.png
---
Se você mantém um blog e quer transformar o conteúdo de um canal do YouTube em posts de texto, dá pra automatizar praticamente tudo: descobrir os vídeos do canal, puxar a transcrição pública (legendas) de cada um, e gerar um arquivo Markdown pronto — sem baixar nenhum vídeo. 

Neste tutorial eu mostro o script Python que uso pra isso, o passo a passo de instalação, e como resolver os erros mais comuns (bloqueio de bot, cookies, formatos indisponíveis).

## O que o script faz

- Recebe a URL da página de vídeos de um canal público do YouTube.
- Descobre todos os vídeos daquele canal.
- Para cada vídeo, tenta baixar a legenda pública (manual ou automática), com preferência por português.
- Preserva a descrição completa do vídeo (que geralmente contém links e fontes citadas).
- Gera um arquivo `.md` por vídeo, com metadados no cabeçalho (front matter) e o embed do player do YouTube no final.
- ⏭ Pula vídeos que já foram processados antes, então dá pra rodar de novo sem duplicar trabalho.

Importante: o script **não baixa vídeo nenhum**. Ele só lê metadados e legendas, que são informações públicas expostas pela própria página do YouTube.

## Dependências

Você precisa de Python 3.10+ e da biblioteca `yt-dlp`, que é o motor por trás da extração de metadados e legendas.

```bash
pip install yt-dlp
```

Não precisa de `ffmpeg` nem de nada relacionado a vídeo/áudio, já que não há download de mídia. 

## Autenticação (cookies)

O YouTube passou a exigir autenticação para várias operações em lote, mesmo em conteúdo público — sem isso, você recebe um erro do tipo `Sign in to confirm you're not a bot`.

A forma mais estável de resolver isso é exportar um arquivo `cookies.txt` do seu navegador, em vez de deixar o yt-dlp ler o cookie do navegador em tempo real (isso costuma falhar no Windows por causa de lock de arquivo enquanto o navegador está aberto).

1. Instale a extensão [Get cookies.txt LOCALLY](https://chromewebstore.google.com/detail/get-cookiestxt-locally/cclelndahbckbenkjhflpdbgdldlbecc) no seu navegador (Chrome, Brave, Edge etc.).
2. Acesse `youtube.com` estando logado normalmente.
3. Use a extensão para exportar os cookies do site como `cookies.txt`.
4. Salve esse arquivo na mesma pasta do script.

O script detecta automaticamente o `cookies.txt` se ele existir; caso contrário, tenta ler os cookies direto do navegador (menos confiável para execuções longas).

## Como o script está organizado

Em linhas gerais, o fluxo é:

1. **Descoberta**: usa `extract_flat` do yt-dlp pra listar rapidamente todos os IDs de vídeo do canal, sem processar cada página individualmente.
2. **Extração por vídeo**: para cada ID, busca os metadados completos (título, data, descrição) e tenta localizar uma faixa de legenda.
3. **Prioridade de idioma**: a busca por legenda tenta primeiro `pt-BR`, depois `pt`, depois `pt-PT`, e só então qualquer outro idioma disponível — manual antes de automática.
4. **Parsing da legenda**: as legendas vêm em formatos como VTT, SRV3 ou TTML; o script converte tudo pra texto simples, removendo timestamps, tags e linhas repetidas.
5. **Geração do Markdown**: monta um arquivo com front matter (título, data, fontes, link do vídeo) seguido da transcrição e, no final, o embed do player.

Um trecho central é a função que decide qual legenda usar:

```python
preferred = ["pt-BR", "pt", "pt-PT"]

for lang in preferred:
    if lang in subtitles:
        candidates.append(("manual", lang, subtitles[lang]))
    if lang in automatic:
        candidates.append(("automática", lang, automatic[lang]))
```

E o front matter gerado para cada post segue este formato:

```python
return f"""---
title: {yaml_quote(title)}
date: {yaml_quote(upload_date)}
sources: |
{indent_block(description, 2)}
youtube: {yaml_quote(video_url)}
---

{transcript_text}

## Vídeo

<iframe ...></iframe>
"""
```

## Rodando o script

Com o `cookies.txt` na mesma pasta, execute:

```bash
python youtube_to_posts.py "https://www.youtube.com/@nome-do-canal/videos"
```

Se você não passar nenhuma URL, o script usa uma URL padrão definida na constante `DEFAULT_CHANNEL_URL` — vale editar isso no topo do arquivo pro canal que você acompanha com mais frequência. 

Os arquivos `.md` são criados numa pasta `posts/`, um por vídeo, nomeados com um slug do título + o ID do vídeo (isso evita colisão de nomes e permite identificar vídeos já processados em execuções futuras).

## Problemas comuns e como resolver

`Sign in to confirm you're not a bot`
Falta autenticação. Siga o Passo 1 e gere o `cookies.txt`.

`Could not copy Chrome cookie database`
Isso acontece quando o yt-dlp tenta ler os cookies direto do navegador e ele está aberto (o arquivo fica travado). A solução é usar o `cookies.txt` exportado manualmente em vez de depender do navegador.

`Requested format is not available`
Esse erro é sobre formato de vídeo, mesmo o script não baixando vídeo nenhum — ele aparece porque o yt-dlp tenta resolver um formato "padrão" internamente antes de retornar os metadados. Duas coisas ajudam:

- Atualizar o yt-dlp, já que o YouTube muda o player com frequência: `pip install -U yt-dlp`.
- Passar a opção `ignore_no_formats_error: True` nas configurações do `YoutubeDL`, que instrui a biblioteca a não travar quando não encontra formatos de vídeo — o que é irrelevante pro nosso caso.

## Adaptando pra outros canais

Pra reaproveitar em qualquer canal público, só troque a URL passada como argumento (ou a constante `DEFAULT_CHANNEL_URL`). Se o conteúdo majoritário for em outro idioma, ajuste a lista `preferred` na função de extração de legendas.

Vale lembrar que a transcrição automática do YouTube nem sempre tem pontuação ou formatação de parágrafos — o script faz uma limpeza mínima, mas uma revisão editorial (manual ou com apoio de alguma IA) antes de publicar costuma deixar o texto bem mais legível.

## Download ⬇️

[YouTube to Markdown](/downloads/youtube-to-markdown.zip)
