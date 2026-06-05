---
tags:
  - sistemas-operacionais
  - so/syscalls
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.6.3"
---
# Syscalls para Gerenciamento de Diretórios

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.6.3

---

# 📂 O que são Syscalls para Gerenciamento de Diretórios?

Enquanto a seção anterior tratou de chamadas que operam sobre **arquivos individuais**, esta seção examina as chamadas que se relacionam mais aos **diretórios ou ao sistema de arquivos como um todo** — criar e remover diretórios, criar links entre arquivos, montar sistemas de arquivos e muito mais.

---

# 📋 As Principais Syscalls

| Chamada | Descrição |
| --- | --- |
| `s = mkdir(name, mode)` | Cria um novo diretório vazio |
| `s = rmdir(name)` | Remove um diretório vazio |
| `s = link(name1, name2)` | Cria uma nova entrada `name2` apontando para o mesmo arquivo que `name1` |
| `s = unlink(name)` | Remove uma entrada de diretório |
| `s = mount(special, name, flag)` | Monta um sistema de arquivos |
| `s = umount(special)` | Desmonta um sistema de arquivos |

---

# 📁 mkdir() e rmdir() — Criando e Removendo Diretórios

As primeiras duas chamadas são simples e diretas:

```c
s = mkdir("/home/user/projetos", 0755);  // cria o diretório
s = rmdir("/home/user/projetos");        // remove o diretório (deve estar vazio)
```

`rmdir()` só funciona em diretórios **vazios** — se houver qualquer arquivo ou subdiretório dentro, a chamada falha. Para remover um diretório com conteúdo, é preciso remover tudo dentro dele primeiro.

---

# 🔗 link() — Fazendo um Arquivo Aparecer em Dois Lugares

`link()` é uma das chamadas mais interessantes e menos intuitivas. Sua finalidade é permitir que o **mesmo arquivo apareça sob dois ou mais nomes**, muitas vezes em diretórios diferentes.

```c
s = link("/usr/jim/memo", "/usr/ast/note");
```

Após essa chamada, o arquivo `memo` no diretório de `jim` estará aparecendo agora no diretório de `ast` sob o nome `note`. Daí em diante, `/usr/jim/memo` e `/usr/ast/note` **referem-se ao mesmo arquivo**.

## Por que isso é útil?

Um uso típico é permitir que vários membros da mesma equipe de programação compartilhem um arquivo comum, com cada um deles vendo o arquivo como se estivesse no seu próprio diretório, possivelmente sob nomes diferentes.

**Compartilhar um arquivo não é o mesmo que dar a cada membro uma cópia particular.** Ter um arquivo compartilhado significa que as mudanças feitas por qualquer membro da equipe são instantaneamente visíveis para os demais — mas há apenas um arquivo. Quando são feitas cópias em outro local, as mudanças subsequentes feitas em uma cópia não afetam as outras.

## Como link() funciona por baixo — i-nodes e i-numbers

Para entender o `link()`, é preciso entender como o UNIX organiza arquivos internamente:

**Todo arquivo UNIX tem um número único — seu i-number** — que o identifica. Esse i-number é um índice em uma tabela de **i-nodes** (*index nodes*), dizendo quem possui o arquivo, onde seus blocos de disco estão e assim por diante.

```
Tabela de i-nodes:
i-number | proprietário | blocos no disco | tamanho | ...
─────────────────────────────────────────────────────────
   16    | jim          | bloco 42, 43... | 1024 B  | ...
   81    | jim          | bloco 99...     | 512 B   | ...
   40    | jim          | bloco 7...      | 256 B   | ...
```

Um **diretório é apenas um arquivo** contendo um conjunto de pares — `(i-number, nome em ASCII)`. Nas primeiras versões do UNIX, cada entrada de diretório tinha 16 bytes — 2 bytes para o i-number e 14 bytes para o nome. Hoje é mais complexo para suportar nomes longos.

```
Diretório /usr/jim:          Diretório /usr/ast:
i-number | nome              i-number | nome
────────────────             ────────────────
   16    | mail                 ...
   81    | games
   40    | test
```

O que `link()` faz é criar uma **nova entrada de diretório com um nome (possivelmente novo), usando o i-number de um arquivo existente**:

```
Depois de link("/usr/jim/memo", "/usr/ast/note"):

Diretório /usr/jim:          Diretório /usr/ast:
i-number | nome              i-number | nome
────────────────             ────────────────
   16    | mail                 70    | memo   ← mesmo i-number!
   81    | games                40    | test
   40    | test                 59    | f.c.
                                70    | note   ← mesmo i-number do memo!
```

Duas entradas têm o mesmo i-number (70) — referem-se ao mesmo arquivo físico no disco. O i-node do arquivo mantém um **contador de links** — o número de entradas de diretório que apontam para ele.

## O campo link count no i-node

O i-node registra o número de entradas de diretório apontando para o arquivo. Quando `unlink()` remove uma entrada, o contador decrementa. **Quando o contador chega a zero** — não existem mais entradas para o arquivo — o arquivo é removido do disco e seus blocos são retornados ao pool de blocos livres:

```
link("/usr/jim/memo", "/usr/ast/note")
→ i-node do arquivo: link_count = 2

unlink("/usr/jim/memo")
→ i-node do arquivo: link_count = 1  (arquivo ainda existe!)

unlink("/usr/ast/note")
→ i-node do arquivo: link_count = 0  → arquivo removido do disco!
```

> 💡 É por isso que a chamada de remoção de arquivo no UNIX se chama `unlink()` e não `delete()` — você não está apagando o arquivo, está **removendo uma entrada de diretório** que aponta para ele. O arquivo só é realmente removido quando não há mais nenhuma entrada apontando para ele.
> 

---

# 🔌 mount() e umount() — Montando Sistemas de Arquivos

A chamada `mount()` permite que dois sistemas de arquivos separados sejam fundidos em um. Uma situação comum é ter o sistema de arquivos-raiz contendo as versões binárias dos comandos comuns em uma partição, e os arquivos do usuário em outra.

> 📌 **Figura 1.22 — Sistema de arquivos antes e depois da montagem**
> 

```
(a) Antes da montagem:              (b) Depois de mount("/dev/sdb0", "/mnt", 0):

Raiz (disco rígido)                  Raiz (unificada)
         /                                    /
    ┌────┴────┐                     ┌─────────┴─────────┐
   bin       usr                  bin                  usr
             │                                          │
          ┌──┴──┐                                    ┌──┴──┐
         ast   jim                                  ast   jim

Unidade USB (separada)
         /
    ┌────┴────┐
    x         y
   (inacessíveis!)
```

Após `mount("/dev/sdb0", "/mnt", 0)`, a unidade USB é acessível através da árvore principal — sem precisar se preocupar em qual dispositivo o arquivo se encontra. Arquivos na unidade USB podem ser acessados usando o caminho do diretório-raiz ou do diretório de trabalho.

Os parâmetros de `mount()` são:

- **special** — nome do arquivo especial de blocos para o dispositivo (ex: `/dev/sdb0`)
- **name** — lugar na árvore onde ele deve ser montado (ex: `/mnt`)
- **flag** — se o sistema de arquivos deve ser montado como leitura e escrita ou somente leitura

> 💡 A chamada `mount` torna possível integrar meios removíveis em uma única hierarquia de arquivos integrada, sem precisar se preocupar em qual dispositivo se encontra um arquivo. Partes de discos rígidos (partições ou **dispositivos secundários**), discos rígidos externos e SSDs também podem ser montados dessa maneira.
> 

Quando um sistema de arquivos não é mais necessário, ele pode ser **desmontado** com `umount()`. Depois disso, não estará mais acessível — mas pode ser montado novamente quando necessário.

---

# 🗂️ Outras Syscalls Diversas

Existe também uma série de outras chamadas de sistema. O Tanenbaum examina quatro delas na seção 1.6.4:

**chdir(dirname)** — altera o **diretório de trabalho** atual do processo:

```c
chdir("/usr/ast/test");
// Agora caminhos relativos partem de /usr/ast/test
// Ao invés do diretório anterior
```

**chmod(name, mode)** — altera os **bits de proteção** de um arquivo:

```c
chmod("arquivo.txt", 0644);  // rw-r--r--
chmod("script.sh", 0755);    // rwxr-xr-x
```

**kill(pid, signal)** — envia um **sinal** para um processo. Se um processo estiver preparado para receber aquele sinal, ele poderá tratar o sinal; caso contrário, o processo é finalizado:

```c
kill(1234, SIGTERM);  // pede ao processo 1234 para terminar graciosamente
kill(1234, SIGKILL);  // mata o processo 1234 imediatamente (não pode ser ignorado)
```

**time(&seconds)** — obtém o **tempo decorrido desde 1º de janeiro de 1970** (Unix timestamp) em segundos:

```c
time_t t;
time(&t);  // t agora contém segundos desde 01/01/1970 00:00:00 UTC
```

---

# ✅ Resumo do Conceito

- `mkdir()` — **cria um diretório** vazio com as permissões especificadas
- `rmdir()` — **remove um diretório** — deve estar completamente vazio
- `link()` — **cria uma nova entrada de diretório** apontando para um arquivo existente pelo seu i-number. Permite que o mesmo arquivo apareça em múltiplos locais sob nomes diferentes. Incrementa o link count do i-node
- `unlink()` — **remove uma entrada de diretório**. Decrementa o link count. O arquivo só é removido do disco quando link count chega a zero
- Todo arquivo UNIX tem um **i-number** único que indexa na tabela de **i-nodes** — um diretório é apenas um arquivo com pares (i-number, nome)
- `mount()` — **funde dois sistemas de arquivos** em uma única hierarquia. Aceita o dispositivo, o ponto de montagem e flags (leitura/escrita ou somente leitura)
- `umount()` — **desmonta** um sistema de arquivos previamente montado
- Syscalls diversas: `chdir()` (muda diretório de trabalho), `chmod()` (muda permissões), `kill()` (envia sinal para processo), `time()` (obtém timestamp Unix)