---
title: "Convertendo vídeos em lote com Python: GPU, áudio e legendas automatizados"
date: 2026-08-10T05:10:00.000-03:00
draft: false
image: "/images/uploads/convert-videos.png"
description: Script para converter lotes de vídeos (MKV/MP4), com aceleração por
  GPU NVIDIA, normalização de áudio e legendas automáticas, usando Python e
  ffmpeg
tags:
  - 📼 ffmpeg
  - 📺 vídeo
  - 📻 áudio
  - 🐍 Python
  - ""
categories: []
cover:
  image: /images/uploads/convert-videos.png
  alt: Script para converter lotes de vídeos (MKV/MP4), com aceleração por GPU
    NVIDIA, normalização de áudio e legendas automáticas, usando Python e ffmpeg
ShowToc: true
comments: true
---
Se você converte lote de vídeos antigos (séries, gravações, arquivos pessoais) e cansou de configurar o ffmpeg na mão toda vez — escolher codec, resolução, cuidar de faixas de áudio e legenda, lembrar dos parâmetros de GPU — dá pra automatizar isso tudo num script interativo que faz as perguntas certas e monta o comando do ffmpeg sozinho. 🎬⚙️

Neste tutorial eu mostro o script Python que uso pra isso, como ele decide entre GPU e CPU, como monta a cadeia de filtros de áudio, e como ele processa vários vídeos em paralelo com um painel ao vivo no terminal.

![](/images/uploads/captura-de-tela-2026-08-10-053146.png "Script Conversor de Vídeos em Lote")

## 🧠 O que o script faz

* 📂 Recebe uma ou mais pastas (separadas por vírgula) e varre recursivamente atrás de vídeos.
* 📦 Deixa escolher o container de saída (MKV ou MP4).
* 🎯 Deixa escolher a resolução alvo (Original, 480p, 720p, 1080p, 2160p) — só redimensiona se for necessário.
* 🔊 Três modos de áudio: normalização para -14 LUFS, manter original (copy), ou um "Voice Enhancer" experimental para áudio abafado de fitas antigas.
* ⚡ Detecta GPU NVIDIA (NVENC) automaticamente, com teste real de encode — não só verifica se está compilado no ffmpeg — e cai pra CPU (libx264) se algo falhar.
* 📝 Preserva nomes e idiomas das faixas de áudio/legenda, e embute `.srt` externo automaticamente se encontrar um com o mesmo nome do vídeo.
* 🧵 Roda em paralelo (1 a 4 vídeos simultâneos) com um painel fixo no terminal, uma linha por worker.
* 🗂️ Salva tudo numa pasta `_NEW/` espelhando a estrutura de subpastas original, sem tocar nos arquivos fonte.

⚠️ Importante: o script **não apaga nem sobrescreve os originais** — a saída sempre vai para `_NEW/`, ao lado dos arquivos de entrada.

## 📦 Dependências

Você precisa de Python 3.10+ e do `ffmpeg` + `ffprobe` no PATH.

```bash
# Windows: baixe em ffmpeg.org e adicione ao PATH
# Linux: sudo apt install ffmpeg
```

Não precisa de nenhuma biblioteca Python externa — só a standard library (`subprocess`, `pathlib`, `threading`, `queue`, `json`).

## ⚙️ Como o script está organizado

Em linhas gerais, o fluxo é:

1. 🔎 **Detecção de GPU**: roda um encode de teste curtinho com `h264_nvenc` pra confirmar que a GPU funciona de verdade nesta máquina, não só que o ffmpeg tem o encoder compilado.
2. 📊 **Probe único por arquivo**: uma chamada ao `ffprobe` traz duração, resolução, codec de vídeo e todas as faixas de áudio/legenda de uma vez — evita abrir vários processos por arquivo em lotes grandes.
3. 🧭 **Menus interativos**: pergunta formato, resolução, modo de áudio e grau de paralelismo antes de começar.
4. 🛠️ **Montagem do comando ffmpeg**: monta os parâmetros de vídeo, áudio e legenda dinamicamente com base nas respostas e no que o probe encontrou.
5. 🧵 **Execução**: sequencial (relatório detalhado por arquivo) ou paralela (painel compacto com N workers consumindo uma fila compartilhada).

Um trecho central é a detecção real de GPU, que não confia só na listagem de encoders:

```python
def detect_gpu():
    global GPU_ENCODER, GPU_NAME, GPU_FAIL_REASON

    r = subprocess.run(["ffmpeg", "-hide_banner", "-encoders"], ...)
    if "h264_nvenc" not in r.stdout:
        GPU_FAIL_REASON = "h264_nvenc nao compilado neste ffmpeg"
        return

    test = subprocess.run(
        ["ffmpeg", "-y", "-hide_banner", "-loglevel", "warning",
         "-f", "lavfi", "-i", "color=c=black:s=320x240:d=1",
         "-c:v", "h264_nvenc", "-f", "null", "-"], ...
    )
    if test.returncode != 0:
        GPU_FAIL_REASON = ...  # extrai a linha de erro real do stderr
        return

    GPU_ENCODER = "h264_nvenc"
```

E o modo de áudio "normalize" monta um `filter_complex` de `loudnorm` por faixa, preservando o número de faixas originais:

```python
for t in tracks:
    ai  = t["audio_index"]
    lbl = f"a{ai}norm"
    fc_parts.append(
        f"[0:a:{ai}]loudnorm=I={TARGET_LUFS}:TP={TARGET_TP}"
        f":LRA={TARGET_LRA}[{lbl}]"
    )
```

## 🩺 Voice Enhancer \[BETA]

Esse modo é pensado pra vídeos antigos (VHS/Hi8) com áudio abafado. Ele encadeia vários filtros de áudio antes da normalização: corte de graves (`highpass`), redução de ruído por FFT (`afftdn`), realce de diálogo (`dialoguenhance`), um leve boost na faixa de presença vocal (2-4kHz) e compressão, só então normalizando para -14 LUFS. É experimental — o resultado varia de vídeo pra vídeo.

## 🧵 Paralelismo

Ao escolher rodar 2, 3 ou 4 vídeos ao mesmo tempo, o script usa uma fila (`queue.Queue`) e N threads fixas, cada uma dona de uma linha do painel. O painel redesenha a tela a cada atualização, movendo o cursor pra cima e limpando até o fim — evita sobra de texto quando uma linha nova é mais curta que a anterior.

## ▶️ Rodando o script

```bash
python conversor_video.py
```

O script vai pedir a(s) pasta(s), depois formato, resolução, modo de áudio e grau de paralelismo, mostra um resumo e pede confirmação antes de começar.

Os arquivos convertidos vão para `_NEW/` dentro de cada pasta informada, espelhando as subpastas originais.

## 🚑 Problemas comuns e como resolver

**❌ `'ffmpeg' nao encontrado no PATH`**
Instale o ffmpeg e garanta que `ffmpeg` e `ffprobe` estejam acessíveis no terminal.

**❌ Conversão cai pra CPU mesmo tendo GPU NVIDIA**
O teste de encode real (`detect_gpu`) pode estar falhando por driver desatualizado ou falta do `nvidia-smi`. O motivo exato aparece no banner inicial, na linha "Motivo: ...".

**❌ Legenda em imagem (PGS) sumiu ao converter pra MP4**
Esperado — o container MP4 só aceita legenda de texto (`mov_text`). Pra preservar legendas bitmap, use MKV.

**❌ GPU falha no meio da conversão**
O script já tenta automaticamente de novo por CPU (`force_cpu=True`) antes de reportar erro — então normalmente nem é preciso intervir.

## 🔧 Adaptando

Pra mudar o padrão de qualidade da GPU, ajuste `-cq` em `build_cmd` (menor = mais qualidade, mais lento). Pra mudar o alvo de loudness, edite `TARGET_LUFS`/`TARGET_TP`/`TARGET_LRA` no topo do arquivo.

## 💻 Código completo

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
conversor_video.py  —  gsFALCON Video Converter
Converte videos em lote: escolha de formato (MKV/MP4), resolucao alvo e
modo de audio (normalizado -14 LUFS ou mantido original). Preserva nomes
das faixas de audio/legenda, embute legendas externas .srt automaticamente.
Acelera por GPU (NVIDIA NVENC) quando disponivel, com fallback pra CPU.
Multiplas pastas separadas por virgula. Saida em _NEW/ com estrutura espelhada.
Requer: ffmpeg + ffprobe no PATH.  Python 3.10+.
"""

import os, sys, re, json, shutil, subprocess, time, threading, queue
from pathlib import Path

# ── Auto-terminal: se executado fora de um terminal, reabre dentro de um ──────
def _in_terminal():
    try:
        return os.isatty(sys.stdin.fileno())
    except Exception:
        return True

if not _in_terminal():
    script = os.path.abspath(sys.argv[0])
    terminals = [
        ["gnome-terminal", "--", "bash", "-c", f'python3 "{script}"; exec bash'],
        ["xterm", "-e", f'bash -c ''python3 "{script}"; exec bash'''],
        ["konsole", "--noclose", "-e", f'python3 "{script}"'],
        ["xfce4-terminal", "--hold", "-e", f'python3 "{script}"'],
        ["x-terminal-emulator", "-e", f'bash -c ''python3 "{script}"; exec bash'''],
    ]
    for term_cmd in terminals:
        try:
            subprocess.Popen(term_cmd)
            sys.exit()
        except FileNotFoundError:
            continue

# Forçar UTF-8 no stdout/stderr do Windows
if sys.platform == "win32":
    import io
    sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding="utf-8", errors="replace")
    sys.stderr = io.TextIOWrapper(sys.stderr.buffer, encoding="utf-8", errors="replace")

# ── Cores ANSI ────────────────────────────────────────────────────────────────
R   = "\033[0m"
B   = "\033[1m"
DIM = "\033[2m"
CY  = "\033[96m"
GR  = "\033[92m"
YE  = "\033[93m"
RE  = "\033[91m"
MA  = "\033[95m"
BL  = "\033[94m"
WH  = "\033[97m"
OR  = "\033[38;5;208m"

# ── Constantes ────────────────────────────────────────────────────────────────
VIDEO_EXT = {
    ".mp4", ".ts", ".mkv", ".avi", ".mov", ".wmv",
    ".flv", ".webm", ".m4v", ".mpg", ".mpeg", ".3gp",
    ".vob", ".m2ts", ".rmvb", ".divx", ".f4v",
}

TARGET_LUFS = -14.0
TARGET_TP   = -2.0
TARGET_LRA  = 11.0
OUT_FOLDER  = "_NEW"

# Formato de saida do container
FORMAT_MENU = [
    ("1", "MKV", ".mkv"),
    ("2", "MP4", ".mp4"),
]

# Resolucao alvo — "Original" (tw=th=None) so remuxa, nao redimensiona
RES_MENU = [
    ("1", "Original", None, None),
    ("2", "480p",     854,  480),
    ("3", "720p",     1280, 720),
    ("4", "1080p",    1920, 1080),
    ("5", "2160p",    3840, 2160),
]

# Modo de audio — "normalize" primeiro pois e o comportamento padrao/mais usado
AUDIO_MENU = [
    ("1", "Padrao universal (-14 LUFS)",  "normalize"),
    ("2", "Manter audio original",        "copy"),
    ("3", "Voice Enhancer [BETA]",        "enhance"),
]

# Cadeia de filtros do Voice Enhancer: remove ruido de fita/rumble, reduz
# ruido de fundo (FFT denoise), realca a faixa de dialogo (dialoguenhance),
# reforca a faixa de presenca vocal (2-4kHz) e equaliza o volume da fala
# antes de normalizar. Pensado para VHS/Hi8 antigos com audio abafado.
VOICE_ENHANCE_CHAIN = (
    "highpass=f=100,"
    "afftdn=nr=12:nf=-25,"
    "dialoguenhance=original=1:enhance=1.5:voice=2,"
    "equalizer=f=3000:width_type=o:width=2:g=4,"
    "acompressor=threshold=-18dB:ratio=3:attack=10:release=100:makeup=1.5,"
    "loudnorm=I={lufs}:TP={tp}:LRA={lra}"
)

SPEED_COLOR = {"RAPIDO": GR, "MEDIO": YE, "DEMORADO": RE}
SPEED_EMOJI = {"RAPIDO": "🟢", "MEDIO": "🟡", "DEMORADO": "🔴"}

# Codecs de legenda de texto (cabem em MP4 via mov_text) vs imagem/bitmap
# (PGS, VobSub etc — so cabem em MKV, sao ignorados se o alvo for MP4)
TEXT_SUB_CODECS = {"subrip", "srt", "ass", "ssa", "mov_text", "webvtt", "text"}

# ─────────────────────────────────────────────────────────────────────────────
#  UTILIDADES
# ─────────────────────────────────────────────────────────────────────────────

def cls():
    os.system("cls" if os.name == "nt" else "clear")

def fmt_size(path_or_bytes):
    try:
        s = path_or_bytes.stat().st_size if isinstance(path_or_bytes, Path) else int(path_or_bytes)
        for u in ("B", "KB", "MB", "GB"):
            if s < 1024: return f"{s:.1f} {u}"
            s /= 1024
        return f"{s:.1f} GB"
    except:
        return "?"

def fmt_time(sec):
    m, s = divmod(int(sec), 60)
    h, m = divmod(m, 60)
    if h:  return f"{h}h {m:02d}m {s:02d}s"
    if m:  return f"{m}m {s:02d}s"
    return f"{s}s"

def progress_bar(frac, width=36):
    filled = int(width * min(max(frac, 0), 1))
    bar    = "█" * filled + "░" * (width - filled)
    pct    = frac * 100
    return f"{CY}[{bar}]{R} {GR}{pct:5.1f}%{R}"

def ac3_bitrate_for_channels(channels):
    """Bitrate AC3 proporcional ao numero de canais — 192k e suficiente para
    estereo, mas deixa faixas 5.1 com som abafado/comprimido demais."""
    try:
        ch = int(channels)
    except (TypeError, ValueError):
        ch = 2
    return "448k" if ch > 2 else "192k"

# ─────────────────────────────────────────────────────────────────────────────
#  GPU / NVENC
# ─────────────────────────────────────────────────────────────────────────────

GPU_ENCODER     = None   # "h264_nvenc" se disponivel, senao None (fallback libx264)
GPU_NAME        = None
GPU_FAIL_REASON = None   # motivo do fallback, exibido no banner para diagnostico

def detect_gpu():
    """Testa se h264_nvenc realmente funciona nesta maquina (nao so se esta
    compilado no ffmpeg). Faz um encode curto para confirmar driver/GPU."""
    global GPU_ENCODER, GPU_NAME, GPU_FAIL_REASON

    try:
        r = subprocess.run(
            ["ffmpeg", "-hide_banner", "-encoders"],
            capture_output=True, text=True, encoding="utf-8", errors="replace",
            timeout=10
        )
        if "h264_nvenc" not in r.stdout:
            GPU_FAIL_REASON = "h264_nvenc nao compilado neste ffmpeg"
            return
    except Exception as e:
        GPU_FAIL_REASON = f"falha ao listar encoders ({e})"
        return

    try:
        # 1 segundo (~25 frames) sem limite de -frames:v — o NVENC usa buffer
        # interno (lookahead/B-frames) e um clipe curto demais nao deixa
        # o encoder "soltar" nenhum pacote, dando falso-negativo no teste.
        # loglevel "warning" (nao "error") pois no Windows a causa real do
        # NVENC costuma vir como aviso, nao como erro.
        # 320x240 — NVENC exige uma dimensao minima de frame; valores menores
        # dao falso-negativo mesmo com a GPU 100% funcional.
        test = subprocess.run(
            ["ffmpeg", "-y", "-hide_banner", "-loglevel", "warning",
             "-f", "lavfi", "-i", "color=c=black:s=320x240:d=1",
             "-c:v", "h264_nvenc", "-f", "null", "-"],
            capture_output=True, text=True, encoding="utf-8", errors="replace",
            timeout=15
        )
        if test.returncode != 0:
            lines = [l.strip() for l in test.stderr.strip().splitlines() if l.strip()]
            specific = [l for l in lines if "nvenc" in l.lower() or "cuda" in l.lower()]
            GPU_FAIL_REASON = specific[0] if specific else (lines[0] if lines else f"codigo {test.returncode}")
            return
    except Exception as e:
        GPU_FAIL_REASON = f"falha no teste de encode ({e})"
        return

    GPU_ENCODER = "h264_nvenc"
    GPU_FAIL_REASON = None

    try:
        r = subprocess.run(
            ["nvidia-smi", "--query-gpu=name", "--format=csv,noheader"],
            capture_output=True, text=True, encoding="utf-8", errors="replace",
            timeout=5
        )
        name = r.stdout.strip().splitlines()[0].strip()
        if name:
            GPU_NAME = name
    except Exception:
        GPU_NAME = "NVIDIA (NVENC)"

# ─────────────────────────────────────────────────────────────────────────────
#  FFPROBE HELPERS
# ─────────────────────────────────────────────────────────────────────────────

def _run(cmd, timeout=30):
    return subprocess.run(cmd, capture_output=True, text=True, encoding="utf-8", errors="replace", timeout=timeout)

def probe_media(src):
    """Uma unica chamada ao ffprobe traz duracao, resolucao/codec do video,
    faixas de audio e legendas embutidas de uma vez. Antes eram ate 5
    processos ffprobe separados por arquivo — em lotes de 100+ episodios
    isso soma segundos so de overhead de abrir processo, sem contar o
    tempo de conversao em si."""
    empty = {
        "duration": 0.0, "width": 0, "height": 0, "video_codec": "?",
        "audio_tracks": [], "sub_tracks": [],
    }
    try:
        r = _run(["ffprobe", "-v", "error", "-show_format", "-show_streams",
                  "-of", "json", str(src)])
        data = json.loads(r.stdout)
    except Exception:
        return empty

    try:
        duration = float(data.get("format", {}).get("duration", 0.0))
    except (TypeError, ValueError):
        duration = 0.0

    width = height = 0
    video_codec  = "?"
    audio_tracks = []
    sub_tracks   = []
    a_idx = s_idx = 0

    for s in data.get("streams", []):
        ctype = s.get("codec_type")
        tags  = s.get("tags") or {}

        if ctype == "video" and width == 0:
            width       = s.get("width", 0) or 0
            height      = s.get("height", 0) or 0
            video_codec = (s.get("codec_name") or "?").upper()

        elif ctype == "audio":
            audio_tracks.append({
                "stream_index": s.get("index", a_idx),
                "audio_index":  a_idx,
                "codec":        (s.get("codec_name") or "?").upper(),
                "channels":     s.get("channels", 2),
                "title":        tags.get("title") or tags.get("TITLE") or "",
                "language":     tags.get("language") or tags.get("LANGUAGE") or "",
                "disposition":  s.get("disposition", {}),
            })
            a_idx += 1

        elif ctype == "subtitle":
            codec = (s.get("codec_name") or "").lower()
            sub_tracks.append({
                "sub_index": s_idx,
                "codec":     codec,
                "title":     tags.get("title") or tags.get("TITLE") or "",
                "language":  tags.get("language") or tags.get("LANGUAGE") or "",
                "is_text":   codec in TEXT_SUB_CODECS,
            })
            s_idx += 1

    return {
        "duration": duration, "width": width, "height": height,
        "video_codec": video_codec, "audio_tracks": audio_tracks,
        "sub_tracks": sub_tracks,
    }

def estimate_speed(duration, size_bytes, convert_video):
    size_mb = size_bytes / (1024 * 1024)
    # Sem recodificar video (copy) sempre e rapido, independente do tamanho
    if not convert_video:
        return "RAPIDO"
    # Com GPU, a recodificacao e varias vezes mais rapida que so CPU
    if GPU_ENCODER:
        if size_mb < 1200 and duration < 5400:
            return "RAPIDO"
        elif size_mb > 6000 or duration > 16200:
            return "DEMORADO"
        return "MEDIO"
    else:
        if size_mb < 400 and duration < 1800:
            return "RAPIDO"
        elif size_mb > 2000 or duration > 5400:
            return "DEMORADO"
        return "MEDIO"

# ─────────────────────────────────────────────────────────────────────────────
#  SCAN / PATHS
# ─────────────────────────────────────────────────────────────────────────────

def collect_videos(folder):
    found = []
    for root, dirs, files in os.walk(folder):
        dirs[:] = [d for d in dirs if d != OUT_FOLDER]
        for f in files:
            p = Path(root) / f
            if p.suffix.lower() in VIDEO_EXT:
                found.append(p)
    return found

def ask_folders():
    print(f"\n{B}📂 Digite a(s) pasta(s) com os videos:{R}")
    print(f"{DIM}   Separe multiplas pastas com virgula")
    print(f"   Ex:  D:\\Series\\Chaves   ou   D:\\Series, E:\\Filmes{R}\n")
    raw = input(f"  {CY}Pasta(s):{R} ").strip()
    folders = [Path(p.strip()) for p in raw.split(",") if p.strip()]
    valid = []
    for f in folders:
        if f.exists():
            valid.append(f)
        else:
            print(f"  {YE}Aviso: pasta nao encontrada, ignorada: {f}{R}")
    if not valid:
        print(f"\n{RE}Nenhuma pasta valida informada.{R}")
        input(f"\n{DIM}Pressione ENTER para fechar...{R}")
        sys.exit(1)
    return valid

def build_dst(src, src_root, ext):
    rel     = src.relative_to(src_root)
    out_dir = src_root / OUT_FOLDER / rel.parent
    out_dir.mkdir(parents=True, exist_ok=True)
    return out_dir / (src.stem + ext)

# ─────────────────────────────────────────────────────────────────────────────
#  MENUS
# ─────────────────────────────────────────────────────────────────────────────

def _ask_choice(menu):
    choices = [m[0] for m in menu]
    while True:
        choice = input(f"\n  {B}Opcao [{'/'.join(choices)}]:{R} ").strip()
        for m in menu:
            if m[0] == choice:
                return m
        print(f"  {RE}Opcao invalida.{R}")

def ask_format():
    print(f"\n{B}  📦 Formato de saida:{R}\n")
    for mid, label, ext in FORMAT_MENU:
        print(f"  {CY}{B}[{mid}]{R}  {WH}{B}{label:<6}{R}")
    print(f"\n  {DIM}MKV: aceita qualquer legenda e multiplas faixas sem restricao{R}")
    print(f"  {DIM}MP4: mais compativel com editores; legendas em imagem (PGS) nao entram{R}")
    return _ask_choice(FORMAT_MENU)

def ask_resolution():
    print(f"\n{B}  🎯 Resolucao alvo:{R}\n")
    for mid, label, tw, th in RES_MENU:
        dims = f"({tw}x{th})" if tw else "(sem redimensionar)"
        print(f"  {CY}{B}[{mid}]{R}  {WH}{B}{label:<10}{R}  {DIM}{dims}{R}")
    return _ask_choice(RES_MENU)

def ask_audio():
    print(f"\n{B}  🔊 Audio:{R}\n")
    descs = {
        "normalize": "normaliza todas as trilhas para -14 LUFS (padrao streaming/YouTube)",
        "copy":      "mantem o audio 100% original, sem recodificar (qualidade maxima)",
        "enhance":   "reduz ruido e realca dialogo — para videos antigos com audio abafado",
    }
    for mid, label, mode in AUDIO_MENU:
        color = OR if mode == "enhance" else WH
        print(f"  {CY}{B}[{mid}]{R}  {color}{B}{label}{R}")
        print(f"       {DIM}{descs[mode]}{R}")
    print(f"\n  {DIM}Voice Enhancer e experimental — resultado pode variar de video para video{R}")
    return _ask_choice(AUDIO_MENU)

PARALLEL_MENU = [
    ("1", 1, "Sequencial — relatorio detalhado por arquivo"),
    ("2", 2, "2 ao mesmo tempo — resumo compacto por arquivo"),
    ("3", 3, "3 ao mesmo tempo — resumo compacto por arquivo"),
    ("4", 4, "4 ao mesmo tempo — resumo compacto por arquivo"),
]

def ask_parallel():
    print(f"\n{B}  ⚙️  Converter quantos videos ao mesmo tempo?{R}\n")
    for mid, n, desc in PARALLEL_MENU:
        print(f"  {CY}{B}[{mid}]{R}  {WH}{B}{n}{R}  {DIM}— {desc}{R}")
    if GPU_ENCODER:
        print(f"\n  {DIM}Com GPU ativa, rodar 2-4 de uma vez costuma aproveitar melhor o hardware{R}")
    else:
        print(f"\n  {DIM}Sem GPU, mais de 1 ao mesmo tempo depende de quantos nucleos sua CPU tem{R}")
    choices = [m[0] for m in PARALLEL_MENU]
    while True:
        choice = input(f"\n  {B}Opcao [{'/'.join(choices)}]:{R} ").strip()
        for m in PARALLEL_MENU:
            if m[0] == choice:
                return m[1]
        print(f"  {RE}Opcao invalida.{R}")

# ─────────────────────────────────────────────────────────────────────────────
#  LEGENDAS EXTERNAS (.SRT)
# ─────────────────────────────────────────────────────────────────────────────

LANG_KEYWORDS = {
    "portuguese": "Portuguese", "portugues": "Portuguese",
    "por": "Portuguese", ".pt.": "Portuguese", "_pt_": "Portuguese",
    "-pt-": "Portuguese", "pt-br": "Portuguese", "ptbr": "Portuguese",
    "english": "English", "eng": "English", ".en.": "English",
    "_en_": "English", "-en-": "English",
    "spanish": "Spanish", "espanol": "Spanish", "spa": "Spanish",
    "french": "French", "francais": "French", "fre": "French",
    "german": "German", "deutsch": "German", "ger": "German",
    "italian": "Italian", "italiano": "Italian", "ita": "Italian",
    "japanese": "Japanese", "jpn": "Japanese",
}

def detect_srt_language(srt_path):
    """Detecta idioma pelo nome do arquivo. Fallback: Portuguese."""
    name = srt_path.stem.lower()
    for keyword, lang in LANG_KEYWORDS.items():
        if keyword in name:
            return lang
    return "Portuguese"

def find_srt(video_path):
    """Procura .srt com mesmo stem do video na mesma pasta."""
    srt = video_path.with_suffix(".srt")
    if srt.exists():
        return srt
    for f in video_path.parent.iterdir():
        if f.suffix.lower() == ".srt" and f.stem.lower() == video_path.stem.lower():
            return f
    return None

# ─────────────────────────────────────────────────────────────────────────────
#  COMANDO FFMPEG
# ─────────────────────────────────────────────────────────────────────────────

def build_cmd(src, dst, th, tracks, sub_tracks, convert_video, container, audio_mode,
              srt_path=None, force_cpu=False):
    # -fflags +genpts corrige timestamps ausentes/corrompidos em AVIs antigos
    # sem isso o MKV rejeita pacotes com "unknown timestamp" ao usar -c:v copy
    cmd = ["ffmpeg", "-y", "-fflags", "+genpts", "-i", str(src)]

    srt_input_idx = None
    if srt_path:
        cmd += ["-i", str(srt_path)]
        srt_input_idx = 1

    use_gpu = (GPU_ENCODER is not None) and not force_cpu

    # ── VIDEO ────────────────────────────────────────────────────────────────
    if convert_video:
        cmd += ["-map", "0:v:0", "-vf", f"scale=-2:{th}"]
        if use_gpu:
            # NVENC: qualidade equivalente ao libx264 crf 20, muito mais rapido
            cmd += ["-c:v", "h264_nvenc", "-preset", "p6", "-tune", "hq",
                    "-rc", "vbr", "-cq", "19", "-b:v", "0",
                    "-multipass", "fullres",
                    "-spatial-aq", "1", "-temporal-aq", "1", "-aq-strength", "8",
                    "-bf", "3"]
        else:
            cmd += ["-c:v", "libx264", "-preset", "medium", "-crf", "20"]
    else:
        cmd += ["-map", "0:v:0", "-c:v", "copy"]

    # ── AUDIO ────────────────────────────────────────────────────────────────
    if tracks:
        if audio_mode == "copy":
            # Copia bit-a-bit — zero recodificacao. Titulo/idioma de cada
            # faixa sao preservados automaticamente pelo ffmpeg nesse modo.
            for t in tracks:
                cmd += ["-map", f"0:a:{t['audio_index']}"]
            cmd += ["-c:a", "copy"]

        elif audio_mode == "enhance":
            # Voice Enhancer [BETA]: reduz ruido de fundo e realca a faixa
            # de dialogo antes de normalizar — pensado para VHS/Hi8 antigos.
            fc_parts   = []
            out_labels = []
            for t in tracks:
                ai  = t["audio_index"]
                lbl = f"a{ai}enh"
                chain = VOICE_ENHANCE_CHAIN.format(lufs=TARGET_LUFS, tp=TARGET_TP, lra=TARGET_LRA)
                fc_parts.append(f"[0:a:{ai}]{chain}[{lbl}]")
                out_labels.append(lbl)
            cmd += ["-filter_complex", ";".join(fc_parts)]
            for lbl in out_labels:
                cmd += ["-map", f"[{lbl}]"]
            cmd += ["-c:a", "ac3"]
            for i, t in enumerate(tracks):
                cmd += [f"-b:a:{i}", ac3_bitrate_for_channels(t["channels"])]
                # dialoguenhance pode alterar o layout de canal (ex: estereo
                # vira "3.0") — forca de volta a contagem de canal original
                cmd += [f"-ac:a:{i}", str(t["channels"])]

        else:  # "normalize"
            fc_parts   = []
            out_labels = []
            for t in tracks:
                ai  = t["audio_index"]
                lbl = f"a{ai}norm"
                fc_parts.append(
                    f"[0:a:{ai}]loudnorm=I={TARGET_LUFS}:TP={TARGET_TP}"
                    f":LRA={TARGET_LRA}[{lbl}]"
                )
                out_labels.append(lbl)
            cmd += ["-filter_complex", ";".join(fc_parts)]
            for lbl in out_labels:
                cmd += ["-map", f"[{lbl}]"]
            cmd += ["-c:a", "ac3"]
            # Bitrate por faixa de acordo com o numero de canais — evita
            # audio 5.1 "abafado" por bitrate baixo demais (192k e so pra estereo)
            for i, t in enumerate(tracks):
                cmd += [f"-b:a:{i}", ac3_bitrate_for_channels(t["channels"])]
    else:
        cmd += ["-an"]

    # ── LEGENDAS ─────────────────────────────────────────────────────────────
    if container == "mkv":
        # MKV aceita qualquer codec — copia embutidas + externa de uma vez
        has_subs = bool(sub_tracks) or srt_input_idx is not None
        if has_subs:
            if sub_tracks:
                cmd += ["-map", "0:s"]
            if srt_input_idx is not None:
                cmd += ["-map", f"{srt_input_idx}:s:0"]
            cmd += ["-c:s", "copy"]
    else:
        # MP4 so aceita legenda de texto (mov_text) — bitmap/PGS fica de fora
        text_subs = [s for s in sub_tracks if s["is_text"]]
        has_subs  = bool(text_subs) or srt_input_idx is not None
        if has_subs:
            for s in text_subs:
                cmd += ["-map", f"0:s:{s['sub_index']}"]
            if srt_input_idx is not None:
                cmd += ["-map", f"{srt_input_idx}:s:0"]
            cmd += ["-c:s", "mov_text"]

    # ── METADADOS ────────────────────────────────────────────────────────────
    cmd += ["-map_metadata", "0"]

    if audio_mode in ("normalize", "enhance"):
        for i, t in enumerate(tracks):
            if t["title"]:
                cmd += [f"-metadata:s:a:{i}", f"title={t['title']}"]
            if t["language"]:
                cmd += [f"-metadata:s:a:{i}", f"language={t['language']}"]
    # audio_mode == "copy": titulo/idioma ja preservados automaticamente

    if srt_input_idx is not None:
        srt_lang     = detect_srt_language(srt_path)
        srt_lang_iso = srt_lang.lower()[:3]
        if container == "mkv":
            si = len(sub_tracks)
        else:
            si = len([s for s in sub_tracks if s["is_text"]])
        cmd += [f"-metadata:s:s:{si}", f"language={srt_lang_iso}",
                f"-metadata:s:s:{si}", f"title={srt_lang}"]

    if container == "mp4":
        cmd += ["-movflags", "+faststart"]

    cmd += ["-avoid_negative_ts", "make_zero"]

    cmd += [str(dst)]
    return cmd

# ─────────────────────────────────────────────────────────────────────────────
#  PROGRESSO
# ─────────────────────────────────────────────────────────────────────────────

def run_with_progress(cmd, duration):
    stderr_lines = []
    proc = subprocess.Popen(
        cmd,
        stderr=subprocess.PIPE, stdout=subprocess.DEVNULL,
        encoding="utf-8", errors="replace"
    )
    for line in proc.stderr:
        stderr_lines.append(line)
        if "time=" in line:
            try:
                time_str = line.split("time=")[1].split(" ")[0].strip()
                if time_str and time_str != "N/A":
                    h, m, s = time_str.split(":")
                    sec = int(h) * 3600 + int(m) * 60 + float(s)
                    if duration > 0:
                        frac = min(sec / duration, 1.0)
                        print(f"\r  {progress_bar(frac)}  ", end="", flush=True)
            except:
                pass
    proc.wait()
    print(f"\r  {progress_bar(1.0)}  ", flush=True)
    return proc.returncode, stderr_lines

# ─────────────────────────────────────────────────────────────────────────────
#  PAINEL MULTI-LINHA (modo paralelo)
# ─────────────────────────────────────────────────────────────────────────────

ACTIVE_PROCS      = set()
ACTIVE_PROCS_LOCK = threading.Lock()

class LivePanel:
    """Painel fixo na parte de baixo do terminal, uma linha por worker.
    Redesenha tudo a cada update (move cursor pra cima + limpa ate o fim da
    tela) — evita sobra de caracteres de textos mais curtos que o anterior.
    Linhas 'permanentes' (inicio/fim de arquivo) sao impressas ACIMA do
    painel e ficam no historico normal do terminal, como qualquer print()."""

    def __init__(self, n_slots):
        self.n      = n_slots
        self.lines  = [f"{DIM}  [ocioso]{R}" for _ in range(n_slots)]
        self.height = 0
        self.lock   = threading.Lock()

    def _redraw_locked(self):
        if self.height:
            sys.stdout.write(f"\033[{self.height}A")
            sys.stdout.write("\033[J")
        sys.stdout.write("\n".join(self.lines) + "\n")
        sys.stdout.flush()
        self.height = len(self.lines)

    def update(self, slot, text):
        with self.lock:
            self.lines[slot] = text
            self._redraw_locked()

    def print_permanent(self, text):
        with self.lock:
            if self.height:
                sys.stdout.write(f"\033[{self.height}A")
                sys.stdout.write("\033[J")
                self.height = 0
            print(text)
            self._redraw_locked()

    def close(self):
        with self.lock:
            if self.height:
                sys.stdout.write(f"\033[{self.height}A")
                sys.stdout.write("\033[J")
                sys.stdout.flush()
                self.height = 0

def run_with_progress_slot(cmd, duration, panel, slot, label):
    """Mesma logica do run_with_progress, mas atualiza uma linha do painel
    em vez de imprimir uma barra via carriage-return (que quebraria com
    varios processos escrevendo no terminal ao mesmo tempo)."""
    stderr_lines = []
    proc = subprocess.Popen(
        cmd,
        stderr=subprocess.PIPE, stdout=subprocess.DEVNULL,
        encoding="utf-8", errors="replace"
    )
    with ACTIVE_PROCS_LOCK:
        ACTIVE_PROCS.add(proc)
    try:
        for line in proc.stderr:
            stderr_lines.append(line)
            if "time=" in line:
                try:
                    time_str = line.split("time=")[1].split(" ")[0].strip()
                    if time_str and time_str != "N/A":
                        h, m, s = time_str.split(":")
                        sec = int(h) * 3600 + int(m) * 60 + float(s)
                        if duration > 0:
                            frac = min(sec / duration, 1.0)
                            panel.update(slot, f"  {label}  {progress_bar(frac, width=24)}")
                except:
                    pass
        proc.wait()
    finally:
        with ACTIVE_PROCS_LOCK:
            ACTIVE_PROCS.discard(proc)
    return proc.returncode, stderr_lines

# ─────────────────────────────────────────────────────────────────────────────
#  PROCESSAR UM ARQUIVO (modo paralelo — resumo compacto)
# ─────────────────────────────────────────────────────────────────────────────

def process_file_parallel(src, dst, th, idx, total, src_root, container, audio_mode, panel, slot):
    rel        = src.relative_to(src_root)
    size_src   = src.stat().st_size

    info       = probe_media(src)
    duration   = info["duration"]
    w0, h0     = info["width"], info["height"]
    tracks     = info["audio_tracks"]
    sub_tracks = info["sub_tracks"]

    convert_video = False if th is None else (h0 != th)
    gpu_tag = f" {OR}⚡{R}" if (GPU_ENCODER and convert_video) else ""
    label   = f"{MA}[{idx}/{total}]{R} {rel.name[:40]}{gpu_tag}"

    panel.update(slot, f"  {label}  {DIM}iniciando...{R}")

    srt_path = find_srt(src)

    t0  = time.time()
    cmd = build_cmd(src, dst, th, tracks, sub_tracks, convert_video, container, audio_mode, srt_path)
    ret, ffmpeg_log = run_with_progress_slot(cmd, duration, panel, slot, label)

    if ret != 0 and convert_video and GPU_ENCODER:
        panel.update(slot, f"  {label}  {YE}GPU falhou, tentando CPU...{R}")
        cmd = build_cmd(src, dst, th, tracks, sub_tracks, convert_video, container, audio_mode,
                         srt_path, force_cpu=True)
        ret, ffmpeg_log = run_with_progress_slot(cmd, duration, panel, slot, label)

    elapsed = time.time() - t0

    if ret != 0 or not dst.exists() or dst.stat().st_size < 512:
        error_lines = [l.strip() for l in ffmpeg_log if l.strip() and not l.startswith("frame=") and not l.startswith("fps=") and "out_time" not in l and "progress=" not in l]
        detail = " | ".join(error_lines[-2:]) if error_lines else f"codigo {ret}"
        raise RuntimeError(detail)

    info_dst = probe_media(dst)
    w1, h1   = info_dst["width"], info_dst["height"]
    size_dep = dst.stat().st_size
    diff_pct = ((size_dep - size_src) / size_src * 100) if size_src else 0
    sinal    = "+" if diff_pct > 0 else ""
    res_dep  = f"{w1}x{h1}" if w1 else "?"
    ac_label = {"copy": "copy", "enhance": "AC3-Enh"}.get(audio_mode, "AC3")

    summary = (f"{res_dep}  {ac_label}  {fmt_size(dst)} ({sinal}{diff_pct:.1f}%)  "
               f"{fmt_time(elapsed)}")
    return elapsed, summary

# ─────────────────────────────────────────────────────────────────────────────
#  PROCESSAR UM ARQUIVO
# ─────────────────────────────────────────────────────────────────────────────

def process_file(src, dst, th, idx, total, src_root, container, audio_mode):
    rel        = src.relative_to(src_root)
    size_src   = src.stat().st_size

    info       = probe_media(src)
    duration   = info["duration"]
    w0, h0     = info["width"], info["height"]
    vc0        = info["video_codec"]
    tracks     = info["audio_tracks"]
    sub_tracks = info["sub_tracks"]

    if th is None:
        convert_video = False
    else:
        convert_video = (h0 != th)

    speed    = estimate_speed(duration, size_src, convert_video)
    sc       = SPEED_COLOR[speed]
    sem      = SPEED_EMOJI[speed]
    gpu_tag  = f"  {OR}⚡ GPU{R}" if (GPU_ENCODER and convert_video) else ""

    print(f"\n{CY}{B}{'─'*64}{R}")
    print(f"  {MA}{B}[{idx}/{total}]{R}  {B}{rel}{R}  "
          f"{DIM}({fmt_size(src)} • {fmt_time(duration)}){R}  {sc}{sem} {speed}{R}{gpu_tag}")
    print(f"{CY}{B}{'─'*64}{R}")

    res_antes = f"{w0}x{h0}" if w0 else "?"
    print(f"  {DIM}Antes:  {res_antes}  {vc0}  {len(tracks)} trilha(s) audio  {fmt_size(src)}{R}")
    for t in tracks:
        nome = t["title"] or t["language"] or f"trilha {t['audio_index']}"
        print(f"  {DIM}  🔊 {t['codec']}  {t['channels']}ch  [{nome}]{R}")

    if th is None:
        print(f"  {YE}  Resolucao original — video copiado sem recodificar{R}")
    elif not convert_video:
        print(f"  {YE}  Video ja em {th}p — nao sera redimensionado{R}")

    if audio_mode == "copy":
        print(f"  {DIM}  Audio: mantido original, sem recodificar{R}")
    elif audio_mode == "enhance":
        print(f"  {OR}  Audio: Voice Enhancer [BETA] — realce de dialogo + -14 LUFS (AC3){R}")
    else:
        print(f"  {DIM}  Audio: normalizando para -14 LUFS (AC3){R}")

    if container == "mp4":
        skipped = [s for s in sub_tracks if not s["is_text"]]
        for s in skipped:
            nome = s["title"] or s["language"] or f"legenda {s['sub_index']}"
            print(f"  {YE}  ⚠ Legenda '{nome}' ({s['codec']}) nao suportada em MP4 — ignorada{R}")

    srt_path = find_srt(src)
    if srt_path:
        srt_lang = detect_srt_language(srt_path)
        print(f"  {DIM}  📄 Legenda externa: {srt_path.name}  [{srt_lang}]{R}")

    t0  = time.time()
    cmd = build_cmd(src, dst, th, tracks, sub_tracks, convert_video, container, audio_mode, srt_path)
    ret, ffmpeg_log = run_with_progress(cmd, duration)

    # Fallback automatico: se a GPU falhar, tenta de novo por CPU antes de desistir
    if ret != 0 and convert_video and GPU_ENCODER:
        print(f"\n  {YE}  GPU falhou neste arquivo, tentando por CPU...{R}")
        cmd = build_cmd(src, dst, th, tracks, sub_tracks, convert_video, container, audio_mode,
                         srt_path, force_cpu=True)
        ret, ffmpeg_log = run_with_progress(cmd, duration)

    elapsed = time.time() - t0

    if ret != 0 or not dst.exists() or dst.stat().st_size < 512:
        error_lines = [l.strip() for l in ffmpeg_log if l.strip() and not l.startswith("frame=") and not l.startswith("fps=") and "out_time" not in l and "progress=" not in l]
        detail = "\n    ".join(error_lines[-6:]) if error_lines else ""
        raise RuntimeError(f"FFmpeg erro (codigo {ret})\n    {detail}")

    info_dst = probe_media(dst)
    w1, h1   = info_dst["width"], info_dst["height"]
    vc1      = info_dst["video_codec"]
    res_dep  = f"{w1}x{h1}" if w1 else "?"
    size_dep = dst.stat().st_size
    diff_pct = ((size_dep - size_src) / size_src * 100) if size_src else 0
    sinal    = "+" if diff_pct > 0 else ""
    ac_label = {"copy": "copy", "enhance": "AC3 (Enhanced)"}.get(audio_mode, "AC3")

    print(f"  {GR}Depois: {res_dep}  {vc1}  {ac_label}  {fmt_size(dst)}  ({sinal}{diff_pct:.1f}%){R}")
    print(f"  ✔ {GR}OK  —  {fmt_time(elapsed)}{R}")

    return elapsed

# ─────────────────────────────────────────────────────────────────────────────
#  ETA GERAL
# ─────────────────────────────────────────────────────────────────────────────

def print_eta(done, total, elapsed_sum, ok_count):
    if ok_count == 0 or done >= total:
        return
    avg       = elapsed_sum / ok_count
    remaining = (total - done) * avg
    bar       = progress_bar(done / total, width=28)
    print(f"\n  {DIM}Geral: {bar}  restante ~{GR}{fmt_time(remaining)}{R}"
          f"{DIM}  ({done}/{total}){R}")

# ─────────────────────────────────────────────────────────────────────────────
#  BANNER
# ─────────────────────────────────────────────────────────────────────────────

def banner():
    W  = 48
    l1 = "gsFALCON Video Converter"
    l2 = "Conversao Inteligente de Videos"
    def bline(txt):
        pad   = W - len(txt)
        left  = pad // 2
        right = pad - left
        return f"\u2551{' '*left}{txt}{' '*right}\u2551"
    top = f"\u2554{'\u2550'*W}\u2557"
    bot = f"\u255a{'\u2550'*W}\u255d"
    print(f"\n{CY}{B}{top}")
    print(bline(l1))
    print(bline(l2))
    print(f"{bot}{R}")

    if GPU_ENCODER:
        gpu_label = GPU_NAME or "NVIDIA GPU"
        print(f"  {OR}⚡ {gpu_label}  —  aceleracao NVENC ativa{R}\n")
    elif GPU_FAIL_REASON:
        print(f"  {DIM}GPU NVENC indisponivel — usando CPU (libx264){R}")
        print(f"  {DIM}Motivo: {GPU_FAIL_REASON}{R}\n")
    else:
        print(f"  {DIM}GPU NVENC nao detectada — usando CPU (libx264){R}\n")

# ─────────────────────────────────────────────────────────────────────────────
#  ORQUESTRACAO PARALELA
# ─────────────────────────────────────────────────────────────────────────────

def run_parallel(all_v, ext, th, container, audio_mode, n_workers):
    """N threads fixas, cada uma dona de um slot do painel, consumindo
    arquivos de uma fila compartilhada ate esvaziar."""
    total = len(all_v)
    q = queue.Queue()
    for idx, (src, src_root) in enumerate(all_v, 1):
        q.put((idx, src, src_root))

    # Slot 0 = linha de "Overall", slots 1..n_workers = um por worker
    panel = LivePanel(n_workers + 1)
    panel.update(0, f"{DIM}  Overall: 0/{total}{R}")
    for i in range(1, n_workers + 1):
        panel.update(i, f"{DIM}  [ocioso]{R}")

    results = {"ok": 0, "err": 0, "done": 0, "elapsed": 0.0}
    results_lock = threading.Lock()
    start_wall = time.time()

    def worker(slot_idx):
        while True:
            try:
                idx, src, src_root = q.get_nowait()
            except queue.Empty:
                return
            dst = build_dst(src, src_root, ext)
            rel = src.relative_to(src_root)
            try:
                elapsed, summary = process_file_parallel(
                    src, dst, th, idx, total, src_root, container, audio_mode, panel, slot_idx
                )
                with results_lock:
                    results["ok"]      += 1
                    results["elapsed"] += elapsed
                    results["done"]    += 1
                    done = results["done"]
                panel.print_permanent(f"  {GR}✔{R} [{idx}/{total}] {rel}  —  {summary}")
            except Exception as e:
                with results_lock:
                    results["err"]  += 1
                    results["done"] += 1
                    done = results["done"]
                try:
                    if dst.exists(): dst.unlink(missing_ok=True)
                except: pass
                panel.print_permanent(f"  {RE}✖{R} [{idx}/{total}] {rel}  —  ERRO: {e}")

            with results_lock:
                avg = results["elapsed"] / results["ok"] if results["ok"] else 0
                remaining = (total - results["done"]) * avg if avg else 0
            bar = progress_bar(done / total, width=28)
            panel.update(0, f"  {B}Overall:{R} {bar}  ({done}/{total})  "
                            f"{DIM}restante ~{fmt_time(remaining)}{R}")
            panel.update(slot_idx, f"{DIM}  [ocioso]{R}")
            q.task_done()

    # Slot 0 e a linha de overall — cada worker usa seu proprio slot 1..n_workers
    threads = [threading.Thread(target=worker, args=(i,), daemon=True) for i in range(1, n_workers + 1)]

    try:
        for t in threads:
            t.start()
        for t in threads:
            t.join()
    except KeyboardInterrupt:
        with ACTIVE_PROCS_LOCK:
            for p in list(ACTIVE_PROCS):
                try: p.terminate()
                except: pass
        panel.close()
        raise

    panel.close()
    total_t = time.time() - start_wall
    return results["ok"], results["err"], total_t

# ─────────────────────────────────────────────────────────────────────────────
#  MAIN
# ─────────────────────────────────────────────────────────────────────────────

def run():
    cls()
    detect_gpu()
    banner()

    folders = ask_folders()

    print(f"\n{DIM}  Escaneando pastas...{R}")
    all_v      = []
    subfolders = set()
    for folder in folders:
        for v in collect_videos(folder):
            all_v.append((v, folder))
            if v.parent != folder:
                subfolders.add(str(v.parent))

    if not all_v:
        print(f"\n{YE}  ⚠ Nenhum video encontrado.{R}")
        input(f"\n{DIM}  Pressione ENTER para fechar...{R}")
        sys.exit(0)

    n_sub   = len(subfolders)
    sub_txt = f"em {n_sub} subpasta(s)" if n_sub else "na pasta raiz"
    print(f"  🎬 {GR}{B}{len(all_v)} video(s) encontrado(s){R}  {DIM}{sub_txt}{R}")

    _, fmt_label, ext          = ask_format()
    _, res_label, tw, th       = ask_resolution()
    _, aud_label, audio_mode   = ask_audio()
    n_workers                  = ask_parallel()
    container = fmt_label.lower()

    print(f"\n{CY}{B}{'─'*64}{R}")
    print(f"  📦 {DIM}Formato:    {B}{fmt_label}{R}")
    print(f"  🎯 {DIM}Resolucao:  {B}{res_label}{R}")
    print(f"  🔊 {DIM}Audio:      {B}{aud_label}{R}")
    if audio_mode == "enhance":
        print(f"     {DIM}            {OR}(experimental — qualidade pode variar){R}")
    print(f"  ⚙️  {DIM}Paralelo:   {B}{n_workers}x{R}  {DIM}simultaneo(s){R}")
    print(f"  📁 {DIM}Saida:      pasta '{OUT_FOLDER}' dentro de cada pasta informada{R}")
    print(f"{CY}{B}{'─'*64}{R}\n")
    input(f"  {B}Pressione ENTER para iniciar...{R}")

    if n_workers == 1:
        ok_n = err_n = 0
        elapsed_ok  = 0.0
        start_wall  = time.time()

        for idx, (src, src_root) in enumerate(all_v, 1):
            dst = build_dst(src, src_root, ext)
            try:
                elapsed     = process_file(src, dst, th, idx, len(all_v), src_root, container, audio_mode)
                ok_n       += 1
                elapsed_ok += elapsed
            except Exception as e:
                err_n += 1
                try:
                    if dst.exists(): dst.unlink(missing_ok=True)
                except: pass
                print(f"\n  ❌ {RE}{B}ERRO:{R} {e}{R}")

            print_eta(idx, len(all_v), elapsed_ok, ok_n)

        total_t = time.time() - start_wall
    else:
        print()
        ok_n, err_n, total_t = run_parallel(all_v, ext, th, container, audio_mode, n_workers)

    print(f"\n{CY}{B}{'='*64}{R}")
    print(f"  ✅ {GR}{B}{ok_n} arquivo(s) convertido(s) com sucesso{R}")
    if err_n:
        print(f"  ❌ {RE}{err_n} erro(s){R}")
    print(f"  ⏱  {DIM}Tempo total: {fmt_time(total_t)}{R}")
    print(f"  📁 {DIM}Arquivos salvos em: pasta '{OUT_FOLDER}'{R}")
    print(f"{CY}{B}{'='*64}{R}")
    input(f"\n{DIM}  Pressione ENTER para fechar...{R}")


if __name__ == "__main__":
    for tool in ("ffmpeg", "ffprobe"):
        if not shutil.which(tool):
            print(f"\n{RE}'{tool}' nao encontrado no PATH.")
            print(f"Instale em: https://ffmpeg.org/download.html{R}\n")
            input("Pressione ENTER para fechar...")
            sys.exit(1)
    try:
        run()
    except KeyboardInterrupt:
        with ACTIVE_PROCS_LOCK:
            for p in list(ACTIVE_PROCS):
                try: p.terminate()
                except: pass
        print(f"\n\n{YE}  Interrompido pelo usuario.{R}")
        input(f"\n{DIM}  Pressione ENTER para fechar...{R}")
    except Exception as e:
        import traceback
        print(f"\n{RE}{B}ERRO INESPERADO:{R}")
        print(f"{RE}{traceback.format_exc()}{R}")
        input(f"\n{DIM}  Pressione ENTER para fechar...{R}")
        sys.exit(1)
```
