---
draft: false
featureimage: https://gsfalcon.com/images/uploads/shutdown.png
title: Agendando o desligamento do PC com um menu em Python
date: 2026-08-10T18:01:00.000-03:00
description: Script Python com menu no terminal para agendar o desligamento do
  PC, com confirmação, cancelamento e suporte a Windows e Linux
tags:
  - Windows
  - Python
  - Automação
  - Linux
  - Script
categories:
  - Windows
  - Python
  - Automação
  - Linux
  - Script
cover:
  image: ""
ShowToc: true
comments: true
image: /images/uploads/shutdown.png
---
Script Python que agenda o desligamento do PC por um menu no terminal, com opções fixas de 1 a 12 horas, confirmação antes de executar e suporte a Windows e Linux.

## O que o script faz

* Mostra um menu com atalhos de 1h, 2h, 4h, 6h, 8h, 10h e 12h pra desligar o PC.
* Pede confirmação (ENTER) antes de agendar qualquer desligamento — nada acontece sem aviso.
* Tem uma opção pra cancelar todos os agendamentos ativos.
* Desenha um menu em caixa ANSI colorida, com alinhamento correto mesmo usando emojis/símbolos, que normalmente bagunçam o espaçamento no terminal.
* Funciona tanto no Windows (`shutdown -s -t`) quanto no Linux (`at` + `shutdown -h`).
* Define o título da janela do terminal e limpa a tela entre as telas, pra parecer um app de verdade, não só um script rodando.

Importante: no Linux, o script depende do comando `at` estar instalado e do usuário ter permissão de `sudo` sem senha configurada — senão o agendamento não é silencioso.

![](/images/uploads/captura-de-tela-2026-08-10-175818.png)

## Dependências

Só a standard library do Python (`os`, `sys`, `time`, `re`, `unicodedata`, `subprocess`, `platform`) — nenhum pacote externo.

```bash
# Linux: garanta que o pacote "at" está instalado
sudo apt install at
```

No Windows não precisa instalar nada além do próprio Python — o comando `shutdown` já vem no sistema.

## Como o script está organizado

Em linhas gerais, o fluxo é:

1. **Menu principal**: mostra as opções de horário num loop, lê a escolha do usuário.
2. **Confirmação**: antes de agendar de fato, exibe uma mensagem e espera ENTER (ou Ctrl+C pra cancelar).
3. **Agendamento por SO**: monta o comando certo pra Windows (`shutdown -s -t segundos`) ou Linux (`at` com horário calculado).
4. **Renderização da caixa**: calcula a largura visual real de cada linha — inclusive com emojis — pra alinhar as bordas da caixa ANSI perfeitamente.

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

## Alinhamento visual da caixa

Esse é o detalhe menos óbvio do script: no terminal, nem todo caractere ocupa a mesma largura. Emojis e alguns símbolos (como `⏰`, ``, relógios analógicos) contam como 2 colunas visuais em terminais modernos, mas a função padrão do Python (`unicodedata.east_asian_width`) não reconhece isso pra boa parte deles. O script mantém duas listas manuais — `_WIDE_EXTRA` (caracteres que ocupam 2 colunas) e `_ZERO_WIDTH` (seletores de variação e caracteres invisíveis que ocupam 0) — e usa isso pra preencher cada linha da caixa com a quantidade exata de espaços, mantendo as bordas alinhadas.

## Rodando o script

```bash
python shutdown.py
```

O menu aparece direto: escolha um número de 1 a 7 pra agendar, 8 pra cancelar tudo, ou 9 pra sair.

## Problemas comuns e como resolver

** Agendamento não funciona no Linux**
Confirme se o pacote `at` está instalado (`sudo apt install at`) e se o comando `sudo shutdown` não está pedindo senha interativa — senão o `subprocess.run` trava ou falha silenciosamente.

** Caixa do menu desalinhada no terminal**
Alguns terminais (principalmente emuladores antigos) renderizam emojis com largura diferente da esperada. Se isso acontecer, vale ajustar as listas `_WIDE_EXTRA`/`_ZERO_WIDTH` pros símbolos específicos que estão bugando.

** Cores ANSI não aparecem no Windows**
O script já chama `enable_ansi_windows()` no início, que ativa o modo de cores via `SetConsoleMode`. Se ainda assim não funcionar, geralmente é um terminal muito antigo (cmd.exe legado) que não suporta ANSI.

** Quero cancelar sem abrir o menu**
Não tem atalho de linha de comando pra isso hoje — a opção "8" do menu é o único jeito. Dá pra adicionar suporte a `sys.argv` se quiser esse atalho.

## Adaptando 🔧

Pra mudar as opções de horário, edite as listas `opcoes` (na tela) e `opcoes_map` (na lógica) em paralelo — são duas listas separadas que precisam ficar sincronizadas. Pra mudar a largura da caixa, ajuste a constante `W` no topo do arquivo.

## Download

[Desligamento Agendado](/downloads/desligamento-agendado.zip)
