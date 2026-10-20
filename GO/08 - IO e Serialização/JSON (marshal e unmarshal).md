---
tags:
  - go
  - go/io
aliases:
  - JSON em Go
  - encoding/json
  - json/v2
  - Marshal
  - Unmarshal
  - Serialização
ordem: 35
modulo: "08 - IO e Serialização"
---
# JSON (marshal e unmarshal)

> **Nota 35 da trilha de Go.** JSON é o formato de troca de dados da web: APIs REST, arquivos de configuração, mensagens entre serviços. Em Go, ele é tratado pelo pacote `encoding/json`, que converte structs em JSON (**marshal**) e JSON em structs (**unmarshal**) usando as **tags** de struct. Nesta nota veremos os dois sentidos da conversão, as tags e suas opções (`omitempty`, `omitzero`, `-`, `string`), o mapeamento de tipos, como lidar com campos opcionais e JSON dinâmico, serialização customizada, **streaming** com `Encoder`/`Decoder`, armadilhas de precisão numérica, segurança, e a grande novidade: o **`encoding/json/v2`**, que desde o **Go 1.27** é o motor por trás do `encoding/json`.
>
> **Pré-requisitos:** [[Structs]], [[Interfaces]], [[io.Reader e io.Writer]], [[Type Assertion e Type Switch]].

---

## 🧭 De onde viemos

Em [[Structs]] vimos as **tags** (`json:"nome"`) e dissemos que bibliotecas as leem via reflection. Em [[Type Assertion e Type Switch]] decodificamos JSON em `map[string]any` e vimos que números viram `float64`. Em [[io.Reader e io.Writer]] vimos o poder do streaming. Agora vamos dominar o pacote de JSON.

> 💡 **Serialização (marshaling):** transformar uma estrutura de dados em memória numa sequência de bytes que pode ser gravada ou enviada pela rede. **Desserialização (unmarshaling)** é o caminho inverso. O formato pode ser JSON, XML, Protocol Buffers, MessagePack...

---

## 📤 1. Marshal: de Go para JSON

```go
type Endereco struct {
	Cidade string `json:"cidade"`
	UF     string `json:"uf,omitempty"`
}

type Usuario struct {
	ID         int64     `json:"id"`
	Nome       string    `json:"nome"`
	Email      string    `json:"email,omitempty"`     // omitido se vazio
	Senha      string    `json:"-"`                   // NUNCA serializado
	Idade      *int      `json:"idade"`               // ponteiro: null se nil
	Tags       []string  `json:"tags"`
	Endereco   Endereco  `json:"endereco"`            // struct aninhada → objeto aninhado
	CriadoEm   time.Time `json:"criado_em"`           // RFC 3339
	Atualizado time.Time `json:"atualizado,omitzero"` // omitido se for o zero value (Go 1.24+)
	Saldo      int64     `json:"saldo,string"`        // número codificado como string JSON
	interno    string                                 // não exportado: ignorado
}
```

Saída real:

```go
u := Usuario{ID: 1, Nome: "Ana", Senha: "x", Endereco: Endereco{Cidade: "Curitiba"},
	CriadoEm: time.Date(2026, 10, 20, 14, 0, 0, 0, time.UTC), Saldo: 1999, interno: "y"}

b, err := json.Marshal(u)
fmt.Println(string(b))
// {"id":1,"nome":"Ana","idade":null,"tags":null,"endereco":{"cidade":"Curitiba"},
//  "criado_em":"2026-10-20T14:00:00Z","saldo":"1999"}
```

Repare no que aconteceu:

| Campo | Resultado | Por quê |
| --- | --- | --- |
| `Email` | sumiu | `omitempty` e string vazia |
| `Senha` | sumiu | tag `"-"` |
| `Idade` | `null` | ponteiro nil |
| `Tags` | `null` | slice **nil** (em v1; veja a seção 9) |
| `Endereco.UF` | sumiu | `omitempty` |
| `Atualizado` | sumiu | `omitzero` e `time.Time` zero |
| `Saldo` | `"1999"` | opção `string` |
| `interno` | sumiu | não exportado: o pacote json não enxerga (regra da maiúscula) |

> ⚠️ **Só campos exportados (com letra maiúscula) são serializados.** É a causa número um de "meu JSON veio vazio": `type U struct { nome string }` gera `{}`.

Para JSON legível (saída real, chaves de map saem **ordenadas**):

```go
b, _ := json.MarshalIndent(map[string]any{"b": 2, "a": []int{1, 2}}, "", "  ")
// {
//   "a": [
//     1,
//     2
//   ],
//   "b": 2
// }
```

---

## 📥 2. Unmarshal: de JSON para Go

```go
var v Usuario
err := json.Unmarshal(dados, &v)     // ⚠️ passe um PONTEIRO: a função precisa preencher v
```

Saída real:

```go
err := json.Unmarshal([]byte(`{"ID":7,"NOME":"Bia","idade":30,"tags":["go"],"extra":true,"saldo":"500"}`), &v)
fmt.Println(err, v.ID, v.Nome, *v.Idade, v.Tags, v.Saldo)
// <nil> 7 Bia 30 [go] 500
```

Comportamentos do `encoding/json` (semântica "v1", mantida por compatibilidade):

- **Nomes casam sem diferenciar maiúsculas** (`"NOME"` preencheu `Nome`, cuja tag é `"nome"`)
- **Campos desconhecidos são ignorados** (`"extra"` não deu erro)
- Campos ausentes no JSON **mantêm** o valor que já estava na struct (útil para valores padrão)
- `null` num ponteiro, slice ou map deixa `nil`; num valor comum, não faz nada

### 2.1 Erros de unmarshal

Saídas reais:

```go
var tipo struct{ N int }

err := json.Unmarshal([]byte(`{"N":"dez"}`), &tipo)
// json: cannot unmarshal string into Go struct field .N of type int
var ute *json.UnmarshalTypeError
errors.As(err, &ute)          // true: tipo errado

err = json.Unmarshal([]byte(`{"N":1,}`), &tipo)
// invalid character '}' looking for beginning of object key string
var se *json.SyntaxError
errors.As(err, &se)           // true, se.Offset == 8 (posição do problema)
```

### 2.2 Rejeitar campos desconhecidos

Em APIs, aceitar campos desconhecidos em silêncio esconde erros de digitação do cliente (`"emial"`). Com o `Decoder` (saída real):

```go
dec := json.NewDecoder(strings.NewReader(`{"N":1,"X":2}`))
dec.DisallowUnknownFields()
fmt.Println(dec.Decode(&tipo))      // json: unknown field "X"
```

---

## 🗺️ 3. Mapeamento de tipos

| Go | JSON |
| --- | --- |
| `bool` | `true` / `false` |
| inteiros, floats | número |
| `string` | string (UTF-8; caracteres especiais escapados) |
| `[]T`, `[N]T` | array |
| `[]byte` | string em **base64** |
| `map[string]T` (ou chave inteira / `TextMarshaler`) | objeto |
| struct | objeto (campos exportados) |
| ponteiro | o valor apontado, ou `null` |
| interface | o valor dinâmico |
| `time.Time` | string RFC 3339 (`"2026-10-20T14:00:00Z"`) |
| canal, função, complexo | ❌ erro |

E na volta, para dentro de `any` (JSON sem estrutura definida):

| JSON | Go (`any`) |
| --- | --- |
| objeto | `map[string]any` |
| array | `[]any` |
| número | `float64` |
| string | `string` |
| `true`/`false` | `bool` |
| `null` | `nil` |

---

## 🏷️ 4. Tags em detalhe

```go
`json:"nome"`             // nome do campo no JSON
`json:"nome,omitempty"`   // omite se for "vazio": false, 0, "", nil, slice/map de tamanho 0
`json:"nome,omitzero"`    // omite se for o ZERO VALUE (ou se IsZero() devolver true) — Go 1.24+
`json:",omitempty"`       // mantém o nome do campo Go, mas com omitempty
`json:"-"`                // ignora sempre
`json:"-,"`               // um campo cujo nome JSON é literalmente "-"
`json:"valor,string"`     // número/bool codificado dentro de uma string JSON
```

### 4.1 `omitempty` vs `omitzero`

| Valor | `omitempty` omite? | `omitzero` omite? |
| --- | --- | --- |
| `0`, `""`, `false` | ✅ | ✅ |
| slice/map nil | ✅ | ✅ |
| slice/map **vazio, não nil** | ✅ | ❌ |
| struct com todos os campos zero (ex.: `time.Time{}`) | ❌ (structs nunca são "vazias") | ✅ |
| tipo com método `IsZero()` devolvendo true | ❌ | ✅ |

O caso clássico que motivou o `omitzero`: um `time.Time` com `omitempty` **nunca** é omitido (sai como `"0001-01-01T00:00:00Z"`). Com `omitzero`, sai corretamente.

### 4.2 Campos embutidos

```go
type Base struct {
	ID       int64     `json:"id"`
	CriadoEm time.Time `json:"criado_em"`
}

type Produto struct {
	Base                       // embutida SEM tag: os campos sobem para o objeto do Produto
	Nome string `json:"nome"`
}
// {"id":1,"criado_em":"...","nome":"Caneta"}   ← "achatado", não {"Base":{...}}
```

---

## ❓ 5. Opcional, ausente e nulo

Distinguir "o cliente não mandou o campo", "mandou `null`" e "mandou `0`" é importante em APIs (ex.: um `PATCH` que só altera os campos enviados):

```go
type Atualizacao struct {
	Nome  *string `json:"nome"`    // nil = ausente (ou null); não-nil = enviado
	Idade *int    `json:"idade"`
}

var a Atualizacao
json.Unmarshal([]byte(`{"idade": 0}`), &a)
// a.Nome == nil   → não enviado: não alterar
// a.Idade != nil  → enviado, valor 0: alterar para 0
```

Montar valores com ponteiros ficou mais fácil com o `new(expr)` do **Go 1.26** ([[Ponteiros]]):

```go
json.Marshal(Atualizacao{Idade: new(31)})     // antes: v := 31; Idade: &v
```

> 💡 Para distinguir também "ausente" de "null", decodifique o campo como `json.RawMessage` e verifique se ele é `nil` (ausente) ou `[]byte("null")`.

---

## 🧬 6. JSON dinâmico

### 6.1 `map[string]any`

Quando a estrutura é desconhecida. Exige type assertions, e todo número vira `float64` (veja os detalhes em [[Type Assertion e Type Switch]]).

### 6.2 `json.RawMessage`: adiar a decodificação

Quando parte do JSON depende de outro campo (mensagens com "tipo" + "dados"). Saída real:

```go
var envelope struct {
	Tipo  string          `json:"tipo"`
	Dados json.RawMessage `json:"dados"`    // guarda os bytes crus, sem decodificar
}
json.Unmarshal([]byte(`{"tipo":"ponto","dados":{"x":1,"y":2}}`), &envelope)
fmt.Println(envelope.Tipo, string(envelope.Dados))   // ponto {"x":1,"y":2}

switch envelope.Tipo {
case "ponto":
	var p Ponto
	json.Unmarshal(envelope.Dados, &p)     // agora decodifica no tipo certo
case "linha":
	// ...
}
```

### 6.3 Números grandes: cuidado com `float64`

Um `float64` só representa inteiros exatos até 2⁵³ ([[Tipos de Dados]]). Saídas reais com o número 2⁵³+1:

```go
var nums struct{ Grande int64 }
json.Unmarshal([]byte(`{"Grande": 9007199254740993}`), &nums)
fmt.Println(nums.Grande)                       // 9007199254740993   ✅ tipo concreto: exato

var anyv map[string]any
json.Unmarshal([]byte(`{"Grande": 9007199254740993}`), &anyv)
fmt.Println(int64(anyv["Grande"].(float64)))   // 9007199254740992   ❌ perdeu precisão!

dec := json.NewDecoder(strings.NewReader(`{"Grande": 9007199254740993}`))
dec.UseNumber()                                // números viram json.Number (uma string)
var anyn map[string]any
dec.Decode(&anyn)
fmt.Println(anyn["Grande"])                    // 9007199254740993   ✅
```

> ⚠️ IDs grandes (de Twitter/X, Snowflake, bancos distribuídos) costumam ser enviados como **string** justamente porque JavaScript também usa `float64` para todos os números. Use a opção `,string` na tag para esses campos.

---

## 🎨 7. Serialização customizada

Um tipo pode controlar sua própria representação implementando interfaces (saídas reais):

```go
type Status int

const (
	Ativo Status = iota + 1
	Inativo
)

func (s Status) MarshalJSON() ([]byte, error) {          // json.Marshaler
	switch s {
	case Ativo:
		return []byte(`"ativo"`), nil
	case Inativo:
		return []byte(`"inativo"`), nil
	}
	return nil, fmt.Errorf("status inválido: %d", s)
}

func (s *Status) UnmarshalJSON(b []byte) error {         // json.Unmarshaler (receptor ponteiro!)
	var txt string
	if err := json.Unmarshal(b, &txt); err != nil {
		return err
	}
	switch txt {
	case "ativo":
		*s = Ativo
	case "inativo":
		*s = Inativo
	default:
		return fmt.Errorf("status desconhecido: %q", txt)
	}
	return nil
}

type P struct{ S Status `json:"status"` }
json.Marshal(P{Ativo})                             // {"status":"ativo"}
json.Unmarshal([]byte(`{"status":"zumbi"}`), &p)  // erro: status desconhecido: "zumbi"
```

| Interface | Para | Quando usar |
| --- | --- | --- |
| `json.Marshaler` / `Unmarshaler` | JSON completo do valor | Formatos especiais (enum como string, data em outro formato) |
| `encoding.TextMarshaler` / `TextUnmarshaler` | Representação como **texto** | Valores que viram uma string (UUID, IP, enum); também servem como **chave de map** e para outros formatos (YAML, env) |

> ⚠️ **Recursão infinita:** dentro de `MarshalJSON`, chamar `json.Marshal(s)` no próprio tipo chama `MarshalJSON` de novo. Use um tipo auxiliar sem o método: `type alias Usuario; json.Marshal(alias(u))`.

---

## 🌊 8. Streaming: `Encoder` e `Decoder`

`json.Marshal`/`Unmarshal` trabalham com `[]byte` (tudo na memória). Para ler de arquivos, conexões ou corpos HTTP, use as versões de **fluxo**, que trabalham com `io.Reader`/`io.Writer` ([[io.Reader e io.Writer]]):

```go
// Ler o corpo de uma requisição HTTP
func criar(w http.ResponseWriter, r *http.Request) {
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)   // ✅ limita a 1 MB (segurança)
	var u Usuario
	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()
	if err := dec.Decode(&u); err != nil {
		http.Error(w, "JSON inválido: "+err.Error(), http.StatusBadRequest)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(u)                     // escreve direto na resposta
}
```

Processar muitos objetos em sequência (formato **NDJSON**: um JSON por linha, comum em logs). Saída real:

```go
dec := json.NewDecoder(strings.NewReader("{\"N\":1}\n{\"N\":2}\n{\"N\":3}\n"))
soma := 0
for dec.More() {
	var x struct{ N int }
	if err := dec.Decode(&x); err != nil {
		return err
	}
	soma += x.N
}
fmt.Println("soma:", soma)     // soma: 6   ← memória constante, mesmo com milhões de linhas
```

> 💡 O `Encoder` escapa `<`, `>` e `&` por padrão (para JSON embutido em HTML ser seguro). Saída real: `{"html":"<b>&</b>"}`. Para desligar: `enc.SetEscapeHTML(false)` → `{"html":"<b>&</b>"}`.

---

## 🆕 9. `encoding/json/v2` (Go 1.25 experimental → Go 1.27)

Depois de mais de uma década, o `encoding/json` original acumulou problemas que não podiam ser corrigidos sem quebrar compatibilidade (nomes casando sem diferenciar maiúsculas, nomes duplicados aceitos, UTF-8 inválido aceito, desempenho de unmarshal fraco). A solução foi um pacote novo:

- **Go 1.25:** `encoding/json/v2` e `encoding/json/jsontext` chegam como experimento (`GOEXPERIMENT=jsonv2`)
- **Go 1.27:** os pacotes v2 ficam disponíveis por padrão, **e o próprio `encoding/json` passa a usar o motor do v2 por baixo**, mantendo a semântica antiga. Resultado: unmarshal **bem mais rápido** sem mudar uma linha do seu código (as mensagens de erro podem mudar de texto). `GOEXPERIMENT=nojsonv2` restaura a implementação antiga

### 9.1 A API do v2

```go
import (
	"encoding/json/v2"
	"encoding/json/jsontext"
)

b, err := json.Marshal(v, opcoes...)                 // variádicas: json.Options
err = json.MarshalWrite(w, v)                        // direto num io.Writer
err = json.Unmarshal(b, &v)
err = json.UnmarshalRead(r, &v)                      // direto de um io.Reader
err = json.UnmarshalDecode(dec, &v)                  // com um *jsontext.Decoder (streaming fino)
```

### 9.2 O que muda de comportamento (padrões do v2)

| Aspecto | `encoding/json` (v1) | `encoding/json/v2` |
| --- | --- | --- |
| Casar nomes | Sem diferenciar maiúsculas | **Exato** (opção `MatchCaseInsensitiveNames` ou tag `case:ignore`) |
| Nomes duplicados no objeto | Aceitos (o último vence) | **Rejeitados** |
| UTF-8 inválido | Trocado por `U+FFFD` | **Rejeitado** |
| Slice nil | `null` | **`[]`** (opção `FormatNilSliceAsNull` para `null`) |
| Map nil | `null` | **`{}`** |
| Campos desconhecidos | Ignorados | Ignorados (opção `RejectUnknownMembers`) |
| `time.Duration` | Número em nanossegundos | Sem formato padrão (precisa escolher) |
| `omitempty` | Pelo valor Go ("vazio") | Pelo valor **JSON**: omite se sairia `null`, `""`, `{}` ou `[]` |

Tags novas: `case:ignore`/`case:strict`, `embed` (sobe os campos de um campo para o objeto pai). Opções úteis: `json.Deterministic(true)` (saída estável), `json.StringifyNumbers(true)`, `json.WithMarshalers(...)` (serialização customizada sem precisar implementar métodos no tipo, útil para tipos de outros pacotes).

> 💡 Os padrões do v2 são mais **seguros**: rejeitar nomes duplicados e UTF-8 inválido fecha brechas em que dois sistemas interpretam o mesmo JSON de formas diferentes (um vê o primeiro `"admin"`, outro vê o segundo). Para código novo, vale considerar o v2; para código existente, o `encoding/json` já ganhou o desempenho.

---

## 🔬 10. Por dentro: reflection, cache e alocações

O `encoding/json` não conhece seus tipos em tempo de compilação. Ele descobre a estrutura **em tempo de execução**, com o pacote `reflect` ([[Reflection]]):

```
json.Marshal(u)
  │
  ├─ reflect.TypeOf(u) → descritor do tipo Usuario (a RTTI gravada no binário)
  ├─ primeira vez com esse tipo? → analisa os campos: nomes, tags, tipos, quais são exportados
  │                                 e monta um "codificador" especializado → guarda num CACHE
  ├─ percorre os campos com reflect.Value e escreve os bytes num buffer
  └─ devolve uma cópia do buffer
```

Consequências práticas:

- A **primeira** serialização de um tipo é mais lenta (análise); as seguintes usam o cache
- Reflection custa mais que código direto; bibliotecas como `easyjson` geram código específico em compilação para eliminar isso (com `go generate`). O motor do v2 reduziu bastante essa diferença
- Cada `Marshal` aloca o `[]byte` de saída; `Encoder` sobre um `bufio.Writer` evita cópias
- `Unmarshal` em `map[string]any` aloca muito (um valor de interface por campo); structs concretas alocam bem menos

> 💡 JSON é texto: números viram dígitos, bytes viram base64 (+33%), tudo precisa de escape. Para comunicação interna de alta performance entre serviços, formatos binários com esquema (Protocol Buffers/gRPC) são menores e mais rápidos. JSON vence em interoperabilidade e legibilidade. Você estudou representação de dados e a camada de apresentação em [[Camada de Apresentação - 6]].

---

## ⚠️ 11. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Campos não exportados | JSON vazio `{}` | Letra maiúscula + tag com o nome em minúsculo |
| `Unmarshal(dados, v)` sem `&` | `json: Unmarshal(non-pointer ...)` | `Unmarshal(dados, &v)` |
| `omitempty` em `time.Time` | Data `0001-01-01` no JSON | `omitzero` (Go 1.24+) |
| Números em `map[string]any` | Perda de precisão acima de 2⁵³ | Structs com tipos inteiros, `UseNumber()` ou `,string` |
| Slice nil vs vazio | `null` vs `[]` confunde clientes | Inicialize (`[]T{}`) ou use v2 |
| Aceitar campos desconhecidos em silêncio | Erros de digitação do cliente passam | `DisallowUnknownFields` / `RejectUnknownMembers` |
| Corpo de requisição sem limite | Ataque de memória | `http.MaxBytesReader` / `io.LimitReader` |
| `MarshalJSON` chamando `json.Marshal` do mesmo tipo | Stack overflow | Tipo auxiliar sem o método |
| `UnmarshalJSON` com receptor valor | Alterações perdidas | Receptor ponteiro |
| Senhas/segredos em structs serializadas | Vazamento em respostas e logs | Tag `json:"-"` |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Marshal / Unmarshal | Go → JSON / JSON → Go (passe ponteiro) |
| Visibilidade | Só campos exportados |
| Tags | Nome, `omitempty`, `omitzero` (1.24), `-`, `string` |
| Opcional | Ponteiros (`*int`); `new(expr)` (1.26) ajuda a montar |
| Dinâmico | `map[string]any` (números `float64`), `json.RawMessage` |
| Precisão | Inteiros > 2⁵³ em `any` perdem precisão; `UseNumber` |
| Customizado | `MarshalJSON`/`UnmarshalJSON`, `TextMarshaler` |
| Streaming | `NewDecoder(r)`, `NewEncoder(w)`, NDJSON com `More()` |
| Segurança | Limite de tamanho, campos desconhecidos, nada de segredos |
| v2 | `encoding/json/v2` + `jsontext`; desde o 1.27, motor do `encoding/json` (unmarshal mais rápido) |
| Por dentro | Reflection com cache por tipo |

---

## 🔗 Conexões

**Dentro de Go**
- Tags e structs → [[Structs]]
- `Marshaler`/`Unmarshaler` são interfaces → [[Interfaces]]
- JSON dinâmico e assertions → [[Type Assertion e Type Switch]]
- Streaming com `io.Reader`/`io.Writer` → [[io.Reader e io.Writer]]
- Gravar e ler arquivos JSON → [[Leitura e Escrita de Arquivos]]
- APIs JSON com `net/http` → [[HTTP (net-http)]]
- Como o pacote enxerga seus tipos → [[Reflection]]
- `new(expr)` para campos opcionais → [[Ponteiros]]
- `*json.SyntaxError` e `*json.UnmarshalTypeError` → [[Criando Erros Customizados]]

**Redes**
- [[Camada de Apresentação - 6]] — serialização e formatos de representação de dados
- [[Termos usados Internet-Web]] — APIs, REST e o papel do JSON na web

**Nginx**
- [[API Gateway]] — gateways recebem, validam e repassam payloads JSON
- [[Compressão]] — JSON é texto e comprime muito bem com gzip/brotli

---

⬅️ **Anterior:** [[Leitura e Escrita de Arquivos]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Fundamentos de Concorrência]] ➡️
