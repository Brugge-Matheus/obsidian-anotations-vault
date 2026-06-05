---
tags:
  - sistemas-operacionais
  - so/conceitos
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.6"
---
# Hardware de Proteção

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.6

---

# 🛡️ O que é Hardware de Proteção?

Hardware de proteção é o conjunto de mecanismos físicos dentro do processador que permitem ao SO **isolar processos uns dos outros e proteger a si mesmo**. Sem ele, qualquer programa poderia fazer qualquer coisa — ler a memória de outro processo, travar o sistema, acessar dispositivos diretamente.

A história do hardware de proteção é inseparável da história da **multiprogramação** — os dois caminham juntos, porque só é possível ter múltiplos programas na memória ao mesmo tempo se houver garantia de que eles não se destruirão mutuamente.

---

# 📜 A Evolução Histórica

## Primeiros Mainframes — Sem Proteção Nenhuma

Os primeiros computadores de grande porte, como o **IBM 7090/7094**, não tinham hardware de proteção. Um programa defeituoso poderia facilmente acabar com o sistema operacional e derrubar a máquina inteira:

```
IBM 7090 (anos 50-60):
┌─────────────────────────┐
│  Programa A             │ ← pode ler/escrever QUALQUER endereço
│  Sistema Operacional    │ ← inclusive aqui — sem proteção!
│  Programa B             │ ← e aqui também
└─────────────────────────┘
→ Um bug em qualquer programa derrubava tudo
```

Nesse ambiente, a **monoprogramação** era obrigatória — apenas um programa por vez na memória.

## IBM 360 — Primeira Proteção Primitiva

Com a introdução do **IBM 360**, surgiu uma forma primitiva de proteção de hardware. Essas máquinas podiam armazenar vários programas na memória ao mesmo tempo e deixá-los se alternarem na execução — o início da **multiprogramação**. A monoprogramação tornou-se obsoleta.

## Minicomputadores — PDP-1, PDP-8, PDP-11

Os primeiros minicomputadores (**PDP-1** e **PDP-8**) não tinham hardware de proteção, então a multiprogramação não era possível. O **PDP-11** finalmente teve hardware de proteção — e esse aspecto levou à multiprogramação e, por fim, ao **UNIX**.

## Microcomputadores — Intel 8080 ao 80286

Quando os primeiros microcomputadores foram construídos, usavam o chip **Intel 8080** — sem proteção de hardware. Estávamos de volta à monoprogramação.

Foi somente com o **Intel 80286** que o hardware de proteção foi acrescentado e a multiprogramação tornou-se possível nos PCs:

```
Intel 8080 (1974) → sem proteção → monoprogramação
Intel 8086 (1978) → sem proteção → monoprogramação (MS-DOS)
Intel 80286 (1982) → proteção básica → multiprogramação possível
Intel 80386 (1985) → proteção completa → modo protegido de 32 bits
```

> ⚠️ Até hoje, **muitos sistemas embarcados não têm hardware de proteção** e executam apenas um único programa. Isso é aceitável porque os projetistas têm controle total sobre todo o software que vai rodar neles.
> 

---

# ⚙️ Como o Hardware de Proteção Funciona

## 1. Modo Núcleo vs. Modo Usuário

O mecanismo fundamental é a divisão da CPU em dois modos de execução, controlados por um bit no PSW:

```
┌─────────────────────────────────────┐
│  MODO NÚCLEO (Kernel Mode)          │
│  → SO executa aqui                  │
│  → acesso total ao hardware         │
│  → todas as instruções disponíveis  │
├─────────────────────────────────────┤
│  MODO USUÁRIO (User Mode)           │
│  → programas executam aqui          │
│  → conjunto restrito de instruções  │
│  → sem acesso direto ao hardware    │
└─────────────────────────────────────┘
```

**Instruções privilegiadas** — como desabilitar interrupções, modificar tabelas de páginas, acessar dispositivos de I/O diretamente — só podem ser executadas em modo núcleo. Se um programa em modo usuário tentar executar uma instrução privilegiada, o hardware gera um **trap** e o SO encerra o processo.

## 2. Registradores Base e Limite

Uma das formas mais primitivas de proteção de memória usava dois registradores especiais:

```
Registrador BASE  = 0x4000   (onde começa a memória do processo)
Registrador LIMITE = 0x4000  (tamanho da região permitida)

Processo tenta acessar endereço X:
  Se X < BASE                → TRAP! abaixo da região permitida
  Se X >= BASE + LIMITE      → TRAP! acima da região permitida
  Se BASE <= X < BASE+LIMITE → acesso permitido ✓

┌──────────────────────────────────────┐
│  endereços 0x0000 a 0x3FFF          │ ← PROIBIDO para o processo
├──────────────────────────────────────┤
│  endereços 0x4000 a 0x7FFF          │ ← PERMITIDO (região do processo)
├──────────────────────────────────────┤
│  endereços 0x8000 em diante         │ ← PROIBIDO para o processo
└──────────────────────────────────────┘
```

Isso impedia que um processo lesse ou escrevesse fora de sua região de memória. O IBM 360 usava uma variação desse esquema. O SO carregava os registradores base e limite com os valores corretos para cada processo antes de ceder a CPU a ele.

## 3. MMU — Proteção via Páginas (Mecanismo Moderno)

O mecanismo moderno é a **MMU** (*Memory Management Unit*) com tabelas de páginas. Cada processo tem seu próprio espaço de endereçamento virtual completamente separado — já estudamos esse mecanismo em detalhes na página de **Espaços de Endereçamento**.

A MMU torna os registradores base/limite obsoletos ao oferecer proteção muito mais granular e flexível, permitindo inclusive memória virtual.

---

# 🔗 Por que Hardware de Proteção e Multiprogramação Andam Juntos

Sem hardware de proteção, um processo mal escrito pode:

- Sobrescrever o código do SO na memória
- Ler dados de outro processo (senha, dados bancários, etc.)
- Executar instruções privilegiadas diretamente
- Travar o sistema inteiro

Com hardware de proteção, o pior que um processo pode fazer é ser encerrado pelo SO. O restante do sistema continua funcionando normalmente.

```
Sem proteção de hardware:
  bug no Processo A → sistema inteiro trava ❌

Com proteção de hardware:
  bug no Processo A → SO detecta via trap
                    → encerra apenas o Processo A ✓
                    → Processos B, C, D continuam normalmente ✓
                    → SO continua funcionando ✓
```

---

# ✅ Resumo do Conceito

- Sem hardware de proteção, qualquer programa pode destruir o SO e outros programas — a **monoprogramação** era obrigatória
- A evolução foi gradual: IBM 7090 (sem proteção) → IBM 360 (proteção primitiva) → PDP-11 (proteção + UNIX) → Intel 80286 (proteção em PCs)
- Os mecanismos fundamentais são em ordem histórica: **modo núcleo/usuário** (instruções privilegiadas), **registradores base/limite** (proteção simples de região) e **MMU com tabelas de páginas** (proteção moderna e granular)
- Sistemas embarcados frequentemente **não têm hardware de proteção** — aceitável quando os projetistas controlam todo o software
- Hardware de proteção e multiprogramação são inseparáveis — um depende do outro