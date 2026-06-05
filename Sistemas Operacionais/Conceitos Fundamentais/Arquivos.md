---
tags:
  - sistemas-operacionais
  - so/conceitos
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.5.3"
---
# Arquivos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.5.3

---

# 📁 O que é um Arquivo?

Outro conceito fundamental que conta com o suporte de praticamente todos os sistemas operacionais é o **sistema de arquivos**. Uma função importante do SO é esconder as peculiaridades dos discos e outros dispositivos de I/O e apresentar ao programador um modelo **agradável e claro de arquivos** que sejam independentes dos dispositivos.

Um arquivo é a abstração que o SO oferece para armazenamento persistente — em vez de pensar em blocos, setores e trilhas de disco, o programador trabalha com arquivos que têm nome, conteúdo e podem ser criados, lidos, escritos e removidos via chamadas de sistema.

---

# 📂 Diretórios — Organizando os Arquivos

Para fornecer um lugar para manter os arquivos, a maioria dos SOs tem o conceito de **diretório** — às vezes chamado de **pasta** ou **mapa** — como uma forma de agrupar arquivos.

Um estudante, por exemplo, pode ter um diretório para cada disciplina que estiver cursando (com os programas necessários para aquela disciplina), outro para o correio eletrônico e ainda um para sua página na web.

Entradas de diretório podem ser:

- **Arquivos** — dados propriamente ditos
- **Outros diretórios** — formando uma hierarquia recursiva

Isso cria o **sistema de arquivos hierárquico** — uma estrutura em árvore que surgiu com o Multics e é usada em todos os SOs modernos.

---

# 🌳 Hierarquia de Diretórios

> 📌 **Figura 1.14 — Sistema de arquivos para um departamento universitário**
> 

```
                   Diretório-raiz (/)
                         │
         ┌───────────────┴────────────────┐
         │                                │
    Estudantes                       Professores
   ┌──┬──┬──┐                    ┌──────┬──────┬──────┐
Robbert Matty Leo            Prof.Brown Prof.Green Prof.White
                                  │
                        ┌─────────┼──────────┐
                      Cursos   Artigos    Bolsas   Comitês
                     ┌──┴──┐                         │
                  CS101  CS105                    SOSP  COST-11
                                                   │
                                               Arquivos
```

Chamadas de sistema são necessárias para:

- Criar e remover diretórios
- Colocar um arquivo existente em um diretório
- Remover um arquivo de um diretório

---

# 🗺️ Nomes de Caminho (Path Names)

Todo arquivo dentro de uma hierarquia de diretório pode ser especificado pelo seu **nome de caminho** (*path name*) a partir do topo da hierarquia — o **diretório-raiz** (*root directory*).

**Caminho absoluto** — começa na raiz e lista todos os diretórios até o arquivo:

```
/Professores/Prof.Brown/Cursos/CS101
```

A primeira barra indica que o caminho começa na raiz. No Windows usa-se `\` ao invés de `/` por razões históricas:

```
\Professores\Prof.Brown\Cursos\CS101
```

**Caminho relativo** — começa no **diretório de trabalho** atual do processo. Se o diretório de trabalho for `/Professores/Prof.Brown`, o caminho relativo para o mesmo arquivo seria:

```
Cursos/CS101
```

> 💡 **Diretório de trabalho:** a todo instante, cada processo tem um diretório de trabalho atual, no qual são procurados nomes de caminhos que não começam com uma barra. Processos podem mudar seu diretório de trabalho ao emitir uma chamada de sistema especificando o novo diretório.
> 

---

# 📖 Abrindo e Fechando Arquivos

Antes que um arquivo possa ser lido ou escrito, ele precisa ser **aberto** — momento em que as permissões são verificadas. Se o acesso for permitido, o sistema retorna um pequeno valor inteiro chamado **descritor de arquivo** (*file descriptor*), para usá-lo em operações subsequentes. Se o acesso for proibido, um código de erro é retornado.

```
Fluxo típico de uso de arquivo:

open("arquivo.txt")  → verifica permissões → retorna descritor fd=3
read(fd=3, buffer)   → lê dados do arquivo para o buffer
write(fd=3, dados)   → escreve dados no arquivo
close(fd=3)          → libera o descritor
```

---

# 💾 Montagem de Sistemas de Arquivos

A maioria dos computadores desktop e notebook tem uma ou mais portas USB onde pendrives e SSDs externos podem ser conectados. Para lidar com essa mídia removível de forma elegante, o UNIX permite que o sistema de arquivos nesses dispositivos de armazenamento separados seja **agregado à árvore principal**.

> 📌 **Figura 1.15 — Montagem de sistema de arquivos**
> 

```
(a) Antes da montagem:          (b) Depois da montagem em /b:

Raiz          Unidade USB        Raiz unificada
 /               /                    /
┌─┴─┐         ┌──┴──┐            ┌────┴────┐
a   b          x     y           a          b
              (inacessível)          ┌──────┴──────┐
                                     x             y
                                  (agora acessíveis!)
```

A chamada de sistema `mount` permite que o sistema de arquivos na unidade USB seja agregado ao sistema de arquivos-raiz sempre que seja pedido pelo programa. Após a montagem, os arquivos da unidade USB se tornam acessíveis como `/b/x` e `/b/y`.

> ⚠️ Se o diretório `b` contivesse quaisquer arquivos antes da montagem, eles não seriam acessíveis enquanto a unidade USB estivesse sendo montada — `/b` passaria a se referir ao diretório-raiz da unidade USB.
> 

> 💡 Se um sistema contém múltiplos discos rígidos, todos eles também podem ser montados em uma única árvore — formando um sistema de arquivos unificado independente do número de dispositivos físicos.
> 

---

# 🔧 Arquivos Especiais e o Diretório /dev

Um conceito importante em UNIX é o de **arquivo especial** — permite que dispositivos de I/O se pareçam com arquivos. Dessa forma, eles podem ser lidos e escritos com as mesmas chamadas de sistema usadas para arquivos comuns.

## O Problema que os Arquivos Especiais Resolvem

Sem arquivos especiais, você precisaria de chamadas de sistema completamente diferentes para cada tipo de dispositivo — uma para a impressora, outra para o disco, outra para o teclado. Com arquivos especiais, o UNIX tem uma ideia elegante:

> **Trate dispositivos de hardware como se fossem arquivos — use as mesmas chamadas `open()`, `read()`, `write()` para tudo.**
> 

## O Diretório /dev

`/dev` significa *devices* (dispositivos). Não é uma pasta com arquivos reais no disco — é onde o kernel **expõe os dispositivos de hardware como se fossem arquivos**:

```
/dev/
├── sda          → disco rígido principal       (arquivo especial de bloco)
├── sda1         → primeira partição do disco   (arquivo especial de bloco)
├── sdb          → segundo disco / pendrive     (arquivo especial de bloco)
├── tty0         → terminal 0                  (arquivo especial de caracteres)
├── ttyS0        → porta serial                (arquivo especial de caracteres)
├── lp0          → impressora                  (arquivo especial de caracteres)
├── null         → buraco negro (descarta tudo) (arquivo especial de caracteres)
├── zero         → fonte infinita de zeros     (arquivo especial de caracteres)
└── random       → gerador de números aleatórios (arquivo especial de caracteres)
```

## Arquivo Especial de Bloco

Usado para dispositivos que armazenam dados em **blocos endereçáveis aleatoriamente** — discos e SSDs. Você pode pedir especificamente o bloco 4, o bloco 100, em qualquer ordem — sem precisar ler do início. Útil para ferramentas de backup, recuperação de dados e formatação que precisam acessar o disco em nível físico, sem passar pelo sistema de arquivos.

```bash
# Acessar diretamente o bloco 0 do disco sem passar pelo sistema de arquivos
dd if=/dev/sda bs=512 count=1
```

## Arquivo Especial de Caracteres

Usado para dispositivos que trabalham com **fluxo contínuo de caracteres**, um de cada vez — teclado, mouse, impressora, porta serial. Não faz sentido pedir "o caractere número 1000" — você só recebe o próximo caractere disponível.

```bash
# Enviar texto para a impressora como se fosse um arquivo
echo "Olá, impressora!" > /dev/lp0

# Usar o gerador de números aleatórios do kernel
head -c 16 /dev/random | base64
```

## Por que isso é Poderoso — "Tudo é Arquivo"

A beleza do conceito é a **uniformidade** — o mesmo código que lê um arquivo de texto consegue ler de um dispositivo:

```c
// Ler de um arquivo comum:
int fd = open("/home/user/dados.txt", O_RDONLY);
read(fd, buffer, 100);

// Ler do teclado (dispositivo de caracteres):
int fd = open("/dev/tty0", O_RDONLY);
read(fd, buffer, 100);

// Ler diretamente do disco (dispositivo de bloco):
int fd = open("/dev/sda", O_RDONLY);
read(fd, buffer, 512);

// O código de leitura é IDÊNTICO nos três casos!
```

Ferramentas como `cat`, `cp`, `grep` funcionam em arquivos e dispositivos igualmente — porque para o programa, tudo parece um arquivo. Esse é o princípio **"tudo é arquivo"** (*everything is a file*) que o UNIX introduziu e que influenciou todos os SOs modernos.

---

# 🔗 Pipes

O último aspecto relacionado a arquivos é o **pipe** — uma espécie de pseudoarquivo que pode ser usado para conectar dois processos.

## O Problema que o Pipe Resolve

Sem pipe, para encadear dois programas seria necessário:

1. Executar o primeiro e salvar o resultado em um arquivo temporário
2. Executar o segundo lendo aquele arquivo
3. Deletar o arquivo temporário

Com pipe, a saída de um processo vira diretamente a entrada do próximo — sem arquivo temporário, sem código extra.

## Como Funciona

Um pipe é um **canal unidirecional de comunicação entre dois processos**. Do ponto de vista de cada processo, ele parece exatamente um arquivo — um escreve com `write()`, o outro lê com `read()`:

> 📌 **Figura 1.16 — Dois processos conectados por um pipe**
> 

```
┌─────────────┐                ┌─────────────┐
│  Processo A │ ──── pipe ────▶│  Processo B │
│  (escreve)  │                │   (lê)      │
└─────────────┘                └─────────────┘

         [buffer do SO — tipicamente 64 KB]

Se o buffer encher   → Processo A é bloqueado até B ler
Se o buffer esvaziar → Processo B é bloqueado até A escrever
```

Os processos se sincronizam automaticamente — sem código extra de coordenação.

## Exemplo Concreto no Terminal

O símbolo `|` no shell é literalmente um pipe:

```bash
# Sem pipe — precisaria de arquivo temporário:
ls /etc > /tmp/lista.txt
grep "ssh" /tmp/lista.txt
rm /tmp/lista.txt

# Com pipe — direto:
ls /etc | grep "ssh"

# Três processos em cadeia:
cat log.txt | grep "erro" | wc -l
# cat  → lê o arquivo e escreve no pipe 1
# grep → lê do pipe 1, filtra, escreve no pipe 2
# wc   → lê do pipe 2 e conta as linhas
```

## Pipe Anônimo vs. Pipe Nomeado (FIFO)

**Pipe anônimo** — o mais comum. Existe apenas na memória enquanto os processos estão rodando. Só funciona entre processos relacionados (pai e filho):

```c
int fd[2];
pipe(fd);  // fd[0] para ler, fd[1] para escrever
```

**Pipe nomeado (FIFO)** — fica persistido no sistema de arquivos como um arquivo especial. Qualquer processo pode se conectar, mesmo sem relação de parentesco:

```bash
mkfifo /tmp/meu_pipe           # cria o pipe nomeado
echo "dados" > /tmp/meu_pipe & # processo A escreve
cat /tmp/meu_pipe              # processo B lê
```

|  | Pipe Anônimo | Pipe Nomeado (FIFO) |
| --- | --- | --- |
| **Onde existe** | Só na memória | No sistema de arquivos |
| **Quem pode usar** | Processos relacionados | Qualquer processo |
| **Persiste** | Não | Sim (até ser deletado) |
| **Criação** | `pipe()` | `mkfifo()` |

---

# 🌲 Hierarquia de Processos vs. Hierarquia de Arquivos

Ambas as hierarquias são organizadas em forma de árvore, mas com diferenças importantes:

|  | Hierarquia de Processos | Hierarquia de Arquivos |
| --- | --- | --- |
| **Profundidade típica** | Rasa (mais de 5 níveis é incomum) | Profunda (6, 7 ou mais níveis) |
| **Tempo de vida** | Curta — em geral minutos, no máximo | Pode existir por anos |
| **Controle** | Apenas processo-pai pode controlar filho | Grupo mais amplo pode ler arquivos |
| **Proteção** | Mais rígida entre processos | Mecanismos para compartilhamento |

---

# ✅ Resumo do Conceito

- **Arquivo** é a abstração do SO para armazenamento persistente — esconde a complexidade de discos e dispositivos
- **Diretórios** organizam os arquivos em hierarquias; entradas podem ser arquivos ou outros diretórios
- **Caminhos absolutos** começam na raiz (`/`); **caminhos relativos** começam no diretório de trabalho do processo
- Antes de usar um arquivo é preciso **abrí-lo** — o SO verifica permissões e retorna um **descritor de arquivo**
- **Montagem** permite agregar sistemas de arquivos de dispositivos removíveis à árvore principal, formando uma hierarquia unificada
- **Arquivos especiais** (de bloco e de caracteres) permitem que dispositivos de I/O sejam tratados como arquivos — mesmas chamadas de sistema
- **Pipes** são pseudoarquivos que conectam dois processos, permitindo comunicação via leitura e escrita como em arquivos comuns