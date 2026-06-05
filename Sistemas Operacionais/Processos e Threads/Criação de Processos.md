---
tags:
  - sistemas-operacionais
  - so/processos-e-threads
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 2 — Seção 2.1.2"
---
# Criação de Processos

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 2 — Seção 2.1.2

---

# ➕ 2.1.2 — Criação de Processos

Sistemas operacionais precisam ter alguma maneira de criar processos. Em sistemas muito simples, ou em sistemas projetados para executar apenas uma única aplicação (p. ex., o controlador em um forno micro-ondas), pode ser possível ter todos os processos que serão em algum momento necessários quando o sistema for ligado. Em sistemas para fins gerais, no entanto, algum modo é necessário para criar e terminar processos, na medida do necessário, durante a operação.

## Os quatro eventos que criam processos

Quatro eventos principais fazem os processos serem criados:

1. **Inicialização do sistema** — quando o SO é inicializado, em geral uma série de processos é criada automaticamente: alguns de primeiro plano (interagem com usuários) e outros de segundo plano (*daemons*).
2. **Execução de uma chamada de sistema de criação de processo por um processo em execução** — um processo já em execução emite uma chamada de sistema para criar um ou mais processos novos para ajudá-lo em seu trabalho.
3. **Solicitação de um usuário para criar um novo processo** — em sistemas interativos, o usuário pode iniciar um programa digitando um comando ou clicando duas vezes sobre um ícone, criando um novo processo.
4. **Início de uma tarefa em lote** — aplicável somente a sistemas em lote encontrados em grandes computadores; quando o SO decide que tem recursos para executar outra tarefa, ele cria um novo processo e executa a próxima tarefa da fila de entrada.

> ⚠️ **Em todos esses casos**, tecnicamente um novo processo é sempre criado por um processo já existente, que executa uma chamada de sistema de criação de processo. Esse processo pode ser um processo de sistema invocado do teclado, do *mouse*, ou um processo gerenciador de lotes. A chamada diz ao SO para criar um novo processo e indica, direta ou indiretamente, qual programa executar nele. Para dar início à "partida", o primeiro processo é criado quando o sistema é inicializado.
> 

---

# 🐧 Criação de Processos no UNIX — fork

No UNIX, há **apenas uma chamada de sistema** para criar um novo processo: **`fork`**.

> 💡 **fork:** chamada de sistema UNIX que cria um **clone exato** do processo chamador. Após o `fork`, os dois processos — pai e filho — têm a mesma imagem de memória, as mesmas variáveis de ambiente e os mesmos arquivos abertos. Nenhuma memória para escrita é compartilhada entre eles.
> 

O fluxo típico após um `fork` é:

```
Processo pai executa fork()
         │
         ├────────────────────────────────┐
         │                               │
   (pai continua)                 (filho criado —
         │                         clone do pai)
         │                               │
         │                        execve() ou similar
         │                               │
         │                        (carrega novo programa
         │                         na memagem do filho)
         ▼                               ▼
  pai aguarda ou                  filho executa o
  continua em paralelo            novo programa
```

- Após o `fork`, o processo-filho normalmente executa **`execve`** (ou chamada similar) para substituir sua imagem de memória por um novo programa.
- O objetivo dessa separação em duas etapas é permitir que o processo-filho **manipule seus descritores de arquivo** depois do `fork` mas antes do `execve`, a fim de conseguir o redirecionamento para entrada-padrão, saída-padrão e erro-padrão.

**Exemplo prático:** quando um usuário digita um comando no *shell* — digamos, `sort` —, o *shell* se bifurca gerando um processo-filho, e o processo-filho executa `sort`.

> 💡 **execve:** chamada de sistema que substitui a imagem de memória do processo atual pelo programa especificado. Após o `execve`, o processo continua existindo (mesmo PID), mas passa a executar um programa completamente diferente.
> 

### Espaços de endereços após o fork — UNIX

No UNIX tradicional, o espaço de endereços inicial do filho é uma **cópia** do espaço do pai — há definitivamente dois espaços distintos envolvidos. Se um dos dois processos muda uma palavra no seu espaço de endereços, a mudança **não é visível para o outro**.

Algumas implementações UNIX compartilham o **texto** (código) do programa entre pai e filho, já que o texto não pode ser modificado. Como alternativa, o filho pode compartilhar toda a memória do pai por meio do mecanismo **copy-on-write**:

> 💡 **Copy-on-write (COW) — cópia-na-escrita:** otimização em que pai e filho compartilham as mesmas páginas de memória logo após o `fork`. Sempre que qualquer um dos dois quiser **modificar** parte da memória, aquele pedaço é explicitamente copiado primeiro para uma área de memória privada. Nenhuma memória que pode ser escrita é genuinamente compartilhada — a cópia ocorre sob demanda, apenas quando necessária.
> 

---

# 🪟 Criação de Processos no Windows — CreateProcess

No Windows, em comparação, **uma única chamada de função Win32** lida tanto com a criação do processo quanto com a carga do programa correto no novo processo: **`CreateProcess`**.

> 💡 **CreateProcess:** função Win32 que cria um novo processo e já carrega o programa especificado nele. Retorna uma estrutura com informações sobre o processo recém-criado. Possui **10 parâmetros**, que incluem o programa a ser executado, os parâmetros de linha de comando, atributos de segurança, *bits* que controlam se arquivos abertos são herdados, informações sobre prioridades, a especificação da janela a ser criada e um ponteiro para uma estrutura na qual informações sobre o processo criado são retornadas.
> 

Além do `CreateProcess`, Win32 tem mais ou menos **100 outras funções** para gerenciar e sincronizar processos e tópicos relacionados.

### Espaços de endereços após CreateProcess — Windows

No Windows, os espaços de endereços do pai e do filho são **diferentes desde o início** — não há o conceito de clone seguido de `execve`. O novo processo já nasce com sua própria imagem de memória separada.

---

# 🆚 UNIX vs. Windows — Comparativo de criação de processos

| Aspecto | UNIX (`fork`  • `execve`) | Windows (`CreateProcess`) |
| --- | --- | --- |
| **Chamadas necessárias** | Duas: `fork` depois `execve` | Uma: `CreateProcess` |
| **Processo criado** | Clone exato do pai | Processo novo com programa já carregado |
| **Espaço de endereços** | Cópia do pai (com COW opcional) | Separado desde o início |
| **Herança de arquivos** | Automática após o `fork` | Controlada por parâmetro de `CreateProcess` |
| **Hierarquia de processos** | Explícita — pai/filho com PID | Sem hierarquia formal |
| **Flexibilidade pré-exec** | Alta — filho pode redirecionar I/O antes do `execve` | Limitada — controlada por parâmetros |

---

# ✅ Resumo do Conceito

- **Quatro eventos criam processos:** inicialização do sistema, chamada de sistema por processo existente, solicitação de usuário e início de tarefa em lote
- **Em todos os casos**, um novo processo é criado por um processo já existente via chamada de sistema
- **UNIX — `fork`:** cria um clone exato do pai; filho normalmente executa `execve` em seguida para carregar um novo programa; espaços de endereços são separados (com COW disponível)
- **`execve`:** substitui a imagem de memória do processo por um novo programa, mantendo o mesmo PID
- **Copy-on-write (COW):** otimização que adia a cópia da memória para o momento em que pai ou filho tentar escrever — evita cópias desnecessárias logo após o `fork`
- **Windows — `CreateProcess`:** função única que cria o processo e carrega o programa; espaços de endereços são separados desde o início; ~100 funções Win32 adicionais para gerenciamento
- **Diferença fundamental:** no UNIX, criar e carregar são etapas separadas (flexibilidade de redirecionamento de I/O entre elas); no Windows, é uma operação única