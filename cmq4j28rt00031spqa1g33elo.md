---
title: "PDD na prática: do conceito ao cálculo em Python com modelo ECL"
seoTitle: "Pdd, contabilidade "
seoDescription: "Pdd, finance, accounting."
datePublished: 2026-06-08T01:23:52.942Z
cuid: cmq4j28rt00031spqa1g33elo
slug: pdd-na-pr-tica-do-conceito-ao-c-lculo-em-python-com-modelo-ecl
tags: pdd

---

# PDD na prática: do conceito ao cálculo em Python com modelo ECL

A IFRS 9 mudou a lógica da provisão de crédito de forma fundamental: saímos do modelo de perda incorrida para o modelo de perda esperada. Neste artigo implementamos o ECL completo em Python — da segmentação da carteira ao lançamento contábil.

Se você trabalha com contabilidade de instituições financeiras ou fintechs, já sabe que a **Provisão para Devedores Duvidosos (PDD)** é um dos lançamentos mais sensíveis do balanço. Ela afeta diretamente o resultado, o patrimônio líquido e os indicadores de capital.

O que muita gente ainda não domina é a diferença entre o modelo antigo — baseado na CMN 2.682 — e o modelo IFRS 9, que exige uma lógica completamente diferente de cálculo.

Vamos resolver isso com código.

---

## 1. O problema com o modelo antigo

A **CMN Resolução 2.682** usa um modelo de perda incorrida: você só provisiona quando o cliente já está em atraso. A lógica é reativa — o problema precisa ter acontecido para gerar provisão.

> ⚠️ **O problema central:** no modelo incorrido, uma carteira com histórico ruim pode estar sub-provisionada porque as perdas futuras já previsíveis não são capturadas. O IFRS 9 corrige exatamente isso.

A provisão pela CMN 2.682 é mecânica: cada faixa de atraso tem um percentual mínimo definido em tabela. Simples de calcular, mas cego ao comportamento futuro do portfólio.

---

## 2. O que muda com a IFRS 9

A **IFRS 9 / CPC 48** introduz o modelo de **Perda de Crédito Esperada (ECL — Expected Credit Loss)**. A lógica agora é prospectiva: você provisiona com base na probabilidade de perda futura, mesmo que o cliente ainda não tenha atrasado.

O modelo ECL tem três componentes principais:

```
ECL = PD × LGD × EAD

PD  = Probability of Default (probabilidade de inadimplência)
LGD = Loss Given Default (perda dado o default)
EAD = Exposure at Default (saldo exposto no momento do default)
```

O cálculo varia por **estágio**, determinado pela deterioração de crédito desde a originação:

| Estágio | Situação | Horizonte ECL |
|---|---|---|
| **Stage 1** | Sem deterioração significativa | 12 meses |
| **Stage 2** | Deterioração significativa | Lifetime |
| **Stage 3** | Default confirmado (≥ 90 DPD) | Lifetime integral |

> ℹ️ **DPD (Days Past Due):** dias em atraso. É o indicador mais comum para classificação de estágio no modelo simplificado. O threshold de 90 DPD para Stage 3 é referência da IFRS 9, mas cada instituição pode adotar critérios mais conservadores.

---

## 3. Implementando o modelo ECL em Python

Vamos construir o modelo do zero com **pandas** e **numpy**. O exemplo usa uma carteira simulada de crédito pessoal.

### 3.1 Estrutura da carteira

```python
import pandas as pd
import numpy as np

# Carteira simulada de crédito pessoal
data = {
    'id_contrato':   ['C001', 'C002', 'C003', 'C004', 'C005'],
    'saldo_devedor': [15000,   8500,  32000,   4200,  21000],
    'dpd':           [0,       45,    120,      0,     75],   # dias em atraso
    'score':         [720,     580,   420,     810,    610],
}

df = pd.DataFrame(data)
```

### 3.2 Classificação por estágio

```python
# Classificação por estágio IFRS 9 (modelo simplificado DPD)
def classify_stage(dpd):
    if dpd == 0:
        return 1      # Sem deterioração
    elif dpd <= 90:
        return 2      # Deterioração significativa
    else:
        return 3      # Default

df['stage'] = df['dpd'].apply(classify_stage)
```

### 3.3 Parâmetros PD, LGD e EAD

> 💡 Neste exemplo usamos parâmetros fixos por estágio. Em produção, esses parâmetros são calibrados com modelos estatísticos usando histórico da carteira.

```python
# Parâmetros ECL por estágio (exemplo simplificado)
params = {
    1: {'pd': 0.02,  'lgd': 0.45},   # Stage 1: PD 12 meses
    2: {'pd': 0.15,  'lgd': 0.55},   # Stage 2: PD lifetime
    3: {'pd': 1.00,  'lgd': 0.70},   # Stage 3: default confirmado
}

# EAD = saldo devedor (simplificado — sem CCF)
df['ead'] = df['saldo_devedor']
df['pd']  = df['stage'].map(lambda s: params[s]['pd'])
df['lgd'] = df['stage'].map(lambda s: params[s]['lgd'])
```

### 3.4 Cálculo do ECL e da PDD

```python
# Cálculo do ECL
df['ecl'] = df['ead'] * df['pd'] * df['lgd']

# Resumo por estágio
resumo = df.groupby('stage').agg(
    contratos   =('id_contrato',   'count'),
    saldo_total =('saldo_devedor', 'sum'),
    pdd_total   =('ecl',           'sum')
).reset_index()

resumo['cobertura_pct'] = (
    resumo['pdd_total'] / resumo['saldo_total'] * 100
).round(2)

print(resumo)
```

### 3.5 Resultado esperado

| Stage | Contratos | Saldo Total (R$) | PDD / ECL (R$) | Cobertura % |
|---|---|---|---|---|
| 1 | 2 | 19.200,00 | 172,80 | 0,90% |
| 2 | 2 | 29.500,00 | 2.681,25 | 9,09% |
| 3 | 1 | 32.000,00 | 22.400,00 | 70,00% |

> ✅ **Interpretação:** o Stage 3 representa apenas 1 contrato mas concentra a maior parte da PDD — R$ 22.400 de um total de R$ 25.254. Isso é esperado: clientes em default têm PD de 100% e LGD elevada.

---

## 4. O lançamento contábil da PDD

Com o ECL calculado, o lançamento segue a lógica de constituição da provisão:

```python
# Lançamento contábil da PDD (IFRS 9)
# Débito:  Despesa com PDD (resultado)
# Crédito: Provisão para Perdas Esperadas (ativo redutor)

pdd_total = df['ecl'].sum()

lancamento = {
    'debito': {
        'conta':    'Despesa com Perdas de Crédito Esperadas',
        'natureza': 'Resultado',
        'valor':    round(pdd_total, 2)
    },
    'credito': {
        'conta':    'Provisão para Perdas de Crédito Esperadas',
        'natureza': 'Ativo (redutor)',
        'valor':    round(pdd_total, 2)
    }
}

print(f"PDD total constituída: R$ {pdd_total:,.2f}")
```

> ℹ️ **Ativo redutor:** a Provisão para Perdas fica no ativo como conta redutora da carteira de crédito — não vai para o passivo. No balanço, o valor líquido da carteira já aparece deduzido da provisão constituída.

---

## 5. CMN 2.682 vs IFRS 9 — resumo prático

| Critério | CMN 2.682 | IFRS 9 / CPC 48 |
|---|---|---|
| Modelo | Perda incorrida | **Perda esperada** |
| Gatilho | Atraso observado | **Deterioração de crédito** |
| Cliente em dia | Mínimo (0–3%) | **PD × LGD × EAD (Stage 1)** |
| Horizonte | Curto prazo | **12 meses ou lifetime** |
| Parâmetros | Tabela regulatória | **Calibração interna** |
| Complexidade | Baixa | **Alta — requer modelagem** |

---

## Conclusão

- O modelo ECL da IFRS 9 é **prospectivo**: provisiona perda futura esperada, não apenas perda já incorrida
- A classificação por estágio (1, 2, 3) determina o horizonte de cálculo da PDD
- **ECL = PD × LGD × EAD** — três parâmetros que precisam ser calibrados com dados históricos
- Em Python, o cálculo é direto com pandas — a complexidade real está na calibração dos parâmetros
- O lançamento contábil constitui a PDD como **ativo redutor**, impactando o resultado do período

No próximo artigo vamos aprofundar a **calibração do parâmetro PD** usando curvas de sobrevivência com dados de vintage — a base quantitativa que sustenta qualquer modelo ECL robusto.

---

📂 **Notebook completo disponível no GitHub:**
[github.com/wallacecurto-arch/santo-dado](https://github.com/wallacecurto-arch/santo-dado/tree/main/notebooks)

---

*Wallace Curto — Contador | Accounting Specialist Fintech*
*Blog: [santodado.hashnode.dev](https://santodado.hashnode.dev)*
