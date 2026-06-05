---
tags:
  - redes
  - redes/osi
---

# PDU (Protocol Data Unit)

O conceito de **PDU (Protocol Data Unit)** é essencial para entender como os dados são tratados e nomeados em cada camada do Modelo OSI. É como se cada camada "embrulhasse" o dado de uma forma diferente, dando a ele um nome específico.

---

## 1. PDU (Protocol Data Unit) — A Unidade de Dado de Cada Camada

**Definição:** PDU é a **menor unidade de dados** que uma camada pode manipular. Cada camada do Modelo OSI (ou TCP/IP) tem um nome próprio para sua PDU, e esse nome muda conforme o dado "desce" pelas camadas.

---

## 2. Como o PDU Evolui nas Camadas

À medida que os dados descem do topo até a base, cada camada adiciona seu próprio cabeçalho (e às vezes rodapé) ao PDU da camada superior. Isso é chamado de **Encapsulamento**.

Veja como isso funciona no **Modelo OSI**:

| Camada OSI | Nome da PDU | O que é adicionado? |
| --- | --- | --- |
| **7 - Aplicação** | **Dados** | Conteúdo original (mensagem, arquivo, etc.) |
| **6 - Apresentação** | **Dados** | Formatação, criptografia |
| **5 - Sessão** | **Dados** | Informações de sessão |
| **4 - Transporte** | **Segmento** (TCP) ou **Datagrama** (UDP) | Cabeçalho TCP/UDP (portas, controle de fluxo, etc.) |
| **3 - Rede** | **Pacote** | Cabeçalho IP (endereços de origem e destino) |
| **2 - Enlace** | **Quadro (Frame)** | Cabeçalho e rodapé Ethernet (endereços MAC, CRC) |
| **1 - Física** | **Bits** | Sequência binária (0s e 1s) para transmissão |

---

## 3. Processo de Encapsulamento

```
[ Dados ] ← Camada 7
   ↓
[ Dados + Cabeçalho Sessão ] ← Camada 5
   ↓
[ Dados + Cabeçalho Transporte ] ← Camada 4 → Segmento
   ↓
[ Segmento + Cabeçalho IP ] ← Camada 3 → Pacote
   ↓
[ Pacote + Cabeçalho Ethernet ] ← Camada 2 → Quadro
   ↓
[ Quadro → Sinal Elétrico/Luz/Onda ] ← Camada 1 → Bits
```

---

## 4. Processo de Desencapsulamento (No destino)

No computador receptor, o processo é invertido:

1. **Camada 1** recebe os **bits** e os converte em quadros.
2. **Camada 2** verifica o quadro, extrai o pacote e o envia para a Camada 3.
3. **Camada 3** lê o IP, extrai o segmento e o envia para a Camada 4.
4. **Camada 4** verifica a porta, reorganiza os segmentos e entrega os dados para a Camada 5.
5. E assim sucessivamente até chegar à **Camada 7**, onde o aplicativo recebe os dados originais.

---

## 5. Dica de Memorização

Memorize esta frase:

> **"Todo Segundo, Pacotes Viajam em Quadros de Bits."**

Que corresponde a:

- **T**ransporte → **S**egmento
- **R**ede → **P**acote
- **E**nlace → **Q**uadro
- **F**ísica → **B**its

---

## 6. Resumo

> **PDU é o nome técnico do "pacote" em cada camada.** À medida que o dado desce, ele é encapsulado com novos cabeçalhos, e seu nome muda para refletir a camada em que está sendo tratado.

---

## Conexão com Sistemas Operacionais

Os nomes de PDU resumem a hierarquia em uma linha: **Dados (L7) → Segmento/Datagrama (L4) → Pacote (L3) → Quadro/Frame (L2) → Bits (L1)**.

Do ponto de vista do sistema operacional, cada PDU é simplesmente um **buffer em RAM** — uma região de memória com um cursor indicando onde o cabeçalho da camada atual começa. O encapsulamento não copia dados: apenas move o cursor para um offset menor (adicionando espaço para o cabeçalho) ou maior (removendo o cabeçalho ao desencapsular) → [[Memória]].

No kernel Linux, a estrutura `struct sk_buff` (socket buffer) representa exatamente isso: **uma única alocação de memória** com vários ponteiros (`data`, `mac_header`, `network_header`, `transport_header`) apontando para o início do cabeçalho de cada camada dentro do mesmo buffer. Isso elimina cópias de memória entre as camadas — o encapsulamento é apenas uma atualização de ponteiro → [[Memória]], [[System Calls]].

As funções `skb_push()` e `skb_pull()` do kernel Linux implementam esse mecanismo:
- `skb_push(skb, len)` — move o ponteiro `data` para trás (reserva espaço para adicionar um cabeçalho).
- `skb_pull(skb, len)` — move o ponteiro `data` para frente (descarta o cabeçalho lido).

Nenhuma dessas operações copia bytes — apenas ajustam o ponteiro dentro do buffer já alocado. Isso é análogo à aritmética de ponteiros em C com `void*`, onde avançar ou recuar um ponteiro não mexe nos dados subjacentes → [[Memória Virtual]].
