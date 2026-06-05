---
tags:
  - sistemas-operacionais
  - so/conceitos
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.6 (detalhado no Cap. 3)"
---
# Memória Virtual

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.6 (detalhado no Cap. 3)

---

# 🧠 O Problema que a Memória Virtual Resolve

Nos primeiros computadores, o tamanho de um programa era limitado pela quantidade de memória RAM disponível. Se o programa fosse maior que a RAM, simplesmente não rodava.

Mas e quando um processo precisa de mais memória do que a RAM disponível? E quando múltiplos processos juntos precisam de mais memória do que existe fisicamente?

A **memória virtual** resolve esses dois problemas ao criar a ilusão de que cada processo tem acesso a uma quantidade de memória muito maior do que a RAM física disponível.

---

# 💡 A Ideia Central

A memória virtual funciona sobre um princípio simples observado na prática:

> **Um programa raramente usa toda a sua memória ao mesmo tempo.** A maioria dos programas tem partes que são usadas frequentemente e partes que raramente são acessadas.
> 

Então em vez de carregar o programa inteiro na RAM, o SO mantém na RAM apenas as partes **ativamente usadas**, e guarda o restante no **disco ou SSD**. Quando o programa precisar de uma parte que está no disco, o SO a traz para a RAM — possivelmente movendo outra parte menos usada para o disco no processo.

---

# 🏗️ Como Funciona — Paginação

A memória virtual é implementada dividindo tanto o espaço de endereçamento virtual quanto a RAM física em blocos de tamanho fixo chamados **páginas**. Tipicamente cada página tem **4 KB**.

```
Espaço de endereçamento virtual do processo (ex: 8 GB):
┌──────────────┐
│  Página 0    │ → na RAM (sendo usada agora)
│  Página 1    │ → na RAM (sendo usada agora)
│  Página 2    │ → no disco (não usada recentemente)
│  Página 3    │ → no disco (não usada recentemente)
│  Página 4    │ → na RAM (sendo usada agora)
│  ...         │
│  Página N    │ → no disco
└──────────────┘

RAM física (ex: 4 GB):
┌──────────────┐
│  Página 0    │ ← do processo A
│  Página 1    │ ← do processo A
│  Página 4    │ ← do processo A
│  Página 7    │ ← do processo B
│  Página 2    │ ← do processo B
│  ...         │
└──────────────┘

Disco/SSD (área de swap):
┌──────────────┐
│  Página 2    │ ← do processo A (inativa)
│  Página 3    │ ← do processo A (inativa)
│  ...         │
└──────────────┘
```

---

# 🗺️ Tabelas de Páginas e a MMU

Cada processo tem uma **tabela de páginas** — uma estrutura mantida pelo SO que mapeia cada página virtual para seu local real (na RAM ou no disco):

```
Tabela de páginas do Processo A:

Página Virtual | Localização    | Endereço físico
─────────────────────────────────────────────────
Página 0       | RAM            | Frame 3 (0x3000)
Página 1       | RAM            | Frame 7 (0x7000)
Página 2       | Disco (swap)   | bloco 1024
Página 3       | Disco (swap)   | bloco 2048
Página 4       | RAM            | Frame 1 (0x1000)
```

A **MMU** (*Memory Management Unit*) é o hardware dentro da CPU que faz a tradução de endereços virtuais para físicos consultando a tabela de páginas em tempo real, a cada acesso à memória:

```
Processo acessa endereço virtual 0x1234
         ↓
MMU consulta tabela de páginas
         ↓
Página 1 → está na RAM, Frame 7
         ↓
MMU traduz: endereço físico = 0x7234
         ↓
CPU acessa a RAM no endereço físico 0x7234
```

Tudo isso acontece de forma completamente transparente para o programa — ele nunca sabe que está usando endereços virtuais.

---

# ⚡ Page Fault — Quando a Página Não Está na RAM

Quando o processo acessa uma página que está no disco e não na RAM, a MMU não consegue fazer a tradução — isso gera um **page fault** (falta de página):

```
Processo acessa endereço virtual na Página 2
         ↓
MMU consulta tabela de páginas
         ↓
Página 2 → está no DISCO (não na RAM!)
         ↓
MMU gera um PAGE FAULT (trap para o SO)
         ↓
SO pausa o processo
         ↓
SO escolhe uma página da RAM para remover
(geralmente a menos usada recentemente — LRU)
         ↓
SO salva essa página no disco (se foi modificada)
         ↓
SO carrega a Página 2 do disco para a RAM
         ↓
SO atualiza a tabela de páginas
         ↓
SO retoma o processo — que continua sem saber o que aconteceu
```

Page faults são operações caras — acessar o disco é ordens de magnitude mais lento que acessar a RAM. Por isso o SO tenta minimizá-los escolhendo bem quais páginas manter na RAM.

---

# 🔄 Swapping — O Disco como Extensão da RAM

A área do disco usada para guardar páginas que não cabem na RAM é chamada de **área de swap** (ou partição de swap no Linux, arquivo de paginação no Windows).

## O objetivo do SO é sempre manter o máximo possível na RAM

O page fault é o mecanismo de **último recurso**, não a operação normal. O SO trabalha com base no **princípio da localidade** — observado empiricamente em praticamente todos os programas:

- **Localidade temporal** — se um dado foi acessado agora, provavelmente será acessado novamente em breve
- **Localidade espacial** — se um dado foi acessado, os dados próximos a ele provavelmente também serão

O page fault só acontece em dois cenários:

1. **Primeiro acesso** a uma página que nunca foi carregada
2. **Página foi removida** da RAM porque não estava sendo usada e a RAM estava cheia

```
Processo acessa uma página
        ↓
Página já está na RAM?
   ├── SIM → MMU traduz → acesso direto (rápido — caminho feliz)
   └── NÃO → page fault → SO intervem
              ↓
         RAM tem espaço livre?
            ├── SIM → traz do disco (swap in)
            └── NÃO → remove página menos usada (LRU)
                       traz a página necessária (swap in)
```

## Algoritmo LRU — Decidindo o que fica na RAM

Quando a RAM está cheia e precisa liberar espaço, o SO usa o algoritmo **LRU** (*Least Recently Used* — menos usada recentemente) — remove a página que foi acessada há mais tempo:

```
RAM com 4 frames:
Frame 1: Página A  ← usada há pouco
Frame 2: Página B  ← usada há pouco
Frame 3: Página C  ← usada há pouco
Frame 4: Página D  ← usada há mais tempo → candidata a ser removida
```

## Thrashing — Quando o Page Fault Vira Rotina

Quando a RAM está tão cheia que o SO fica **constantemente** removendo e recarregando páginas, o sistema entra em **thrashing**. A CPU passa mais tempo gerenciando páginas do que executando código:

```
Situação normal:
99% do tempo → acesso direto à RAM (rápido)
1% do tempo  → page fault → disco (lento, mas raro)

Thrashing:
50% do tempo → page fault → disco (lento e frequente)
→ sistema quase para de funcionar
```

Isso é exatamente o que acontece quando o computador tem pouca RAM e você abre muitos programas — o HD/SSD começa a trabalhar constantemente e tudo fica lento. A solução é adicionar mais RAM ou encerrar processos.

---

# 🛡️ Memória Virtual e Isolamento entre Processos

A memória virtual não serve só para aumentar a memória disponível — ela é também um mecanismo fundamental de **isolamento e segurança**:

Cada processo tem sua **própria tabela de páginas**, mapeando seu espaço virtual para frames físicos diferentes. O processo A não tem como acessar os frames do processo B — a MMU simplesmente não tem esse mapeamento na tabela de páginas do processo A:

```
Processo A                    Processo B
Pág. virtual 0 → Frame 3      Pág. virtual 0 → Frame 8
Pág. virtual 1 → Frame 7      Pág. virtual 1 → Frame 2
Pág. virtual 2 → Frame 1      Pág. virtual 2 → Frame 5

Processo A não tem como chegar ao Frame 8 (do processo B)
→ qualquer tentativa gera page fault → SO encerra o processo
```

---

# 📐 Tamanho do Espaço Virtual vs. RAM

Em sistemas de 64 bits, cada processo enxerga um espaço de endereçamento virtual enorme:

```
2⁶⁴ = ~18 exabytes de espaço virtual por processo
RAM física típica = 8 GB a 64 GB

→ O espaço virtual é astronomicamente maior que a RAM
→ A memória virtual permite essa ilusão funcionar
→ Cada processo acha que tem a memória toda para si
```

Na prática, processadores modernos usam apenas 48 bits para endereçamento físico (256 TB) — ainda assim muito mais que qualquer RAM disponível hoje.

---

# ✅ Resumo do Conceito

- A **memória virtual** cria a ilusão de que cada processo tem muito mais memória do que a RAM física disponível
- É implementada via **paginação** — espaço virtual e RAM divididos em páginas de tamanho fixo (tipicamente 4 KB)
- A **tabela de páginas** mapeia cada página virtual para seu local real — na RAM ou no disco
- A **MMU** faz a tradução de endereços virtuais para físicos em hardware, em tempo real e de forma transparente
- **Page fault** ocorre quando uma página acessada está no disco — o SO a traz para a RAM, possivelmente movendo outra para o disco
- A área do disco usada para guardar páginas é o **swap** — quando muito usado causa **thrashing**
- Além de ampliar a memória disponível, a memória virtual garante **isolamento completo** entre processos — cada um tem sua própria tabela de páginas e não acessa a memória dos outros