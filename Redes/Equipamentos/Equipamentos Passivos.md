---
tags:
  - redes
  - redes/equipamentos
---

# Equipamentos Passivos

É importante diferenciar os equipamentos de rede em **passivos** e **ativos**. Os **equipamentos passivos** são aqueles que não precisam de energia elétrica para funcionar e não processam os dados — eles apenas conectam ou organizam os cabos e sinais.

Aqui está o detalhamento técnico dos principais equipamentos passivos:

---

### Equipamentos de Rede Passivos

---

### 1. Cabo de Rede (Par Trançado, Fibra Óptica, Coaxial)

- Meio físico que conecta os dispositivos.
- Não requer energia.
- Transmite sinais elétricos (cobre) ou ópticos (fibra).
- Tipos comuns:
    - Par trançado UTP/STP (Ethernet)
    - Fibra óptica (monomodo, multimodo)
    - Coaxial (TV a cabo, redes antigas)

---

### 2. Patch Panel

- Painel com várias portas RJ-45 para organizar e distribuir cabos de rede.
- Facilita a manutenção e a identificação dos cabos.
- Não processa dados, apenas conecta fisicamente os cabos.
- Usado em racks de servidores e salas de telecomunicações.

---

### 3. Conectores e Adaptadores

- **Conectores RJ-45:** Usados para crimpar cabos de par trançado.
- **Adaptadores:** Permitem conectar cabos com diferentes padrões ou tipos (ex: conversor de fibra para cobre).
- Não possuem eletrônica ativa.

---

### 4. Dutos, Canaletas e Eletrocalhas

- Estruturas físicas para organizar, proteger e conduzir os cabos.
- Mantêm a infraestrutura limpa e segura.
- Não interferem no sinal, apenas protegem o meio físico.

---

### 5. Racks e Gabinetes

- Estruturas metálicas para acomodar equipamentos de rede e servidores.
- Facilitam a organização, ventilação e segurança física.
- São passivos, pois não têm componentes eletrônicos.

---

### Resumo

| Equipamento | Função | Energia | Processamento de Dados |
| --- | --- | --- | --- |
| Cabo de Rede | Meio físico de transmissão | Não | Não |
| Patch Panel | Organização e distribuição de cabos | Não | Não |
| Conectores/Adaptadores | Conexão física entre cabos | Não | Não |
| Dutos/Canaletas | Proteção e organização dos cabos | Não | Não |
| Racks/Gabinetes | Suporte físico para equipamentos | Não | Não |

---

### Dica:

> Equipamentos passivos são essenciais para garantir a **qualidade física** da rede, mas não influenciam diretamente na velocidade ou no roteamento dos dados.

---

## Conexões com SO

- **Patch panel:** é um bloco de terminação punch-down com conectores RJ-45 fêmea na frente e fios cravados por pressão na traseira. Puramente físico — sem lógica, sem buffers, sem eletrônica. Do ponto de vista do SO, é invisível; o SO só enxerga a NIC no final do cabo — [[Dispositivos de IO]].
- **Rack:** largura padronizada em 19 polegadas; altura medida em unidades U (1U = 1,75 polegadas = 44,45 mm). Um rack de 42U comporta servidores, switches, patch panels e PDUs (distribuição de energia). Nenhuma CPU, nenhum processamento — infraestrutura puramente mecânica e elétrica — [[Dispositivos de IO]].
