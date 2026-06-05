---
tags:
  - sistemas-operacionais
  - so/introdução
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Bits | Combinações | Cálculo |"
---
# Bits e Bytes

📚 **Referência:** Conceito fundamental para entender as anotações de Memória e Processadores

---

# ⚡ O que é um Bit?

Um **bit** (*binary digit* — dígito binário) é a menor unidade de informação que existe em computação. Ele só pode ter **dois valores possíveis**: 0 ou 1.

Essa limitação não é por acaso — o hardware trabalha com eletricidade, e é muito fácil distinguir dois estados elétricos:

- **0** → sem tensão / tensão baixa
- **1** → com tensão / tensão alta

Todos os dados de um computador — textos, imagens, vídeos, programas — são, no fundo, sequências de bits.

---

# 📦 O que é um Byte?

Como um único bit carrega muito pouca informação, os bits são agrupados. O agrupamento padrão é o **byte**, que corresponde a **8 bits**.

```
1 byte = 8 bits

Exemplo: 01001101
         ^^^^^^^^
         8 bits = 1 byte
```

O byte virou a unidade básica de armazenamento — cada posição (endereço) na memória RAM armazena exatamente 1 byte.

---

# 🔢 Por que Base 2? De onde vem o "2 elevado a N"?

A chave está em como as combinações crescem quando você empilha bits.

Com **1 bit**, você tem apenas 2 combinações possíveis:

```
0
1
→ 2¹ = 2 combinações
```

Com **2 bits**, cada nova posição pode ser 0 ou 1, então o total **dobra**:

```
00
01
10
11
→ 2² = 4 combinações
```

Com **3 bits**, dobra de novo:

```
000, 001, 010, 011, 100, 101, 110, 111
→ 2³ = 8 combinações
```

Perceba o padrão: **cada bit adicionado dobra o número de combinações**. Por isso a base é sempre 2, e a regra geral é:

> **N bits = 2ᴺ combinações possíveis**
> 

| Bits | Combinações | Cálculo |
| --- | --- | --- |
| 1 | 2 | 2¹ |
| 2 | 4 | 2² |
| 3 | 8 | 2³ |
| 4 | 16 | 2⁴ |
| 8 (1 byte) | 256 | 2⁸ |
| 16 | 65.536 | 2¹⁶ |
| 32 | 4.294.967.296 | 2³² |
| 64 | 18.446.744.073.709.551.616 | 2⁶⁴ |

> 💡 **Analogia com dígitos decimais:** quantos números diferentes cabem em 2 dígitos decimais? Não são 2 — são 10² = 100 (de 00 até 99). Com bits é a mesma lógica, só que a base é 2 em vez de 10, porque cada dígito binário só tem 2 valores possíveis (0 ou 1) em vez de 10 (0 a 9).
> 

---

# 🧮 Multiplicação: o que significa "32 × 32 bits"?

Quando você vê uma notação como **32 × 32 bits** ou **64 × 64 bits**, ela segue o formato:

> **quantidade × tamanho de cada unidade**
> 

É simplesmente multiplicação direta:

- **32 × 32 bits** = 32 registradores × 32 bits cada = **1.024 bits no total**
- **64 × 64 bits** = 64 registradores × 64 bits cada = **4.096 bits no total**

Para converter bits em bytes, divide por 8 (já que 1 byte = 8 bits):

- 1.024 bits ÷ 8 = **128 bytes**
- 4.096 bits ÷ 8 = **512 bytes**

Ambos menores que 1 KB — por isso os registradores têm capacidade ínfima comparada à RAM.

---

# 🏠 Cada Endereço de Memória = 1 Byte

A memória RAM pode ser imaginada como uma sequência gigante de "caixinhas". Cada caixinha:

- Tem um **endereço único** que a identifica
- Armazena exatamente **1 byte (= 8 bits)**

```
Endereço → Conteúdo (sempre 1 byte = 8 bits)
─────────────────────────────────────────────
0x0000   →  01001101
0x0001   →  11001010
0x0002   →  00110101
0x0003   →  10100011
  ...          ...
```

Como cada endereço aponta para 1 byte, o número de endereços possíveis define diretamente o **máximo de memória RAM** que a CPU consegue enxergar.

---

# 💾 O Tamanho do Registrador Define o Limite de RAM

O tamanho do registrador da CPU determina o tamanho dos endereços de memória que ela consegue gerar. Aplicando 2ᴺ:

**CPU de 32 bits:**

- Endereços têm 32 bits → 2³² = 4.294.967.296 endereços = **~4 GB de RAM no máximo**
- O maior endereço possível em binário: `11111111 11111111 11111111 11111111`
- Qualquer byte além disso não tem endereço possível — a CPU simplesmente não o enxerga

**CPU de 64 bits:**

- Endereços têm 64 bits → 2⁶⁴ = **~18 exabytes de RAM no máximo** (teórico)
- Na prática, processadores atuais usam 48 dos 64 bits → 2⁴⁸ = **256 TB**, mais que suficiente

> ⚠️ É por isso que computadores antigos de 32 bits tinham o famoso **limite de 4 GB de RAM**. Adicionar mais RAM era inútil — a CPU não tinha bits suficientes nos endereços para apontar para qualquer coisa além desse limite.
> 

---

# 📏 Dados Maiores que 1 Byte

Quando um dado precisa de mais de 1 byte, ele ocupa **múltiplos endereços consecutivos**:

```
0x0000  →  00000000   ← byte 1 do int
0x0001  →  00000000   ← byte 2 do int
0x0002  →  00000000   ← byte 3 do int
0x0003  →  00000101   ← byte 4 do int
                         = número 5
```

Tamanhos comuns de tipos de dados:

| Tipo | Tamanho | Endereços ocupados |
| --- | --- | --- |
| `char` (caractere) | 1 byte | 1 |
| `short` (inteiro curto) | 2 bytes | 2 consecutivos |
| `int` (inteiro) | 4 bytes | 4 consecutivos |
| `long` / `double` | 8 bytes | 8 consecutivos |
| `float` | 4 bytes | 4 consecutivos |

O compilador e o SO gerenciam isso automaticamente. Em Python e Java o programador não precisa se preocupar — mas por baixo, o mecanismo é sempre o mesmo.

---

# 📐 Unidades de Medida: KB, MB, GB, TB...

Como computadores trabalham em base 2, as unidades de armazenamento também seguem potências de 2:

| Unidade | Valor exato | Aprox. |
| --- | --- | --- |
| 1 Kilobyte (KB) | 2¹⁰ bytes | 1.024 bytes |
| 1 Megabyte (MB) | 2²⁰ bytes | ~1 milhão |
| 1 Gigabyte (GB) | 2³⁰ bytes | ~1 bilhão |
| 1 Terabyte (TB) | 2⁴⁰ bytes | ~1 trilhão |
| 1 Petabyte (PB) | 2⁵⁰ bytes | ~1 quatrilhão |
| 1 Exabyte (EB) | 2⁶⁰ bytes | ~1 quintilhão |

> 💡 Por isso 1 KB não é exatamente 1.000 bytes — é 1.024. Fabricantes de HD usam a definição decimal (1 KB = 1.000 bytes) para fazer os produtos parecerem maiores, enquanto o SO usa a binária (1 KB = 1.024) — daí a confusão clássica de comprar um HD de "1 TB" e o SO mostrar menos.
> 

---

# ✅ Resumo

- **Bit** = menor unidade de informação, vale 0 ou 1
- **Byte** = 8 bits — unidade básica de armazenamento na memória
- **N bits = 2ᴺ combinações** — cada bit adicionado dobra o total (base 2)
- **Multiplicação** como 64 × 64 bits = quantidade × tamanho → dividir por 8 para obter bytes
- **Cada endereço de memória armazena exatamente 1 byte**
- O tamanho do registrador define o limite de RAM: **32 bits → ~4 GB**, **64 bits → ~18 exabytes**
- Dados maiores que 1 byte ocupam múltiplos endereços consecutivos
- Unidades de armazenamento: KB = 2¹⁰, MB = 2²⁰, GB = 2³⁰, TB = 2⁴⁰...