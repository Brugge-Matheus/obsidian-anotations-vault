---
tags:
  - go
  - go/funcoes
aliases:
  - Funções em Go
  - Funções de primeira classe
  - Higher-order functions
  - Stack frame
ordem: 16
modulo: "04 - Funções"
---
# Funções — Fundamentos

> **Nota 16 da trilha de Go.** Funções são a unidade básica de organização de código: um bloco com nome, que recebe **parâmetros**, executa algo e devolve **resultados**. Em Go, funções também são **valores de primeira classe**: podem ser guardadas em variáveis, passadas como argumento e devolvidas por outras funções. Esta nota cobre a declaração, parâmetros, retorno, tipos de função, funções de ordem superior e recursão, e termina mostrando o que acontece na CPU numa chamada: **registradores, stack frame, crescimento da stack da goroutine e inlining**. As notas seguintes do módulo aprofundam múltiplos retornos, variádicas, closures e defer/panic.
>
> **Pré-requisitos:** [[Ponteiros]], [[Structs]].

---

## 🧭 De onde viemos

Já usamos funções desde o primeiro programa (`main`, `fmt.Println`, `strconv.Atoi`, `append`). Em [[Ponteiros]] vimos que **Go sempre passa argumentos por valor**, e em [[Structs]] que passar uma struct copia seus campos. Agora vamos estudar as funções de forma completa, começando pelo básico e descendo até o nível da máquina.

> 💡 **Função:** sequência de instruções com nome, que pode receber valores de entrada (**parâmetros**) e devolver valores de saída (**resultados**). Chamar uma função transfere o controle para ela; ao terminar, o controle volta para o ponto da chamada.

---

## ✍️ 1. Declarando funções

```go
func nome(parametro1 Tipo1, parametro2 Tipo2) TipoDoResultado {
	// corpo
	return valor
}
```

```go
func somar(a int, b int) int {
	return a + b
}

func somar(a, b int) int {     // parâmetros consecutivos do mesmo tipo: tipo uma vez só
	return a + b
}

func saudar(nome string) {     // sem resultado: não tem tipo de retorno
	fmt.Println("Olá,", nome)
}

func agora() time.Time {       // sem parâmetros
	return time.Now()
}
```

Regras:

- A ordem é **nome antes do tipo** (`a int`), coerente com `var a int`
- Funções são declaradas no **nível do pacote** (não dentro de outras funções; para isso existem funções anônimas, ver [[Closures e Funções Anônimas]])
- **Não existe sobrecarga**: dois `func somar` no mesmo pacote não compilam, nem com parâmetros diferentes. Use nomes diferentes (`somarInts`, `somarFloats`) ou [[Generics]]
- **Não existem parâmetros com valor padrão** nem parâmetros nomeados na chamada. Para opções, use structs de configuração ou opções funcionais
- A ordem de declaração não importa: uma função pode chamar outra declarada mais abaixo (escopo de pacote, ver [[Sintaxe, Blocos e Escopo]])
- Nome com letra maiúscula = exportada

---

## 📥 2. Parâmetros

### 2.1 Sempre por valor (revisão)

Cada parâmetro é uma **variável local** da função, inicializada com uma **cópia** do argumento. O que muda é o que está sendo copiado:

| Tipo do argumento | O que é copiado | A função pode alterar o original? |
| --- | --- | --- |
| `int`, `string`, `bool`, struct, array | O valor inteiro | Não |
| `*T` (ponteiro) | O endereço | Sim, via `*p` |
| slice | O cabeçalho (ptr, len, cap) | Os **elementos**, sim; o tamanho, não |
| map, channel | Um ponteiro interno | Sim |
| função | Um ponteiro para o código (e contexto) | — |

Vimos tudo isso em detalhe em [[Ponteiros]] e [[Arrays e Slices]].

### 2.2 Parâmetros não usados

Diferente de variáveis locais, um parâmetro não usado **não** é erro de compilação (funções frequentemente precisam seguir uma assinatura imposta). Para deixar explícito, use `_`:

```go
func handler(_ http.ResponseWriter, r *http.Request) { ... }
```

---

## 📤 3. Resultados

```go
func area(largura, altura float64) float64 {
	return largura * altura
}
```

- `return` encerra a função imediatamente
- Uma função com tipo de resultado **precisa** terminar com `return` (ou `panic`) em todos os caminhos; o compilador verifica (`missing return`)

Go permite **vários resultados** (`func f() (int, error)`) e **resultados nomeados** (`func f() (n int, err error)`). Esse é o assunto da próxima nota, [[Múltiplos Retornos e Erros]], porque é a base de todo o tratamento de erros de Go.

---

## 🧩 4. Funções são valores

Em Go, uma função é um valor como qualquer outro, com um **tipo** definido pela sua assinatura (saída real):

```go
fmt.Printf("%T\n", somar)    // func(int, int) int
```

> 💡 **Funções de primeira classe:** a linguagem trata funções como valores: podem ser atribuídas a variáveis, guardadas em slices e maps, passadas como argumentos e devolvidas como resultado. Go, JavaScript, Python e Ruby têm; C tem apenas ponteiros para função (sem closures); Java só ganhou algo parecido com lambdas no Java 8.

```go
var op func(int, int) int     // variável do tipo função; zero value: nil
op = somar
fmt.Println(op(10, 5))        // 15

var f func(int) int
fmt.Println(f == nil)         // true
f(1)                          // 💥 panic: chamar função nil
```

Funções só podem ser comparadas com `nil` (vimos o porquê em [[Operadores de Comparação e Lógicos]]).

### 4.1 Tipos de função nomeados

Para assinaturas usadas em vários lugares, dê um nome ao tipo (saída real):

```go
type Operacao func(a, b int) int

ops := map[string]Operacao{
	"+": somar,
	"*": func(a, b int) int { return a * b },   // função anônima
}
fmt.Println(ops["*"](3, 4))   // 12
```

A biblioteca padrão faz isso muito: `http.HandlerFunc`, `filepath.WalkFunc`, `sort.Less`...

### 4.2 Funções de ordem superior

> 💡 **Função de ordem superior (higher-order function):** função que **recebe** outra função como parâmetro ou **devolve** uma função.

Saída real:

```go
func aplicar(nums []int, f func(int) int) []int {
	out := make([]int, 0, len(nums))
	for _, n := range nums {
		out = append(out, f(n))
	}
	return out
}

func dobro(n int) int { return n * 2 }

fmt.Println(aplicar([]int{1, 2, 3}, dobro))   // [2 4 6]
```

A biblioteca padrão está cheia de funções que recebem funções:

```go
strings.Map(func(r rune) rune { return r + 1 }, "HAL")   // "IBM"
slices.SortFunc(s, func(a, b string) int { return strings.Compare(a, b) })
slices.IndexFunc(s, func(x string) bool { return x == "go" })
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) { ... })
```

> 💡 Go **não** tem `map`/`filter`/`reduce` genéricos na biblioteca padrão (por escolha: um `for` é igualmente claro em Go). Com [[Generics]] e [[Iteradores]], é fácil escrevê-los quando fazem sentido.

---

## 🔁 5. Recursão

Uma função pode chamar a si mesma (saída real):

```go
func fatorial(n int) int {
	if n <= 1 {
		return 1                // caso base: sem ele, a recursão nunca termina
	}
	return n * fatorial(n-1)    // caso recursivo
}

fmt.Println(fatorial(10))       // 3628800
```

Cada chamada recursiva ocupa um **novo frame na stack** (seção 7). Em Go, a stack de uma goroutine **cresce sob demanda**, então recursões profundas funcionam: 1 milhão de níveis rodou sem problema no teste desta nota. O limite padrão é **1 GB** de stack por goroutine. Ao passar disso (saída real de uma recursão infinita):

```
runtime: goroutine stack exceeds 1000000000-byte limit
fatal error: stack overflow
```

> ⚠️ Go **não** faz otimização de chamada de cauda (*tail call optimization*). Uma recursão "de cauda" consome stack igual a qualquer outra. Para profundidades enormes (percorrer milhões de nós), prefira uma versão iterativa com uma pilha explícita (um slice).

---

## 🏁 6. Funções especiais

| Função | Papel |
| --- | --- |
| `main()` no pacote `main` | Ponto de entrada do programa; sem parâmetros nem resultados |
| `init()` | Roda automaticamente antes de `main`, uma vez por pacote; pode haver várias por pacote; não pode ser chamada manualmente. Ver [[Criando Pacotes]] |
| Funções embutidas (`len`, `append`, `make`, `new`, `panic`...) | Implementadas pelo compilador; algumas são "genéricas" desde sempre |
| Métodos | Funções com um **receptor**: `func (c *Conta) Depositar(v int)`. Ver [[Métodos e Receptores]] |

---

## 🔬 7. Por dentro: o que acontece numa chamada

### 7.1 Argumentos e resultados em registradores (ABI de Go)

> 💡 **ABI (Application Binary Interface):** o contrato, no nível de máquina, de como funções se chamam: em que registradores ou posições da stack vão os argumentos, onde fica o resultado, quem salva quais registradores. É a "convenção de chamada".

Desde o **Go 1.17** (amd64), argumentos e resultados inteiros/ponteiros passam pelos registradores `AX, BX, CX, DI, SI, R8, R9, R10, R11` (nessa ordem); floats usam `X0` a `X14`. Antes disso, tudo ia pela stack. Compilando com `go build -gcflags=-S` (saída real):

```go
//go:noinline
func somar(a, b int) int { return a + b }
```

```
main.somar:
    ADDQ  BX, AX      ; a está em AX, b em BX; o resultado fica em AX
    RET               ; nenhum acesso à memória: tudo em registradores

chamada em main:
    MOVL  $2, AX      ; primeiro argumento
    MOVL  $3, BX      ; segundo argumento
    CALL  main.somar  ; resultado volta em AX
```

> 💡 A ABI interna de Go é **diferente** da ABI do sistema (a System V do Linux usa `RDI, RSI, RDX, RCX, R8, R9`). O Go mantém sua ABI própria para poder evoluí-la livremente; quando código Go conversa com C (cgo) ou com o kernel, o runtime faz a "tradução". Essa mudança para registradores deixou programas Go cerca de 5% mais rápidos.

### 7.2 O stack frame

Quando a função precisa de memória para variáveis locais (ou para chamar outras funções), ela reserva um **frame** na stack da goroutine:

```
stack da goroutine (cresce para endereços menores)

        ┌───────────────────────────┐  endereços altos
        │   frame de main           │
        │   variáveis locais        │
        ├───────────────────────────┤
        │   endereço de retorno     │  ← empilhado pelo CALL
        │   BP salvo                │  ← prólogo: PUSHQ BP; MOVQ SP, BP
        │   frame de calcular       │
        │   variáveis locais,       │
        │   argumentos que não      │
        │   couberam em registrador │
        ├───────────────────────────┤  ← SP (topo da stack)
        │   (espaço livre)          │
        │           ...             │
        └───────────────────────────┘  ← limite (stack guard)
```

- `CALL` empilha o endereço da instrução seguinte e salta para a função
- `RET` desempilha esse endereço e volta
- Uma função "folha" simples (como `somar`) nem cria frame (`NOFRAME` no assembly)

Você viu a pilha de execução de um processo em [[O Modelo de Processos]] e [[Implementação de Processos]]: é o mesmo mecanismo, só que cada **goroutine** tem sua própria stack, gerenciada pelo runtime.

### 7.3 A stack que cresce: o prólogo de verificação

Uma thread do SO tem stack de tamanho **fixo** (tipicamente 8 MB no Linux, reservados na criação). Uma goroutine começa com uma stack **pequena** (por volta de 2 KB; desde o Go 1.19 o runtime ajusta o tamanho inicial pela média histórica de uso) que **cresce quando necessário**.

Como o runtime sabe quando crescer? Quase toda função começa com uma checagem gerada pelo compilador (saída real, prólogo de `main`):

```
main.main:
    CMPQ  SP, 16(R14)        ; R14 aponta para a goroutine atual (g);
                             ; 16(R14) é o "stack guard" (limite)
    JLS   precisa_crescer    ; se SP passou do limite → chama runtime.morestack
    ...corpo...
precisa_crescer:
    CALL  runtime.morestack_noctxt
    JMP   main.main          ; recomeça a função com a stack nova
```

O `runtime.morestack`:

1. Aloca uma stack nova com o **dobro** do tamanho
2. **Copia** a stack antiga inteira para a nova
3. **Ajusta todos os ponteiros** que apontavam para dentro da stack antiga (é possível porque o compilador registra onde há ponteiros em cada frame)
4. Libera a antiga e continua a execução

```
antes:  [ 2 KB  ████████████▓ ]   ← cheia
depois: [ 4 KB  ████████████▓                    ]   ← copiada, com espaço
```

Quando a goroutine volta a usar pouca stack, o GC pode **encolher** a stack pela metade. Esse mecanismo (stacks contíguas copiáveis, desde o Go 1.4) é o que permite criar **milhões** de goroutines: elas só ocupam o que realmente usam. Veremos o restante em [[Goroutines]] e [[Scheduler do Go]].

> 💡 Funções marcadas `NOSPLIT` (como `somar`) pulam a checagem porque usam pouquíssima stack. O linker garante que uma cadeia de funções `NOSPLIT` nunca estoure a margem de segurança.

### 7.4 Inlining: chamadas que desaparecem

Para funções pequenas, o compilador **copia o corpo** para o local da chamada, eliminando `CALL`/`RET`, o frame e a checagem de stack. Saída real de `go build -gcflags=-m`:

```
./main.go:9:6: can inline somar
./main.go:19:6: can inline dobro
./main.go:23:6: can inline fatorial
```

```go
x := somar(2, 3)      // depois do inlining, vira simplesmente:  x := 2 + 3  →  x := 5
```

O ganho não é só evitar o salto: com o corpo "colado" no chamador, o compilador consegue **otimizar junto** (propagar constantes, eliminar checagens, decidir que algo não escapa para a heap). O compilador mede o "custo" de cada função (um orçamento de ~80 nós da AST) para decidir; com **PGO** (ver [[Profiling e Diagnóstico]]), funções quentes ganham um orçamento maior.

```go
//go:noinline         // diretiva para impedir o inlining (útil em benchmarks e para estudar o assembly)
func somar(a, b int) int { return a + b }
```

### 7.5 Chamar por valor de função custa um pouco mais

```go
dobro(5)          // chamada DIRETA: o endereço é conhecido em compilação (e pode ser inlinada)
f := dobro
f(5)              // chamada INDIRETA: lê o endereço da variável e salta (CALL via registrador)
```

Chamadas indiretas não podem ser inlinadas (a não ser que o compilador descubra qual função é) e são um pouco mais difíceis para o preditor de desvios da CPU. A diferença é de nanossegundos e só importa em laços extremamente quentes.

---

## ⚠️ 8. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Esperar que a função altere um argumento por valor | Original intacto | Passe `*T` ou devolva o novo valor |
| Tentar sobrecarregar funções | `somar redeclared in this block` | Nomes diferentes ou generics |
| Querer parâmetro com valor padrão | Não existe | Struct de opções ou opções funcionais |
| Chamar variável de função `nil` | panic | Verifique `f != nil` |
| Recursão muito profunda | `fatal error: stack overflow` (1 GB) | Versão iterativa com pilha explícita |
| Contar com otimização de cauda | Stack cresce mesmo assim | Laço em vez de recursão |
| Esquecer `return` num caminho | `missing return` | Cubra todos os caminhos |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Declaração | `func nome(p T) R { ... }`; parâmetros do mesmo tipo agrupados |
| Restrições | Sem sobrecarga, sem valores padrão, sem argumentos nomeados |
| Passagem | Sempre por valor (copia o valor, o cabeçalho ou o ponteiro) |
| Funções como valores | Têm tipo (`func(int) int`), zero value `nil`, só comparáveis com `nil` |
| Tipo nomeado | `type Operacao func(a, b int) int` |
| Ordem superior | Funções que recebem/devolvem funções (`slices.SortFunc`, `http.HandleFunc`) |
| Recursão | Funciona com stack crescente; limite de 1 GB; sem TCO |
| ABI | Argumentos e resultados em registradores (`AX`, `BX`, `CX`...) desde 1.17 |
| Stack | ~2 KB inicial, cresce por cópia (`morestack`), encolhe no GC |
| Inlining | Funções pequenas copiadas no chamador; `-gcflags=-m` mostra |

---

## 🔗 Conexões

**Dentro de Go**
- Vários resultados e o padrão `(valor, error)` → [[Múltiplos Retornos e Erros]]
- Número variável de argumentos → [[Funções Variádicas]]
- Funções anônimas e closures → [[Closures e Funções Anônimas]]
- Execução adiada e panics → [[Defer, Panic e Recover]]
- Métodos: funções com receptor → [[Métodos e Receptores]]
- Funções genéricas → [[Generics]]
- `init()` e ordem de inicialização → [[Criando Pacotes]]
- Stacks de goroutines → [[Goroutines]], [[Scheduler do Go]]
- PGO e perfis → [[Profiling e Diagnóstico]]

**Sistemas Operacionais**
- [[Processadores]] — registradores, `CALL`/`RET`, pipeline e predição de desvios em chamadas indiretas
- [[O Modelo de Processos]] e [[Implementação de Processos]] — a pilha de execução e o contexto salvo de cada processo/thread
- [[Threads POSIX]] — threads do SO com stack fixa (8 MB) vs stacks pequenas e crescentes das goroutines
- [[System Calls]] — a convenção de chamada do kernel é outra ABI; o runtime traduz ao fazer syscalls

**Algoritmos e Estruturas de Dados**
- [[Pilha (Stack)]] — a call stack é literalmente uma pilha LIFO de frames
- [[Complexidade de Algoritmos]] — custo de espaço da recursão (profundidade × tamanho do frame)

---

⬅️ **Anterior:** [[Structs]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Múltiplos Retornos e Erros]] ➡️
