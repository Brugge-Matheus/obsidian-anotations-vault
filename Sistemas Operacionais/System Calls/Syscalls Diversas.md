---
tags:
  - sistemas-operacionais
  - so/syscalls
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.6.4"
---
# Syscalls Diversas

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.6.4

---

# 🔧 O que são Syscalls Diversas?

Além das chamadas de sistema para gerenciamento de processos, arquivos e diretórios, existe uma série de outras chamadas que não se encaixam facilmente em uma única categoria. O Tanenbaum examina quatro delas nesta seção — cada uma resolvendo um problema específico e independente.

---

# 📋 As Quatro Syscalls

| Chamada | Descrição |
| --- | --- |
| `s = chdir(dirname)` | Altera o diretório de trabalho atual do processo |
| `s = chmod(name, mode)` | Altera os bits de proteção de um arquivo |
| `s = kill(pid, signal)` | Envia um sinal para um processo |
| `seconds = time(&seconds)` | Obtém o tempo decorrido desde 1º de janeiro de 1970 |

---

# 📂 chdir() — Mudando o Diretório de Trabalho

Todo processo tem um **diretório de trabalho atual** — o ponto de referência para caminhos relativos. Por padrão, quando um processo é criado, herda o diretório de trabalho do seu pai.

`chdir()` muda esse diretório de trabalho:

```c
chdir("/usr/ast/test");
// A partir daqui, uma abertura de arquivo como
// open("xyz", O_RDONLY)
// abrirá /usr/ast/test/xyz — não mais o xyz do diretório anterior
```

Isso elimina a necessidade de digitar nomes de caminhos absolutos longos a toda hora — você navega para o diretório desejado uma vez e depois usa caminhos relativos.

> 💡 Cada processo tem seu próprio diretório de trabalho — mudar o diretório em um processo não afeta nenhum outro processo, nem mesmo o pai que o criou.
> 

---

# 🔒 chmod() — Alterando as Permissões de um Arquivo

`chmod()` altera os bits de proteção de um arquivo. Todo arquivo UNIX tem 9 bits de permissão — três campos (owner, group, others) com três bits cada (r, w, x):

```c
s = chmod(name, mode);
```

- **name** — nome do arquivo
- **mode** — novo valor dos bits de permissão, normalmente em octal

```c
// Exemplos práticos:
chmod("arquivo.txt", 0644);  // rw-r--r-- → dono lê/escreve, grupo e outros só leem
chmod("script.sh",   0755);  // rwxr-xr-x → dono executa, grupo e outros executam
chmod("privado.txt", 0600);  // rw------- → só o dono pode ler e escrever
chmod("publico.txt", 0444);  // r--r--r-- → todos só podem ler, ninguém escreve
```

> 💡 Para que todos usem um arquivo somente como leitura, exceto o proprietário que pode executá-lo, seria `chmod("arquivo", 0644)`. Se o proprietário quisesse poder executar: `chmod("arquivo", 0744)`.
> 

Apenas o **proprietário do arquivo** (ou o root) pode alterar suas permissões.

---

# 📡 kill() — Enviando Sinais para Processos

`kill()` é a maneira pela qual usuários e processos enviam **sinais** para outros processos:

```c
s = kill(pid, signal);
```

- **pid** — o PID do processo destinatário
- **signal** — o número do sinal a ser enviado

## O que é um sinal?

Um sinal é uma notificação assíncrona enviada a um processo para informá-lo de que algum evento ocorreu. É o equivalente software das **interrupções de hardware** — assim como uma interrupção de hardware avisa a CPU sobre um evento externo, um sinal avisa um processo sobre um evento do SO ou de outro processo.

Quando um sinal chega a um processo, duas coisas podem acontecer:

- Se o processo estiver **preparado para capturá-lo** — uma rotina de tratamento é executada, e após encerrar sua ação, o processo é reiniciado no estado em que estava antes do sinal
- Se o processo **não estiver preparado** — o processo é finalizado (comportamento padrão para a maioria dos sinais)

## Sinais mais comuns

| Sinal | Número | Descrição | Pode ser ignorado? |
| --- | --- | --- | --- |
| `SIGTERM` | 15 | Pedido de encerramento gracioso | Sim |
| `SIGKILL` | 9 | Encerramento imediato e forçado | **Não** |
| `SIGINT` | 2 | Interrupção (Ctrl+C no terminal) | Sim |
| `SIGHUP` | 1 | Terminal desconectado | Sim |
| `SIGSEGV` | 11 | Violação de segmentação (segfault) | Sim |
| `SIGALRM` | 14 | Temporizador expirou | Sim |

```c
kill(1234, SIGTERM);  // pede ao processo 1234 para terminar graciosamente
kill(1234, SIGKILL);  // mata o processo 1234 imediatamente — não pode ser ignorado
kill(1234, SIGALRM);  // envia sinal de alarme (temporizador expirou)
```

> ⚠️ `SIGKILL` é o único sinal que **não pode ser ignorado ou capturado** pelo processo destinatário. É a única garantia de que um processo será encerrado. Por isso o comando `kill -9 PID` no terminal é o "último recurso" — mata qualquer processo, sem exceção.
> 

> 💡 O nome `kill()` é um tanto enganoso — a chamada não mata o processo necessariamente. Ela apenas **envia um sinal**. O que acontece depende do sinal enviado e de como o processo está configurado para tratá-lo. `kill(pid, SIGTERM)` pede educadamente para o processo terminar; o processo pode ignorar esse pedido.
> 

## Sinais gerados pelo hardware

Vale notar que sinais também podem ser gerados automaticamente pelo hardware e pelo SO — não apenas por `kill()`:

| Origem | Sinal gerado |
| --- | --- |
| Divisão por zero | `SIGFPE` (floating point exception) |
| Acesso a endereço inválido | `SIGSEGV` (segmentation fault) |
| Instrução ilegal | `SIGILL` |
| Ctrl+C no terminal | `SIGINT` |
| Processo filho terminou | `SIGCHLD` |

---

# ⏰ time() — Obtendo o Timestamp Unix

`time()` retorna o **tempo atual em segundos desde 1º de janeiro de 1970, 00:00:00 UTC** — o famoso **Unix timestamp** (ou Unix epoch):

```c
time_t t;
seconds = time(&t);
// t e seconds agora contêm o número de segundos desde 01/01/1970
```

## Por que 1970?

A data de 1º de janeiro de 1970 foi escolhida arbitrariamente pelos criadores do UNIX como ponto de referência (chamado de **época Unix** ou *Unix epoch*). É uma convenção histórica que se tornou padrão universal.

## O Problema do Ano 2106 — Bug do Milênio do UNIX

O Tanenbaum faz uma observação importante: em computadores usando plataformas de **32 bits**, o valor máximo que `time()` pode retornar é 2³² − 1 segundos, o que corresponde a um pouco mais de **136 anos** a partir da época Unix:

```
1970 + 136 anos = 2106
```

Isso significa que em **19 de janeiro de 2038**, sistemas UNIX de 32 bits que usam `time_t` como inteiro com sinal de 32 bits entrarão em pane — o contador vai "dar a volta" e o tempo voltará para 1970. Esse problema é chamado de **Y2K38** (o Bug do Milênio do UNIX), semelhante ao famoso bug do ano 2000.

```
Inteiro com sinal de 32 bits:
Valor máximo = 2³¹ - 1 = 2.147.483.647 segundos
              = 19 de janeiro de 2038

Depois disso → overflow → retorna para 13 de dezembro de 1901
```

> 💡 Sistemas de 64 bits não têm esse problema — com 64 bits, `time_t` pode representar datas até o ano **292 bilhões de 2038**, bem além de qualquer preocupação prática. Por isso o Tanenbaum recomenda que se você tem um sistema UNIX de 32 bits, troque-o por um de 64 bits em algum momento antes do ano 2106.
> 

---

# 🪟 1.6.5 — A API do Windows

O UNIX e o Windows diferem de uma maneira fundamental na forma como os programas interagem com o SO.

## O Modelo UNIX

Um programa UNIX consiste em um código que faz uma coisa ou outra e **realiza chamadas de sistema** para ter determinados serviços executados. Há uma relação quase de um para um entre as chamadas de sistema e os procedimentos de biblioteca — para cada chamada de sistema, há aproximadamente um procedimento de biblioteca para invocá-la.

## O Modelo Windows — Orientado a Eventos

Em comparação, um programa Windows é normalmente **direcionado por eventos**. O programa principal espera algum evento acontecer, então chama uma rotina para lidar com ele. Eventos típicos são:

- Teclas sendo pressionadas
- O mouse sendo movido
- Um botão do mouse acionado
- Uma unidade USB sendo inserida ou removida

**Tratadores** (*handlers*) são, então, chamados para processar o evento e atualizar a tela e o estado do programa interno. Isso leva a um estilo de programação diferente do UNIX.

## Win32 / Win64 API

A Microsoft definiu um conjunto de rotinas chamadas de **WinAPI**, **API Win32** ou **Win64 API** (*application programming interface*) que os programadores usam para acessar os serviços do SO. Essa interface tem suporte parcial de todas as versões do Windows desde o Windows 95.

> 💡 O número de chamadas da API Win32 é extremamente grande, chegando a **milhares**. Além disso, enquanto muitas delas são chamadas de sistema, um número substancial é executado inteiramente no espaço do usuário. Com o Windows, é impossível distinguir o que é uma chamada de sistema (realizada pelo núcleo) e o que é apenas uma chamada de biblioteca do espaço do usuário.
> 

Ao desacoplar a interface API das chamadas de sistema reais, a Microsoft retém a capacidade de mudar as chamadas de sistema reais a qualquer tempo sem invalidar os programas existentes — o que é uma vantagem estratégica significativa.

---

# ✅ Resumo do Conceito

- `chdir()` — **muda o diretório de trabalho** do processo. Caminhos relativos passam a ser relativos ao novo diretório. Cada processo tem o seu próprio, independente dos demais
- `chmod()` — **altera os bits de permissão** de um arquivo (owner, group, others × r, w, x). Apenas o dono ou root pode alterar
- `kill()` — **envia um sinal** para um processo. O nome é enganoso — não necessariamente mata o processo, apenas envia uma notificação. `SIGKILL` (9) é o único que não pode ser ignorado
- Sinais também são gerados automaticamente pelo hardware (segfault, divisão por zero) e pelo SO (Ctrl+C → SIGINT)
- `time()` — retorna o **Unix timestamp** — segundos desde 01/01/1970. Em sistemas de 32 bits com sinal, há o problema do **Y2K38** em 2038. Sistemas de 64 bits não têm esse problema
- O **Windows** usa um modelo diferente do UNIX — orientado a eventos com a **WinAPI** (milhares de chamadas), onde é difícil distinguir chamadas de sistema de rotinas de biblioteca