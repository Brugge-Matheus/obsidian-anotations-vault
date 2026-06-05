---
tags:
  - redes
  - redes/meios
---

# Cabos Par Trançado (UTP e STP)

Este é um detalhamento técnico essencial sobre o meio de transmissão mais utilizado em redes locais (LANs). A principal diferença entre o **UTP** e o **STP** está na **blindagem** e na capacidade de resistir a interferências externas.

---

### Cabos de Par Trançado: UTP vs. STP

Ambos os cabos utilizam 4 pares de fios de cobre trançados entre si. O trançamento serve para cancelar a **diafonia (crosstalk)**, que é a interferência de um fio sobre o outro.

### 1. UTP (Unshielded Twisted Pair)

**Par Trançado Sem Blindagem.**
É o cabo de rede padrão que vemos em quase todos os lugares (casas, escritórios comuns).

- **Construção:** Possui apenas o isolamento de plástico individual nos fios e uma capa externa de PVC. Não tem nenhuma proteção metálica extra.
- **Vantagens:**
    - **Baixo custo:** É o mais barato do mercado.
    - **Flexibilidade:** Fácil de passar por conduítes e fazer curvas.
    - **Fácil instalação:** Simples de crimpar (colocar o conector RJ-45).
- **Desvantagens:** Sensível a interferências eletromagnéticas (EMI) externas (motores, lâmpadas fluorescentes, cabos de energia próximos).
- **Uso ideal:** Ambientes internos residenciais e comerciais comuns.

### 2. STP (Shielded Twisted Pair)

**Par Trançado Com Blindagem.**
É uma versão "reforçada" do cabo de rede, projetada para ambientes hostis.

- **Construção:** Além da capa externa, cada par de fios (ou o conjunto todo) é envolvido por uma **folha de alumínio** ou uma malha metálica.
- **Vantagens:**
    - **Alta imunidade:** Protege os dados contra interferências eletromagnéticas pesadas.
    - **Segurança:** Mais difícil de sofrer interceptação de sinal por indução.
- **Desvantagens:**
    - **Caro:** O preço é significativamente maior que o UTP.
    - **Rígido:** Mais grosso e difícil de manusear em curvas apertadas.
    - **Aterramento:** Exige conectores especiais metálicos e **aterramento correto**. Se não for aterrado, a blindagem pode agir como uma antena e piorar a interferência.
- **Uso ideal:** Fábricas (perto de máquinas), tubulações que dividem espaço com cabos de alta tensão ou torres de rádio.

---

### Tabela Comparativa

| Característica | UTP (Sem Blindagem) | STP (Com Blindagem) |
| --- | --- | --- |
| **Custo** | Baixo | Alto |
| **Flexibilidade** | Alta | Baixa |
| **Proteção contra EMI** | Baixa | **Alta** |
| **Instalação** | Simples | Complexa (exige aterramento) |
| **Conector** | RJ-45 de plástico comum | RJ-45 metálico (blindado) |

---

### Variações Modernas (Nomenclatura Técnica)

Hoje em dia, a indústria usa siglas mais precisas que você pode encontrar em cabos **CAT6** ou **CAT7**:

- **U/UTP:** Sem blindagem nenhuma (o UTP clássico).
- **F/UTP:** Uma folha de alumínio envolve todos os 4 pares.
- **S/FTP:** Cada par tem sua blindagem individual e ainda há uma malha global por fora (o nível máximo de proteção).

---

### Dica de Ouro:

> **A Regra da Separação:**
Se você estiver usando cabos **UTP**, nunca os passe no mesmo conduíte que cabos de energia elétrica. A corrente alternada da rede elétrica gera um campo magnético que "suja" o sinal de dados, causando lentidão e perda de pacotes. Se for impossível separar, use cabos **STP**.

---

## Conexões com SO

- **Sinalização diferencial:** cada par conduz o sinal e seu complemento (invertido). O receptor subtrai os dois — ruídos que afetaram os dois fios igualmente (modo comum) se cancelam. Isso chama-se *common-mode rejection* e é a razão pela qual o par trançado funciona em ambientes barulhentos — [[Bits e Bytes]] (codificação diferencial).
- **O trançamento:** cada par é torcido em passo levemente diferente. A indutância mútua gerada é igual e oposta nos dois fios, então as interferências externas induzem tensões iguais e canceláveis — [[Dispositivos de IO]].
- **Categorias de cabo e largura de banda:** Cat5e suporta até 100 MHz, Cat6 até 250 MHz, Cat6A até 500 MHz. Maior frequência = maior taxa de bits possível. A CPU/controlador de rede precisa processar esses sinais em alta frequência — [[Processadores]].
