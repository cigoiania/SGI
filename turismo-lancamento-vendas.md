# Turismo — Lógica de Lançamento de Vendas
### Controle · Conferência · Conciliação

> **O que é este documento:** o detalhamento **lógico e funcional** do
> **SGI-6** (módulo de Pós-vendas / aba Turismo) descrito em `ajustes-sgi.md`.
> O SGI-6 diz **o que** o módulo precisa ter (campos, telas, relatórios). Este
> arquivo diz **como a informação se comporta**: entidades, cadeia de cálculo,
> estados, travas, checagens automáticas e regras de conciliação.
>
> **Para quem:** time de dev (implementação) e time financeiro/operacional
> (validação das regras de negócio).
>
> **Status:** 🟡 **Em construção conjunta.** O corpo do documento já está
> fechado o suficiente para orçar/implementar; os pontos que dependem de
> decisão do cliente estão isolados na seção **14. Decisões pendentes** —
> cada um com uma **sugestão de padrão** para destravar a implementação.

---

## Índice

1. [Por que esta camada existe](#1-por-que-esta-camada-existe)
2. [Princípios (regras de ouro)](#2-princípios-regras-de-ouro)
3. [Modelo de dados](#3-modelo-de-dados)
4. [Cadeia de cálculo](#4-cadeia-de-cálculo)
5. [Ciclo de vida e status](#5-ciclo-de-vida-e-status)
6. [Fluxo de lançamento, passo a passo](#6-fluxo-de-lançamento-passo-a-passo)
7. [Conferência (dupla checagem)](#7-conferência-dupla-checagem)
8. [Conciliação](#8-conciliação)
9. [Painel de divergências](#9-painel-de-divergências)
10. [Cancelamento, reembolso e estorno](#10-cancelamento-reembolso-e-estorno)
11. [Fechamento de período](#11-fechamento-de-período)
12. [Relatórios](#12-relatórios)
13. [Auditoria e permissões](#13-auditoria-e-permissões)
14. [Decisões pendentes](#14-decisões-pendentes)
15. [Glossário](#15-glossário)

---

## 1. Por que esta camada existe

O SGI-6 descreve o lançamento como **um registro plano** ("uma venda = uma
linha com custo, venda e comissão"). Isso resolve o cadastro, mas **não
sustenta controle, conferência nem conciliação**, por cinco motivos:

| # | Problema do registro plano | Consequência prática |
|---|---------------------------|----------------------|
| 1 | Uma viagem costuma ter **vários fornecedores** (hotel + transfer + passeios), cada um com custo, markup e comissão próprios. | Não dá para fechar com o fornecedor nem montar a NF de comissão por fornecedor (exigência do SGI-6 "f"). |
| 2 | "Status de pagamento" **digitado à mão** (Pago/Pendente/Parcelado...). | O status vira opinião, não fato. Ninguém confia no relatório de contas a receber. |
| 3 | Não existe o conceito de **parcela** nem de **baixa**. | Impossível saber *quanto* falta receber, *quando* vence, e conciliar com o extrato. |
| 4 | "Tudo editável" (SGI-6 "g") **sem versionamento e sem travas**. | O resultado de um mês fechado muda sozinho quando alguém corrige uma venda antiga. |
| 5 | A IA preenche campos do voucher **direto no registro final**. | Erro de leitura entra no relatório sem ninguém ter olhado. |

A lógica abaixo resolve os cinco pontos **sem mudar nada do que o SGI-6 já
definiu** — ela apenas abre o registro plano em camadas e acrescenta os
estados e checagens que faltavam.

**Resumo da mudança estrutural:**

```
SGI-6 (hoje)          →   Esta spec (proposta)

[ VENDA ]                 [ VENDA ]  ........... cliente, vendedor, datas, status
  custo                     └─ [ ITEM ] ........ 1 por voucher/fornecedor
  venda                          custo, venda, markup, comissões
  markup                    └─ [ PARCELA ] ..... a receber (cliente) / a pagar (fornecedor)
  comissão                       vencimento, valor
  status pgto (texto)            └─ [ BAIXA ] .. pagamento real, data, meio, comprovante
                                        └─ [ CONCILIAÇÃO ] .. casa com o extrato
                            └─ [ COMISSÃO ] .... a receber do fornecedor → NF → crédito
                            └─ [ LOG ] ......... quem mudou o quê, quando, de → para
```

---

## 2. Princípios (regras de ouro)

Estas oito regras valem para **todo** o módulo. Toda decisão de
implementação em caso de dúvida deve ser resolvida a favor delas.

**P1 · Nada se apaga.** Cancelamento, correção e devolução geram
**estorno/ajuste**, nunca exclusão. Registro errado é **cancelado com
motivo**, e o número do lançamento **não é reaproveitado**.

**P2 · Status financeiro é calculado, nunca digitado.** "Pago / Pendente /
Parcelado" é **consequência** das baixas lançadas (§5.3). O usuário lança
fatos (parcelas e pagamentos); o sistema deduz o status.

**P3 · Nada gerado por IA vale sem conferência humana.** Campo extraído de
voucher/NF nasce marcado como **`sugerido`** e só vira **`confirmado`**
quando uma pessoa valida (§7.1). Relatório só considera lançamento
**conferido**.

**P4 · Todo valor tem lastro documental.** Item sem voucher, pagamento sem
comprovante e comissão sem NF ficam visíveis como **pendência**, não somem.

**P5 · Parâmetro usado no cálculo é congelado (snapshot).** Se o fornecedor
mudar o prazo, ou a comissão padrão do vendedor mudar de 40% para 35%, as
vendas **já lançadas não mudam** (§4.4).

**P6 · A identidade `Venda = Custo + Markup` nunca é violada.** O sistema
sempre calcula o terceiro valor a partir de dois (§4.1) e **bloqueia** o
salvamento de uma combinação inconsistente.

**P7 · Período fechado é imutável.** Depois do fechamento do mês, correção
vira **lançamento de ajuste no período aberto**, referenciando o original
(§11).

**P8 · Toda divergência é visível.** O sistema não "conserta em silêncio":
diferença encontrada vira item no **Painel de divergências** (§9) com
responsável e prazo.

---

## 3. Modelo de dados

### 3.1 Visão geral

```
CLIENTE ──┐
VENDEDOR ─┼──> VENDA ──┬──> ITEM DA VENDA ──> FORNECEDOR
          │            │         │
          │            │         ├──> VOUCHER (anexo + campos extraídos pela IA)
          │            │         └──> COMISSÃO DO FORNECEDOR ──> NF DE COMISSÃO
          │            │
          │            ├──> PARCELA A RECEBER (cliente) ──> BAIXA ──> CONCILIAÇÃO
          │            ├──> PARCELA A PAGAR (fornecedor) ──> BAIXA ──> CONCILIAÇÃO
          │            └──> COMISSÃO DO VENDEDOR ──> BAIXA
          │
          └──> (cadastros mestres do SGI-7)
```

### 3.2 `VENDA` — cabeçalho comercial

Representa **a negociação com o cliente**. É o nível em que se fala de
"uma venda" no relatório.

| Campo | Tipo | Regra |
|-------|------|-------|
| `numero` | sequencial | **Imutável**, por ano (ex.: `2026-0417`). Não reaproveita número cancelado. |
| `data_venda` | data | **Data de competência** — é ela que rege os relatórios de dia/mês/ano (§12). |
| `data_lancamento` | data/hora | Automática, não editável. Diferente de `data_venda`. |
| `cliente_id` | FK | Obrigatório na conferência. Vem de Cadastros → Clientes (SGI-7). |
| `vendedor_id` | FK | Obrigatório na conferência. Cadastros → Vendedores (SGI-7). |
| `pct_comissao_vendedor` | % | **Snapshot** do cadastro (padrão 40%), ajustável por venda. |
| `data_embarque` / `data_retorno` | data | Controle operacional; não regem o relatório financeiro. |
| `status_venda` | enum | `Ativa` / `Cancelada` (SGI-6 "a"). |
| `status_lancamento` | enum | `Rascunho` / `Em conferência` / `Conferido` / `Fechado` (§5.1). |
| `observacao` | texto | Livre. |
| `origem_lancamento` | enum | `Manual` / `IA (voucher)` / `Importação`. Rastreabilidade. |

> Totais (custo, venda, markup, comissões) **não são campos digitados** no
> cabeçalho: são **somas dos itens** (§4.2). Digitar total e itens em
> paralelo é a origem clássica de divergência.

### 3.3 `ITEM DA VENDA` — um por serviço/voucher/fornecedor

É aqui que mora o dinheiro. **Uma venda com hotel + transfer + 2 passeios =
4 itens.**

| Campo | Tipo | Regra |
|-------|------|-------|
| `fornecedor_id` | FK | Obrigatório. Cadastros → Fornecedores (SGI-7). |
| `tipo_servico` | enum | Hospedagem / Transfer / Passeio / Seguro / Pacote / Outro. |
| `localizador` | texto | Nº/localizador da reserva (extraído do voucher). **Chave anti-duplicidade** junto com o fornecedor (§7.4). |
| `descricao` | texto | Nome do prestador local, cidade, período. |
| `qtd_pax` | inteiro | Do voucher. |
| `hospedes` | lista | Nomes do voucher; idealmente vinculados a Clientes. |
| `data_servico_inicio` / `fim` | data | Check-in/check-out, data do passeio. |
| `moeda` | enum | `BRL` (padrão) — ver decisão **Q5** (§14). |
| `valor_custo` | decimal(2) | O que se deve ao fornecedor. |
| `valor_venda` | decimal(2) | O que o cliente paga. |
| `markup` | decimal(2) | `Venda − Custo` (§4.1). |
| `pct_comissao_fornecedor` | decimal(4) | % aberto, varia por venda (SGI-6 "a"). |
| `base_comissao_fornecedor` | enum | `Custo` (sugerido) / `Venda` — ver decisão **Q1**. |
| `valor_comissao_fornecedor` | decimal(2) | Calculado (§4.1). |
| `modelo_comissao` | enum | `Repasse` (fornecedor paga depois, via NF) / `Dedução` (abatida no pagamento ao fornecedor) — ver **Q2**. |
| `status_item` | enum | `Ativo` / `Cancelado`. Permite cancelar **um serviço** sem cancelar a venda inteira. |
| `voucher_id` | FK | Anexo obrigatório para conferir (P4). |

> **Por que item, e não venda:** a NF de comissão é **por fornecedor**
> (SGI-6 "f") e o pagamento ao fornecedor tem **prazo por fornecedor**
> (SGI-7). Sem a quebra por item, uma venda com 3 fornecedores não fecha
> com nenhum dos três.

### 3.4 `PARCELA` — a receber e a pagar

Uma linha por **compromisso financeiro com vencimento**. Duas naturezas:

- `A_RECEBER` — do **cliente**, vinculada à **venda**.
- `A_PAGAR` — ao **fornecedor**, vinculada ao **item** (portanto ao fornecedor).

| Campo | Regra |
|-------|-------|
| `natureza` | `A_RECEBER` / `A_PAGAR`. |
| `numero_parcela` / `total_parcelas` | Ex.: 2/6. |
| `valor_previsto` | Σ das parcelas **tem** que bater com o valor de origem (§9 · D2/D3). |
| `data_vencimento` | Cliente: definida na negociação. Fornecedor: **calculada** pelo `prazo_pagamento` do cadastro do fornecedor (SGI-7), editável. |
| `forma_pagamento` | Pix / Cartão / Boleto / Transferência / Dinheiro / Link. |
| `taxas_previstas` | Taxa de cartão/gateway, quando houver (§4.6). |
| `status` | **Derivado** das baixas (§5.3): Em aberto / Parcial / Quitada / Vencida / Cancelada / Estornada. |

### 3.5 `BAIXA` — o pagamento que realmente aconteceu

| Campo | Regra |
|-------|-------|
| `parcela_id` | A que compromisso se refere. Uma parcela aceita **N baixas** (pagamento parcial). |
| `data_pagamento` | **Data caixa** (≠ competência). |
| `valor_bruto` | Valor da operação. |
| `taxas` | Taxa efetiva cobrada (cartão/gateway). |
| `valor_liquido` | `bruto − taxas`. É este que aparece no extrato bancário. |
| `identificador` | End-to-End do Pix, NSU do cartão, nº do documento. **Chave de conciliação** (§8.5). |
| `comprovante` | Anexo (P4). |
| `status_conciliacao` | `Não conciliada` / `Conciliada` / `Divergente`. |

### 3.6 `COMISSÃO DO FORNECEDOR` e `NF DE COMISSÃO`

Segue o SGI-6 "f": **uma NF por fornecedor**, valor = somatória das
comissões das reservas **selecionadas manualmente**.

`COMISSAO_FORNECEDOR` (1 por item ativo, quando há %):
`item_id`, `fornecedor_id`, `valor_previsto`, `nf_id` (nulo até faturar),
`valor_recebido`, `data_recebimento`, `status` (§5.4), `motivo_glosa`.

`NF_COMISSAO`:
`numero`, `serie`, `data_emissao`, `fornecedor_id`, `valor_total`,
`itens_vinculados[]` (seleção manual), `arquivo` (PDF — a IA lê e vincula,
SGI-6 "f"), `status` (Emitida / Enviada / Recebida / Recebida parcial / Cancelada),
`data_credito`, `valor_creditado`.

> **Trava:** `NF.valor_total` **precisa** ser igual à soma das comissões
> vinculadas. Diferença → divergência **D13** (§9), nunca salvamento
> silencioso.

### 3.7 `COMISSÃO DO VENDEDOR`

`venda_id`, `vendedor_id`, `pct` (snapshot), `base` (= markup), `valor_apurado`,
`status` (§5.5), `competencia`, `baixa_id` (quando paga), `valor_estornado`.

### 3.8 `LOG` — trilha de auditoria

**Toda** alteração em venda, item, parcela, baixa, comissão e NF grava:
`entidade`, `registro_id`, `campo`, `valor_anterior`, `valor_novo`,
`usuario`, `data_hora`, `origem` (`Manual` / `IA` / `Importação` / `Sistema`),
`motivo` (obrigatório quando o registro já está **Conferido**).

---

## 4. Cadeia de cálculo

### 4.1 Por item

```
(1)  markup            = valor_venda − valor_custo
(2)  valor_venda       = valor_custo + markup          [quando o markup é digitado]
(3)  base_com_forn     = valor_custo   (padrão — ver Q1)
(4)  valor_com_forn    = round(base_com_forn × pct_com_forn / 100 ; 2)
(5)  receita_agencia   = markup + valor_com_forn
```

**Regra dos 3 campos (P6):** custo, venda e markup são interdependentes.
A tela trabalha com **dois digitados + um calculado**:

| Usuário informa | Sistema calcula |
|-----------------|-----------------|
| Custo + Venda | Markup = Venda − Custo |
| Custo + Markup | Venda = Custo + Markup |
| Venda + Markup | Custo = Venda − Markup |

Se os três forem informados (ex.: importação) e não fecharem, **bloqueia**
e aponta a diferença. Nunca "ajusta" o terceiro em silêncio.

### 4.2 Por venda (consolidação)

```
custo_total       = Σ custo dos itens ATIVOS
venda_total       = Σ venda dos itens ATIVOS
markup_total      = Σ markup dos itens ATIVOS          (= venda_total − custo_total)
com_forn_total    = Σ comissão de fornecedor dos itens ATIVOS
com_vendedor      = round(markup_total × pct_comissao_vendedor / 100 ; 2)
receita_bruta     = markup_total + com_forn_total
resultado_liquido = receita_bruta − com_vendedor − taxas
```

> Itens **cancelados** saem de todos os totais e aparecem em bloco separado
> (§10), nunca misturados às vendas efetivas (exigência do SGI-6 "c").

### 4.3 Arredondamento e precisão

- Valores monetários: **2 casas**, arredondamento **meio para cima** (HALF_UP).
- Percentuais: **4 casas** (permite 8,3333%).
- **Some sempre valores já arredondados** (linha a linha), nunca arredonde a
  soma de valores cheios — é o que evita o clássico "total difere R$ 0,01 da
  soma das linhas".
- Divisão de parcelas: as parcelas recebem o valor arredondado e **a
  diferença de centavos vai na última parcela**.
  `Ex.: R$ 1.000,00 em 3× → 333,33 + 333,33 + 333,34`

### 4.4 Congelamento de parâmetros (snapshot) — P5

No momento da **conferência**, a venda grava **cópia própria** de:
`pct_comissao_vendedor`, `pct_comissao_fornecedor`, `base_comissao`,
`prazo_pagamento_fornecedor`, `modelo_comissao`.

Alterar o cadastro mestre depois **não** recalcula vendas conferidas.
Recalcular exige ação explícita ("Reaplicar parâmetros atuais"), com
permissão, motivo e log — e só é permitida em venda **não fechada**.

### 4.5 Fluxo de caixa por modelo de comissão (decisão **Q2**)

| | `Repasse` (padrão sugerido) | `Dedução` |
|---|---|---|
| A pagar ao fornecedor | `custo_total` (cheio) | `custo_total − com_forn_total` |
| A receber de comissão | `com_forn_total` (via NF, §8.4) | — (já abatida) |
| Conciliação | 2 movimentos (saída cheia + entrada da comissão) | 1 movimento (saída líquida) |

Ambos devem ser suportados — o modelo é **definido no cadastro do
fornecedor** e copiado para o item (snapshot). O **resultado** da venda é
idêntico nos dois; o que muda é o **caixa** e o que se concilia.

### 4.6 Taxas de meio de pagamento

Quando o cliente paga com cartão/gateway, **o banco credita menos do que a
venda**. Sem tratar isso, a conciliação nunca fecha.

```
valor_liquido_recebido = valor_bruto − taxa_administradora
```

Cada parcela guarda `taxas_previstas` e cada baixa guarda `taxas` efetivas.
A diferença entre previsto e efetivo vira divergência **D18** (§9).
Ver decisão **Q7** (§14): taxa absorvida pela agência (reduz resultado) ou
repassada ao cliente (aumenta o valor de venda).

---

## 5. Ciclo de vida e status

São **seis** dimensões independentes. Misturá-las num único campo "status" é
o erro mais comum — e é o que impede a conferência hoje.

### 5.1 `status_lancamento` — o eixo da conferência

```
   Rascunho ──> Em conferência ──> Conferido ──> Fechado
       ^               │               │
       └───────────────┘               │  (reabertura: permissão + motivo + log)
          (devolvido com apontamento)   └──> Reaberto ──> Conferido
```

| Status | O que significa | O que já vale |
|--------|-----------------|---------------|
| **Rascunho** | Lançado (manual ou IA), incompleto. | Não entra em relatório. Não gera parcela. |
| **Em conferência** | Enviado para validação. | Não entra em relatório. Bloqueado para o autor. |
| **Conferido** | Validado por outra pessoa (§7.3). | **Gera contas a pagar/receber**, entra nos relatórios, congela parâmetros (§4.4). |
| **Fechado** | Período encerrado (§11). | Imutável. Correção só por ajuste. |

### 5.2 `status_venda` (negócio) — SGI-6 "a"
`Ativa` (padrão) / `Cancelada` (+ reembolso sim/não + valor reembolsado, §10).

### 5.3 `status_financeiro` — **derivado**, nunca digitado (P2)

Calculado por parcela e consolidado:

```
soma_baixas = Σ baixas ativas da parcela

soma_baixas = 0            e hoje <= vencimento  → Em aberto
soma_baixas = 0            e hoje >  vencimento  → Vencida
0 < soma_baixas < previsto e hoje <= vencimento  → Parcial
0 < soma_baixas < previsto e hoje >  vencimento  → Parcial em atraso
soma_baixas >= previsto                          → Quitada
parcela cancelada                                → Cancelada
baixa revertida                                  → Estornada
```

Consolidado da venda: **Quitada** (todas quitadas) · **Parcial** (alguma
baixa) · **Em aberto** (nenhuma) · **Em atraso** (alguma vencida).

> Isso **substitui** o campo digitado "Pago / Pendente / Parcelado /
> Cancelado / Estornado" do SGI-6 "e" — mantendo exatamente a mesma lista
> de rótulos na tela, mas agora **sempre verdadeira**.

### 5.4 `status_comissao_fornecedor`

```
A faturar ──> Faturada (NF) ──> Enviada ──> Recebida
                                    │            └──> Recebida parcial (glosa, §8.4)
                                    └──> Em atraso (D+X sem crédito)
                                                 └──> Cancelada
```

### 5.5 `status_comissao_vendedor`

```
Apurada ──> Liberada ──> Paga
    │           (quando o cliente quita — ver Q3)
    └──> Estornada (cancelamento/reembolso, §10)
```

### 5.6 `status_conciliacao` (por baixa)
`Não conciliada` / `Conciliada` / `Divergente` (§8).

### 5.7 Matriz de edição — o que pode mudar em cada status

| Campo | Rascunho | Em conferência | Conferido | Fechado |
|-------|:--------:|:--------------:|:---------:|:-------:|
| Cliente / vendedor / fornecedor | ✅ | ✅ | 🔐 gestor + motivo | ❌ |
| Custo / venda / markup | ✅ | ✅ | 🔐 gestor + motivo + log | ❌ |
| % de comissão | ✅ | ✅ | 🔐 gestor + motivo | ❌ |
| Datas de serviço, observação, anexos | ✅ | ✅ | ✅ | ✅ (só anexo/observação) |
| Parcelas (criar/alterar) | — | — | ✅ financeiro | ❌ |
| Baixas | — | — | ✅ financeiro | 🔐 só estorno no período aberto |
| Cancelar a venda | ✅ | ✅ | ✅ com motivo | 🔐 gera ajuste (§11) |

✅ livre · 🔐 restrito (permissão + motivo + log) · ❌ bloqueado
(a "edição total" do SGI-6 "g" é preservada — o que se acrescenta é
**quem** pode editar e **o rastro** de cada edição).

---

## 6. Fluxo de lançamento, passo a passo

```
1. NOVO LANÇAMENTO
   └─ origem: [ Subir voucher(s) ]  ou  [ Preencher manual ]

2. LEITURA POR IA  (SGI-6 "b")
   └─ extrai: localizador, data, qtd hóspedes, prestador, cidade,
              endereço, nomes  → todos marcados como `sugerido`

3. CONFERÊNCIA DO DOCUMENTO  (§7.1)
   └─ tela lado a lado: voucher × campos extraídos
   └─ humano confirma/corrige campo a campo → `confirmado`

4. VÍNCULOS
   └─ Cliente (busca no cadastro, ou cadastra na hora — SGI-7)
   └─ Fornecedor (traz prazo de pagamento, modelo de comissão, markup padrão)
   └─ Vendedor (traz % de comissão vigente → snapshot)

5. VALORES  (§4.1)
   └─ custo + venda (ou markup) → sistema calcula o terceiro
   └─ % comissão do fornecedor → valor calculado
   └─ % comissão do vendedor (padrão 40%) → valor calculado sobre o markup

6. PLANO DE PAGAMENTO  (§3.4)
   └─ cliente: nº de parcelas, vencimentos, forma → gera parcelas A_RECEBER
   └─ fornecedor: vencimento sugerido pelo prazo do cadastro → parcelas A_PAGAR
   └─ TRAVA: Σ parcelas = valor total (D2/D3)

7. [ Salvar rascunho ]   ou   [ Enviar para conferência ]

8. CONFERÊNCIA DA VENDA  (§7)  — por outra pessoa
   └─ checklist automático + apontamentos
   └─ [ Devolver ] → volta para Rascunho com apontamento
   └─ [ Conferir ] → status CONFERIDO
        ├─ congela parâmetros (§4.4)
        ├─ publica contas a pagar/receber
        └─ apura comissão do vendedor

9. EXECUÇÃO FINANCEIRA
   └─ baixas de recebimento e pagamento (§3.5)
   └─ conciliação bancária (§8.2/8.3)

10. COMISSÃO DO FORNECEDOR  (§8.4)
    └─ seleção manual das reservas → NF por fornecedor → envio → crédito

11. FECHAMENTO DO PERÍODO  (§11)
```

---

## 7. Conferência (dupla checagem)

### 7.1 Conferência do voucher — IA × humano (P3)

Cada campo extraído carrega três metadados: **valor**, **confiança da
extração** (0–100) e **estado** (`sugerido` / `confirmado` / `corrigido`).

Regras:
1. A tela mostra **documento e campos lado a lado**, com o trecho de origem
   destacado.
2. Campo com confiança abaixo do limite (sugestão: **85**) já entra
   **realçado** e **exige** toque humano.
3. **Valores financeiros (custo, venda, %) nunca são aceitos automaticamente**,
   mesmo com confiança alta — são o que gera dinheiro.
4. Campo `corrigido` alimenta um relatório de **qualidade da IA** (taxa de
   acerto por campo e por fornecedor) — insumo para melhorar a extração.
5. Nenhuma venda passa para **Conferido** com campo em `sugerido` (§9 · D7).

### 7.2 Obrigatoriedade progressiva

| Para... | É obrigatório |
|---------|---------------|
| Salvar **rascunho** | Praticamente nada — o objetivo é não perder o trabalho. |
| Enviar para **conferência** | Cliente, fornecedor, vendedor, custo, venda, data da venda, ≥1 item. |
| **Conferir** | Tudo acima + voucher anexado por item + plano de pagamento fechando (D2/D3) + zero campo `sugerido` + zero divergência bloqueante (§9). |
| **Fechar período** | Zero venda em Rascunho/Em conferência no período + divergências bloqueantes tratadas (§11). |

### 7.3 Segregação de funções

Padrão: **quem lança não confere** (`autor ≠ conferente`).

Como a operação pode ser pequena, a trava é **configurável**:
`Bloquear` (padrão) · `Avisar e permitir com justificativa` · `Desligado`.
Na segunda opção, a autoconferência fica **marcada no log e no relatório de
divergências** — controle não é impedir, é deixar rastro.

### 7.4 Anti-duplicidade

Chave natural do item: **`fornecedor_id` + `localizador`**.

- Repetição exata → **bloqueia** e mostra o lançamento existente
  ("Localizador ABC123 do Fornecedor X já lançado na venda 2026-0311").
- Mesmo cliente + mesmo valor + mesma data em até 24h → **avisa** (não bloqueia).
- Mesmo arquivo de voucher (hash) já anexado → **avisa** com link para o original.

---

## 8. Conciliação

### 8.1 Os quatro laços

| # | Laço | Confronta | Fecha quando |
|---|------|-----------|--------------|
| **C1** | Documental | Voucher / NF / comprovante × lançamento | Todo item tem voucher, toda baixa tem comprovante, toda comissão tem NF. |
| **C2** | Recebimento | Parcelas a receber × extrato bancário | Todo crédito do extrato está amarrado a uma parcela, e vice-versa. |
| **C3** | Fornecedor | Parcelas a pagar × fatura/extrato do fornecedor | O saldo do fornecedor no SGI = o saldo que o fornecedor cobra. |
| **C4** | Comissão | Comissão prevista × NF emitida × crédito recebido | Previsto = faturado = recebido (ou glosa justificada). |

### 8.2 Conciliação bancária (C2)

**Fase 1 (sugerida):** importação de extrato em **CSV/OFX** + tela de
casamento. **Fase futura:** integração bancária direta, já prevista no
SGI-6 "f".

Tela de conciliação em três colunas:
`Extrato (não conciliado)` | `Sugestões do sistema` | `Parcelas em aberto`

### 8.3 Conciliação com o fornecedor (C3)

Relatório **Extrato do fornecedor** — por fornecedor e período:

```
Reservas do período ............ Σ custo
(−) Pagamentos efetuados ....... Σ baixas A_PAGAR
(=) Saldo a pagar

Comissões previstas ............ Σ comissão dos itens
(−) Comissões faturadas (NF)
(−) Comissões recebidas
(=) Comissão em aberto
```

Esse relatório é **o documento de fechamento com o fornecedor** — a peça que
hoje falta para conferir "quanto eu devo / quanto ele me deve".

### 8.4 Conciliação da comissão (C4) e **glosa**

```
Comissão prevista (Σ itens selecionados)
   → NF emitida (valor da NF)
      → Crédito recebido (extrato)

Prevista ≠ NF          → D13  (seleção de reservas errada ou % errado)
NF ≠ Crédito recebido  → D14  GLOSA — exige motivo:
                              · reserva cancelada/no-show
                              · tarifa recalculada pelo fornecedor
                              · retenção de imposto
                              · erro nosso de %
                              · a esclarecer com o fornecedor
```

Glosa **não some**: registra-se `valor_recebido` (real) e a diferença fica
como divergência aberta até ser aceita (baixa por glosa, com motivo) ou
cobrada.

### 8.5 Regras de casamento automático (matching)

Casa quando **todas** as condições valem:

| Critério | Regra |
|----------|-------|
| Valor | Igual ao previsto, com tolerância de **R$ 0,05** (arredondamento). |
| Data | Vencimento **± 5 dias corridos** (configurável). |
| Identificador | Bate `identificador` (E2E do Pix, NSU, nº do boleto) **ou** nome do pagador ≈ nome do cliente. |

Comportamento:
- **1 movimento ↔ 1 parcela**, valor e identificador batendo → **sugere
  conciliação automática** (confirmação em 1 clique; conciliação 100%
  automática só se o cliente quiser — ver **Q8**).
- **1 movimento ↔ N parcelas** (pagamento agrupado) → permite selecionar
  várias parcelas até fechar o valor.
- **N movimentos ↔ 1 parcela** (pagamento em partes) → baixas parciais.
- Diferença **acima** da tolerância → **não concilia**: abre divergência com
  classificação (taxa, juros, desconto, glosa, pagamento a maior/menor).
- Movimento sem correspondência após **X dias** (sugestão: 7) → **D15**.

---

## 9. Painel de divergências

Tela única com **tudo que não fecha**, com filtro por tipo, responsável e
idade. Cada regra é verificada automaticamente a cada gravação e por rotina
diária.

**🔴 Bloqueantes** (impedem conferir a venda ou fechar o período):

| ID | Regra |
|----|-------|
| **D1** | `Venda ≠ Custo + Markup` (item ou consolidado). |
| **D2** | Σ parcelas a receber ≠ valor de venda da venda. |
| **D3** | Σ parcelas a pagar ≠ valor de custo do fornecedor. |
| **D4** | Σ baixas > valor da parcela (pagamento a maior não tratado). |
| **D5** | Item ativo **sem voucher** anexado (P4). |
| **D6** | Localizador **duplicado** no mesmo fornecedor (§7.4). |
| **D7** | Campo extraído pela IA ainda em `sugerido` (P3). |
| **D13** | Valor da NF ≠ Σ comissões vinculadas a ela. |
| **D16** | Venda **cancelada** com parcelas ativas e sem tratamento de reembolso (§10). |
| **D17** | Tentativa de alterar lançamento em **período fechado** (P7). |

**🟡 De atenção** (não bloqueiam; entram na fila de tratamento):

| ID | Regra |
|----|-------|
| **D8** | Markup **negativo**, ou markup% fora da faixa esperada do fornecedor. |
| **D9** | % de comissão do fornecedor **diferente do padrão** do cadastro. |
| **D10** | Comissão do vendedor ≠ `% vigente × markup` (ajuste manual sem motivo). |
| **D11** | Pagamento ao fornecedor **vencendo em ≤ 3 dias** sem programação, ou **vencido**. |
| **D12** | Recebimento do cliente **vencido**. |
| **D14** | Crédito de comissão ≠ valor da NF → **glosa** (§8.4). |
| **D15** | Movimento bancário **não conciliado** há mais de X dias. |
| **D18** | Taxa efetiva ≠ taxa prevista (cartão/gateway) acima da tolerância. |
| **D19** | `data_embarque` anterior à `data_venda`, ou `data_servico` incoerente. |
| **D20** | Cliente/fornecedor/vendedor preenchido como **texto livre** em vez de vínculo com o cadastro (SGI-7). |
| **D21** | Venda **conferida pelo próprio autor** (§7.3, quando permitido). |
| **D22** | Comissão de fornecedor **a faturar** há mais de X dias (sugestão: 30). |
| **D23** | Venda conferida **editada** depois (mostra o quê, quem e quando). |

Cada divergência tem: **origem** (venda/parcela/NF), **valor envolvido**,
**data de detecção**, **responsável**, **status** (Aberta / Em tratamento /
Resolvida / Aceita com justificativa) e **link direto** para o registro.

---

## 10. Cancelamento, reembolso e estorno

Cancelar **não apaga** (P1) e **não zera** o histórico — cancelar é um
evento datado.

**Ao cancelar uma venda ou um item:**

1. Registrar `data_cancelamento`, `motivo` e `usuario`.
2. `Reembolso? sim/não` + `valor_reembolsado` (SGI-6 "a").
3. **Parcelas ainda em aberto** → `Canceladas`.
4. **Baixas já recebidas** → geram **devolução** (movimento de saída, com
   comprovante) até o `valor_reembolsado`. O que ficar retido aparece como
   **multa/retenção** — receita da agência, não some.
5. **A pagar ao fornecedor** → cancela o que ainda não venceu; o que já foi
   pago vira **crédito a recuperar com o fornecedor** (entra no extrato §8.3).
6. **Comissão do fornecedor** → `Cancelada`; se a NF já foi emitida e paga,
   abre divergência para **acerto na próxima NF**.
7. **Comissão do vendedor** → `Estornada`. Regra de estorno — ver **Q4**:
   sugestão = **proporcional ao markup que sobrou** após a retenção
   (`40% × markup_remanescente`), e não estorno integral automático.
8. Relatórios: a venda cancelada sai dos totais efetivos e aparece em
   **bloco próprio** com valor original, valor reembolsado e valor retido
   (exigência do SGI-6 "c").

**Estorno de baixa** (pagamento lançado errado, Pix devolvido, chargeback):
não se apaga a baixa — cria-se baixa de estorno **vinculada**, com motivo.
O status financeiro se recalcula sozinho (P2).

---

## 11. Fechamento de período

**Competência = `data_venda`.** Fechamento **mensal**.

Checklist do fechamento (o sistema não deixa fechar com item pendente):

```
[ ] Nenhuma venda em Rascunho / Em conferência no período
[ ] Zero divergência BLOQUEANTE aberta (§9)
[ ] Extrato bancário do período importado e conciliado (ou pendências justificadas)
[ ] Comissões do período faturadas ou justificadas (D22)
[ ] Conferência de caixa do último dia fechada (§12)
```

Ao fechar: todas as vendas do período vão para `Fechado`, os totais são
**materializados** (o relatório do mês fechado não muda mais, mesmo que um
cadastro mude depois) e o período fica bloqueado para escrita.

**Correção depois do fechamento (P7):** não se reabre por padrão. Cria-se um
**lançamento de ajuste** no período aberto, com `venda_origem_id`, motivo e
valor da diferença — o histórico mostra os dois. Reabertura de período existe,
mas exige perfil **Gestor**, motivo obrigatório e fica registrada no log e em
um relatório de reaberturas.

---

## 12. Relatórios

Atendem ao SGI-6 "c" e acrescentam as visões de controle.

**a) Resultado de vendas — dia / mês / ano** *(SGI-6 "c")*
Por venda e em totais: **custo, venda, markup, comissão do fornecedor,
comissão do vendedor, resultado**. Vendas canceladas e reembolsos em bloco
separado. Comparativo com o período anterior.

**b) Conferência do dia (fechamento diário)**
Lançado hoje · conferido hoje · recebido hoje (previsto × real) · pago hoje ·
divergências abertas no dia. É o relatório que a operação olha todo fim de tarde.

**c) Contas a receber / a pagar (aging)**
A vencer · vencido 1–30 · 31–60 · 60+ , por cliente e por fornecedor.

**d) Extrato por fornecedor** *(§8.3)* — a peça de fechamento com cada fornecedor.

**e) Posição de comissões**
A faturar · faturado (NF) · recebido · glosado · em atraso, por fornecedor.

**f) Comissões a pagar por vendedor**
Apurada · liberada · paga · estornada, por vendedor e competência.

**g) Divergências abertas** *(§9)* — por tipo, idade e responsável.

**h) DRE simplificada do período**

```
Receita de venda (Σ venda)
(−) Custo dos serviços (Σ custo)
(=) Markup
(+) Comissão de fornecedores
(=) Receita bruta da agência
(−) Comissão de vendedores
(−) Taxas de meio de pagamento
(=) Resultado do período
```

**i) Qualidade da extração por IA** *(§7.1)* — acerto por campo/fornecedor.

**Todos** os relatórios: filtro por período, cliente, fornecedor, vendedor,
forma de pagamento, status da venda, status de pagamento e status de
conciliação (SGI-6 "i") + exportação para **CSV/Excel** — sem exportação não
há conferência de verdade.

---

## 13. Auditoria e permissões

### 13.1 Trilha de auditoria
Ver §3.8. Requisitos adicionais:
- Log **imutável** (append-only), sem edição/exclusão pela interface.
- Cada venda tem aba **"Histórico"** legível pelo usuário
  (`"12/08 14:32 — Maria alterou Valor de venda de 3.400,00 para 3.560,00 — motivo: correção do voucher"`).
- Registrar também **quem conferiu**, **quando** e **quem fechou o período**.

### 13.2 Perfis

| Perfil | Pode |
|--------|------|
| **Vendedor** | Lançar e editar **as próprias** vendas em Rascunho; ver as próprias comissões. Não confere, não dá baixa. |
| **Financeiro / Conferente** | Conferir vendas (de terceiros), criar parcelas, dar baixa, conciliar, lançar NF de comissão. |
| **Gestor** | Tudo acima + editar venda conferida, cancelar, fechar/reabrir período, editar cadastros, ver todos os relatórios. |
| **Consulta** | Só leitura (contador/auditoria). |

---

## 14. Decisões pendentes

Cada item traz uma **sugestão de padrão** — se aprovada como está, a
implementação segue sem esperar.

| # | Decisão | Sugestão de padrão |
|---|---------|--------------------|
| **Q1** | Base da **comissão do fornecedor**: sobre o **custo** ou sobre a **venda**? | **Custo** (tarifa do fornecedor). Campo configurável por fornecedor. |
| **Q2** | Comissão vem por **repasse** (fornecedor paga depois, via NF) ou por **dedução** (abatida no pagamento)? | Suportar os dois (§4.5); **Repasse** como padrão, definido no cadastro do fornecedor. |
| **Q3** | **Quando** a comissão do vendedor fica devida? | **Apurada** na conferência; **liberada** para pagamento quando o cliente quitar 100%. |
| **Q4** | Estorno da comissão do vendedor em cancelamento com reembolso: integral ou proporcional? | **Proporcional ao markup remanescente** (§10.7). |
| **Q5** | Existe venda com custo em **moeda estrangeira** (USD/EUR)? | Fase 1 só **BRL**; campo `moeda` + `taxa_cambio` já previstos no modelo para não precisar migrar depois. |
| **Q6** | Qual data rege os relatórios: **data da venda** ou **data de embarque**? | **Data da venda** (competência), com filtro alternativo por embarque. |
| **Q7** | Taxa de cartão/gateway é **absorvida** pela agência ou **repassada** ao cliente? | Configurável por forma de pagamento; padrão **absorvida** (reduz o resultado, §4.6). |
| **Q8** | Conciliação bancária pode ser **100% automática** ou sempre com confirmação humana? | **Sugestão + confirmação em 1 clique** (P3). |
| **Q9** | Há gente suficiente para **quem lança ≠ quem confere**? | Trava ligada por padrão, com opção "avisar e permitir com justificativa" (§7.3). |
| **Q10** | Prazos de alerta: comissão a faturar (D22), movimento não conciliado (D15). | **30 dias** e **7 dias**. |
| **Q11** | Existe **numeração de pedido/reserva** já usada hoje pela operação? | Se existir, usar como campo adicional; o `numero` do lançamento é sempre sequencial próprio. |
| **Q12** | Uma venda pode ter **vários fornecedores** (modelo venda → itens)? | Confirmado como sim — é a premissa central desta spec (§3.3). |

---

## 15. Glossário

| Termo | Significado aqui |
|-------|------------------|
| **Lançamento** | O registro da venda no sistema (≠ a venda em si, que é o negócio). |
| **Item** | Um serviço/voucher de um fornecedor dentro da venda. |
| **Markup** | `Venda − Custo`. Margem da agência sobre o serviço. |
| **Comissão do fornecedor** | O que o fornecedor paga à agência (%, por venda). |
| **Comissão do vendedor** | 40% (padrão, ajustável) sobre o markup. |
| **Parcela** | Compromisso financeiro **previsto**, com vencimento. |
| **Baixa** | Pagamento **realizado**, com data, valor e comprovante. |
| **Conciliação** | Casar o previsto (parcela) com o realizado (extrato). |
| **Glosa** | Fornecedor pagar comissão **menor** que a NF/prevista. |
| **Competência** | Período a que o resultado pertence (data da venda). |
| **Caixa** | Período em que o dinheiro entrou/saiu (data do pagamento). |
| **Snapshot** | Cópia congelada de um parâmetro no momento do lançamento. |
| **Divergência** | Qualquer diferença detectada entre o que deveria e o que é. |

---

> **Próximo passo sugerido:** validar a seção **14** com o cliente e, em
> paralelo, o dev pode começar por **§3 (modelo de dados)** + **§5 (status)**
> — são a base de tudo o mais e não dependem de nenhuma decisão pendente.
