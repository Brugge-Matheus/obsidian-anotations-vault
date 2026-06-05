---
tags:
  - sistemas-operacionais
  - so/conceitos
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.5.6"
---
# Interpretador de comandos Shell

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.5.6

---

# 🖥️ O que é o Shell?

O sistema operacional é o código que executa as chamadas de sistema. Editores, compiladores, montadores, ligadores (*linkers*), programas utilitários e interpretadores de comandos definitivamente **não fazem parte do sistema operacional**, mesmo que sejam importantes e úteis.

O **shell** é o interpretador de comandos UNIX — um programa que roda fora do kernel e usa intensamente as chamadas de sistema do SO. Ele é a principal **interface entre o usuário e o sistema operacional**, a menos que o usuário esteja usando uma interface gráfica (GUI).

> 💡 O shell não faz parte do SO — é um programa comum que roda sobre ele. Mas serve como um excelente exemplo de como as chamadas de sistema são usadas, pois praticamente tudo que o shell faz envolve uma chamada de sistema.
> 

Existem vários shells diferentes, todos derivados do shell original (`sh`):

| Shell | Observação |
| --- | --- |
| `sh` | Shell original (Bourne Shell) |
| `csh` | C Shell — sintaxe inspirada em C |
| `ksh` | Korn Shell |
| `bash` | Bourne Again Shell — o mais comum no Linux |
| `zsh` | Z Shell — popular no macOS |

---

# 🔄 Como o Shell Funciona

Quando qualquer usuário se conecta ao sistema, um shell é iniciado. O shell tem o terminal como entrada-padrão e exibe um **prompt** — geralmente `$` para usuários comuns e `#` para o root — indicando que está pronto para aceitar um comando.

O ciclo de vida básico do shell:

```
Shell exibe prompt: $
        ↓
Usuário digita um comando (ex: date)
        ↓
Shell analisa o comando
        ↓
Shell cria um PROCESSO-FILHO via fork()
        ↓
Processo-filho executa o programa (date)
        ↓
Shell AGUARDA o processo-filho terminar
        ↓
Processo-filho termina
        ↓
Shell exibe o prompt novamente: $
```

**Exemplo:**

```bash
$ date
Qui Mai  8 23:00:00 2026
$
```

O shell criou um processo-filho, executou o programa `date` como esse filho, aguardou ele terminar, e então voltou a exibir o prompt.

---

# ↪️ Redirecionamento de I/O

Uma das funcionalidades mais poderosas do shell é o **redirecionamento** — mudar de onde um programa lê sua entrada e para onde envia sua saída, sem modificar o programa em si.

## Redirecionamento de Saída (`>`)

Por padrão, a saída de um programa vai para o terminal. Com `>`, ela vai para um arquivo:

```bash
# Sem redirecionamento — saída aparece na tela:
date
Qui Mai  8 23:00:00 2026

# Com redirecionamento — saída vai para o arquivo:
date > arquivo.txt
# nada aparece na tela, mas arquivo.txt contém a data
```

## Redirecionamento de Entrada (`<`)

Por padrão, a entrada de um programa vem do teclado. Com `<`, ela vem de um arquivo:

```bash
# sort lê do teclado por padrão
# com redirecionamento, lê de arquivo1 e escreve em arquivo2:
sort < arquivo1 > arquivo2
```

## Pipes (`|`)

A saída de um programa pode ser usada como entrada de outro — conectando-os com pipes:

```bash
# cat concatena os três arquivos → sort organiza alfabeticamente → impressora imprime
cat arquivo1 arquivo2 arquivo3 | sort > /dev/lp
```

Essa cadeia de comandos é poderosa porque cada programa faz uma coisa simples, e o shell os conecta via pipes para realizar tarefas complexas. É o princípio **"faça uma coisa e faça bem"** do UNIX.

---

# ⚡ Execução em Segundo Plano (`&`)

Normalmente o shell aguarda o processo-filho terminar antes de mostrar o prompt novamente. Mas se o usuário colocar um `&` após o comando, o shell **não espera** — ele emite o prompt imediatamente e o processo roda em **segundo plano** (*background*):

```bash
# Sem & — shell aguarda o sort terminar antes de mostrar o prompt
cat arquivo1 arquivo2 arquivo3 | sort > /dev/lp

# Com & — shell volta imediatamente, sort continua em background
cat arquivo1 arquivo2 arquivo3 | sort > /dev/lp &
[1] 4231              ← número do job e PID do processo
$                     ← prompt imediato, usuário pode continuar trabalhando
```

Isso é especialmente útil para tarefas demoradas — compilações longas, transferências de arquivos, processamento de dados — que não precisam de interação do usuário.

---

# 🖱️ GUI — O Shell Gráfico

A maioria dos computadores pessoais usa hoje uma **GUI** (*Graphical User Interface* — interface gráfica do usuário). Na realidade, a GUI é apenas um **programa sendo executado em cima do sistema operacional** — exatamente como o shell, só que com janelas, ícones e mouse ao invés de texto.

```
┌─────────────────────────────────────────┐
│           Programas do usuário          │
│   (editor, navegador, reprodutor...)    │
├─────────────────────────────────────────┤
│    GUI (Gnome, KDE, Windows Explorer)   │  ← programa rodando sobre o SO
│    ou Shell (bash, zsh, cmd.exe)        │  ← ambos são apenas programas
├─────────────────────────────────────────┤
│         Sistema Operacional             │  ← kernel — executa chamadas de sistema
├─────────────────────────────────────────┤
│              Hardware                   │
└─────────────────────────────────────────┘
```

No Linux, o usuário tem uma escolha de diversas GUIs: **Gnome**, **KDE**, ou nenhuma (usando uma janela de terminal no X11). No Windows, a troca da área de trabalho GUI padrão não é muito comum.

> 💡 **Distinção importante:** Tanto o shell quanto a GUI são programas comuns que rodam *sobre* o SO e fazem chamadas de sistema para tudo que precisam. Nenhum deles faz parte do kernel. Um usuário Linux pode inclusive rodar múltiplas GUIs diferentes, ou nenhuma, e o SO continua funcionando normalmente.
> 

---

# 🔗 Relação com Chamadas de Sistema

O shell é um excelente exemplo de como chamadas de sistema são usadas na prática. Cada ação simples que o shell faz envolve várias chamadas de sistema:

```
Usuário digita: ls -la /home

Shell executa:
  fork()          → cria processo-filho
  execve()        → processo-filho vira o programa "ls"
  wait()          → shell aguarda ls terminar
  write()         → exibe o prompt novamente

Com redirecionamento (ls > arquivo.txt):
  open()          → abre arquivo.txt para escrita
  dup2()          → redireciona stdout para o arquivo
  fork() + execve() → executa ls
  close()         → fecha o arquivo
  wait()          → aguarda terminar
```

---

# ✅ Resumo do Conceito

- O **shell** é um programa que roda *sobre* o SO — **não faz parte do kernel**
- É a principal interface entre o usuário e o SO quando não há GUI
- Existem vários shells: `sh`, `csh`, `ksh`, `bash`, `zsh` — todos com a mesma funcionalidade básica
- O shell cria um **processo-filho** para cada comando executado, aguarda terminar e exibe o prompt novamente
- **Redirecionamento de saída** (`>`) envia a saída para um arquivo; **de entrada** (`<`) lê de um arquivo
- **Pipes** (`|`) conectam a saída de um programa à entrada de outro — sem arquivo temporário
- O operador `&` executa um processo em **segundo plano**, devolvendo o prompt imediatamente
- A **GUI** é apenas um programa rodando sobre o SO — equivalente a um shell gráfico, não parte do kernel