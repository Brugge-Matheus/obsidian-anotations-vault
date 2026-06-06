---
tags:
  - algoritmos
  - complexidade
  - fundamentos
---

# Complexidade de Algoritmos

**Complexidade de algoritmos** é a análise de quanto **tempo** e **memória** um algoritmo consome em função do tamanho da entrada. Permite comparar eficiência de soluções antes mesmo de executá-las.

> Resumo: mede a **velocidade e eficiência** de um algoritmo — não em segundos, mas em número de operações em relação ao tamanho da entrada.

---

## Por que analisar complexidade?

- **Eficiência**: algoritmos com menor complexidade são mais rápidos para entradas grandes
- **Escalabilidade**: um algoritmo O(n²) pode ser aceitável para 100 elementos, mas catastrófico para 1 milhão
- **Recursos**: estima tempo de CPU e uso de RAM antes de colocar em produção

---

## Como medir?

Usamos a [[Big O Notation]], que descreve o **comportamento assintótico** — como o algoritmo se comporta quando a entrada cresce indefinidamente.

### Três passos para calcular

1. **Identificar repetições**: loops são o que mais importa
2. **Verificar funções da linguagem**: `.sort()`, `.includes()` têm complexidade própria
3. **Descartar constantes e termos menores**: `3n² + 2n + 5` → O(n²)

### Exemplo prático

```go
// O(n²) — dois loops aninhados sobre o mesmo array
func encontrarPares(nums []int, alvo int) [][2]int {
    var pares [][2]int
    for i := 0; i < len(nums); i++ {        // n iterações
        for j := i + 1; j < len(nums); j++ { // n iterações
            if nums[i]+nums[j] == alvo {
                pares = append(pares, [2]int{nums[i], nums[j]})
            }
        }
    }
    return pares
}
```

---

## Complexidade de Tempo vs Espaço

| Tipo | O que mede | Exemplo |
|------|-----------|---------|
| **Temporal** | Número de operações | Busca binária: O(log n) |
| **Espacial** | Memória extra usada | Merge Sort: O(n) auxiliar |

---

## Conexão com Sistemas Operacionais

- O SO também analisa complexidade ao escolher [[Algoritmos de Busca]] e [[Algoritmos de Ordenação]] internos — por exemplo, o scheduler do Linux usa O(log n) com red-black trees para gerenciar processos
- **Page fault handling**: O SO busca a página em tabelas — estruturas como [[B-Tree]] ou [[Hashmap]] determinam o custo dessa busca
- **Virtual memory**: endereçamento de páginas usa aritmética O(1) — a mesma lógica de acesso a [[Arrays]]
- O **sistema de arquivos** (ext4, NTFS) organiza metadados em [[B-Tree]] justamente porque minimiza acessos a disco (O(log_m n))

## Conexão com Go

- O compilador Go analisa complexidade ao otimizar código — loops com complexidade alta recebem dicas de inline ou escape analysis
- `sort.Slice()` no pacote `sort` usa **pdqsort** (pattern-defeating quicksort), O(n log n) no caso médio
- `map` em Go tem complexidade amortizada O(1) — implementação de [[Hashmap]] com rehashing automático
- Goroutines têm overhead O(1) para criação — o runtime do Go usa estruturas de dados eficientes para o scheduler
- **Profiling**: `pprof` no Go mostra onde seu programa passa mais tempo — diretamente relacionado à complexidade dos algoritmos usados
