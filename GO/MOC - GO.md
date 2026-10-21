---
tags:
  - go
  - moc
---

# GO — Mapa de Conteúdo

> **Fontes:** [Documentação oficial de Go](https://go.dev/doc/), [especificação](https://go.dev/ref/spec), [modelo de memória](https://go.dev/ref/mem), notas de versão e o código-fonte do runtime.
> **Versão de referência:** Go 1.27 (agosto/2026). Os exemplos com "saída real" foram executados nessa versão.
> **Status:** trilha completa ✅ — 54 notas em 12 módulos

---

## 🧭 Como usar esta trilha

As notas formam uma **sequência**: cada uma tem pré-requisitos no topo, uma seção "De onde viemos" que retoma as anteriores, e navegação **⬅️ Anterior / Próxima ➡️** no fim. Os módulos 01–08 cobrem a linguagem; 09–10, concorrência e rede; 11, as ferramentas; 12, o funcionamento interno do runtime.

Toda nota tem uma seção **🔗 Conexões** ligando o assunto a [[MOC - Sistemas Operacionais|Sistemas Operacionais]] (Tanenbaum), [[MOC - Redes|Redes]], [[MOC - Nginx|Nginx]], [[MOC - Docker|Docker]] e [[MOC - Algoritmos e Estruturas de Dados|Algoritmos e ED]].

---

## 01 · Introdução

| # | Nota | Em uma linha |
| --- | --- | --- |
| 1 | [[O que é Go]] | Origem, filosofia, onde Go brilha e onde não |
| 2 | [[Como Go Funciona]] | Compilador, binário, runtime como "mini SO" |
| 3 | [[Ambiente e Primeiro Programa]] | Instalação, `go.mod`, `go run`/`build`, primeiros erros |
| 4 | [[Convenções da linguagem]] | `gofmt`, visibilidade, nomes, ferramentas, provérbios |

## 02 · Fundamentos

| # | Nota | Em uma linha |
| --- | --- | --- |
| 5 | [[Sintaxe, Blocos e Escopo]] | Tokens, ponto e vírgula implícito, blocos, sombreamento |
| 6 | [[Variáveis e constantes]] | Declaração, zero value, `iota`, constantes sem tipo |
| 7 | [[Tipos de Dados]] | Inteiros, floats, `rune`, `byte`, conversões |
| 8 | [[Operações Aritméticas]] | Overflow, divisão inteira, ponto flutuante, `math/big` |
| 9 | [[Operadores de Comparação e Lógicos]] | Comparabilidade, curto-circuito, bits |
| 10 | [[Controle de Fluxo]] | `if`, `for`/`range`, `switch`, rótulos |
| 11 | [[Ponteiros]] | Endereços, `new`, stack vs heap, o que o GC segue |

## 03 · Tipos Compostos

| # | Nota | Em uma linha |
| --- | --- | --- |
| 12 | [[Strings em Profundidade]] | Bytes vs runas, UTF-8, imutabilidade, `strings.Builder` |
| 13 | [[Arrays e Slices]] | Cabeçalho do slice, `append`, capacidade, aliasing |
| 14 | [[Maps]] | Hash tables (Swiss Tables), ordem, concorrência |
| 15 | [[Structs]] | Campos, literais, tags, alinhamento e padding |

## 04 · Funções

| # | Nota | Em uma linha |
| --- | --- | --- |
| 16 | [[Funções — Fundamentos]] | Parâmetros por valor, funções como valores, inlining |
| 17 | [[Múltiplos Retornos e Erros]] | `(valor, error)`, resultados nomeados, vírgula ok |
| 18 | [[Funções Variádicas]] | `...T`, repasse de slices |
| 19 | [[Closures e Funções Anônimas]] | Captura de variáveis, geradores, armadilhas |
| 20 | [[Defer, Panic e Recover]] | Limpeza garantida, panics, recuperação na fronteira |

## 05 · Métodos, Interfaces e Generics

| # | Nota | Em uma linha |
| --- | --- | --- |
| 21 | [[Métodos e Receptores]] | Receptor valor vs ponteiro, conjuntos de métodos |
| 22 | [[Interfaces]] | Satisfação implícita, `any`, itab, interface nil |
| 23 | [[Composição vs Herança]] | Embutir tipos, promoção de métodos |
| 24 | [[Type Assertion e Type Switch]] | Descobrir o tipo concreto em execução |
| 25 | [[Generics]] | Parâmetros de tipo, restrições, quando usar |
| 26 | [[Iteradores]] | `iter.Seq`, `range` sobre funções, `yield` |

## 06 · Tratamento de Erros

| # | Nota | Em uma linha |
| --- | --- | --- |
| 27 | [[O Padrão error em Go]] | Erros como valores, sentinelas, boas mensagens |
| 28 | [[Criando Erros Customizados]] | Tipos de erro com contexto |
| 29 | [[Wrapping e Unwrapping de Erros]] | `%w`, `errors.Is`/`As`, `errors.Join` |

## 07 · Pacotes e Módulos

| # | Nota | Em uma linha |
| --- | --- | --- |
| 30 | [[Criando Pacotes]] | Exportação, `internal/`, `init`, `//go:embed` |
| 31 | [[Sistema de Módulos (go mod)]] | `go.mod`, `go.sum`, MVS, proxy, workspaces |
| 32 | [[Pacotes Importantes da Stdlib]] | O mapa da biblioteca padrão |

## 08 · IO e Serialização

| # | Nota | Em uma linha |
| --- | --- | --- |
| 33 | [[io.Reader e io.Writer]] | As duas interfaces de todo I/O, composição de streams |
| 34 | [[Leitura e Escrita de Arquivos]] | `os`, `bufio`, `fs.FS`, syscalls de arquivo |
| 35 | [[JSON (marshal e unmarshal)]] | Tags, streaming, `json/v2` |

## 09 · Concorrência

| # | Nota | Em uma linha |
| --- | --- | --- |
| 36 | [[Fundamentos de Concorrência]] | Concorrência vs paralelismo, CSP, data races, `-race`, happens-before |
| 37 | [[Goroutines]] | `go`, ciclo de vida, custo real, stack que cresce, vazamentos (`goroutineleak`) |
| 38 | [[Channels]] | Com e sem buffer, `close`, direção, posse, `hchan` |
| 39 | [[Select]] | Esperar vários channels, timeouts, channel nil, `selectgo` |
| 40 | [[sync.WaitGroup e sync.Mutex]] | O pacote `sync` inteiro e `sync/atomic`; deadlocks |
| 41 | [[context.Context]] | Cancelamento em árvore, prazos, causas, sinais |
| 42 | [[Padrões de Concorrência]] | Pipeline, worker pool, `errgroup`, `singleflight`, rate limiting, retry |

## 10 · Rede e Web

| # | Nota | Em uma linha |
| --- | --- | --- |
| 43 | [[Sockets e o Pacote net]] | TCP/UDP/Unix, enquadramento, deadlines, erros, netpoller com `strace` |
| 44 | [[HTTP (net-http)]] | `ServeMux`, middlewares, servidor de produção, desligamento gracioso, cliente |

## 11 · Tooling e Testes

| # | Nota | Em uma linha |
| --- | --- | --- |
| 45 | [[Testes com go test]] | Tabelas, exemplos, fuzzing, dublês, `httptest`, `synctest` |
| 46 | [[Benchmarks]] | `b.Loop`, alocações, `benchstat`, efeitos de cache |
| 47 | [[Profiling e Diagnóstico]] | pprof, `/debug/pprof`, traces, flight recorder, PGO |
| 48 | [[Build, Flags e Cross-compilation]] | Compilação cruzada, versões no binário, build tags, cgo, Docker |

## 12 · Go por Dentro

| # | Nota | Em uma linha |
| --- | --- | --- |
| 49 | [[Memória e Escape Analysis]] | Stack vs heap, escape analysis, o alocador, RSS |
| 50 | [[Garbage Collector]] | Tricolor concorrente, `GOGC`/`GOMEMLIMIT`, Green Tea, ponteiros fracos |
| 51 | [[Scheduler do Go]] | G-M-P, roubo de trabalho, syscalls, preempção, `sysmon` |
| 52 | [[Modelo de Memória]] | DRF-SC, happens-before formal, reordenamento real no x86 |
| 53 | [[Reflection]] | `reflect.Type`/`Value`, tags, custo, descritores de tipo |
| 54 | [[unsafe e cgo]] | Layout, `unsafe.Pointer`, chamar C, regras de ponteiros, custo |

---

## 🗺️ Por tema (atalhos)

| Quero entender... | Comece por |
| --- | --- |
| Como Go se relaciona com o SO | [[Como Go Funciona]] → [[Scheduler do Go]] → [[Memória e Escape Analysis]] |
| Concorrência do zero | [[Fundamentos de Concorrência]] → [[Goroutines]] → [[Channels]] |
| Escrever um serviço web | [[HTTP (net-http)]] → [[context.Context]] → [[Build, Flags e Cross-compilation]] |
| Deixar um programa mais rápido | [[Profiling e Diagnóstico]] → [[Benchmarks]] → [[Memória e Escape Analysis]] |
| Por que meu programa trava ou vaza | [[Goroutines]] (vazamentos) → [[sync.WaitGroup e sync.Mutex]] (deadlocks) → [[Profiling e Diagnóstico]] |
| Memória em containers | [[Garbage Collector]] (`GOMEMLIMIT`) → [[Goroutines]] (`GOMAXPROCS`) |

---

## 🔗 Pontes com as outras áreas

| Área | Notas de Go mais ligadas |
| --- | --- |
| [[MOC - Sistemas Operacionais\|Sistemas Operacionais]] — processos, threads, sincronização | [[Goroutines]], [[Scheduler do Go]], [[sync.WaitGroup e sync.Mutex]], [[Channels]], [[Modelo de Memória]] |
| [[MOC - Sistemas Operacionais\|Sistemas Operacionais]] — memória | [[Memória e Escape Analysis]], [[Garbage Collector]], [[Ponteiros]] |
| [[MOC - Sistemas Operacionais\|Sistemas Operacionais]] — syscalls e arquivos | [[Como Go Funciona]], [[Leitura e Escrita de Arquivos]], [[Sockets e o Pacote net]] |
| [[MOC - Redes\|Redes]] | [[Sockets e o Pacote net]], [[HTTP (net-http)]] |
| [[MOC - Nginx\|Nginx]] | [[HTTP (net-http)]], [[Sockets e o Pacote net]], [[Padrões de Concorrência]] |
| [[MOC - Docker\|Docker]] | [[Build, Flags e Cross-compilation]], [[context.Context]], [[Garbage Collector]], [[Goroutines]] |
| [[MOC - Algoritmos e Estruturas de Dados\|Algoritmos e ED]] | [[Maps]], [[Arrays e Slices]], [[Benchmarks]], [[Channels]] |
