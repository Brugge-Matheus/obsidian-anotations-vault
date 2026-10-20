---
tags:
  - go
  - go/tipos-compostos
aliases:
  - Strings em Go
  - Runes
  - UTF-8 em Go
  - strings.Builder
  - fmt verbos
ordem: 12
modulo: "03 - Tipos Compostos"
---
# Strings em Profundidade

> **Nota 12 da trilha de Go.** Abrimos o módulo de tipos compostos com o tipo que você mais vai usar: `string`. Em Go, uma string é uma **sequência imutável de bytes**, normalmente texto em **UTF-8**. Entender a diferença entre **byte** e **rune** resolve o mistério do "João" com 5 letras, evita bugs com acentos e emojis e explica as escolhas de performance (`strings.Builder`, conversões, imutabilidade). Também veremos os pacotes `strings`, `strconv`, `unicode/utf8` e os verbos de formatação do `fmt`.
>
> **Pré-requisitos:** [[Tipos de Dados]], [[Controle de Fluxo]], [[Ponteiros]].

---

## 🧭 De onde viemos

No [[Ambiente e Primeiro Programa]], `len("João")` retornou **5**. Em [[Tipos de Dados]] vimos que `byte` é `uint8`, `rune` é `int32` e que `unsafe.Sizeof("")` é 16 bytes, não importa o tamanho do texto. Em [[Controle de Fluxo]] o `range` sobre `"café"` pulou do índice 3 direto para o fim. Em [[Ponteiros]] vimos que alguns tipos são "cabeçalhos" que apontam para dados em outro lugar. Esta nota junta todas essas pistas.

---

## 🔤 1. Texto no computador: de caracteres a bytes

O computador só guarda números. Para guardar texto, é preciso uma tabela que associe cada caractere a um número, e uma forma de gravar esse número em bytes.

> 💡 **Unicode:** o padrão que atribui um número único, chamado **code point** (ponto de código), a cada caractere de todos os sistemas de escrita: `A` = U+0041, `é` = U+00E9, `ã` = U+00E3, `世` = U+4E16, `😀` = U+1F600. São mais de 150 mil caracteres.

> 💡 **Codificação (encoding):** a regra para transformar code points em bytes. **UTF-8** é a codificação dominante (quase 99% da web) e a usada por Go.

### 1.1 Como o UTF-8 funciona

UTF-8 usa de **1 a 4 bytes** por caractere. Os bits do primeiro byte dizem quantos bytes o caractere tem:

```
Faixa de code points       Bytes   Padrão de bits
─────────────────────────  ─────   ───────────────────────────────────────
U+0000  a U+007F (ASCII)     1     0xxxxxxx
U+0080  a U+07FF             2     110xxxxx 10xxxxxx
U+0800  a U+FFFF             3     1110xxxx 10xxxxxx 10xxxxxx
U+10000 a U+10FFFF           4     11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
                                   ▲        ▲
                                   │        └ bytes de continuação começam com 10
                                   └ o primeiro byte diz o tamanho
```

Exemplo: `é` = U+00E9 = `1110 1001` em binário (8 bits significativos) → cabe na faixa de 2 bytes:

```
U+00E9 = 000 1110 1001
          ───┬─  ──┬───
            00011   101001
110 00011 = 0xC3      10 101001 = 0xA9      →   "é" = C3 A9
```

Por que UTF-8 é tão bom (e por que Go o adotou):

1. **Compatível com ASCII:** todo texto ASCII já é UTF-8 válido, byte a byte
2. **Autossincronizável:** olhando qualquer byte, dá para saber se é início de caractere ou continuação
3. **Sem bytes nulos:** nenhum caractere além do NUL gera o byte `0x00`, então funciona com funções de C
4. **Inventado por Ken Thompson e Rob Pike** em 1992, num jantar (desenharam num guardanapo). Os mesmos dois criaram o Go

Você estudou a base binária disso em [[Bits e Bytes]].

---

## 📦 2. O que é uma string em Go

> **Uma `string` em Go é uma sequência imutável de bytes.** Por convenção (e porque o código-fonte Go é UTF-8), ela costuma conter texto UTF-8, mas pode conter **quaisquer bytes**.

### 2.1 O cabeçalho de 16 bytes

Por baixo, uma string é uma estrutura de duas palavras:

```
variável s := "café"                      dados (em .rodata ou na heap)
┌────────────────────┐                    ┌────┬────┬────┬────┬────┐
│ ptr  ──────────────┼──────────────────▶ │ 63 │ 61 │ 66 │ C3 │ A9 │
├────────────────────┤                    └────┴────┴────┴────┴────┘
│ len  = 5           │                       c    a    f    é (2 bytes)
└────────────────────┘
  16 bytes em amd64 (8 do ponteiro + 8 do tamanho)
```

Consequências diretas:

| Operação | Custo | Por quê |
| --- | --- | --- |
| `b := a` | O(1), copia 16 bytes | Copia só o cabeçalho; os dois apontam para os mesmos bytes |
| Passar string para função | O(1) | Idem (Go passa por valor, mas o valor é o cabeçalho) |
| `len(s)` | O(1) | O tamanho está guardado no cabeçalho |
| `s[2:5]` (fatiar) | O(1), sem cópia | Novo cabeçalho apontando para o meio dos mesmos bytes |
| `a == b` | O(n) | Compara os bytes (mas tamanhos diferentes saem na hora) |
| `a + b` | O(n), aloca | Precisa de uma área nova com os dois conteúdos |

> 💡 Diferente de C, a string de Go **não termina com `\0`**: o tamanho está no cabeçalho. Por isso `len` é O(1) (em C, `strlen` percorre tudo) e uma string pode conter o byte zero no meio.

### 2.2 Imutabilidade

```go
s := "hello"
s[0] = 'H'        // ❌ cannot assign to s[0] (neither addressable nor a map index expression)
s = "Hello"       // ✅ isto NÃO altera os bytes: faz s apontar para outra string
```

Uma vez criados, os bytes de uma string **nunca mudam**. Isso permite:

- **Compartilhar sem medo:** `b := a` e o fatiamento são seguros, ninguém altera os bytes do outro
- **Usar como chave de map** com segurança (a chave não pode mudar depois de inserida)
- **Literais em memória somente-leitura:** os literais ficam na seção `.rodata` do binário, mapeada sem permissão de escrita (ver [[Arquivos de Cabeçalho e o Modelo de Execução]] e [[Proteção]])
- **Segurança em concorrência:** várias goroutines podem ler a mesma string sem sincronização

---

## 🔠 3. Bytes vs runes

Este é o coração da nota:

| Conceito | Tipo em Go | O que representa |
| --- | --- | --- |
| **byte** | `byte` (= `uint8`) | Uma unidade de armazenamento (8 bits) |
| **rune** | `rune` (= `int32`) | Um **code point** Unicode (um "caractere") |

> 💡 **Rune:** nome que Go dá a um code point Unicode. O nome vem do Plan 9 (sistema da Bell Labs onde Pike e Thompson trabalharam). Uma rune é só um `int32` com o número do caractere: `'é'` vale 233.

### 3.1 `len` conta bytes; indexar devolve bytes

Saída real:

```go
s := "café"
fmt.Println(len(s))                    // 5   ← bytes, não caracteres
fmt.Println(utf8.RuneCountInString(s)) // 4   ← caracteres (runes)
fmt.Println(s[3], s[4])                // 195 169   ← os 2 bytes do "é"
fmt.Println(s[0:3])                    // caf
fmt.Printf("% x\n", s)                 // 63 61 66 c3 a9
```

E o mistério do primeiro programa:

```go
fmt.Println(len("João"), utf8.RuneCountInString("João"))   // 5 4
fmt.Println(len("😀"), len([]rune("😀")))                   // 4 1
```

> ⚠️ `string(s[3])` **não** é "a metade do é". `s[3]` é o byte 195, e converter um número para string o interpreta como code point: U+00C3 = `Ã`. Saída real: `string(s[3])` → `"Ã"`.

### 3.2 Duas formas de percorrer

```go
s := "café"

// 1) Por BYTE: índice clássico
for i := 0; i < len(s); i++ {
	fmt.Printf("%d ", s[i])        // 99 97 102 195 169
}

// 2) Por RUNE: range decodifica o UTF-8
for i, r := range s {
	fmt.Printf("%d:%c ", i, r)     // 0:c 1:a 2:f 3:é
}
```

O `range` sobre string faz duas coisas: **decodifica** cada rune e devolve o **índice do byte** onde ela começa (por isso pula de 3 para o fim).

```
bytes:   [63] [61] [66] [C3 A9]
índice:   0    1    2    3  4
range:   'c'  'a'  'f'  'é'  ← um passo por RUNE; i = 0, 1, 2, 3
```

### 3.3 UTF-8 inválido não quebra o `range`

Como uma string pode conter qualquer byte, ela pode ter UTF-8 inválido. O `range` substitui cada byte inválido por `U+FFFD` (�, o "caractere de substituição") e avança 1 byte (saída real):

```go
ruim := string([]byte{0x63, 0xff, 0x61})
for i, r := range ruim {
	fmt.Printf("%d:%q ", i, r)      // 0:'c' 1:'�' 2:'a'
}
fmt.Println(utf8.ValidString(ruim)) // false
```

> ✅ Valide texto que vem de fora (arquivos, rede, banco) com `utf8.ValidString` quando o conteúdo precisa ser UTF-8 correto.

### 3.4 `[]rune` para indexar por caractere

Para acessar "o caractere número N" ou inverter um texto, converta para `[]rune` (saída real):

```go
r := []rune("ação")
for i, j := 0, len(r)-1; i < j; i, j = i+1, j-1 {
	r[i], r[j] = r[j], r[i]
}
fmt.Println(string(r))     // oãça
```

> ⚠️ `[]rune(s)` aloca 4 bytes por caractere e percorre a string inteira. Para textos grandes, prefira o `range` ou as funções de `unicode/utf8`.

### 3.5 Caractere visível ≠ rune

Um caractere visível na tela (o que o Unicode chama de **grafema**) pode ser formado por **várias runes**. Saída real:

```go
e := "e\u0301"                        // "e" + acento agudo combinante
fmt.Println(e, len(e), utf8.RuneCountInString(e))   // é 3 2
fmt.Println("caf\u00e9" == "cafe\u0301")            // false  ← mesma aparência, bytes diferentes!
```

```
"é" pré-composto (NFC):   U+00E9               → C3 A9          (1 rune)
"é" decomposto  (NFD):    U+0065 + U+0301      → 65 CC 81       (2 runes)
```

Emojis com tom de pele ou famílias (👨‍👩‍👧) chegam a ter 5+ runes. Para comparar texto vindo de fontes diferentes, **normalize** com `golang.org/x/text/unicode/norm` (`norm.NFC.String(s)`). Para contar grafemas, use uma biblioteca como `github.com/rivo/uniseg`.

| Pergunta | Ferramenta |
| --- | --- |
| Quantos bytes ocupa? | `len(s)` |
| Quantos code points? | `utf8.RuneCountInString(s)` |
| Quantos "caracteres que a pessoa vê"? | biblioteca de grafemas (`uniseg`) |

---

## ✍️ 4. Literais de string

### 4.1 String interpretada: `"..."`

Aceita sequências de escape:

```go
"linha 1\nlinha 2"      // \n nova linha   \t tab   \r retorno de carro
"aspas: \"Go\""         // \" aspas        \\ barra invertida
"\u00e9"                // é  (code point com 4 dígitos hex)
"\U0001F600"            // 😀 (code point com 8 dígitos hex)
"\xC3\xA9"              // é  (bytes em hex: UTF-8 de é)
"\101"                  // A  (byte em octal)
```

### 4.2 String bruta (raw): `` `...` ``

Tudo entre crases é literal: sem escapes, pode ter várias linhas.

```go
query := `
	SELECT id, nome
	FROM usuarios
	WHERE ativo = true`

regex := `^\d{3}\.\d{3}\.\d{3}-\d{2}$`   // CPF: sem escapar cada \
caminho := `C:\Users\matheus\go`
json := `{"nome": "Ana", "tags": ["go"]}`
```

> 💡 Strings brutas não podem conter crase. E o `\r` de arquivos Windows dentro delas é descartado pelo compilador.

### 4.3 Literal de rune: `'...'`

```go
'a'        // rune 97
'é'        // rune 233
'\n'       // rune 10
'\u00e9'   // rune 233
"a"        // ⚠️ isto é uma STRING de 1 byte, não uma rune
```

---

## 🔁 5. Conversões e quando elas copiam

| Conversão | Resultado | Custo |
| --- | --- | --- |
| `[]byte(s)` | Cópia mutável dos bytes | Aloca e copia O(n) |
| `string(b)` | String com cópia dos bytes | Aloca e copia O(n) |
| `[]rune(s)` | Decodifica para code points | Aloca 4 bytes/rune |
| `string(r)` com `r rune` | String com a rune codificada em UTF-8 | Pequena alocação |
| `string(runes)` | Codifica de volta para UTF-8 | Aloca |

Por que copiar? Porque string é imutável e slice é mutável. Se `[]byte(s)` compartilhasse a memória, alterar o slice alteraria a string "imutável".

Para modificar uma string, o caminho é: converter, alterar, converter de volta:

```go
b := []byte("hello")
b[0] = 'H'
s := string(b)       // "Hello"
```

### 5.1 Otimizações do compilador (conversões sem cópia)

O compilador **evita a cópia** quando consegue provar que ela não é necessária:

```go
if string(b) == "GET" { }        // comparação: não aloca
m[string(b)]                     // busca em map com chave []byte: não aloca
for i, r := range string(b) { }  // range: não aloca
s := "a" + string(b) + "c"       // concatenação: converte direto no resultado
```

Para os casos que o compilador não otimiza, e quando você **garante** que os bytes nunca mais mudam, existe a conversão sem cópia de `unsafe` (`unsafe.String(&b[0], len(b))`). Use só em código crítico e bem medido (ver [[unsafe e cgo]]).

---

## ➕ 6. Construindo strings: concatenação e `strings.Builder`

Cada `+` cria uma string nova. Num laço, isso é **O(n²)**:

```go
// ❌ cada volta copia tudo o que já foi acumulado
s := ""
for i := 0; i < 1000; i++ {
	s += "item "
}

volta 1: copia 5 bytes     volta 2: copia 10     ...     volta 1000: copia 5000
total ≈ 5 × (1 + 2 + ... + 1000) ≈ 2,5 milhões de bytes copiados
```

O jeito certo é `strings.Builder`, que acumula num buffer interno (um `[]byte` que cresce como um slice) e só no final entrega a string, **sem cópia extra** (saída real):

```go
var sb strings.Builder
sb.Grow(64)                          // opcional: pré-aloca se você sabe o tamanho
for i := range 3 {
	fmt.Fprintf(&sb, "item%d;", i)   // Builder implementa io.Writer
}
sb.WriteString("fim")
sb.WriteByte('!')
sb.WriteRune('✓')
fmt.Println(sb.String())             // item0;item1;item2;fim!✓
```

| Situação | Ferramenta |
| --- | --- |
| Juntar 2 a 5 strings numa expressão | `a + b + c` (o compilador faz uma única alocação) |
| Montar texto num laço | `strings.Builder` |
| Juntar um slice de strings com separador | `strings.Join(partes, ", ")` |
| Montar com formatação | `fmt.Sprintf` / `fmt.Fprintf(&sb, ...)` |
| Montar bytes (para I/O) | `bytes.Buffer` ou `append` em `[]byte` |

> ⚠️ Não copie um `strings.Builder` depois de usar (passe `*strings.Builder`); ele detecta e dá panic, porque duas cópias compartilhariam o mesmo buffer.

---

## 🧰 7. Pacote `strings` — o canivete suíço

Todas as funções recebem e devolvem strings novas (a original nunca muda).

```go
s := "  Olá, Go!  "

// Busca e verificação
strings.Contains(s, "Go")            // true
strings.ContainsAny(s, "xyz!")       // true
strings.HasPrefix(s, "  Olá")        // true
strings.HasSuffix(s, "!  ")          // true
strings.Index(s, "Go")               // índice em BYTES (ou -1)
strings.Count("banana", "a")         // 3
strings.EqualFold("AÇÃO", "ação")    // true (sem diferenciar maiúsculas, com Unicode)

// Limpeza
strings.TrimSpace(s)                 // "Olá, Go!"
strings.Trim(s, " !")                // "Olá, Go"
strings.TrimPrefix("v1.2", "v")      // "1.2"
strings.TrimSuffix("foto.png", ".png")

// Transformação
strings.ToUpper("ação")              // "AÇÃO"  (Unicode correto)
strings.ToLower("GO")                // "go"
strings.ReplaceAll("a-b-c", "-", "+")
strings.Repeat("=", 20)
strings.Map(func(r rune) rune { ... }, s)    // transforma rune a rune

// Divisão
strings.Split("a,b,,c", ",")         // ["a" "b" "" "c"]   (4 partes: inclui a vazia)
strings.Fields("  a  b   c ")        // ["a" "b" "c"]      (divide por espaços, ignora vazios)
strings.Join([]string{"a", "b"}, "-")

// Cut: divide em DUAS partes no primeiro separador (Go 1.18+)
antes, depois, achou := strings.Cut("chave=valor", "=")   // "chave" "valor" true
strings.CutPrefix("Bearer abc123", "Bearer ")             // "abc123" true
strings.CutLast("a/b/c", "/")                             // Go 1.27+: "a/b" "c" true
```

### 7.1 Versões que retornam iteradores (Go 1.24+)

Em vez de montar um slice inteiro, estas funções devolvem um **iterador** consumido com `range` (economizam memória em textos grandes). Saída real:

```go
for linha := range strings.Lines("um\ndois\ntrês") {
	fmt.Printf("%q ", linha)     // "um\n" "dois\n" "três"
}
for parte := range strings.SplitSeq("a,b,c", ",") {
	fmt.Print(parte, " ")        // a b c
}
// também: strings.FieldsSeq, strings.FieldsFuncSeq, strings.SplitAfterSeq
```

Veremos como iteradores funcionam em [[Iteradores]].

> 💡 O pacote `bytes` tem **as mesmas funções** para `[]byte` (`bytes.Contains`, `bytes.Split`, `bytes.Buffer`...). Use-o quando os dados já estão em bytes (leitura de arquivo, rede) para evitar conversões.

---

## 🔢 8. Pacote `strconv` — strings ↔ números

Saídas reais:

```go
strconv.Itoa(42)                         // "42"
strconv.FormatInt(255, 2)                // "11111111" (qualquer base de 2 a 36)
strconv.FormatFloat(3.14159, 'f', 2, 64) // "3.14"
strconv.Quote("olá\n")                   // "\"olá\\n\"" (com aspas e escapes)

n, err := strconv.Atoi("123")            // 123, nil
_, err = strconv.Atoi("12a")
fmt.Println(err)                         // strconv.Atoi: parsing "12a": invalid syntax

v, err := strconv.ParseInt("ff", 16, 64) // 255
f, err := strconv.ParseFloat("3.14", 64)
b, err := strconv.ParseBool("true")      // aceita 1, t, T, TRUE, true, True (e os falsos)

// Versões "Append": escrevem num []byte existente, sem alocar uma string
buf := make([]byte, 0, 32)
buf = strconv.AppendInt(buf, 42, 10)
```

> ✅ `strconv` é bem mais rápido que `fmt.Sprintf("%d", n)` para conversões simples. Use `fmt` quando a legibilidade importa mais (a maior parte do código).

---

## 🖨️ 9. Formatação com `fmt`

As funções `fmt.Printf` (imprime), `fmt.Sprintf` (devolve string), `fmt.Fprintf` (escreve num `io.Writer`) e `fmt.Errorf` (cria erro) usam **verbos**. Saídas reais:

```go
type P struct {
	Nome  string
	Idade int
}
p := P{"Ana", 30}
fmt.Printf("%v | %+v | %#v | %T\n", p, p, p, p)
// {Ana 30} | {Nome:Ana Idade:30} | main.P{Nome:"Ana", Idade:30} | main.P

fmt.Printf("[%5s][%-5s][%05d][%x][%X][%08b][%c][%U][%.3s]\n",
	"go", "go", 42, "hi", 255, 5, 'é', 'é', "golang")
// [   go][go   ][00042][6869][FF][00000101][é][U+00E9][gol]
```

| Verbo | Significado |
| --- | --- |
| `%v` | Valor no formato padrão (serve para qualquer tipo) |
| `%+v` | Como `%v`, com nomes dos campos de structs |
| `%#v` | Sintaxe Go (ótimo para depurar) |
| `%T` | Tipo do valor |
| `%d` `%b` `%o` `%x` `%X` | Inteiro em decimal, binário, octal, hexa |
| `%f` `%.2f` `%e` `%g` | Float: decimal, 2 casas, científico, o mais curto |
| `%s` `%q` | String; string entre aspas com escapes |
| `%c` `%U` | Rune como caractere; como `U+XXXX` |
| `%t` | bool |
| `%p` | Ponteiro (endereço) |
| `%w` | Só em `fmt.Errorf`: embrulha um erro (ver [[Wrapping e Unwrapping de Erros]]) |
| `%5s` `%-5s` `%05d` | Largura, alinhamento à esquerda, preencher com zeros |

> 💡 Se um tipo tem o método `String() string`, o `fmt` usa esse método para `%v` e `%s` (interface `fmt.Stringer`, veremos em [[Interfaces]]). Foi o que fizemos com o enum `Nivel` em [[Variáveis e constantes]].

> ⚠️ Verbo incompatível com o tipo não é erro de compilação: aparece como `%!d(string=texto)` na saída. O `go vet` detecta esses erros, por isso ele roda junto com `go test`.

---

## 🔬 10. Por dentro: memória, interning e o fim do `StringHeader`

### 10.1 Onde os bytes de uma string moram

```
const/literal "café"      →  seção .rodata do binário (somente leitura)
string(b), a + b, Sprintf →  heap (alocada pelo runtime)
s[2:5]                    →  os mesmos bytes da string original (sem alocar)
```

> ⚠️ **Retenção de memória por fatiamento:** `pequena := enorme[0:10]` mantém **todos** os bytes de `enorme` vivos enquanto `pequena` existir (o GC não libera "parte" de uma alocação). Se você guarda um pedaço pequeno de uma string gigante (ex.: um campo de um arquivo de 100 MB), faça uma cópia: `strings.Clone(enorme[0:10])`.

### 10.2 Interning: uma cópia para valores repetidos

Quando muitas strings iguais são criadas (nomes de host, rótulos, chaves repetidas lidas de arquivo), cada uma ocupa memória própria. O pacote **`unique`** (Go 1.23+) faz *interning*: guarda uma única cópia e devolve um "identificador" barato de comparar:

```go
h1 := unique.Make("us-east-1")
h2 := unique.Make("us-east-1")
fmt.Println(h1 == h2)       // true, e a comparação é só de ponteiros (O(1))
fmt.Println(h1.Value())     // "us-east-1"
```

### 10.3 O cabeçalho na API

Antigamente usava-se `reflect.StringHeader` para mexer no cabeçalho. Ele foi **descontinuado** (Go 1.20) por ser fácil de usar errado. Hoje existem funções em `unsafe`:

```go
p := unsafe.StringData(s)          // *byte: ponteiro para o primeiro byte
s2 := unsafe.String(p, len(s))     // monta uma string a partir de ponteiro + tamanho
```

### 10.4 Strings e syscalls

O kernel não conhece strings de Go: as syscalls `read`/`write` recebem um **ponteiro e um tamanho**. Por isso a biblioteca de I/O trabalha com `[]byte`, e escrever uma string num arquivo implica, em algum ponto, entregar ao kernel o par (ponteiro, tamanho) dos bytes dela. Quando um caminho de arquivo é passado ao kernel (que espera uma string de C terminada em `\0`), o runtime precisa **copiar** a string e acrescentar o zero final. Você viu essa interface em [[Syscalls para Gerenciamento de Arquivos]].

---

## ⚠️ 11. Armadilhas

| Armadilha | Exemplo | Correção |
| --- | --- | --- |
| `len` como número de caracteres | `len("João") == 5` | `utf8.RuneCountInString` |
| Indexar esperando caractere | `s[3]` é um byte | `range` ou `[]rune(s)` |
| `string(byte)` em byte de UTF-8 | `string(s[3])` vira `"Ã"` | Trabalhe com runes |
| Fatiar no meio de um caractere | `"café"[0:4]` corta o `é` ao meio | Use índices do `range` ou `utf8` |
| Concatenar em laço | `s += x` mil vezes | `strings.Builder` |
| Comparar texto de fontes diferentes | NFC ≠ NFD | Normalize antes |
| `strings.Title` | Descontinuado | `golang.org/x/text/cases` |
| Reter string gigante via fatia | `enorme[:10]` guardado | `strings.Clone` |
| `string(65)` | `"A"` | `strconv.Itoa(65)` |
| Verbo errado no `Printf` | `%!d(string=...)` | Rode `go vet` |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| `string` | Sequência **imutável** de bytes; normalmente UTF-8 |
| Cabeçalho | Ponteiro + tamanho, 16 bytes; sem `\0` final |
| Cópia e fatiamento | O(1), compartilham os bytes |
| UTF-8 | 1 a 4 bytes por code point; compatível com ASCII |
| `byte` vs `rune` | Unidade de armazenamento vs code point (`int32`) |
| `len(s)` | Bytes; caracteres com `utf8.RuneCountInString` |
| `s[i]` | Um byte |
| `for i, r := range s` | Decodifica runes; `i` é o índice do byte |
| Grafema | Pode ter várias runes; normalização e `uniseg` |
| Literais | `"..."` com escapes, `` `...` `` bruta, `'x'` rune |
| Conversões | `[]byte(s)`, `string(b)`, `[]rune(s)` copiam |
| Montar texto | `strings.Builder`, `strings.Join` |
| Pacotes | `strings`, `bytes`, `strconv`, `unicode/utf8`, `fmt`, `unique` |

---

## 🔗 Conexões

**Dentro de Go**
- Slices, o primo mutável das strings → [[Arrays e Slices]]
- Strings como chave de map → [[Maps]]
- `fmt.Stringer` → [[Interfaces]]
- Iteradores como `strings.Lines` → [[Iteradores]]
- Leitura de texto de arquivos (`bufio.Scanner`) → [[Leitura e Escrita de Arquivos]]
- `%w` em `fmt.Errorf` → [[Wrapping e Unwrapping de Erros]]
- Conversões sem cópia com `unsafe` → [[unsafe e cgo]]

**Sistemas Operacionais**
- [[Bits e Bytes]] — a base binária da codificação UTF-8
- [[Arquivos de Cabeçalho e o Modelo de Execução]] e [[Proteção]] — literais em `.rodata`, mapeado sem permissão de escrita
- [[Syscalls para Gerenciamento de Arquivos]] — `read`/`write` trabalham com (ponteiro, tamanho), o mesmo formato do cabeçalho de string
- [[A Linguagem C]] — strings de C terminadas em `\0` vs strings de Go com tamanho explícito

**Redes**
- [[Endereços, URLs e Domínios]] — URLs, cabeçalhos HTTP e nomes de domínio são strings; domínios internacionais usam Punycode para caber em ASCII
- [[Camada de Apresentação - 6]] — codificação de caracteres é uma responsabilidade clássica da camada de apresentação

**Algoritmos e Estruturas de Dados**
- [[Tries]] — estrutura especializada em busca por prefixo de strings

---

⬅️ **Anterior:** [[Ponteiros]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Arrays e Slices]] ➡️
