---
title: Agendando o desligamento do PC com um menu em Python
date: 2026-08-10T18:01:00.000-03:00
draft: false
image: "/images/uploads/shutdown.png"
description: Script Python com menu no terminal para agendar o desligamento do
  PC, com confirmação, cancelamento e suporte a Windows e Linux
tags:
  - 🐍 Python
  - 🪟 Windows
  - 🐧 Linux
  - 🤖 Automação
categories: []
cover:
  image: /images/uploads/shutdown.png
ShowToc: true
comments: true
---
Script Python que agenda o desligamento do PC por um menu no terminal, com opções fixas de 1 a 12 horas, confirmação antes de executar e suporte a Windows e Linux.

## 🧠 O que o script faz

* 🕐 Mostra um menu com atalhos de 1h, 2h, 4h, 6h, 8h, 10h e 12h pra desligar o PC.
* ✅ Pede confirmação (ENTER) antes de agendar qualquer desligamento — nada acontece sem aviso.
* ❌ Tem uma opção pra cancelar todos os agendamentos ativos.
* 🖼️ Desenha um menu em caixa ANSI colorida, com alinhamento correto mesmo usando emojis/símbolos, que normalmente bagunçam o espaçamento no terminal.
* 🪟🐧 Funciona tanto no Windows (`shutdown -s -t`) quanto no Linux (`at` + `shutdown -h`).
* 🏷️ Define o título da janela do terminal e limpa a tela entre as telas, pra parecer um app de verdade, não só um script rodando.

⚠️ Importante: no Linux, o script depende do comando `at` estar instalado e do usuário ter permissão de `sudo` sem senha configurada — senão o agendamento não é silencioso.

![](/images/uploads/captura-de-tela-2026-08-10-175818.png)

## 📦 Dependências

Só a standard library do Python (`os`, `sys`, `time`, `re`, `unicodedata`, `subprocess`, `platform`) — nenhum pacote externo.

```bash
# Linux: garanta que o pacote "at" está instalado
sudo apt install at
```

No Windows não precisa instalar nada além do próprio Python — o comando `shutdown` já vem no sistema.

## ⚙️ Como o script está organizado

Em linhas gerais, o fluxo é:

1. 🧭 **Menu principal**: mostra as opções de horário num loop, lê a escolha do usuário.
2. ✅ **Confirmação**: antes de agendar de fato, exibe uma mensagem e espera ENTER (ou Ctrl+C pra cancelar).
3. 🛠️ **Agendamento por SO**: monta o comando certo pra Windows (`shutdown -s -t segundos`) ou Linux (`at` com horário calculado).
4. 🖼️ **Renderização da caixa**: calcula a largura visual real de cada linha — inclusive com emojis — pra alinhar as bordas da caixa ANSI perfeitamente.

Um trecho central é o cálculo de largura visual, que trata emojis como ocupando 2 colunas mesmo quando o Python não detecta isso automaticamente:

```python
def char_width(ch):
    cp = ord(ch)
    if cp in _ZERO_WIDTH:
        return 0
    if cp in _WIDE_EXTRA:
        return 2
    if cp >= 0x10000:
        return 2
    eaw = unicodedata.east_asian_width(ch)
    if eaw in ('W', 'F'):
        return 2
    return 1
```

E o agendamento em si, que trata os dois sistemas operacionais de forma diferente:

```python
if platform.system() == "Windows":
    subprocess.run(["shutdown", "-a"], capture_output=True)
    subprocess.run(["shutdown", "-s", "-t", str(seconds), "-f", "-c", msg], capture_output=True)
else:
    subprocess.run(["sudo", "shutdown", "-c"], capture_output=True)
    at_t = time.strftime("%H:%M", time.localtime(int(time.time()) + seconds))
    subprocess.run(f'echo "sudo shutdown -h now" | at {at_t}', shell=True, capture_output=True)
```

## 🖼️ Alinhamento visual da caixa

Esse é o detalhe menos óbvio do script: no terminal, nem todo caractere ocupa a mesma largura. Emojis e alguns símbolos (como `⏰`, `☀️`, relógios analógicos) contam como 2 colunas visuais em terminais modernos, mas a função padrão do Python (`unicodedata.east_asian_width`) não reconhece isso pra boa parte deles. O script mantém duas listas manuais — `_WIDE_EXTRA` (caracteres que ocupam 2 colunas) e `_ZERO_WIDTH` (seletores de variação e caracteres invisíveis que ocupam 0) — e usa isso pra preencher cada linha da caixa com a quantidade exata de espaços, mantendo as bordas alinhadas.

## ▶️ Rodando o script

```bash
python shutdown.py
```

O menu aparece direto: escolha um número de 1 a 7 pra agendar, 8 pra cancelar tudo, ou 9 pra sair.

## 🚑 Problemas comuns e como resolver

**❌ Agendamento não funciona no Linux**
Confirme se o pacote `at` está instalado (`sudo apt install at`) e se o comando `sudo shutdown` não está pedindo senha interativa — senão o `subprocess.run` trava ou falha silenciosamente.

**❌ Caixa do menu desalinhada no terminal**
Alguns terminais (principalmente emuladores antigos) renderizam emojis com largura diferente da esperada. Se isso acontecer, vale ajustar as listas `_WIDE_EXTRA`/`_ZERO_WIDTH` pros símbolos específicos que estão bugando.

**❌ Cores ANSI não aparecem no Windows**
O script já chama `enable_ansi_windows()` no início, que ativa o modo de cores via `SetConsoleMode`. Se ainda assim não funcionar, geralmente é um terminal muito antigo (cmd.exe legado) que não suporta ANSI.

**❌ Quero cancelar sem abrir o menu**
Não tem atalho de linha de comando pra isso hoje — a opção "8" do menu é o único jeito. Dá pra adicionar suporte a `sys.argv` se quiser esse atalho.

## 🔧 Adaptando

Pra mudar as opções de horário, edite as listas `opcoes` (na tela) e `opcoes_map` (na lógica) em paralelo — são duas listas separadas que precisam ficar sincronizadas. Pra mudar a largura da caixa, ajuste a constante `W` no topo do arquivo.

## 💻 Código completo

```python
import os
import sys
import time
import re
import unicodedata
import subprocess
import platform

# ── ANSI Colors ───────────────────────────────────────────────────────────────
RED     = "\033[91m"
YELLOW  = "\033[93m"
GREEN   = "\033[92m"
CYAN    = "\033[96m"
MAGENTA = "\033[95m"
WHITE   = "\033[97m"
BOLD    = "\033[1m"
DIM     = "\033[2m"
RESET   = "\033[0m"

# Largura interna da caixa (colunas visíveis entre ║ e ║)
W = 62

# ── Largura visual precisa ────────────────────────────────────────────────────

# Emojis/símbolos que terminals modernos renderizam com 2 colunas,
# mas unicodedata.east_asian_width não detecta corretamente como Wide.
_WIDE_EXTRA = {
    # Relógios analógicos 🕐-🕛 e 🕜-🕧
    *range(0x1F550, 0x1F568),
    # Rostos / emojis comuns do plano BMP que aparecem com 2 cols
    0x231A, 0x231B,  # ⌚ ⌛
    0x23E9, 0x23EA, 0x23EB, 0x23EC, 0x23ED, 0x23EE, 0x23EF,
    0x23F0, 0x23F3,  # ⏰ ⏳
    0x25AA, 0x25AB, 0x25B6, 0x25C0,
    0x25FB, 0x25FC, 0x25FD, 0x25FE,
    0x2600, 0x2601, 0x2602, 0x2603, 0x2604,
    0x260E, 0x2611, 0x2614, 0x2615,
    0x2618, 0x261D, 0x2620, 0x2622, 0x2623,
    0x2626, 0x262A, 0x262E, 0x262F,
    0x2638, 0x2639, 0x263A,
    0x2640, 0x2642,
    *range(0x2648, 0x2654),  # signos
    0x265F, 0x2660, 0x2663, 0x2665, 0x2666, 0x2668,
    0x267B, 0x267E, 0x267F,
    0x2692, 0x2693, 0x2694, 0x2695, 0x2696, 0x2697,
    0x2699, 0x269B, 0x269C,
    0x26A0, 0x26A1, 0x26AA, 0x26AB,
    0x26B0, 0x26B1,
    0x26BD, 0x26BE,
    0x26C4, 0x26C5, 0x26C8,
    0x26CE, 0x26CF, 0x26D1, 0x26D3, 0x26D4,
    0x26E9, 0x26EA,
    *range(0x26F0, 0x26F6),
    0x26F7, 0x26F8, 0x26F9, 0x26FA, 0x26FD,
    0x2702, 0x2705,
    *range(0x2708, 0x2710),
    0x2712, 0x2714, 0x2716, 0x271D, 0x2721, 0x2728,
    0x2733, 0x2734, 0x2744, 0x2747,
    0x274C, 0x274E,
    0x2753, 0x2754, 0x2755, 0x2757,
    0x2763, 0x2764,
    0x2795, 0x2796, 0x2797,
    0x27A1, 0x27B0, 0x27BF,
    0x2934, 0x2935,
    *range(0x2B05, 0x2B08),
    0x2B1B, 0x2B1C, 0x2B50, 0x2B55,
    0x3030, 0x303D, 0x3297, 0x3299,
    # ➤ (seta do prompt) — no Windows Terminal conta como 1; deixa como 1
}

# Seletores de variação e combinadores invisíveis
_ZERO_WIDTH = {
    *range(0xFE00, 0xFE10),   # variation selectors
    *range(0xE0100, 0xE01F0), # variation selectors supplement
    0x200B, 0x200C, 0x200D,   # zero-width space/non-joiner/joiner
    0x200E, 0x200F,            # LRM / RLM
    0xFEFF,                    # BOM / zero-width no-break space
}

def char_width(ch):
    cp = ord(ch)
    if cp in _ZERO_WIDTH:
        return 0
    if cp in _WIDE_EXTRA:
        return 2
    # Plano suplementar (emojis, símbolos, etc.)
    if cp >= 0x10000:
        return 2
    eaw = unicodedata.east_asian_width(ch)
    if eaw in ('W', 'F'):
        return 2
    return 1

def visible_width(text):
    """Calcula a largura visual ignorando ANSI escapes."""
    stripped = re.sub(r'\033\[[0-9;]*m', '', text)
    return sum(char_width(ch) for ch in stripped)

def rpad(text, total_width):
    """Preenche com espaços à direita até total_width colunas visíveis."""
    return text + " " * max(total_width - visible_width(text), 0)

# ── Box helpers ───────────────────────────────────────────────────────────────

def bline(color, content):
    print(f"  {color}{BOLD}║{RESET}{content}{color}{BOLD}║{RESET}")

def htop(color): print(f"  {color}{BOLD}╔{'═' * W}╗{RESET}")
def hmid(color): print(f"  {color}{BOLD}╠{'═' * W}╣{RESET}")
def hbot(color): print(f"  {color}{BOLD}╚{'═' * W}╝{RESET}")

# ── Telas ─────────────────────────────────────────────────────────────────────

def enable_ansi_windows():
    if platform.system() == "Windows":
        os.system("color")
        try:
            import ctypes
            kernel32 = ctypes.windll.kernel32
            kernel32.SetConsoleMode(kernel32.GetStdHandle(-11), 7)
        except Exception:
            pass

def clear():
    os.system("cls" if platform.system() == "Windows" else "clear")

def set_title(title):
    if platform.system() == "Windows":
        os.system(f"title {title}")

def print_header(icon="[*]"):
    inner = f"  {icon}  {WHITE}{BOLD}SHUTDOWN PRO  --  DESLIGAMENTO AUTOMATICO  {icon}{RESET}"
    print()
    htop(RED)
    bline(RED, rpad(inner, W))
    hbot(RED)
    print()

def print_menu():
    titulo = f"   {YELLOW}[>] Escolha quando desligar o PC:{RESET}"
    print()
    htop(CYAN)
    bline(CYAN, rpad(titulo, W))
    hmid(CYAN)

    opcoes = [
        ("1", "[1h]",  "1 hora"),
        ("2", "[2h]",  "2 horas"),
        ("3", "[4h]",  "4 horas"),
        ("4", "[6h]",  "6 horas"),
        ("5", "[8h]",  "8 horas"),
        ("6", "[10h]", "10 horas"),
        ("7", "[12h]", "12 horas"),
    ]

    for num, badge, label in opcoes:
        row = f"   {GREEN}{BOLD}[{num}]{RESET}  {CYAN}{badge}{RESET}  {WHITE}{label}{RESET}"
        bline(CYAN, rpad(row, W))

    hmid(CYAN)
    cancel_row = f"   {RED}{BOLD}[8]{RESET}  {RED}[X]{RESET}  {WHITE}Cancelar todos os agendamentos{RESET}"
    bline(CYAN, rpad(cancel_row, W))
    sair_row   = f"   {DIM}[9]{RESET}  {DIM}[>>]{RESET}  {DIM}Sair{RESET}"
    bline(CYAN, rpad(sair_row, W))
    hbot(CYAN)
    print()

def print_status_bar():
    now      = time.strftime("%H:%M:%S")
    sys_name = platform.node()
    print(f"  {DIM}[PC] {sys_name}   [T] {now}{RESET}")
    print()

def animate_header():
    frames = ["[*]", "[!]", "[*]", "[!]", "[*]"]
    for f in frames:
        clear()
        print_header(f)
        time.sleep(0.12)

def countdown():
    print(f"\n  {DIM}Voltando ao menu em 3 segundos", end="", flush=True)
    for _ in range(3):
        time.sleep(1)
        print(".", end="", flush=True)
    print(RESET)

def confirm(message):
    print(f"\n  {YELLOW}{BOLD}[!]  {message}{RESET}")
    print(f"  {DIM}Pressione {BOLD}ENTER{RESET}{DIM} para confirmar ou {BOLD}Ctrl+C{RESET}{DIM} para cancelar...{RESET}")
    try:
        input()
        return True
    except KeyboardInterrupt:
        return False

def schedule_shutdown(seconds, label):
    if not confirm(f"O PC sera desligado em {label}. Confirmar?"):
        print(f"\n  {YELLOW}[!] Acao cancelada.{RESET}\n")
        time.sleep(1.5)
        return

    msg = f"O PC sera desligado em {label}."
    if platform.system() == "Windows":
        subprocess.run(["shutdown", "-a"], capture_output=True)
        subprocess.run(["shutdown", "-s", "-t", str(seconds), "-f", "-c", msg], capture_output=True)
    else:
        subprocess.run(["sudo", "shutdown", "-c"], capture_output=True)
        at_t = time.strftime("%H:%M", time.localtime(int(time.time()) + seconds))
        subprocess.run(f'echo "sudo shutdown -h now" | at {at_t}', shell=True, capture_output=True)

    clear()
    print()
    htop(GREEN)
    bline(GREEN, rpad(f"  [OK]  {WHITE}{BOLD}DESLIGAMENTO AGENDADO COM SUCESSO!{RESET}", W))
    hbot(GREEN)
    print(f"\n  {WHITE}  [T] PC sera desligado em: {YELLOW}{BOLD}{label}{RESET}\n")
    countdown()

def cancel_shutdown():
    if platform.system() == "Windows":
        subprocess.run(["shutdown", "-a"], capture_output=True)
    else:
        subprocess.run(["sudo", "shutdown", "-c"], capture_output=True)

    clear()
    print()
    htop(RED)
    bline(RED, rpad(f"  [X]  {WHITE}{BOLD}TODOS OS AGENDAMENTOS FORAM CANCELADOS!{RESET}", W))
    hbot(RED)
    print(f"\n  {WHITE}  Desligue o PC manualmente quando quiser.{RESET}\n")
    countdown()

def show_goodbye():
    clear()
    print()
    htop(MAGENTA)
    bline(MAGENTA, rpad(f"  [>>]  {WHITE}{BOLD}Ate logo! Shutdown Pro encerrado.{RESET}", W))
    hbot(MAGENTA)
    print()
    time.sleep(1)

# ── Main ──────────────────────────────────────────────────────────────────────

def main():
    enable_ansi_windows()
    set_title("SHUTDOWN PRO - Desligamento Automatico")

    opcoes_map = {
        "1": (3600,  "1 hora"),
        "2": (7200,  "2 horas"),
        "3": (14400, "4 horas"),
        "4": (21600, "6 horas"),
        "5": (28800, "8 horas"),
        "6": (36000, "10 horas"),
        "7": (43200, "12 horas"),
    }

    animate_header()

    while True:
        clear()
        print_header()
        print_menu()
        print_status_bar()

        try:
            choice = input(f"  {BOLD}{CYAN}>>  Digite a opcao:{RESET} ").strip()
        except KeyboardInterrupt:
            show_goodbye()
            sys.exit(0)

        if choice in opcoes_map:
            schedule_shutdown(*opcoes_map[choice])
        elif choice == "8":
            cancel_shutdown()
        elif choice == "9":
            show_goodbye()
            sys.exit(0)
        else:
            print(f"\n  {RED}[!] Opcao invalida! Tente novamente.{RESET}")
            time.sleep(1.2)

if __name__ == "__main__":
    main()
```
