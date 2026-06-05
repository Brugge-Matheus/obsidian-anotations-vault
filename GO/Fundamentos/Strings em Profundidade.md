---
tags:
  - go
  - go/fundamentos
---
# Strings em Profundidade

> Strings em Go são imutáveis e intrinsecamente UTF-8. Mas ao contrário de muitas linguagens, Go expõe os bytes diretamente — o que exige entender a diferença entre byte e rune para trabalhar corretamente com texto internacional.
> 

---

## 1. Representação Interna de uma String

Internamente, uma string em Go é representada como um **string header** — uma struct com dois campos:

```go
type StringHeader struct {
    Data uintptr  // ponteiro para o primeiro byte no segmento de dados
    Len  int      // comprimento em bytes
}

// Em memória:
// ┌──────────────────┬──────┐
// │ ptr → [c][a][f][é]... │ Len=5 │
// └──────────────────┴──────┘
//           ↑
//       segmento de dados (pode ser somente leitura)
```

Isso tem implicações importantes:

1. Atribuir `b = a` (onde `a` e `b` são strings) copia apenas o header (16 bytes no x86-64), não os dados
2. Slicing `s[1:3]` cria um novo header apontando para o meio do mesmo array de bytes — O(1), sem cópia
3. Strings são **imutáveis** — o segmento de dados nunca é modificado após a criação

```go
a := "hello"
b := a          // copia apenas (ptr, len) — O(1)
_ = b

s := "hello, world"
sub := s[7:12]  // novo header: ptr = s.ptr + 7, len = 5 — O(1), sem cópia
fmt.Println(sub) // "world"
```

---

## 2. UTF-8 e Por Que Go o Usa

UTF-8 é um encoding de **comprimento variável**: cada code point Unicode ocupa 1 a 4 bytes.

```
ASCII (U+0000 a U+007F):   1 byte   → 0xxxxxxx
Latin-1 etc (até U+07FF):  2 bytes  → 110xxxxx 10xxxxxx
Maioria do BMP (até U+FFFF): 3 bytes → 1110xxxx 10xxxxxx 10xxxxxx
Plano suplementar (até U+10FFFF): 4 bytes → 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
```

Go usa UTF-8 porque:

1. Arquivos `.go` são UTF-8 por definição
2. Compatibilidade com ASCII — strings ASCII são válidas sem mudanças
3. Ken Thompson e Rob Pike (criadores do Go) também criaram o UTF-8

---

## 3. Bytes vs Runes

| Conceito | Tipo em Go | Tamanho | Representa |
| --- | --- | --- | --- |
| Byte | `byte` (alias de `uint8`) | 1 byte | unidade de armazenamento |
| Rune | `rune` (alias de `int32`) | 4 bytes | um code point Unicode |

```go
s := "café"

// len() retorna BYTES, não caracteres
fmt.Println(len(s))   // 5 — 'c','a','f' ocupam 1 byte, 'é' ocupa 2 bytes (0xC3 0xA9)

// Acessar por índice retorna BYTES
fmt.Println(s[3])          // 195  (0xC3 — primeiro byte de 'é')
fmt.Println(s[4])          // 169  (0xA9 — segundo byte de 'é')

// Iterar por bytes — pode partir um caractere no meio
for i := 0; i < len(s); i++ {
	fmt.Printf("byte[%d] = %d\n", i, s[i])
}
// byte[0]=99 byte[1]=97 byte[2]=102 byte[3]=195 byte[4]=169

// Iterar por runes — correto para caracteres Unicode
// O índice é o offset de BYTES, o valor é o rune completo
for i, r := range s {
	fmt.Printf("rune[byteOffset=%d] = %c (%d)\n", i, r, r)
}
// rune[byteOffset=0] = c (99)
// rune[byteOffset=1] = a (97)
// rune[byteOffset=2] = f (102)
// rune[byteOffset=3] = é (233)
```

### Contando Caracteres Corretamente

```go
import "unicode/utf8"

s := "café"

fmt.Println(len(s))                        // 5 — bytes
fmt.Println(utf8.RuneCountInString(s))     // 4 — runes (caracteres)
fmt.Println(utf8.ValidString(s))           // true — UTF-8 válido

// Converter para []rune para indexação por caractere
runes := []rune(s)
fmt.Println(len(runes))    // 4
fmt.Println(runes[3])      // 233 (rune de 'é')
fmt.Println(string(runes[3]))  // "é"

// Inverter string — ERRADO (parte runes multibyte)
errado := s[::-1]   // não existe em Go, mas se existisse seria errado

// CORRETO — inverte por rune
func inverter(s string) string {
	runes := []rune(s)
	for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
		runes[i], runes[j] = runes[j], runes[i]
	}
	return string(runes)
}
```

---

## 4. Literais de String

### String com Aspas Duplas (Interpreted String Literal)

Processa sequências de escape:

```go
s1 := "linha 1\nlinha 2"      // \n = nova linha (U+000A)
s2 := "tab\taqui"              // \t = tabulação (U+0009)
s3 := "aspas: \"Go\""          // \" = aspas literais
s4 := "barra: \\"              // \\ = barra invertida literal
s5 := "\u00e9"                 // é — code point Unicode (4 hex digits)
s6 := "\U0001F600"             // 😀 — code point Unicode (8 hex digits)
s7 := "\xC3\xA9"               // é — bytes hexadecimais (UTF-8 de 'é')
```

### Raw String Literal (Backtick)

Tudo entre backticks é literal — sem escape processing, pode ter múltiplas linhas, pode conter qualquer byte exceto backtick:

```go
s := `Esta é uma string
que ocupa múltiplas linhas.
Barras invertidas \ são literais.
Aspas "também" não precisam de escape.`

// Muito útil para: SQL, JSON, HTML, regex
query := `
	SELECT u.id, u.nome, p.titulo
	FROM usuarios u
	JOIN posts p ON p.user_id = u.id
	WHERE u.ativo = true
	ORDER BY u.nome
`

regex := `^\d{3}-\d{4}$`   // sem precisar escapar \d

json := `{"nome": "Alice", "tags": ["go", "dev"]}`
```

> 💡 Raw strings literais preservam os bytes exatamente como escritos, incluindo espaços iniciais e finais. Para queries SQL em produção, considere `strings.TrimSpace` para remover espaço em branco desnecessário.
> 

---

## 5. Imutabilidade e Conversões

Strings são imutáveis — o compilador pode compartilhar o mesmo segmento de dados entre várias strings:

```go
s := "hello"
// s[0] = 'H'   ❌ erro de compilação: cannot assign to s[0] (strings are immutable)

// Para modificar, converta para []byte (cria uma cópia)
b := []byte(s)      // alocação nova, cópia dos bytes
b[0] = 'H'
resultado := string(b)   // nova string a partir dos bytes
fmt.Println(resultado)   // "Hello"
fmt.Println(s)           // "hello" — imutável, não foi alterado

// string → []byte → string: DUAS alocações
// O compilador elimina alocações em casos simples (escape analysis)
```

### Otimização: Conversão sem Cópia

O compilador elimina cópias desnecessárias em padrões específicos:

```go
// Estes padrões NÃO alocam:
b := []byte("hello")
if string(b) == "hello" { }        // conversão inline comparada → sem alocação
m[string(b)] = 1                   // chave de map → sem alocação (Go 1.6+)
for _, r := range string(b) { }    // range de conversão → sem alocação
```

---

## 6. Concatenação — Performance

```go
// Operador + — cria uma nova string a cada concatenação
// Para N concatenações: O(N²) tempo, O(N²) memória
resultado := ""
for i := 0; i < 1000; i++ {
	resultado += "item "   // 1000 alocações intermediárias!
}

// strings.Builder — O(N) amortizado, usa buffer que cresce como slice
var sb strings.Builder
sb.Grow(5000)   // pré-alocar capacidade estimada (opcional mas recomendado)
for i := 0; i < 1000; i++ {
	sb.WriteString("item ")
}
resultado = sb.String()   // apenas uma cópia no final

// strings.Join — quando já tem os pedaços em um slice
partes := []string{"a", "b", "c", "d"}
resultado = strings.Join(partes, ", ")   // uma única alocação

// fmt.Sprintf — conveniente mas relativamente lento (usa reflection)
resultado = fmt.Sprintf("%s, %s", "Alice", "Bob")
```

### Benchmark orientativo

```
BenchmarkPlus-8          10000   125 µs/op   (O(N²))
BenchmarkBuilder-8      100000    12 µs/op   (O(N))
BenchmarkJoin-8         100000     8 µs/op   (O(N), menos alocações)
```

---

## 7. Pacote `strings` — Funções Essenciais

```go
import "strings"

s := "  Hello, Go World!  "

// Verificações — todas O(n)
strings.Contains(s, "Go")              // true
strings.ContainsAny(s, "aeiou")        // true — contém alguma dessas letras
strings.ContainsRune(s, 'G')           // true
strings.HasPrefix(s, "  Hello")        // true
strings.HasSuffix(s, "!  ")            // true
strings.Count(s, "o")                  // 2

// Busca — retorna índice de byte, -1 se não encontrar
strings.Index(s, "Go")                 // 9
strings.LastIndex(s, "o")             // 14
strings.IndexByte(s, 'G')             // 9
strings.IndexRune(s, 'G')             // 9
strings.IndexAny(s, "ABCDEFG")        // 9 (primeiro de qualquer char do set)

// Transformações
strings.ToUpper(s)
strings.ToLower(s)
strings.ToTitle(s)                     // HELLO, GO WORLD!  (todos maiúsculos)
strings.Title(s)                       // ⚠️ Deprecated — use golang.org/x/text/cases
strings.TrimSpace(s)                   // "Hello, Go World!"
strings.Trim(s, " !")                  // "Hello, Go World"
strings.TrimLeft(s, " ")               // "Hello, Go World!  "
strings.TrimRight(s, " ")              // "  Hello, Go World!"
strings.TrimPrefix(s, "  ")            // "Hello, Go World!  "
strings.TrimSuffix(s, "  ")            // "  Hello, Go World!"
strings.TrimFunc(s, unicode.IsSpace)   // "Hello, Go World!"

// Substituição
strings.Replace(s, "o", "0", 1)       // substitui primeira ocorrência
strings.ReplaceAll(s, "o", "0")       // substitui todas

// Divisão e junção
strings.Split("a,b,c", ",")           // ["a", "b", "c"]
strings.SplitN("a,b,c", ",", 2)       // ["a", "b,c"]
strings.SplitAfter("a,b,c", ",")      // ["a,", "b,", "c"]
strings.Fields("  foo bar  baz  ")    // ["foo", "bar", "baz"]
strings.FieldsFunc(s, func(r rune) bool { return !unicode.IsLetter(r) })
strings.Join([]string{"a", "b", "c"}, "-")   // "a-b-c"

// Outras utilidades
strings.Repeat("ab", 3)               // "ababab"
strings.Map(unicode.ToUpper, "hello") // "HELLO" — transforma rune a rune
strings.Cut("user:alice", ":")        // "user", "alice", true — Go 1.18+
strings.CutPrefix("Hello, World", "Hello, ")   // "World", true — Go 1.20+
strings.CutSuffix("Hello, World", " World")    // "Hello,", true — Go 1.20+
```

---

## 8. Pacote `strconv` — Conversões

```go
import "strconv"

// int → string
strconv.Itoa(42)                         // "42"
strconv.FormatInt(255, 16)               // "ff" (base 16)
strconv.FormatInt(8, 2)                  // "1000" (base 2)
strconv.FormatInt(-42, 10)               // "-42"

// string → int (com tratamento de erro)
n, err := strconv.Atoi("123")            // 123, nil
n2, err := strconv.ParseInt("ff", 16, 64)   // 255, nil
n3, err := strconv.ParseInt("abc", 10, 32)  // 0, *NumError

// float
strconv.FormatFloat(3.14159, 'f', 2, 64)   // "3.14"
strconv.FormatFloat(3.14159, 'e', 4, 64)   // "3.1416e+00"
strconv.FormatFloat(3.14159, 'g', -1, 64)  // "3.14159" (mais curto de f/e)
f, err := strconv.ParseFloat("3.14", 64)

// bool
strconv.FormatBool(true)    // "true"
b, err := strconv.ParseBool("1")   // true (aceita: 1, t, T, TRUE, true, True)
b, err = strconv.ParseBool("0")    // false

// Append variants — escrevem no buffer sem alocação
buf := make([]byte, 0, 64)
buf = strconv.AppendInt(buf, 42, 10)      // adiciona "42" ao buffer
buf = strconv.AppendFloat(buf, 3.14, 'f', 2, 64)
```

---

## 9. Formatação com `fmt.Sprintf`

```go
nome := "Alice"
idade := 30
altura := 1.75

s := fmt.Sprintf("Nome: %s, Idade: %d, Altura: %.2f", nome, idade, altura)
```

### Verbos de Formatação

| Verbo | Para | Exemplo |
| --- | --- | --- |
| `%s` | string | `"hello"` |
| `%q` | string com aspas e escapes | `"\"hello\\n\""` |
| `%d` | inteiro decimal | `42` |
| `%b` | inteiro binário | `101010` |
| `%o` | inteiro octal | `52` |
| `%x` | inteiro hexadecimal (minúsculas) | `2a` |
| `%X` | inteiro hexadecimal (maiúsculas) | `2A` |
| `%f` | float decimal | `3.140000` |
| `%.2f` | float com 2 casas | `3.14` |
| `%e` | float notação científica | `3.14e+00` |
| `%g` | float mais curto entre f/e | `3.14` |
| `%t` | bool | `true` |
| `%c` | rune (caractere) | `'A'` |
| `%v` | valor padrão de qualquer tipo | varia |
| `%+v` | struct com nomes dos campos | `{Nome:Alice Idade:30}` |
| `%#v` | representação Go syntax | `main.Pessoa{Nome:"Alice"}` |
| `%T` | tipo do valor | `int`, `main.Usuario` |
| `%p` | ponteiro (endereço hex) | `0xc000018050` |

```go
// Largura e alinhamento
fmt.Sprintf("%10s", "Go")     // "        Go" — alinha à direita
fmt.Sprintf("%-10s|", "Go")  // "Go        |" — alinha à esquerda
fmt.Sprintf("%010d", 42)      // "0000000042" — preenche com zeros
fmt.Sprintf("%+d", 42)        // "+42" — força sinal

// Para strings muito performáticas, fmt.Sprintf é 3-5x mais lento que strconv
// Use strconv.AppendInt/AppendFloat para hot paths
```

---

## 10. Normalização Unicode (Go 1.21+)

Para comparação correta de texto internacional, use o pacote `golang.org/x/text`:

```go
import "golang.org/x/text/unicode/norm"

// "é" pode ser representado de duas formas:
// NFC: U+00E9 (precomposed — 'é' como um único code point)
// NFD: U+0065 + U+0301 (decomposed — 'e' + combining accent)

s1 := "caf\u00e9"       // NFC: café (4 runes)
s2 := "cafe\u0301"      // NFD: café (5 runes, mesmo visual)

fmt.Println(s1 == s2)   // false! — bytes diferentes

// Normalizar antes de comparar
n1 := norm.NFC.String(s1)
n2 := norm.NFC.String(s2)
fmt.Println(n1 == n2)   // true

// strings.EqualFold — case-insensitive, mas não normaliza Unicode
strings.EqualFold("café", "CAFÉ")   // true (funciona para maioria dos casos)
```

---

## 11. Resumo — Quando Usar Cada Abordagem

| Situação | Abordagem |
| --- | --- |
| Concatenar poucas strings (≤ 5) | `+` |
| Concatenar muitas strings em loop | `strings.Builder` |
| Juntar slice de strings | `strings.Join` |
| Formatar com variáveis (código normal) | `fmt.Sprintf` |
| Formatar (hot path, performance) | `strconv.AppendInt/Float` |
| Converter número → string | `strconv.Itoa`, `strconv.FormatFloat` |
| Converter string → número | `strconv.Atoi`, `strconv.ParseFloat` |
| Contar caracteres Unicode | `utf8.RuneCountInString` |
| Iterar caractere a caractere | `for i, r := range s` |
| Modificar caracteres | `[]rune(s)` → modificar → `string(runes)` |
| Dividir string por delimitador | `strings.Split`, `strings.Cut` (Go 1.18+) |
| Verificar prefixo/sufixo | `strings.HasPrefix`, `strings.CutPrefix` |

---

## Conexão com Sistemas Operacionais

### String header (ptr + len) — [[Processos]]

A representação interna de uma string (`StringHeader{Data uintptr; Len int}`, 16 bytes) é estruturalmente idêntica à metade de um slice header. Em [[Processos]] estudamos como o SO representa buffers e regiões de memória como par `(ponteiro, tamanho)` — por exemplo, ao fazer uma syscall `write(fd, buf, n)`. O Go segue o mesmo padrão: toda string é um ponteiro para dados mais um comprimento, sem terminador nulo (diferente de C). Slicing é O(1) porque cria apenas um novo par `(ptr+offset, novoLen)` apontando para os mesmos bytes.

### UTF-8 como codificação de bytes — [[Bits e Bytes]]

Em [[Bits e Bytes]] estudamos como bytes representam informação binária. O encoding UTF-8 mostrado na seção 2 é a aplicação direta desse modelo: cada code point Unicode é codificado em 1 a 4 bytes usando prefixos de bits para indicar o comprimento da sequência (`0xxxxxxx` para ASCII, `110xxxxx 10xxxxxx` para dois bytes, etc.). Um `byte` em Go é um `uint8` — exatamente 8 bits — e a aritmética binária que determina se um byte é "líder" ou "continuação" de um rune UTF-8 é operação de bits pura.

### Imutabilidade = segmento de dados protegido — [[Processos]]

Strings são imutáveis em Go: `s[0] = 'H'` é erro de compilação. O motivo mais profundo é que literais de string são armazenados em segmentos de dados **somente leitura** do binário. Em [[Processos]] vimos que o mapa de memória de um processo inclui segmentos com permissões diferentes — o segmento de código (`.text`) e o segmento de dados somente leitura (`.rodata`) são mapeados sem permissão de escrita. Tentar escrever neles gera um page fault que o SO converte em SIGSEGV. A imutabilidade de string em Go reflete essa proteção de hardware.

### Literais de string em `.rodata` — [[Arquivos de Cabeçalho e o Modelo de Execução]]

Em [[Arquivos de Cabeçalho e o Modelo de Execução]] estudamos os segmentos de um binário ELF: `.text` para código, `.data` para variáveis globais inicializadas, `.bss` para variáveis globais não inicializadas. Literais de string compilados (como `"hello"`) ficam no segmento **`.rodata`** (read-only data) — são carregados na memória durante a inicialização do processo e nunca modificados. Quando duas variáveis recebem o mesmo literal, o compilador pode apontar ambos para o mesmo endereço em `.rodata`, sem duplicar os bytes.

### String interning — [[Espaços de Endereçamento]]

O fato de que o compilador pode compartilhar um mesmo literal de string entre múltiplas variáveis é análogo ao conceito de **shared memory** em [[Espaços de Endereçamento]]: múltiplos endereços virtuais podem mapear para a mesma página física (read-only). No contexto de strings, dois ponteiros diferentes contendo o mesmo endereço de `.rodata` são "aliased" — acessos de leitura são seguros porque a imutabilidade garante que nenhuma escrita vai ocorrer. Essa é a mesma razão pela qual bibliotecas compartilhadas (`.so`/`.dylib`) podem ser mapeadas somente-leitura em múltiplos processos ao mesmo tempo.

### Conversão `[]byte` → `string` = alocação na heap — [[Processos]]

Converter `[]byte` para `string` (e vice-versa) cria uma cópia dos bytes na heap (exceto nas otimizações inline do compilador). Em [[Processos]] vimos que toda alocação na heap passa pelo alocador do runtime, que eventualmente chama `mmap()` para obter páginas do SO. É por isso que código de alto throughput em Go evita essas conversões em hot paths: cada `string(b)` pode resultar em uma alocação de heap e pressão no GC.

### I/O usa `[]byte` — [[Arquivos]]

Em [[Arquivos]] e [[System Calls]] vimos que as syscalls de I/O (`read`, `write`) operam em buffers de bytes crus — o kernel não sabe nada sobre strings ou encodings. Por isso todas as interfaces de I/O em Go (`io.Reader`, `io.Writer`) usam `[]byte` como tipo fundamental. Uma `string` precisa ser convertida para `[]byte` antes de ser escrita em um arquivo ou socket, porque o kernel espera um ponteiro para bytes e um tamanho — exatamente o par `(unsafe.Pointer, len)` que um slice de bytes fornece ao ser passado para a syscall.