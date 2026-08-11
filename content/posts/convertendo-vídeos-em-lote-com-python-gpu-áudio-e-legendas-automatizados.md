---
title: "Convertendo vídeos em lote com Python: GPU, áudio e legendas automatizados"
date: 2026-08-10T05:10:00.000-03:00
draft: false
description: Script para converter lotes de vídeos (MKV/MP4), com aceleração por
  GPU NVIDIA, normalização de áudio e legendas automáticas, usando Python e
  ffmpeg
tags:
  - Python
  - Windows
  - Vídeo
  - Áudio
  - ffmpeg
categories:
  - Python
  - Windows
  - Vídeo
  - Áudio
  - ffmpeg
cover:
  image: /images/uploads/convert-videos.png
  alt: Script para converter lotes de vídeos (MKV/MP4), com aceleração por GPU
    NVIDIA, normalização de áudio e legendas automáticas, usando Python e ffmpeg
ShowToc: true
comments: true
image: /images/uploads/convert-videos.png
---
Se você converte lote de vídeos antigos (séries, gravações, arquivos pessoais) e cansou de configurar o ffmpeg na mão toda vez — escolher codec, resolução, cuidar de faixas de áudio e legenda, lembrar dos parâmetros de GPU — dá pra automatizar isso tudo num script interativo que faz as perguntas certas e monta o comando do ffmpeg sozinho. 

Neste tutorial eu mostro o script Python que uso pra isso, como ele decide entre GPU e CPU, como monta a cadeia de filtros de áudio, e como ele processa vários vídeos em paralelo com um painel ao vivo no terminal.

![](/images/uploads/captura-de-tela-2026-08-10-053146.png "Script Conversor de Vídeos em Lote")

## O que o script faz 🧠

* Recebe uma ou mais pastas (separadas por vírgula) e varre recursivamente atrás de vídeos.
* Deixa escolher o container de saída (MKV ou MP4).
* Deixa escolher a resolução alvo (Original, 480p, 720p, 1080p, 2160p) — só redimensiona se for necessário.
* Três modos de áudio: normalização para -14 LUFS, manter original (copy), ou um "Voice Enhancer" experimental para áudio abafado de fitas antigas.
* Detecta GPU NVIDIA (NVENC) automaticamente, com teste real de encode — não só verifica se está compilado no ffmpeg — e cai pra CPU (libx264) se algo falhar.
* Preserva nomes e idiomas das faixas de áudio/legenda, e embute `.srt` externo automaticamente se encontrar um com o mesmo nome do vídeo.
* Roda em paralelo (1 a 4 vídeos simultâneos) com um painel fixo no terminal, uma linha por worker.
* Salva tudo numa pasta `_NEW/` espelhando a estrutura de subpastas original, sem tocar nos arquivos fonte.

Importante: o script **não apaga nem sobrescreve os originais** — a saída sempre vai para `_NEW/`, ao lado dos arquivos de entrada.

## Dependências 📦

Você precisa de Python 3.10+ e do `ffmpeg` + `ffprobe` no PATH.

```bash
# Windows: baixe em ffmpeg.org e adicione ao PATH
# Linux: sudo apt install ffmpeg
```

Não precisa de nenhuma biblioteca Python externa — só a standard library (`subprocess`, `pathlib`, `threading`, `queue`, `json`).

## Como o script está organizado ⚙️

Em linhas gerais, o fluxo é:

1. **Detecção de GPU**: roda um encode de teste curtinho com `h264_nvenc` pra confirmar que a GPU funciona de verdade nesta máquina, não só que o ffmpeg tem o encoder compilado.
2. **Probe único por arquivo**: uma chamada ao `ffprobe` traz duração, resolução, codec de vídeo e todas as faixas de áudio/legenda de uma vez — evita abrir vários processos por arquivo em lotes grandes.
3. **Menus interativos**: pergunta formato, resolução, modo de áudio e grau de paralelismo antes de começar.
4. **Montagem do comando ffmpeg**: monta os parâmetros de vídeo, áudio e legenda dinamicamente com base nas respostas e no que o probe encontrou.
5. **Execução**: sequencial (relatório detalhado por arquivo) ou paralela (painel compacto com N workers consumindo uma fila compartilhada).

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

## Voice Enhancer \[BETA] 🩺

Esse modo é pensado pra vídeos antigos (VHS/Hi8) com áudio abafado. Ele encadeia vários filtros de áudio antes da normalização: corte de graves (`highpass`), redução de ruído por FFT (`afftdn`), realce de diálogo (`dialoguenhance`), um leve boost na faixa de presença vocal (2-4kHz) e compressão, só então normalizando para -14 LUFS. É experimental — o resultado varia de vídeo pra vídeo.

## Paralelismo 🧵

Ao escolher rodar 2, 3 ou 4 vídeos ao mesmo tempo, o script usa uma fila (`queue.Queue`) e N threads fixas, cada uma dona de uma linha do painel. O painel redesenha a tela a cada atualização, movendo o cursor pra cima e limpando até o fim — evita sobra de texto quando uma linha nova é mais curta que a anterior.

## Rodando o script ▶️

```bash
python conversor_video.py
```

O script vai pedir a(s) pasta(s), depois formato, resolução, modo de áudio e grau de paralelismo, mostra um resumo e pede confirmação antes de começar.

Os arquivos convertidos vão para `_NEW/` dentro de cada pasta informada, espelhando as subpastas originais.

## Problemas comuns e como resolver 🚑

** `'ffmpeg' nao encontrado no PATH`**
Instale o ffmpeg e garanta que `ffmpeg` e `ffprobe` estejam acessíveis no terminal.

** Conversão cai pra CPU mesmo tendo GPU NVIDIA**
O teste de encode real (`detect_gpu`) pode estar falhando por driver desatualizado ou falta do `nvidia-smi`. O motivo exato aparece no banner inicial, na linha "Motivo: ...".

** Legenda em imagem (PGS) sumiu ao converter pra MP4**
Esperado — o container MP4 só aceita legenda de texto (`mov_text`). Pra preservar legendas bitmap, use MKV.

** GPU falha no meio da conversão**
O script já tenta automaticamente de novo por CPU (`force_cpu=True`) antes de reportar erro — então normalmente nem é preciso intervir.

## Adaptando 🔧

Pra mudar o padrão de qualidade da GPU, ajuste `-cq` em `build_cmd` (menor = mais qualidade, mais lento). Pra mudar o alvo de loudness, edite `TARGET_LUFS`/`TARGET_TP`/`TARGET_LRA` no topo do arquivo.

## Download ⬇️

[Conversor de Vídeo em Lote](/downloads/conversor-de-video-em-lote.zip)
