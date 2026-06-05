---
tags:
  - go
  - go/módulos
---
# Sistema de Módulos (go mod)

> O sistema de módulos é o gerenciador de dependências oficial de Go desde 1.11. Cada módulo tem um nome único (geralmente URL), um arquivo `go.mod` com dependências versionadas, e um `go.sum` com checksums criptográficos. Builds são 100% reproduzíveis.
> 

---

## 1. O Que é um Módulo

```
Módulo = conjunto de pacotes Go com:
- Nome único (module path)
- Versão semântica
- Arquivo go.mod declarando dependências

Repositório GitHub: github.com/usuario/projeto
    ↓
Módulo Go:          module github.com/usuario/projeto
    ↓
Pacotes:            github.com/usuario/projeto/internal/auth
                    github.com/usuario/projeto/pkg/logger

Todos os arquivos .go no diretório interno a go.mod
fazem parte do mesmo módulo.
```

---

## 2. Criar um Módulo

```bash
# Inicializar — cria go.mod
go mod init github.com/usuario/meu-projeto

# Para projetos locais/privados também funciona
go mod init meuapp
```

**`go.mod` criado:**

```go
module github.com/usuario/meu-projeto

go 1.24   // versão mínima do Go — garante comportamentos específicos
```

---

## 3. O Arquivo `go.mod` em Detalhe

```go
module github.com/usuario/meu-projeto

// go directive: versão mínima necessária E ativa features de cada versão
// go 1.21: inicialization ordering garantido, novas builtins (min, max, clear)
// go 1.22: variável de loop por iteração, range over integers
// go 1.23: range over iterators (iter.Seq), slices.All/Values/Collect
// go 1.24: benchmark b.Loop(), weak pointers, finalizers
go 1.24

// toolchain directive (Go 1.21+): versão exata da toolchain
// Se a toolchain disponível for menor, Go baixa automaticamente
toolchain go1.24.0

require (
	// Dependências diretas
	github.com/gin-gonic/gin v1.9.1
	golang.org/x/sync v0.6.0

	// Dependências indiretas (das suas deps) — geradas pelo go mod tidy
	github.com/bytedance/sonic v1.10.2 // indirect
)

// Substituir dependência por versão local (durante desenvolvimento)
replace github.com/meu/pacote => ../pacote-local

// Excluir versão específica com bug
exclude github.com/algum/pacote v1.2.3
```

---

## 4. O Arquivo `go.sum` — Verificação de Integridade

```bash
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQnU5cB7...=
github.com/gin-gonic/gin v1.9.1/go.mod h1:hPLL8HtnrpJm...=
```

Cada linha: `módulo versão hash`

- `h1:` = hash SHA-256 do conteúdo do zip do módulo (base64)
- Verificado contra o **Checksum Database** do Google (`sum.golang.org`)
- Imutável — não edite manualmente

> ⚠️ **Sempre faça commit do `go.sum`**. Ele garante que todos na equipe baixam exatamente o mesmo código. O CI/CD deve falhar se `go.sum` estiver desatualizado.
> 

---

## 5. Comandos Essenciais

```bash
# Inicializar
go mod init module/path

# Adicionar dependência
go get github.com/gin-gonic/gin                  # versão mais recente
go get github.com/gin-gonic/gin@v1.9.1           # versão específica
go get github.com/gin-gonic/gin@latest           # tag "latest"
go get github.com/gin-gonic/gin@main             # branch main
go get github.com/user/repo@abc1234def           # commit específico

# Atualizar
go get -u github.com/gin-gonic/gin               # atualiza (inclui minor)
go get -u=patch github.com/gin-gonic/gin         # apenas patches
go get -u ./...                                  # atualiza todas

# Manutenção — MAIS IMPORTANTE
go mod tidy       # remove não usados, baixa os que faltam, atualiza go.sum
go mod verify     # verifica checksums (detecta corrupção ou alteração)
go mod download   # baixa todas as deps para o cache

# Inspecionar
go list -m all              # todos os módulos usados
go list -m -versions gin    # versões disponíveis
go mod graph                # árvore de dependências
go mod why github.com/pkg   # por que esta dep está incluída?

# Vendor
go mod vendor               # copia deps para vendor/
go build -mod=vendor ./...  # build usando vendor/
```

---

## 6. Versionamento Semântico (SemVer)

```
vMAJOR.MINOR.PATCH

v1.2.3
│ │ └── PATCH — bug fixes sem quebrar API
│ └──── MINOR — novas funcionalidades sem quebrar API
└────── MAJOR — mudanças que quebram a API (breaking changes)
```

### Versão Major v2+ — Import Path Muda

```go
// v1: import normal
import "github.com/usuario/pacote"

// v2+: o path inclui /v2
import "github.com/usuario/pacote/v2"

// go.mod do módulo v2:
module github.com/usuario/pacote/v2

// go.mod do seu projeto:
require github.com/usuario/pacote/v2 v2.1.0
```

### Versões Pre-release e Pseudo-versões

```go
v1.0.0-alpha.1       — pre-release
v1.0.0-rc.2          — release candidate
v0.0.0-20240315-abc1234   — pseudo-versão (sem tag de release)
```

---

## 7. Proxy e Privacidade

```bash
# GOPROXY — proxy de módulos (padrão)
# proxy.golang.org: cache público do Google
# direct: baixa diretamente do VCS
# off: não tenta baixar (falha se não estiver no cache)
export GOPROXY=https://proxy.golang.org,direct

# Para módulos privados — pular o proxy
export GONOSUMCHECK=github.com/empresa/*
export GOPRIVATE=github.com/empresa/*         # pula proxy E sumdb
export GONOSUMDB=github.com/empresa/*         # apenas pula sumdb

# Proxy corporativo
export GOPROXY=https://proxy.corp.com,https://proxy.golang.org,direct

# Verificar
go env GOPROXY
go env GOPRIVATE

# Configurar permanentemente
go env -w GOPROXY=https://proxy.golang.org,direct
go env -w GOPRIVATE=github.com/minhaempresa/*
```

---

## 8. Workspace (Go 1.18+) — Múltiplos Módulos Locais

Workspaces permitem trabalhar com múltiplos módulos locais simultaneamente sem editar `go.mod`:

```bash
# Criar workspace
go work init ./modulo-a ./modulo-b

# Adicionar módulo ao workspace
go work use ./novo-modulo

# Sincronizar (baixa deps dos módulos no workspace)
go work sync
```

**`go.work` criado:**

```go
go 1.24

use (
	./modulo-a
	./modulo-b
	./novo-modulo
)

// Substitui também funcionam em go.work
replace github.com/externo => ../fork-local
```

> 💡 `go.work` permite que `modulo-a` importe `modulo-b` usando o código local, sem precisar de `replace` em cada `go.mod`. Ideal para:
> 

> - Desenvolver uma biblioteca e um projeto que a usa simultaneamente
> 

> - Microserviços que compartilham pacotes internos
> 

> - Bifurcar uma dependência para testar patches
> 

---

## 9. Modo Vendor

```bash
# Copia todas as dependências para vendor/
go mod vendor

# Verificar que vendor/ está sincronizado com go.mod
go mod verify

# Build usando vendor/ (CI/CD offline, auditoria de código)
go build -mod=vendor ./...
go test  -mod=vendor ./...
```

```bash
meu-projeto/
├── go.mod
├── go.sum
├── vendor/
│   ├── modules.txt      ← inventário de módulos
│   └── github.com/
│       └── gin-gonic/
│           └── gin/     ← código fonte da dependência
└── main.go
```

---

## 10. Ciclo de Vida de uma Dependência

```
1. go get github.com/pkg@v1.0.0
   → baixa do GOPROXY
   → verifica hash no GOSUM database
   → salva em $GOMODCACHE/github.com/pkg@v1.0.0/
   → atualiza go.mod e go.sum

2. go mod tidy
   → analisa imports do código
   → adiciona deps faltantes
   → remove deps não usadas
   → atualiza go.sum

3. go build
   → lê go.mod
   → verifica go.sum
   → compila usando cache em $GOMODCACHE

Cache local:
$GOPATH/pkg/mod/           ← módulos por versão (imutáveis)
$GOPATH/pkg/mod/cache/     ← zips e metadados

# Ver localização
go env GOMODCACHE
```

---

## 11. Resumo dos Comandos

| Comando | O que faz |
| --- | --- |
| `go mod init m` | Cria o módulo |
| `go get pkg` | Adiciona/atualiza dependência |
| `go get pkg@v` | Versão específica |
| `go get -u pkg` | Atualiza para latest |
| `go mod tidy` | Sincroniza go.mod e go.sum |
| `go mod download` | Baixa para cache |
| `go mod verify` | Verifica integridade |
| `go mod vendor` | Cria vendor/ |
| `go mod graph` | Árvore de deps |
| `go mod why pkg` | Por que esta dep existe |
| `go list -m all` | Todos os módulos |
| `go work init` | Cria workspace |
| `go work use ./m` | Adiciona módulo ao workspace |

---

## 12. Como Funciona Internamente

### O Pipeline de Build — Do Source ao Binário

```
Analogia com o pipeline C (gcc):

  C:                          Go:
  preprocessor                (não existe — sem macros)
      ↓                           ↓
  compiler (cc1)              go tool compile
  source → .s (assembly)      .go → .a (archive com código objeto)
      ↓                           ↓
  assembler (as)              go tool asm
  .s → .o (object file)       .s (asm inline) → .o
      ↓                           ↓
  linker (ld)                 go tool link
  .o + libs → ELF binary      .a + runtime → ELF/PE/Mach-O binary

  go build ./... faz tudo isso automaticamente, com cache.
  O cache fica em $GOCACHE (normalmente ~/.cache/go/build/).
  Pacotes não alterados não são recompilados — igual ao make.
```

### Download de Dependências — Analogia com Gerenciador de Pacotes

```
apt install nginx              go get github.com/gin-gonic/gin@v1.9.1
     ↓                                    ↓
   HTTP para                        HTTP para
   deb.debian.org                   proxy.golang.org
     ↓                                    ↓
   verifica assinatura GPG          verifica hash no sum.golang.org
     ↓                                    ↓
   extrai para /usr/                extrai para $GOMODCACHE
   (global, root)                   (~/.cache/go/pkg/mod/, por usuário)
     ↓                                    ↓
   /etc/apt/sources.list            go.mod + go.sum atualizados
```

### `go.sum` e SHA-256 — Verificação Criptográfica

```
go.sum:
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQ...= 
                                │  └─ SHA-256 do zip do módulo, em base64
                                └─ "h1:" = algoritmo de hash versão 1

Como é calculado:
  1. go mod download baixa o .zip do módulo
  2. Calcula SHA-256 de cada arquivo no zip (árvore de hash)
  3. SHA-256 final = hash da árvore (Hash1 no código do Go)
  4. Compara com sum.golang.org (checksum database público)
  5. Grava no go.sum

Propriedade garantida: se alguém alterar UMA linha do código da dependência,
o hash SHA-256 muda completamente (efeito avalanche).
go mod verify detecta isso relendo os arquivos do cache.
```

### Semver e Versionamento de Shared Libraries

```
Linux shared libraries:           Go modules:

libssl.so.1.1.1                   github.com/user/pkg v1.9.1
       │ │ │                                           │ │ │
       │ │ └── patch (bug fixes)                       │ │ └── patch
       │ └──── minor (compatible)                      │ └──── minor
       └────── major = soname                          └────── major
               (breaking change →                             (breaking change →
                novo arquivo .so)                              novo import path /v2)

dlopen("libssl.so.1")    →  import "pkg/v1"
dlopen("libssl.so.3")    →  import "pkg/v3"

Ambos podem coexistir no mesmo processo — isolamento de versão garantido.
```

---

## 13. Conexão com Sistemas Operacionais

**`go mod download` busca do module proxy e verifica hash → [[Bits e Bytes]]**
`go.sum` registra o hash SHA-256 (codificado em base64 com prefixo `h1:`) de cada dependência. SHA-256 produz 256 bits de saída; qualquer alteração de 1 bit no fonte muda completamente o hash (efeito avalanche). `go mod verify` recalcula e compara — a mesma técnica usada por gerenciadores de pacotes (`apt`, `yum`) para verificar integridade de downloads.

**Isolamento de versões de módulos espelha versionamento de shared libraries no Linux → [[Processos]]**
No Linux, `libssl.so.1` e `libssl.so.3` podem coexistir porque têm nomes de arquivo diferentes (soname). Go usa a mesma ideia: `pkg/v1` e `pkg/v3` são import paths distintos — dois módulos com breaking changes podem coexistir no mesmo binário sem conflito.

**`go build` executa o mesmo pipeline de compilação do C → [[Processadores]]**
O pipeline `go tool compile → go tool asm → go tool link` é exatamente `cc → as → ld`: compila para código objeto, monta assembly inline, linka com o runtime e bibliotecas. O resultado é um binário ELF (Linux) ou Mach-O (macOS) com o runtime Go embutido.

**Espaços de nomes de módulos evitam conflitos — análogo a namespaces de processos → [[Processos]]**
Cada módulo tem seu próprio espaço de versões: `github.com/user/pkg@v1.9.1` e `github.com/user/pkg@v2.0.0` são entidades separadas no cache. Isso é análogo aos namespaces do Linux (PID namespace, mount namespace) que isolam recursos entre processos — cada módulo vive em seu próprio "namespace de versão".