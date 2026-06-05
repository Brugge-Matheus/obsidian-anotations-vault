---
tags:
  - sistemas-operacionais
  - so/conceitos
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.5.2"
---
# Espaços de Endereçamento

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.5.2

---

# 🧠 O Problema Fundamental da Memória

Todo computador tem memória principal para armazenar programas em execução. Em um SO simples, apenas um programa de cada vez está na memória — para executar um segundo, o primeiro precisa ser removido e o segundo colocado. Isso é chamado de **swapping**.

Sistemas operacionais mais sofisticados permitem que múltiplos programas estejam na memória ao mesmo tempo. Para evitar que interfiram entre si — e com o próprio SO — é preciso algum mecanismo de proteção no hardware, controlado pelo SO.

---

# 📦 O que é um Espaço de Endereçamento?

Cada processo tem um **espaço de endereçamento** — o conjunto de endereços de memória que ele pode usar. Comumente vai de 0 até algum máximo, onde o processo pode ler e escrever livremente.

O espaço de endereçamento é **desacoplado da memória física** — ele pode ser maior ou menor do que a memória RAM disponível na máquina. O SO cria essa abstração para cada processo de forma independente.

> 💡 **Analogia:** é como se cada processo acreditasse ter um apartamento inteiro só para ele — mas na prática muitos apartamentos existem no mesmo prédio (RAM), gerenciados pelo condomínio (SO + MMU). Cada morador não sabe dos outros e não consegue entrar no apartamento alheio.
> 

---

# 🗺️ O que Há Dentro do Espaço de Endereçamento

O espaço de endereçamento de um processo é organizado em regiões com funções distintas:

```
Endereço máximo ┌─────────────────────────────┐
                │         Pilha (Stack)        │ → variáveis locais,
                │      (cresce para baixo ↓)   │   parâmetros de funções,
                │                              │   endereços de retorno
                ├─────────────────────────────┤
                │         espaço livre         │ → buffer entre pilha e heap
                ├─────────────────────────────┤
                │         Heap                 │ → memória alocada
                │      (cresce para cima ↑)    │   dinamicamente
                │                              │   (malloc, new, GC)
                ├─────────────────────────────┤
                │    Dados (BSS + Data)        │ → variáveis globais
                │                              │   e estáticas
                ├─────────────────────────────┤
                │    Código (Text)             │ → instruções do programa
                │    (somente leitura)         │   (o executável)
Endereço 0      └─────────────────────────────┘
```

Cada região tem um propósito específico:

| Região | O que armazena | Tamanho |
| --- | --- | --- |
| **Código (Text)** | Instruções do programa | Fixo — definido na compilação |
| **Dados** | Variáveis globais e estáticas | Fixo — definido na compilação |
| **Heap** | Memória alocada dinamicamente | Cresce conforme o programa pede |
| **Pilha (Stack)** | Contexto das funções em execução | Varia com as chamadas de função |

---

# 🔗 Endereços Virtuais vs. Endereços Físicos

O espaço de endereçamento que o processo enxerga é **virtual** — os endereços 0x0000 até 0xFFFFFFFF não correspondem diretamente à RAM física. A **MMU** (*Memory Management Unit*) traduz cada endereço virtual em um endereço físico real na RAM:

```
Processo A pensa que está em:     MMU traduz para:       RAM física:
  endereço virtual 0x0001    →    endereço físico 0x4A20
  endereço virtual 0x0002    →    endereço físico 0x4A21

Processo B também usa 0x0001:
  endereço virtual 0x0001    →    endereço físico 0x9F10  ← lugar diferente!
```

Os dois processos usam os mesmos endereços virtuais, mas a MMU os mapeia para regiões **completamente diferentes** da RAM física. Cada processo vive em seu próprio universo de memória, sem saber da existência do outro.

---

# 📐 Tamanho do Espaço de Endereçamento

Em muitos computadores, os endereços são de 32 ou 64 bits:

- **32 bits** → espaço de endereçamento de 2³² = **4 GB** por processo
- **64 bits** → espaço de endereçamento de 2⁶⁴ = **~18 exabytes** por processo

No caso mais simples, o máximo de espaço de endereços que um processo tem é **menor do que a memória principal** — o processo pode preencher todo seu espaço de endereçamento e haverá espaço suficiente na RAM para armazená-lo inteiramente.

No entanto, quando o espaço de endereçamento do processo é **maior que a RAM disponível**, entra em cena a **memória virtual**.

---

# 🔄 Memória Virtual — Quando o Espaço é Maior que a RAM

A memória virtual permite que um processo tenha um espaço de endereçamento maior do que a memória física disponível. O SO mantém **parte do espaço de endereçamento na RAM** e **parte no SSD ou disco**, enviando trechos entre eles conforme a necessidade:

```
Espaço de endereçamento do processo (ex: 8 GB virtual):

┌─────────────────┐
│  páginas ativas  │ ← mantidas na RAM (acesso rápido)
│  (usadas agora)  │
├─────────────────┤
│  páginas inativas│ ← guardadas no SSD/disco (swap)
│  (não usadas)    │   trazidas para RAM quando necessário
└─────────────────┘

RAM física (ex: 4 GB):
┌─────────────────┐
│ páginas do proc A│
│ páginas do proc B│
│ páginas do SO    │
└─────────────────┘
```

Quando o processo precisa de uma página que está no disco, ocorre um **page fault** — o SO pausa o processo, traz a página do disco para a RAM (possivelmente jogando outra no disco), e retoma o processo. Tudo transparente para o programa.

---

# 🛡️ Proteção e Isolamento

O gerenciamento de espaços de endereçamento é uma das funções mais importantes do SO justamente por garantir **isolamento entre processos**:

- Processo A **não consegue ler nem escrever** na memória do Processo B
- Processo A **não consegue acessar** a memória do kernel do SO
- Qualquer tentativa de acesso fora do espaço mapeado gera um **trap** → o SO encerra o processo com *segmentation fault*

Esse mecanismo de proteção precisa estar no **hardware** (MMU), mas é **controlado pelo SO** — que define os mapeamentos e garante que cada processo enxergue apenas o que lhe é permitido.

> 💡 O gerenciamento de espaços de endereçamento e da memória física forma uma parte tão importante do SO que o **Capítulo 3 inteiro do Tanenbaum** é dedicado a esse assunto.
> 

---

# 🔁 Relação com o que Já Estudamos

```
MMU (hardware dentro da CPU)
    ↓ traduz endereços virtuais → físicos
SO (software)
    ↓ configura as tabelas de páginas da MMU
    ↓ decide o que fica na RAM e o que vai para o disco
    ↓ garante isolamento entre processos

Cada processo enxerga:
    ↓ um espaço de endereçamento virtual completo
    ↓ com suas próprias regiões de código, dados, heap e pilha
    ↓ sem interferência de outros processos
```

---

# ✅ Resumo do Conceito

- Em SOs simples, apenas um programa fica na memória por vez (*swapping*) — SOs modernos permitem múltiplos programas simultâneos com proteção
- Cada processo tem seu próprio **espaço de endereçamento virtual** — um conjunto de endereços que vai de 0 a um máximo, desacoplado da RAM física
- O espaço de endereçamento é dividido em regiões: **código** (fixo), **dados** (fixo), **heap** (cresce dinamicamente) e **pilha** (varia com chamadas de função)
- A **MMU** traduz endereços virtuais em físicos — processos diferentes podem usar os mesmos endereços virtuais sem conflito
- Em sistemas de 32 bits, cada processo tem até **4 GB** de espaço virtual; em 64 bits, até **~18 exabytes**
- Quando o espaço virtual é maior que a RAM, a **memória virtual** usa o disco como extensão, trazendo páginas conforme necessário (*page fault*)
- O isolamento entre processos é garantido pela MMU — acessos fora do espaço mapeado geram *segmentation fault*