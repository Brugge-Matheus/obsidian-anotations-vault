---
tags:
  - go
  - go/introducao
aliases:
  - Compilação e Runtime do Go
  - Runtime do Go
  - Como Go funciona por dentro
ordem: 2
modulo: "01 - Introdução"
---
# Como Go Funciona

> **Nota 2 da trilha de Go.** Agora que sabemos *por que* Go existe, vamos entender *como* um arquivo `.go` vira um programa rodando. Esta nota apresenta as duas peças que acompanham todo programa Go: o **compilador** (que transforma o código em binário nativo) e o **runtime** (um "mini sistema operacional" embutido no binário que gerencia goroutines, memória e GC).
>
> **Pré-requisitos:** [[O que é Go]].

---

## 🧭 De onde viemos

Em [[O que é Go]] vimos que Go é **compilada**, tem **garbage collector** e **concorrência embutida**. Isso levanta perguntas:

- Se Go compila para código nativo como C, **quem roda o garbage collector**? Não existe uma JVM.
- Se goroutines não são threads do sistema operacional, **quem escalona as goroutines**?
- Por que um "Olá, mundo" em Go tem **2 MB**, enquanto em C tem 16 KB?

A resposta para as três é a mesma: **o runtime do Go é compilado junto com o seu programa, dentro do mesmo binário.**

---

## 🗺️ 1. Os três modelos de execução

Existem basicamente três formas de uma linguagem rodar código:

```
1) INTERPRETADA  (PHP, Ruby, Python)
   ┌──────────┐     ┌─────────────────┐     ┌─────┐
   │ script.rb│ ──▶ │  interpretador  │ ──▶ │ CPU │
   └──────────┘     │ (lê e executa   │     └─────┘
                    │  em tempo real) │
                    └─────────────────┘
   ✔ sem etapa de build   ✘ mais lenta   ✘ precisa do interpretador instalado

2) MÁQUINA VIRTUAL  (Java, C#, Kotlin)
   ┌──────────┐  javac  ┌──────────┐     ┌─────────┐     ┌─────┐
   │ App.java │ ──────▶ │ bytecode │ ──▶ │   JVM   │ ──▶ │ CPU │
   └──────────┘         │ (.class) │     │ JIT+GC  │     └─────┘
                        └──────────┘     └─────────┘
   ✔ portável   ✔ JIT otimiza em execução   ✘ precisa da JVM instalada

3) NATIVA  (C, C++, Rust, Go)
   ┌──────────┐ compilador ┌──────────────────┐     ┌─────┐
   │ main.go  │ ─────────▶ │ código de máquina│ ──▶ │ CPU │
   └──────────┘            │   (binário ELF)  │     └─────┘
                           └──────────────────┘
   ✔ mais rápida para iniciar   ✔ sem dependência externa   ✘ um binário por plataforma
```

Go está no grupo **3**, mas com uma diferença em relação ao C: o binário carrega dentro de si um **runtime** com GC e escalonador. É como se cada programa Go levasse junto uma "JVM bem pequena", só que compilada para código nativo e sem interpretar nada.

> 💡 **Runtime:** conjunto de código que dá suporte à execução de um programa enquanto ele roda. Em C o runtime é mínimo (basicamente a inicialização da libc e o `crt0`). Em Go o runtime é grande: inclui o escalonador de goroutines, o alocador de memória, o garbage collector, o gerenciamento de pilhas e o *netpoller* de rede.

> 💡 **JIT (Just-In-Time):** técnica em que a VM compila trechos de bytecode para código nativo durante a execução, com base no que observa do programa. Go **não** usa JIT: tudo é compilado antes (*ahead-of-time*, AOT).

---

## 🏭 2. O compilador — do `.go` ao binário

Quando você roda `go build`, acontece um pipeline parecido com o de C (que você viu em [[Arquivos de Cabeçalho e o Modelo de Execução]]), mas com algumas diferenças importantes:

```
  main.go  util.go  (todos os arquivos .go de um pacote)
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│ 1. ANÁLISE LÉXICA E SINTÁTICA (parsing)                 │
│    texto → tokens → AST (árvore de sintaxe abstrata)    │
├─────────────────────────────────────────────────────────┤
│ 2. VERIFICAÇÃO DE TIPOS (type checking)                 │
│    cada expressão tem um tipo? conversões são válidas?  │
│    variáveis/imports não usados? → ERRO aqui            │
├─────────────────────────────────────────────────────────┤
│ 3. IR + OTIMIZAÇÕES DE ALTO NÍVEL                       │
│    inlining (copiar funções pequenas para o chamador)   │
│    escape analysis (decidir stack vs heap)              │
│    devirtualização (trocar chamada via interface por    │
│    chamada direta quando o tipo é conhecido)            │
├─────────────────────────────────────────────────────────┤
│ 4. SSA (Static Single Assignment)                       │
│    eliminação de código morto, propagação de constantes,│
│    remoção de checagens de limite desnecessárias,       │
│    x*8 → x<<3 ...                                       │
├─────────────────────────────────────────────────────────┤
│ 5. GERAÇÃO DE CÓDIGO DE MÁQUINA                         │
│    backend por arquitetura: amd64, arm64, riscv64, wasm │
└─────────────────────────────────────────────────────────┘
     │  um arquivo objeto por PACOTE (não por arquivo .go)
     ▼
┌─────────────────────────────────────────────────────────┐
│ 6. LINKER                                               │
│    junta: seu código + pacotes importados + RUNTIME     │
│    remove funções nunca chamadas (dead code elimination)│
└─────────────────────────────────────────────────────────┘
     │
     ▼
  binário executável (ELF no Linux, Mach-O no macOS, PE no Windows)
```

> 💡 **AST (Abstract Syntax Tree):** representação em árvore da estrutura do código. `a + b * c` vira um nó `+` com filhos `a` e um nó `*` (que tem filhos `b` e `c`). Ferramentas como `gofmt` e `go vet` trabalham sobre a AST.

> 💡 **SSA (Static Single Assignment):** forma intermediária em que cada variável recebe valor **uma única vez**. Isso facilita muito as otimizações, porque o compilador sabe exatamente de onde veio cada valor.

> 💡 **Inlining:** substituir a chamada de uma função pequena pelo corpo dela, eliminando o custo do `CALL`/`RET`. Veremos em [[Funções — Fundamentos]].

> 💡 **Escape analysis:** análise que decide se uma variável pode morar na **stack** (barata, liberada ao sair da função) ou precisa ir para a **heap** (gerenciada pelo GC). Veremos em [[Memória e Escape Analysis]].

### Por que o build de Go é tão rápido?

Essa foi uma das motivações de criar Go. As decisões que garantem isso:

| Decisão de design | Efeito |
| --- | --- |
| **Sem `#include`** de headers textuais | Em C/C++, cada `#include` reprocessa o header inteiro; um projeto grande reprocessa os mesmos headers milhares de vezes. Em Go, o compilador lê a **interface já compilada** do pacote importado |
| **Imports não usados são erro** | O compilador nunca processa dependência desnecessária |
| **Sem dependências circulares entre pacotes** | O grafo de pacotes é acíclico, então pacotes podem ser compilados em paralelo e em ordem |
| **Gramática simples** | O parsing é rápido e não precisa de informação de tipos (diferente de C++) |
| **Cache de build** | Pacotes que não mudaram não são recompilados (`go env GOCACHE`) |

### Vendo o compilador trabalhar

```bash
go build -x .                 # mostra cada comando executado (compile, link...)
go build -gcflags='-m' .      # mostra decisões de inlining e escape analysis
go build -gcflags='-S' .      # mostra o assembly gerado
go tool objdump -s main.main ./hello   # desmonta a função main do binário
```

---

## 🧰 3. O que existe dentro de um binário Go

Vamos compilar o programa mais simples possível e inspecionar o resultado (saída real, Linux amd64):

```go
package main

import "fmt"

func main() {
	fmt.Println("Olá, Go!")
}
```

```bash
$ go build -o hello .
$ ls -la hello
-rwxr-xr-x 1 root root 2209199 hello           # ~2,2 MB

$ file hello
hello: ELF 64-bit LSB executable, x86-64, statically linked, ...

$ ldd hello
	not a dynamic executable                     # não depende de nenhuma .so

$ go build -ldflags="-s -w" -o hello2 .        # remove tabela de símbolos e DWARF
$ ls -la hello2
-rwxr-xr-x 1 root root 1437880 hello2          # ~1,4 MB
```

> 💡 **Estaticamente linkado (statically linked):** todo o código necessário está dentro do próprio executável. Não há dependência de bibliotecas compartilhadas (`.so`) como a `libc.so`. Compare com [[Sistemas Monolíticos]], onde você viu bibliotecas compartilhadas e DLLs.

Por que 2 MB? Porque o binário contém:

```
┌───────────────────────────── hello (ELF) ─────────────────────────────┐
│ .text        código de máquina: main.main + fmt + os + RUNTIME        │
│              (scheduler, GC, alocador, stacks, netpoller, sinais...)  │
│ .rodata      dados somente leitura: a string "Olá, Go!", tabelas      │
│ .typelink    metadados de tipos (usados por interfaces e reflection)  │
│ .itablink    tabelas de interface pré-calculadas                      │
│ .gopclntab   tabela PC → linha/função (stack traces legíveis no panic)│
│ .go.buildinfo versão do Go e módulos usados (go version -m hello)     │
│ .noptrdata   variáveis globais SEM ponteiros (o GC não precisa olhar) │
│ .data        variáveis globais COM ponteiros (o GC escaneia)          │
│ .bss/.noptrbss variáveis globais zeradas                              │
│ .debug_*     informação DWARF para depuradores (removida com -w)      │
│ .symtab      tabela de símbolos (removida com -s)                     │
└───────────────────────────────────────────────────────────────────────┘
```

Repare que Go separa as variáveis globais **com** e **sem** ponteiros (`.data` vs `.noptrdata`). Isso existe por causa do GC: ele só precisa escanear regiões que podem conter ponteiros. É um exemplo de como o runtime influencia até o layout do executável. Os segmentos `.text`, `.data` e `.bss` são os mesmos que você estudou em [[Arquivos de Cabeçalho e o Modelo de Execução]].

> 💡 Dá para descobrir com qual versão de Go e com quais dependências um binário foi compilado: `go version -m ./hello`. Isso é muito usado para auditoria de segurança (`govulncheck` faz isso).

---

## 🫀 4. O runtime — o "mini sistema operacional" do Go

O runtime é a parte mais interessante para quem estuda SO, porque ele **reimplementa em espaço de usuário** várias ideias que o Tanenbaum descreve para o kernel:

```
┌──────────────────────── PROCESSO GO (espaço de usuário) ───────────────────────┐
│                                                                                │
│   seu código:   main.main()   handler()   worker()   worker()   ...            │
│                     │            │           │          │                      │
│                     ▼            ▼           ▼          ▼                      │
│   ┌────────────────────────── RUNTIME DO GO ─────────────────────────────┐     │
│   │  ESCALONADOR (G-M-P)     milhares de goroutines (G) multiplexadas    │     │
│   │                          sobre poucas threads do SO (M), limitadas   │     │
│   │                          por GOMAXPROCS processadores lógicos (P)    │     │
│   │  ALOCADOR DE MEMÓRIA     pede memória grande ao SO (mmap) e divide   │     │
│   │                          em pedaços pequenos por tamanho             │     │
│   │  GARBAGE COLLECTOR       mark & sweep concorrente (Green Tea)        │     │
│   │  PILHAS DINÂMICAS        cada goroutine começa com ~2 KB de stack    │     │
│   │                          que cresce e encolhe conforme a necessidade │     │
│   │  NETPOLLER               epoll (Linux) / kqueue (macOS) para I/O de  │     │
│   │                          rede sem bloquear threads                   │     │
│   │  SYSMON                  thread de monitoramento: preempção, GC,     │     │
│   │                          retomada de syscalls bloqueadas             │     │
│   └──────────────────────────────────────────────────────────────────────┘     │
│                     │                 │                  │                     │
└─────────────────────┼─────────────────┼──────────────────┼─────────────────────┘
                      ▼                 ▼                  ▼
              ┌──────────────── KERNEL (Linux) ─────────────────┐
              │  threads (clone), memória (mmap), epoll, futex,  │
              │  sinais (SIGURG para preempção), read/write ...  │
              └──────────────────────────────────────────────────┘
```

Fazendo o paralelo com o que você já estudou:

| Conceito no SO (Tanenbaum) | Equivalente no runtime do Go | Onde estudar em Go |
| --- | --- | --- |
| Processo / thread | Goroutine | [[Goroutines]] |
| Escalonador do kernel | Escalonador G-M-P em espaço de usuário | [[Scheduler do Go]] |
| Threads de usuário sobre threads de kernel | Modelo M:N | [[Implementações Híbridas]] |
| Gerenciamento de memória livre | Alocador com classes de tamanho (inspirado no TCMalloc) | [[Memória e Escape Analysis]] |
| Pilha do processo | Pilha de cada goroutine, que cresce dinamicamente | [[Goroutines]] |
| Preempção por interrupção de relógio | Preempção via sinal `SIGURG` enviado pelo `sysmon` | [[Scheduler do Go]] |
| I/O multiplexado | Netpoller (epoll/kqueue) | [[Sockets e o Pacote net]] |
| Troca de mensagens | Channels | [[Channels]] |

### O runtime em números: o que um "Olá, mundo" faz no kernel

Rodando o binário acima com `strace -f -c ./hello` (saída real, resumida):

```
syscall          chamadas   para quê
──────────────────────────────────────────────────────────────────────
execve                1     o shell carrega o binário (ver Criação de Processos)
arch_prctl            1     configura o registrador de TLS (dados por thread)
mmap                 22     o alocador reserva regiões de memória (arenas, stacks)
rt_sigaction        114     o runtime instala handlers para todos os sinais
sigaltstack           8     pilhas alternativas para tratar sinais
clone                 3     cria threads do SO (sysmon, workers do GC, ...)
futex                 8     sincronização entre as threads do runtime
sched_getaffinity     1     descobre quantas CPUs pode usar (GOMAXPROCS)
write                 1     ← o único syscall que o SEU código pediu: imprimir
(+ openat, read, fcntl, nanosleep, madvise... e o exit_group final)
──────────────────────────────────────────────────────────────────────
total               ~200    para imprimir uma linha!
```

Isso mostra bem o papel do runtime: **seu programa pediu um `write`**; o resto (cerca de 200 syscalls) é o runtime se preparando: reservando memória, criando threads, instalando tratadores de sinais. Em C, o mesmo programa faz em torno de 30 syscalls. É o preço (pequeno, uma única vez na inicialização) de ter GC e goroutines.

> 💡 **`futex` (fast userspace mutex):** syscall do Linux usada para implementar locks. A ideia é resolver a disputa em espaço de usuário (com operações atômicas) e só chamar o kernel quando uma thread precisa dormir. É como o runtime de Go implementa a espera de threads. Conecta com [[Mutexes]] e [[Dormir e Despertar]].

---

## 🚀 5. A vida de um programa Go: do `execve` ao `exit`

O que acontece entre o shell executar `./hello` e o processo terminar:

```
 shell: execve("./hello")             ← kernel cria o processo (ver Criação de Processos)
   │
   ▼
 _rt0_amd64_linux                     ← ponto de entrada do ELF (assembly do runtime),
   │                                     NÃO é o seu main!
   ▼
 runtime·rt0_go
   ├─ runtime.args()                  lê argc/argv/envp e o vetor auxiliar do kernel
   ├─ runtime.osinit()                descobre nº de CPUs, tamanho de página
   ├─ runtime.schedinit()             inicializa alocador, GC, GOMAXPROCS, cria os Ps
   ├─ newproc(runtime.main)           cria a PRIMEIRA goroutine
   └─ mstart()                        a thread principal (M0) começa a escalonar
         │
         ▼
 runtime.main  (roda na goroutine principal)
   ├─ inicia o sysmon (thread de monitoramento)
   ├─ inicializa os pacotes do runtime
   ├─ habilita o GC
   ├─ executa as funções init() de TODOS os pacotes,
   │  em ordem de dependência (ver Criando Pacotes)
   ├─ chama  main.main()   ◀───────── finalmente, o SEU código
   └─ quando main.main retorna → exit(0)
                                   │
                                   ▼
                    o processo termina. Goroutines que ainda
                    estavam rodando são simplesmente encerradas,
                    sem aviso e sem rodar seus defers.
```

Duas consequências práticas importantes, que vão aparecer muitas vezes na trilha:

1. **`main` retornar encerra o programa inteiro**, mesmo que existam outras goroutines trabalhando. Por isso precisamos de mecanismos de espera como `sync.WaitGroup` (ver [[sync.WaitGroup e sync.Mutex]]).
2. **`init()` roda antes de `main()`**, uma vez por pacote. Veremos em [[Criando Pacotes]].

> 💡 **Vetor auxiliar (auxv):** informações que o kernel coloca na pilha do processo junto com `argv` e `envp` quando executa um binário, como o tamanho de página e o endereço do vDSO. O runtime de Go lê isso logo no início.

---

## 🌐 6. Um código, várias plataformas: GOOS e GOARCH

Como o compilador gera código nativo, um binário só roda na plataforma para a qual foi compilado. Mas trocar de plataforma é só mudar duas variáveis de ambiente:

```bash
go env GOOS GOARCH          # plataforma atual, ex.: linux amd64

GOOS=linux   GOARCH=arm64 go build -o app-linux-arm64      # Raspberry Pi, AWS Graviton
GOOS=darwin  GOARCH=arm64 go build -o app-mac              # Mac com Apple Silicon
GOOS=windows GOARCH=amd64 go build -o app.exe              # Windows

go tool dist list           # lista todas as combinações suportadas
```

> 💡 **GOOS / GOARCH:** sistema operacional e arquitetura de CPU de destino. O compilador troca o backend de geração de código (instruções x86 vs ARM) e o runtime troca a camada de syscalls (Linux usa `epoll`, macOS usa `kqueue`, Windows usa IOCP).

Isso é possível porque o runtime de Go fala **diretamente com o kernel** através de syscalls (no Linux, sem passar pela libc). Veremos os detalhes em [[Build, Flags e Cross-compilation]].

---

## ⚖️ 7. Go vs C vs Java por dentro

| Aspecto | C | Java | Go |
| --- | --- | --- | --- |
| Formato executado | Código nativo | Bytecode | Código nativo |
| Precisa de algo instalado? | libc (normalmente já existe) | JVM | **Nada** |
| Tamanho do "hello world" | ~16 KB (dinâmico) | ~1 KB `.class` + JVM de ~100 MB | ~2 MB (tudo incluído) |
| Tempo de inicialização | ~1 ms | dezenas a centenas de ms (aquecer a JVM) | ~1 ms |
| GC | Não tem | Sim, vários algoritmos configuráveis | Sim, um único algoritmo com poucos ajustes |
| Escalonamento de tarefas | Kernel (pthreads 1:1) | Kernel (e virtual threads M:N desde Java 21) | Runtime do Go (M:N) |
| Otimização | AOT (GCC/Clang, muito agressivos) | JIT em execução | AOT (mais conservador, priorizando build rápido) |

> 💡 O compilador de Go otimiza **menos** que GCC/Clang de propósito: a prioridade é compilar rápido. Desde o Go 1.21 existe **PGO (Profile-Guided Optimization)**: você coleta um perfil de produção (`default.pgo`) e o compilador usa para decidir o que otimizar mais, como inlining em funções quentes. Veremos em [[Profiling e Diagnóstico]].

---

## ⚠️ 8. Equívocos comuns

| Equívoco | Realidade |
| --- | --- |
| "Go roda numa VM como Java" | Não. Go é compilado para código nativo; o runtime é código nativo linkado ao programa |
| "Goroutine é uma thread" | Não. Goroutine é uma tarefa de usuário; várias goroutines rodam sobre uma mesma thread do SO |
| "Go é interpretado porque tem `go run`" | `go run` apenas compila para um diretório temporário e executa o binário |
| "Binário estático roda em qualquer lugar" | Roda em qualquer máquina **da mesma plataforma** (GOOS/GOARCH). E se o programa usar cgo, pode voltar a depender da libc |
| "O GC deixa Go lento" | O GC tem custo, mas é concorrente e com pausas de microssegundos; para a maioria dos servidores, não é o gargalo |

---

## 📋 Resumo

| Conceito | Ideia central |
| --- | --- |
| Modelo de execução | Compilação nativa *ahead-of-time*, sem VM e sem JIT |
| Compilador | parse → type check → IR (inlining, escape analysis) → SSA → código de máquina → linker |
| Build rápido | Sem headers textuais, sem imports não usados, sem ciclos, cache de build |
| Binário | Estático, ~2 MB, contém seu código + dependências + runtime |
| Runtime | Escalonador M:N, alocador, GC, pilhas dinâmicas, netpoller, sysmon |
| Inicialização | `_rt0` → `schedinit` → `runtime.main` → `init()`s → `main.main` |
| Fim do programa | Quando `main.main` retorna, o processo termina, com goroutines ou não |
| Multiplataforma | `GOOS` + `GOARCH` |

---

## 🔗 Conexões

**Dentro de Go**
- Instalando e rodando seu primeiro programa → [[Ambiente e Primeiro Programa]]
- Escalonador em profundidade → [[Scheduler do Go]]
- Stack, heap e o alocador → [[Memória e Escape Analysis]]
- Garbage collector → [[Garbage Collector]]
- `init()` e ordem de inicialização → [[Criando Pacotes]]
- Flags de compilação, cross-compilation e cgo → [[Build, Flags e Cross-compilation]]

**Sistemas Operacionais**
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — o pipeline de C (pré-processador → compilador → linker) e os segmentos `.text/.data/.bss`; Go dispensa o pré-processador e os headers
- [[Criação de Processos]] e [[Syscalls para Gerenciamento de Processos]] — o `execve` que carrega o binário Go
- [[Implementações Híbridas]] e [[Implementando Threads em User Space]] — o modelo M:N do escalonador do Go
- [[Memória Virtual]] e [[Paginação]] — o alocador do Go reserva grandes regiões virtuais com `mmap` e o kernel só entrega páginas físicas quando são tocadas
- [[Mutexes]] e [[Dormir e Despertar]] — o `futex` usado pelo runtime para fazer threads dormirem
- [[Paravirtualização e a Máquina Virtual Java]] — o contraste com o modelo de VM

**Docker**
- [[Imagens]] e [[Dockerfile]] — um binário estático permite imagem `FROM scratch` com só o executável

---

⬅️ **Anterior:** [[O que é Go]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Ambiente e Primeiro Programa]] ➡️
