---
tags:
  - sistemas-operacionais
  - so/linguagem-c
source: "Sistemas Operacionais Modernos — Tanenbaum, 5ª Ed."
chapter: "Cap. 1 — Seção 1.8 e 1.8.1"
---
# A linguagem C

📚 **Referência:** Sistemas Operacionais Modernos — Andrew S. Tanenbaum, 5ª Edição | Cap. 1 — Seção 1.8 e 1.8.1

---

# 🌍 Por que Sistemas Operacionais são Escritos em C?

Sistemas operacionais normalmente são grandes programas C (ou às vezes C++) consistindo em muitas partes escritas por muitos programadores. O ambiente usado para desenvolver os sistemas operacionais é muito diferente daquele ao qual os indivíduos estão acostumados quando estão escrevendo programas pequenos em Java.

Esta seção é uma tentativa de fazer uma introdução muito breve ao mundo da escrita de um sistema operacional para programadores Java ou Python modestos.

> 💡 **Por que C e não Java ou Python?** Sistemas operacionais são, até certo ponto, basicamente sistemas em tempo real — mesmo sistemas com propósito geral. Quando ocorre uma interrupção, o sistema operacional pode ter apenas alguns microssegundos para realizar alguma ação ou perder informações críticas. **A entrada do coletor de lixo em um momento arbitrário é algo intolerável** — você não pode pausar o tratamento de uma interrupção porque o garbage collector resolveu rodar. Isso elimina imediatamente linguagens com GC como Java e Python para o núcleo do SO.
> 

---

# 📖 1.8.1 — A Linguagem C

Este não é um guia para a linguagem C, mas um breve resumo de algumas das diferenças fundamentais entre C e linguagens como Python e, especialmente, Java.

## Semelhanças com Java e Python

Java é baseado em C, portanto há muitas semelhanças entre as duas. Python é de certa maneira diferente, mas também ligeiramente semelhante. Java, Python e C são todas **linguagens imperativas** com:

- Tipos de dados
- Variáveis
- Comandos de controle (`if`, `switch`, `for`, `while`)

Os tipos de dados primitivos em C são inteiros (incluindo curtos e longos), caracteres e números de ponto flutuante. Os tipos de dados compostos podem ser construídos usando arranjos, estruturas e uniões. Funções e parâmetros são mais ou menos os mesmos em ambas as linguagens.

## A Grande Diferença — Ponteiros Explícitos

Uma característica de C que Java e Python não têm são os **ponteiros explícitos**.

> 💡 **O que é um ponteiro?** Um ponteiro é uma variável que **guarda um endereço de memória** ao invés de um valor diretamente. Lembra que cada byte da RAM tem um endereço único? Um ponteiro é uma variável cujo conteúdo é um desses endereços — ele "aponta" para onde outro dado está armazenado na memória.
> 

> Uma analogia: imagine que a memória RAM é um prédio com milhares de apartamentos numerados. Uma variável normal é um apartamento que guarda um objeto dentro. Um ponteiro é um papel que tem escrito o **número do apartamento** onde o objeto está — ele não guarda o objeto, guarda o *endereço* de onde encontrá-lo.
> 

## Por que ponteiros existem?

Em linguagens como Java e Python, quando você passa um objeto para uma função, a linguagem gerencia as referências automaticamente nos bastidores. Em C, você faz isso explicitamente com ponteiros. Isso permite:

- **Modificar variáveis fora do escopo atual** — uma função pode modificar uma variável de outra função passando seu endereço
- **Trabalhar com estruturas grandes de forma eficiente** — em vez de copiar um array de 1 MB para uma função, passa-se apenas o endereço (8 bytes)
- **Alocar memória dinamicamente** — `malloc()` retorna o endereço do bloco alocado no heap
- **Acessar hardware diretamente** — em SOs, endereços de hardware são acessados via ponteiros

## Os Operadores de Ponteiro

| Operador | Nome | O que faz |
| --- | --- | --- |
| `&` | "endereço de" | Retorna o endereço de memória de uma variável |
| `*` | "conteúdo de" | Retorna o valor armazenado no endereço apontado |

Considere as instruções:

```c
char c1, c2, *p;
c1 = 'c';
p = &c1;
c2 = *p;
```

**Linha a linha:**

- `char c1, c2, *p` — declara `c1` e `c2` como variáveis de caracteres e `p` como sendo uma variável que aponta para um caractere (`*p` indica que `p` é um ponteiro para `char`)
- `c1 = 'c'` — armazena o código ASCII para o caractere "c" na variável `c1`
- `p = &c1` — designa o **endereço de** `c1` para o ponteiro `p`. Agora `p` não contém 'c', contém o endereço onde 'c' está guardado
- `c2 = *p` — lê o **conteúdo do endereço apontado por** `p` e copia para `c2`. Como `p` aponta para `c1`, e `c1` contém 'c', então `c2` recebe 'c'

```
Memória (endereços fictícios):

  Endereço  Conteúdo  Variável
  ────────  ────────  ────────
  0x1000      'c'      c1      ← p aponta para aqui
  0x1001     0x1000    p       ← guarda o endereço de c1, não 'c'
  0x1002      'c'      c2      ← recebeu o valor lido via p

p aponta para c1:
p → [0x1000] → 'c'
*p = conteúdo do endereço 0x1000 = 'c'
```

Após esses comandos, `c2` também contém o código ASCII para "c".

> ⚠️ Na teoria, ponteiros são divididos em tipos — assim não se supõe que você vá designar o endereço de um número em ponto flutuante a um ponteiro de caractere. Na prática, porém, compiladores aceitam tais atribuições, embora algumas vezes com um aviso. **Ponteiros são uma construção muito poderosa, mas também uma grande fonte de erros quando usados de modo descuidado.**
> 

## O que C Não Tem

Algumas coisas que C **não** tem incluem:

| Recurso | C tem? | Impacto |
| --- | --- | --- |
| **Strings incorporadas** | ❌ | Strings são arrays de char terminados em `\0` |
| **Threads** | ❌ | Precisam de bibliotecas externas (pthreads) |
| **Pacotes** | ❌ | Organização via arquivos `.h` e diretórios |
| **Classes e objetos** | ❌ | Sem orientação a objetos (C++ tem) |
| **Segurança de tipos** | ❌ | Ponteiros permitem acesso arbitrário à memória |
| **Coleta de lixo (GC)** | ❌ | Gerenciamento de memória é manual — `malloc`/`free` |

**A ausência de GC é especialmente importante** — é a segunda propriedade, junto com ponteiros explícitos, que torna C atraente para a escrita de sistemas operacionais. Todo armazenamento em C é estático ou explicitamente alocado e liberado pelo programador, normalmente com as funções de biblioteca `malloc` e `free`. É a segunda propriedade — **controle total do programador sobre a memória** — junto com ponteiros explícitos que torna C atraente para a escrita de sistemas operacionais.

---

# ✅ Resumo do Conceito

- **Sistemas operacionais são escritos em C** (ou C++) porque são sistemas em tempo real — interrupções exigem resposta em microssegundos, e **a entrada imprevisível de um garbage collector é intolerável**
- C tem as mesmas estruturas de controle e tipos primitivos que Java e Python, mas difere fundamentalmente em dois aspectos
- **Ponteiros explícitos** — variáveis que guardam endereços de memória. Operador `&` = "endereço de", operador `*` = "conteúdo do endereço". Extremamente poderosos e extremamente perigosos
- **Sem garbage collector** — toda memória é gerenciada manualmente com `malloc`/`free`. Isso dá controle total ao programador — essencial para SO — mas exige disciplina para evitar memory leaks e dangling pointers
- C não tem: strings incorporadas, threads nativas, pacotes, classes, segurança de tipos nem GC — tudo isso torna a linguagem mais baixo nível e adequada para programação de sistemas