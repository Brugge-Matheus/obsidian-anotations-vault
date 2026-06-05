---
tags:
  - redes
  - redes/meios
---

# Meios de Transmissão (Meios Guiados e Não Guiados)

Este tópico é o que define a **camada física** da rede. É a escolha entre usar cabos (guiados) ou o ar/vácuo (não guiados) para levar a informação de um ponto a outro.

### Meios de Transmissão: A Base Física da Rede

Os meios de transmissão são as "estradas" por onde os dados viajam. Eles são classificados em dois grandes grupos:

---

### 1. Meios Guiados (Com Fio / Cabeados)

Neste modelo, as ondas (elétricas ou de luz) são contidas e conduzidas através de um caminho físico sólido.

- **Cabo de Par Trançado (Twisted Pair):**
    - **O que é:** O famoso "cabo de rede" (RJ-45). São 8 fios de cobre trançados em pares para reduzir a interferência eletromagnética.
    - **Categorias (CAT):**
        - **CAT5e:** Até 1 Gbps (padrão doméstico).
        - **CAT6/6a:** Até 10 Gbps (padrão corporativo atual).
    - **Limitação:** O sinal degrada após **100 metros**.
- **Cabo Coaxial:**
    - **O que é:** Um condutor central de cobre protegido por uma malha metálica.
    - **Uso:** Muito usado por empresas de TV a cabo e Internet (Claro/NET). É mais resistente a interferências que o par trançado em distâncias maiores.
- **Fibra Óptica:**
    - **O que é:** Um filamento de vidro ou plástico que transmite **pulsos de luz**.
    - **Vantagens:** Velocidade altíssima (Tbps), **imune a interferências elétricas** e alcança dezenas de quilômetros sem precisar de repetidores.
    - **Tipos:** **Monomodo** (longas distâncias) e **Multimodo** (curtas distâncias/Data Centers).

---

### 2. Meios Não Guiados (Sem Fio / Wireless)

Aqui, os dados são transmitidos através de ondas eletromagnéticas que se propagam pelo espaço (ar, vácuo ou água). O sinal não tem um caminho físico fixo.

- **Ondas de Rádio:**
    - **O que é:** Sinais que se espalham em todas as direções (omnidirecionais) e conseguem atravessar obstáculos como paredes.
    - **Exemplos:** Wi-Fi, Bluetooth e Rádio AM/FM.
- **Micro-ondas:**
    - **O que é:** Sinais de alta frequência que exigem **visada direta** (as antenas precisam "se enxergar").
    - **Uso:** Links entre torres de celular, comunicação entre prédios e **Satélites** (Starlink, GPS).
- **Infravermelho:**
    - **O que é:** Luz invisível de curto alcance. Não atravessa paredes.
    - **Exemplo:** Controle remoto da TV e sensores de proximidade.

---

### Tabela Comparativa

| Meio | Tipo | Velocidade | Distância Máx. | Custo |
| --- | --- | --- | --- | --- |
| **Par Trançado** | Guiado | Alta (10 Gbps) | 100 metros | Baixo |
| **Fibra Óptica** | Guiado | **Altíssima** | **Quilômetros** | Alto |
| **Wi-Fi** | Não Guiado | Média | ~50-100 metros | Médio |
| **Satélite** | Não Guiado | Média/Alta | Global | Muito Alto |

---

### Dicas Técnicas

1. **Interferência Eletromagnética (EMI):** Cabos de cobre (par trançado e coaxial) sofrem com motores elétricos e lâmpadas fluorescentes. A **Fibra Óptica** é a única 100% imune a isso.
2. **Segurança:** Meios guiados são mais seguros, pois para interceptar o dado é preciso "cortar" o cabo. Meios não guiados (Wi-Fi) podem ser "ouvidos" por qualquer um no alcance do sinal, por isso exigem criptografia forte (WPA3).
3. **Atenuação:** É a perda de força do sinal conforme a distância aumenta. A fibra tem a menor atenuação de todos os meios.

---

## Conexões com SO

- **Meios guiados (cobre, fibra):** o sinal fica fisicamente contido no condutor — o SO enxerga o meio como um [[Dispositivos de IO]] de bloco/caractere. A NIC abstrai o meio e entrega quadros ao driver.
- **Fibra óptica:** transmite pulsos de luz por reflexão interna total; sem campo elétrico, impossível sofrer EMI. Cada fóton carrega um bit — [[Bits e Bytes]].
- **Cobre:** o sinal elétrico sofre atenuação com a distância; repetidores regeneram o sinal antes que caia abaixo do limiar de detecção — [[Dispositivos de IO]].
- **Wireless (meio compartilhado):** como o ar é compartilhado por todos, é necessário protocolo de acesso ao meio. Wi-Fi usa **CSMA/CA** (Carrier Sense Multiple Access / Collision Avoidance) — escuta antes de transmitir e usa backoff aleatório para evitar colisão. Ethernet legada usava **CSMA/CD** (Collision Detection) — detecta colisão e retransmite — [[Dispositivos de IO]].
