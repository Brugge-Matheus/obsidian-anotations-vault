---
tags:
  - sistemas-operacionais
  - so/hardware
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.3.3"
---
# Armazenamento não volátil

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.3.3

---

# 💾 O que é Armazenamento Não Volátil?

Armazenamento não volátil é qualquer tipo de armazenamento que **mantém os dados mesmo quando a energia é desligada**. É onde ficam o sistema operacional, os programas instalados, os arquivos do usuário — tudo que precisa persistir entre reinicializações.

Diferente da RAM (volátil — perde tudo sem energia), o armazenamento não volátil é a "memória de longo prazo" do computador.

Na hierarquia de memória, ele ocupa a posição mais baixa:

```
Registradores  →  Cache  →  RAM  →  Armazenamento não volátil
   < 1 ns          1-8 ns   10-50 ns      19ms / 100µs
   Rápido                                    Lento
   Caro                                      Barato
   Pequeno                                   Grande
```

O armazenamento em disco é **duas ordens de grandeza mais barato** por bit do que a RAM, e frequentemente **duas ordens de grandeza maior** também. O único problema é que o tempo para acessar dados aleatoriamente é cerca de **três ordens de grandeza mais lento** — daí a necessidade de toda a hierarquia de cache e memória virtual que já estudamos.

---

# 🔵 Disco Magnético (HD — Hard Disk)

O membro mais antigo e até pouco tempo mais comum da família. É um dispositivo **mecânico** — tem partes físicas que se movem, o que explica sua velocidade limitada.

## Estrutura Física

Um disco rígido consiste em um ou mais **pratos metálicos** que giram em alta velocidade — tipicamente 5.400, 7.200 ou 10.800 RPM. Um **braço mecânico** se move sobre esses pratos, semelhante ao braço de um toca-discos de vinil.

> 📌 **Figura 1.10 — Estrutura física de uma unidade de disco**
> 

```
               Cabeça de leitura/escrita (uma por superfície)
                             ↓
Superfície 7 → ══════════════════════════════

Superfície 6 → ══════════════════════════════  ← trilhas
Superfície 5 → ══════════════════════════════    (círculos
                                                  concêntricos)

Superfície 4 → ══════════════════════════════  → direção do
Superfície 3 → ══════════════════════════════    movimento
                                                   do braço

Superfície 2 → ══════════════════════════════
Superfície 1 → ══════════════════════════════
Superfície 0 → ══════════════════════════════
                     │
                  Spindle
           (eixo que gira todos os pratos)

Cilindro = todas as trilhas na mesma posição
do braço em todos os pratos simultaneamente
```

- **Superfícies** — cada prato tem duas faces utilizáveis
- **Trilhas** — círculos concêntricos em cada superfície onde os dados são gravados
- **Cilindro** — o conjunto de todas as trilhas na mesma posição do braço em todos os pratos
- **Setores** — cada trilha é dividida em setores, geralmente de **512 bytes** cada
- Cilindros externos têm mais setores que os internos

## Como ocorre uma leitura

Para ler um dado, o HD precisa de três etapas mecânicas:

1. **Seek (busca)** — o braço se move até o cilindro correto → ~1 ms para o cilindro adjacente, 5–10 ms para um cilindro aleatório
2. **Rotational latency (latência rotacional)** — espera o setor desejado girar até ficar sob a cabeça → 5–10 ms dependendo do RPM
3. **Transfer (transferência)** — leitura dos bits à medida que passam sob a cabeça → 50–160 MB/s

> ⚠️ O grande inimigo do HD é o **acesso aleatório** — cada leitura em um lugar diferente do disco exige mover o braço e esperar a rotação. É por isso que fragmentação de arquivos degrada tanto o desempenho: força o HD a mover o braço repetidamente para montar um único arquivo espalhado em pedaços pelo disco.
> 

## Papel do Controlador

Toda essa complexidade mecânica (converter número de setor em cilindro/trilha/cabeça, verificar checksums, remapear setores danificados) fica escondida atrás de uma **interface simples** fornecida pelo controlador do disco. O SO enxerga apenas uma sequência linear de blocos — a complexidade interna é transparente.

---

# 🟡 SSD (Solid State Drive)

O SSD substitui os pratos giratórios por **memória Flash** — sem peças móveis, sem movimento mecânico.

## Vantagens sobre o HD

- **Muito mais rápido** no acesso aleatório — dezenas de microssegundos vs. milissegundos do HD
- **Mais resistente** — sem peças mecânicas para quebrar
- **Mais silencioso e leve**
- **Melhor acesso a dados em locais aleatórios** — não precisa esperar rotação nem mover braço

## Desvantagens

- **Mais caro por byte** que o HD
- **Escrita mais complexa** — um bloco de dados precisa ser apagado antes de ser reescrito, o que leva mais tempo que a leitura
- **Desgaste por gravação** — cada célula Flash suporta um número limitado de ciclos de escrita/apagamento. O firmware do SSD usa **balanceamento de carga** (*wear leveling*) para distribuir as gravações uniformemente e maximizar a vida útil

## Do ponto de vista do SO

Apesar da tecnologia completamente diferente, do ponto de vista do SO os SSDs se comportam de forma **semelhante aos discos magnéticos** — são vistos como uma sequência de blocos endereçáveis. O mesmo sistema de arquivos funciona nos dois.

---

# 🟢 Memória Persistente

O membro mais novo e mais rápido da família do armazenamento estável. O exemplo mais conhecido é o **Intel Optane**, lançado em 2016.

A memória persistente pode ser vista como uma **camada adicional entre os SSDs e a RAM**:

- É **mais rápida** que SSD e HD
- É **ligeiramente mais lenta** que a RAM convencional
- **Mantém o conteúdo** entre os ciclos de energia (não volátil)
- Pode ser conectada **diretamente ao barramento de memória** — sem necessidade de controlador especial
- Pode ser usada como **memória normal** para armazenar estruturas de dados, com a vantagem de que os dados ainda estarão lá quando a energia voltar
- O acesso ocorre em **nível de byte**, não em blocos como discos e SSDs — isso elimina a necessidade de transferir grandes blocos de dados

> 💡 É por esse conjunto de características que a memória persistente é tão promissora — ela combina a velocidade próxima da RAM com a persistência do armazenamento, podendo eventualmente mudar como os SOs gerenciam dados.
> 

---

# 🔴 Dispositivos de I/O e Drivers

O armazenamento não volátil é acessado pelo SO através de **dispositivos de I/O**, e cada dispositivo de armazenamento tem dois componentes principais:

## Controlador

Um **chip** (ou conjunto de chips) que controla fisicamente o dispositivo. Ele aceita comandos de alto nível do SO (ex: "leia o setor 11.206 do disco 2") e os traduz para as operações de baixo nível necessárias — posicionamento do braço, espera pela rotação, verificação de checksum, remapeamento de setores danificados, etc.

O controlador apresenta uma **interface mais simples** para o SO do que a realidade interna do dispositivo.

## Interface Padronizada

Para que qualquer controlador de disco SATA possa controlar qualquer disco SATA, existe a padronização de interfaces. **SATA** (*Serial ATA*) é o padrão dominante atualmente — define como o controlador e o disco se comunicam, independentemente do fabricante.

> 💡 **Curiosidade do Tanenbaum:** SATA significa *Serial ATA*, e ATA significa *AT Attachment* — referência ao processador 80286 de 6 MHz que a IBM introduziu em 1984 na linha "AT" (*Advanced Technology*). Como o Tanenbaum observa: "aprendemos que um adjetivo como 'avançado' deve ser usado com cuidado, ou ele se tornará ridículo 40 anos depois."
> 

## Driver de Dispositivo

Como cada controlador é diferente, diferentes softwares são necessários para controlar cada um. O software que conversa com um controlador, enviando comandos e aceitando respostas, é chamado de **driver de dispositivo**.

- Cada fabricante fornece drivers para os principais SOs (Windows, Linux, macOS)
- Para funcionar, o driver precisa ser colocado dentro do SO e executado em **modo núcleo**
- Embora alguns SOs ofereçam suporte a drivers em modo usuário, a grande maioria ainda opera dentro do nível do núcleo por questões de desempenho
- É por isso que instalar um driver mal-feito ou malicioso pode comprometer todo o sistema — ele roda com privilégio total

---

# ⚖️ Comparativo dos Tipos de Armazenamento

| Característica | HD (Disco Magnético) | SSD | Memória Persistente |
| --- | --- | --- | --- |
| **Tecnologia** | Mecânica (pratos + braço) | Flash eletrônica | Nova tecnologia (ex: Optane) |
| **Velocidade leitura** | 50–200 MB/s | Centenas de MB/s a GB/s | Próxima da RAM |
| **Acesso aleatório** | Lento (ms) | Rápido (µs) | Muito rápido |
| **Persistência** | Sim | Sim | Sim |
| **Custo por GB** | Baixo | Médio | Alto |
| **Desgaste por escrita** | Baixo | Sim (limitado) | Baixo |
| **Barulho/vibração** | Sim | Não | Não |
| **Visão do SO** | Sequência de blocos | Sequência de blocos | Memória endereçável por byte |

---

# 🔄 Como o SO Interage com o Armazenamento

O SO gerencia o armazenamento não volátil em várias camadas:

1. **Driver de dispositivo** — fala diretamente com o controlador do hardware via comandos padronizados (SATA, NVMe, etc.)
2. **Subsistema de I/O** — coordena as requisições de acesso ao disco, podendo reordená-las para minimizar movimentos do braço (no caso do HD)
3. **Sistema de arquivos** — organiza os blocos físicos em arquivos e diretórios, mantém metadados (nome, tamanho, datas, permissões)
4. **Cache de disco** — mantém em RAM as páginas de disco mais recentemente acessadas, evitando leituras repetidas

---

# ✅ Resumo do Conceito

- Armazenamento não volátil **preserva dados sem energia** — é onde ficam o SO, programas e arquivos do usuário
- É **muito mais barato e grande** que a RAM, mas **muito mais lento** — daí a hierarquia de memória
- **HD** usa pratos metálicos giratórios e braço mecânico — lento no acesso aleatório por natureza física; estruturado em superfícies, trilhas, cilindros e setores
- **SSD** usa memória Flash sem partes móveis — muito mais rápido no acesso aleatório, mais caro, tem desgaste por escrita
- **Memória persistente** é a fronteira entre armazenamento e RAM — rápida, persistente, endereçável por byte
- O **controlador** esconde a complexidade do hardware; o **driver** é o software que fala com o controlador em modo núcleo
- Do ponto de vista do SO, HD e SSD são vistos da mesma forma — uma sequência de blocos endereçáveis