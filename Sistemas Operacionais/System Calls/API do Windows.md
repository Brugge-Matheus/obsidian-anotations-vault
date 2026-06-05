---
tags:
  - sistemas-operacionais
  - so/syscalls
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.6.5"
---
# API do Windows

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.6.5

---

# 🪟 O que é a API do Windows?

Até a seção 1.6.4, o Tanenbaum focou primariamente no UNIX. Agora ele examina o Windows — que difere do UNIX de uma maneira fundamental na forma como os programas interagem com o SO.

O Windows e o UNIX têm modelos de programação radicalmente diferentes:

**UNIX** — um programa consiste em um código que faz uma coisa ou outra e **realiza chamadas de sistema** para ter determinados serviços executados. Há uma relação quase de um para um entre chamadas de sistema e procedimentos de biblioteca.

**Windows** — um programa é normalmente **direcionado por eventos** (*event-driven*). O programa principal espera algum evento acontecer, então chama uma rotina para lidar com ele.

```
Modelo UNIX:                    Modelo Windows:
────────────────────            ────────────────────────────
código executa tarefas          programa fica aguardando
          ↓                               ↓
chama syscall quando            evento acontece (clique, tecla)
precisa de serviço                        ↓
          ↓                     handler é chamado para tratar
continua executando                       ↓
                                programa volta a aguardar
```

---

# 🎯 Eventos Típicos no Windows

Eventos típicos que disparam handlers no Windows:

- Teclas sendo pressionadas
- O mouse sendo movido
- Um botão do mouse acionado
- Uma unidade USB sendo inserida ou removida do computador

**Tratadores** (*handlers*) são chamados para processar o evento e atualizar a tela e o estado do programa interno. Como um todo, isso leva a um estilo de programação diferente do UNIX — mas como o foco do livro está na função e na estrutura do SO, esses modelos de programação diferentes não são aprofundados.

---

# 📦 WinAPI — Win32 e Win64

A Microsoft definiu um conjunto de rotinas chamadas de **WinAPI**, **API Win32** ou **Win64 API** (*application programming interface* — interface de programação de aplicativos) que os programadores usam para acessar os serviços do SO.

Essa interface tem suporte (parcial) de todas as versões do Windows desde o **Windows 95**. Na prática:

- **Win32** — interface que conta com o suporte de todas as versões do Windows
- **Win64** — é, em grande parte, a Win32 com ponteiros maiores (64 bits)
- O livro se concentra na **Win32**

> 💡 O número de chamadas da API Win32 é extremamente grande, chegando a **milhares**. Em comparação, o POSIX tem apenas em torno de 100 chamadas de procedimento.
> 

---

# 🔀 Win32 vs. UNIX — Comparativo de Chamadas

A Figura 1.23 do Tanenbaum mapeia as principais chamadas UNIX para seus equivalentes Win32:

> 📌 **Figura 1.23 — Chamadas da API Win32 correspondentes às chamadas UNIX**
> 

| UNIX | Win32 | Descrição |
| --- | --- | --- |
| `fork` | `CreateProcess` | Cria um novo processo |
| `waitpid` | `WaitForSingleObject` | Pode esperar que um processo termine |
| `execve` | *(nenhuma)* | `CreateProcess` = fork + execve |
| `exit` | `ExitProcess` | Encerra a execução |
| `open` | `CreateFile` | Cria um arquivo ou abre um arquivo existente |
| `close` | `CloseHandle` | Fecha um arquivo |
| `read` | `ReadFile` | Lê dados a partir de um arquivo |
| `write` | `WriteFile` | Escreve dados em um arquivo |
| `lseek` | `SetFilePointer` | Move o ponteiro do arquivo |
| `stat` | `GetFileAttributesEx` | Obtém vários atributos do arquivo |
| `mkdir` | `CreateDirectory` | Cria um novo diretório |
| `rmdir` | `RemoveDirectory` | Remove um diretório vazio |
| `link` | *(nenhuma)* | Win32 não dá suporte a links |
| `unlink` | `DeleteFile` | Destrói um arquivo existente |
| `mount` | *(nenhuma)* | Win32 não dá suporte a mount |
| `umount` | *(nenhuma)* | Win32 não dá suporte a umount |
| `chdir` | `SetCurrentDirectory` | Altera o diretório de trabalho atual |
| `chmod` | *(nenhuma)* | Win32 não dá suporte à segurança (embora o NT suporte) |
| `kill` | *(nenhuma)* | Win32 não dá suporte a sinais |
| `time` | `GetLocalTime` | Obtém o horário atual |

---

# 🔍 Diferenças Fundamentais

## CreateProcess = fork + execve

No UNIX, criar um processo e executar um programa são operações separadas — `fork()` cria o processo e `execve()` substitui o programa. No Windows, **`CreateProcess` faz as duas coisas de uma vez** — cria um novo processo e especifica o programa a executar.

Além disso, `CreateProcess` tem muitos parâmetros especificando as propriedades do processo recentemente criado. No UNIX, essas configurações são feitas entre o fork e o exec pelo processo-filho.

## Sem hierarquia de processos

O Windows **não tem uma hierarquia de processos** como o UNIX. No UNIX, existe sempre um processo-pai e um processo-filho — o pai pode controlar o filho. No Windows, após um processo ser criado, o criador e a criatura são iguais. `WaitForSingleObject` é usado para esperar por um evento, como um processo terminar.

> 💡 **Criar processos no Windows é mais caro que no UNIX/Linux.** A razão é diretamente ligada à ausência de herança via COW:
> 

> 
> 

> No **UNIX**, `fork()` é extremamente barato — não copia nada imediatamente. Cria uma nova tabela de páginas apontando para os mesmos frames físicos e marca tudo como COW. Nenhum dado é copiado fisicamente até alguém modificar algo.
> 

> 
> 

> No **Windows**, `CreateProcess` precisa fazer muito mais trabalho desde o início: aloca novo espaço de endereçamento do zero, carrega o executável e todas as DLLs do disco, aloca e inicializa pilha e heap, configura handles e variáveis de ambiente.
> 

> 
> 

> Essa diferença tem consequências reais de design — historicamente o UNIX favoreceu **processos** como unidade de concorrência (fork é barato), enquanto o Windows favoreceu **threads** (CreateProcess é caro, threads são baratas em qualquer sistema). Servidores web no UNIX como o Apache usam múltiplos processos filhos. Servidores no Windows como o IIS usam pools de threads dentro de um único processo.
> 

## Sem links, mount e sinais

A interface Win32 **não tem links** para arquivos, **tampouco sistemas de arquivos montados** — esses são conceitos tipicamente UNIX. E a Win32 também **não tem suporte a sinais** (o equivalente do `kill()`).

> ⚠️ É claro, o Windows tem um número enorme de outras chamadas que o UNIX não tem — especialmente para gerenciar janelas, figuras geométricas, texto, fontes, barras de rolagem, caixas de diálogo, menus e outros aspectos da interface gráfica (GUI). Como essas chamadas não são realmente relacionadas à função do SO em si, o Tanenbaum decide não discuti-las no livro.
> 

---

# 🔓 A Estratégia da Microsoft — Desacoplamento

Uma característica importante da WinAPI é que a Microsoft **desacoplou a interface das chamadas de sistema reais**:

```
Programa Windows
        ↓ chama
   WinAPI (interface estável)
        ↓ pode chamar
   Chamadas de sistema reais (internas, podem mudar)
```

Ao desacoplar a interface API das chamadas de sistema reais, a Microsoft retém a capacidade de **mudar as chamadas de sistema reais a qualquer tempo** — mesmo de uma versão do Windows para outra — **sem invalidar os programas existentes**. Os programas continuam usando a mesma WinAPI, mesmo que por baixo as syscalls tenham mudado completamente.

Isso é diferente do UNIX/POSIX, onde as chamadas de sistema são mais diretamente expostas e o padrão é público e imutável.

> 💡 Com o Windows, é impossível distinguir o que é uma chamada de sistema (realizada pelo núcleo) e o que é apenas uma chamada de biblioteca do espaço do usuário. Na realidade, o que é uma chamada de sistema em uma versão do Windows pode ser feito no espaço do usuário em uma versão diferente, e vice-versa.
> 

---

# ✅ Resumo do Conceito

- O Windows usa um modelo **orientado a eventos** — o programa aguarda eventos (clique, tecla, USB) e chama handlers para tratá-los. Diferente do UNIX onde o programa executa código e chama syscalls conforme necessário
- A **WinAPI** (Win32/Win64) é a interface de programação do Windows — tem **milhares** de chamadas, muito mais que as ~100 do POSIX
- `CreateProcess` faz o equivalente de `fork + execve` em uma única chamada — o Windows não separa criação de processo de execução de programa
- O Windows **não tem hierarquia de processos** — após a criação, criador e criatura são iguais
- Win32 **não suporta** links de arquivos, montagem de sistemas de arquivos (`mount`) nem sinais (`kill`)
- A Microsoft **desacopla a WinAPI das syscalls reais** — pode mudar as syscalls internamente entre versões sem quebrar os programas existentes, o que é uma vantagem estratégica significativa