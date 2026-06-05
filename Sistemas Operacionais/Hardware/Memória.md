---
tags:
  - sistemas-operacionais
  - so/hardware
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.3.2"
---
# Memória

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.3.2

---

# 🧠 O Problema: Memória Ideal vs. Memória Real

Para que a CPU trabalhe bem, ela precisa de memória com três características:

- **Muito rápida** — mais rápida do que executar uma instrução, para não segurar a CPU esperando
- **Muito grande** — capaz de armazenar todos os programas e dados necessários
- **Muito barata** — viável para ser produzida em massa

O problema é que **nenhuma tecnologia atual satisfaz esses três requisitos ao mesmo tempo**. Memória rápida é cara e pequena; memória grande é lenta e barata.

A solução encontrada pelos engenheiros foi construir o sistema de memória como uma **hierarquia de camadas**, onde cada camada é mais lenta, maior e mais barata que a anterior.

---

# 🏗️ A Hierarquia de Memória

> 📌 **Figura 1.9 — Uma hierarquia de memória típica**
> 

```
  Tempo típico                                        Capacidade
  de acesso                                           típica

              ┌─────────────────────────┐
   < 1 ns     │      Registradores      │    < 1 KB
              └─────────────────────────┘
                          │
         ┌────────────────────────────────┐
   1–8 ns │            Cache              │    4–8 MB
         └────────────────────────────────┘
                          │
    ┌───────────────────────────────────────┐
  10–50 ns │       Memória principal         │   16–64 GB ┐
    └───────────────────────────────────────┘            │ Memória
                          │                              │ persistente
┌──────────────────────────────────────────────┐         │ opcional
19ms/100µs │      Disco magnético / SSD         │  2–16+ TB┘
└──────────────────────────────────────────────┘

  ↑ Rápido, pequeno, caro               Lento, grande, barato ↑
```

Quanto mais alta na hierarquia, mais rápida, menor e mais cara. Quanto mais baixa, mais lenta, maior e mais barata. O SO e o hardware trabalham juntos para dar a **ilusão** de que o computador tem uma memória grande e rápida ao mesmo tempo.

---

# 1️⃣ Registradores

Os registradores ficam **dentro da própria CPU**, feitos do mesmo material que ela. Por isso, são tão rápidos quanto a CPU — **não há atraso algum** para acessá-los.

- Capacidade: geralmente **32 × 32 bits** em CPUs de 32 bits ou **64 × 64 bits** em CPUs de 64 bits — menos de 1 KB no total *(consulte a página Bits e Bytes  para entender essa notação em detalhes)*
- Os programas precisam gerenciar os próprios registradores **via software** (o compilador decide o que colocar neles)
- São a memória mais rápida que existe no sistema, mas também a menor

---

# 2️⃣ Cache

A cache é controlada principalmente pelo **hardware** e funciona como um intermediário entre os registradores e a memória principal.

## Como funciona

A memória principal é dividida em **linhas de cache**, normalmente de **64 bytes** cada. Quando o programa precisa ler uma palavra da memória, o hardware verifica se aquela linha já está na cache:

- ✅ **Cache hit** (acerto): a linha está na cache → a requisição é atendida em poucos ciclos de CPU, **sem ir até a memória principal**
- ❌ **Cache miss** (falha): a linha não está → é preciso buscá-la na memória principal, com uma **penalidade de dezenas a centenas de ciclos de CPU**

> 💡 **Analogia:** Imagine que você está estudando. Os livros na sua mesa são a cache — acesso rápido. Os livros na estante no outro cômodo são a memória principal — você precisa se levantar para buscá-los (lento). Você naturalmente mantém na mesa os livros que está usando com mais frequência.
> 

## Níveis de Cache

CPUs modernas têm **dois ou mais níveis** de cache:

| Nível | Localização | Velocidade | Tamanho típico |
| --- | --- | --- | --- |
| **Cache L1** | Dentro da CPU, sempre presente | Sem atraso algum | 32 KB |
| **Cache L2** | Dentro ou muito próximo da CPU | Alguns ciclos de atraso | Vários MB |

Em chips multinúcleo, a L2 pode ser **compartilhada entre todos os núcleos** (mais simples, exige controlador mais complexo) ou **individual por núcleo** (mais difícil manter consistência entre as caches).

## O Princípio do Caching

O conceito de caching é muito mais amplo do que só a memória — ele aparece em toda a computação. A ideia central é:

> **Quando um recurso pode ser dividido em partes, e algumas partes são usadas com muito mais frequência do que outras, vale a pena guardar as mais usadas em um local de acesso mais rápido.**
> 

O SO usa caching o tempo todo. Exemplos:

- Mantém **partes de arquivos mais acessados** na memória principal para não precisar ir ao disco toda vez
- Armazena em cache a **conversão de nomes de arquivo** (ex: `/home/ast/projects/minix3/src/kernel/clock.c`) para o endereço no disco, evitando buscas repetidas
- Guarda em cache a **conversão de URLs** (endereço web) para endereços IP de rede

---

# 3️⃣ Memória Principal (RAM)

A memória principal — **RAM** (*Random Access Memory*, memória de acesso aleatório) — é a "locomotiva" do sistema de memória. É onde ficam os programas em execução e os dados que estão sendo processados ativamente.

- Tempo de acesso: **10 a 50 nanossegundos**
- Capacidade típica atual: **16 a 64 GB**
- É **volátil** — perde tudo quando a energia é cortada
- Todas as requisições que não são atendidas pela cache chegam até a RAM

> ⚠️ A RAM também é chamada de **memória de núcleo** (*core memory*) em alguns textos antigos — nas décadas de 1950 e 1960, as memórias principais usavam minúsculos núcleos de ferrite magnetizáveis. Hoje o nome persiste, mas a tecnologia é completamente diferente.
> 

---

# 4️⃣ Memórias Não Voláteis: ROM, EEPROM, Flash e CMOS

Além da RAM, existem outros tipos de memória com características importantes:

| Tipo | Características | Uso típico |
| --- | --- | --- |
| **ROM** (*Read Only Memory*) | Gravada na fábrica, não pode ser modificada, não volátil, rápida e barata | Bootloader em computadores antigos |
| **EEPROM** (*Electrically Erasable PROM*) | Não volátil, pode ser apagada e regravada eletricamente, mas muito mais lenta que a RAM para escrita | Firmware, armazenamento de configurações |
| **Memória Flash** | Não volátil, pode ser apagada e regravada, mais rápida que disco mas mais lenta que RAM, se desgasta com muitas gravações | SSDs, smartphones, código de inicialização (BIOS/UEFI) |
| **CMOS** | Volátil, consome pouquíssima energia, alimentada por bateria interna | Armazenar hora, data e parâmetros de configuração do sistema mesmo com o computador desligado |

> 💡 **Sobre a CMOS:** é por isso que quando a bateria da placa-mãe morre, o computador "esquece" as configurações e a hora — como o Tanenbaum brinca, ele começa a ter a "doença de Alzheimer", esquecendo coisas como de qual disco deve inicializar o sistema operacional.
> 

> 💡 **Sobre a Flash:** o firmware de inicialização do computador (antigamente chamado de **BIOS**, hoje geralmente **UEFI**) fica armazenado em memória Flash. É por isso que ele persiste mesmo sem energia, mas pode ser atualizado (*flash do BIOS*).
> 

---

# 5️⃣ Memória Virtual e a MMU

Muitos computadores modernos suportam um esquema chamado **memória virtual**, que é um dos conceitos mais importantes de sistemas operacionais e resolve dois problemas fundamentais ao mesmo tempo:

1. **E quando um programa é maior do que a RAM disponível?**
2. **Como impedir que um processo acesse a memória de outro?**

## Como funciona a Memória Virtual

A ideia central é dar a cada processo a **ilusão de que ele tem um espaço de memória enorme e exclusivo** — como se a RAM fosse infinita e só existisse para ele. Na prática:

1. O programa inteiro fica armazenado no **SSD ou disco** (armazenamento não volátil)
2. Apenas as partes **atualmente em uso** são carregadas na RAM
3. A RAM funciona como uma **cache para o disco** — guarda os pedaços mais ativos do programa
4. Quando o programa precisa de uma parte que não está na RAM, o SO libera espaço (gravando partes não usadas de volta no disco) e carrega o que é necessário

Cada processo trabalha com **endereços virtuais** — números que ele mesmo inventou, começando do zero, como se tivesse a memória toda para si. Por baixo, o hardware mapeia esses endereços virtuais para os **endereços físicos reais** na RAM.

## O Papel da MMU

Quem faz essa tradução de endereços em tempo real é uma parte da CPU chamada **MMU** (*Memory Management Unit* — Unidade de Gerenciamento de Memória):

```
Processo A pensa que está em:    MMU traduz para:      RAM física:

  endereço virtual 0x0001    →   endereço físico 0x4A20
  endereço virtual 0x0002    →   endereço físico 0x4A21
  endereço virtual 0x0003    →   endereço físico 0x4A22

Processo B também pensa que está em:

  endereço virtual 0x0001    →   endereço físico 0x9F10  (lugar diferente!)
  endereço virtual 0x0002    →   endereço físico 0x9F11
```

Os dois processos usam os mesmos endereços virtuais (0x0001, 0x0002...), mas a MMU os mapeia para regiões **completamente diferentes** da RAM física. Cada um vive em seu próprio universo de memória — sem saber da existência do outro.

A MMU usa **tabelas de páginas** (*page tables*) mantidas pelo SO para saber como fazer esse mapeamento. Cada processo tem sua própria tabela de páginas.

## Benefícios da Memória Virtual

**1. Isolamento e segurança entre processos**

Como cada processo tem seu próprio espaço de endereçamento virtual, um processo **não consegue enxergar nem acessar a memória de outro**. Se o processo A tentar acessar um endereço que não pertence ao seu mapeamento, a MMU detecta a violação e lança um trap — o SO encerra o processo com o famoso *segmentation fault*. Isso é o que faz um programa travado não corromper os dados de outros programas rodando ao mesmo tempo.

**2. Proteção do Kernel**

O código do SO (kernel) também é mapeado no espaço de endereçamento virtual de cada processo, mas em uma região marcada como **inacessível em modo usuário**. Se um programa tentar acessar essa região, a MMU bloqueia imediatamente.

**3. Programas maiores que a RAM**

Um programa pode ter 16 GB de dados e rodar em um computador com 8 GB de RAM — o SO cuida de manter na RAM apenas as páginas mais ativas, colocando o resto no disco.

**4. Simplicidade para o programador**

Todo processo enxerga um espaço de endereçamento limpo e contíguo começando do zero. O programador não precisa se preocupar com onde na RAM física seu programa vai parar, nem com outros processos ocupando partes da memória.

**5. Compartilhamento controlado**

Processos podem *voluntariamente* compartilhar regiões de memória — isso é chamado de **memória compartilhada** (*shared memory*). O SO configura as tabelas de páginas de dois processos para que certos endereços virtuais diferentes apontem para o **mesmo endereço físico** na RAM. Mas isso só acontece explicitamente — por padrão, total isolamento.

## Impacto na Troca de Contexto

A MMU tem impacto direto no desempenho: **todo acesso à memória passa pela MMU**, que precisa consultar a tabela de páginas para traduzir o endereço. Quando o SO troca de um processo para outro (**troca de contexto**), as tabelas de mapeamento da MMU precisam ser trocadas também — porque cada processo tem sua própria tabela de páginas. Tanto a tradução de endereços quanto a troca das tabelas podem ser operações demoradas, razão pela qual troca de contexto é considerada uma operação cara em sistemas operacionais.

---

# 🔄 Como Tudo se Conecta: O SO Gerenciando a Hierarquia

- **Registradores** → gerenciados pelo compilador e pelo próprio programa
- **Cache** → gerenciada pelo hardware automaticamente (o SO não interfere diretamente)
- **RAM** → gerenciada pelo SO — ele decide quais processos ficam na RAM e quais vão para o disco
- **Memória Virtual** → gerenciada em conjunto pelo SO (decide o que sobe/desce) e pela MMU (faz a tradução de endereços)
- **Disco/SSD** → gerenciado pelo SO por meio do sistema de arquivos

---

# ✅ Resumo do Conceito

- Nenhuma tecnologia oferece memória **rápida + grande + barata** ao mesmo tempo — por isso existe a **hierarquia de memória**
- **Registradores** são os mais rápidos (< 1 ns), ficam dentro da CPU, mas têm menos de 1 KB
- A **cache** (L1 e L2) fica entre a CPU e a RAM, reduzindo drasticamente o número de acessos lentos à memória principal. Cache hit = rápido; cache miss = penalidade pesada
- O **princípio do caching** — guardar as partes mais usadas no lugar mais rápido — é universal na computação e o SO o utiliza extensivamente
- A **RAM** é onde os programas em execução ficam — volátil, 16–64 GB típicos
- **ROM, EEPROM e Flash** são não voláteis e guardam firmware e configurações; a **CMOS** guarda hora e configurações usando bateria
- A **memória virtual** resolve dois problemas: permite rodar programas maiores que a RAM, e isola completamente os processos uns dos outros em espaços de endereçamento separados
- A **MMU** traduz endereços virtuais em físicos dinamicamente usando tabelas de páginas — cada processo tem a sua
- O isolamento da MMU garante que um processo não consegue enxergar nem corromper a memória de outro — qualquer tentativa gera um trap e o SO encerra o processo (*segmentation fault*)
- Processos podem compartilhar memória **voluntariamente** via *shared memory*, mas por padrão o isolamento é total
- A troca de contexto envolve trocar as tabelas de páginas da MMU, o que a torna uma operação cara em termos de desempenho