---
tags:
  - redes
  - redes/ipv4
---

# Converter máscaras CIDR em Padrão e vice-versa

Converter uma **Máscara CIDR** (ex: `/25`) para uma **Máscara Padrão Decimal** (ex: `255.255.255.128`) é um processo de preencher 32 espaços com o número de bits "1" indicados pela barra.

Aqui estão os dois métodos para fazer essa conversão:

---

## Método 1: O Método dos Octetos (Mais Rápido)

Como cada octeto do IPv4 tem exatamente **8 bits**, você pode converter a barra rapidamente dividindo por 8:

1. **Divida o número CIDR por 8.**
2. O resultado inteiro indica quantos octetos serão **255**.
3. O resto da divisão indica quantos bits "1" sobraram para o próximo octeto.
4. O que sobrar depois disso será **0**.

### Exemplo: Converter `/20`

- $20 \div 8 = \mathbf{2}$ (com resto **4**).
- Isso significa: **Dois** octetos cheios (`255.255`).
- O terceiro octeto tem **4 bits** ligados (`11110000` em binário).
- O quarto octeto é zero (`0`).
- **Resultado:** `255.255.240.0`

---

## Método 2: A Tabela de Pesos do Último Octeto

Para converter o "resto" de bits que sobrou no octeto incompleto, use esta tabela de soma acumulada (sempre da esquerda para a direita):

| Bits Ligados | Binário      | Valor Decimal         |
| ------------ | ------------ | --------------------- |
| **1 bit**    | `10000000`   | **128**               |
| **2 bits**   | `11000000`   | **192** ($128+64$)    |
| **3 bits**   | `11100000`   | **224** ($192+32$)    |
| **4 bits**   | `11110000`   | **240** ($224+16$)    |
| **5 bits**   | `11111000`   | **248** ($240+8$)     |
| **6 bits**   | `11111100`   | **252** ($248+4$)     |
| **7 bits**   | `11111110`   | **254** ($252+2$)     |
| **8 bits**   | `11111111`   | **255** ($254+1$)     |

---

## Exemplos Práticos

### Exemplo A: Converter `/25`

- Temos 25 bits "1".
- Os primeiros 24 bits formam: `255.255.255`.
- Sobra **1 bit** para o 4º octeto.
- Olhando na tabela, 1 bit = **128**.
- **Máscara:** `255.255.255.128`

### Exemplo B: Converter `/18`

- Temos 18 bits "1".
- Os primeiros 16 bits formam: `255.255`.
- Sobram **2 bits** para o 3º octeto.
- Olhando na tabela, 2 bits = **192**.
- O 4º octeto fica zerado.
- **Máscara:** `255.255.192.0`

### Exemplo C: Converter `/30`

- Temos 30 bits "1".
- Os primeiros 24 bits formam: `255.255.255`.
- Sobram **6 bits** para o 4º octeto.
- Olhando na tabela, 6 bits = **252**.
- **Máscara:** `255.255.255.252`

---

## Resumo Visual

Você pode criar um pequeno guia de referência rápida:

- `/8` = `255.0.0.0`
- `/16` = `255.255.0.0`
- `/24` = `255.255.255.0`
- **Qualquer número entre eles:** Use a tabela de pesos para preencher o octeto "quebrado".

**Dica de Ouro:** Se o número CIDR for `/24`, `/25`, `/26`... a mudança é sempre no **4º octeto**. Se for entre `/16` e `/23`, a mudança é no **3º octeto**. Se for entre `/8` e `/15`, a mudança é no **2º octeto**.

### A matemática por trás (visão de programador)

Converter /25 para `255.255.255.128` em código é simplesmente:

```
mask = (0xFFFFFFFF << (32 - 25)) & 0xFFFFFFFF
     = (0xFFFFFFFF << 7) & 0xFFFFFFFF
     = 0xFFFFFF80
     = 255.255.255.128
```

O processo inverso (decimal para CIDR) é contar os bits 1: pode ser feito com `popcount` (instrução `POPCNT` em x86) ou iterando pelos bits.

Cada grupo de 8 bits (octeto) mapeia para um byte. Separar o resultado em octetos é simplesmente um shift right de 8, 16, 24 bits respectivamente para extrair cada byte.

---

## Conexão com Sistemas Operacionais

- **[[Bits e Bytes]]** — A conversão /N → decimal é preencher 32 bits com N uns e depois agrupar em grupos de 8. Cada grupo de 8 bits é um byte (octeto). Em termos de código: `mask = ~0u << (32 - N)`, depois extrair cada byte com shifts e AND.
- **[[Operações Aritméticas]]** — A fórmula `mask = (0xFFFFFFFF << (32-N))` usa shift left e AND — operações aritméticas básicas de nível de máquina. O processo inverso conta bits 1 com `popcount` (instrução de hardware em CPUs modernas).
- **[[Processadores]]** — A instrução `POPCNT` (Population Count) em x86 conta os bits 1 de um inteiro em um único ciclo de clock — usada para converter máscara decimal em notação CIDR.

## Conexão com Go

- **[[Bits e Bytes]]** — `bits.OnesCount32(uint32(mask))` do pacote `math/bits` em Go conta os bits 1 — exatamente a conversão de máscara decimal para CIDR.
- **[[Operações Aritméticas]]** — `net.CIDRMask(25, 32)` implementa `(0xFFFFFFFF << 7)` internamente para gerar `255.255.255.128`. O cálculo do octeto parcial é: `octet = 0xFF & (0xFF << (8 - remainder))`.
- **[[Tipos de Dados]]** — `net.IPMask` é `[]byte`. Para extrair o CIDR de uma máscara: `ones, bits := mask.Size()` retorna `(25, 32)` para uma máscara /25.
