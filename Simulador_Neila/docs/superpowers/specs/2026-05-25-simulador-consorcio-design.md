# Simulador de Consórcio — Design Spec

**Data:** 2026-05-25  
**Projeto:** Simulador de Alavancagem Financeira: Consórcio vs CDB/Poupança  
**Responsável:** Neila (usuária final) / Ademilton (dev)  
**Status:** Aprovado ✅

---

## 1. Contexto & Objetivo

Ferramenta web interativa que demonstra, de forma visual e clara, a vantagem estratégica da **alavancagem via consórcio** em comparação com CDB e Poupança ao longo de 24 meses.

**Propósito duplo:**
- **Ferramenta de venda:** Neila apresenta ao vivo para o cliente (notebook/celular)
- **Self-service:** Cliente acessa o link sozinho e explora os valores

**Sucesso:** O cliente vê o gráfico, entende o "salto" do mês 12, e solicita atendimento com a Neila.

---

## 1.1 Configuração Necessária Antes do Deploy

| Constante | Arquivo | Descrição |
|---|---|---|
| `VITE_NEILA_WHATSAPP` | `.env` | Número da Neila com DDI: ex. `5519999999999` |
| `VITE_NEILA_NAME` | `.env` | Nome exibido no header e na mensagem de lead: ex. `"Neila Souza"` |
| `VITE_HERO_TAGLINE` | `.env` | Frase do header hero: ex. `"Descubra o poder do consórcio"` |

Esses valores ficam em `.env` (não versionado) e são injetados no build pelo Vite.

---

## 2. Stack Técnica

| Camada | Tecnologia |
|---|---|
| Framework | Vite + React + TypeScript |
| Estilo | Tailwind CSS |
| Gráfico | Chart.js (via react-chartjs-2) |
| Testes | Vitest |
| PDF export | html2canvas + jsPDF |
| Deploy | Agnóstico — Vercel (git import) ou VPS (nginx / Coolify / Dokploy) |
| Build output | `dist/` estático (zero backend) |
| Skill de UI | `frontend-design` (fase de implementação) |

---

## 3. Modelo Financeiro

### 3.1 Entrada

| Campo | Tipo | Validação |
|---|---|---|
| Valor Mensal (`V`) | number | Mínimo R$ 269,00 |

### 3.2 Parâmetros (defaults — modo avançado permite alterar)

| Parâmetro | Valor padrão |
|---|---|
| Taxa Poupança (`p`) | 0,67% ao mês |
| Taxa CDB líquido (`c`) | 1,00% ao mês |
| Selic/CDI (exibição) | 14,50% a.a. |
| Multiplicador de crédito | 100× → `Crédito = 100 × V` |
| Mês da contemplação | 12 |
| Ágio na venda | 30% do crédito |

### 3.3 Fórmulas (aporte no fim do mês, juros sobre saldo anterior)

**Poupança** (meses 1–24):
```
Saldo[0] = 0
Saldo[m] = Saldo[m-1] × (1 + 0,0067) + V
```

**CDB** (meses 1–24):
```
Saldo[0] = 0
Saldo[m] = Saldo[m-1] × (1 + 0,01) + V
```

**Alavancagem — Estratégia de Consórcio:**
```
Crédito = 100 × V
Ágio    = 0,30 × Crédito = 30 × V

Meses 1–11:   Saldo[m] = m × V          (capital aportado, sem rendimento)
Mês 12:       Saldo[12] = 30 × V        (recebe ágio à vista; parcelas anteriores absorvidas)
Meses 13–24:  Saldo[m] = Saldo[m-1] × (1 + 0,01) + V
```

> **Nota:** As parcelas pagas nos meses 1–11 são absorvidas na venda da carta — o cliente sai com os 30×V em caixa, não com "parcelas + 30%". O gráfico mostra essa transição de forma clara.

### 3.4 Resumo Mês 24

| Métrica | Fórmula |
|---|---|
| Ganho vs CDB (R$) | `Alavancagem[24] − CDB[24]` |
| Ganho vs CDB (%) | `(Alavancagem[24] / CDB[24] − 1) × 100` |
| Ganho vs Poupança (R$) | `Alavancagem[24] − Poupança[24]` |
| Ganho vs Poupança (%) | `(Alavancagem[24] / Poupança[24] − 1) × 100` |

### 3.5 Disclaimer obrigatório

> *"Simulação ilustrativa. A contemplação no mês 12 e o ágio de 30% são premissas de cálculo, não garantias. Rentabilidades passadas não asseguram resultados futuros. Consulte um especialista antes de investir."*

---

## 4. Telas & UX

### 4.1 Estrutura — Página Única com Scroll (mobile-first)

```
┌─────────────────────────┐
│         Header          │  Nome/tagline da Neila (sem logo Ademicon)
├─────────────────────────┤
│    Hero + Input         │  Campo grande "Quanto quer guardar/mês?"
│    Botão "Simular"      │  Gradiente #CE2616 → #FE7236
├─────────────────────────┤
│    Gráfico de linhas    │  3 linhas × 24 meses (aparece ao simular)
├─────────────────────────┤
│    3 Cards de saldo     │  Poupança | CDB | Alavancagem (mês 24)
├─────────────────────────┤
│    Box Ganho Excedente  │  Destaque com gradiente: +R$ X.XXX vs CDB/Poupança
├─────────────────────────┤
│    Tabela detalhada     │  Colapsável — mês a mês os 3 cenários
├─────────────────────────┤
│    Botões de ação       │  Compartilhar WhatsApp | Exportar PDF
├─────────────────────────┤
│    Disclaimer           │  Rodapé fixo em resultados
├─────────────────────────┤
│    Engrenagem (⚙)       │  Painel Avançado (Neila) — acesso discreto no rodapé
└─────────────────────────┘
```

### 4.2 Modal de Captura de Lead

- Aparece **uma única vez**, quando o usuário clica em "Simular" e os resultados são exibidos pela primeira vez (controle via `sessionStorage` para não repetir na mesma sessão)
- Campos: **Nome** + **WhatsApp**
- CTA: "Enviar minha simulação para a Neila"
- Ação: abre `wa.me/${VITE_NEILA_WHATSAPP}?text=[mensagem_formatada]`
- Mensagem inclui: nome do cliente, V, saldo mês 24 dos 3 cenários, ganho excedente

### 4.3 Painel Avançado (Neila)

- Acesso: ícone ⚙ no rodapé (sem senha — discrição por ocultação)
- Campos editáveis: taxa poupança, taxa CDB, multiplicador de crédito, mês contemplação, % ágio, Selic exibida
- Botão "Restaurar padrões"
- Alterações se refletem no gráfico em tempo real

---

## 5. Arquitetura de Código

```
src/
├── lib/
│   └── simulator.ts          ← PURO: sem efeitos colaterais, 100% testado
├── components/
│   ├── SimulatorInput.tsx    ← campo V + validação R$269
│   ├── ResultsChart.tsx      ← Chart.js wrapper (3 linhas, 24 pontos)
│   ├── ScenarioCards.tsx     ← 3 cards de saldo final
│   ├── GainBox.tsx           ← destaque ganho excedente
│   ├── DetailTable.tsx       ← tabela colapsável mês a mês
│   ├── LeadModal.tsx         ← captura nome/WA + link WhatsApp
│   ├── ShareButtons.tsx      ← WhatsApp share + PDF export
│   ├── AdvancedPanel.tsx     ← parâmetros avançados para Neila
│   └── Disclaimer.tsx        ← texto obrigatório
├── hooks/
│   └── useSimulation.ts      ← estado derivado: V + params → séries de dados
├── utils/
│   ├── whatsapp.ts           ← formata URL wa.me com resumo da simulação
│   ├── pdf.ts                ← html2canvas + jsPDF
│   └── formatters.ts         ← formatBRL(), formatPct(), etc.
└── App.tsx                   ← layout + estado global
```

### 5.1 Estado Global (sem Redux)

```typescript
interface SimulatorState {
  V: number                    // valor mensal
  params: SimulatorParams      // taxas, mês contemplação, ágio, multiplicador
  showLeadModal: boolean
  advancedOpen: boolean
}

interface SimulatorParams {
  ratePoupanca: number         // 0.0067
  rateCDB: number              // 0.01
  creditMultiplier: number     // 100
  contemplationMonth: number   // 12
  agioPercent: number          // 0.30
  selicDisplay: number         // 14.50
}
```

### 5.2 Interface do Módulo `simulator.ts`

```typescript
type Series = number[]   // índice 0 = mês 1, índice 23 = mês 24

export function calcPoupanca(V: number, params: SimulatorParams): Series
export function calcCDB(V: number, params: SimulatorParams): Series
export function calcAlavancagem(V: number, params: SimulatorParams): Series
export function calcSummary(results: SimulationResults): Summary

interface SimulationResults {
  poupanca: Series
  cdb: Series
  alavancagem: Series
}

interface Summary {
  gainVsCDB_BRL: number
  gainVsCDB_PCT: number
  gainVsPoupanca_BRL: number
  gainVsPoupanca_PCT: number
}
```

---

## 6. Identidade Visual

### 6.1 Paleta (inspirada na Ademicon, sem vínculo direto)

| Papel | Hex | Uso |
|---|---|---|
| Primária A (vermelho) | `#CE2616` | Gradiente início |
| Primária B (laranja) | `#FE7236` | Gradiente fim, linha Alavancagem |
| Fundo principal | `#FFFFFF` | Background geral |
| Fundo cards | `#F8F8F8` | Cards neutros |
| Texto principal | `#1A1A1A` | Headings, valores |
| Texto secundário | `#6B7280` | Labels, disclaimer |
| Linha CDB | `#3B82F6` | Azul médio |
| Linha Poupança | `#9CA3AF` | Cinza muted |

**Gradiente primário:** `linear-gradient(90deg, #CE2616, #FE7236)`  
→ Aplicado em: botão CTA, box ganho excedente, accent hero

### 6.2 Tipografia

| Uso | Fonte | Peso |
|---|---|---|
| Títulos e headings | Ubuntu | Bold (700) |
| Corpo e labels | Ubuntu | Regular (400) |
| Valores monetários | Ubuntu | Bold + tabular-nums |
| (fallback sistema) | sans-serif | — |

**Import:** Google Fonts — `Ubuntu:wght@400;700`

### 6.3 Gráfico

- Tipo: Line chart, bordas suavizadas (`tension: 0.3`)
- Linha Alavancagem: `#FE7236`, `borderWidth: 3`, ponto de destaque no mês 12
- Linha CDB: `#3B82F6`, `borderWidth: 2`
- Linha Poupança: `#9CA3AF`, `borderWidth: 2`
- Grid: cinza claro, tooltip formatado em R$
- Animação de entrada ao simular (impacto visual)

---

## 7. Funcionalidades de Saída

### 7.1 Compartilhar no WhatsApp

```
URL: https://wa.me/${VITE_NEILA_WHATSAPP}?text=[encoded]

Mensagem:
Olá! Acabei de simular R$ {V}/mês no Simulador de Alavancagem.

📊 Resultado em 24 meses:
💰 Poupança:      R$ {poupanca_24}
📈 CDB:           R$ {cdb_24}
🚀 Alavancagem:   R$ {alav_24}

Ganho extra vs CDB: +R$ {gain_cdb} (+{gain_cdb_pct}%)
```

### 7.2 Lead → WhatsApp da Neila

Mesma URL wa.me, com nome do cliente prefixado:
```
Olá Neila! Meu nome é {nome} e simulei R$ {V}/mês...
```

### 7.3 Exportar PDF

- Captura a seção de resultados com `html2canvas`
- Gera PDF A4 paisagem com `jsPDF`
- Nome do arquivo: `simulacao-consorcio-{V}.pdf`

---

## 8. Testes

| Módulo | Cobertura | Casos |
|---|---|---|
| `simulator.ts` | 100% | V=269 (mínimo), V=1000, V=5000 |
| Mês 11 Alavancagem | = 11×V | Verificado |
| Mês 12 Alavancagem | = 30×V | Verificado (ágio = 30% × 100V) |
| Mês 24 CDB > Poupança | sempre | Verificado |
| Mês 24 Alavancagem > CDB | sempre (com params padrão) | Verificado |
| Validação V < 269 | bloqueia | Verificado |

---

## 9. Deploy

| Plataforma | Método |
|---|---|
| Vercel | Import GitHub repo → zero config |
| VPS | `npm run build` → servir `dist/` com nginx |
| VPS (auto-deploy) | Coolify ou Dokploy apontando pro repo |

---

## 10. O que NÃO está no escopo (v1)

- Logo da Ademicon ou menção de "consultor autorizado"
- Backend próprio / banco de dados de leads
- Multi-idioma
- Comparação com mais de 3 cenários
- Autenticação no painel avançado (protegido por discrição)
