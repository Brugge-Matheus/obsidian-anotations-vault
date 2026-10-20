---
tags:
  - go
  - go/erros
aliases:
  - error em Go
  - Tratamento de erros
  - Errors are values
  - Erros sentinela
ordem: 27
modulo: "06 - Tratamento de Erros"
---
# O Padrão error em Go

> **Nota 27 da trilha de Go.** Abrimos o módulo de tratamento de erros. Em [[Múltiplos Retornos e Erros]] vimos a mecânica: o `error` é o último resultado e verificamos com `if err != nil`. Agora vamos à **filosofia e ao desenho**: por que erros são **valores** e não exceções, o que é exatamente o tipo `error`, as três formas de **inspecionar** um erro (sentinela, tipo, comportamento), as **estratégias** de tratamento (propagar, tentar de novo, degradar, registrar, encerrar), onde **registrar** em log, como traduzir erros nas **fronteiras** de um sistema e como `error` funciona por dentro.
>
> **Pré-requisitos:** [[Múltiplos Retornos e Erros]], [[Interfaces]], [[Defer, Panic e Recover]].

---

## 🧭 De onde viemos

Já sabemos que falhas **esperadas** são `error` e que `panic` é para bugs ([[Defer, Panic e Recover]]). Sabemos que `error` é uma **interface** com um método ([[Interfaces]]) e que existe a armadilha do ponteiro nil tipado. Nesta nota, juntamos tudo num modelo mental completo; nas duas próximas, veremos como **criar** erros ricos e como **embrulhá-los** em cadeias.

---

## 💡 1. "Errors are values"

> *"Errors are values."* — Rob Pike (Go Proverbs e um post homônimo no blog oficial, 2015)

Em Go, um erro **não é um evento especial** que interrompe o fluxo do programa. É um **valor comum**, do mesmo jeito que um `int` ou uma `string`: pode ser devolvido, guardado numa variável, comparado, passado para funções, colocado num slice, embrulhado em outro. Toda a força de programação da linguagem está disponível para lidar com ele.

```
Exceções (Java, Python, PHP)              Valores (Go)
─────────────────────────────────         ─────────────────────────────────
f() lança → fluxo PULA para algum         f() devolve (v, err) → o fluxo
catch, possivelmente muitas funções       continua na linha seguinte;
acima, sem estar visível no código        o tratamento é código comum, visível
```

Por que os criadores escolheram isso:

1. **Visibilidade:** olhando a assinatura `func Abrir(p string) (*File, error)`, você sabe que pode falhar. Olhando o código, você vê o que acontece em cada falha
2. **Fluxo de controle simples:** não há saltos invisíveis; cada linha que pode falhar está marcada por um `if err != nil`
3. **Erros são tratados perto de onde acontecem,** com o contexto disponível
4. **Desempenho previsível:** devolver um erro custa o mesmo que devolver qualquer valor (não há desenrolar de stack)

> 💡 O custo é a verbosidade (`if err != nil` repetido). A comunidade discutiu várias propostas para encurtar isso (`try`, `check/handle`, `?`); até 2026, o time do Go decidiu não adotar nenhuma, por não encontrar uma que mantivesse a clareza.

### 1.1 Erros como valores programáveis: um exemplo clássico

Do post *"Errors are values"*: em vez de checar o erro a cada escrita, encapsular o erro num tipo e checar **uma vez**:

```go
type escritorComErro struct {
	w   io.Writer
	err error
}

func (e *escritorComErro) escrever(dados []byte) {
	if e.err != nil {
		return                    // depois do primeiro erro, não faz mais nada
	}
	_, e.err = e.w.Write(dados)
}

ew := &escritorComErro{w: arquivo}
ew.escrever(cabecalho)
ew.escrever(corpo)
ew.escrever(rodape)
if ew.err != nil {                // uma checagem só, no final
	return ew.err
}
```

É exatamente o padrão de `bufio.Writer` (o erro fica guardado e aparece no `Flush`) e de `bufio.Scanner` (o erro aparece em `scanner.Err()` depois do laço).

---

## 🔌 2. O tipo `error`

```go
type error interface {
	Error() string
}
```

É uma interface **pré-declarada**, com um único método. Qualquer tipo que tenha `Error() string` é um erro. O valor `nil` significa "sem erro".

### 2.1 Criando erros simples

```go
err := errors.New("conexão recusada")                        // mensagem fixa
err := fmt.Errorf("porta %d inválida", porta)               // mensagem formatada
err := fmt.Errorf("ler config: %w", errOriginal)            // embrulhando outro erro
```

> 💡 Desde o Go 1.26, `fmt.Errorf("texto fixo")` sem verbos aloca tão pouco quanto `errors.New`, então a escolha entre os dois é só de estilo.

### 2.2 Convenções de mensagem

| Regra | ✅ | ❌ |
| --- | --- | --- |
| Minúsculas | `"conexão recusada"` | `"Conexão recusada"` |
| Sem pontuação final | `"arquivo não encontrado"` | `"arquivo não encontrado."` |
| Sem prefixo redundante | `"timeout"` | `"Error: timeout"`, `"erro: timeout"` |
| Contexto do que estava sendo feito | `"ler config: ..."` | `"falhou"` |
| Prefixo do pacote em erros exportados | `"calc: divisão por zero"` | — |

Por quê? Porque erros são **concatenados**: `"servidor: carregar config: ler arquivo: open app.yaml: no such file or directory"`. Maiúsculas e pontos no meio dessa cadeia ficariam estranhos.

---

## 🔍 3. Três formas de inspecionar um erro

Às vezes o chamador precisa **reagir diferente** conforme o tipo de falha (o arquivo não existe → criar; sem permissão → abortar). Há três estratégias, da mais acoplada para a menos acoplada.

### 3.1 Erros sentinela (comparar com um valor conhecido)

> 💡 **Erro sentinela:** uma variável de erro exportada, criada uma vez, que representa uma condição específica. O chamador compara com ela.

```go
var ErrNaoEncontrado = errors.New("não encontrado")     // convenção: prefixo Err
```

Sentinelas famosas da biblioteca padrão:

| Sentinela | Significado |
| --- | --- |
| `io.EOF` | Fim da entrada (não é falha! é o sinal normal de "acabou") |
| `io.ErrUnexpectedEOF` | A entrada acabou no meio de algo |
| `os.ErrNotExist` / `fs.ErrNotExist` | Arquivo ou diretório não existe |
| `os.ErrExist`, `os.ErrPermission` | Já existe / sem permissão |
| `sql.ErrNoRows` | A consulta não retornou linhas |
| `context.Canceled`, `context.DeadlineExceeded` | Operação cancelada / prazo esgotado |
| `http.ErrServerClosed` | O servidor foi encerrado normalmente |

Como comparar (saídas reais):

```go
err := ler()     // "carregar config: open /nao/existe: no such file or directory"

err == os.ErrNotExist             // ❌ false: o erro está embrulhado em outras camadas
errors.Is(err, os.ErrNotExist)    // ✅ true: procura em toda a cadeia
```

> ✅ Sempre use **`errors.Is`**, nunca `==`, para comparar com sentinelas. Veremos o porquê em [[Wrapping e Unwrapping de Erros]].

> ⚠️ Sentinelas viram **parte da API pública** do seu pacote: quem compara com elas passa a depender delas para sempre. Exporte só as que o chamador realmente precisa distinguir.

### 3.2 Tipos de erro (extrair dados estruturados)

Quando o chamador precisa de **informação** além da mensagem, o erro é uma struct com campos (saída real):

```go
var pe *fs.PathError
if errors.As(err, &pe) {
	fmt.Println("op:", pe.Op, "path:", pe.Path, "err:", pe.Err)
}
// op: open path: /nao/existe err: no such file or directory
```

Convenção de nome: sufixo `Error` (`PathError`, `SyntaxError`, `NumError`, `ParseError`). Veremos como criar os seus em [[Criando Erros Customizados]].

### 3.3 Comportamento (perguntar o que o erro sabe fazer)

A forma mais desacoplada: o chamador não conhece o tipo, só pergunta se o erro tem um **método**:

```go
type temporario interface{ Temporary() bool }

var t temporario
if errors.As(err, &t) && t.Temporary() {
	// tentar de novo
}

var ne net.Error                   // interface da biblioteca padrão
if errors.As(err, &ne) && ne.Timeout() {
	// foi timeout de rede
}
```

### 3.4 Erros opacos (não inspecionar nada)

Na maioria das vezes, o chamador **não precisa** saber nada: só propaga ou mostra a mensagem. Esse é o caso ideal, o mais desacoplado. Dave Cheney resumiu: *"assert errors for behaviour, not type"*, e de preferência nem isso.

| Estratégia | Acoplamento | Quando usar |
| --- | --- | --- |
| Opaco (só `err != nil`) | Nenhum | A maioria dos casos |
| Comportamento (`Timeout()`, `Temporary()`) | Baixo | Decisões genéricas (tentar de novo?) |
| Sentinela (`errors.Is`) | Médio | Condições bem definidas e estáveis (`io.EOF`, "não encontrado") |
| Tipo (`errors.As`) | Alto | Quando o chamador precisa dos **dados** do erro |

> ❌ **Nunca** inspecione a **mensagem** do erro (`strings.Contains(err.Error(), "not found")`). Mensagens mudam, são traduzidas, ganham contexto. É a forma mais frágil de todas.

---

## 🧭 4. Estratégias de tratamento

Ao receber um erro, a função tem basicamente cinco opções:

```
                       recebeu err != nil
                              │
     ┌──────────────┬─────────┼─────────────┬───────────────┐
     ▼              ▼         ▼             ▼               ▼
  PROPAGAR      TENTAR DE   DEGRADAR     REGISTRAR E     ENCERRAR
  (com          NOVO        (valor       CONTINUAR       (só no main /
  contexto)     (retry)     padrão)      (log)           inicialização)
```

### 4.1 Propagar com contexto (o caso mais comum)

```go
if err != nil {
	return fmt.Errorf("salvar pedido %d: %w", pedido.ID, err)
}
```

> ✅ Cada camada acrescenta **o que ela estava fazendo** e **com qual dado**, não a mensagem genérica "erro ao...". O resultado lê como uma história: `"processar lote 7: salvar pedido 42: inserir no banco: connection refused"`.

### 4.2 Tentar de novo (retry) com espera crescente

```go
for tentativa := 1; tentativa <= 3; tentativa++ {
	err = enviar(msg)
	if err == nil {
		return nil
	}
	var t interface{ Temporary() bool }
	if !errors.As(err, &t) || !t.Temporary() {
		return err                                   // erro permanente: não adianta insistir
	}
	time.Sleep(time.Duration(tentativa*tentativa) * 100 * time.Millisecond)   // espera crescente
}
return fmt.Errorf("enviar após 3 tentativas: %w", err)
```

### 4.3 Degradar

```go
cfg, err := carregarConfig()
if err != nil {
	slog.Warn("usando configuração padrão", "erro", err)
	cfg = ConfigPadrao
}
```

### 4.4 Registrar e continuar

Típico em laços de processamento, onde um item com problema não deve parar o lote:

```go
for _, item := range itens {
	if err := processar(item); err != nil {
		slog.Error("item ignorado", "id", item.ID, "erro", err)
		continue
	}
}
```

### 4.5 Encerrar

Só no topo do programa (o `main`) ou na inicialização, quando não há como continuar:

```go
func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, "erro:", err)
		os.Exit(1)
	}
}
```

---

## 📝 5. Onde registrar o erro: uma vez só

> ⚠️ **Antipadrão "registrar e devolver":**
> ```go
> if err != nil {
> 	log.Printf("erro ao salvar: %v", err)     // registra aqui...
> 	return err                               // ...e devolve, para ser registrado de novo lá em cima
> }
> ```
> Cada camada registra o mesmo erro, e o log fica com 5 linhas para uma falha só.

> ✅ **Regra:** **ou** você trata o erro (e então pode registrar), **ou** devolve para cima (com contexto, sem registrar). O registro acontece **uma vez**, no ponto em que o erro é efetivamente tratado (tipicamente na fronteira: o handler HTTP, o laço de um worker, o `main`).

---

## 🚧 6. Erros nas fronteiras do sistema

Nas fronteiras (API HTTP, CLI, mensagens para usuários), os erros internos precisam ser **traduzidos**: o cliente HTTP não deve receber `"pq: duplicate key value violates unique constraint"`.

```go
func (h *Handler) CriarUsuario(w http.ResponseWriter, r *http.Request) {
	err := h.servico.Criar(r.Context(), dados)
	switch {
	case err == nil:
		w.WriteHeader(http.StatusCreated)
	case errors.Is(err, ErrEmailEmUso):
		http.Error(w, "e-mail já cadastrado", http.StatusConflict)               // 409
	case errors.As(err, new(ErroValidacao)):
		http.Error(w, err.Error(), http.StatusBadRequest)                       // 400
	case errors.Is(err, context.DeadlineExceeded):
		http.Error(w, "tempo esgotado", http.StatusGatewayTimeout)              // 504
	default:
		slog.Error("criar usuário", "erro", err)                                // registra o detalhe
		http.Error(w, "erro interno", http.StatusInternalServerError)           // 500 genérico para fora
	}
}
```

Princípios:

- **Erros esperados** viram respostas específicas (4xx)
- **Erros inesperados** são registrados com todo o detalhe e viram uma resposta genérica (500), **sem vazar** detalhes internos (segurança)

Você estudou os códigos de status em [[HTTP e HTTPS]]. Veremos o servidor em [[HTTP (net-http)]].

---

## 🔬 7. Por dentro

### 7.1 `errors.New` devolve um ponteiro

A implementação inteira de `errors.New`:

```go
func New(text string) error {
	return &errorString{text}
}

type errorString struct {
	s string
}

func (e *errorString) Error() string {
	return e.s
}
```

O detalhe importante é o `&`: cada chamada cria um objeto **novo**, e a comparação de erros compara **ponteiros** (saída real):

```go
e1 := errors.New("a")
e2 := errors.New("a")
fmt.Println(e1 == e2, e1.Error() == e2.Error())    // false true
```

É por isso que sentinelas funcionam: `ErrNaoEncontrado` é criado **uma vez**, e todo mundo compara com **aquele** ponteiro. Duas sentinelas com o mesmo texto continuam diferentes.

### 7.2 Um `error` na memória

Como toda interface ([[Interfaces]]), uma variável `error` ocupa 2 palavras: a itab de `(tipo concreto, error)` e o ponteiro para os dados. `err != nil` compara essas duas palavras com zero, um custo de poucos ciclos. Devolver um erro é colocar essas duas palavras em dois registradores (`CX` e `DI`, por exemplo, quando são o 2º resultado), como vimos em [[Múltiplos Retornos e Erros]].

### 7.3 A linhagem: do `errno` ao `error`

```
UNIX / C (1970)                Go (2009)
───────────────────────        ──────────────────────────────────────────
open() devolve -1              os.Open devolve (nil, err)
errno = ENOENT (global)        err = &fs.PathError{Op: "open",
perror() → "No such file..."        Path: "a.txt", Err: syscall.ENOENT}
                               err.Error() → "open a.txt: no such file or directory"
```

No Linux, o fluxo completo é:

```
syscall openat devolve -2 no registrador RAX   (−ENOENT)
       │
       ▼
pacote syscall: Errno(2)  →  syscall.ENOENT    (um tipo error numérico)
       │
       ▼
pacote os: &fs.PathError{Op: "open", Path: p, Err: syscall.ENOENT}
       │
       ▼
seu código: errors.Is(err, fs.ErrNotExist) → true
            (Errno implementa Is: ENOENT "é" ErrNotExist)
```

Você estudou os códigos de erro das syscalls em [[System Calls]] e [[Syscalls para Gerenciamento de Arquivos]]. O Go mantém o código numérico original (acessível via `errors.As(err, &errno)`) e acrescenta contexto legível.

---

## ⚠️ 8. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| `err == ErrX` com erros embrulhados | Comparação falha | `errors.Is(err, ErrX)` |
| Inspecionar `err.Error()` | Quebra quando a mensagem muda | Sentinelas, tipos ou comportamento |
| Registrar e devolver | Mesmo erro várias vezes no log | Trate **ou** devolva |
| Devolver erro sem contexto | "connection refused" sem saber onde | `fmt.Errorf("op %s: %w", x, err)` |
| Vazar erro interno para o cliente | Exposição de detalhes (segurança) | Traduza nas fronteiras |
| Tratar `io.EOF` como falha | Erro reportado no fim normal da leitura | `errors.Is(err, io.EOF)` → fim normal |
| Exportar sentinelas demais | API rígida | Exporte só o que o chamador precisa distinguir |
| Ponteiro nil tipado devolvido como `error` | `err != nil` sem erro | Devolva `nil` literal (ver [[Interfaces]]) |
| `panic` para falhas esperadas | Programa cai por motivos normais | `error` |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Filosofia | Erros são **valores**: devolvidos, guardados, comparados, programados |
| `error` | Interface `Error() string`; `nil` = sucesso |
| Criação | `errors.New`, `fmt.Errorf` (com `%w` para embrulhar) |
| Mensagens | Minúsculas, sem ponto, com contexto |
| Sentinela | `var ErrX = errors.New(...)`; compare com `errors.Is` |
| Tipo | Struct com dados; extraia com `errors.As` |
| Comportamento | Interfaces como `Timeout()`, `Temporary()` |
| Opaco | O ideal: só propagar |
| Estratégias | Propagar, retry, degradar, registrar e continuar, encerrar |
| Log | Uma vez, onde o erro é tratado |
| Fronteiras | Traduza para respostas adequadas; não vaze detalhes |
| Por dentro | `errors.New` = ponteiro novo; sentinela = identidade; herança do `errno` |

---

## 🔗 Conexões

**Dentro de Go**
- A mecânica de `(valor, error)` → [[Múltiplos Retornos e Erros]]
- Tipos de erro próprios → [[Criando Erros Customizados]]
- `%w`, `errors.Is`, `errors.As`, `errors.Join` → [[Wrapping e Unwrapping de Erros]]
- `error` como interface e a armadilha do nil → [[Interfaces]]
- Panic vs erro → [[Defer, Panic e Recover]]
- `io.EOF` e erros de I/O → [[io.Reader e io.Writer]]
- `context.Canceled` e `DeadlineExceeded` → [[context.Context]]
- Respostas HTTP de erro → [[HTTP (net-http)]]

**Sistemas Operacionais**
- [[System Calls]] e [[Syscalls para Gerenciamento de Arquivos]] — retorno `-1` + `errno`, a origem do modelo de Go
- [[Syscalls Diversas]] — códigos como `EPERM`, `EINTR`, `EAGAIN` e o seu significado

**Nginx e Redes**
- [[HTTP e HTTPS]] — códigos de status 4xx (erro do cliente) e 5xx (erro do servidor), o destino final de muitos erros de Go
- [[Servidores de Backup e Health Check]] — tentar de novo e degradar são as mesmas estratégias usadas por balanceadores diante de falhas de backends

---

⬅️ **Anterior:** [[Iteradores]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Criando Erros Customizados]] ➡️
