---
tags:
  - go
  - go/pacotes
aliases:
  - Pacotes em Go
  - Packages
  - init()
  - internal
  - go:embed
ordem: 30
modulo: "07 - Pacotes e Módulos"
---
# Criando Pacotes

> **Nota 30 da trilha de Go.** Até aqui escrevemos quase tudo num único arquivo `main.go`. Programas reais são divididos em **pacotes**: unidades de código com um propósito, que escondem detalhes e expõem uma API. Nesta nota veremos o que é um pacote, como ele se relaciona com pastas e arquivos, as regras de **importação** (caminhos, aliases, imports "em branco"), a pasta especial **`internal/`**, a **ordem de inicialização** (variáveis de pacote e `init()`), a proibição de **ciclos**, como organizar e nomear pacotes, testes "de fora" do pacote e como **embutir arquivos** no binário com `//go:embed`.
>
> **Pré-requisitos:** [[Convenções da linguagem]], [[Sintaxe, Blocos e Escopo]], [[Interfaces]].

---

## 🧭 De onde viemos

Em [[Ambiente e Primeiro Programa]] vimos que `package main` gera um executável e que todo arquivo começa declarando seu pacote. Em [[Convenções da linguagem]] vimos a regra da letra maiúscula e um panorama de layout de projeto. Em [[Sintaxe, Blocos e Escopo]] vimos o **bloco do pacote** (compartilhado por todos os arquivos) e o **bloco do arquivo** (onde ficam os imports). E em [[Como Go Funciona]] vimos que o compilador gera um objeto **por pacote** e que `init()` roda antes de `main()`. Agora vamos amarrar tudo.

---

## 📦 1. O que é um pacote

> 💡 **Pacote:** um conjunto de arquivos `.go` que ficam **na mesma pasta** e declaram o **mesmo nome de pacote**. É a unidade de **organização** (agrupa código relacionado), de **encapsulamento** (a regra da maiúscula vale na fronteira do pacote) e de **compilação** (o compilador processa um pacote por vez).

```
app/
├── go.mod                       module exemplo.com/app
├── main.go                      package main
└── calc/
    ├── calc.go                  package calc     ┐
    ├── stats.go                 package calc     ├── um pacote só: "calc"
    └── calc_test.go             package calc     ┘
```

Regras:

| Regra | Detalhe |
| --- | --- |
| Uma pasta = um pacote | Todos os `.go` de uma pasta precisam ter o mesmo `package` (exceção: testes `_test`) |
| Arquivos compartilham tudo | Funções, tipos e variáveis de um arquivo são visíveis nos outros do mesmo pacote, **sem import** |
| Imports são por arquivo | Cada arquivo importa o que usa |
| Nome do pacote ≠ caminho | O caminho de import é `exemplo.com/app/calc`; o nome usado no código é o declarado em `package calc` (por convenção, o último elemento do caminho) |
| Subpastas são pacotes independentes | `calc/avancado` **não** tem acesso especial a `calc`: é outro pacote |

> 💡 Repare no último ponto: em Go **não existe hierarquia** de pacotes. `net/http` e `net` são pacotes separados, e `http` só enxerga o que `net` exporta. O caminho com barras é só organização de pastas.

---

## 📥 2. Importando pacotes

### 2.1 O caminho de import

```go
import (
	"fmt"                              // biblioteca padrão: caminho curto
	"net/http"                         // biblioteca padrão com subpasta

	"github.com/google/uuid"           // módulo externo: começa com um domínio

	"exemplo.com/app/calc"             // pacote do PRÓPRIO módulo: module path + pasta
)
```

O caminho de um pacote do seu módulo é o **module path** (do `go.mod`) seguido da pasta. Veremos módulos em [[Sistema de Módulos (go mod)]].

### 2.2 Formas especiais de import

```go
import (
	calcv2 "exemplo.com/app/v2/calc"   // ALIAS: usar como calcv2.Somar (resolve conflito de nomes)
	_ "github.com/lib/pq"              // IMPORT EM BRANCO: só para executar o init() do pacote
	_ "embed"                          // idem: necessário para usar //go:embed com string/[]byte
	. "math"                           // DOT IMPORT: usar Sqrt em vez de math.Sqrt (evite!)
)
```

| Forma | Uso legítimo |
| --- | --- |
| Alias | Dois pacotes com o mesmo nome (`crypto/rand` e `math/rand`), ou versões diferentes |
| `_` | Registrar drivers de banco (`database/sql`), formatos de imagem (`image/png`), `net/http/pprof` |
| `.` | Quase nunca; às vezes em testes. Esconde de onde vem cada nome |

O import em branco existe porque Go proíbe imports não usados. Ele diz: "não vou usar nenhum nome deste pacote, só quero que ele seja inicializado".

```go
import (
	"database/sql"
	_ "github.com/jackc/pgx/v5/stdlib"   // o init() deste pacote chama sql.Register("pgx", driver)
)

db, err := sql.Open("pgx", dsn)          // agora "pgx" é um driver registrado
```

---

## 🔒 3. Encapsulamento: exportado, não exportado e `internal/`

### 3.1 A regra da maiúscula (revisão)

Vimos em [[Convenções da linguagem]]: nomes com letra **maiúscula** são exportados; minúscula, privados ao pacote. Pense na API do pacote como **tudo o que começa com maiúscula**. Mantenha-a pequena: cada nome exportado é uma promessa de compatibilidade.

### 3.2 A pasta `internal/`: privado ao módulo

Às vezes um pacote precisa ser usado por **vários pacotes do seu projeto**, mas não deve ser usado por **ninguém de fora**. Para isso, o compilador dá um significado especial à pasta `internal`:

> **Um pacote dentro de uma pasta `internal/` só pode ser importado por código que esteja na árvore de pastas do PAI de `internal`.**

```
exemplo.com/app/
├── internal/
│   └── segredo/          ← importável por exemplo.com/app/...
├── calc/                 ← pode importar internal/segredo ✅
└── main.go               ← pode importar internal/segredo ✅

exemplo.com/outro/        ← NÃO pode importar exemplo.com/app/internal/segredo ❌
```

Saída real, tentando importar de outro módulo:

```
package exemplo.com/outro
	main.go:6:2: use of internal package exemplo.com/app/internal/segredo not allowed
```

> ✅ Coloque em `internal/` tudo que **não** for API pública. Assim você pode mudar à vontade sem quebrar usuários do seu módulo. É a forma de Go de ter "pacotes privados".

---

## 🚀 4. Inicialização de pacotes

### 4.1 A ordem completa

Antes de `main()` rodar, todos os pacotes precisam ser inicializados. Saída real de um programa com dois pacotes (`main` importa `calc`):

```
  [calc] inicializando var Versao (base = 10 )
  [calc] init() 1
  [calc] init() 2
  [main] var x
  [main] init()
main() começou
```

As regras:

```
1. Os pacotes são inicializados em ordem de DEPENDÊNCIA:
   se main importa calc, calc é inicializado ANTES de main.
   (cada pacote é inicializado uma única vez, mesmo que importado por vários)

2. Dentro de um pacote:
   a) as VARIÁVEIS de pacote são inicializadas, na ordem das dependências entre elas
      (e, sem dependência, na ordem em que aparecem nos arquivos)
   b) depois, as funções init(), na ordem em que aparecem
      (os arquivos são apresentados ao compilador em ordem alfabética do nome)

3. Por último: main.main()
```

Repare no detalhe do item 2a, visível na saída: `Versao` foi declarada **antes** de `base` no arquivo, mas a função que inicializa `Versao` **usa** `base`, então `base` foi inicializada primeiro (por isso apareceu `base = 10` e não `0`).

```go
var Versao = carregarVersao()   // depende de base (via a função)
var base = 10                   // inicializada ANTES, apesar de vir depois no texto
```

### 4.2 A função `init()`

```go
func init() {
	// configurar algo que não dá para expressar numa declaração de variável
}
```

| Regra | Detalhe |
| --- | --- |
| Assinatura | Sem parâmetros e sem resultados |
| Quantidade | Pode haver **várias** por pacote e até várias no mesmo arquivo |
| Chamada | Automática; **não** pode ser chamada nem referenciada pelo seu código |
| Momento | Depois das variáveis do pacote, antes de quem importa o pacote |
| Goroutine | Roda na goroutine principal, uma de cada vez |

Usos legítimos:

- Registrar algo num registro global (drivers, codecs, formatos de imagem, plugins)
- Inicializar tabelas que exigem código (pré-calcular uma tabela de lookup)
- Verificar pré-condições do ambiente

> ⚠️ **Evite `init()` com efeitos colaterais pesados** (abrir conexões de rede, ler arquivos de configuração, iniciar goroutines). Eles tornam o programa difícil de testar (o `init` roda em todo teste do pacote), deixam a inicialização lenta e escondem dependências. Prefira inicialização **explícita** chamada pelo `main` (`cfg, err := config.Carregar()`), que pode devolver erro.

### 4.3 Por dentro

O compilador gera, para cada pacote, uma "tarefa de inicialização" com as inicializações de variáveis e os `init()`s, e o linker monta a lista em ordem de dependência. O `runtime.main` percorre essa lista antes de chamar `main.main` (vimos a sequência completa em [[Como Go Funciona]]). É a mesma ideia das seções `.init_array` de binários ELF em C/C++, onde construtores globais rodam antes do `main`.

---

## 🔁 5. Ciclos de importação são proibidos

Se o pacote `a` importa `b`, `b` **não pode** importar `a` (nem indiretamente). Saída real:

```
package exemplo.com/app/a
	imports exemplo.com/app/b from a.go
	imports exemplo.com/app/a from b.go: import cycle not allowed
```

Por quê?

1. **Inicialização:** com ciclo, não existe uma ordem válida de inicialização (quem vem primeiro?)
2. **Compilação rápida e paralela:** o grafo de pacotes acíclico permite compilar em ordem, cada pacote uma vez, e pacotes independentes em paralelo (uma das razões do build rápido, ver [[Como Go Funciona]])
3. **Design:** ciclos indicam que os dois pacotes são, na verdade, um só, ou que falta uma abstração

Como quebrar um ciclo:

| Técnica | Como |
| --- | --- |
| Extrair o que é comum | Criar um pacote `c` com os tipos compartilhados; `a` e `b` importam `c` |
| Inverter a dependência com interface | `b` define uma interface com o que precisa de `a`; `a` satisfaz implicitamente; quem monta tudo (o `main`) liga os dois |
| Juntar os pacotes | Se estão tão acoplados, talvez devam ser um só |

```go
// pacote notificacao: precisa de algo que busque e-mails de usuários,
// mas NÃO importa o pacote usuario (que importa notificacao)
type buscadorEmail interface {
	Email(idUsuario string) (string, error)
}
```

É a satisfação implícita de interfaces ([[Interfaces]]) resolvendo um problema de arquitetura.

---

## 🗂️ 6. Organizando e nomeando pacotes

### 6.1 Nomes

Vimos as regras em [[Convenções da linguagem]]: curto, minúsculo, uma palavra, sem `_`, substantivo, e **sem repetir** o nome do pacote nos identificadores (`http.Client`, não `http.HTTPClient`).

```go
package util       // ❌ genérico: "útil para quê?"
package common     // ❌
package models     // ❌ organiza por tipo técnico, não por assunto

package pedido     // ✅ assunto claro
package slug       // ✅
package retry      // ✅
```

### 6.2 Organize por domínio, não por camada

```
❌ por camada técnica (hábito de MVC)        ✅ por domínio/funcionalidade
app/                                         app/
├── controllers/                             ├── cmd/api/main.go
├── models/                                  └── internal/
├── services/                                    ├── pedido/      (tipos, regras, repositório)
└── repositories/                                ├── pagamento/
                                                 ├── estoque/
                                                 └── platform/    (banco, http, log)
```

Por camada, toda mudança numa funcionalidade toca 4 pacotes, e os pacotes viram dependências de todos (ciclos aparecem rápido). Por domínio, cada pacote é coeso.

### 6.3 O pacote `main` deve ser fino

```go
// cmd/api/main.go
func main() {
	if err := run(context.Background(), os.Args, os.Getenv); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}

func run(ctx context.Context, args []string, getenv func(string) string) error {
	cfg, err := config.Carregar(getenv)
	if err != nil {
		return err
	}
	db, err := banco.Conectar(ctx, cfg.DSN)
	if err != nil {
		return err
	}
	defer db.Close()
	srv := api.NovoServidor(cfg, pedido.NovoServico(db))
	return srv.Executar(ctx)
}
```

O `main` só **monta** as peças (ligação das dependências). Toda a lógica fica em pacotes testáveis.

---

## 📝 7. Documentação do pacote

O comentário imediatamente antes de `package` documenta o pacote inteiro. Em pacotes grandes, é comum um arquivo `doc.go` só para isso:

```go
// Package calc fornece operações aritméticas com tratamento seguro
// de overflow e divisão por zero.
//
// Uso básico:
//
//	soma := calc.Somar(2, 3)
package calc
```

```bash
go doc exemplo.com/app/calc       # documentação do pacote
go doc calc.Somar                 # de um símbolo
```

Pacotes públicos aparecem automaticamente em **pkg.go.dev**. Veremos os detalhes de formato em [[Convenções da linguagem]].

---

## 🧪 8. Testes: "de dentro" e "de fora" do pacote

Arquivos `_test.go` podem declarar dois nomes de pacote na mesma pasta:

```go
// calc_test.go
package calc                // teste "caixa-branca": vê tudo, inclusive o não exportado

// calc_externo_test.go
package calc_test           // teste "caixa-preta": um pacote separado que importa calc
import "exemplo.com/app/calc"
```

O pacote `_test` só enxerga a API pública, exatamente como um usuário real. É ótimo para garantir que a API faz sentido e para escrever **exemplos** executáveis. Veremos em [[Testes com go test]].

---

## 📎 9. Embutindo arquivos no binário: `//go:embed` (Go 1.16+)

Templates HTML, arquivos estáticos, migrações SQL e configurações padrão podem ir **dentro** do binário, mantendo a vantagem do deploy de um arquivo só. Saída real (o arquivo `versao.txt` contém `v2.3.4`):

```go
import _ "embed"

//go:embed versao.txt
var versaoArquivo string            // o conteúdo do arquivo, em compilação

fmt.Println(versaoArquivo)          // v2.3.4
```

```go
import "embed"

//go:embed templates/*.html static
var arquivos embed.FS               // um sistema de arquivos somente-leitura com vários arquivos

tmpl := template.Must(template.ParseFS(arquivos, "templates/*.html"))
http.Handle("/static/", http.FileServerFS(arquivos))   // servir arquivos estáticos embutidos
```

| Tipo da variável | Conteúdo |
| --- | --- |
| `string` | Um arquivo, como texto |
| `[]byte` | Um arquivo, como bytes |
| `embed.FS` | Vários arquivos/pastas, acessados como sistema de arquivos (`fs.FS`) |

Regras: a diretiva fica **imediatamente acima** de uma variável de pacote; os caminhos são relativos à pasta do arquivo `.go` e não podem sair dela (`..`); arquivos começando com `.` ou `_` são ignorados, a menos que se use o prefixo `all:`.

> 💡 Os dados embutidos vão para a seção de dados somente-leitura do binário (`.rodata`), que você viu em [[Arquivos de Cabeçalho e o Modelo de Execução]]. Na imagem Docker, isso significa que um `FROM scratch` com o binário já carrega templates e arquivos estáticos (ver [[Imagens]]).

---

## ⚠️ 10. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Dois nomes de pacote na mesma pasta | `found packages a and b in ...` | Uma pasta, um pacote (só `_test` é exceção) |
| Esperar que subpasta veja o privado do pai | `undefined` | Subpastas são pacotes independentes |
| Ciclo de importação | `import cycle not allowed` | Extraia um pacote comum ou use interfaces |
| Import de `internal/` de outro módulo | `use of internal package ... not allowed` | É intencional: use a API pública |
| `init()` com I/O e rede | Testes lentos, erros sem tratamento | Inicialização explícita no `main` |
| Depender da ordem entre `init()`s de arquivos diferentes | Frágil | Não dependa; use variáveis com dependências explícitas |
| Pacotes `util`/`common`/`helpers` | Pacote "gaveta de bagunça" | Nomeie pelo propósito |
| `//go:embed` sem `import _ "embed"` (com string/[]byte) | Erro de compilação | Importe `embed` |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Pacote | Arquivos de uma pasta com o mesmo `package`; compartilham o escopo |
| Caminho de import | Module path + pasta; sem hierarquia entre pacotes |
| Imports especiais | Alias, `_` (só init), `.` (evite) |
| Encapsulamento | Maiúscula = exportado; `internal/` = privado ao módulo |
| Inicialização | Dependências primeiro; variáveis (por dependência) → `init()`s → `main` |
| `init()` | Várias por pacote; automáticas; evite efeitos pesados |
| Ciclos | Proibidos; quebre com pacote comum ou interfaces |
| Organização | Por domínio; `main` fino; `cmd/` + `internal/` |
| Testes | `package x` (caixa-branca) e `package x_test` (caixa-preta) |
| `//go:embed` | Arquivos dentro do binário (`string`, `[]byte`, `embed.FS`) |

---

## 🔗 Conexões

**Dentro de Go**
- Módulos, versões e dependências → [[Sistema de Módulos (go mod)]]
- Pacotes que você usará sempre → [[Pacotes Importantes da Stdlib]]
- Regra da maiúscula e layout → [[Convenções da linguagem]]
- Bloco do pacote e do arquivo → [[Sintaxe, Blocos e Escopo]]
- `runtime.main` e a ordem de inicialização → [[Como Go Funciona]]
- Interfaces para quebrar dependências → [[Interfaces]]
- Testes caixa-branca e caixa-preta → [[Testes com go test]]
- Build tags e arquivos por plataforma → [[Build, Flags e Cross-compilation]]

**Sistemas Operacionais**
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — em C, a interface de um módulo é o `.h` e a inicialização global usa `.init_array`; em Go, a interface é o conjunto de nomes exportados e a ordem de inicialização é calculada pelo compilador
- [[Sistema de Camadas]] e [[Micronúcleos]] — dividir um sistema em módulos com dependências numa só direção, a mesma disciplina do grafo acíclico de pacotes

**Docker**
- [[Imagens]] e [[Dockerfile]] — com `//go:embed`, templates e arquivos estáticos vão dentro do binário, permitindo imagens mínimas

---

⬅️ **Anterior:** [[Wrapping e Unwrapping de Erros]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Sistema de Módulos (go mod)]] ➡️
