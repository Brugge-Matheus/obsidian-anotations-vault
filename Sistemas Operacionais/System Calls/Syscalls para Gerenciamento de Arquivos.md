---
tags:
  - sistemas-operacionais
  - so/syscalls
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.6.2"
---
# Syscalls para Gerenciamento de Arquivos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.6.2

---

# 📁 O que são Syscalls para Gerenciamento de Arquivos?

Muitas chamadas de sistema relacionam-se ao sistema de arquivos. Nesta seção examinamos as chamadas que operam sobre **arquivos individuais** — abrir, fechar, ler, escrever, mover o ponteiro e obter informações sobre um arquivo.

Essas são as syscalls que você mais usará indiretamente no dia a dia — toda vez que um programa lê um arquivo de configuração, salva um documento ou escreve em um log, ele está usando essas chamadas.

---

# 📋 As Principais Syscalls

| Chamada | Descrição |
| --- | --- |
| `fd = open(file, how, ...)` | Abre um arquivo para leitura, escrita ou ambos |
| `s = close(fd)` | Fecha um arquivo aberto |
| `n = read(fd, buffer, nbytes)` | Lê dados de um arquivo para um buffer |
| `n = write(fd, buffer, nbytes)` | Escreve dados de um buffer para um arquivo |
| `position = lseek(fd, offset, whence)` | Move o ponteiro de posição do arquivo |
| `s = stat(name, &buf)` | Obtém informações sobre um arquivo |

---

# 🔓 open() — Abrindo um Arquivo

Antes de qualquer operação sobre um arquivo, é preciso **abri-lo** com `open()`. Essa chamada especifica o nome do arquivo e o modo de acesso desejado:

```c
fd = open(file, how, ...);
```

- **file** — nome do arquivo a ser aberto, seja como caminho absoluto (`/home/user/dados.txt`) ou relativo ao diretório de trabalho (`dados.txt`)
- **how** — modo de abertura:

| Flag | Significado |
| --- | --- |
| `O_RDONLY` | Aberto apenas para leitura |
| `O_WRONLY` | Aberto apenas para escrita |
| `O_RDWR` | Aberto para leitura e escrita |
| `O_CREAT` | Cria o arquivo se não existir |

Para criar um novo arquivo, usa-se `O_CREAT`. As flags podem ser combinadas com `|`:

```c
// Abrir para leitura:
fd = open("arquivo.txt", O_RDONLY);

// Criar ou sobrescrever para escrita:
fd = open("arquivo.txt", O_WRONLY | O_CREAT, 0644);
//                                            ↑
//                                     permissões (rwxr--r--)
```

## O Descritor de Arquivo (fd)

Quando `open()` tem sucesso, ela retorna um **descritor de arquivo** (*file descriptor*) — um pequeno inteiro não negativo que identifica o arquivo aberto naquele processo. Esse descritor é usado em todas as operações subsequentes sobre o arquivo.

```
fd = 0  →  stdin  (entrada padrão — teclado)
fd = 1  →  stdout (saída padrão — terminal)
fd = 2  →  stderr (saída de erro — terminal)
fd = 3  →  primeiro arquivo aberto pelo programa
fd = 4  →  segundo arquivo aberto
...
```

Os três primeiros descritores (0, 1, 2) são abertos automaticamente pelo SO quando o processo inicia. Quando o programa abre um arquivo, o SO escolhe o menor inteiro disponível — geralmente 3, 4, 5...

> 💡 O descritor de arquivo é apenas um índice em uma tabela interna do processo — a **tabela de arquivos abertos**. Cada entrada dessa tabela contém informações sobre o arquivo: posição atual de leitura/escrita, flags de acesso, referência ao arquivo real no disco, etc.
> 

---

# 🔒 close() — Fechando um Arquivo

Quando o programa termina de usar um arquivo, ele deve fechá-lo com `close()`:

```c
s = close(fd);
```

Fechar o arquivo libera o descritor para ser reutilizado em um `open()` subsequente. Se um processo terminar sem fechar seus arquivos, o SO os fecha automaticamente — mas é boa prática fechá-los explicitamente.

---

# 📖 read() e write() — Lendo e Escrevendo

As chamadas mais usadas de todo o sistema de arquivos:

```c
n = read(fd, buffer, nbytes);   // lê até nbytes bytes do arquivo para buffer
n = write(fd, buffer, nbytes);  // escreve até nbytes bytes do buffer para o arquivo
```

Ambas retornam o número de bytes **efetivamente** lidos ou escritos — e esse número **pode ser menor que nbytes**.

## read() — lê até nbytes, pode retornar menos

`read()` tenta ler **até** `nbytes` bytes do arquivo a partir da posição atual do ponteiro. O retorno `n` pode ser:

| Retorno | Significado |
| --- | --- |
| `= nbytes` | Leu tudo que pediu (caso mais comum) |
| `< nbytes` | Chegou no EOF antes de completar |
| `= 0` | EOF — final do arquivo, sem bytes restantes |
| `= -1` | Ocorreu um erro |

```
Arquivo tem 100 bytes, ponteiro está no byte 90:

read(fd, buffer, 50)  ← pede 50 bytes
→ só tem 10 bytes restantes no arquivo
→ n = 10  (leu apenas 10, não 50)
→ próximo read retornará 0 (EOF)
```

## write() — escreve até nbytes, pode escrever menos

`write()` tenta escrever **até** `nbytes` bytes do buffer para o arquivo. O retorno `n` pode ser:

| Retorno | Significado |
| --- | --- |
| `= nbytes` | Escreveu tudo (caso mais comum) |
| `< nbytes` | Disco cheio, interrompido por sinal, etc. |
| `= -1` | Ocorreu um erro |

Na prática, `write()` quase sempre escreve tudo — a escrita parcial é muito mais rara que a leitura parcial (que acontece normalmente no EOF).

## O fluxo correto sempre verifica o retorno

```c
// Leitura robusta — loop até EOF:
while ((n = read(fd, buffer, sizeof(buffer))) > 0) {
    // processa os n bytes lidos
    // n pode ser menor que sizeof(buffer) — tudo bem!
}
if (n < 0) perror("erro na leitura");

// Escrita robusta:
n = write(fd, buffer, nbytes);
if (n != nbytes) {
    // algo deu errado — disco cheio, etc.
    perror("erro na escrita");
}
```

```c
// Exemplo: lendo um arquivo inteiro
char buffer[512];
int fd = open("dados.txt", O_RDONLY);

while ((n = read(fd, buffer, sizeof(buffer))) > 0) {
    // processa os n bytes lidos
    write(1, buffer, n);  // escreve no stdout (fd=1)
}
close(fd);
```

## Leitura e Escrita Sequencial vs. Aleatória

A maioria dos programas lê e escreve arquivos **sequencialmente** — do começo ao fim. Mas alguns programas de aplicação precisam acessar **qualquer parte** de um arquivo de modo aleatório (bancos de dados, por exemplo).

Para isso, existe um **ponteiro de posição** associado a cada arquivo aberto. Durante a leitura (ou escrita) sequencial, ele aponta automaticamente para o próximo byte a ser lido (ou escrito). Mas você pode movê-lo para qualquer posição com `lseek()`.

---

# ↕️ lseek() — Movendo o Ponteiro de Posição

```c
position = lseek(fd, offset, whence);
```

`lseek` muda o valor do ponteiro de posição do arquivo, de maneira que chamadas subsequentes a `read` ou `write` podem começar de qualquer parte no arquivo.

Os três parâmetros:

- **fd** — descritor do arquivo
- **offset** — deslocamento em bytes (pode ser negativo)
- **whence** — ponto de referência para o deslocamento:

| whence | Referência |
| --- | --- |
| `SEEK_SET` | Início do arquivo |
| `SEEK_CUR` | Posição atual |
| `SEEK_END` | Final do arquivo |

O valor retornado por `lseek` é a **posição absoluta** no arquivo (em bytes) após mover o ponteiro.

```c
// Exemplos de lseek:
lseek(fd, 0, SEEK_SET);    // vai para o início do arquivo
lseek(fd, 100, SEEK_SET);  // vai para o byte 100
lseek(fd, -10, SEEK_END);  // vai para 10 bytes antes do final
lseek(fd, 5, SEEK_CUR);    // avança 5 bytes da posição atual

// Descobrir o tamanho do arquivo:
size = lseek(fd, 0, SEEK_END);  // vai para o final e retorna a posição
```

---

# 📊 stat() — Obtendo Informações sobre um Arquivo

Para cada arquivo, o UNIX registra um conjunto de metadados — tipo do arquivo (regular, especial, diretório), tamanho, data da última modificação, proprietário, permissões, etc. Os programas podem ver essas informações via `stat()`:

```c
s = stat(name, &buf);
```

- **name** — nome do arquivo a ser inspecionado
- **&buf** — ponteiro para uma estrutura onde as informações serão colocadas

A estrutura retornada contém campos como:

```c
struct stat {
    mode_t  st_mode;   // tipo e permissões do arquivo (rwxr-xr-x)
    off_t   st_size;   // tamanho em bytes
    time_t  st_mtime;  // data da última modificação
    uid_t   st_uid;    // UID do proprietário
    gid_t   st_gid;    // GID do grupo
    // ... e outros campos
};
```

Existe também `fstat()` que faz a mesma coisa para um arquivo **já aberto** (usando o descritor ao invés do nome):

```c
s = fstat(fd, &buf);  // para arquivo aberto
```

---

# 🔄 O Ciclo Completo de uso de um Arquivo

Juntando todas as syscalls, o ciclo típico de uso de um arquivo é:

```
1. open()  → obter o descritor fd
      ↓
2. read()  → ler dados do arquivo para um buffer em memória
   write() → escrever dados de um buffer para o arquivo
   lseek() → mover o ponteiro de posição se necessário
      ↓
3. close() → liberar o descritor
```

```c
// Exemplo completo — copiar arquivo1 para arquivo2:
int fd1 = open("arquivo1.txt", O_RDONLY);
int fd2 = open("arquivo2.txt", O_WRONLY | O_CREAT, 0644);

char buf[4096];
int n;
while ((n = read(fd1, buf, sizeof(buf))) > 0) {
    write(fd2, buf, n);
}

close(fd1);
close(fd2);
```

---

# 📌 Relação com o Conceito de Descritor de Arquivo

O descritor de arquivo é a abstração central que conecta todas essas syscalls. Ele é um **inteiro simples** que representa um arquivo aberto dentro do processo. O SO mantém, para cada processo, uma **tabela de descritores de arquivos abertos**:

```
Tabela de descritores do processo:

fd | Arquivo              | Posição atual | Modo
─────────────────────────────────────────────────
0  | stdin  (teclado)     | -             | leitura
1  | stdout (terminal)    | -             | escrita
2  | stderr (terminal)    | -             | escrita
3  | /home/user/dados.txt | 1.024 bytes   | leitura
4  | /var/log/app.log     | 8.192 bytes   | escrita
```

Quando `close(fd)` é chamado, aquela linha da tabela é liberada. O próximo `open()` reutilizará aquele número.

---

# ✅ Resumo do Conceito

- `open()` — **abre um arquivo** e retorna um descritor (`fd`). Verifica permissões. Pode criar o arquivo com `O_CREAT`
- `close()` — **fecha o arquivo** e libera o descritor para reutilização
- `read()` — **lê dados** do arquivo para um buffer em memória. Retorna o número de bytes efetivamente lidos
- `write()` — **escreve dados** de um buffer para o arquivo. Retorna o número de bytes efetivamente escritos
- `lseek()` — **move o ponteiro de posição** do arquivo para permitir acesso aleatório. Recebe fd, offset e ponto de referência (início, posição atual ou fim)
- `stat()` / `fstat()` — **obtém metadados** do arquivo (tamanho, permissões, datas, proprietário)
- O **descritor de arquivo** (`fd`) é o inteiro que conecta todas essas chamadas — representa o arquivo aberto dentro do processo, com sua posição atual e modo de acesso
- O ciclo completo é sempre: **open → read/write/lseek → close**