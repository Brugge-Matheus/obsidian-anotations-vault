---
tags:
  - redes
  - redes/ipv4
---

# Máscaras de rede CIDR

A **Máscara CIDR** (*Classless Inter-Domain Routing* ou Roteamento Entre Domínios Sem Classe) é a forma moderna e simplificada de representar a máscara de rede.

Ela foi criada para substituir o sistema rígido de classes (A, B, C) e permitir que a divisão entre **Rede** e **Host** ocorra em qualquer bit, não apenas nos pontos dos octetos.

---

## 1. O que é a Notação CIDR?

Em vez de escrever a máscara no formato decimal pontuado (ex: `255.255.255.0`), usamos uma **barra (/)** seguida pelo número de bits que estão "ligados" (valor 1) na máscara.

- **Exemplo:** `/24` significa que os primeiros **24 bits** do endereço IP são destinados à identificação da rede.

## 2. Como converter Decimal para CIDR?

Basta somar quantos bits "1" existem na máscara binária. Como cada octeto tem 8 bits:

- **255.0.0.0** → `11111111.00000000.00000000.00000000` → **8 bits** → `/8`
- **255.255.0.0** → `11111111.11111111.00000000.00000000` → **16 bits** → `/16`
- **255.255.255.0** → `11111111.11111111.11111111.00000000` → **24 bits** → `/24`

## 3. A "Quebra" das Classes (O poder do CIDR)

A grande vantagem do CIDR é permitir máscaras que não terminam em 0 ou 255. Isso permite criar sub-redes de tamanhos variados.

**Exemplo: Máscara `/26`**

- **Binário:** `11111111.11111111.11111111.11000000`
- **Cálculo:** Os primeiros 3 octetos estão cheios (255.255.255). No último octeto, temos os dois primeiros bits ligados $(128 + 64 = 192)$.
- **Decimal:** `255.255.255.192`

### Como gerar uma máscara CIDR programaticamente

Uma máscara /N pode ser gerada por: `(0xFFFFFFFF << (32 - N)) & 0xFFFFFFFF`

Isso é um shift left de 32 bits seguido de um AND para limitar a 32 bits. Por exemplo, /25:
- `0xFFFFFFFF << 7 = 0xFFFFFF80` = `255.255.255.128`

## 4. Tabela de Referência Rápida

Esta tabela é muito útil para ter à mão ao planejar redes:

| CIDR   | Máscara Decimal     | Bits de Host | IPs Totais | IPs Úteis (Hosts) |
| ------ | ------------------- | ------------ | ---------- | ----------------- |
| **/32**| 255.255.255.255     | 0            | 1          | 1 (IP único)      |
| **/30**| 255.255.255.252     | 2            | 4          | 2 (Links Ponto-a-Ponto) |
| **/29**| 255.255.255.248     | 3            | 8          | 6                 |
| **/28**| 255.255.255.240     | 4            | 16         | 14                |
| **/27**| 255.255.255.224     | 5            | 32         | 30                |
| **/26**| 255.255.255.192     | 6            | 64         | 62                |
| **/24**| 255.255.255.0       | 8            | 256        | 254               |
| **/16**| 255.255.0.0         | 16           | 65.536     | 65.534            |
| **/8** | 255.0.0.0           | 24           | 16.777.216 | 16.777.214        |

## 5. Por que subtraímos 2 dos IPs Totais?

Em qualquer máscara CIDR (menor que /31), você sempre perde dois endereços:

1. **O primeiro IP:** Endereço de Rede (ID da rede).
2. **O último IP:** Endereço de Broadcast (para falar com todos).

Fórmula:

$$
N^\circ \text{ de Hosts} = 2^{(32 - CIDR)} - 2
$$

Exemplo para /24:

$$
2^{(32 - 24)} - 2 = 2^8 - 2 = 256 - 2 = 254 \text{ hosts.}
$$

## A Regra dos Bits Contíguos

Uma máscara de rede válida **deve** ter todos os seus bits `1` à esquerda e todos os seus bits `0` à direita. Você nunca verá uma máscara onde um bit `0` aparece antes de um bit `1` (ex: `10110000` é inválido para máscara).

Por causa dessa regra, existem apenas **9 valores possíveis** para cada octeto de uma máscara de rede.

---

## Os 9 Valores Permitidos (Tabela de Referência)

Aqui estão os únicos números que você encontrará em uma máscara de rede IPv4, do mais "fechado" (mais rede) para o mais "aberto" (mais hosts):

| Valor Decimal | Valor Binário | O que significa?                    |
| ------------- | ------------- | ----------------------------------- |
| **255**       | `11111111`    | Todos os 8 bits são de **Rede**.    |
| **254**       | `11111110`    | 7 bits de Rede, 1 bit de Host.      |
| **252**       | `11111100`    | 6 bits de Rede, 2 bits de Host.     |
| **248**       | `11111000`    | 5 bits de Rede, 3 bits de Host.     |
| **240**       | `11110000`    | 4 bits de Rede, 4 bits de Host.     |
| **224**       | `11100000`    | 3 bits de Rede, 5 bits de Host.     |
| **192**       | `11000000`    | 2 bits de Rede, 6 bits de Host.     |
| **128**       | `10000000`    | 1 bit de Rede, 7 bits de Host.      |
| **0**         | `00000000`    | Todos os 8 bits são de **Host**.    |

---

## Por que esses números específicos?

Eles são o resultado da soma acumulada das potências de 2 (da esquerda para a direita):

- **128** = $128$
- **192** = $128 + 64$
- **224** = $128 + 64 + 32$
- **240** = $128 + 64 + 32 + 16$
- **248** = $128 + 64 + 32 + 16 + 8$
- **252** = $128 + 64 + 32 + 16 + 8 + 4$
- **254** = $128 + 64 + 32 + 16 + 8 + 4 + 2$
- **255** = $128 + 64 + 32 + 16 + 8 + 4 + 2 + 1$

### Exemplos de Máscaras Válidas vs. Inválidas

- **Válida:** `255.255.255.192` (Binário: `11111111.11111111.11111111.11000000`) → Todos os `1` estão juntos no início.
- **Inválida:** `255.255.255.170` (Binário: `11111111.11111111.11111111.10101010`) → Os bits `1` e `0` estão misturados. Isso não existe em máscaras de rede.

### Longest Prefix Match

No roteamento moderno com CIDR, os roteadores usam **longest prefix match** (correspondência de prefixo mais longo): para cada destino, comparam `IP & mask` para todas as entradas da tabela de roteamento e selecionam a que tiver a máscara mais específica (maior /N). Isso é uma busca linear ou em árvore (trie) na tabela de roteamento — processado a alta velocidade em hardware especializado.

---

## Conexão com Sistemas Operacionais

- **[[Bits e Bytes]]** — CIDR /N = N bits 1 seguidos de zeros. Uma máscara /24 = `0xFFFFFF00`. Gerar a máscara programaticamente: `mask = (0xFFFFFFFF << (32-N))`. Isso é um shift left de bits — operação fundamental de CPU.
- **[[Operações Aritméticas]]** — O número de hosts $2^{(32-N)} - 2$ é calculado com exponenciação de base 2 (bit shift: `1 << (32-N)`), seguido de subtração. Toda a aritmética de redes se reduz a operações de bits.
- **[[Processadores]]** — Longest prefix match em roteamento: roteadores comparam `IP & mask` para cada entrada da tabela e escolhem a mais específica. Hardware moderno implementa isso com estruturas de dados trie (árvore de prefixos) em memória especializada (CAM/TCAM) para decisão em tempo O(1).

## Conexão com Go

- **[[Tipos de Dados]]** — `net.CIDRMask(24, 32)` retorna um `net.IPMask` com os primeiros 24 bits em 1. `net.ParseCIDR("192.168.1.0/24")` retorna a rede como `*net.IPNet`. `net.IPNet` tem campos `IP` (endereço) e `Mask` (máscara).
- **[[Bits e Bytes]]** — Internamente, `net.CIDRMask(n, 32)` faz exatamente o shift: preenche bytes com `0xFF` para os octetos completos e calcula o octeto parcial com `0xFF << (8 - bits%8)`.
- **[[Operações Aritméticas]]** — Calcular quantos hosts cabem: `1<<(32-cidr) - 2` em Go — shift de bits seguido de subtração.
