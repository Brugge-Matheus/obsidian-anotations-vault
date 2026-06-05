---
tags:
  - sistemas-operacionais
  - so/estrutura
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.7.2"
---
# Sistema de Camadas

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.7.2

---

# 🏛️ O que é um Sistema de Camadas?

Uma generalização da abordagem monolítica é organizar o sistema operacional como uma **hierarquia de camadas**, cada uma construída sobre a camada abaixo dela. A ideia central é simples:

> **Cada camada só pode usar serviços das camadas inferiores — nunca das superiores.**
> 

Isso cria um sistema com fronteiras bem definidas, onde cada camada esconde os detalhes de implementação das camadas acima, oferecendo apenas uma interface limpa.

---

# 📜 O Sistema THE — O Primeiro Sistema em Camadas

O primeiro sistema construído dessa maneira foi o sistema **THE**, desenvolvido na *Technische Hogeschool Eindhoven* na Holanda por **E. W. Dijkstra** (1968) e seus alunos.

O sistema THE era um sistema em lote simples para um computador holandês — o **Electrologica X8** — que tinha 32K palavras de 27 bits (os bits eram caros na época).

O sistema tinha **seis camadas**, como mostrado na Figura 1.25:

> 📌 **Figura 1.25 — Estrutura do sistema operacional THE**
> 

```
┌─────────────────────────────────────────────────┐
│  Camada 5 — O operador                          │
│  (interface com o usuário humano do sistema)    │
├─────────────────────────────────────────────────┤
│  Camada 4 — Programas de usuário                │
│  (aplicações dos usuários)                      │
├─────────────────────────────────────────────────┤
│  Camada 3 — Gerenciamento de entrada/saída      │
│  (dispositivos de E/S abstratos e acessíveis)   │
├─────────────────────────────────────────────────┤
│  Camada 2 — Comunicação operador-processo       │
│  (console virtual para cada processo)           │
├─────────────────────────────────────────────────┤
│  Camada 1 — Gerenciamento de memória e tambor   │
│  (alocação de espaço na RAM e no tambor)        │
├─────────────────────────────────────────────────┤
│  Camada 0 — Alocação do processador e           │
│             multiprogramação                    │
│  (chaveamento de processos, interrupções,       │
│   temporizadores)                               │
└─────────────────────────────────────────────────┘
```

## O que cada camada fazia

**Camada 0** — lidava com a alocação do processador, realizando o chaveamento de processos quando ocorriam interrupções ou quando os temporizadores expiravam. Acima da camada 0, o sistema consistia em processos sequenciais, e cada um deles podia ser programado sem precisar se preocupar com o fato de que múltiplos processos estavam sendo executados em um único processador. Em outras palavras, **a camada 0 fornecia a multiprogramação básica da CPU**.

**Camada 1** — realizava o gerenciamento de memória. Ela alocava espaço para processos na memória principal e em um **tambor magnético** de 512K palavras usado para armazenar partes de processos (páginas) para as quais não havia espaço na memória principal. Acima da camada 1, os processos não precisavam se preocupar se estavam na memória ou no tambor magnético — o software da camada 1 certificava-se de que as páginas fossem trazidas à memória no momento em que eram necessárias e removidas quando não eram mais. O tambor magnético era o equivalente do SSD/disco de swap da época.

**Camada 2** — encarregava-se da comunicação entre cada processo e o console do operador (o usuário). Acima dessa camada, cada processo efetivamente tinha o seu próprio console de operação — ou seja, a camada 2 criava uma **abstração de console virtual** para cada processo.

**Camada 3** — encarregava-se do gerenciamento dos dispositivos de E/S e armazenava temporariamente os fluxos de informação que iam ou vinham desses dispositivos. Acima da camada 3, cada processo podia lidar com **dispositivos de E/S abstratos** e mais acessíveis, em vez de dispositivos reais com muitas peculiaridades.

**Camada 4** — era onde se encontravam os programas dos usuários. Eles não precisavam se preocupar com o gerenciamento de processos, memória, console ou E/S — tudo isso já estava resolvido pelas camadas inferiores.

**Camada 5** — o processo operador do sistema estava localizado aqui — a interface com o ser humano que operava o computador.

---

# 🔵 O Sistema MULTICS — Anéis Concêntricos

Outra generalização do conceito de camadas estava presente no sistema **MULTICS**. Em vez de camadas, o MULTICS foi descrito como tendo uma série de **anéis concêntricos**, com os anéis internos sendo mais privilegiados que os externos — o que é efetivamente a mesma ideia das camadas, mas implementada como proteção de hardware.

```
┌─────────────────────────────────┐
│    Anel 0 (mais privilegiado)   │ ← kernel
│   ┌─────────────────────────┐   │
│   │       Anel 1            │   │ ← serviços do SO
│   │   ┌─────────────────┐   │   │
│   │   │     Anel 2      │   │   │ ← subsistemas
│   │   │  ┌───────────┐  │   │   │
│   │   │  │  Anel N   │  │   │   │ ← programas de usuário
│   │   │  └───────────┘  │   │   │   (menos privilegiado)
│   │   └─────────────────┘   │   │
│   └─────────────────────────┘   │
└─────────────────────────────────┘
```

Quando um procedimento em um anel exterior queria chamar um procedimento em um anel interior, ele tinha de fazer o equivalente a uma chamada de sistema — uma instrução de captura (**TRAP**), cujos parâmetros eram cuidadosamente conferidos por sua validade antes de a chamada ter permissão para prosseguir.

Embora o sistema operacional inteiro fosse parte do espaço de endereços de cada processo de usuário no MULTICS, o **hardware** tornou possível que se designassem rotinas individuais (segmentos de memória, na realidade) como protegidos contra leitura, escrita ou execução.

A **vantagem do mecanismo de anéis** é que ele pode ser facilmente estendido para abranger subsistemas de usuário. Por exemplo, um professor poderia escrever um programa para testar e atribuir notas a programas de estudantes executando-o no anel *n*, com os programas dos estudantes sendo executados no anel *n* + 1, de maneira que eles não pudessem mudar suas notas.

> 💡 Enquanto o esquema de camadas THE era apenas um suporte para o projeto (pois em última análise todas as camadas estavam unidas em um único programa executável), em MULTICS o mecanismo de anéis estava bastante presente no momento de execução e era imposto pelo hardware.
> 

---

# ⚖️ Vantagens e Desvantagens do Sistema de Camadas

|  | Sistemas de Camadas |
| --- | --- |
| ✅ **Modularidade** | Cada camada esconde seus detalhes — a camada acima só precisa conhecer a interface |
| ✅ **Verificabilidade** | Mais fácil de provar a correção de cada camada individualmente |
| ✅ **Manutenibilidade** | Mudanças em uma camada não afetam as outras, desde que a interface seja mantida |
| ❌ **Performance** | Cada chamada entre camadas tem overhead — atravessar várias camadas é mais lento que uma chamada direta no monolítico |
| ❌ **Dificuldade de definição** | Nem sempre é óbvio como dividir os componentes em camadas estritamente ordenadas |

---

# 🌍 Legado e Influência

O sistema THE foi mais importante como **conceito** do que como produto — ele demonstrou que era possível estruturar um SO de forma hierárquica e verificável. Dijkstra ficou famoso por esse trabalho e por suas contribuições à programação estruturada.

O mecanismo de anéis do MULTICS influenciou diretamente a arquitetura de proteção do **Intel x86** — que até hoje tem 4 anéis de proteção (ring 0 a ring 3), onde ring 0 é o kernel e ring 3 são os programas de usuário.

```
Anéis de proteção x86 modernos:
Ring 0 → Kernel do SO              (mais privilegiado)
Ring 1 → Drivers de dispositivos   (raramente usado na prática)
Ring 2 → Drivers de dispositivos   (raramente usado na prática)
Ring 3 → Programas de usuário      (menos privilegiado)
```

> 💡 **Por que rings 1 e 2 são raramente usados?** O Intel projetou os rings 1 e 2 originalmente para que drivers de dispositivos e serviços do SO que precisassem de mais privilégio que programas comuns, mas menos que o kernel, pudessem rodar de forma isolada. Na prática, porém, os SOs modernos — Linux, Windows, macOS — adotaram um modelo simplificado de **apenas dois níveis efetivos**: ring 0 (kernel) e ring 3 (usuário). Drivers de dispositivos no Linux e Windows rodam majoritariamente no ring 0 junto com o kernel, por simplicidade e performance. O resultado é que rings 1 e 2 existem no hardware mas ficam vazios na maioria dos sistemas.
> 

---

# ✅ Resumo do Conceito

- **Sistemas de camadas** organizam o SO em uma hierarquia onde cada camada só pode usar serviços das camadas abaixo — criando fronteiras bem definidas e ocultação de informação
- O primeiro sistema de camadas foi o **THE** (Dijkstra, 1968) com **6 camadas** — da multiprogramação básica na camada 0 até o operador humano na camada 5
- Cada camada abstraía os detalhes das inferiores — processos na camada 4 não precisavam se preocupar com E/S, memória, CPU ou console
- O **MULTICS** generalizou o conceito com **anéis concêntricos** — mais privilegiados por dentro, menos por fora — implementados em hardware com instrução TRAP para transitar entre anéis
- O mecanismo de anéis do MULTICS influenciou diretamente os **anéis de proteção x86** usados até hoje
- A principal vantagem é a **modularidade e verificabilidade** — a principal desvantagem é o **overhead de performance** ao atravessar várias camadas