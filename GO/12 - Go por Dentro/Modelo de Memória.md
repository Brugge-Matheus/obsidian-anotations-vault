---
tags:
  - go
  - go/runtime
aliases:
  - Go Memory Model
  - Modelo de memória do Go
  - Happens-before
  - DRF-SC
  - Consistência sequencial
  - Reordenamento de memória
ordem: 52
modulo: "12 - Go por Dentro"
---
# Modelo de Memória

> **Nota 52 da trilha de Go.** Em [[Fundamentos de Concorrência]] vimos a intuição de **happens-before** e a promessa de que um programa sem data races se comporta "de forma simples". Esta nota abre a caixa: **por que** um programa com corrida pode ver coisas impossíveis (o compilador apaga escritas, o hardware reordena leituras e escritas, valores de duas palavras se "rasgam"), o que o **modelo de memória de Go** promete formalmente, a lista exata de operações que sincronizam, exemplos clássicos de sincronização **incorreta** (como o *double-checked locking*), e como Go se posiciona entre C/C++ e Java. Os três efeitos estranhos foram **observados de verdade**, num processador x86 comum.
>
> **Pré-requisitos:** [[Fundamentos de Concorrência]], [[Channels]], [[sync.WaitGroup e sync.Mutex]].

---

## 🧭 De onde viemos

Em [[Fundamentos de Concorrência]] definimos data race, vimos o contador que perde incrementos e uma tabela de operações que criam happens-before. Em [[Channels]] listamos as regras de memória dos channels, e em [[sync.WaitGroup e sync.Mutex]] vimos que atômicos viram `LOCK XADD` e `XCHG`. Do lado do SO, [[Processadores]] e [[Memória]] apresentaram caches e pipelines, e [[Race Condition]] e [[Regiões Críticas]], o problema. Faltava a pergunta de fundo: o que exatamente uma goroutine **pode ver** das escritas de outra?

> 💡 **Modelo de memória:** a especificação que define, para uma linguagem (ou um processador), quais valores uma leitura de memória **pode** observar quando várias linhas de execução leem e escrevem as mesmas posições. É o contrato entre o programador, o compilador e o hardware.

---

## 🤯 1. Três coisas "impossíveis" que acontecem

Num programa **sequencial**, cada leitura vê a última escrita. Com várias goroutines e **sem sincronização**, isso deixa de valer, por três motivos diferentes. Todos os três abaixo são saídas reais (Go 1.27, x86-64, 2 núcleos).

### 1.1 O compilador apaga suas escritas

```go
var contador int

func main() {
	go func() {
		for {
			contador++ // ❌ data race: ninguém sincroniza com esta goroutine
		}
	}()
	time.Sleep(100 * time.Millisecond)
	fmt.Println("após 100 ms, o main vê contador =", contador)
}
```

```
após 100 ms, o main vê contador = 0
```

Zero. O `go tool objdump` da goroutine mostra por quê:

```
TEXT main.main.func1(SB)
main.go:12: NOPL
main.go:1:  JMP main.main.func1(SB)      ← um laço VAZIO: os incrementos foram removidos
```

Do ponto de vista de **uma** goroutine, o laço infinito nunca termina e ninguém sincroniza com ela, então nenhum outro código tem o **direito** de observar `contador`: o compilador pode eliminar as escritas. Um teste com uma interface teve o mesmo destino: as duas atribuições a uma variável global, num laço sem sincronização, sumiram do código gerado.

> 💡 A otimização inversa (ler uma variável **uma vez só** antes de um laço `for !pronto {}`, transformando-o num laço infinito) é igualmente **permitida** pelo modelo. Nos meus testes com o compilador atual, ela não aconteceu (o laço leu a memória a cada volta), mas compiladores de C/C++ a fazem rotineiramente e nada impede o de Go de fazê-la amanhã.

### 1.2 O processador reordena leituras e escritas

O teste clássico **store buffering** (SB): cada goroutine escreve na sua variável e depois lê a da outra.

```
goroutine A:  x = 1;  r1 = y          goroutine B:  y = 1;  r2 = x
```

Se as operações acontecessem em **alguma** ordem intercalada (consistência sequencial), pelo menos uma escrita viria antes da leitura do outro lado: o resultado `r1 == 0 && r2 == 0` seria **impossível**. Medição real, um milhão de rodadas (com uma barreira atômica para as duas começarem juntas a cada rodada):

```
variáveis comuns:   (r1,r2)=(0,0) em 215880 de 1000000 rodadas
variáveis atômicas: (r1,r2)=(0,0) em 0 de 1000000 rodadas
```

**21,6%** das rodadas produziram o "impossível". O compilador não reordenou nada (o assembly tem a escrita antes da leitura); quem reordenou foi o **hardware**:

```
           núcleo 0                                núcleo 1
  x = 1  → entra no STORE BUFFER do núcleo 0      y = 1  → STORE BUFFER do núcleo 1
  r1 = y → lê da cache: y ainda é 0               r2 = x → lê da cache: x ainda é 0
           ... depois os buffers são descarregados na cache (tarde demais)
```

Cada núcleo tem um **buffer de escritas**: para não esperar a cache, a escrita fica "pendurada" enquanto o núcleo segue executando, e uma leitura **posterior** de outro endereço pode ser servida antes. É o único reordenamento que o modelo **x86-TSO** (*Total Store Order*) permite; processadores ARM e POWER permitem muitos outros.

Com atômicos, a escrita vira outra instrução (saída real do `objdump`):

```
x = 1          →  MOVQ $0x1, 0(AX)       (vai para o store buffer)
x.Store(1)     →  XCHGQ R9, 0(AX)        (XCHG com memória tem LOCK implícito:
                                          esvazia o store buffer antes de continuar)
```

### 1.3 Valores de duas palavras se "rasgam"

Uma **interface** ocupa duas palavras: (ponteiro para o tipo, ponteiro para o dado). Uma goroutine alterna `atual = q` (um `*Quadrado`) e `atual = l` (um `*Lista`); outra lê e chama `atual.Area()`, que só poderia devolver 9 ou 2. Saída real:

```go
type Forma interface{ Area() int }
func (q *Quadrado) Area() int { return q.Lado * q.Lado }   // 3×3 = 9
func (l *Lista) Area() int    { return len(l.Itens) }      // 2

var atual Forma // uma interface = 2 palavras: (tipo, ponteiro)
// escritor: atual = q; nada(); atual = l; nada()   (as chamadas impedem a eliminação)
// leitor:   f := atual; a := f.Area()
```

```
valor impossível na leitura 205823 após 0s: Area() = 20000
```

O leitor pegou a palavra de **tipo** de um valor e a de **dado** do outro: executou o método de um tipo sobre a memória do outro. Aqui deu um número sem sentido; com outros tipos, dá **corrupção de memória** e falhas de segmentação. O mesmo vale para **slices** (ponteiro, tamanho, capacidade: um tamanho novo com o ponteiro antigo lê além do array), **strings** e maps.

---

## 📜 2. O que o modelo de Go promete

O documento oficial (*The Go Memory Model*, revisado em 2022 por Russ Cox) começa com um conselho que vale a pena citar: *"Se você precisa ler o resto deste documento para entender o comportamento do seu programa, você está sendo esperto demais. Não seja esperto."*

### 2.1 A promessa central: DRF-SC

> ✅ **DRF-SC (*data-race-free → sequentially consistent*):** se um programa **não tem data races**, ele se comporta como se as operações de todas as goroutines fossem **intercaladas numa única ordem**, respeitando a ordem de cada goroutine. Nada de store buffers, reordenamentos ou rasgos visíveis.

É a mesma garantia de Java, C++ e Rust para programas sem corrida. Na prática: **sincronize corretamente e você pode raciocinar como se o programa fosse simples**. Os três experimentos da seção 1 só foram possíveis porque tinham corridas.

### 2.2 As definições

| Termo | Definição |
| --- | --- |
| **Sequenced before** | A ordem dentro de **uma** goroutine, dada pelo código |
| **Synchronized before** | Uma operação de sincronização num ponto **antes** de outra, em outra goroutine (lista na seção 3) |
| **Happens before** | O fecho transitivo das duas: A acontece antes de C se houver uma cadeia A → B → C de "sequenced" e "synchronized" |
| **Data race** | Uma escrita e outro acesso (leitura ou escrita) à **mesma posição**, em goroutines diferentes, sem que um **aconteça antes** do outro (e sem serem ambos atômicos) |

Uma leitura `r` de uma variável `x` **pode** observar uma escrita `w` em `x` se `w` acontece antes de `r` (ou é concorrente a `r`, numa corrida), e não existe outra escrita `w'` em `x` com `w → w' → r`.

---

## 🔗 3. As operações que sincronizam

| Operação | Garantia |
| --- | --- |
| Inicialização de pacotes | O `init` de um pacote importado acontece antes do `init` de quem importa; todos os `init` acontecem antes de `main.main` |
| `go f()` | A instrução `go` acontece antes do início de `f` |
| Fim de uma goroutine | **Nenhuma** garantia: sair não sincroniza com ninguém (é preciso um channel ou WaitGroup) |
| Envio num channel | Acontece antes de o recebimento correspondente **completar** |
| `close(ch)` | Acontece antes de um recebimento que retorna porque o channel fechou |
| Recebimento num channel **sem buffer** | Acontece antes de o envio correspondente **completar** |
| k-ésimo recebimento num channel de capacidade C | Acontece antes de o (k+C)-ésimo envio completar (semáforos) |
| `mu.Unlock()` (n-ésimo) | Acontece antes do `mu.Lock()` (n+1-ésimo) retornar |
| `rw.RUnlock()` / `rw.Unlock()` | Regras análogas entre leitores e escritores |
| `once.Do(f)` | O retorno de `f` acontece antes de qualquer `once.Do` retornar |
| Operações **atômicas** | Se uma leitura atômica observa uma escrita atômica, a escrita acontece antes; e **todas** as operações atômicas se comportam como numa ordem sequencialmente consistente |
| `wg.Done()` → `wg.Wait()`, `cond.Signal()` → `Wait`, `context` cancelado → `Done()` fechado, etc. | Documentadas em cada pacote; implementadas com as primitivas acima |

> 💡 Os atômicos de Go são **sequencialmente consistentes** (equivalentes ao `memory_order_seq_cst` de C++). Não há atômicos "relaxados", de propósito: são mais simples de usar corretamente, e o custo extra é pequeno em x86 (a escrita vira `XCHG`) e moderado em ARM (instruções `LDAR`/`STLR`).

---

## ❌ 4. Sincronização incorreta (exemplos do documento oficial)

### 4.1 Publicar um ponteiro sem sincronização

```go
var cfg *Config

func carregar() { c := &Config{Porta: 8080, Host: "x"}; cfg = c }   // goroutine A
func usar()     { if cfg != nil { conectar(cfg.Host, cfg.Porta) } }  // goroutine B ❌
```

B pode ver `cfg != nil` e, ainda assim, ver os **campos** com valores antigos (zero): nada ordena as escritas dos campos antes da escrita do ponteiro, do ponto de vista de B. Em x86 é raro (o TSO preserva a ordem entre escritas), mas em **ARM** (servidores Graviton, Macs, celulares) acontece. Correção: `atomic.Pointer[Config]` (o padrão estilo RCU de [[sync.WaitGroup e sync.Mutex]]), um mutex, ou um channel.

### 4.2 *Double-checked locking*

```go
var instancia *Servico
var mu sync.Mutex

func Obter() *Servico {
	if instancia == nil {          // ❌ leitura sem trava: corrida com a escrita abaixo
		mu.Lock()
		if instancia == nil {
			instancia = novoServico()
		}
		mu.Unlock()
	}
	return instancia               // pode ver o ponteiro sem ver o objeto inicializado
}
```

O padrão (famoso também em Java antes de 2004) tenta evitar o lock no caminho comum, mas a primeira leitura não sincroniza com a escrita. Use **`sync.Once`** ou **`sync.OnceValue`**: fazem exatamente isso, corretamente, com um atômico no caminho rápido.

### 4.3 Espera ocupada numa variável comum

```go
var pronto bool
go func() { dado = calcular(); pronto = true }()
for !pronto {}                     // ❌ pode nunca terminar (seção 1.1), e mesmo
fmt.Println(dado)                  //    terminando, pode ver 'dado' antigo
```

Correção: `atomic.Bool`, ou melhor, um channel (`<-feito`), que também evita queimar CPU girando.

---

## ⚖️ 5. Go entre C++ e Java

O que acontece com um programa **que tem** corrida?

| Linguagem | Programa com data race | Consequência |
| --- | --- | --- |
| C / C++ | **Comportamento indefinido** | Literalmente qualquer coisa: o compilador pode supor que corridas não existem e gerar código absurdo |
| Java | Definido: cada leitura vê **alguma** escrita; a segurança de tipos é mantida | Resultados estranhos, mas nunca corrupção de memória |
| **Go** | Corridas em valores de **uma palavra** se comportam "como Java": cada leitura vê uma escrita que realmente aconteceu | Mas valores de **várias palavras** (interfaces, slices, strings, maps) podem se rasgar e **corromper a memória** (seção 1.3) |

O documento também restringe o **compilador**: ele não pode inventar escritas que não estão no programa (por exemplo, escrever um valor temporário numa variável compartilhada), nem fazer uma leitura de palavra observar um valor que nunca foi escrito. Ele **pode** eliminar escritas e leituras redundantes, reordenar acessos independentes, e manter valores em registradores, como vimos.

> ⚠️ Conclusão prática: em Go, **não existe corrida benigna**. Mesmo um "só estou lendo um contador para log" pode, se o valor tiver várias palavras, derrubar o programa. Use o detector de corridas (`-race`) e sincronize.

---

## 🖥️ 6. O que o hardware faz

| Teste clássico | Padrão | x86 (TSO) | ARM / POWER |
| --- | --- | --- | --- |
| **SB** (store buffering) | `x=1; r1=y` ∥ `y=1; r2=x` → (0,0)? | **Permitido** (21,6% no experimento) | Permitido |
| **MP** (message passing) | `dado=1; flag=1` ∥ `r1=flag; r2=dado` → flag visto, dado não? | Proibido | **Permitido** |
| **LB** (load buffering) | `r1=x; y=1` ∥ `r2=y; x=1` → (1,1)? | Proibido | Permitido (raro) |

É por isso que o modelo de memória é definido em termos **da linguagem**, não do hardware: o mesmo programa Go precisa funcionar em x86 e em ARM. O compilador traduz as operações de sincronização para as **barreiras** (*fences*) de cada arquitetura (`XCHG`/`LOCK` no x86; `LDAR`/`STLR`/`DMB` no ARM64), e o código sem sincronização fica livre para ser otimizado ao máximo.

---

## 🔬 7. Por dentro: como as primitivas cumprem o contrato

| Primitiva | O que garante a ordem |
| --- | --- |
| Atômicos | Instruções com semântica de barreira (`XCHG`, `LOCK CMPXCHG` no x86) + o compilador não move acessos através delas |
| `sync.Mutex` | O `Lock` termina com um CAS (barreira de aquisição); o `Unlock` faz uma operação atômica (barreira de liberação) |
| Channels | O lock interno da `hchan` (ver [[Channels]]) ordena quem envia e quem recebe |
| `go f()` | Os argumentos são escritos na nova goroutine antes de ela ser colocada na fila (operações com o lock/atômicos do escalonador) |
| Detector de corridas | Mantém relógios vetoriais por goroutine e por posição de memória, atualizados exatamente nos pontos de sincronização da seção 3 (ver [[Fundamentos de Concorrência]]) |

---

## ⚠️ 8. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| "Corrida benigna" num contador ou flag | Escritas apagadas, laços eternos | `sync/atomic` |
| Ler interface/slice/string compartilhada sem trava | Valores rasgados, crash | Mutex, `atomic.Pointer`, channel |
| Publicar um ponteiro sem sincronização | Campos com valores zero (em ARM) | `atomic.Pointer`, mutex, channel |
| *Double-checked locking* | Objeto meio inicializado | `sync.Once`/`OnceValue` |
| Supor que o fim de uma goroutine sincroniza | Leituras antigas | `WaitGroup`, channel |
| Testar só em x86 | Bugs que aparecem em ARM | `-race`; não depender do hardware |
| `time.Sleep` como sincronização | Funciona "quase sempre" | Primitivas de sincronização |
| Ler o modelo de memória para justificar código esperto | Bugs sutis | "Não seja esperto": mutex/channel |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Por que existe | Compiladores e processadores reordenam e eliminam acessos à memória |
| Compilador | Apagou um laço de incrementos inteiro (`contador = 0` após 100 ms) |
| Hardware | Store buffer do x86: (0,0) "impossível" em 21,6% das rodadas |
| Rasgos | Interfaces, slices e strings (várias palavras) se misturam sob corrida |
| DRF-SC | Sem data races → comportamento sequencialmente consistente |
| Happens-before | Fecho de "sequenced before" e "synchronized before" |
| Sincronizam | `init`, `go`, channels (4 regras), mutexes, `Once`, atômicos (seq. consistentes) |
| Não sincroniza | O fim de uma goroutine |
| Go vs outras | Melhor que C++ (sem UB para uma palavra), mas multipalavra pode corromper memória |
| Regra prática | Não existe corrida benigna; use `-race`; não seja esperto |

---

## 🔗 Conexões

**Dentro de Go**
- A introdução de happens-before e o detector de corridas → [[Fundamentos de Concorrência]]
- Regras de memória dos channels → [[Channels]]
- Mutex, `Once`, atômicos, `atomic.Pointer` → [[sync.WaitGroup e sync.Mutex]]
- Valores de várias palavras: interfaces, slices, strings → [[Interfaces]], [[Arrays e Slices]], [[Strings em Profundidade]]
- `go f()` e o escalonador → [[Goroutines]], [[Scheduler do Go]]
- Ordem de inicialização de pacotes → [[Criando Pacotes]]
- `-race` em testes → [[Testes com go test]]
- Ler o assembly gerado → [[Build, Flags e Cross-compilation]]

**Sistemas Operacionais**
- [[Race Condition]] e [[Regiões Críticas]] — o problema de base
- [[Exclusão mútua com espera ocupada]] — por que o algoritmo de Peterson **falha** em processadores modernos sem barreiras (é um teste "store buffering")
- [[Mutexes]] — as barreiras embutidas nas primitivas de trava
- [[Processadores]] — pipelines, execução fora de ordem, store buffers
- [[Memória]] — caches e coerência entre núcleos
- [[Inversão de Prioridade e RCU]] — RCU depende de publicar ponteiros com as barreiras certas

---

⬅️ **Anterior:** [[Scheduler do Go]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Reflection]] ➡️
