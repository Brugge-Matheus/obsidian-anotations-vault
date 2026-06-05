---
tags:
  - sistemas-operacionais
  - so/syscalls
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.6.1"
---
# Syscalls para Gerenciamento de Processos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.6.1

---

# 🔄 O que são Syscalls para Gerenciamento de Processos?

O primeiro grupo de chamadas de sistema POSIX lida com o **gerenciamento de processos** — criar processos, executar programas, aguardar conclusão e encerrar. Essas syscalls são o coração de como qualquer programa interativo funciona no UNIX/Linux.

Antes de entrar em cada chamada, é importante ter em mente o modelo mental:

> **Todo programa em execução é um processo. Todo processo foi criado por outro processo. Existe sempre uma hierarquia — pai cria filho, filho pode criar neto, e assim por diante.**
> 

## O Processo init — O Pai de Todos

Isso levanta uma pergunta natural: se todo processo vem de um fork, quem criou o primeiro? A resposta é o **kernel** — durante o boot, após inicializar o hardware, o kernel cria **manualmente** o processo de PID 1, chamado de **init** (ou **systemd** nos sistemas Linux modernos). Esse é o único processo que não vem de um fork. A partir dele, tudo é fork + exec:

```
Kernel boot
     ↓ cria manualmente
  init/systemd (PID 1)    ← o único processo não criado por fork
     ↓ fork + exec
  Serviços do sistema (sshd, cron, networkd...)
     ↓ fork + exec
  Login / Display Manager
     ↓ fork + exec
  Shell (bash, zsh...)    ← é filho do processo de login
     ↓ fork + exec
  ls, grep, vim...        ← são filhos do shell
```

Você pode ver essa árvore completa rodando `pstree` num terminal Linux:

```
systemd (PID 1)
├── networkd
├── sshd
│   └── bash              ← shell da sessão SSH
│       └── ls            ← comando rodado no shell
├── gdm (display manager)
│   └── gnome-terminal
│       └── bash          ← shell gráfico
└── cron
    └── (jobs agendados)
```

Cada linha é um fork. Cada filho tem um pai. Tudo conectado até o PID 1.

---

# 📋 As Principais Syscalls

| Chamada | Descrição |
| --- | --- |
| `pid = fork()` | Cria um processo-filho idêntico ao pai |
| `pid = waitpid(pid, &statloc, options)` | Pai espera que um processo-filho termine |
| `s = execve(name, argv, environp)` | Substitui o programa em execução por outro |
| `exit(status)` | Encerra o processo e retorna estado ao pai |

---

# fork() — Criando um Processo

`fork()` é a **única maneira de criar um processo novo em POSIX**. Ela cria uma cópia exata do processo original — incluindo todos os descritores de arquivos abertos, registradores, espaço de endereçamento e tudo mais.

Após a fork, existem dois processos rodando o **mesmo código** a partir do mesmo ponto. Como distinguir quem é quem? Pelo valor de retorno:

```c
pid = fork();

if (pid < 0) {
    // erro — fork falhou
} else if (pid == 0) {
    // estamos no PROCESSO-FILHO
    // fork() retorna 0 para o filho
} else {
    // estamos no PROCESSO-PAI
    // fork() retorna o PID do filho para o pai
}
```

```
Antes do fork:
┌─────────────────┐
│   Processo A    │
│   pid = fork()  │
└─────────────────┘

Após o fork:
┌─────────────────┐     ┌─────────────────┐
│  Processo A     │     │  Processo B      │
│  (pai)          │     │  (filho)         │
│  fork() = PID_B │     │  fork() = 0      │
└─────────────────┘     └─────────────────┘
         │                      │
    executa código          executa código
    do pai                  do filho
```

## Copy-on-write

A memória do filho pode ser **compartilhada com o pai por cópia-na-escrita** (*copy-on-write*). Em vez de copiar todos os gigabytes de memória do processo pai na hora do fork, o SO marca as páginas como compartilhadas e só faz a cópia quando um dos dois **efetivamente modifica** algum dado.

Isso é fundamental para performance — a maioria dos processos-filhos usa `execve()` logo depois do fork para trocar de programa completamente, então nunca chegam a modificar a memória do pai.

```
Fork acontece:
Pai e filho compartilham as mesmas páginas físicas na RAM

Filho modifica uma variável:
SO faz uma cópia APENAS daquela página
→ pai e filho agora têm cópias independentes
→ o restante da memória ainda é compartilhado
```

> 💡 Além do mais, parte da memória — como o **segmento de texto** (código do programa) — é **inalterável**, de modo que sempre pode ser compartilhada entre pai e filho. O código não muda, então não precisa ser copiado.
> 

---

# ⚡ execve() — Trocando o Programa em Execução

`execve()` **substitui completamente o programa em execução dentro do processo atual**. O processo continua existindo com o mesmo PID, mas seu código, dados e pilha são completamente substituídos pelo novo programa.

> 💡 **Analogia:** Um processo é como um ator de teatro. O ator existe como entidade — tem nome e identidade (o PID). O `execve` é como trocar completamente o papel que ele vai interpretar — novo texto, novo figurino, nova personalidade. O ator continua sendo o mesmo (mesmo PID), mas agora interpreta um personagem completamente diferente.
> 

```
ANTES do execve (processo é cópia do shell):
┌──────────────────────────────────────┐
│  Segmento de texto: código do shell  │
│  Segmento de dados: variáveis shell  │
│  Pilha: contexto do shell            │
│  PID: 4231                           │
└──────────────────────────────────────┘
          ↓ execve("ls", argv, env)

DEPOIS do execve (processo virou ls):
┌──────────────────────────────────────┐
│  Segmento de texto: código do ls     │ ← completamente substituído
│  Segmento de dados: variáveis do ls  │ ← completamente substituído
│  Pilha: nova pilha do ls             │ ← completamente substituído
│  PID: 4231                           │ ← o único que permanece igual!
└──────────────────────────────────────┘
```

## execve nunca retorna quando tem sucesso

Quando `execve` tem sucesso, o código que o chamou **não existe mais** — foi apagado e substituído pelo novo programa. Não tem para onde retornar. O novo programa começa do zero, do seu próprio `main()`.

`execve` só retorna se der **erro** — arquivo não encontrado, sem permissão de execução, etc:

```c
execve("/bin/ls", argv, env);
// Se chegou aqui, é porque deu erro — execve bem-sucedido nunca retorna
perror("execve falhou");
exit(1);
```

## Os três parâmetros

```c
s = execve(name, argv, environp);
```

- **name** — caminho do arquivo a executar (`/bin/ls`, `./meu_programa`, etc.)
- **argv** — array de argumentos (o famoso `argc/argv` do `main`)
- **environp** — array de variáveis de ambiente (`PATH`, `HOME`, etc.)

**Por que fork + exec? Por que não criar um processo diretamente com o programa?**

O modelo fork+exec é elegante porque o filho pode **configurar seu ambiente** entre o fork e o exec — redirecionar arquivos, fechar descritores, mudar diretório de trabalho — antes de carregar o novo programa. Isso dá total flexibilidade sem complicar a interface.

```
Shell executando "ls -la":

1. shell chama fork()
         ↓
   ┌─────────────┐     ┌─────────────┐
   │    shell    │     │  filho      │
   │  (aguarda)  │     │  (novo proc)│
   └─────────────┘     └─────────────┘
                              ↓
               2. filho configura ambiente
                  (ex: redirecionar stdout)
                              ↓
               3. filho chama execve("ls", ["-la"], env)
                              ↓
               4. filho VIRA o programa ls
                  (mesmo PID, mas código novo)
                              ↓
               5. ls executa, imprime resultado
                              ↓
               6. ls chama exit(0)
                              ↓
   shell recebe sinal       filho termina
   de que o filho acabou
         ↓
   shell exibe o prompt
```

## argv — Os Argumentos do Programa

O segundo parâmetro do `execve` é o `argv` — um array de strings com os argumentos. O elemento `argv[0]` é sempre o nome do programa em si:

```
Comando digitado:   cp arquivo1 arquivo2

argv[0] = "cp"
argv[1] = "arquivo1"
argv[2] = "arquivo2"
argv[3] = NULL        ← indica o fim do array
```

O `main` em C recebe esses argumentos:

```c
int main(int argc, char* argv[], char* envp[]) {
    // argc = 3 (quantidade de argumentos)
    // argv[0] = "cp"
    // argv[1] = "arquivo1"
    // argv[2] = "arquivo2"
    // envp = variáveis de ambiente
}
```

## envp — O Ambiente

O terceiro parâmetro é o array de variáveis de ambiente — strings no formato `nome=valor`:

```
envp[0] = "PATH=/usr/bin:/bin"
envp[1] = "HOME=/home/usuario"
envp[2] = "TERM=xterm"
envp[3] = NULL
```

Há rotinas de biblioteca para acessar variáveis de ambiente (`getenv("PATH")`), listar todas e personalizar comportamentos como a impressora padrão a ser utilizada.

> 💡 Na Figura 1.19, nenhum ambiente é passado para o processo-filho, então o terceiro parâmetro de `execve` é zero. Na prática, o ambiente é quase sempre repassado.
> 

---

# ⏳ waitpid() — Aguardando o Filho Terminar

Depois de criar um filho com `fork()`, o pai geralmente precisa saber quando o filho terminou. Para isso existe `waitpid()`:

```c
pid_t resultado = waitpid(pid, &statloc, options);
```

- **pid** — qual filho aguardar. Se for `-1`, aguarda qualquer filho
- **&statloc** — ponteiro onde será guardado o **estado de saída** do filho (se terminou normalmente ou com erro, e o valor de retorno)
- **options** — comportamento adicional (ex: retornar imediatamente se nenhum filho terminou)

```
Pai chama waitpid(-1, &status, 0)
         ↓
SO bloqueia o pai
         ↓
Filho executa e eventualmente chama exit(0)
         ↓
SO desbloqueia o pai
         ↓
statloc contém o código de saída do filho
         ↓
Pai continua
```

> 💡 `waitpid` pode aguardar um processo-filho específico (passando seu PID) ou qualquer filho (passando `-1`). Diversas opções também são fornecidas, especificadas pelo terceiro parâmetro — por exemplo, retornar imediatamente se nenhum processo-filho já tiver terminado.
> 

---

# 🚪 exit() — Encerrando o Processo

`exit(status)` encerra o processo e retorna o estado de saída ao pai via `waitpid`. O status é um número de 0 a 255:

- **0** — convencionalmente significa sucesso
- **diferente de 0** — indica algum tipo de erro

```c
exit(0);   // sucesso
exit(1);   // erro genérico
exit(2);   // erro de uso (parâmetros incorretos)
```

---

# 💻 O Shell Simplificado — Tudo junto

A Figura 1.19 do Tanenbaum mostra como um shell simplificado combina todas essas syscalls:

```c
#define TRUE 1

while (TRUE) {
    type_prompt();                     // mostra prompt na tela
    read_command(command, parameters); // lê entrada do terminal

    if (fork() != 0) {
        /* Código do processo-pai */
        waitpid(-1, &status, 0);       // aguarda o processo-filho acabar
    } else {
        /* Código do processo-filho */
        execve(command, parameters, 0); // executa o comando
    }
}
```

Esse loop simples é a essência de qualquer shell:

1. Mostra o prompt e lê o comando
2. `fork()` cria um filho
3. O **pai** chama `waitpid()` e aguarda
4. O **filho** chama `execve()` e vira o programa do comando
5. O comando executa, termina com `exit()`
6. O pai é desbloqueado e volta ao passo 1

---

# 🧱 Os Três Segmentos de Memória de um Processo

Processos em UNIX têm sua memória dividida em três segmentos:

> 📌 **Figura 1.20 — Os processos têm três segmentos: texto, dados e pilha**
> 

```
Endereço (hexa)
FFFF ┌─────────────┐
     │    Pilha    │ ← cresce para baixo
     │  ↓          │   variáveis locais, parâmetros,
     │             │   endereços de retorno
     │   Lacuna    │ ← espaço de endereço não utilizado
     │             │   a pilha cresce automaticamente
     │  ↑          │   o segmento de dados cresce com brk()
     │    Dados    │ ← variáveis globais e estáticas
     │             │   cresce para cima com brk()
     │    Texto    │ ← código do programa (somente leitura)
0000 └─────────────┘
```

- **Segmento de texto** — as instruções do programa (código). É somente leitura e pode ser compartilhado entre pai e filho após fork
- **Segmento de dados** — variáveis globais e estáticas. Cresce para cima quando necessário através da chamada de sistema `brk()` — que especifica o novo endereço onde o segmento de dados deve terminar
- **Segmento de pilha** — variáveis locais e contexto de funções. Cresce automaticamente para baixo conforme o necessário

> 💡 A chamada `brk()` não é definida pelo padrão POSIX — os programadores são encorajados a usar a rotina de biblioteca `malloc` para alocar dinamicamente memória. A implementação subjacente do `malloc` usa `brk()` internamente, mas isso é transparente para o programador.
> 

---

# ✅ Resumo do Conceito

- `fork()` — **única forma de criar processos em POSIX**. Cria uma cópia exata do processo atual. Pai recebe o PID do filho, filho recebe 0. Usa **copy-on-write** para eficiência
- `execve()` — **substitui o programa em execução** por outro. O processo mantém o mesmo PID mas seu código, dados e pilha são completamente trocados. Recebe nome do programa, array de argumentos (`argv`) e variáveis de ambiente (`envp`)
- `waitpid()` — **pai aguarda o filho terminar**. Recebe o PID do filho desejado (ou -1 para qualquer filho) e armazena o estado de saída em `statloc`
- `exit()` — **encerra o processo** e retorna um código de saída (0 = sucesso, diferente de 0 = erro) ao pai via `waitpid`
- O modelo **fork + exec** é elegante porque o filho pode configurar seu ambiente entre os dois calls antes de carregar o novo programa
- Processos têm três segmentos de memória: **texto** (código, fixo), **dados** (variáveis globais, cresce com `brk()`), **pilha** (contexto de funções, cresce automaticamente)