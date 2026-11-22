---
tags:
  - c
  - c/funcoes
source: "Padrão ISO C (C17/C23: funções, protótipos, passagem de argumentos, linkagem); experimentos com gcc 13.3 em Linux x86-64"
---
# Funções em C

📚 **Referência:** Padrão ISO C (C17/C23) e experimentos reais com `gcc` 13.3 (`-S`, `nm`, `-O0` e `-O2`) | Anterior: [[Strings em C]] | Próxima: [[Recursão em C]] | Compare com [[Funções — Fundamentos]] (Go)

---

# 🧩 O que é uma função

> 💡 **Função:** um bloco de código **com nome**, que recebe **valores de entrada** (parâmetros), executa e pode devolver **um valor** de saída. Serve para **reutilizar** código, **dividir** um problema em partes e **dar nome** ao que o trecho faz.

```c
int soma(int a, int b) {       // tipo de retorno, nome, parâmetros
    return a + b;               // corpo: um bloco (veja Sintaxe, Blocos e Escopo)
}
```

| Parte | Exemplo | Observação |
|---|---|---|
| **Tipo de retorno** | `int` | `void` se não devolve nada ([[Tipos de Dados em C]]) |
| **Nome** | `soma` | segue as convenções de [[Convenções e Estilo]] (`snake_case`, verbo) |
| **Parâmetros** | `int a, int b` | **cada um precisa do seu tipo** (`int a, b` é erro) |
| **Corpo** | `{ ... }` | um bloco; tem escopo próprio ([[Sintaxe, Blocos e Escopo em C]]) |
| **Chamada** | `soma(2, 3)` | os valores passados são os **argumentos** |

> 💡 **Parâmetro × argumento:** *parâmetro* é a variável na **definição** (`a`, `b`); *argumento* é o valor **passado na chamada** (`2`, `3`).

---

# 📜 Declaração, protótipo e definição

O compilador lê o arquivo **de cima para baixo**, então precisa **conhecer** a função antes da primeira chamada. Daí a distinção:

| Termo | O que é | Exemplo |
|---|---|---|
| **Protótipo** (*declaração*) | só o **cabeçalho**, terminado por `;`: nome, parâmetros e retorno | `int soma(int a, int b);` |
| **Definição** | o cabeçalho **com o corpo** | `int soma(int a, int b) { ... }` |

O protótipo permite chamar a função **antes** de defini-la (ou definida em **outro arquivo**, tema da modularização) e deixa o compilador **verificar** os argumentos: número, tipos e conversões automáticas ([[Conversões de Tipo em C]]). É a mesma ideia de declaração × definição de [[Variáveis e Constantes em C]].

## Chamar sem declarar

Chamar `soma` **antes** de ela existir, sem protótipo, mostra o perigo. Com `-std=c17` o `gcc` emitiu apenas um **aviso** (`-Wimplicit-function-declaration`: *implicit declaration of function 'soma'*), e o compilador **supõe** que a função devolve `int` e aceita qualquer argumento. Isso é regra do C89, **removida** no C99; compiladores mais novos já a tratam como **erro**. **Sempre declare antes de usar.**

## `()` × `(void)`: uma pegadinha histórica

| Declaração | Em C17 e anterior | Em C23 |
|---|---|---|
| `int f(void)` | **nenhum** parâmetro | nenhum |
| `int f()` | parâmetros **não especificados** (aceita qualquer coisa) | **igual a `(void)`** |

Teste real: com `int vazio() {...}`, a chamada `vazio(1, 2, 3)` **compilou sem aviso** em `-std=c17`, mas em `-std=c2x` deu `error: too many arguments`. Regra: **use sempre `(void)`** para "sem parâmetros", como em `int main(void)` ([[Convenções e Estilo]]).

---

# 📦 Passagem de argumentos: sempre por valor

Cada parâmetro é uma **variável local** da função, **inicializada com uma cópia** do argumento. Modificá-la **não afeta** o original (testado em [[Ponteiros em C]]: `falha_copia(m)` deixou `m = 1`). Duas consequências:

- Para a função **alterar** o dado do chamador, passe o seu **endereço** ([[Ponteiros em C]]);
- Para **arrays**, a "cópia" é só o **endereço do primeiro elemento**, por isso a função altera o original e perde o tamanho: passe `size_t n` junto ([[Arrays e Ponteiros em C]]).

A **ordem** em que os argumentos são avaliados é **não especificada** (testado em [[Operadores em C]]: `gcc` e `clang` avaliaram em ordens opostas). Não escreva `f(i++, i++)`.

---

# ↩️ O retorno

- `return expr;` **encerra** a função e **devolve** o valor, **convertido** para o tipo de retorno ([[Conversões de Tipo em C]]). `return;` (sem valor) encerra uma função `void`.
- Uma função devolve **um só valor**. Para devolver vários, use **parâmetros de saída** (ponteiros, [[Ponteiros em C]]) ou, mais adiante, uma `struct`.
- **Nunca devolva o endereço de uma variável local**: ela some quando a função termina (ponteiro pendurado, [[Sintaxe, Blocos e Escopo em C]]).

> ⚠️ **Esquecer o `return` num caminho** de uma função não-`void` é **UB** se o chamador usar o valor. Teste real: `int f(int x) { if (x > 0) return 1; }` chamada com `-1` imprimiu **`1214759267`** (lixo). O `-Wall` avisa: `control reaches end of non-void function` (`-Wreturn-type`). Em `main`, o `return 0;` final é **implícito** ([[Como C Funciona]]).

---

# 📚 O que acontece numa chamada: a pilha

Cada chamada cria, na **pilha** (*stack*), um bloco chamado ***stack frame*** (quadro) com os **parâmetros**, as **variáveis locais** e o **endereço de retorno** (para onde voltar). Ao terminar, o quadro é **descartado**. É por isso que as variáveis locais **deixam de existir** no `return` ([[Sintaxe, Blocos e Escopo em C]]).

Teste real: `nivel1` chama `nivel2`, que chama `nivel3`, cada uma com uma variável local:

```
nivel1: &a = 0x7ffcde822cc4
nivel2: &b = 0x7ffcde822ca4      ← 32 bytes MAIS BAIXO
nivel3: &c = 0x7ffcde822c84      ← mais 32 bytes abaixo
```

Cada chamada **empilha** um quadro (32 bytes aqui) em endereços **cada vez menores**: no x86 a pilha **cresce para baixo**. Quando as chamadas retornam, o espaço é reaproveitado. A pilha é **limitada** (8 MiB no Linux, [[Arrays em C]]), tema de [[Recursão em C]] e de *Stack vs Heap*.

## Como os argumentos chegam (convenção de chamada)

No Linux x86-64 ([[Processadores]]; chamada *System V ABI*, citada em [[Como C Funciona]]) os primeiros argumentos inteiros vão em **registradores** (`rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`) e o **retorno** volta em `rax`/`eax`. O `gcc -O0 -S` de `int soma(int a, int b)` confirma:

```asm
soma:
    mov  DWORD PTR -4[rbp], edi    ; a chega em edi → vai para a pilha
    mov  DWORD PTR -8[rbp], esi    ; b chega em esi
    mov  edx, DWORD PTR -4[rbp]
    mov  eax, DWORD PTR -8[rbp]
    add  eax, edx                  ; resultado em eax
    ret                            ; volta ao endereço de retorno
```

E a chamada vira uma instrução **`call`** (que empilha o endereço de retorno). Com `-O2`, uma função pequena como `dobra(x) { return soma(x, x); }` **nem chama nada**: o compilador **embute** (*inline*) o corpo e o código final é só `lea eax, [rdi+rdi]` + `ret` ([[O Compilador]]).

---

# 🔒 Visibilidade: `static` e `inline`

Por padrão, uma função é **visível para todo o programa** (*ligação externa*). Isso tem uma consequência:

- **Nomes são globais e únicos:** C **não tem sobrecarga** (duas funções com o mesmo nome e parâmetros diferentes). Definir `f` em dois arquivos dá erro do *linker*: `multiple definition of 'f'` ([[Fases da Compilação]]).
- **`static`** em uma função a torna **privada ao arquivo** (*ligação interna*): outros arquivos nem a enxergam, e o nome pode se repetir. É a forma de esconder funções auxiliares. Conferido com `nm`: `publica` aparece como **`T`** (símbolo exportado) e `privada` como **`t`** minúsculo (local). A mesma lógica vale para variáveis globais ([[Variáveis e Constantes em C]]).
- **`inline`** (C99) é só uma **sugestão** ao compilador para embutir o corpo no ponto da chamada, evitando o custo do `call`. O compilador decide sozinho (o `-O2` já embute funções pequenas sem pedir). Usa-se `static inline` em funções minúsculas definidas em cabeçalhos.

O `-Wmissing-prototypes` do `gcc` (fora do `-Wall`) avisa quando uma função **exportada** não tem protótipo anterior (*no previous prototype for 'publica'*), um bom sinal de que ela devia ser `static` ou ter um protótipo num `.h`. Cabeçalhos e linkagem estão em *Arquivos de Cabeçalho* e *`static`, `extern` e Linkagem*.

---

# 🧼 Boas práticas

| Prática | Motivo |
|---|---|
| **Uma função, uma tarefa** e nome que diga qual (`calcular_media`, não `proc`) | leitura e testes |
| **Protótipo antes do uso**; `(void)` para "sem parâmetros" | o compilador confere os argumentos |
| `const T *` em parâmetros **somente leitura** | documenta e protege ([[Ponteiros em C]]) |
| Passe o **tamanho** junto de qualquer array ou buffer | o tamanho se perde na chamada |
| **Valide** ponteiros e tamanhos de entrada | C não confere nada por você |
| Evite **variáveis globais**; prefira parâmetros e retorno | funções previsíveis e testáveis |
| Funções auxiliares como **`static`** | não poluem o espaço de nomes global |
| Compile com `-Wall -Wextra` (e `-Wmissing-prototypes`) | acha `return` esquecido, parâmetro não usado, protótipo ausente |

---

# ✅ Resumo do Conceito

- **Função** = bloco nomeado com **parâmetros** (cópias dos **argumentos**) e **um** valor de retorno; cada parâmetro leva seu próprio tipo
- **Protótipo** (`tipo nome(params);`) declara; a **definição** traz o corpo; **declare antes de usar** (chamada implícita foi removida no C99)
- **`f()` ≠ `f(void)`** até o C17 (parâmetros não especificados); **sempre `(void)`**; no C23 os dois são iguais
- **Sempre por valor**; para alterar o chamador, passe **endereços**; em arrays chega só o endereço do 1º elemento
- **Retorno:** convertido ao tipo declarado; **não devolva endereço de local**; esquecer o `return` num caminho é **UB** (`-Wreturn-type`)
- Cada chamada cria um ***stack frame*** na pilha (cresce para baixo; 32 bytes por nível no teste); args em `rdi`/`rsi`... e retorno em `eax`; `-O2` pode **embutir** funções pequenas
- **Nomes são globais**, sem sobrecarga; **`static`** torna a função privada ao arquivo (`t` no `nm`); `inline` é só sugestão
- Boas práticas: uma tarefa por função, `const` em leitura, passar tamanho, evitar globais, `-Wall -Wextra`

---

# 🔗 Conexões com outras anotações

- [[Strings em C]] e [[Arrays em C]] — arrays e strings como parâmetros, `const char *` e `size_t n`
- [[Ponteiros em C]] — passagem por ponteiro e parâmetros de saída
- [[Arrays e Ponteiros em C]] — por que o array chega como endereço
- [[Sintaxe, Blocos e Escopo em C]] — escopo e tempo de vida dos parâmetros e locais
- [[Conversões de Tipo em C]] — conversão de argumentos e do valor de retorno
- [[Operadores em C]] — ordem de avaliação dos argumentos
- [[Como C Funciona]] — `main`, ABI e regiões de memória; [[Processadores]] — registradores e `call`/`ret`
- [[Fases da Compilação]] e [[O Compilador]] — `gcc -S`, linker, símbolos e *inlining*
- [[Convenções e Estilo]] — nomes, `(void)` e avisos
- [[Funções — Fundamentos]] — o equivalente em Go (vários retornos, funções como valores)
