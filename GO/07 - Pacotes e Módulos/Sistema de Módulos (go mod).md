---
tags:
  - go
  - go/pacotes
aliases:
  - go mod
  - Go Modules
  - go.mod
  - go.sum
  - Minimal Version Selection
  - Dependências em Go
ordem: 31
modulo: "07 - Pacotes e Módulos"
---
# Sistema de Módulos (go mod)

> **Nota 31 da trilha de Go.** Um **módulo** é um conjunto de pacotes versionados juntos, identificado por um caminho (o *module path*) e descrito por um arquivo **`go.mod`**. É assim que Go gerencia **dependências**: quais bibliotecas externas o projeto usa, em quais versões, e como garantir que todo mundo compile exatamente o mesmo código. Nesta nota veremos o `go.mod` linha por linha, o `go.sum` e a verificação criptográfica, **versionamento semântico** e a regra das versões major (`/v2`), o algoritmo **Minimal Version Selection**, o proxy de módulos, os comandos do dia a dia, workspaces, vendoring e as novidades recentes (diretivas `tool`, `ignore`, `toolchain`).
>
> **Pré-requisitos:** [[Criando Pacotes]].

---

## 🧭 De onde viemos

Em [[Ambiente e Primeiro Programa]] criamos um módulo com `go mod init` e vimos que o GOPATH antigo virou só um cache. Em [[Criando Pacotes]] vimos que o caminho de import de um pacote é o **module path** + a pasta. Agora vamos entender o módulo em si e o gerenciamento de dependências.

> 💡 **Dependência:** código de terceiros que o seu projeto usa. **Gerenciador de dependências:** ferramenta que baixa, versiona e verifica essas dependências. Em PHP é o Composer, em Ruby o Bundler, em JavaScript o npm. Em Go, é o próprio comando `go`, sem ferramenta extra.

---

## 📜 1. Um pouco de história

| Época | Modelo | Problema |
| --- | --- | --- |
| 2009–2018 | **GOPATH**: todo código em `$GOPATH/src/<url>`; `go get` baixava a versão mais recente do repositório | Sem versões: dois projetos não podiam usar versões diferentes da mesma biblioteca; builds não reproduzíveis |
| 2015–2018 | Ferramentas da comunidade (`godep`, `glide`, `dep`) + pasta `vendor/` | Fragmentação |
| Go 1.11 (2018) | **Go Modules** (experimental) | — |
| Go 1.16 (2021) | Modules como padrão | — |
| Hoje | Módulos em todo lugar; GOPATH só como cache | — |

---

## 📄 2. O `go.mod` linha por linha

```go
module github.com/brugge-matheus/loja           // 1. module path

go 1.26.0                                       // 2. versão mínima da linguagem

toolchain go1.27.2                              // 3. toolchain sugerida (opcional)

require (                                       // 4. dependências diretas
	github.com/google/uuid v1.6.0
	github.com/jackc/pgx/v5 v5.7.1
)

require (                                       // 5. dependências indiretas
	github.com/jackc/pgpassfile v1.0.0 // indirect
	golang.org/x/crypto v0.31.0 // indirect
)

tool golang.org/x/tools/cmd/stringer            // 6. ferramentas do projeto (Go 1.24+)

replace github.com/meu/fork => ../fork-local    // 7. substituição (desenvolvimento local)

exclude github.com/lib/ruim v1.2.3              // 8. nunca usar esta versão

retract v1.0.1                                  // 9. "versão minha que não deve ser usada"

ignore ./node_modules                           // 10. pastas a ignorar (Go 1.25+)

godebug default=go1.21                          // 11. comportamentos de compatibilidade
```

| Diretiva | Significado |
| --- | --- |
| `module` | O nome do módulo; prefixo dos caminhos de import. Normalmente a URL do repositório |
| `go` | A **versão mínima** do Go exigida, e as regras de linguagem em vigor (ex.: a variável de laço por iteração só vale com `go 1.22`+) |
| `toolchain` | Versão de toolchain preferida para trabalhar no módulo (o `go` baixa se `GOTOOLCHAIN=auto`) |
| `require` | Dependência e sua versão **mínima** (veremos por que "mínima" na seção 5) |
| `// indirect` | Dependência que o seu código não importa diretamente (é dependência de uma dependência) |
| `tool` | Ferramentas executáveis usadas no projeto, rodadas com `go tool nome` (substitui o antigo truque do `tools.go`) |
| `replace` | Troca um módulo por outro caminho ou versão (fork, desenvolvimento local) |
| `exclude` | Proíbe uma versão específica |
| `retract` | O autor do módulo declara que versões suas estão quebradas |
| `ignore` | Pastas que o `go` deve ignorar ao procurar pacotes (útil em repositórios com `node_modules`) |
| `godebug` | Ajusta comportamentos que mudaram entre versões (veja `GODEBUG`) |

> 💡 Desde o **Go 1.27**, `go mod tidy` reorganiza o arquivo para ter no máximo **dois** blocos `require`: um de diretas e um de indiretas (preservando comentários).

> 💡 Desde o **Go 1.26**, `go mod init` escreve na linha `go` a versão **anterior** à toolchain instalada (com Go 1.27, escreve `go 1.26.0`), para que o módulo funcione para quem ainda está uma versão atrás. Saída real com Go 1.24 instalado (antes dessa mudança): `go 1.24.7`.

---

## 🔐 3. O `go.sum`: verificação criptográfica

O `go.sum` guarda o **hash** (SHA-256, codificado em base64) do conteúdo de cada versão de dependência:

```
github.com/google/uuid v1.6.0 h1:NIvaJDMOsjHA8n1jAhLSgzrAzy1Hgr+hNrb57e+94F0=
github.com/google/uuid v1.6.0/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
```

- A primeira linha é o hash do código do módulo; a segunda, só do `go.mod` dele
- Ao baixar uma dependência, o `go` recalcula o hash e **compara**. Se não bater, o build falha: alguém alterou o código daquela versão (ataque, repositório reescrito)
- **Commite o `go.sum`** junto com o `go.mod`

> 💡 **Hash criptográfico:** função que transforma qualquer conteúdo num valor de tamanho fixo, de forma que qualquer mudança mínima no conteúdo muda completamente o hash, e é inviável fabricar outro conteúdo com o mesmo hash. É a mesma ideia usada para verificar imagens Docker por *digest* ([[Imagens]]) e certificados TLS ([[TLS e SSL]]).

### 3.1 O banco de checksums público

Na primeira vez que **qualquer pessoa no mundo** baixa uma versão de um módulo público, o hash é registrado em **sum.golang.org**, um log público e à prova de adulteração (uma *transparency log*, como o Certificate Transparency do TLS). O `go` confere o seu `go.sum` com esse log. Assim, se o autor de uma biblioteca reescrever a tag `v1.2.3` com código diferente, todo mundo percebe.

---

## 🔢 4. Versionamento semântico

Versões de módulos seguem o **SemVer**: `vMAJOR.MINOR.PATCH`

```
v1.4.2
 │ │ └── PATCH: correção de bug, compatível
 │ └──── MINOR: funcionalidade nova, compatível
 └────── MAJOR: mudança INCOMPATÍVEL (quebra quem usa)
```

| Versão | Significado |
| --- | --- |
| `v0.x.y` | Instável: pode quebrar a qualquer momento |
| `v1.0.0` | Primeira versão estável: a API é uma promessa |
| `v1.5.0-beta.1` | Pré-lançamento |
| `v0.0.0-20261015120000-abcdef123456` | **Pseudo-versão**: um commit sem tag (data + hash do commit) |

As versões são **tags do git** no repositório (`git tag v1.4.2`).

### 4.1 A regra de importação das versões major

Esta é uma decisão muito particular de Go:

> **Se uma versão major nova (v2+) quebra a compatibilidade, ela é um MÓDULO DIFERENTE, com um caminho diferente.**

```go
import "github.com/jackc/pgx/v5"     // o "/v5" faz parte do caminho!

// go.mod do próprio pgx:
module github.com/jackc/pgx/v5
```

Consequências:

- `v1` e `v2` de uma biblioteca podem **coexistir** no mesmo programa (são pacotes diferentes), o que permite migração gradual
- Nunca acontece de um `go get -u` trazer uma versão incompatível sem você mudar os imports
- `v0` e `v1` não têm sufixo

> 💡 Esse princípio se chama **import compatibility rule**: *"se um pacote antigo e um novo têm o mesmo caminho de import, o novo precisa ser compatível com o antigo"*.

---

## 🧮 5. Minimal Version Selection (MVS)

Como o `go` decide **qual versão** de cada dependência usar quando várias partes do projeto pedem versões diferentes? A maioria dos gerenciadores (npm, Bundler, pip) escolhe a versão **mais nova** permitida e precisa de um arquivo de *lock* para congelar. Go usa um algoritmo diferente, criado por Russ Cox:

> **Minimal Version Selection:** cada `require` declara a versão **mínima** aceitável. O `go` escolhe, para cada módulo, **a maior entre as mínimas exigidas** por todo o grafo de dependências. Nunca uma versão mais nova do que alguém pediu.

```
                   seu módulo
                  /          \
     require A v1.2         require B v1.0
            │                      │
     A v1.2 requer C v1.3    B v1.0 requer C v1.5
                  \                /
                   C: exigências mínimas = { v1.3, v1.5 }

   MVS escolhe C v1.5  (a MAIOR das mínimas)
   mesmo que já exista C v1.9 publicada → v1.9 NÃO é usada
```

Por que isso é bom:

| Propriedade | Efeito |
| --- | --- |
| **Reproduzível sem lock file** | O resultado depende só dos `go.mod`, não de "o que é o mais novo hoje". O `go.mod` + `go.sum` já são o "lock" |
| **Previsível** | Uma versão nova publicada por terceiros nunca entra no seu build sozinha |
| **Simples e rápido** | O algoritmo é linear no grafo; não precisa de um resolvedor de restrições (problema NP-completo nos outros modelos) |
| **Atualizar é explícito** | Só sobe de versão quando alguém roda `go get` |

```bash
go list -m all          # a lista FINAL de versões escolhidas pelo MVS
go mod graph            # o grafo completo de "quem requer o quê"
go mod why -m golang.org/x/crypto   # por que esta dependência está aqui?
```

---

## 🌐 6. De onde vêm os módulos: o proxy

```bash
$ go env GOPROXY
https://proxy.golang.org,direct
```

Por padrão, o `go` não baixa direto do GitHub: pede ao **proxy de módulos** do Google, que guarda uma cópia imutável de cada versão. Vantagens: rapidez, disponibilidade (o módulo continua disponível mesmo se o repositório for apagado) e imutabilidade.

| Variável | Para quê |
| --- | --- |
| `GOPROXY` | Lista de proxies; `direct` = buscar no repositório (git) |
| `GOSUMDB` | Banco de checksums (padrão `sum.golang.org`) |
| `GOPRIVATE` | Padrões de módulos **privados**: não passam pelo proxy nem pelo banco de checksums público |
| `GONOSUMDB`, `GONOPROXY` | Controle mais fino das mesmas coisas |
| `GOFLAGS=-mod=mod` | Flags padrão do comando `go` |

```bash
go env -w GOPRIVATE=github.com/minha-empresa/*
# + configurar o git para autenticar (token ou SSH) nos repositórios privados
```

Os módulos baixados ficam em `$GOPATH/pkg/mod` (`go env GOMODCACHE`), somente-leitura, compartilhados entre todos os seus projetos.

---

## ⌨️ 7. Comandos do dia a dia

```bash
# Começar
go mod init github.com/usuario/projeto

# Adicionar ou atualizar dependências
go get github.com/google/uuid              # versão mais recente (respeitando a major)
go get github.com/google/uuid@v1.5.0       # versão específica
go get github.com/google/uuid@latest
go get github.com/google/uuid@abc1234      # um commit (vira pseudo-versão)
go get -u ./...                            # sobe minor/patch de todas as dependências
go get -u=patch ./...                      # só patches
go get github.com/google/uuid@none         # remover

# Arrumar (o mais usado!)
go mod tidy          # adiciona o que falta e remove o que não é mais importado

# Inspecionar
go list -m all                             # versões finais
go list -m -u all                          # mostra atualizações disponíveis
go mod why -m <módulo>                     # por que está aqui
go mod graph                               # grafo
go mod verify                              # confere o cache com o go.sum

# Ferramentas do projeto (Go 1.24+)
go get -tool golang.org/x/tools/cmd/stringer   # adiciona diretiva tool
go tool stringer -type=Nivel                   # roda a ferramenta na versão do go.mod

# Ferramentas globais (instala em $GOPATH/bin)
go install golang.org/x/vuln/cmd/govulncheck@latest
```

> ✅ Rode `go mod tidy` antes de cada commit que mexe em imports, e faça o CI verificar que o `go.mod`/`go.sum` estão arrumados (`go mod tidy -diff`, disponível desde o Go 1.23).

---

## 🧳 8. Ciclo de vida de uma dependência

```
1. Você escreve:  import "github.com/google/uuid"
2. go mod tidy  (ou go get)
     ├─ consulta o proxy: qual a versão mais recente? → v1.6.0
     ├─ baixa o .zip da versão → $GOPATH/pkg/mod/github.com/google/uuid@v1.6.0
     ├─ calcula o hash e confere com sum.golang.org
     ├─ adiciona "require github.com/google/uuid v1.6.0" no go.mod
     └─ adiciona as duas linhas de hash no go.sum
3. go build
     ├─ MVS calcula as versões finais de todo o grafo
     ├─ confere os hashes com o go.sum
     └─ compila (com cache: só recompila o que mudou)
4. Atualizar depois:  go get github.com/google/uuid@latest  → go.mod e go.sum mudam
5. Remover: apagar o import + go mod tidy
```

---

## 🧩 9. Workspaces: vários módulos locais (Go 1.18+)

Quando você desenvolve **dois módulos ao mesmo tempo** (uma biblioteca e o app que a usa), o `replace` no `go.mod` funciona, mas é fácil commitá-lo por engano. Um **workspace** resolve isso fora do `go.mod`:

```bash
mkdir meu-espaco && cd meu-espaco
go work init ./app ./biblioteca
```

```go
// go.work (normalmente NÃO é commitado)
go 1.27

use (
	./app
	./biblioteca
)
```

Com o `go.work` presente, o `app` usa a cópia **local** da `biblioteca`, sem tocar no `go.mod` de nenhum dos dois.

---

## 📦 10. Vendoring

```bash
go mod vendor     # copia todas as dependências para a pasta vendor/
go build          # com vendor/ presente, o go usa as cópias locais (-mod=vendor)
```

Útil quando o build precisa funcionar **sem rede** (ambientes isolados), ou quando a empresa exige revisar todo código de terceiros no repositório. Para a maioria dos projetos, o proxy + `go.sum` já garantem reprodutibilidade, e `vendor/` é dispensável.

---

## 🛡️ 11. Segurança da cadeia de suprimentos

| Ameaça | Defesa em Go |
| --- | --- |
| Alguém altera o código de uma versão já publicada | `go.sum` + `sum.golang.org` |
| Repositório apagado ("left-pad") | Proxy guarda cópias imutáveis |
| Atualização maliciosa entra sozinha | MVS: nada sobe sem `go get` explícito |
| Dependência com vulnerabilidade conhecida | `govulncheck` (analisa se o código vulnerável é de fato **chamado**) |
| Scripts de instalação maliciosos | Go **não executa código** ao baixar dependências (diferente de `npm install` com `postinstall`) |

```bash
govulncheck ./...    # relata só vulnerabilidades em funções que o seu código alcança
```

---

## 🔬 12. Por dentro: do `go.mod` ao binário

```
go build
  │
  ├─ lê go.mod (e go.work, se houver)
  ├─ MVS → lista final de módulos@versões
  ├─ localiza cada pacote importado: em qual módulo ele mora?
  ├─ garante que os módulos estão no cache ($GOPATH/pkg/mod), baixando do proxy se preciso
  ├─ verifica os hashes (go.sum)
  ├─ compila cada pacote (ou reaproveita do cache de build: $GOCACHE, indexado
  │  por hash do código-fonte + flags + versão do compilador)
  └─ linka tudo num binário, gravando as versões usadas na seção .go.buildinfo
```

Por isso dá para descobrir de onde veio cada pedaço de um binário Go (exemplo de saída):

```bash
$ go version -m ./meuapp
./meuapp: go1.27.2
	path	github.com/brugge-matheus/loja
	mod	github.com/brugge-matheus/loja	(devel)
	dep	github.com/google/uuid	v1.6.0	h1:NIvaJDMOsjHA8n1jAhLSgzrAzy1Hgr+hNrb57e+94F0=
	build	-buildmode=exe
	build	GOOS=linux
```

> 💡 Compare com o mundo C/Linux que você estudou em [[Sistemas Monolíticos]]: lá, bibliotecas compartilhadas (`.so`) são resolvidas em **tempo de execução** pelo *dynamic linker*, e a versão usada depende do que está instalado no sistema ("dependency hell", versionamento por `soname` como `libssl.so.3`). Em Go, as versões são resolvidas em **compilação** e tudo vai para dentro de um binário estático.

---

## ⚠️ 13. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Não commitar o `go.sum` | Builds falham ou não são verificados | Sempre commite `go.mod` e `go.sum` |
| `replace` local commitado | Build quebra em outras máquinas | Use `go.work` para desenvolvimento local |
| Esquecer o `/v2` no caminho ao lançar v2 | Usuários não conseguem atualizar | `module exemplo.com/lib/v2` + tag `v2.0.0` |
| Editar `go.mod` à mão e não rodar `tidy` | `go.sum` desatualizado, erros "missing go.sum entry" | `go mod tidy` |
| Módulo privado passando pelo proxy público | Erro 404/410 ou vazamento do nome | `GOPRIVATE` |
| Achar que `go get -u` é seguro sempre | Atualizações minor podem ter bugs | Rode os testes; atualize com critério |
| Linha `go` muito alta numa biblioteca | Quem usa versões anteriores não consegue compilar | Use a menor versão que você realmente precisa |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Módulo | Conjunto de pacotes versionados juntos; definido pelo `go.mod` |
| `go.mod` | `module`, `go`, `toolchain`, `require`, `replace`, `exclude`, `retract`, `tool`, `ignore`, `godebug` |
| `go.sum` | Hashes SHA-256 das dependências; verificado sempre |
| `sum.golang.org` | Log público à prova de adulteração |
| SemVer | `vMAJOR.MINOR.PATCH`; v2+ muda o caminho (`/v2`) |
| MVS | A maior das versões **mínimas** exigidas; reproduzível sem lock file |
| Proxy | `proxy.golang.org`; `GOPRIVATE` para privados |
| Comandos | `go mod init`, `go get`, `go mod tidy`, `go list -m all`, `go mod why` |
| Workspace | `go.work` para desenvolver vários módulos juntos |
| Vendor | `go mod vendor` para builds sem rede |
| Segurança | Hashes, proxy imutável, MVS, `govulncheck`, sem scripts de instalação |

---

## 🔗 Conexões

**Dentro de Go**
- Pacotes, imports e `internal/` → [[Criando Pacotes]]
- `GOTOOLCHAIN` e a primeira configuração → [[Ambiente e Primeiro Programa]]
- `.go.buildinfo` e o binário → [[Como Go Funciona]]
- Ferramentas (`go tool`, `go generate`) → [[Build, Flags e Cross-compilation]]
- `govulncheck` e qualidade → [[Convenções da linguagem]]

**Sistemas Operacionais**
- [[Sistemas Monolíticos]] — bibliotecas compartilhadas e DLLs resolvidas em execução, o modelo que o binário estático de Go evita
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — o linker de C e a ligação com bibliotecas
- [[Proteção]] — o cache de módulos é somente-leitura para evitar alterações acidentais

**Docker**
- [[Dockerfile]] e [[Imagens]] — copiar `go.mod` e `go.sum` antes do código e rodar `go mod download` aproveita o cache de camadas: as dependências só são baixadas de novo quando os arquivos de módulo mudam
- [[Docker Compose]] — workspaces e `replace` em ambientes de desenvolvimento com vários serviços

**Nginx / Redes**
- [[TLS e SSL]] — o banco de checksums usa a mesma ideia de log de transparência dos certificados

---

⬅️ **Anterior:** [[Criando Pacotes]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Pacotes Importantes da Stdlib]] ➡️
