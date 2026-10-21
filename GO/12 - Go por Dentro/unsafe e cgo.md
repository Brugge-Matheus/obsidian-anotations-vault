---
tags:
  - go
  - go/runtime
aliases:
  - unsafe
  - unsafe.Pointer
  - cgo
  - Chamar C a partir de Go
  - runtime.Pinner
  - checkptr
ordem: 54
modulo: "12 - Go por Dentro"
---
# unsafe e cgo

> **Nota 54 da trilha de Go, a última.** Go protege você de muita coisa: não há aritmética de ponteiros, o GC sabe onde está cada ponteiro, os tipos são verificados, os índices também. Esta nota é sobre as duas **saídas de emergência** dessa proteção. O pacote **`unsafe`** permite contornar o sistema de tipos e manipular memória diretamente (layout de structs, conversões sem cópia, aritmética de ponteiros), e o **cgo** permite chamar (e ser chamado por) código **C**. Veremos o que cada um oferece, as regras que precisam ser seguidas para não corromper o runtime, as ferramentas que vigiam essas regras (`checkptr`, `go vet`, `cgocheck`), o custo real de uma chamada cgo (e as threads que ela prende), e quando vale a pena usar, ou evitar, cada um.
>
> **Pré-requisitos:** [[Ponteiros]], [[Structs]], [[Garbage Collector]], [[Scheduler do Go]], [[Build, Flags e Cross-compilation]].

---

## 🧭 De onde viemos

Muitas notas apontaram para cá. Em [[Ponteiros]] vimos que Go proíbe aritmética de ponteiros porque o GC precisa saber **exatamente** onde há ponteiros; em [[Tipos de Dados]], que `uintptr` é um número, não um ponteiro; em [[Structs]], alinhamento e *padding*, e que Go **não garante** o layout de structs para conversar com C; em [[Goroutines]], que stacks **se movem** quando crescem, e que por isso o C não pode guardar ponteiros para memória de Go; em [[Strings em Profundidade]], conversões entre `string` e `[]byte` que copiam; em [[Build, Flags e Cross-compilation]], que cgo torna o binário dinâmico e complica a compilação cruzada; em [[Scheduler do Go]], que chamadas cgo prendem threads. Do lado do SO, [[A Linguagem C]] e [[Arquivos de Cabeçalho e o Modelo de Execução]] mostraram o mundo do C: ponteiros livres, compilação, linkagem.

> 💡 **Segurança de memória (*memory safety*):** garantia de que um programa só acessa memória que pertence a objetos válidos, do tipo certo, dentro dos limites. Go a oferece por padrão; `unsafe` e cgo são as formas de abrir mão dela, conscientemente.

---

# Parte 1: o pacote `unsafe`

## 📐 1. Medindo o layout da memória

`unsafe.Sizeof`, `Alignof` e `Offsetof` são avaliados **em tempo de compilação** e são completamente seguros: só informam. Saída real:

```go
type Ruim struct {
	A bool  // 1 byte
	B int64 // 8 bytes (precisa de alinhamento 8)
	C bool  // 1 byte
}

type Bom struct {
	B int64
	A bool
	C bool
}
```

```
Ruim: Sizeof=24  offsets A=0 B=8 C=16  Alignof(B)=8
Bom:  Sizeof=16  offsets B=0 A=8 C=9
Sizeof string: 16 | slice: 24 | interface: 16 | ponteiro: 8
```

```
Ruim (24 bytes)                              Bom (16 bytes)
┌─┬───────┬────────┬─┬───────┐               ┌────────┬─┬─┬──────┐
│A│padding│   B    │C│padding│               │   B    │A│C│ pad  │
└─┴───────┴────────┴─┴───────┘               └────────┴─┴─┴──────┘
0 1       8        16 17     24              0        8 9 10     16
```

Os mesmos campos, **8 bytes a menos** só pela ordem: o `int64` precisa começar num múltiplo de 8, então um `bool` antes dele desperdiça 7 bytes. Num slice de milhões de structs, isso é memória e cache (ver [[Structs]] e [[Memória e Escape Analysis]]). E os tamanhos de `string` (ponteiro + tamanho), slice (ponteiro + tamanho + capacidade) e interface (duas palavras) confirmam os cabeçalhos que vimos em [[Strings em Profundidade]], [[Arrays e Slices]] e [[Interfaces]].

---

## 🔓 2. `unsafe.Pointer`: o ponteiro sem tipo

`unsafe.Pointer` é o equivalente ao `void*` do C: um ponteiro que pode ser convertido de e para **qualquer** tipo de ponteiro, contornando o sistema de tipos. A documentação lista os **únicos padrões válidos** de uso:

| # | Padrão | Exemplo |
| --- | --- | --- |
| 1 | `*T1` → `Pointer` → `*T2`, se `T2` não é maior que `T1` e os layouts são compatíveis | Ver um `float64` como `uint64` (é o que `math.Float64bits` faz) |
| 2 | `Pointer` → `uintptr` **só para imprimir/comparar** (não para voltar) | Logar um endereço |
| 3 | Aritmética: `Pointer` → `uintptr` → ± deslocamento → `Pointer`, **na mesma expressão** | Prefira `unsafe.Add` |
| 4 | Converter para `uintptr` **na chamada** de `syscall.Syscall` | Passar ponteiros ao kernel |
| 5 | Converter o resultado de `reflect.Value.Pointer`/`UnsafeAddr` **imediatamente** | — |
| 6 | Construir `reflect.SliceHeader`/`StringHeader` | **Obsoleto**: use `unsafe.Slice`/`unsafe.String` |

### 2.1 Por que `uintptr` não é ponteiro

O GC só considera **ponteiros** (`*T`, `unsafe.Pointer`) ao decidir o que está vivo, e só **eles** são atualizados quando uma stack cresce e é copiada ([[Goroutines]]). Um `uintptr` é só um número: se o objeto se mover ou for coletado, o número vira lixo. O `go vet` avisa. Saída real:

```go
x := 42
endereco := uintptr(unsafe.Pointer(&x)) // vira só um número: o GC não o rastreia
fmt.Println("endereço:", endereco)
p := (*int)(unsafe.Pointer(endereco))   // ❌ de volta, numa instrução separada
```

```
main.go:12:14: possible misuse of unsafe.Pointer
```

### 2.2 Aritmética e fatias

Desde o Go 1.17/1.20, há funções que tornam os padrões mais seguros que contas manuais com `uintptr`. Saída real:

```go
arr := [4]int32{10, 20, 30, 40}
p := unsafe.Pointer(&arr[0])
for k := range 4 {
	e := (*int32)(unsafe.Add(p, k*int(unsafe.Sizeof(arr[0])))) // aritmética de ponteiros
	fmt.Print(*e, " ")
}
// 10 20 30 40 ← percorrido com unsafe.Add
fmt.Println(unsafe.Slice(&arr[1], 2))
// [20 30]
```

| Função | Faz |
| --- | --- |
| `unsafe.Add(p, n)` | Ponteiro + n bytes |
| `unsafe.Slice(ptr, n)` | Um slice de n elementos começando em `ptr` (ex.: memória vinda do C) |
| `unsafe.SliceData(s)` | O ponteiro para o array de um slice |
| `unsafe.String(ptr, n)` / `unsafe.StringData(s)` | Uma string a partir de bytes / os bytes de uma string |

### 2.3 Conversão sem cópia, e o seu perigo

`string(b)` **copia** os bytes, porque strings são imutáveis e o slice pode mudar depois ([[Strings em Profundidade]]). Com `unsafe`, dá para criar a string **apontando para os mesmos bytes**. Saída real:

```go
func semCopia(b []byte) string { return unsafe.String(unsafe.SliceData(b), len(b)) }

b := []byte("imutável?")
s := semCopia(b)
fmt.Println("antes:", s)
b[0] = 'I' // ❌ mexer nos bytes depois da conversão
fmt.Println("depois de alterar b[0]:", s)
```

```
antes: imutável?
depois de alterar b[0]: Imutável? ← a string "imutável" mudou!
allocs: string(b) = 1 | unsafe.String = 0
```

A alocação sumiu, mas a **string mudou**, quebrando uma garantia em que o resto do programa confia: uma string usada como chave de map, por exemplo, mudaria de conteúdo **dentro** do map. Só é aceitável quando se pode **provar** que os bytes não serão alterados enquanto a string existir. E muitas vezes o compilador já faz isso sozinho: em `m[string(b)]` (busca em map) e `string(b) == "x"` (comparação), a conversão não copia.

---

## 🛡️ 3. As redes de proteção

### 3.1 `checkptr`

Compilando com `-race` (ou `-msan`, ou `-gcflags=all=-d=checkptr`), o compilador insere verificações nas conversões de `unsafe.Pointer`. Saída real, tratando 24 bytes como se fossem 512:

```go
x := new([3]int64) // 24 bytes na heap
p := (*[64]int64)(unsafe.Pointer(x)) // ❌ trata 24 bytes como 512
fmt.Println(p[40])                    // lê memória de OUTRO objeto (ou de lugar nenhum)
```

```
$ go run .
0                                                   ← "funcionou": leu memória alheia em silêncio
$ go run -race .
fatal error: checkptr: converted pointer straddles multiple allocations
```

Sem a verificação, o programa leu 8 bytes de um objeto que não era seu e imprimiu `0` como se nada fosse. É exatamente o tipo de bug (um *buffer over-read*) que em C vira falha de segurança.

### 3.2 Resumo das ferramentas

| Ferramenta | Pega |
| --- | --- |
| `go vet` (analisador `unsafeptr`) | `uintptr` convertido de volta fora do padrão |
| `checkptr` (`-race`, `-d=checkptr`) | Conversões para tipos maiores que o objeto, ponteiros desalinhados, aritmética que sai do objeto |
| `cgocheck` (padrão) | Violações das regras de ponteiros do cgo (seção 6) |
| `-asan` / `-msan` | Erros de memória no lado C (AddressSanitizer / MemorySanitizer) |

---

# Parte 2: cgo

## 🤝 4. Chamando C

Um comentário imediatamente antes de `import "C"` (o **preâmbulo**) é código C, compilado junto com o pacote. Tudo declarado nele fica acessível como `C.nome`. Saídas reais:

```go
/*
#cgo LDFLAGS: -lm
#include <math.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

static void maiusculas(char *s) {           // altera a string C no lugar
	for (; *s; s++) *s = toupper((unsigned char)*s);
}
*/
import "C"

fmt.Println("sqrt(2) pela libm:", C.sqrt(2))
// sqrt(2) pela libm: 1.4142135623730951

cs := C.CString("olá, cgo")         // COPIA para memória do C (malloc)
defer C.free(unsafe.Pointer(cs))    // ...que o GC não conhece: libere você
C.maiusculas(cs)
fmt.Println("de volta para Go:", C.GoString(cs), "| strlen:", C.strlen(cs))
// de volta para Go: OLá, CGO | strlen: 9
```

Dois detalhes da saída ilustram o abismo entre os dois mundos:

- O `toupper` do C não converteu o **`á`**: ele trabalha com **bytes** e não entende UTF-8, enquanto `strings.ToUpper` de Go trabalha com runas ([[Strings em Profundidade]])
- `strlen` contou **9** bytes para 8 caracteres, e procura o terminador `\0` (strings C não guardam o tamanho)

| Go | C | Observação |
| --- | --- | --- |
| `C.int`, `C.long`, `C.double`, `C.char` | `int`, `long`, `double`, `char` | Tamanhos do C da plataforma |
| `C.CString(s)` → `*C.char` | `char*` (malloc) | Copia; precisa de `C.free` |
| `C.GoString(p)`, `C.GoBytes(p, n)` | — | Copia de volta para memória Go |
| `C.struct_nome`, typedefs | `struct nome` | O cgo gera os tipos Go equivalentes |
| `#cgo CFLAGS:` / `LDFLAGS:` / `pkg-config:` | flags do compilador e do linker | Ex.: `-lm` para a libm |

## 📞 5. C chamando Go: `//export`

Uma função Go marcada com `//export` vira um símbolo C. Saída real:

```go
/*
extern int dobrarEmGo(int);                 // implementada em Go
static int chamarGo(int x) { return dobrarEmGo(x) + 1; }
*/
import "C"

//export dobrarEmGo
func dobrarEmGo(x C.int) C.int { return x * 2 }

fmt.Println("C chamando Go (//export):", C.chamarGo(20))
// C chamando Go (//export): 41
```

É a base de *callbacks* (bibliotecas C que chamam funções suas) e de compilar Go como biblioteca para outras linguagens (`-buildmode=c-shared`, ver [[Build, Flags e Cross-compilation]]).

---

## 📌 6. As regras de ponteiros

O GC de Go precisa saber onde estão os ponteiros para memória Go, e o C não colabora. Daí as regras:

| Regra | Por quê |
| --- | --- |
| Go pode passar ao C um ponteiro para memória Go, **se essa memória não contiver ponteiros Go** (a menos que estejam *pinned*) | O GC não enxerga ponteiros guardados do lado C |
| O C **não pode guardar** um ponteiro para memória Go depois que a chamada retorna | O objeto pode ser coletado (ou, no caso de stacks, movido) |
| Memória alocada pelo C (`malloc`) **não é gerenciada** pelo GC | Precisa de `free` (ou de um `runtime.AddCleanup` num invólucro Go, ver [[Garbage Collector]]) |
| Go pode guardar ponteiros para memória C à vontade | O GC simplesmente os ignora |

O runtime **verifica** a primeira regra em toda chamada (`GODEBUG=cgocheck=1`, o padrão). Saída real: uma struct Go cujo campo aponta para um slice Go, passada ao C:

```go
/*
typedef struct { int *dados; } Caixa;
static int primeiro(Caixa *c) { return c->dados[0]; }
*/

dados := []C.int{7, 8, 9}
caixa := C.Caixa{dados: &dados[0]}   // struct Go que contém ponteiro para memória Go
C.primeiro(&caixa)
```

```
panic: runtime error: argument of cgo function has Go pointer to unpinned Go pointer
```

A correção, desde o Go 1.21, é **fixar** (*pin*) o objeto apontado com `runtime.Pinner`: enquanto fixado, o GC não o move nem o libera. Saída real:

```go
var pin runtime.Pinner
pin.Pin(&dados[0])     // fixa o objeto: o GC não pode movê-lo nem liberá-lo
defer pin.Unpin()
caixa := C.Caixa{dados: &dados[0]}
fmt.Println("com runtime.Pinner →", C.primeiro(&caixa))
// com runtime.Pinner → 7
```

---

## 💸 7. O custo de atravessar a fronteira

### 7.1 Por chamada

Medição real (Go 1.27), uma função que só devolve o argumento:

```
chamada de função Go: 1.5 ns | chamada cgo: 40.6 ns
```

Cerca de **27×**. Para cada chamada, o runtime precisa:

```
C.f(x)  →  runtime.cgocall
   1. entersyscall: avisa o escalonador (a M vai sair do mundo Go; o P pode ser
      entregue a outra M se a chamada demorar, como numa syscall)
   2. troca para a stack do SISTEMA (g0) da thread: o C precisa de uma stack
      grande e fixa, não da stack pequena e móvel da goroutine
   3. converte da convenção de chamada de Go (registradores próprios, ABIInternal)
      para a do C (System V AMD64 ABI); salva registradores
   4. chama f
   5. desfaz tudo; exitsyscall: tenta recuperar um P
```

O Go 1.26 reduziu esse custo base em ~30%, e o Go 1.24 trouxe as anotações `#cgo noescape` e `#cgo nocallback` para funções C que não guardam ponteiros e não chamam Go de volta, permitindo ao compilador pular parte do trabalho. Mesmo assim: **não chame C em laços apertados**. Agrupe o trabalho (uma chamada que processa um array inteiro, em vez de uma por elemento).

### 7.2 Threads presas

Como uma chamada C se comporta como uma syscall, cada goroutine **dentro** de C prende uma thread. Saída real, com 50 goroutines numa função C que dorme 300 ms:

```
threads antes: 5
threads com 50 goroutines dentro de C: 54
```

É o mesmo resultado das syscalls bloqueantes em [[Scheduler do Go]]: o P migra para outra thread, mas a thread que entrou no C fica com ele. Um servidor que faz milhares de chamadas C lentas simultâneas precisa **limitar** essa concorrência (ver [[Padrões de Concorrência]]).

---

## 🧭 8. Vale a pena?

> 💡 *"Cgo is not Go."* (Dave Cheney, num texto conhecido da comunidade, e um dos *Go Proverbs*.)

| O cgo traz | Custo |
| --- | --- |
| Acesso a bibliotecas C maduras (SQLite, codecs de vídeo, OpenGL, bibliotecas do sistema) | ~40 ns por chamada; threads presas |
| — | Build mais lento; exige compilador C |
| — | Binário dinâmico; adeus `FROM scratch` (ver [[Build, Flags e Cross-compilation]]) |
| — | Compilação cruzada difícil |
| — | Bugs de memória do C podem derrubar o processo inteiro, sem stack trace de Go |
| — | Ferramentas de Go (race detector, profiler) enxergam pouco do lado C |

Alternativas, da preferida para a menos:

| Alternativa | Exemplo |
| --- | --- |
| Biblioteca em **Go puro** | `modernc.org/sqlite` (SQLite traduzido para Go), `crypto/*` da stdlib |
| Chamar a biblioteca **sem cgo**, carregando-a em execução | Projetos como `purego` (usam `dlopen` e o mecanismo de chamada do runtime) |
| Outro **processo**, conversando por IPC | Um binário auxiliar via pipes, sockets Unix ou gRPC (ver [[Sockets e o Pacote net]]) |
| WebAssembly | Rodar código C compilado para WASM dentro de um runtime WASM em Go |
| cgo | Quando nada acima serve |

---

## 🔬 9. Por dentro

O comando `go build` detecta `import "C"` e roda a ferramenta **`cgo`**, que:

1. Extrai o preâmbulo e o compila com o compilador C do sistema (gcc/clang)
2. Gera arquivos Go (`_cgo_gotypes.go`) com os tipos traduzidos e uma função de ponte para cada `C.f` usada
3. Gera arquivos C com as funções de ponte do outro lado (e os símbolos de `//export`)
4. O linker junta tudo, normalmente delegando a etapa final ao linker do sistema (*linkagem externa*), que adiciona a libc dinâmica

| Conceito de SO / C | Onde aparece |
| --- | --- |
| Convenção de chamada (ABI) | Tradução entre `ABIInternal` de Go e a System V AMD64 ABI |
| Pilha do processo/thread de tamanho fixo ([[Processos]]) | A stack `g0` de cada M, usada pelo código C |
| Bibliotecas compartilhadas ([[Sistemas Monolíticos]]) | `libc.so`, `libm.so` carregadas pelo `ld-linux.so` |
| `malloc`/`free` ([[A Linguagem C]], [[Gerenciando a Memória Livre]]) | Memória C, invisível ao GC |
| Syscalls bloqueantes ([[System Calls]]) | Chamadas cgo tratadas como syscalls pelo escalonador |
| `void*` | `unsafe.Pointer` |
| `sizeof`, `offsetof` | `unsafe.Sizeof`, `unsafe.Offsetof` |

---

## ⚠️ 10. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Guardar endereço em `uintptr` e converter depois | Ponteiro para lixo após GC ou crescimento de stack | `unsafe.Pointer`; aritmética na mesma expressão ou `unsafe.Add` |
| Converter para um tipo maior que o objeto | Leitura de memória alheia, em silêncio | Teste com `-race` (checkptr) |
| String sem cópia de bytes que mudam | "Strings imutáveis" mudando; maps corrompidos | Só com bytes comprovadamente imutáveis |
| Supor layout de struct igual ao do C | Campos desalinhados | Tipos gerados pelo cgo (`C.struct_x`) |
| Esquecer `C.free` | Vazamento de memória C (o GC não vê) | `defer C.free(...)` ou cleanup no invólucro |
| Passar struct Go com ponteiros Go ao C | Panic do `cgocheck` | `runtime.Pinner` ou memória C |
| C guardando ponteiro Go após retornar | Corrupção aleatória | Copie para memória C |
| Chamadas C em laço apertado | 40 ns de overhead cada | Agrupe o trabalho |
| Muitas chamadas C lentas simultâneas | Explosão de threads | Limite a concorrência |
| Usar `unsafe`/cgo "por desempenho" sem medir | Complexidade sem ganho | Benchmark e profile primeiro |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| `Sizeof`/`Alignof`/`Offsetof` | Seguros; revelam layout e padding (24 vs 16 bytes pela ordem dos campos) |
| `unsafe.Pointer` | `void*` de Go; só os 6 padrões documentados são válidos |
| `uintptr` | Número, não ponteiro: o GC não rastreia, a stack pode mover |
| `Add`, `Slice`, `String`, `SliceData`, `StringData` | Formas modernas e mais seguras |
| Sem cópia | Elimina alocação, mas quebra a imutabilidade se os bytes mudarem |
| `checkptr` / `vet` | Pegam abusos; rode testes com `-race` |
| cgo | Preâmbulo C + `import "C"`; `CString`/`free`; `//export` para callbacks |
| Regras de ponteiros | Sem ponteiros Go dentro do que vai ao C (salvo *pinned*); C não guarda ponteiros Go |
| `runtime.Pinner` (1.21) | Fixa objetos Go para o C usar |
| Custo | ~40 ns por chamada (vs 1,5 ns); uma thread presa por chamada em andamento |
| Quando usar | Em último caso: prefira Go puro, `purego`, processos separados |

---

## 🔗 Conexões

**Dentro de Go**
- Por que Go proíbe aritmética de ponteiros → [[Ponteiros]], [[Garbage Collector]]
- `uintptr` → [[Tipos de Dados]]
- Layout, alinhamento e padding → [[Structs]]
- Stacks que se movem → [[Goroutines]]
- Cabeçalhos de string, slice e interface → [[Strings em Profundidade]], [[Arrays e Slices]], [[Interfaces]]
- Reflection, o outro jeito de contornar o sistema de tipos → [[Reflection]]
- Chamadas cgo e threads → [[Scheduler do Go]]
- Memória C invisível ao GC e `AddCleanup` → [[Garbage Collector]], [[Memória e Escape Analysis]]
- `CGO_ENABLED`, linkagem dinâmica, `-buildmode=c-shared` → [[Build, Flags e Cross-compilation]]
- `-race` e o `checkptr` → [[Testes com go test]], [[Fundamentos de Concorrência]]
- Medir antes de recorrer a `unsafe` → [[Benchmarks]], [[Profiling e Diagnóstico]]
- Processos auxiliares via sockets → [[Sockets e o Pacote net]]

**Sistemas Operacionais**
- [[A Linguagem C]] — ponteiros livres, `malloc`/`free`, sem GC
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — `#include`, compilação e linkagem, que o cgo reutiliza
- [[Sistemas Monolíticos]] — bibliotecas compartilhadas
- [[System Calls]] — chamadas cgo tratadas como syscalls
- [[Gerenciando a Memória Livre]] — o heap do C (malloc), separado do de Go
- [[Processos]] — a stack de tamanho fixo das threads

---

⬅️ **Anterior:** [[Reflection]] · 🗺️ [[MOC - GO|Mapa de Go]] · 🏁 **Fim da trilha**
