---
tags:
  - go
  - go/fundamentos
---
# Controle de Fluxo

> Go tem apenas três estruturas de controle de fluxo: `if`, `for` e `switch`. A escolha por essa minimalidade é deliberada — o compilador SSA (Static Single Assignment) de Go pode otimizar essas construções de forma muito mais eficaz quando o conjunto é pequeno e bem definido.
> 

---

## 1. Como o Compilador Processa Controle de Fluxo

Internamente, o compilador de Go converte todo controle de fluxo em um **grafo de blocos básicos** (Control Flow Graph — CFG). Cada bloco básico é uma sequência linear de instruções sem branches, com exatamente uma entrada e uma ou mais saídas condicionais.

```
// Este código:
if x > 0 {
    return "positivo"
} else {
    return "não positivo"
}

// Vira este CFG:
[bloco 1: CMP x, 0]
      |           |
   (x > 0)     (x <= 0)
      |              |
[bloco 2:        [bloco 3:
 return           return
 "positivo"]     "não positivo"]
```

O compilador aplica otimizações como **dead code elimination** (remove código que nunca executa), **branch prediction hints**, e em alguns casos converte branches em instruções de movimentação condicional (**CMOV**) para evitar pipeline stalls.

---

## 2. `if` / `else if` / `else`

### Forma Básica

```go
idade := 20

if idade >= 18 {
	fmt.Println("maior de idade")
}
```

> ⚠️ As chaves `{}` são **obrigatórias** em Go, mesmo para blocos de uma linha. Isso é uma decisão de design que elimina toda uma classe de bugs do tipo `goto fail` que afetou o iOS/macOS.
> 

### Cadeia `if` / `else if` / `else`

```go
nota := 75

if nota >= 90 {
	fmt.Println("A")
} else if nota >= 80 {
	fmt.Println("B")
} else if nota >= 70 {
	fmt.Println("C")
} else {
	fmt.Println("Reprovado")
}
```

### `if` com Declaração Curta (Init Statement)

Go permite um **init statement** antes da condição, separado por `;`. A variável declarada existe apenas dentro do escopo do `if/else`:

```go
// Sintaxe: if inicialização; condição { }
if err := fazerAlgo(); err != nil {
	fmt.Println("erro:", err)
	return
}
// err não existe aqui fora — escopo encerrado

// Muito comum com funções que retornam (valor, error)
if usuario, err := buscarUsuario(id); err != nil {
	log.Fatal(err)
} else {
	fmt.Println(usuario.Nome)   // usuario só existe no if/else
}
```

> 💡 O init statement é compilado como um bloco simples: a variável entra no escopo, a condição é avaliada, e o bloco é executado. Não há custo extra — o compilador cria apenas uma variável de registro.
> 

---

## 3. `for` — O Único Loop de Go

Go tem apenas `for`, mas cobre todos os casos. Internamente, todos os formatos compilam para o mesmo bytecode de loop com jump condicional.

### Loop Clássico (três componentes)

```go
// init; condition; post
for i := 0; i < 5; i++ {
	fmt.Println(i)   // 0, 1, 2, 3, 4
}

// i++ em Go é uma *statement*, não uma *expression*
// Portanto: for i := 0; i < n; i++ = padrão correto
// Mas: for i := 0; i < n; i += 2 também funciona

// Múltiplas variáveis (usando vírgula)
for i, j := 0, 10; i < j; i, j = i+1, j-1 {
	fmt.Println(i, j)
}
```

### Loop Estilo `while`

Quando apenas a condição é fornecida:

```go
n := 1
for n < 100 {
	n *= 2
}
fmt.Println(n)   // 128

// O compilador trata isso exatamente como while(n < 100) em C
// gerando: loop_start: CMP n, 100; JGE loop_end; n *= 2; JMP loop_start
```

### Loop Infinito

```go
for {
	// equivale a for true { } — o compilador não emite checagem da condição
	entrada := lerEntrada()
	if entrada == "sair" {
		break
	}
	processar(entrada)
}
```

### `for range` — Iteração sobre Coleções

O `range` é açúcar sintático que o compilador expande para acesso por índice. Ele itera sobre arrays, slices, maps, strings, channels e inteiros (Go 1.22+):

```go
// Slice — retorna índice e valor (cópia do elemento)
frutas := []string{"maçã", "banana", "laranja"}
for i, fruta := range frutas {
	fmt.Printf("%d: %s\n", i, fruta)
}

// ⚠️ O valor é uma CÓPIA — modificar não altera o slice original
for _, v := range frutas {
	v = strings.ToUpper(v)   // não altera frutas!
}
// Para modificar: use o índice
for i := range frutas {
	frutas[i] = strings.ToUpper(frutas[i])
}

// Map — retorna chave e valor (ordem aleatória — garantido pela spec)
idades := map[string]int{"Alice": 30, "Bob": 25}
for nome, idade := range idades {
	fmt.Printf("%s: %d\n", nome, idade)
}

// String — itera sobre runes (código Unicode), NÃO bytes
// O índice é o offset de BYTES, não o índice do caractere!
for i, r := range "café" {
	fmt.Printf("byte[%d]: %c (%d)\n", i, r, r)
}
// byte[0]: c (99)
// byte[1]: a (97)
// byte[2]: f (102)
// byte[3]: é (233)  ← byte[3] e byte[4] formam 'é' em UTF-8

// Channel — lê valores até o canal ser fechado
for valor := range canal {
	fmt.Println(valor)
}

// Inteiro (Go 1.22+) — itera de 0 até n-1
for i := range 5 {
	fmt.Println(i)   // 0, 1, 2, 3, 4
}
```

> 💡 Go 1.22 mudou o comportamento da variável de loop: cada iteração cria uma **nova variável**, eliminando a armadilha clássica de closures em loops que capturavam a mesma variável.
> 

---

## 4. `break` e `continue`

### `break` — Sai do Loop Mais Interno

```go
for i := 0; i < 10; i++ {
	if i == 5 {
		break
	}
	fmt.Println(i)   // 0, 1, 2, 3, 4
}
```

### `continue` — Pula para a Próxima Iteração

```go
for i := 0; i < 10; i++ {
	if i%2 == 0 {
		continue   // pula os pares
	}
	fmt.Println(i)   // 1, 3, 5, 7, 9
}
```

### Labels — `break` e `continue` em Loops Aninhados

Labels permitem especificar qual loop o `break`/`continue` afeta. São a alternativa segura ao `goto` para sair de loops aninhados:

```go
externo:
for i := 0; i < 3; i++ {
	for j := 0; j < 3; j++ {
		if j == 1 {
			continue externo   // pula para próxima iteração do loop externo
		}
		fmt.Printf("i=%d j=%d\n", i, j)
	}
}
// Imprime apenas: i=0 j=0 / i=1 j=0 / i=2 j=0

// break com label — sai do loop externo completamente
busca:
for _, linha := range matriz {
	for _, val := range linha {
		if val == alvo {
			fmt.Println("encontrado!")
			break busca   // sai de ambos os loops
		}
	}
}
```

---

## 5. `switch`

O `switch` de Go é compilado de forma diferente dependendo do número de cases:

- **Poucos cases (≤ 5):** compilado como cadeia de `if/else if` (jump condicional)
- **Muitos cases com valores densos:** compilado como **jump table** (O(1))
- **Cases dispersos ou com ranges:** compilado como **binary search** (O(log n))

```go
diaSemana := 3

switch diaSemana {
case 1:
	fmt.Println("Segunda")
case 2:
	fmt.Println("Terça")
case 3:
	fmt.Println("Quarta")
case 4:
	fmt.Println("Quinta")
case 5:
	fmt.Println("Sexta")
default:
	fmt.Println("Final de semana")
}
```

> 💡 Em Go, o `break` é implícito — cada `case` para automaticamente. Diferente de C/Java onde é fácil esquecer o `break` e causar fall-through acidental.
> 

### Múltiplos Valores por Case

```go
switch diaSemana {
case 1, 2, 3, 4, 5:
	fmt.Println("Dia útil")
case 6, 7:
	fmt.Println("Final de semana")
}
```

### Switch sem Expressão — Substitui if/else if

Quando o `switch` não tem expressão, cada `case` é uma condição booleana. O compilador avalia cada case na ordem até encontrar um `true`:

```go
nota := 85

switch {
case nota >= 90:
	fmt.Println("A")
case nota >= 80:
	fmt.Println("B")
case nota >= 70:
	fmt.Println("C")
default:
	fmt.Println("Reprovado")
}
```

### Switch com Init Statement

```go
switch os := runtime.GOOS; os {
case "darwin":
	fmt.Println("macOS")
case "linux":
	fmt.Println("Linux")
default:
	fmt.Printf("outro: %s\n", os)
}
```

### `fallthrough` — Fall-through Explícito

`fallthrough` executa o próximo case **sem verificar sua condição**. É raro e geralmente indica que um switch deveria ser refatorado:

```go
n := 2

switch n {
case 2:
	fmt.Println("dois")
	fallthrough
case 3:
	fmt.Println("três")   // executado por causa do fallthrough
default:
	fmt.Println("outro")  // não executado
}
// Imprime: dois / três
```

> ⚠️ `fallthrough` não pode ser o último statement de um `switch`. Só funciona em switches com expressão (não em type switches).
> 

### Type Switch

Verifica o tipo dinâmico de uma interface. Internamente, acessa o campo `_type` do header da interface:

```go
// Uma interface em Go tem este layout na memória:
// ┌────────┬────────┐
// │ *itab  │ *data  │
// └────────┴────────┘
// itab contém: ponteiro para tipo + ponteiro para tabela de métodos
// O type switch inspeciona o campo itab._type

func descrever(i interface{}) {
	switch v := i.(type) {
	case int:
		fmt.Printf("inteiro: %d\n", v)
	case string:
		fmt.Printf("string: %q\n", v)
	case bool:
		fmt.Printf("bool: %v\n", v)
	case nil:
		fmt.Println("nil")
	default:
		fmt.Printf("tipo desconhecido: %T\n", v)
	}
}
```

---

## 6. Padrões Comuns

### Early Return — Guarda Clauses

O estilo preferido em Go: verificações de erro e condições de saída primeiro, lógica principal no final sem aninhamento:

```go
// ❌ Estilo "arrow" — difícil de ler
func processar(usuario *Usuario) error {
	if usuario != nil {
		if usuario.Ativo {
			if usuario.Email != "" {
				// lógica aqui...
				return nil
			}
		}
	}
	return errors.New("dados inválidos")
}

// ✅ Guard clauses — estilo Go idiomático
func processar(usuario *Usuario) error {
	if usuario == nil {
		return errors.New("usuário nulo")
	}
	if !usuario.Ativo {
		return errors.New("usuário inativo")
	}
	if usuario.Email == "" {
		return errors.New("email obrigatório")
	}

	// lógica principal sem aninhamento
	return nil
}
```

### Switch como Dispatcher de Comandos

```go
type Comando string

const (
	CmdCriar   Comando = "criar"
	CmdEditar  Comando = "editar"
	CmdDeletar Comando = "deletar"
)

func executar(cmd Comando, dados string) error {
	switch cmd {
	case CmdCriar:
		return criar(dados)
	case CmdEditar:
		return editar(dados)
	case CmdDeletar:
		return deletar(dados)
	default:
		return fmt.Errorf("comando desconhecido: %q", cmd)
	}
}
```

---

## 7. Resumo Comparativo

| Construção | Go | Equivalente em outras linguagens |
| --- | --- | --- |
| Loop com contador | `for i := 0; i < n; i++` | `for (i = 0; i < n; i++)` |
| Loop while | `for condicao { }` | `while (cond) { }` |
| Loop infinito | `for { }` | `while (true) { }` |
| Loop sobre coleção | `for i, v := range s` | `forEach`, `for...of`, `each` |
| Loop sobre inteiro | `for i := range n` (Go 1.22+) | `for i in range(n)` (Python) |
| Condicional | `if / else if / else` | igual |
| Seleção múltipla | `switch` com break implícito | `switch` com break manual |
| Sair do loop externo | `break label` | `break label` (JS), limitado em outras |
| Fall-through | `fallthrough` (explícito) | comportamento padrão em C/Java |

---

## 8. Por Dentro: Controle de Fluxo, CPU e Scheduler

### `for` → CMP + branch instruction

```
for i := 0; i < 5; i++

Código Go           x86-64 assembly
──────────          ────────────────────────────────
                    MOV   rbx, 0          ; i = 0
loop_start:
  i < 5       →     CMP   rbx, 5
                    JGE   loop_end        ; se i >= 5 → sai

  // corpo do loop

  i++         →     ADD   rbx, 1
                    JMP   loop_start      ; volta ao início
loop_end:

Branch prediction no loop:
  Primeiras N-1 iterações: branch predictor prevê "loop continua" → acerta
  Última iteração: predictor prevê errado (sai do loop) → mispredict
  Custo: ~1 mispredict por execução do loop (geralmente desprezível)

Loop unrolling (otimização do compilador):
  for i := 0; i < 8; i++ { arr[i] = 0 }
  → compilador pode "desenrolar" para:
     arr[0]=0; arr[1]=0; arr[2]=0; arr[3]=0; (sem branch!)
     arr[4]=0; arr[5]=0; arr[6]=0; arr[7]=0;
```

Conecta com [[Processadores]] — conditional jumps, branch prediction unit, loop unrolling, instruction pipeline.

### `if/else` → branches e CMOV

```go
// if/else simples
if x > 0 {
    y = 1
} else {
    y = -1
}

// Compilação 1: com branch (JG/JLE)
    CMP  rax, 0
    JLE  else_branch
    MOV  rbx, 1
    JMP  end
else_branch:
    MOV  rbx, -1
end:

// Compilação 2: branchless com CMOV (quando o compilador julga melhor)
    CMP  rax, 0
    MOV  rbx, -1          ; valor else
    MOV  rcx, 1           ; valor then
    CMOVG rbx, rcx        ; Conditional MOVe if Greater: rbx = rcx se x > 0

// CMOV não cria branch → sem mispredict penalty
// Usado quando: corpo do if/else é simples e condição é imprevisível
```

Conecta com [[Processadores]] — pipeline stalls de branch misprediction, instruções CMOV (Conditional Move) que eliminam branches.

### `switch` → jump table vs cadeia de comparações

```
switch denso (valores contínuos 1..7):

  switch n {
  case 1: ...    case 2: ...    case 3: ...
  case 4: ...    case 5: ...    case 6: ...    case 7: ...
  }

Compilado como JUMP TABLE (O(1)):
  ┌────────────────────────────────────────────────┐
  │  table[0] = addr(case1)                        │
  │  table[1] = addr(case2)                        │
  │  ...                                           │
  │  table[6] = addr(case7)                        │
  └────────────────────────────────────────────────┘
  
  Assembly:
    CMP   rax, 7          ; bounds check
    JA    default
    JMP   [table + rax*8] ; indirect branch pelo índice — O(1)!

switch esparso (valores dispersos):
  switch n {
  case 1: ...    case 100: ...    case 10000: ...
  }
  
  → cadeia de CMP + JE: O(n) no pior caso
  → ou binary search: O(log n) se o compilador detectar suficientes casos

switch em strings:
  switch s {
  case "GET": ...    case "POST": ...    case "DELETE": ...
  }
  
  → série de comparações de string (MEMCMP ou comparação byte-a-byte)
  → sem jump table (strings não têm valor numérico direto)
  → O(n) casos × O(len) por comparação
```

Conecta com [[Processadores]] — jump table (indexed indirect branch), branch prediction para indirect jumps, comparações de memória.

### `goto` → JMP incondicional

```go
goto cleanup

// ...

cleanup:
    os.Remove(tmpFile)

// Compila para:
    JMP   cleanup_label    ← instrução de salto incondicional
    ; ...código entre ...
cleanup_label:
    ; código de cleanup

// goto em Go tem restrições que C não tem:
//   ✅ Dentro da mesma função
//   ❌ Não pode pular sobre declarações de variáveis
//   ❌ Não pode entrar em bloco (só sair)
// Uso idiomático: sair de múltiplos loops sem label aninhado,
//                 ou cleanup em código sem defer
```

Conecta com [[Processadores]] — instrução JMP (unconditional jump), salto direto para endereço, sem condição checada.

### Preempção de goroutines em loops e back-edges

```
Go 1.14+ introduziu preempção assíncrona de goroutines:

// Goroutine com loop infinito (antes do Go 1.14 — travava o scheduler!)
for {
    // sem function call → sem safe point → scheduler não conseguia preemptir
    contador++
}

// Desde Go 1.14: o compilador insere safe points em back-edges de loops:
loop_start:
    // verificação de preempção inserida automaticamente:
    // if *stackguard0 == stackPreempt { CALL runtime.morestack }
    
    contador++
    JMP loop_start

// Preempção assíncrona (via SIGURG no Linux):
//   1. runtime envia SIGURG para thread OS que roda a goroutine
//   2. signal handler salva contexto (PC, SP, registradores)
//   3. goroutine é marcada como preempted
//   4. scheduler escolhe outra goroutine para rodar

Safe points:
  - chamadas de função (traditional Go)
  - back-edges de loops (Go 1.14+ async preemption)
```

Conecta com [[Processos]] — preemptive scheduling, context switch. O scheduler Go precisou de suporte do compilador (inserção de safe points em loops) para preempção assíncrona — análogo ao suporte de hardware (timer interrupt) para preempção de processos pelo kernel.

### `range` sobre channel → bloqueio e wakeup de goroutine

```go
for v := range ch {   // bloqueia quando ch está vazio
    processar(v)
}

Internamente:

goroutine bloqueada em range/channel:
  1. ch está vazio → goroutine chama runtime.chanrecv
  2. runtime.chanrecv: sem dados disponíveis
     → goroutine vai para estado "waiting" (parked)
     → goroutine é removida da run queue do scheduler
     → thread OS fica livre para rodar outra goroutine

  3. outro goroutine envia valor: ch <- valor
     → runtime.chansend: goroutine waiting detectada
     → goroutine é movida de "waiting" → "runnable"
     → inserida na run queue do scheduler
     → próxima oportunidade: goroutine executa processar(v)

  4. ch fechado → range termina (loop exit)

Estados de goroutine (análogo a estados de processo):
  runnable  ← → running  ← → waiting (parked)
```

Conecta com [[Estados de Processos]] — estado "bloqueado" vs "pronto" vs "executando". Goroutines em channels bloqueados são análogos a processos bloqueados esperando I/O.

---

## 9. Conexão com Sistemas Operacionais

- **[[Processadores]]**: `for` compila para `CMP` + `JGE`/`JL` (loop back-edge). `if/else` gera branches condicionais ou `CMOV` (branchless quando o compilador julga mais eficiente). `switch` com valores inteiros densos compila para jump table — O(1) dispatch via `JMP [table + idx*8]`. `goto` é `JMP` incondicional.

- **[[Processadores]]** (branch prediction): O branch predictor aprende o padrão dos loops (N-1 iterações continuam, 1 sai). Condições de `if/else` imprevisíveis causam misprediction penalty de ~15 ciclos — compilador pode usar `CMOV` para evitar o branch nesses casos.

- **[[Processos]]** (preempção): Desde Go 1.14, o compilador insere verificações de preempção em back-edges de loops. Antes, um loop sem chamada de função travava o scheduler. A preempção assíncrona usa `SIGURG` — análogo ao timer interrupt que o kernel usa para preempção de processos.

- **[[Estados de Processos]]**: Goroutines em `for v := range ch` vão para estado "waiting" quando o canal está vazio — o scheduler remove a goroutine da run queue. Quando outro goroutine envia para o canal, a goroutine bloqueada volta para "runnable". Exato análogo ao estado bloqueado/pronto de processos esperando I/O.

- **[[Processadores]]** (switch/strings): `switch` em strings não usa jump table — strings não têm valor numérico direto. O compilador emite comparações sequenciais (MEMCMP por caso). Para dispatcher de comandos HTTP com muitos verbos, um `map[string]handler` pode ser mais eficiente que `switch` com muitos casos de string.