---
tags:
  - redes
  - redes/ipv4
---

# Máscaras de rede PADRÃO

**A Máscara de Rede Padrão** (também chamada de *Máscara Natural*) é o "filtro" que acompanha as classes de IP que acabamos de ver.

A função da máscara é dizer ao computador quais bits do endereço IP pertencem à **Rede** e quais pertencem ao **Host**.

---

## O que é a Máscara de Rede?

A máscara é um número de 32 bits (assim como o IP) composto por uma sequência de números `1` seguidos por uma sequência de números `0`.

- Onde houver o número **1** (em binário) ou **255** (em decimal), ali está a **Rede**.
- Onde houver o número **0**, ali está o **Host**.

Uma máscara de rede é, portanto, sempre N bits 1 consecutivos seguidos de (32-N) bits 0 — nunca com bits misturados. As máscaras "naturais" /8, /16 e /24 são apenas blocos de 8, 16 ou 24 bits 1 seguidos de zeros.

## As Máscaras Padrão por Classe

No modelo *Classful*, cada classe possui uma máscara "de fábrica" que nunca muda:

### 1. Classe A (Padrão)

- **Endereço IP**: `10.0.0.10`
- **Máscara Decimal:** `255.0.0.0`
- **Máscara Binária:** `11111111 . 00000000 . 00000000 . 00000000`
- **Notação CIDR:** `/8` (Significa que os primeiros 8 bits são a rede).
- **Significado:** Apenas o primeiro octeto é rede. Os outros 24 bits (3 octetos) estão livres para endereçar hosts.

### 2. Classe B (Padrão)

- **Endereço IP**: `128.12.5.103`
- **Máscara Decimal:** `255.255.0.0`
- **Máscara Binária:** `11111111 . 11111111 . 00000000 . 00000000`
- **Notação CIDR:** `/16` (Os primeiros 16 bits são a rede).
- **Significado:** Os dois primeiros octetos são rede. Os outros 16 bits (2 octetos) são para hosts.

### 3. Classe C (Padrão)

- **Endereço IP**: `192.168.1.10`
- **Máscara Decimal:** `255.255.255.0`
- **Máscara Binária:** `11111111 . 11111111 . 11111111 . 00000000`
- **Notação CIDR:** `/24` (Os primeiros 24 bits são a rede).
- **Significado:** Os três primeiros octetos são rede. Apenas o último octeto (8 bits) é para hosts.

---

## Como a Máscara funciona na prática? (A lógica AND)

O computador realiza uma operação matemática binária chamada **AND** entre o endereço IP e a Máscara para descobrir o **Endereço de Rede**.

**Exemplo com IP de Loopback:**

- **IP:** `127.0.0.1`
- **Máscara Padrão (Classe A):** `255.0.0.0`

| Componente          | Valor Decimal | Valor Binário                               |
| ------------------- | ------------- | ------------------------------------------- |
| **Endereço IP**     | `127.0.0.1`   | `01111111 . 00000000 . 00000000 . 00000001` |
| **Máscara**         | `255.0.0.0`   | `11111111 . 00000000 . 00000000 . 00000000` |
| **Resultado (Rede)**| **127.0.0.0** | `01111111 . 00000000 . 00000000 . 00000000` |

---

## Resumo:

- **Máscara Padrão:** É a máscara "natural" de uma classe de IP.
- **Função:** Separar o ID de Rede do ID de Host.
- **Regra Visual:** Onde tem `255` é rede, onde tem `0` é host.
- **Notação:** Pode ser escrita em decimal (`255.255.255.0`) ou simplificada com a barra (`/24`).

### Por que 255 e 0?

`255` em decimal é `11111111` em binário — todos os 8 bits ligados. `0` em decimal é `00000000` — todos desligados. O AND com 255 preserva todos os bits; o AND com 0 zera tudo. Por isso, nas máscaras padrão, os octetos são sempre 255 ou 0 — sem valores intermediários (ao contrário das máscaras CIDR).

---

## Conexão com Sistemas Operacionais

- **[[Bits e Bytes]]** — Uma máscara de rede é sempre N bits 1 seguidos de (32-N) bits 0. As máscaras naturais /8, /16 e /24 são blocos de 8, 16 ou 24 bits 1 — potências de 2 alinhadas em octetos. O valor 255 (`11111111`) preserva bits no AND; o valor 0 (`00000000`) zera bits. Pura aritmética binária.
- **[[Processadores]]** — O AND entre IP e máscara é executado pelo processador em uma única instrução de 32 bits. Roteadores fazem isso em hardware para cada pacote — a CPU executa uma instrução AND por endereço para determinar a qual rede ele pertence.

## Conexão com Go

- **[[Bits e Bytes]]** — As máscaras padrão são representadas em Go como `net.IPMask`: Classe A = `net.IPMask{255, 0, 0, 0}`, Classe B = `net.IPMask{255, 255, 0, 0}`, Classe C = `net.IPMask{255, 255, 255, 0}`.
- **[[Tipos de Dados]]** — `net.CIDRMask(8, 32)` retorna a máscara de Classe A; `net.CIDRMask(16, 32)` retorna a de Classe B; `net.CIDRMask(24, 32)` retorna a de Classe C. Internamente, são bytes onde cada bit 1 marca a parte de rede.
- **[[Operações Aritméticas]]** — `ip.Mask(net.CIDRMask(24, 32))` realiza o AND entre IP e máscara, retornando o endereço de rede.
