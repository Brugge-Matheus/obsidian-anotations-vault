---
tags:
  - go
  - go/fundamentos
aliases:
  - if em Go
  - for em Go
  - switch em Go
  - range
  - Loops em Go
ordem: 10
modulo: "02 - Fundamentos"
---
# Controle de Fluxo

> **Nota 10 da trilha de Go.** Até aqui nossos programas rodavam de cima para baixo, linha por linha. Controle de fluxo é o que permite **decidir** (`if`, `switch`) e **repetir** (`for`). Go tem só essas três estruturas (mais `goto`, raramente usado), e cada uma é mais versátil do que em outras linguagens: o `for` substitui `while`, `do-while` e `foreach`; o `switch` não precisa de `break`; e `if`/`switch` aceitam uma declaração curta antes da condição.
>
> **Pré-requisitos:** [[Sintaxe, Blocos e Escopo]], [[Operadores de Comparação e Lógicos]].

---

## 🧭 De onde viemos

Em [[Operadores de Comparação e Lógicos]] aprendemos a produzir valores `bool`. Em [[Sintaxe, Blocos e Escopo]] vimos que `if`, `for` e `switch` criam **blocos implícitos**, nos quais variáveis declaradas na própria estrutura vivem só ali. Agora juntamos as duas coisas.

> 💡 **Fluxo de controle:** a ordem em que as instruções de um programa são executadas. Por padrão é sequencial; estruturas de controle desviam esse fluxo (pulam trechos, voltam para trás, escolhem entre caminhos).

---

## 🔀 1. `if` / `else`

### 1.1 Forma básica

```go
if idade >= 18 {
	fmt.Println("maior de idade")
} else if idade >= 16 {
	fmt.Println("pode votar")
} else {
	fmt.Println("menor de idade")
}
```

Regras de Go:

- **Sem parênteses** em volta da condição (o `gofmt` remove se você colocar)
- **Chaves obrigatórias**, mesmo para uma linha só
- A condição precisa ser `bool` (nada de `if n {` com número)
- `else` precisa estar na **mesma linha** do `}` (por causa do ponto e vírgula automático, ver [[Sintaxe, Blocos e Escopo]])

> 💡 As chaves obrigatórias evitam bugs reais. O caso mais famoso é o **"goto fail"** da Apple (2014): uma linha `goto fail;` duplicada, fora das chaves de um `if`, desativava a verificação de certificados SSL no iOS e no macOS. Em Go esse erro de indentação é impossível.

### 1.2 `if` com declaração curta

`if` aceita uma **declaração curta** antes da condição, separada por `;`. As variáveis criadas ali existem **só dentro do `if` e dos seus `else`**:

```go
if n, err := strconv.Atoi(texto); err != nil {
	fmt.Println("erro:", err)        // erro: strconv.Atoi: parsing "x": invalid syntax
} else {
	fmt.Println("número:", n)        // n e err visíveis também no else
}
// n e err não existem mais aqui
```

O uso mais comum é com funções que retornam erro ou com o "vírgula ok":

```go
if err := salvar(dados); err != nil {
	return err
}

if v, ok := cache[chave]; ok {        // "vírgula ok" de maps (ver Maps)
	return v
}
```

> ✅ Use a forma curta quando a variável só importa para a condição. Ela limita o escopo e deixa claro que o valor não é usado depois.

### 1.3 Retorno antecipado (guard clauses) — o estilo Go

Em Go, o "caminho feliz" do código fica **sem indentação**, encostado à esquerda. Os casos de erro são tratados primeiro e saem cedo:

```go
// ❌ "código em seta": a lógica principal fica enterrada
func processar(u *Usuario) error {
	if u != nil {
		if u.Ativo {
			if u.Email != "" {
				// ... lógica principal aqui, 3 níveis à direita
				return nil
			}
			return errors.New("email vazio")
		}
		return errors.New("inativo")
	}
	return errors.New("usuário nulo")
}

// ✅ guard clauses: cada problema é tratado e descartado
func processar(u *Usuario) error {
	if u == nil {
		return errors.New("usuário nulo")
	}
	if !u.Ativo {
		return errors.New("inativo")
	}
	if u.Email == "" {
		return errors.New("email vazio")
	}

	// ... lógica principal, sem indentação extra
	return nil
}
```

> ✅ Quando um `if` termina com `return`, `break`, `continue` ou `panic`, **não use `else`**: o código depois do `if` já é o "senão". O linter `revive` (regra *indent-error-flow*) aponta isso.

---

## 🔁 2. `for` — o único laço

Go tem **uma única palavra-chave** para laços, com quatro formas.

### 2.1 Forma clássica: inicialização; condição; pós

```go
for i := 0; i < 5; i++ {
	fmt.Println(i)          // 0 1 2 3 4
}

for i, j := 0, 10; i < j; i, j = i+1, j-1 {   // várias variáveis
	fmt.Println(i, j)
}
```

As três partes são opcionais. Como `i++` é um comando (ver [[Operações Aritméticas]]), ele cabe na parte "pós".

### 2.2 Só condição: o "while" de Go

```go
n := 1
for n < 100 {
	n *= 2
}
fmt.Println(n)              // 128
```

### 2.3 Sem nada: laço infinito

```go
for {
	linha, err := leitor.ReadString('\n')
	if err != nil {
		break               // a saída fica dentro do corpo
	}
	processar(linha)
}
```

É muito usado em servidores (laço de aceitar conexões), workers e leitura de streams. Para um "do-while" (executar ao menos uma vez), use `for { ...; if !cond { break } }`.

### 2.4 `for range`: percorrer coleções

`range` percorre os elementos de uma coleção. O que ele entrega depende do tipo:

| Expressão | 1º valor | 2º valor |
| --- | --- | --- |
| slice / array / ponteiro para array | índice | **cópia** do elemento |
| string | índice do **byte** onde a rune começa | a **rune** (caractere Unicode) |
| map | chave | valor (ordem **aleatória**) |
| channel | elemento recebido | — |
| inteiro `n` (Go 1.22+) | 0, 1, ..., n−1 | — |
| função iteradora (Go 1.23+) | o que a função produzir | o que a função produzir |

```go
frutas := []string{"maçã", "uva", "kiwi"}
for i, f := range frutas { fmt.Println(i, f) }   // índice e valor
for i := range frutas { }                        // só o índice
for _, f := range frutas { }                     // só o valor
for range frutas { }                             // nenhum dos dois

for i := range 3 {                               // Go 1.22+: 0 1 2
	fmt.Print(i, " ")
}

for chave, valor := range idades { }             // map: ordem muda a cada execução

for msg := range canal { }                       // até o channel ser fechado
```

Com strings (saída real), o índice pula porque `é` ocupa 2 bytes:

```go
for i, r := range "café" {
	fmt.Printf("%d:%c ", i, r)
}
// 0:c 1:a 2:f 3:é
```

Veremos os detalhes de cada caso nas notas dos tipos: [[Strings em Profundidade]], [[Arrays e Slices]], [[Maps]], [[Channels]] e [[Iteradores]].

### 2.5 Detalhes importantes do `range`

**O valor é uma cópia.** Alterar a variável não altera a coleção (saída real):

```go
nums := []int{1, 2, 3}
for _, v := range nums {
	v *= 2                   // muda só a cópia
}
fmt.Println(nums)            // [1 2 3]

for i := range nums {
	nums[i] *= 10            // ✅ use o índice para alterar
}
fmt.Println(nums)            // [10 20 30]
```

**A expressão do `range` é avaliada uma única vez, no início.** Adicionar elementos durante o laço não aumenta o número de voltas (saída real):

```go
s := []int{1, 2, 3}
for i := range s {
	if i == 0 {
		s = append(s, 99)    // o laço continua com 3 voltas
	}
}
fmt.Println(s)               // [1 2 3 99]
```

### 2.6 Uma variável nova a cada iteração (Go 1.22+)

Até o Go 1.21, a variável do `for` era **uma só**, reaproveitada em todas as voltas. Isso causava um dos bugs mais famosos da linguagem quando closures ou goroutines capturavam a variável: todas viam o último valor.

Desde o **Go 1.22**, cada iteração cria uma **variável nova** (saída real):

```go
var fs []func()
for i := 0; i < 3; i++ {
	fs = append(fs, func() { fmt.Print(i, " ") })
}
for _, f := range fs {
	f()
}
// Go 1.22+:  0 1 2
// Go ≤1.21:  3 3 3     ← todas as closures viam a MESMA variável i
```

> 💡 A mudança vale conforme a linha `go` do `go.mod`: módulos que declaram `go 1.22` ou maior têm a semântica nova. Esse é um exemplo de como a promessa de compatibilidade é mantida (código antigo continua com o comportamento antigo até atualizar a versão). Closures são estudadas em [[Closures e Funções Anônimas]].

---

## ⏭️ 3. `break`, `continue` e labels

```go
for i := 0; i < 10; i++ {
	if i%2 == 0 {
		continue            // pula para a próxima volta
	}
	if i > 7 {
		break               // sai do laço
	}
	fmt.Print(i, " ")       // 1 3 5 7
}
```

`break` e `continue` afetam o **laço mais interno**. Para afetar um laço externo, use um **label** (saída real):

```go
externo:
	for i := 0; i < 3; i++ {
		for j := 0; j < 3; j++ {
			if j == 1 {
				continue externo    // próxima volta do laço EXTERNO
			}
			fmt.Printf("i=%d j=%d | ", i, j)
		}
	}
// i=0 j=0 | i=1 j=0 | i=2 j=0 |
```

```go
busca:
	for _, linha := range matriz {
		for _, v := range linha {
			if v == alvo {
				break busca         // sai dos DOIS laços
			}
		}
	}
```

> ⚠️ Dentro de um `switch` ou `select` que está dentro de um `for`, um `break` sozinho sai **do switch**, não do laço. Para sair do laço, use um label:
> ```go
> laco:
> 	for {
> 		switch evento {
> 		case "sair":
> 			break laco      // sem o label, só sairia do switch
> 		}
> 	}
> ```

---

## 🎛️ 4. `switch`

### 4.1 Forma básica

```go
switch dia {
case 1:
	fmt.Println("domingo")
case 2, 3, 4, 5, 6:              // vários valores no mesmo case
	fmt.Println("dia útil")
case 7:
	fmt.Println("sábado")
default:
	fmt.Println("inválido")
}
```

Diferenças importantes em relação a C, Java e PHP:

| Característica | C / Java / PHP | Go |
| --- | --- | --- |
| `break` no fim de cada case | Obrigatório (esquecer causa *fall-through*) | **Implícito**: cada case para sozinho |
| Valores dos cases | Constantes inteiras (C) | **Qualquer expressão** comparável, inclusive strings e chamadas de função |
| Vários valores | Vários `case` empilhados | `case 2, 3, 4:` |
| Cair no próximo case | Comportamento padrão | Só com `fallthrough` explícito |

Os cases são avaliados **de cima para baixo**, e o primeiro que casar é executado.

### 4.2 `switch` sem expressão: um `if/else if` mais limpo

```go
switch {
case nota >= 90:
	conceito = "A"
case nota >= 80:
	conceito = "B"
case nota >= 70:
	conceito = "C"
default:
	conceito = "Reprovado"
}
```

Cada `case` é uma condição booleana. É o jeito idiomático de escrever cadeias longas de `if/else if`.

### 4.3 `switch` com declaração curta

```go
switch sistema := runtime.GOOS; sistema {
case "linux":
	fmt.Println("Linux")
case "darwin":
	fmt.Println("macOS")
default:
	fmt.Println("outro:", sistema)
}
```

### 4.4 `fallthrough`

Força a execução do **próximo case**, **sem testar** a condição dele (saída real):

```go
switch n := 2; n {
case 2:
	fmt.Println("dois")
	fallthrough
case 3:
	fmt.Println("três")       // executa, mesmo n não sendo 3
default:
	fmt.Println("outro")      // não executa: o fallthrough só desce um nível
}
// dois
// três
```

Regras: precisa ser o **último comando** do case, não pode aparecer no **último** case (`cannot fallthrough final case in switch`) e não funciona em *type switch*. Na prática é raríssimo; prefira listar vários valores num case.

### 4.5 *Type switch* (prévia)

Um `switch` especial que decide pelo **tipo** guardado numa interface:

```go
switch v := x.(type) {
case int:
	fmt.Println("inteiro", v+1)
case string:
	fmt.Println("texto", strings.ToUpper(v))
case nil:
	fmt.Println("nada")
default:
	fmt.Printf("outro tipo: %T\n", v)
}
```

Ele depende de interfaces e será explicado em [[Type Assertion e Type Switch]].

---

## 🦘 5. `goto`

Go tem `goto`, mas com restrições que o tornam bem menos perigoso que em C:

```go
	goto fim
	x := 1          // ❌ goto fim jumps over declaration of x
fim:
```

- Só salta para um label **na mesma função**
- Não pode pular **por cima de declarações** de variáveis
- Não pode saltar **para dentro** de um bloco

Na prática quase não se usa: `break`/`continue` com label e `defer` cobrem os casos de C (como "saltar para o cleanup"). Aparece em código gerado e em partes muito otimizadas da biblioteca padrão.

---

## 🧭 6. Escolhendo a estrutura certa

| Situação | Estrutura |
| --- | --- |
| Uma ou duas condições | `if` / `else` |
| Tratar erro e sair | `if err != nil { return ... }` |
| Muitas condições sobre valores diferentes | `switch` sem expressão |
| Comparar um valor com várias possibilidades | `switch valor { case ... }` |
| Decidir pelo tipo dinâmico | *type switch* |
| Repetir N vezes | `for i := range n` |
| Percorrer uma coleção | `for ... range` |
| Repetir enquanto uma condição vale | `for condição { }` |
| Repetir até um evento interno | `for { ... break }` |
| Sair de laços aninhados | `break label` |

---

## 🔬 7. Por dentro: como o compilador traduz o controle de fluxo

### 7.1 Tudo vira blocos básicos e saltos

O compilador transforma cada função num **grafo de fluxo de controle** (CFG):

> 💡 **Bloco básico:** sequência de instruções sem desvios no meio: entra-se pela primeira e sai-se pela última. O CFG é o grafo cujos nós são blocos básicos e cujas arestas são os desvios possíveis. As otimizações SSA (que vimos em [[Como Go Funciona]]) trabalham sobre esse grafo.

```
for i := 0; i < n; i++ { soma += i }

        ┌──────────────┐
        │ i = 0        │
        └──────┬───────┘
               ▼
        ┌──────────────┐   i >= n
   ┌──▶ │ CMP i, n     │ ──────────────┐
   │    └──────┬───────┘               │
   │           │ i < n                 ▼
   │    ┌──────▼───────┐        ┌────────────┐
   │    │ soma += i    │        │ (depois do │
   │    │ i++          │        │   laço)    │
   │    └──────┬───────┘        └────────────┘
   └───────────┘  ← aresta de volta (back-edge)
```

Em assembly x86-64, isso é um `CMP` seguido de um salto condicional (`JGE`, `JLT`) e um `JMP` de volta.

### 7.2 `switch`: comparações, busca binária ou tabela de saltos

Dependendo dos valores dos cases, o compilador escolhe a estratégia:

| Cases | Estratégia | Custo |
| --- | --- | --- |
| Poucos | Sequência de comparações | O(n) |
| Muitos inteiros constantes espalhados | **Busca binária** sobre os valores ordenados | O(log n) |
| Muitos inteiros constantes **densos** (ex.: 1 a 20) | **Tabela de saltos** (*jump table*): `JMP [tabela + valor*8]` | O(1) |
| Muitas strings constantes | Primeiro separa pelo **tamanho** da string (tabela de saltos), depois busca binária no conteúdo | ≈ O(log n) |

> 💡 **Tabela de saltos (jump table):** um array de endereços de código indexado pelo valor do switch. Em vez de testar case por case, a CPU calcula o endereço e salta direto. Desde o Go 1.19 o compilador gera jump tables para switches grandes de inteiros e strings (em amd64 e arm64).

### 7.3 Preempção: laços infinitos não travam mais o programa

Uma goroutine num laço apertado sem chamadas de função, como `for { x++ }`, poderia, até o Go 1.13, monopolizar uma thread e travar o escalonador (e até o GC). Desde o **Go 1.14**, o runtime tem **preempção assíncrona**: o `sysmon` percebe uma goroutine rodando há mais de ~10 ms e envia o sinal **`SIGURG`** à thread; o tratador de sinal interrompe a goroutine em qualquer ponto seguro e devolve a CPU ao escalonador.

É o mesmo princípio da interrupção de relógio que o kernel usa para preemptar processos, que você estudou em [[Introdução ao Escalonamento]] e [[Escalonamento em Sistemas Interativos]], só que implementado em espaço de usuário com sinais. Veremos os detalhes em [[Scheduler do Go]].

---

## ⚠️ 8. Armadilhas

| Armadilha | Exemplo | Correção |
| --- | --- | --- |
| Alterar a cópia no `range` | `for _, v := range s { v++ }` | `for i := range s { s[i]++ }` |
| `break` dentro de `switch` em laço | Sai só do switch | `break label` |
| Esperar ordem fixa no `range` de map | Ordem muda a cada execução | Ordene as chaves antes (ver [[Maps]]) |
| Índice de string como posição do caractere | `range "café"` pula de 3 para... | O índice é o offset em **bytes** |
| `else` na linha de baixo | Erro de sintaxe | `} else {` |
| `else` depois de `return` | Indentação desnecessária | Remova o `else` |
| Código antigo (go ≤ 1.21) com closures em laço | Todas veem o último valor | Atualize a linha `go` do `go.mod` |
| `fallthrough` esperando testar a condição | Executa o próximo case **sem testar** | Use vários valores num case |

---

## 📋 Resumo

| Estrutura | Pontos-chave |
| --- | --- |
| `if` | Sem parênteses, chaves obrigatórias, declaração curta opcional (`if v, err := f(); err != nil`) |
| Guard clauses | Trate o erro e saia; caminho feliz sem indentação |
| `for` clássico | `for init; cond; pós { }` |
| `for` condição | `for cond { }` (o "while") |
| `for` infinito | `for { }` + `break` |
| `for range` | slice, array, string (runes), map (ordem aleatória), channel, inteiro (1.22), função (1.23) |
| Variável de laço | Nova a cada iteração desde o Go 1.22 |
| Labels | `break label` / `continue label` para laços aninhados |
| `switch` | Sem `break`, vários valores por case, qualquer expressão, sem expressão = `if/else` |
| `fallthrough` | Executa o próximo case sem testar; raro |
| `goto` | Restrito; quase nunca necessário |
| Por dentro | CFG, saltos condicionais, jump tables, preempção por `SIGURG` |

---

## 🔗 Conexões

**Dentro de Go**
- Escopo dos blocos implícitos → [[Sintaxe, Blocos e Escopo]]
- Condições booleanas → [[Operadores de Comparação e Lógicos]]
- `range` em detalhe → [[Strings em Profundidade]], [[Arrays e Slices]], [[Maps]], [[Channels]], [[Iteradores]]
- Closures e a variável de laço → [[Closures e Funções Anônimas]]
- *Type switch* → [[Type Assertion e Type Switch]]
- `select`, o "switch de channels" → [[Select]]
- Retorno antecipado com erros → [[O Padrão error em Go]]
- Preempção de goroutines → [[Scheduler do Go]]

**Sistemas Operacionais**
- [[Processadores]] — saltos condicionais, pipeline e predição de desvios
- [[Introdução ao Escalonamento]] e [[Escalonamento em Sistemas Interativos]] — preempção por interrupção de relógio, que o runtime de Go imita com o sinal `SIGURG`
- [[Syscalls Diversas]] — sinais como mecanismo de notificação assíncrona de processos

---

⬅️ **Anterior:** [[Operadores de Comparação e Lógicos]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Ponteiros]] ➡️
