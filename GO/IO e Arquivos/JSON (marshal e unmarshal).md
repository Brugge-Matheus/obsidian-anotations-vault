---
tags:
  - go
  - go/io
---
# JSON (marshal e unmarshal)

> O pacote `encoding/json` usa reflection para serialização/deserialização. Entender como ele usa as struct tags e como o ciclo de vida da serialização funciona ajuda a evitar armadilhas comuns de performance e segurança.
> 

---

## 1. Como `encoding/json` Funciona Internamente

```go
json.Marshal(v):
1. Usa reflect.TypeOf(v) para inspecionar a struct
2. Itera sobre os campos da struct (reflect.StructField)
3. Lê as tags `json:"..."` de cada campo
4. Serializa cada campo de acordo com seu tipo
5. Retorna []byte

O compilador gera código específico para cada tipo chamado "encoder"
que é cacheado após a primeira chamada via sync.Map interno.

Custo: reflection tem overhead. Para hot paths, considere:
- encoding/json/v2 (proposta em andamento)
- github.com/bytedance/sonic (JSON via JIT — x86 apenas)
- github.com/goccy/go-json (JSON otimizado, drop-in replacement)
- github.com/json-iterator/go
```

---

## 2. Marshal — Go para JSON

```go
import "encoding/json"

type Pessoa struct {
	Nome  string `json:"nome"`
	Idade int    `json:"idade"`
	Email string `json:"email"`
}

p := Pessoa{Nome: "Alice", Idade: 30, Email: "alice@ex.com"}

// Marshal — []byte
dados, err := json.Marshal(p)
if err != nil {
	return fmt.Errorf("marshal: %w", err)
}
fmt.Println(string(dados))
// {"nome":"Alice","idade":30,"email":"alice@ex.com"}

// MarshalIndent — formatado
dados, err = json.MarshalIndent(p, "", "  ")
// {
//   "nome": "Alice",
//   "idade": 30,
//   "email": "alice@ex.com"
// }

// Encoder — para io.Writer (sem buffer intermediário)
json.NewEncoder(os.Stdout).Encode(p)   // mais eficiente para respostas HTTP
```

---

## 3. Unmarshal — JSON para Go

```go
jsonStr := `{"nome":"Bob","idade":25,"email":"bob@ex.com"}`

var p Pessoa
err := json.Unmarshal([]byte(jsonStr), &p)
if err != nil {
	return fmt.Errorf("unmarshal: %w", err)
}
fmt.Println(p.Nome)   // "Bob"

// Decoder — para io.Reader
dec := json.NewDecoder(r.Body)
dec.DisallowUnknownFields()   // rejeita campos não declarados na struct
if err := dec.Decode(&p); err != nil {
	http.Error(w, "JSON inválido: "+err.Error(), http.StatusBadRequest)
	return
}
```

---

## 4. Struct Tags — Controle Completo da Serialização

```go
type Usuario struct {
	// Nome customizado
	ID int `json:"id"`

	// Omitir quando zero value (0, "", false, nil, slice/map vazio)
	Apelido string `json:"apelido,omitempty"`

	// Nunca serializar — campos sensíveis
	SenhaHash string `json:"-"`

	// Serializar número como string (útil para IDs grandes no JavaScript)
	// JavaScript Number tem precisão de 53 bits — int64 pode não caber
	IDExterno int64 `json:",string"`

	// Múltiplas tags para diferentes pacotes
	Email     string `json:"email" db:"email" validate:"required,email"`
	CreatedAt time.Time `json:"created_at" db:"created_at"`
}

// Casos especiais:
u := Usuario{
	ID: 1, IDExterno: 9007199254740993,   // > 2^53 — JavaScript perde precisão sem ,string
}
dados, _ := json.Marshal(u)
// {"id":1,"id_externo":"9007199254740993"}  ← com string, sem perda de precisão
```

---

## 5. Tipos Especiais no JSON

### Ponteiros — Representam null

```go
type Produto struct {
	Nome     string   `json:"nome"`
	Preco    *float64 `json:"preco"`           // *float64 pode ser null
	Desconto *float64 `json:"desconto,omitempty"` // null E omite
}

preco := 29.90
p := Produto{Nome: "Livro", Preco: &preco}   // desconto = nil

dados, _ := json.Marshal(p)
// {"nome":"Livro","preco":29.9}  ← desconto omitido (nil + omitempty)

// Deserializar null → nil
jsonNull := `{"nome":"Livro","preco":null}`
var p2 Produto
json.Unmarshal([]byte(jsonNull), &p2)
fmt.Println(p2.Preco == nil)   // true
```

### Time — RFC3339

```go
type Evento struct {
	Titulo string    `json:"titulo"`
	Data   time.Time `json:"data"`   // serializa como RFC3339 por padrão
}

e := Evento{Titulo: "Go Conf", Data: time.Now()}
dados, _ := json.Marshal(e)
// {"titulo":"Go Conf","data":"2024-03-15T10:30:00-03:00"}
```

### `json.Number` — Preservar Precisão Numérica

```go
// Por padrão, números em interface{} viram float64
// Isso pode perder precisão para inteiros grandes

dec := json.NewDecoder(strings.NewReader(`{"valor": 9007199254740993}`))
dec.UseNumber()   // números ficam como json.Number (string internamente)

var dados map[string]interface{}
dec.Decode(&dados)

n := dados["valor"].(json.Number)
i64, _ := n.Int64()
fmt.Println(i64)   // 9007199254740993 — sem perda de precisão
```

---

## 6. Serialização Customizada — `MarshalJSON` / `UnmarshalJSON`

```go
// Tipo com formato de data customizado
type DataBR struct {
	time.Time
}

func (d DataBR) MarshalJSON() ([]byte, error) {
	return json.Marshal(d.Format("02/01/2006"))
}

func (d *DataBR) UnmarshalJSON(data []byte) error {
	var s string
	if err := json.Unmarshal(data, &s); err != nil {
		return err
	}
	t, err := time.Parse("02/01/2006", s)
	if err != nil {
		return fmt.Errorf("data inválida %q: %w", s, err)
	}
	d.Time = t
	return nil
}

// Implementar apenas MarshalText/UnmarshalText para tipos mais simples
// (mais eficiente que MarshalJSON pois evita uma camada de reflection)
type MoedaBRL struct {
	Centavos int64
}

func (m MoedaBRL) MarshalText() ([]byte, error) {
	reais := m.Centavos / 100
	centavos := m.Centavos % 100
	return []byte(fmt.Sprintf("%d.%02d", reais, centavos)), nil
}

func (m *MoedaBRL) UnmarshalText(data []byte) error {
	parts := strings.Split(string(data), ".")
	reais, _ := strconv.ParseInt(parts[0], 10, 64)
	var centavos int64
	if len(parts) > 1 {
		centavos, _ = strconv.ParseInt(parts[1], 10, 64)
	}
	m.Centavos = reais*100 + centavos
	return nil
}
```

---

## 7. JSON Dinâmico

### `map[string]interface{}`

```go
// Deserializar JSON estrutura desconhecida
var dados map[string]interface{}
json.Unmarshal([]byte(`{"nome":"Alice","idade":30,"tags":["go","dev"]}`), &dados)

// Tipos que json usa para interface{}:
// string → string
// number → float64 (padrão) ou json.Number (com UseNumber)
// boolean → bool
// array → []interface{}
// object → map[string]interface{}
// null → nil

nome := dados["nome"].(string)   // type assertion necessária
idade := int(dados["idade"].(float64))   // conversão de float64
tags := dados["tags"].([]interface{})
```

### `json.RawMessage` — Adiar Deserialização

```go
type Evento struct {
	Tipo    string          `json:"tipo"`
	Payload json.RawMessage `json:"payload"`   // mantém como bytes JSON brutos
}

var e Evento
json.Unmarshal([]byte(`{
	"tipo": "usuario_criado",
	"payload": {"nome": "Alice", "email": "alice@ex.com"}
}`), &e)

// Deserializar o payload depois, baseado no tipo
switch e.Tipo {
case "usuario_criado":
	var u Usuario
	json.Unmarshal(e.Payload, &u)
case "pedido_criado":
	var p Pedido
	json.Unmarshal(e.Payload, &p)
}
```

---

## 8. Streaming — Encoder/Decoder

Para arquivos grandes ou streams HTTP, use Encoder/Decoder diretamente:

```go
// Decoder — ler múltiplos JSONs de um stream (NDJSON — newline-delimited JSON)
f, _ := os.Open("usuarios.ndjson")
defer f.Close()

dec := json.NewDecoder(f)
dec.DisallowUnknownFields()

for dec.More() {   // More() retorna false quando chega no fim
	var u Usuario
	if err := dec.Decode(&u); err != nil {
		return fmt.Errorf("decodificar linha: %w", err)
	}
	processar(u)
}

// Encoder — escrever em stream
enc := json.NewEncoder(w)
enc.SetIndent("", "  ")    // opcional: formatar
enc.SetEscapeHTML(false)   // não escapar < > & em strings (útil para APIs)

for _, u := range usuarios {
	if err := enc.Encode(u); err != nil {
		return err
	}
}
```

---

## 9. Performance — Dicas

```go
// 1. Reutilize os.Encoder/Decoder quando possível
// 2. Para hot paths, considere alternativas mais rápidas:

// github.com/bytedance/sonic — 2-5x mais rápido que stdlib (x86 via JIT)
import "github.com/bytedance/sonic"
dados, err := sonic.Marshal(p)
sonic.Unmarshal(dados, &p)

// github.com/goccy/go-json — drop-in replacement, sem CGo
import json "github.com/goccy/go-json"
// mesma API da stdlib — apenas troque o import

// 3. Evite json.Marshal para tipos simples em loops
// Use strings.Builder ou fmt.Fprintf direto

// 4. SetEscapeHTML(false) quando não há HTML nos dados
enc := json.NewEncoder(w)
enc.SetEscapeHTML(false)   // muda & → &amp; é desnecessário para APIs JSON

// 5. Pré-alocar buffers para marshal repetitivo
buf := bytes.NewBuffer(make([]byte, 0, 1024))
enc := json.NewEncoder(buf)
for _, item := range items {
	buf.Reset()
	enc.Encode(item)
	enviar(buf.Bytes())
}
```

---

## 10. Padrões com HTTP

```go
// Handler que recebe JSON
func criarUsuario(w http.ResponseWriter, r *http.Request) {
	// Limitar tamanho do body — proteção contra DoS
	r.Body = http.MaxBytesReader(w, r.Body, 1<<20)   // 1MB

	dec := json.NewDecoder(r.Body)
	dec.DisallowUnknownFields()   // rejeita campos extras

	var u Usuario
	if err := dec.Decode(&u); err != nil {
		var syntaxErr *json.SyntaxError
		var unmarshalErr *json.UnmarshalTypeError

		switch {
		case errors.As(err, &syntaxErr):
			jsonError(w, 400, fmt.Sprintf("JSON inválido na posição %d", syntaxErr.Offset))
		case errors.As(err, &unmarshalErr):
			jsonError(w, 400, fmt.Sprintf("campo %q: tipo incorreto", unmarshalErr.Field))
		case errors.Is(err, io.EOF):
			jsonError(w, 400, "corpo da requisição vazio")
		default:
			jsonError(w, 400, "JSON inválido")
		}
		return
	}

	// ... processar

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	json.NewEncoder(w).Encode(u)
}

func jsonError(w http.ResponseWriter, status int, msg string) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	json.NewEncoder(w).Encode(map[string]string{"erro": msg})
}
```

---

## 11. Resumo

| Operação | Função | Destino/Fonte |
| --- | --- | --- |
| Struct → `[]byte` | `json.Marshal(v)` | memória |
| Struct → `io.Writer` | `json.NewEncoder(w).Encode(v)` | stream |
| `[]byte` → Struct | `json.Unmarshal(data, &v)` | memória |
| `io.Reader` → Struct | `json.NewDecoder(r).Decode(&v)` | stream |
| Verificar JSON válido | `json.Valid(data)` | memória |
| Formatar JSON | `json.MarshalIndent(v, "", "  ")` | memória |
| JSON bruto sem parse | `json.RawMessage` | — |
| Manter precisão numérica | `dec.UseNumber()`  • `json.Number` | — |
| Performance | `sonic`, `go-json` ou `json-iterator` | — |

---

## 12. Por Dentro: JSON, Reflection e Memória

### Como `encoding/json` usa reflection para inspecionar structs

```
json.Marshal(p)  onde p é do tipo Pessoa
       │
       ▼
reflect.TypeOf(p)  →  retorna *rtype (descritor de tipo no runtime)
       │
       ▼
itera sobre campos: reflect.Type.NumField(), .Field(i)
       │                    │
       │                    └── lê a tag `json:"nome"` de cada campo
       │
       ▼
reflect.Value.Field(i)  →  acessa o valor concreto do campo
       │
       ▼
codificadores cacheados em sync.Map[reflect.Type → encoderFunc]
       │          (após primeira chamada, reflection é evitada)
       ▼
escreve bytes JSON no buffer de saída
```

Reflection funciona lendo **type descriptors** gerados pelo compilador e embebidos no binário. Cada struct tem seu `rtype` com metadados (tamanho, campos, tags). Isso acessa estruturas de dados da runtime Go em memória — conecta com [[Gerenciamento de Memória]] (RTTI — Runtime Type Information, similar ao `typeinfo` do C++).

### Alocações de memória durante parsing

```go
// json.Unmarshal de um JSON complexo gera múltiplas alocações no heap:

input := `{"nome":"Alice","tags":["go","dev"],"endereco":{"cidade":"SP"}}`

// Fase 1: scanning — lê bytes, identifica tokens (string, {, }, [, etc.)
//         sem allocs significativas nesta fase

// Fase 2: parsing — constrói estrutura de dados
//   ┌─ alloc: string "Alice" (cópia para heap, strings são imutáveis em Go)
//   ├─ alloc: slice []string para tags
//   ├─ alloc: string "go"
//   ├─ alloc: string "dev"
//   └─ alloc: Endereco struct (se campo é ponteiro — *Endereco)

// Cada alocação → trabalho futuro para o GC coletar
```

Para hot paths (ex: endpoint que recebe 10k req/s), alocações de JSON são uma fonte comum de pressão no GC. Isso conecta com [[Gerenciamento de Memória]] (heap allocations, GC pressure, escape analysis).

### Geração de código vs reflection: a abordagem de alternativas rápidas

```
encoding/json (reflection):
  Marshal:   ~500ns/op, ~200 B/op, ~3 allocs/op

easyjson (geração de código):
  Marshal:   ~120ns/op,   ~0 B/op, ~0 allocs/op  ← sem reflection, zero alloc

sonic (JIT x86):
  Marshal:   ~80ns/op,    ~0 B/op, ~0 allocs/op  ← compila codecs em runtime

Por que zero alloc é possível?
  easyjson gera código como este (em compilação, não runtime):
  func (v *Pessoa) MarshalJSON() ([]byte, error) {
      buf.WriteString(`{"nome":"`)     // string literal — sem alloc
      buf.WriteString(v.Nome)          // referência direta — sem alloc
      buf.WriteString(`","idade":`)
      buf.AppendInt(v.Idade)           // append no buffer existente
      ...
  }
  Nenhuma reflection, nenhum type descriptor, nenhum heap alloc intermediário.
```

Conecta com [[Gerenciamento de Memória]] — allocation overhead, escape analysis (o compilador decide se o buffer fica no stack ou heap).

### `[]byte` vs `string` no contexto de JSON

```
Em Go:
  string  = imutável, header {ptr, len}
  []byte  = mutável,  header {ptr, len, cap}

json.Marshal retorna []byte (não string) porque:
  1. buffers são acumulados incrementalmente — precisam de cap (crescimento)
  2. strings são imutáveis — cada "adição" criaria nova string no heap
  3. []byte permite append eficiente com backing array pré-alocado

Conversão string(dados) após Marshal:
  → cria nova cópia no heap (a string não pode compartilhar o []byte mutável)
  → use apenas quando necessário (ex: fmt.Println, comparação)
```

Conecta com [[Gerenciamento de Memória]] — imutabilidade de strings força cópias; []byte usa buffer reutilizável.

### Streaming JSON: analogia com buffered I/O do kernel

```
json.Decoder (streaming):
  ┌─────────────────────────────────────────────────┐
  │  io.Reader (ex: HTTP body, arquivo)             │
  │       ↓  lê em chunks (4KB por padrão)          │
  │  buffer interno do Decoder                      │
  │       ↓  parse token a token                    │
  │  struct preenchida incrementalmente             │
  └─────────────────────────────────────────────────┘

Analogia com fread() / setvbuf() em C:
  fread(buf, size, nmemb, fp)  → lê chunk do kernel buffer
  json.Decoder.Decode(&v)      → consome do buffer interno do Decoder

Vantagem: um arquivo JSON de 1GB pode ser processado com uso constante de ~4KB de RAM
  → sem carregar tudo na memória de uma vez
```

Conecta com [[Arquivos]] (buffered I/O, fread analogy) e [[Dispositivos de IO]] (buffering layers entre userspace e kernel).

---

## 13. Conexão com Sistemas Operacionais

- **[[Gerenciamento de Memória]]**: `encoding/json` usa reflection para ler type descriptors em runtime (RTTI). Parsing JSON gera múltiplas alocações no heap — strings, slices, structs aninhadas. Cada alocação é trabalho futuro para o GC. Alternativas como `easyjson` usam geração de código para eliminar allocs (zero alloc marshal).

- **[[Gerenciamento de Memória]]** (allocation overhead): `[]byte` é preferido a `string` em JSON porque é mutável e permite `append` eficiente com buffer crescente. Strings em Go são imutáveis — qualquer modificação cria nova cópia no heap. `json.Marshal` retorna `[]byte` exatamente por este motivo.

- **[[Arquivos]]**: `json.Decoder` com `io.Reader` é análogo a `fread()` com `setvbuf()` em C — lê do stream em chunks, buffereando internamente para reduzir número de chamadas ao reader subjacente. Permite processar JSON de gigabytes com memória constante.

- **[[Dispositivos de IO]]**: O `json.Decoder` acrescenta uma camada de buffer em cima do `io.Reader`, reduzindo operações de leitura — princípio idêntico ao de buffers de I/O do kernel (page cache, buffer cache) que amortizam o custo de acessos ao disco.