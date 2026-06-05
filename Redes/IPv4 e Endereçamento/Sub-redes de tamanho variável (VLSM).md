---
tags:
  - redes
  - redes/ipv4
---

# Sub-redes de tamanho variável (VLSM)

### Guia de Cálculo VLSM (Máscara de Tamanho Variável)

O VLSM é utilizado quando precisamos dividir uma rede principal em sub-redes de tamanhos diferentes para evitar o desperdício de IPs.

### 1. Organização e Priorização

O primeiro passo é listar as necessidades de cada setor e **ordenar do maior para o menor**.

- **Rede Principal:** `192.168.0.0/24` (Classe C)
- **Ordem de Cálculo:**
    1. Call-Center: **120** computadores
    2. Secretaria: **60** computadores
    3. Lab: **20** computadores
    4. Coordenação: **6** computadores

---

### 2. Escolha da Máscara por Setor

Para cada setor, encontramos a menor potência de 2 que suporte a quantidade de hosts (lembrando de subtrair 2 para Rede e Broadcast).

- **Call-Center (120):** $2^7 = 128$. Úteis: $128 - 2 = \mathbf{126}$
    - Máscara: `255.255.255.128` (1 bit de rede no último octeto).
- **Secretaria (60):** $2^6 = 64$ Úteis: $64 - 2 = \mathbf{62}$
    - Máscara: `255.255.255.192` (2 bits de rede: $128+64$).
- **Lab (20):** $2^5 = 32$ Úteis: $32 - 2 = \mathbf{30}$
    - Máscara: `255.255.255.224` (3 bits de rede: $128+64+32$).
- **Coordenação (6):** $2^3 = 8$ Úteis: $8 - 2 = \mathbf{6}$
    - Máscara: `255.255.255.248` (5 bits de rede: $128+64+32+16+8$).

---

### 3. Cálculo dos Saltos e Montagem da Tabela

Agora, usamos o "Número Mágico" ($256 - \text{máscara}$) para definir o **Salto** de cada rede. No VLSM, o salto muda a cada nova sub-rede.

| Setor | Máscara Decimal | Salto $(256 - \text{Masc})$ | Endereço de Rede | Faixa de Hosts | Broadcast |
| --- | --- | --- | --- | --- | --- |
| **Call-Center** | `.128` | **128** | `192.168.0.0` | `.1` até `.126` | `192.168.0.127` |
| **Secretaria** | `.192` | **64** | `192.168.0.128` | `.129` até `.190` | `192.168.0.191` |
| **Lab** | `.224` | **32** | `192.168.0.192` | `.193` até `.222` | `192.168.0.223` |
| **Coordenação** | `.248` | **8** | `192.168.0.224` | `.225` até `.230` | `192.168.0.231` |

**Próxima Rede Disponível:** `192.168.0.232`

---

### Pontos Chave:

1. **O Salto é Dinâmico:** Diferente do Subnetting comum, no VLSM o salto da próxima rede depende da máscara da rede atual.
2. **Sem Sobreposição:** Note como o Broadcast de uma rede é sempre o número anterior ao início da próxima rede.
3. **Eficiência:** Se tivéssemos usado a máscara do Call-Center (/25) para todos, teríamos apenas 2 redes. Com VLSM, conseguimos atender todos os setores e ainda sobraram IPs (do `.232` ao `.255`).

---

## Conexões com SO e Go

**Por que isso importa além das redes:**

- VLSM é, conceitualmente, uma **árvore binária de particionamento** do espaço de endereços — cada nível da árvore corresponde a um bit da máscara. Alocar sempre a maior sub-rede primeiro evita fragmentação, exatamente como alocadores de memória do SO evitam fragmentação externa ao servir blocos maiores antes. Ver [[Bits e Bytes]].
- O roteamento CIDR (que torna VLSM possível) usa **longest prefix match**: o roteador compara o IP de destino bit a bit contra as entradas da tabela de roteamento e escolhe a entrada com o prefixo mais longo. É uma busca eficiente em árvore binária (trie) que a CPU executa a cada pacote. Ver [[Processadores]].
- Sem VLSM/CIDR, a internet seria inviável por desperdício de endereços. O mesmo problema ocorre com alocação ingênua de memória em processos — motivo pelo qual o kernel usa slab allocator e buddy system. Ver [[Processos]].
