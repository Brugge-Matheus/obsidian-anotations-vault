---
tags:
  - sistemas-operacionais
  - so/virtualização
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.7.5"
---
# A Origem das Máquinas Virtuais

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.7.5

---

# 💡 O Problema Original — OS/360 e o Timesharing

As primeiras versões do OS/360 eram estritamente sistemas em lote (*batch*). No entanto, muitos usuários do 360 queriam poder trabalhar interativamente em um terminal — tanto dentro quanto fora da IBM — e decidiram escrever sistemas de compartilhamento de tempo para ele.

O sistema de compartilhamento de tempo oficial da IBM, o **TSS/360**, foi lançado tarde, e quando enfim chegou, era tão grande e lento que poucos locais se converteram a ele. Foi finalmente abandonado após um desenvolvimento que consumiu algo em torno de **US$ 50 milhões** — um fracasso monumental para a época.

Mas um grupo no **Centro Científico da IBM em Cambridge, Massachusetts**, produziu um sistema radicalmente diferente que a IBM por fim aceitou como produto — o **VM/370**.

---

# 🔑 A Observação Astuta — Duas Funções Separadas

O VM/370 foi baseado em uma observação astuta: um sistema de compartilhamento de tempo fornece duas coisas:

1. **Multiprogramação** — a capacidade de executar múltiplos processos ao mesmo tempo
2. **Uma máquina estendida** — uma interface mais conveniente do que apenas o hardware bruto (*bare*)

> 💡 **A essência do VM/370 é separar completamente essas duas funções.** Em vez de tentar oferecer multiprogramação e uma máquina estendida ao mesmo tempo em um único sistema complexo, o VM/370 divide o problema em duas partes independentes e muito mais simples.
> 

---

# 🏗️ A Estrutura do VM/370

O cerne do sistema, conhecido como o **monitor de máquina virtual** (*VMM — Virtual Machine Monitor*), opera diretamente no hardware e realiza a multiprogramação, fornecendo não uma, mas **várias máquinas virtuais** para a camada seguinte.

No entanto, diferentemente de todos os outros sistemas operacionais, essas máquinas virtuais **não são máquinas estendidas**, com arquivos e outros recursos interessantes. Em vez disso, elas são **cópias exatas do hardware bruto**, incluindo:

- Modos núcleo/usuário
- E/S
- Interrupções
- Tudo mais que a máquina real tem

```
Camada superior — Programas:
┌─────────┐  ┌─────────┐  ┌─────────┐
│   CMS   │  │   CMS   │  │   CMS   │  ← 370s virtuais
│(usuário)│  │(usuário)│  │(usuário)│
└────┬────┘  └────┬────┘  └────┬────┘
     │ Chamadas de sistema aqui │
     │ Captura aqui             │
┌────▼──────────────────────────▼────┐
│              VM/370                │
│   (monitor de máquina virtual)     │
└────────────────────────────────────┘
┌────────────────────────────────────┐
│         Hardware 370 bruto         │
│  Instruções de E/S aqui            │
└────────────────────────────────────┘
```

> 📌 **Figura 1.28 — A estrutura do VM/370 com CMS**
> 

Como cada máquina virtual é idêntica ao hardware original, **cada uma delas pode executar qualquer sistema operacional capaz de ser executado diretamente sobre o hardware bruto**. Máquinas virtuais diferentes podem — e frequentemente o fazem — executar diferentes sistemas operacionais.

No sistema VM/370 original da IBM, em algumas é executado o sistema operacional OS/360 ou um dos outros grandes sistemas operacionais de processamento de lotes, enquanto em outras é executado um sistema operacional monousuário interativo chamado de **CMS** (*Conversational Monitor System* — sistema monitor conversacional).

---

# 🔄 Como CMS e VM/370 Interagem

Quando um programa CMS executava uma chamada de sistema, ela era desviada para o sistema operacional **na sua própria máquina virtual**, não para o VM/370. O CMS então emitia as instruções de E/S normais de hardware para leitura do seu disco virtual — ou o que quer que fosse necessário para executar a chamada.

Essas instruções de E/S eram então **interceptadas pelo VM/370**, que as executava como parte da sua simulação do hardware real:

```
Programa CMS faz chamada de sistema
        ↓
CMS processa a chamada em sua própria máquina virtual
        ↓
CMS emite instrução de E/S de hardware (ex: ler bloco do disco)
        ↓
VM/370 intercepta a instrução
        ↓
VM/370 simula o hardware real e executa a E/S
        ↓
Resultado retorna para o CMS
        ↓
CMS retorna para o programa
```

Ao separar completamente as funções de multiprogramação e de provisão de uma máquina estendida, **cada uma das partes podia ser muito mais simples, mais flexível e muito mais fácil de manter**.

---

# 📈 O VM/370 Moderno — z/VM

Em sua versão moderna, o **z/VM** é normalmente usado para executar sistemas operacionais completos em vez de sistemas monousuário despojados como o CMS. Por exemplo, o **zSeries** é capaz de executar uma ou mais máquinas virtuais Linux junto com sistemas operacionais IBM tradicionais.

Um descendente linear do VM/370, o **z/VM** é hoje amplamente usado nos computadores de grande porte da IBM, os **zSeries** — muito utilizados em grandes centros de processamento de dados corporativos, como:

- Servidores de comércio eletrônico que lidam com centenas ou milhares de transações por segundo
- Bancos de dados cujos tamanhos chegam a milhões de gigabytes

---

# 🔄 Máquinas Virtuais Redescobertas

Embora a IBM tenha um produto de máquina virtual disponível há quatro décadas, a ideia da virtualização foi em grande parte **ignorada no mundo dos PCs** até há pouco tempo.

Duas necessidades impulsionaram o ressurgimento:

**Necessidade 1 — Consolidação de servidores:** Muitas empresas tradicionais tinham seus próprios servidores de correio, web, FTP e outros em computadores separados — às vezes com sistemas operacionais diferentes. Elas viram a virtualização como uma maneira de executar todos eles na mesma máquina **sem correr o risco de um travamento em um servidor derrubar todo o restante**.

**Necessidade 2 — Hospedagem web:** A virtualização também é popular no mundo da hospedagem na web. Sem ela, os clientes de hospedagem são obrigados a escolher entre:

- **Hospedagem compartilhada** — conta de acesso a um servidor web, mas nenhum controle sobre ele
- **Servidor dedicado** — controle total, mas custo muito maior

Com virtualização, surge uma terceira opção: cada cliente recebe sua própria máquina virtual com controle total, mas múltiplos clientes compartilham o mesmo hardware físico.

---

# ✅ Resumo do Conceito

- O **VM/370** nasceu da observação de que um sistema de timesharing fornece duas funções distintas — multiprogramação e máquina estendida — e que separar essas duas funções torna cada uma muito mais simples
- O **monitor de máquina virtual (VMM)** roda diretamente no hardware e fornece múltiplas **cópias exatas do hardware bruto** — não máquinas estendidas, mas hardware idêntico ao real
- Cada máquina virtual pode executar **qualquer SO** que rodaria no hardware físico — máquinas diferentes podem rodar SOs diferentes simultaneamente
- Quando um programa executa instruções de E/S, o VMM **intercepta** e as simula no hardware real — transparentemente
- O **z/VM** é o descendente moderno, amplamente usado em mainframes IBM (zSeries) para consolidação de cargas de trabalho
- A virtualização foi redescoberta no mundo PC por duas necessidades: **consolidação de servidores** (rodar múltiplos SOs numa máquina sem risco de interferência) e **hospedagem web** (dar controle total a clientes sem dedicar hardware exclusivo)