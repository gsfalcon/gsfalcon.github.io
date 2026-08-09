---
title: Transformando vídeos do YouTube em posts de blog com Python
date: 2026-08-09T18:59:00.000-03:00
draft: false
description: Tutorial passo a passo para extrair transcrições de vídeos públicos
  do YouTube e gerar posts em Markdown automaticamente, usando Python e yt-dlp
tags:
  - 🐍 Python
  - 🧩 yt-dlp
  - 🤖 Automação
  - ""
categories: []
cover:
  image: /images/uploads/youtube-to-post.png
  alt: Transformando vídeos do YouTube em posts de blog com Python
  caption: Transformando vídeos do YouTube em posts de blog com Python
ShowToc: true
comments: true
---
Se você mantém um blog e quer transformar o conteúdo de um canal do YouTube em posts de texto, dá pra automatizar praticamente tudo: descobrir os vídeos do canal, puxar a transcrição pública (legendas) de cada um, e gerar um arquivo Markdown pronto — sem baixar nenhum vídeo.

Neste tutorial eu mostro o script Python que uso pra isso, o passo a passo de instalação, e como resolver os erros mais comuns (bloqueio de bot, cookies, formatos indisponíveis).

## O que o script faz

- Recebe a URL da página de vídeos de um canal público do YouTube.
- Descobre todos os vídeos daquele canal.
- Para cada vídeo, tenta baixar a legenda pública (manual ou automática), com preferência por português.
- Preserva a descrição completa do vídeo (que geralmente contém links e fontes citadas).
- Gera um arquivo `.md` por vídeo, com metadados no cabeçalho (front matter) e o embed do player do YouTube no final.
- Pula vídeos que já foram processados antes, então dá pra rodar de novo sem duplicar trabalho.

Importante: o script **não baixa vídeo nenhum**. Ele só lê metadados e legendas, que são informações públicas expostas pela própria página do YouTube.

## Dependências

Você precisa de Python 3.10+ e da biblioteca `yt-dlp`, que é o motor por trás da extração de metadados e legendas.

```bash
pip install yt-dlp
```

Não precisa de `ffmpeg` nem de nada relacionado a vídeo/áudio, já que não há download de mídia.

## Passo 1: autenticação (cookies)

O YouTube passou a exigir autenticação para várias operações em lote, mesmo em conteúdo público — sem isso, você recebe um erro do tipo `Sign in to confirm you're not a bot`.

A forma mais estável de resolver isso é exportar um arquivo `cookies.txt` do seu navegador, em vez de deixar o yt-dlp ler o cookie do navegador em tempo real (isso costuma falhar no Windows por causa de lock de arquivo enquanto o navegador está aberto).

1. Instale a extensão **"Get cookies.txt LOCALLY"** no seu navegador (Chrome, Brave, Edge etc.).
2. Acesse `youtube.com` estando logado normalmente.
3. Use a extensão para exportar os cookies do site como `cookies.txt`.
4. Salve esse arquivo na mesma pasta do script.

O script detecta automaticamente o `cookies.txt` se ele existir; caso contrário, tenta ler os cookies direto do navegador (menos confiável para execuções longas).

## Passo 2: como o script está organizado

Em linhas gerais, o fluxo é:

1. **Descoberta**: usa `extract_flat` do yt-dlp pra listar rapidamente todos os IDs de vídeo do canal, sem processar cada página individualmente.
2. **Extração por vídeo**: para cada ID, busca os metadados completos (título, data, descrição) e tenta localizar uma faixa de legenda.
3. **Prioridade de idioma**: a busca por legenda tenta primeiro `pt-BR`, depois `pt`, depois `pt-PT`, e só then qualquer outro idioma disponível — manual antes de automática.
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

## Passo 3: rodando o script

Com o `cookies.txt` na mesma pasta, execute:

```bash
python youtube_to_posts.py "https://www.youtube.com/@nome-do-canal/videos"
```

Se você não passar nenhuma URL, o script usa uma URL padrão definida na constante `DEFAULT_CHANNEL_URL` — vale editar isso no topo do arquivo pro canal que você acompanha com mais frequência.

Os arquivos `.md` são criados numa pasta `posts/`, um por vídeo, nomeados com um slug do título + o ID do vídeo (isso evita colisão de nomes e permite identificar vídeos já processados em execuções futuras).

## Problemas comuns e como resolver

**`Sign in to confirm you're not a bot`**
Falta autenticação. Siga o Passo 1 e gere o `cookies.txt`.

**`Could not copy Chrome cookie database`**
Isso acontece quando o yt-dlp tenta ler os cookies direto do navegador e ele está aberto (o arquivo fica travado). A solução é usar o `cookies.txt` exportado manualmente em vez de depender do navegador.

**`Requested format is not available`**
Esse erro é sobre formato de *vídeo*, mesmo o script não baixando vídeo nenhum — ele aparece porque o yt-dlp tenta resolver um formato "padrão" internamente antes de retornar os metadados. Duas coisas ajudam:

- Atualizar o yt-dlp, já que o YouTube muda o player com frequência: `pip install -U yt-dlp`.
- Passar a opção `ignore_no_formats_error: True` nas configurações do `YoutubeDL`, que instrui a biblioteca a não travar quando não encontra formatos de vídeo — o que é irrelevante pro nosso caso.

## Adaptando pra outros canais

Pra reaproveitar em qualquer canal público, só troque a URL passada como argumento (ou a constante `DEFAULT_CHANNEL_URL`). Se o conteúdo majoritário for em outro idioma, ajuste a lista `preferred` na função de extração de legendas.

Vale lembrar que a transcrição automática do YouTube nem sempre tem pontuação ou formatação de parágrafos — o script faz uma limpeza mínima, mas uma revisão editorial (manual ou com apoio de alguma IA) antes de publicar costuma deixar o texto bem mais legível.

## Código completo

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
YouTube -> Markdown (etapa 1)

- Não baixa vídeos.
- Tenta obter a transcrição/legendas públicas de cada vídeo.
- Gera um .md por vídeo na pasta "posts".
- Preserva a descrição completa do vídeo como "fontes".
- Coloca o embed do YouTube no final.
- Se não houver transcrição, usa "TRANSCRIÇÃO NÃO DISPONÍVEL".
- Idioma preferencial: português do Brasil.

Requisitos:
    pip install yt-dlp

Uso:
    python youtube_to_posts.py "https://www.youtube.com/@nome-do-canal/videos"
"""

from __future__ import annotations

import re
import sys
import time
from datetime import datetime, timezone
from pathlib import Path

try:
    from yt_dlp import YoutubeDL
except ImportError:
    print("ERRO: yt-dlp não está instalado.")
    print("Execute: pip install yt-dlp")
    sys.exit(1)


DEFAULT_CHANNEL_URL = "https://www.youtube.com/@nome-do-canal/videos"
OUTPUT_DIR = Path("posts")
COOKIES_FILE = Path("cookies.txt")
COOKIES_FROM_BROWSER = "brave"


def cookie_opts() -> dict:
    """Usa cookies.txt se existir; senão tenta ler direto do navegador."""
    if COOKIES_FILE.exists():
        return {"cookiefile": str(COOKIES_FILE)}
    return {"cookiesfrombrowser": (COOKIES_FROM_BROWSER,)}


def slugify(text: str, max_length: int = 100) -> str:
    """Cria um nome de arquivo seguro, mantendo caracteres Unicode."""
    text = (text or "").strip().lower()
    text = re.sub(r"[\\/:*?\"<>|]", "", text)
    text = re.sub(r"\s+", "-", text)
    text = re.sub(r"-+", "-", text)
    text = text.strip("-.")
    return text[:max_length].rstrip("-") or "video"


def yaml_quote(value: str) -> str:
    """Escapa uma string para YAML com aspas duplas."""
    value = str(value or "")
    value = value.replace("\\", "\\\\").replace('"', '\\"')
    value = value.replace("\r", "").replace("\n", "\\n")
    return f'"{value}"'


def format_date(upload_date: str | None) -> str:
    """Converte YYYYMMDD para YYYY-MM-DD."""
    if not upload_date:
        return ""
    try:
        return datetime.strptime(upload_date, "%Y%m%d").strftime("%Y-%m-%d")
    except ValueError:
        return ""


def clean_transcript(text: str) -> str:
    """Limpeza mínima; a revisão editorial ficará para a etapa de IA."""
    if not text:
        return ""

    text = text.replace("\r\n", "\n").replace("\r", "\n")
    text = re.sub(r"[ \t]+", " ", text)

    # Remove excesso de linhas, mas preserva parágrafos.
    text = re.sub(r"\n{3,}", "\n\n", text)

    return text.strip()


def extract_transcript(info: dict) -> str | None:
    """
    Tenta obter legendas públicas.

    Prioridade:
    1. pt-BR
    2. pt
    3. pt-PT
    4. outras legendas disponíveis

    O yt-dlp retorna as faixas como URL; nós as baixamos como texto,
    sem baixar o vídeo.
    """
    subtitles = info.get("subtitles") or {}
    automatic = info.get("automatic_captions") or {}

    candidates = []

    # Preferência explícita por português brasileiro.
    preferred = ["pt-BR", "pt", "pt-PT"]

    for lang in preferred:
        if lang in subtitles:
            candidates.append(("manual", lang, subtitles[lang]))
        if lang in automatic:
            candidates.append(("automática", lang, automatic[lang]))

    # Depois, qualquer legenda manual.
    for lang, tracks in subtitles.items():
        if lang not in preferred:
            candidates.append(("manual", lang, tracks))

    # Por último, qualquer legenda automática.
    for lang, tracks in automatic.items():
        if lang not in preferred:
            candidates.append(("automática", lang, tracks))

    for kind, lang, tracks in candidates:
        # Prefere VTT, depois SRV3/TTML.
        formats = []
        for track in tracks:
            ext = (track.get("ext") or "").lower()
            if ext in {"vtt", "srv3", "ttml"}:
                formats.append(track)

        if not formats:
            formats = tracks

        # Ordenação simples por preferência de formato.
        formats.sort(
            key=lambda x: {
                "vtt": 0,
                "srv3": 1,
                "ttml": 2,
            }.get((x.get("ext") or "").lower(), 9)
        )

        for track in formats:
            url = track.get("url")
            if not url:
                continue

            try:
                with YoutubeDL({
                    "quiet": True,
                    "no_warnings": True,
                }) as ydl:
                    response = ydl.urlopen(url)
                    raw = response.read().decode("utf-8", errors="replace")

                text = parse_caption_text(raw, track.get("ext", ""))
                text = clean_transcript(text)

                if text:
                    print(f"    Transcrição: {kind}, idioma {lang}")
                    return text

            except Exception as exc:
                print(f"    Aviso: falha ao ler legenda {lang}: {exc}")

    return None


def parse_caption_text(raw: str, ext: str) -> str:
    """Converte VTT/SRV3/TTML em texto simples."""
    ext = (ext or "").lower()

    if ext == "ttml":
        # Remoção simples das tags XML.
        text = re.sub(r"<br\s*/?>", "\n", raw, flags=re.I)
        text = re.sub(r"<[^>]+>", "", text)
        return text

    if ext == "srv3":
        # Alguns SRV3 contêm tags <text>.
        parts = re.findall(r"<text[^>]*>(.*?)</text>", raw, flags=re.I | re.S)
        if parts:
            text = "\n".join(parts)
            text = re.sub(r"<[^>]+>", "", text)
            return text

    # VTT e formatos semelhantes.
    lines = []
    for line in raw.splitlines():
        line = line.strip()

        if not line:
            continue

        if line.upper() == "WEBVTT":
            continue

        if "-->" in line:
            continue

        if re.fullmatch(r"\d+", line):
            continue

        # Metadados comuns de VTT.
        if line.startswith(("NOTE", "STYLE", "REGION")):
            continue

        line = re.sub(r"<[^>]+>", "", line)
        line = re.sub(r"&nbsp;", " ", line, flags=re.I)
        line = re.sub(r"&amp;", "&", line)
        line = re.sub(r"\s+", " ", line).strip()

        if line:
            lines.append(line)

    # Evita repetir linhas idênticas consecutivas, algo comum em captions.
    result = []
    for line in lines:
        if not result or line != result[-1]:
            result.append(line)

    return "\n".join(result)


def build_markdown(info: dict, transcript: str | None) -> str:
    title = info.get("title") or "Sem título"
    upload_date = format_date(info.get("upload_date"))
    description = info.get("description") or ""
    video_id = info.get("id") or ""
    video_url = f"https://www.youtube.com/watch?v={video_id}"

    transcript_text = transcript or "TRANSCRIÇÃO NÃO DISPONÍVEL"

    # A descrição é preservada integralmente.
    # Isso inclui URLs/fontes citadas no vídeo.
    return f"""---
title: {yaml_quote(title)}
date: {yaml_quote(upload_date)}
sources: |
{indent_block(description, 2)}
youtube: {yaml_quote(video_url)}
---

{transcript_text}

## Vídeo

<iframe
  width="560"
  height="315"
  src="https://www.youtube.com/embed/{video_id}"
  title={yaml_quote(title)}
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
  allowfullscreen>
</iframe>
"""


def indent_block(text: str, spaces: int) -> str:
    prefix = " " * spaces
    if not text:
        return prefix
    return "\n".join(prefix + line for line in text.replace("\r\n", "\n").replace("\r", "\n").split("\n"))


def unique_output_path(title: str, video_id: str) -> Path:
    slug = slugify(title)

    # O ID no final evita colisão caso existam títulos iguais.
    filename = f"{slug}-{video_id}.md"
    return OUTPUT_DIR / filename


def process_channel(channel_url: str) -> None:
    OUTPUT_DIR.mkdir(parents=True, exist_ok=True)

    print("=" * 70)
    print("YouTube -> Markdown")
    print("=" * 70)
    print(f"Canal:   {channel_url}")
    print(f"Saída:   {OUTPUT_DIR.resolve()}")
    print()

    # Flat extraction primeiro: descobre os vídeos sem processar cada página.
    discovery_opts = {
        "quiet": False,
        "ignoreerrors": True,
        "extract_flat": True,
        "skip_download": True,
        **cookie_opts(),
    }

    with YoutubeDL(discovery_opts) as ydl:
        print("Descobrindo vídeos do canal...")
        result = ydl.extract_info(channel_url, download=False)

    if not result:
        print("Não foi possível obter o canal.")
        return

    entries = result.get("entries") or []
    entries = [e for e in entries if e and e.get("id")]

    print(f"Vídeos encontrados: {len(entries)}")
    print()

    # Uma execução dedicada para metadados/transcrições.
    info_opts = {
        "quiet": True,
        "no_warnings": True,
        "skip_download": True,
        "ignore_no_formats_error": True,
        "sleep_interval_requests": 1,
        **cookie_opts(),
    }

    success = 0
    unavailable = 0
    errors = 0
    skipped = 0

    with YoutubeDL(info_opts) as ydl:
        for index, entry in enumerate(entries, start=1):
            video_id = entry.get("id")
            video_url = f"https://www.youtube.com/watch?v={video_id}"

            print(f"[{index}/{len(entries)}] {video_id}")

            try:
                # Primeiro consulta o caminho esperado.
                # Se já existir, não refaz o trabalho.
                # Como o título pode mudar, o arquivo final é localizado
                # também pelo ID.
                existing = list(OUTPUT_DIR.glob(f"*-{video_id}.md"))
                if existing:
                    print(f"    Já existe: {existing[0].name}")
                    skipped += 1
                    continue

                info = ydl.extract_info(video_url, download=False)

                if not info:
                    print("    ERRO: não foi possível obter informações.")
                    errors += 1
                    continue

                title = info.get("title") or video_id
                print(f"    Título: {title}")

                transcript = extract_transcript(info)

                if transcript:
                    success += 1
                else:
                    print("    Transcrição não disponível.")
                    unavailable += 1

                output_path = unique_output_path(title, video_id)

                markdown = build_markdown(info, transcript)
                output_path.write_text(markdown, encoding="utf-8")

                print(f"    OK: {output_path.name}")

            except KeyboardInterrupt:
                print("\nInterrompido pelo usuário.")
                print("Execute novamente para continuar; arquivos já criados serão pulados.")
                break

            except Exception as exc:
                errors += 1
                print(f"    ERRO: {type(exc).__name__}: {exc}")

            # Pequena pausa para não disparar requisições em sequência.
            time.sleep(0.5)

    print()
    print("=" * 70)
    print("FINALIZADO")
    print("=" * 70)
    print(f"Com transcrição:       {success}")
    print(f"Sem transcrição:       {unavailable}")
    print(f"Erros:                 {errors}")
    print(f"Já existentes/pulados: {skipped}")
    print(f"Arquivos em:           {OUTPUT_DIR.resolve()}")
    print("=" * 70)


if __name__ == "__main__":
    channel = sys.argv[1] if len(sys.argv) > 1 else DEFAULT_CHANNEL_URL
    process_channel(channel)
```
