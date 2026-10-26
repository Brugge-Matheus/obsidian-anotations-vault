---
tags:
  - sistemas-operacionais
  - so/gerenciamento-de-memoria
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 3 — Seção 3.5.7"
---
# Bibliotecas Compartilhadas

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 3 — Seção 3.5.7

---

# 📚 3.5.7 — Bibliotecas Compartilhadas

## O compartilhamento além da página individual

O compartilhamento pode ser feito em **outras granularidades** além das páginas individuais ([[Páginas Compartilhadas]]).

```
SE um programa for inicializado DUAS vezes, a maioria
dos SOs vai compartilhar AUTOMATICAMENTE todas as
páginas de TEXTO (código), de maneira que apenas UMA
CÓPIA esteja na memória.

  → páginas de código são SEMPRE somente leitura
    → NÃO há problema nesse caso

  → quanto às páginas de DADOS, dependendo do SO:
      • cada processo pode ficar com sua PRÓPRIA
        cópia privada, OU
      • elas podem ser COMPARTILHADAS e marcadas como
        SOMENTE LEITURA

    SE qualquer processo MODIFICAR uma página de dados:
      → será feita uma cópia privada para ela
        (copy on write)
```

## O problema das bibliotecas grandes

Nos sistemas modernos, há **muitas bibliotecas grandes** usadas por **muitos processos** (por exemplo, múltiplas bibliotecas gráficas e de E/S).

> ⚠️ **Ligar estaticamente** todas essas bibliotecas a todo programa executável no disco as tornaria **ainda mais infladas** do que já são.

## A solução: bibliotecas compartilhadas (DLLs)

Uma técnica comum é usar **bibliotecas compartilhadas** (no Windows, chamadas de **DLLs**, ou ***dynamic link libraries*** — bibliotecas de ligação dinâmica).

---

## 🔗 Antes: como funciona a ligação tradicional (estática)

Para entender a biblioteca compartilhada, primeiro considere a **ligação tradicional**. Quando um programa é ligado, um ou mais **arquivos-objeto** e possivelmente algumas **bibliotecas** são indicados no comando para o **linker**, como o comando do UNIX:

```
ld *.o -lc -lm
```

```
O QUE ESSE COMANDO FAZ:

  1. LIGA todos os arquivos .o (objeto) no diretório
     atual

  2. depois VARRE duas bibliotecas:
       /usr/lib/libc.a
       /usr/lib/libm.a
```

```
EXTERNAS NÃO DEFINIDAS:

  Quaisquer FUNÇÕES chamadas nos arquivos-objeto, mas
  AUSENTES ali (p. ex. printf), são chamadas de
  "EXTERNAS NÃO DEFINIDAS" e são BUSCADAS nas
  bibliotecas.

  SE forem encontradas:
    → são INCLUÍDAS no arquivo binário executável

  Quaisquer funções que ELAS chamam, mas que ainda não
  estão presentes, TAMBÉM se tornam externas não
  definidas.

  EXEMPLO: printf precisa de write. Se write ainda não
  foi incluída, o linker procurará por ela e a
  incluirá quando for encontrada.
```

```
QUANDO o linker TERMINAR:

  → um ARQUIVO BINÁRIO é escrito para o armazenamento
    não volátil contendo TODAS as funções necessárias

  → funções presentes nas bibliotecas, mas NÃO CHAMADAS,
    NÃO são incluídas

  → quando o programa é CARREGADO na memória e
    executado, TODAS as funções de que ele precisa
    estão ali
```

---

## 💡 Por que ligar estaticamente não escala

```
Suponha que programas comuns usem 20 a 50 MB em funções
GRÁFICAS e de INTERFACE com o usuário.

LIGAR ESTATICAMENTE centenas de programas com todas
essas bibliotecas:

  → DESPERDIÇARIA uma quantidade TREMENDA de espaço no
    armazenamento não volátil

  → DESPERDIÇARIA espaço em RAM quando eles fossem
    carregados, já que o sistema NÃO TERIA COMO SABER
    que poderia COMPARTILHÁ-LAS
```

## ✅ Como a biblioteca compartilhada funciona

```
Quando um programa é LIGADO a bibliotecas
COMPARTILHADAS (ligeiramente diferentes das estáticas):

  → em vez de INCLUIR a função realmente chamada, o
    LINKER inclui uma pequena rotina de STUB que LIGA a
    função chamada NO MOMENTO DA EXECUÇÃO

QUANDO a biblioteca compartilhada é carregada, DEPENDE
do sistema e da configuração:

  → quando o PROGRAMA é carregado, OU
  → quando suas FUNÇÕES são chamadas pela PRIMEIRA vez

SE outro programa JÁ carregou a biblioteca:
  → NÃO há necessidade de fazê-lo novamente
  → ESSE é o ponto da questão

A biblioteca INTEIRA NÃO é lida na memória de uma única
vez:
  → as PÁGINAS entram UMA A UMA, na medida do necessário
  → assim, as funções que NÃO SÃO CHAMADAS não serão
    trazidas à RAM
```

> 💡 **Stub:** pequeno "esqueleto" de função no executável, que na hora da execução redireciona a chamada para a função real dentro da biblioteca compartilhada.

## Vantagens

| Vantagem | Explicação |
|---|---|
| **Executáveis menores** | a biblioteca não é copiada para dentro de cada programa |
| **Economia de memória** | uma única cópia na RAM serve a vários processos |
| **Atualização sem recompilar** | se uma função é corrigida (p. ex. um *bug*), **não é necessário recompilar** os programas que a chamam; **os binários antigos continuam funcionando** |

```
Essa última característica é de ESPECIAL IMPORTÂNCIA
para SOFTWARES COMERCIAIS, em que o código-fonte NÃO é
DISTRIBUÍDO ao cliente.

EXEMPLO: se a Microsoft encontrar e consertar um BUG de
segurança em alguma DLL padrão, o Windows Update fará o
DOWNLOAD da nova DLL e SUBSTITUIRÁ a antiga, e TODOS os
programas que usam a DLL AUTOMATICAMENTE usarão a nova
versão na PRÓXIMA vez que forem iniciados.
```

---

## ⚠️ O pequeno problema: relocação

Bibliotecas compartilhadas vêm com um **pequeno problema** que precisa ser resolvido.

> 📌 **Figura 3.26 — Uma biblioteca compartilhada sendo usada por dois processos**

```
   Processo 1                RAM               Processo 2

                                           
                                           
   36K ┌────────┐         ┌────────┐
       │ 4 KB   │────────►│ 4 KB   │◄────── ┌────────┐ 12K
       │ 4 KB   │────────►│ 4 KB   │◄────── │ 4 KB   │
       │ 4 KB   │────────►│ 4 KB   │◄────── │ 4 KB   │
       │ 4 KB   │────────►│ 4 KB   │◄────── │ 4 KB   │
       │ 4 KB   │────────►│ 4 KB   │◄────── │ 4 KB   │
       └────────┘         └────────┘        └────────┘
     biblioteca (20 KB)  (uma só cópia)   biblioteca (20 KB)
     começa em 36K                        começa em 12K

   Mesma biblioteca, mesmos 5 quadros físicos na RAM,
   mas ENDEREÇOS VIRTUAIS DIFERENTES em cada processo.
```

```
A situação do livro:

  → dois processos compartilham uma biblioteca de 20 KB
    (cada caixa = 4 KB)

  → a biblioteca está localizada em ENDEREÇOS
    DIFERENTES em cada processo (provavelmente porque os
    programas em si não são do mesmo tamanho):
        Processo 1: começa em 36K
        Processo 2: começa em 12K

  → suponha que a PRIMEIRA função da biblioteca tenha de
    SALTAR para o endereço 16 da biblioteca
```

### Se a biblioteca NÃO fosse compartilhada

```
Ela poderia ser REALOCADA DINAMICAMENTE quando
carregada:

  → o salto (no processo 1) seria para o endereço
    virtual 36K + 16

  → o endereço FÍSICO na RAM onde a biblioteca está NÃO
    IMPORTA, pois todas as páginas são mapeadas de
    endereços virtuais para físicos pela MMU, em
    hardware
```

### Mas como ELA É compartilhada...

```
A REALOCAÇÃO durante a execução NÃO FUNCIONARÁ.

  → quando a primeira função é chamada pelo PROCESSO 2
    (no endereço 12K), a instrução de salto tem de ir
    para 12K + 16, NÃO 36K + 16

  → a MESMA página física é usada pelos dois, mas o
    endereço absoluto escrito no código só serve para um
```

**Duas soluções:**

| Solução | Funciona? |
|---|---|
| **Cópia na escrita**: criar páginas novas para cada processo e **realocá-las** dinamicamente | Funciona, mas **frustra o propósito** do compartilhamento (cada processo ficaria com sua própria cópia) |
| **Compilar com uma *flag* especial** que **proíbe endereços absolutos**: só instruções com **endereços relativos** | ✅ **Solução melhor** |

```
SOLUÇÃO MELHOR:

  Compilar bibliotecas compartilhadas com uma FLAG
  ESPECIAL do compilador, dizendo-lhe para NÃO produzir
  quaisquer instruções que usem ENDEREÇOS ABSOLUTOS.

  Em vez disso, apenas instruções usando ENDEREÇOS
  RELATIVOS são usadas.

  EXEMPLO: quase sempre há uma instrução "salte para a
  frente (ou para trás) n bytes" (em oposição a uma
  instrução que dá um endereço ESPECÍFICO para saltar).

  Essa instrução funciona CORRETAMENTE não importa onde
  a biblioteca compartilhada for colocada no espaço de
  endereços virtuais.
```

> ✅ O código que usa **apenas deslocamentos relativos** é chamado de **código independente do posicionamento** (*position-independent code*, ou **PIC**).

---

# ✅ Resumo do Conceito

- O compartilhamento pode ocorrer em outras granularidades. Programas iniciados duas vezes compartilham as páginas de **texto** (somente leitura); as de **dados** podem ser privadas ou compartilhadas com copy on write
- **Ligação estática:** o linker (`ld *.o -lc -lm`) resolve as **externas não definidas** copiando as funções necessárias das bibliotecas para o executável. Com bibliotecas de 20 a 50 MB e centenas de programas, isso desperdiça disco e RAM
- **Bibliotecas compartilhadas (DLLs no Windows):** o linker coloca no executável só um **stub**, que liga a função chamada **em tempo de execução**. A biblioteca é carregada uma vez e usada por todos; suas **páginas entram uma a uma, sob demanda**
- **Vantagens:** executáveis menores, economia de memória e **atualização sem recompilar** (como o Windows Update corrigindo uma DLL)
- **Problema:** a mesma biblioteca fica em **endereços virtuais diferentes** em cada processo (36K × 12K na Figura 3.26), então **endereços absolutos não funcionam**. Copiar e realocar frustra o compartilhamento. A solução é compilar com uma flag que gere apenas **endereços relativos**: **código independente do posicionamento**

---

## 🔗 Notas Relacionadas

- [[Páginas Compartilhadas]] — o compartilhamento em nível de página e copy on write (3.5.6)
- [[Arquivos Mapeados em Memória]] — mecanismo mais geral que as bibliotecas compartilhadas usam (3.5.8)
- [[Ausência de abstração de memória]] — o problema de relocação que aparece aqui de novo (endereços absolutos × relocação)
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — o modelo de compilação e ligação em C
- [[Acelerando a Paginação]] — como as páginas entram uma a uma, sob demanda, por falta de página
