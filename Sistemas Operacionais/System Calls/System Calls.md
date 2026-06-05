---
tags:
  - sistemas-operacionais
  - so/syscalls
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.6"
---
# System Calls

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.6

---

# 📞 O que são Chamadas de Sistema?

Os sistemas operacionais têm duas funções principais: fornecer abstrações para os programas de usuários e gerenciar os recursos do computador. A interação entre programas de usuários e o SO lida com a primeira — criar, escrever, ler e excluir arquivos, gerenciar processos, realizar entradas e saídas.

Uma **chamada de sistema** (*system call* ou *syscall*) é o mecanismo pelo qual um programa em modo usuário solicita um serviço ao sistema operacional — que roda em modo núcleo. É a única porta de entrada legítima para o kernel.

> 💡 Fazer uma chamada de sistema é como fazer um **tipo especial de chamada de procedimento** — mas com a propriedade adicional de chavear do modo usuário para o modo núcleo. Chamadas de procedimento normais não mudam o modo de execução.
> 

---

# 🗂️ Conceitos Fundamentais — Glossário

Antes de entender como as chamadas de sistema funcionam, é essencial entender alguns termos que aparecem frequentemente:

## Linguagem de Montagem (Assembly)

**Linguagem de montagem** (*assembly language*) é uma linguagem de programação de baixíssimo nível — um passo acima do código de máquina binário. Cada instrução em assembly corresponde diretamente a uma instrução do processador.

```nasm
; Exemplo em assembly x86-64:
MOV RAX, 1        ; coloca o número 1 no registrador RAX
MOV RDI, 1        ; coloca 1 no registrador RDI (stdout)
MOV RSI, msg      ; endereço da mensagem no RSI
MOV RDX, 13       ; tamanho da mensagem no RDX
SYSCALL           ; executa a chamada de sistema (write)
```

Por que isso importa para chamadas de sistema? Porque **os parâmetros de uma chamada de sistema são passados em registradores específicos**, e a instrução que dispara a chamada (`SYSCALL` em x86-64) é uma instrução de assembly. Mesmo quando você usa C ou Python, por baixo dos panos é gerado código de montagem que executa essa sequência.

## Chamada de Procedimento (Function Call)

Uma **chamada de procedimento** é simplesmente chamar uma função — o fluxo salta para o código daquela função, executa, e retorna. Em C:

```c
int resultado = soma(3, 5);  // chamada de procedimento normal
```

O que acontece por baixo:

1. Parâmetros são colocados na pilha ou em registradores
2. CPU salta para o endereço da função
3. Função executa
4. Resultado é retornado
5. CPU volta para a instrução seguinte à chamada

**Importante:** chamadas de procedimento normais **não mudam o modo de execução** — tudo acontece em modo usuário.

## Rotina de Biblioteca (Library Routine)

Uma **rotina de biblioteca** é uma função que faz parte de uma biblioteca de código — como a **libc** (*C Standard Library*) no Linux. Funções como `printf()`, `malloc()`, `fopen()` são rotinas de biblioteca.

Elas existem para facilitar a vida do programador — em vez de você escrever assembly para cada chamada de sistema, a biblioteca fornece funções em C de alto nível que fazem isso por você.

```
Você chama:   printf("Olá")
              ↓
Biblioteca:   rotina printf() da libc
              ↓ formata a string
              ↓ chama write() (outra rotina da libc)
              ↓ write() executa a instrução SYSCALL
              ↓ chamada de sistema real ao kernel
```

## Chamada de Sistema vs. Chamada de Procedimento

|  | Chamada de Procedimento | Chamada de Sistema |
| --- | --- | --- |
| **Destino** | Função no mesmo processo | Kernel do SO |
| **Modo** | Permanece em modo usuário | Chaveia para modo núcleo |
| **Mecanismo** | Instrução CALL normal | Instrução SYSCALL/trap |
| **Custo** | Barato | Caro (troca de contexto) |
| **Exemplo** | `soma(3, 5)` | `read(fd, buffer, n)` |

---

# ⚙️ Como uma Chamada de Sistema Funciona — Os 11 Passos

O Tanenbaum usa a chamada `read(fd, buffer, nbytes)` como exemplo para ilustrar o mecanismo completo. Essa chamada tem três parâmetros:

- `fd` — o arquivo a ser lido
- `buffer` — ponteiro para onde os dados serão colocados
- `nbytes` — quantos bytes ler

> 📌 **Figura 1.17 — Os 11 passos na realização da chamada de sistema read**
> 

```
ESPAÇO DO USUÁRIO:
┌────────────────────────────────────────────────┐
│  Programa do usuário chamando read             │
│                                                │
│  1. Coloca fd no registrador RDI               │
│  2. Coloca buffer no registrador RSI           │
│  3. Coloca nbytes no registrador RDX           │
│  4. Chama a rotina read da biblioteca (libc)   │
│                                                │
│  Procedimento read da biblioteca:              │
│  5. Coloca o NÚMERO da syscall read no RAX     │
│  6. Executa a instrução SYSCALL (trap)    ↓    │
│                                                │
│  9. Retorno para o procedimento biblioteca     │
│  10. Procedimento retorna para o programa      │
│  11. Programa continua na próxima instrução    │
└────────────────────────────────────────────────┘

ESPAÇO DO NÚCLEO (Sistema Operacional):
┌────────────────────────────────────────────────┐
│  6. SYSCALL chaveia para modo núcleo           │
│     CPU salta para endereço fixo no kernel     │
│                                                │
│  7. Despacho: kernel examina RAX               │
│     encontra o número da syscall read          │
│     usa tabela de ponteiros para encontrar     │
│     o tratador correto                         │
│                                                │
│  8. Tratador da syscall read executa           │
│     realiza a leitura do arquivo               │
│                                                │
│  9. Retorna para o espaço do usuário           │
└────────────────────────────────────────────────┘
```

**Detalhando cada passo:**

**Passos 1-3** — O programa prepara os parâmetros. Nos processadores x86-64, a convenção AMD64 ABI define que os seis primeiros parâmetros vão nos registradores `RDI`, `RSI`, `RDX`, `RCX`, `R8` e `R9`. Se houver mais de seis argumentos, o restante vai para a pilha.

**Passo 4** — O programa chama a rotina `read()` da biblioteca libc como se fosse uma função normal.

**Passo 5** — A biblioteca coloca o **número da chamada de sistema** no registrador `RAX`. Cada syscall tem um número único — `read` é syscall número 0 no Linux x86-64, `write` é 1, `open` é 2, etc.

**Passo 6** — A biblioteca executa a instrução `SYSCALL`. Essa instrução chaveia a CPU do modo usuário para o modo núcleo e salta para um **endereço fixo** dentro do kernel.

> 💡 A instrução `SYSCALL` difere de uma chamada de procedimento normal em dois aspectos: (1) ela troca para o modo núcleo, e (2) em vez de saltar para um endereço arbitrário, ela salta para um **único local fixo** — o ponto de entrada do kernel. Programas não podem escolher para onde "entrar" no kernel.
> 

**Passo 7** — O kernel examina o número da syscall no registrador `RAX` e usa como índice em uma **tabela de vetores de chamadas de sistema** para encontrar o tratador correto para `read`.

**Passo 8** — O **tratador da chamada de sistema** executa — realiza a leitura do arquivo de fato.

**Passo 9** — O tratador termina. O controle *pode* ser retornado para a rotina de biblioteca no espaço do usuário.

> ⚠️ **Por que "pode ser retornado"?** A chamada de sistema pode **bloquear** quem a chamou. Por exemplo, se o programa está tentando ler do teclado e nada foi digitado, o SO vai bloquear o processo — e executar outro processo enquanto aguarda. Quando a entrada estiver disponível, esse processo receberá a atenção do sistema e executará os passos 9 e 10.
> 

**Passos 10-11** — A rotina de biblioteca retorna para o programa do usuário. O programa continua com a próxima instrução.

---

# ❌ Quando a Chamada de Sistema Falha

Se a chamada de sistema não puder ser realizada — por parâmetro inválido ou erro de disco — o **contador** (valor de retorno) passa a valer `-1`, e o número do erro é colocado em uma variável global chamada `errno`.

```c
contador = read(fd, buffer, nbytes);

if (contador == -1) {
    perror("Erro na leitura");  // imprime mensagem baseada em errno
}
```

Os programas devem **sempre** conferir os resultados de uma chamada de sistema para verificar se ocorreu algum erro.

---

# 🌐 POSIX — O Padrão de Interfaces do SO

O padrão **POSIX** (*Portable Operating System Interface*, padrão internacional IEEE 9945-1) é um **padrão técnico internacional** criado pelo **IEEE** que define como um sistema operacional deve se comportar para ser considerado compatível. Não é um produto, não é uma empresa, não é um SO — é um **documento que define um contrato de comportamento**.

## Por que o POSIX foi criado?

Nos anos 80, havia uma proliferação de sistemas UNIX diferentes — HP-UX, AIX, SunOS, Xenix... Todos eram "UNIX", mas com diferenças suficientes para que um programa escrito para um não rodasse em outro sem modificações. Em 1988, o IEEE publicou o primeiro padrão POSIX com uma ideia simples:

> **"Se o seu SO implementar essas interfaces, programas escritos para POSIX rodarão no seu sistema."**
> 

## O que o POSIX define exatamente?

O POSIX define **chamadas de procedimento (interfaces)**, não necessariamente chamadas de sistema. É um contrato de comportamento — especifica o que cada função deve fazer e quais erros pode retornar — mas **não dita como cada uma deve ser implementada por baixo**. Cada procedimento POSIX pode ser implementado de três formas:

**1. Syscall direta** — wrapper fino para uma chamada de sistema:

```
read()  → sempre syscall
fork()  → sempre syscall (só o kernel pode criar processos)
open()  → sempre syscall (só o kernel gerencia arquivos)
```

**2. Rotina de biblioteca pura** — inteiramente em modo usuário, sem syscall:

```
sqrt()    → cálculo matemático puro, zero syscalls
strlen()  → percorre bytes na memória, zero syscalls
```

**3. Misto** — trabalho em modo usuário + syscall só quando necessário:

```
printf() → formata string em modo usuário
         → acumula no buffer
         → quando buffer enche → chama write() → syscall

malloc() → se há espaço livre no heap → retorna direto (modo usuário)
         → se o heap está cheio → chama brk()/mmap() → syscall
```

Essa distinção importa para **performance** — syscalls são caras (trap, mudança de modo, salvamento de contexto). A libc minimiza o número delas fazendo o máximo possível em modo usuário.

## Analogia

POSIX é como o **manual de especificações de uma tomada elétrica** — define dimensões, voltagem e formato de pinos, mas não como a tomada deve ser fabricada internamente. Qualquer aparelho fabricado seguindo o padrão funciona em qualquer tomada do padrão. O POSIX faz o mesmo para software e sistemas operacionais.

## Quem segue o POSIX?

| Sistema | Status POSIX |
| --- | --- |
| **macOS** | Certificado oficialmente |
| **Linux** | Compatível mas não certificado (certificação custa dinheiro) |
| **FreeBSD, OpenBSD** | Amplamente compatível |
| **Windows** | Não compatível nativamente (WSL2 adiciona compatibilidade) |
| **Android** | Parcialmente compatível (baseado em Linux) |

O POSIX tem cerca de **100 chamadas de procedimento** — e em grande medida os serviços que elas determinam definem o que o SO tem de fazer.

---

# 📊 Categorias de Chamadas de Sistema

As chamadas de sistema POSIX são agrupadas em categorias principais:

**Gerenciamento de Processos:**

| Chamada | Descrição |
| --- | --- |
| `pid = fork()` | Cria um processo-filho idêntico ao pai |
| `pid = waitpid(pid, &statloc, options)` | Espera que um processo-filho seja concluído |
| `s = execve(name, argv, environp)` | Substitui a imagem do núcleo de um processo |
| `exit(status)` | Conclui a execução do processo e retorna o estado |

**Gerenciamento de Arquivos:**

| Chamada | Descrição |
| --- | --- |
| `fd = open(file, how, ...)` | Abre um arquivo para leitura, escrita ou ambos |
| `s = close(fd)` | Fecha um arquivo aberto |
| `n = read(fd, buffer, nbytes)` | Lê dados de um arquivo para um buffer |
| `n = write(fd, buffer, nbytes)` | Escreve dados de um buffer para um arquivo |

---

# 🔑 fork() e copy-on-write — A Criação de Processos

A chamada `fork()` é a **única maneira para se criar um processo novo em POSIX**. Ela cria uma cópia exata do processo original — incluindo todos os descritores de arquivos, registradores e tudo mais.

```c
pid = fork();

if (pid == 0) {
    execve("programa", argv, env);  // filho substitui sua imagem
} else {
    waitpid(pid, &status, 0);       // pai aguarda o filho terminar
}
```

**Copy-on-write:** pai e filho compartilham uma única cópia física da memória — até que um dos dois modifique um valor. Quando isso acontece, o SO faz uma cópia apenas do pequeno trecho modificado. Isso reduz drasticamente a quantidade de memória copiada.

---

# ✅ Resumo do Conceito

- Uma **chamada de sistema** é o mecanismo pelo qual programas em modo usuário solicitam serviços ao SO em modo núcleo — a única porta de entrada legítima para o kernel
- **Linguagem de montagem** (*assembly*) — linguagem de baixíssimo nível onde cada instrução corresponde a uma instrução do processador; é o nível onde syscalls são efetivamente disparadas
- **Chamada de procedimento** — chamar uma função normal, sem mudar o modo de execução
- **Rotina de biblioteca** — função de alto nível (como `printf`, `malloc`) que internamente pode fazer chamadas de sistema
- O fluxo completo envolve **11 passos**: preparar parâmetros em registradores → chamar rotina de biblioteca → biblioteca coloca número da syscall em RAX → executa SYSCALL (trap) → kernel despacha para o tratador correto → tratador executa → retorna para biblioteca → retorna para o programa
- A instrução `SYSCALL` **chaveia para modo núcleo** e salta para um endereço fixo — não arbitrário — por segurança
- Syscalls podem **bloquear** o processo — o SO executa outro processo enquanto aguarda
- Falhas retornam `-1` e definem `errno` — sempre verificar o retorno
- **POSIX** é um padrão técnico do IEEE que define interfaces de comportamento — não apenas syscalls, mas qualquer procedimento que o SO deve oferecer
- Cada procedimento POSIX pode ser: **syscall direta**, **rotina de biblioteca pura** ou **misto** — dependendo do que é mais eficiente
- `fork()` é a única forma de criar processos em POSIX — usa **copy-on-write** para eficiência de memória