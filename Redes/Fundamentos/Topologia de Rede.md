---
tags:
  - redes
  - redes/fundamentos
---

# Topologia de Rede

A **Topologia** é o "desenho" da rede, enquanto o **Método de Conexão** é como os fios (ou ondas) ligam os aparelhos.

---

### 1. Métodos de Conexão (O "Link" entre aparelhos)

Antes de desenhar a rede, definimos como dois ou mais aparelhos conversam fisicamente.

- **Ponto a Ponto (Point-to-Point):**
    - **O que é:** Um canal dedicado exclusivamente entre **dois** dispositivos.
    - **Vantagem:** Toda a velocidade do cabo é só para eles dois.
    - **Exemplo:** Um cabo ligando o Roteador ao Modem, ou seu celular pareado com um fone Bluetooth.
- **Multiponto (Multipoint):**
    - **O que é:** Três ou mais dispositivos compartilham o **mesmo meio** de transmissão.
    - **Vantagem:** Economiza cabos e infraestrutura.
    - **Exemplo:** O Wi-Fi da sua casa (todos os celulares dividem o "ar") ou o antigo cabo coaxial de redes em barramento.

---

### 2. Topologias de Rede (O "Desenho" da Organização)

A topologia define o mapa da rede. Ela determina a resiliência (se um cair, o resto continua?) e o custo.

| Topologia | Desenho Visual | Tipo de Conexão | Resumo Técnico |
| --- | --- | --- | --- |
| **Barramento (Bus)** | Uma linha central | **Multiponto** | Todos "pendurados" no mesmo cabo. Se o cabo central romper, a rede toda morre. (Obsoleta). |
| **Anel (Ring)** | Um círculo fechado | **Ponto a Ponto** | Cada PC liga no vizinho. O dado viaja em um sentido. Se um PC falhar, o anel quebra. |
| **Estrela (Star)** | Um nó central | **Ponto a Ponto** | **A mais usada hoje.** Todos ligam em um Switch central. Se um PC falhar, só ele cai. Se o Switch falhar, tudo cai. |
| **Malha (Mesh)** | Teia de aranha | **Ponto a Ponto** | Vários caminhos para o mesmo destino. Altíssima segurança (redundância). Usada na Internet e Wi-Fi Mesh. |
| **Árvore (Tree)** | Hierarquia | **Ponto a Ponto** | Várias "estrelas" conectadas entre si. Ótima para prédios com vários andares. |

---

### 3. Como esses conceitos se encaixam na prática?

No seu dia a dia, você provavelmente usa uma mistura de tudo isso:

1. **Na sua mesa:** Você tem uma conexão **Ponto a Ponto** entre seu PC e o Switch da parede.
2. **No seu escritório:** A rede inteira está organizada em uma **Topologia em Estrela** (todos os cabos vão para o mesmo rack de TI).
3. **No seu celular:** Você está em uma conexão **Multiponto** via Wi-Fi, dividindo o sinal com seus colegas.
4. **Na Internet:** Os grandes servidores do Google e Facebook estão conectados em uma **Topologia em Malha**, para garantir que o serviço nunca fique fora do ar.

---

### 4. Topologia Física vs. Lógica

> **Topologia Física vs. Lógica:**
Nem sempre o que você vê é como a rede funciona.
>
> - **Física:** É onde os cabos passam (ex: todos os cabos saindo dos PCs e indo para um Switch central = **Estrela**).
> - **Lógica:** É como o sinal viaja. Antigamente, usávamos aparelhos chamados *Hubs* que, embora parecessem uma Estrela (física), funcionavam como um **Barramento** (lógica), pois o sinal era "gritado" para todo mundo ao mesmo tempo.

---

## 5. Aprofundamento: Detalhes internos de cada topologia

**Barramento (Bus) — Ethernet legado:** todos os nós compartilham o mesmo cabo coaxial. Quando um nó transmite, o sinal elétrico se propaga para ambas as extremidades. Todos os nós "ouvem" — apenas o nó com o MAC de destino processa o quadro. Isso cria um **domínio de colisão**: se dois nós transmitem ao mesmo tempo, os sinais colidem e precisam ser retransmitidos (CSMA/CD).

**Token Ring — acesso determinístico:** em vez de concorrer pelo meio, cada nó aguarda o "token" (um quadro especial que circula pelo anel). Apenas quem tem o token pode transmitir. Isso elimina colisões, mas adiciona latência variável. A topologia é circular — a falha de um nó quebra o anel inteiro (a menos que haja bypass automático).

**Estrela (modern Ethernet):** todos os cabos vão até um switch. O switch tem um **backplane** interno — um barramento de alta velocidade que conecta todos os chips de porta. Internamente, o switch é como um barramento, mas cada porta tem buffer e o switching fabric encaminha quadros sem colisão. A estrela física esconde um barramento lógico de alta velocidade no chip.

**Árvore/Hierárquica (redes corporativas):** estrutura em 3 camadas:
- **Core** (núcleo): switches de altíssima velocidade, sem configurações complexas, só encaminham tráfego rapidamente.
- **Distribution** (distribuição): aplica políticas, roteamento entre VLANs, ACLs.
- **Access** (acesso): switches que conectam os hosts finais (PCs, impressoras).

**Malha Completa (Full Mesh):** cada nó conecta diretamente a todos os outros. Número de links = `n*(n-1)/2` — para 10 nós, são 45 links. Extremamente caro e difícil de gerenciar, mas máxima resiliência. Usado em data centers de alta criticidade.

**Topologia física ≠ lógica:** um switch moderno tem topologia física em estrela, mas internamente opera como barramento de alta velocidade no backplane. Um hub tinha topologia física em estrela, mas operava logicamente como barramento (broadcast para todas as portas).

---

## Conexão com Sistemas Operacionais

- **NIC e domínios de colisão:** o kernel precisa gerenciar a NIC, que recebe ou descarta quadros dependendo do MAC de destino → [[Dispositivos de IO]].
- **Backplane do switch:** é análogo a um [[Barramentos]] interno de alta velocidade — a mesma ideia de barramento compartilhado que conecta CPU, memória e periféricos dentro do computador.
- **Topologia física vs. lógica:** análogo à distinção entre hardware físico e abstração do SO — [[Dispositivos de IO]] fornece uma interface uniforme independente da topologia física subjacente.
