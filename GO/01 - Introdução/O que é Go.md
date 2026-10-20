---
tags:
  - go
  - go/introducao
aliases:
  - Golang
  - Linguagem Go
  - Introdução ao Go
ordem: 1
modulo: "01 - Introdução"
---
# O que é Go

> **Nota 1 da trilha de Go.** Antes de escrever uma linha de código, vale entender **por que Go existe**, que problemas ele tenta resolver e quais escolhas de design moldam tudo o que vem depois. Quase toda "regra estranha" de Go (sem classes, sem exceções, erros como valores, variável não usada é erro de compilação) é consequência direta das ideias desta nota.
>
> **Pré-requisitos:** nenhum.
> **Versão de referência:** Go 1.27 (agosto/2026).

---

## 🧭 De onde viemos — o problema que Go resolve

Em **2007**, dentro do Google, três engenheiros estavam esperando um build de C++ terminar. A compilação de um único binário do Google levava **dezenas de minutos**, às vezes mais de uma hora. Esses três eram:

| Pessoa | Por que importa |
| --- | --- |
| **Robert Griesemer** | Trabalhou na VM V8 (JavaScript) e na Java HotSpot VM |
| **Rob Pike** | Co-criador do UTF-8, do Plan 9 e da linguagem Limbo (com concorrência no estilo CSP) |
| **Ken Thompson** | Co-criador do **UNIX** e da linguagem **B** (avó do C), prêmio Turing |

O cenário que eles observavam era este:

```
Google, ~2007
──────────────────────────────────────────────────────────────
  Hardware mudou:      CPUs multi-core se tornando padrão
                       → linguagens antigas não foram feitas para isso
  Software mudou:      sistemas enormes, em rede, com milhares
                       de engenheiros mexendo no mesmo código
  Linguagens da época:
    C / C++  → rápidas, mas builds lentíssimos, gerenciamento
               manual de memória, complexidade enorme (C++)
    Java     → GC e segurança, mas verbosa, hierarquias de
               classes gigantes, depende de uma VM pesada
    Python   → produtiva, mas lenta e com tipagem dinâmica
               (erros só aparecem em produção)
──────────────────────────────────────────────────────────────
  Ninguém oferecia, ao mesmo tempo:
    ✔ compilação rápida      ✔ execução rápida
    ✔ segurança de memória   ✔ concorrência fácil
    ✔ simplicidade de leitura para times grandes
```

A resposta foi criar uma linguagem nova. O desenho começou em setembro de 2007, foi anunciada publicamente em **10 de novembro de 2009** e chegou à versão estável **Go 1.0 em março de 2012**.

> 💡 **Linguagem de programação de sistemas:** linguagem usada para construir infraestrutura — servidores, ferramentas de linha de comando, bancos de dados, orquestradores, compiladores. O foco é desempenho, previsibilidade e controle, em oposição a linguagens focadas em scripts ou em interfaces gráficas.

---

## 🎯 1. A definição em uma frase

> **Go é uma linguagem compilada, estaticamente tipada, com coleta de lixo (garbage collector) e concorrência embutida na linguagem, projetada para ser simples de ler, rápida de compilar e fácil de manter em times grandes.**

Vamos destrinchar cada palavra:

| Característica | O que significa | Consequência prática |
| --- | --- | --- |
| **Compilada** | O código-fonte vira código de máquina nativo antes de rodar | Executa rápido, sem interpretador nem VM |
| **Estaticamente tipada** | Todo valor tem um tipo conhecido em tempo de compilação | Muitos bugs são pegos antes de rodar |
| **Inferência de tipos** | O compilador deduz o tipo quando é óbvio (`x := 10`) | Segurança de tipos sem verbosidade |
| **Garbage collector** | A memória não usada é liberada automaticamente | Sem `free()` manual, sem *use-after-free* |
| **Concorrência embutida** | `go` e `chan` são palavras-chave da linguagem | Milhares de tarefas concorrentes com pouca memória |
| **Binário estático** | O resultado é um único executável autossuficiente | Deploy = copiar um arquivo |
| **Biblioteca padrão rica** | HTTP, JSON, criptografia, testes já vêm com a linguagem | Dá para fazer muito sem dependências externas |

> 💡 **Tipagem estática vs dinâmica:** na tipagem **estática** (Go, C, Java), o tipo de cada variável é verificado pelo compilador antes do programa rodar. Na tipagem **dinâmica** (PHP, Ruby, Python, JavaScript), o tipo é verificado durante a execução — um erro de tipo só aparece quando aquela linha é executada.

> 💡 **Garbage Collector (GC):** componente do runtime que descobre quais objetos na memória não são mais alcançáveis pelo programa e libera esse espaço automaticamente. Veremos o GC de Go em detalhe em [[Garbage Collector]].

---

## 🧠 2. A filosofia de Go — as ideias que explicam tudo

Go tem uma filosofia muito clara, e entender ela evita frustração ("por que Go não tem X?").

### 2.1 Simplicidade acima de tudo

Go tem apenas **25 palavras-chave**. Para comparação: C tem 32 (no padrão C89), Java tem mais de 50 e C++ passa de 90. A especificação da linguagem inteira pode ser lida em uma tarde.

```
break     default      func    interface  select
case      defer        go      map        struct
chan      else         goto    package    switch
const     fallthrough  if      range      type
continue  for          import  return     var
```

A ideia central é: **código é lido muito mais vezes do que é escrito**. Em um time de mil engenheiros, a clareza vale mais do que a esperteza.

> "Clear is better than clever." — um dos *Go Proverbs* de Rob Pike

### 2.2 Uma única forma de fazer as coisas

- Só existe **um** laço: `for` (não há `while` nem `do-while`)
- Só existe **um** estilo de formatação: o do `gofmt`
- Só existe **uma** regra de visibilidade: letra maiúscula = público

Isso significa que qualquer código Go, de qualquer empresa, **tem a mesma cara**.

### 2.3 Ortogonalidade

> 💡 **Ortogonalidade:** propriedade de um design em que os recursos são independentes e combináveis. Poucos conceitos que se combinam bem, em vez de muitos conceitos especializados.

Em Go, você tem tipos, métodos, interfaces e composição. Essas peças são independentes e se combinam livremente. Não existe uma hierarquia de classes amarrando tudo.

### 2.4 Explícito em vez de mágico

- Conversões de tipo são **sempre explícitas**: `float64(x)`, nunca automáticas
- Erros são **valores retornados**, não exceções que "pulam" pelo código
- Não existe sobrecarga de operadores nem de funções: `a + b` sempre significa a mesma coisa

### 2.5 Composição em vez de herança

Go **não tem classes nem herança**. Em vez de "um Cachorro **é um** Animal", Go pensa em "um Cachorro **tem** um comportamento de andar". O comportamento compartilhado é descrito por **interfaces**, que são satisfeitas implicitamente. Veremos isso em [[Composição vs Herança]] e [[Interfaces]].

### 2.6 Concorrência como cidadã de primeira classe

Inspirado no modelo **CSP (Communicating Sequential Processes)**, de Tony Hoare (1978):

> "Do not communicate by sharing memory; instead, share memory by communicating."
> (Não se comunique compartilhando memória; compartilhe memória se comunicando.)

Em vez de várias threads disputando a mesma variável protegida por locks, Go incentiva tarefas independentes (**goroutines**) que trocam dados por **channels**. Isso é o mesmo princípio de [[Troca de Mensagens]] que você estudou em SO, trazido para dentro da linguagem. Veremos tudo isso no módulo de concorrência, começando em [[Fundamentos de Concorrência]].

---

## 🚫 3. O que Go **não** tem (de propósito)

Muitas ausências de Go são decisões deliberadas, não esquecimentos:

| Recurso ausente | Por que ficou de fora | O que Go usa no lugar |
| --- | --- | --- |
| Classes e herança | Hierarquias rígidas, acoplamento forte, "problema do diamante" | Structs + métodos + interfaces + embedding |
| Exceções (`try/catch`) | Fluxo de controle invisível, difícil de raciocinar | Erros como valores: `(resultado, error)` |
| Sobrecarga de funções/operadores | `a + b` pode significar qualquer coisa | Nomes diferentes para funções diferentes |
| Conversão implícita de tipos | Fonte clássica de bugs em C (`int` → `unsigned`) | Conversão sempre explícita |
| Operador ternário (`a ? b : c`) | Leva a expressões ilegíveis quando aninhado | `if/else` |
| Aritmética de ponteiros | Fonte de falhas de segurança em C | Ponteiros seguros + slices com checagem de limites |
| Variáveis e imports não usados | Código morto e builds lentos | **Erro de compilação** |

> ⚠️ **Generics** ficaram de fora por 10 anos e só chegaram no **Go 1.18 (2022)**, depois de muito debate sobre como adicioná-los sem complicar a linguagem. No Go 1.27 (2026) a linguagem passou a permitir também **métodos genéricos**. Veremos em [[Generics]].

---

## ⚙️ 4. Como Go se compara às linguagens que você já conhece

Você já trabalhou com **PHP, Ruby e Go** e estudou **C** junto com o Tanenbaum. A tabela abaixo situa Go nesse mapa:

| Aspecto | C | Java | PHP / Ruby / Python | **Go** |
| --- | --- | --- | --- | --- |
| Execução | Nativa | Bytecode na JVM | Interpretada (com JIT em alguns casos) | **Nativa** |
| Tipagem | Estática, fraca | Estática | Dinâmica | **Estática, forte, com inferência** |
| Memória | Manual (`malloc`/`free`) | GC | GC / contagem de referências | **GC** |
| Concorrência | Threads do SO (pthreads) | Threads (e virtual threads desde Java 21) | Processos/threads; modelo limitado | **Goroutines (M:N) + channels** |
| Deploy | Binário (depende da libc) | JAR + JVM instalada | Código + interpretador + dependências | **Um binário estático** |
| Tempo de build | Médio (lento em C++) | Médio | Não compila | **Muito rápido** |
| Orientação a objetos | Não tem | Classes e herança | Classes e herança | **Métodos + interfaces, sem herança** |
| Tratamento de erros | Código de retorno + `errno` | Exceções | Exceções | **Valores `error`** |

> 💡 **M:N:** modelo de threads em que **M** tarefas de nível de usuário (goroutines) são multiplexadas sobre **N** threads do sistema operacional. É o modelo híbrido que você viu em [[Implementações Híbridas]]. O escalonador de Go roda em espaço de usuário, como em [[Implementando Threads em User Space]], mas usa várias threads do kernel por baixo.

### Onde Go se encaixa

```
         mais controle / mais rápido
                    ▲
          C, C++,   │
          Rust      │
                    │       ★ Go: perto de C/Java em desempenho,
          Go ───────┼──────    perto de Python em simplicidade
                    │
          Java, C#  │
                    │
   Python, PHP,     │
   Ruby, JS         │
                    └────────────────────────────▶
                     mais simples / mais produtivo
```

Go não tenta ser a linguagem mais rápida (C, C++ e Rust são mais rápidos e dão mais controle) nem a mais expressiva. Ele busca o **melhor equilíbrio** entre desempenho, simplicidade e produtividade para software de servidor e infraestrutura.

---

## 🌍 5. Onde Go é usado

Go virou a **linguagem da infraestrutura cloud**. Várias ferramentas que você já estudou no seu vault são escritas em Go:

| Projeto | O que é | No seu vault |
| --- | --- | --- |
| **Docker** (engine e CLI) | Plataforma de containers | [[Arquitetura]], [[Runtime e Engine]] |
| **containerd** | Runtime de alto nível de containers | [[Container.d]] |
| **runc** | Runtime OCI de baixo nível | [[RunC]], [[OCI]] |
| **Kubernetes** | Orquestração de containers | — |
| **Terraform** | Infraestrutura como código | — |
| **Prometheus** e **Grafana** (backend) | Monitoramento e métricas | — |
| **etcd** | Banco chave-valor distribuído (usado pelo Kubernetes) | — |
| **Caddy** e **Traefik** | Servidores web / proxies reversos (alternativas ao Nginx) | [[O que é o NGINX]], [[Proxy x Proxy Reverso]] |
| **CockroachDB** | Banco de dados SQL distribuído | — |
| **Hugo** | Gerador de sites estáticos | — |

Por que Go dominou essa área?

1. **Binário estático** → perfeito para imagens Docker mínimas (`FROM scratch`), ver [[Imagens]]
2. **Concorrência barata** → um servidor atende dezenas de milhares de conexões com uma goroutine por conexão (ver [[HTTP (net-http)]])
3. **Cross-compilation trivial** → compilar para Linux/ARM a partir do Mac com uma variável de ambiente
4. **Biblioteca padrão forte em rede** → `net`, `net/http`, `crypto/tls` prontos para produção

---

## 📅 6. Linha do tempo das versões

Go lança uma versão maior **a cada 6 meses** (fevereiro e agosto). Cada versão é suportada até saírem **duas versões mais novas**.

| Versão | Ano | Marco principal |
| --- | --- | --- |
| Go 1.0 | 2012 | Primeira versão estável + **promessa de compatibilidade do Go 1** |
| Go 1.5 | 2015 | Compilador reescrito **em Go** (antes era em C) + GC concorrente |
| Go 1.11 | 2018 | **Go Modules** (gerenciamento de dependências) |
| Go 1.13 | 2019 | Wrapping de erros (`%w`, `errors.Is`, `errors.As`) |
| Go 1.14 | 2020 | Preempção assíncrona de goroutines |
| Go 1.16 | 2021 | `embed` (arquivos dentro do binário); modules como padrão |
| Go 1.18 | 2022 | **Generics**, fuzzing nativo, workspaces |
| Go 1.21 | 2023 | `min`, `max`, `clear`; pacotes `slices`, `maps`, `log/slog`; gerenciamento de toolchain |
| Go 1.22 | 2024 | Nova semântica da variável de `for` (uma por iteração); `range` sobre inteiros; roteamento por método no `ServeMux` |
| Go 1.23 | 2024 | **Iteradores** (`range` sobre funções), pacote `iter` |
| Go 1.24 | 2025 | Maps reimplementados com **Swiss Tables**; aliases genéricos; `os.Root`; `testing.B.Loop` |
| Go 1.25 | 2025 | `GOMAXPROCS` ciente de containers (cgroups); `sync.WaitGroup.Go`; `testing/synctest` |
| Go 1.26 | 2026 | GC **Green Tea** por padrão; `new(expr)`; `go fix` com modernizadores; `errors.AsType` |
| **Go 1.27** | **2026** | **Métodos genéricos**; `encoding/json` passa a usar o motor do `json/v2`; profile de vazamento de goroutines |

> 💡 **Promessa de compatibilidade do Go 1:** garantia oficial de que um programa escrito para Go 1.0 continua compilando e funcionando em qualquer Go 1.x. Por isso a linguagem evolui devagar e com cuidado: código de 2012 ainda compila hoje. Quando uma mudança de comportamento é inevitável (como a variável de loop no 1.22), ela só vale para módulos que declaram a versão nova no `go.mod`.

---

## 🐹 7. Curiosidades e vocabulário

- **Go ou Golang?** O nome oficial é **Go**. "Golang" surgiu porque o site original era `golang.org` (o domínio `go.org` não estava disponível) e porque "go" é uma palavra difícil de pesquisar. Hoje o site oficial é **go.dev**, mas "golang" ainda é usado em buscas e hashtags.
- **Gopher:** o mascote de Go (um esquilo-da-terra), criado por Renée French. Programadores Go são chamados de *gophers*.
- **Go Proverbs:** frases curtas de Rob Pike que resumem a filosofia da linguagem, como *"A little copying is better than a little dependency"* (um pouco de cópia é melhor do que uma pequena dependência) e *"Errors are values"* (erros são valores).
- **Go Playground:** ambiente online oficial (go.dev/play) para rodar código Go sem instalar nada.

---

## ⚠️ 8. Críticas comuns (e o contraponto)

Ser honesto sobre as fraquezas ajuda a entender as escolhas:

| Crítica | Contraponto |
| --- | --- |
| "`if err != nil` em todo lugar é repetitivo" | É intencional: o caminho de erro fica visível. A comunidade discutiu várias propostas de açúcar sintático e, até agora, todas foram rejeitadas |
| "Faltam recursos (enums de verdade, tipos soma, `Option`)" | Simplicidade é a prioridade; recursos só entram quando não complicam o resto |
| "O GC causa pausas" | As pausas do GC de Go são da ordem de **microssegundos** a poucos milissegundos, e o Green Tea (1.26) reduziu bastante o custo do GC |
| "Binários grandes" | Um "hello world" tem ~2 MB porque carrega o runtime inteiro, em troca de não ter dependências externas |
| "Menos controle que C/Rust" | Verdade. Para kernels, drivers e sistemas embarcados de tempo real, C e Rust continuam sendo as escolhas certas |

> 💡 Isso conversa direto com o seu roadmap: para **kernel e drivers**, C (e cada vez mais Rust) é o caminho. Go brilha na camada logo acima: ferramentas de sistema, runtimes de containers, servidores e infraestrutura.

---

## 📋 Resumo

| Pergunta | Resposta |
| --- | --- |
| Quem criou? | Robert Griesemer, Rob Pike e Ken Thompson, no Google |
| Quando? | Desenho em 2007, anúncio em 2009, Go 1.0 em 2012 |
| Por quê? | Builds lentos de C++, CPUs multi-core, sistemas enormes em rede, times grandes |
| O que é? | Linguagem compilada, estática, com GC e concorrência embutida |
| Filosofia | Simplicidade, uma forma de fazer, explícito > mágico, composição > herança, CSP |
| O que não tem? | Classes, herança, exceções, sobrecarga, conversão implícita, ternário |
| Onde brilha? | Infraestrutura cloud, servidores, CLIs, ferramentas DevOps |
| Versão atual | Go 1.27 (agosto/2026) |

---

## 🔗 Conexões

**Dentro de Go**
- Como o código vira executável e o que é o runtime → [[Como Go Funciona]]
- Concorrência no estilo CSP → [[Fundamentos de Concorrência]], [[Goroutines]], [[Channels]]
- Composição no lugar de herança → [[Composição vs Herança]], [[Interfaces]]
- Erros como valores → [[O Padrão error em Go]]

**Sistemas Operacionais**
- [[A Linguagem C]] — Go foi pensado como um "C moderno" por pessoas que criaram o próprio C e o UNIX; herda a sintaxe enxuta, mas troca o gerenciamento manual de memória por GC
- [[Implementações Híbridas]] e [[Implementando Threads em User Space]] — as goroutines são threads de usuário multiplexadas sobre threads do kernel (modelo M:N)
- [[Troca de Mensagens]] — os channels de Go são troca de mensagens dentro de um mesmo processo
- [[Paravirtualização e a Máquina Virtual Java]] — contraste: Java roda sobre uma VM; Go compila direto para código nativo

**Docker, Redes e Nginx**
- [[RunC]], [[Container.d]], [[Runtime e Engine]] — toda a pilha de runtime do Docker é escrita em Go
- [[Imagens]] — binários estáticos de Go permitem imagens `FROM scratch` com poucos MB

---

⬅️ Início da trilha · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Como Go Funciona]] ➡️
