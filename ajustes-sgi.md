# Ajustes SGI — Itens para o Time de Dev

> Documento de acompanhamento de bugs e ajustes necessários no **SGI** (sistema
> de gestão de leads/CRM da CI Intercâmbio, integrado ao DataCrazy). Diferente
> de `contexto-comercial-ci-intercambio.md` (que treina a IA "Cibele"), este
> arquivo é **voltado para o time de desenvolvimento** corrigir o sistema.
>
> Alimentado incrementalmente pelo Marcelo, a partir de prints/observações do
> uso real do SGI. Itens vagos são marcados com ⚠️ **A ESCLARECER** até
> confirmação.

---

## Legenda de status

- 🔴 **Aberto** — bug/ajuste confirmado, aguardando correção do dev.
- ⚠️ **A esclarecer** — pergunta em aberto, precisa de investigação/resposta
  antes de virar tarefa de dev.
- ✅ **Resolvido** — corrigido e validado.

---

## Resumo (triagem)

Visão rápida para o time de dev. As **prioridades são sugestões iniciais** — ajuste
conforme sua necessidade. Cada item tem um **ID estável** (SGI-N) para facilitar a
referência nas conversas com o dev.

| ID | Item | Prioridade (sugerida) | Status |
|----|------|-----------------------|--------|
| SGI-1 | Origem "SITE CI - WHATSAPP" não existe no CI GO | Média | 🔴 Aberto |
| SGI-2 | Token técnico (JWT) vazando no campo "Histórico/Mensagem" | Alta | 🔴 Aberto |
| SGI-3 | "Cidade de Interesse" preenchida com a cidade de residência | Média | 🔴 Aberto |
| SGI-4 | Sobreposição de campos em lead cadastrado mais de uma vez | A definir | ⚠️ A esclarecer |
| SGI-5 | Falha ao enviar mensagem via Evolution (link não entregue) | Alta | 🔴 Aberto |
| SGI-6 | [Novo módulo] Pós-vendas (Turismo): lançamento de vendas, financeiro e NF de comissão | Alta | 🔴 Aberto |

---

## Itens

### SGI-1 · 🔴 Opção de origem "SITE CI - WHATSAPP" não existe no CI GO (sistema da franqueadora)

No SGI, o campo de **origem/mídia (publicidade)** do lead tem uma opção
chamada **"SITE CI - WHATSAPP"**. Essa origem **não existe** no **CI GO**
(sistema da franqueadora, fonte de verdade). No CI GO, as únicas opções
equivalentes de canal WhatsApp são:
- **WhatsApp Chatbot**
- **WhatsApp Form**

**Ação sugerida:** excluir a opção **"SITE CI - WHATSAPP"** da lista de mídias
do SGI, mantendo apenas as opções que espelham as existentes no CI GO (ex.:
**SITE CI - WHATSAPP CHATBOT**, **SITE CI - WHATSAPP FORM**).

> Evidência: print do dropdown "Mídia (publicidade)" do SGI mostrando
> `SITE CI - WHATSAPP` marcado (✓) ao lado de `SITE CI - WHATSAPP CHATBOT`,
> `SITE CI - WHATSAPP FORM`, `SITE CI - REDES SOCIAIS` etc.; e print do CI GO
> mostrando apenas `SITE CI - WHATSAPP CHATBOT` e `SITE CI - WHATSAPP FORM`
> como opções ao digitar "wha".

### SGI-2 · 🔴 Token técnico vazando no campo "Histórico/Mensagem"

No cadastro de leads vindos de **SITE CI - WHATSAPP CHATBOT**, o campo
**Histórico/Mensagem** (que deveria conter só origem, produto, perguntas e
intenções do lead) está vindo com um **token técnico bruto** (JWT) misturado
antes dos dados estruturados. Exemplo observado:

```
...ao_boas-vindas&mcp_token=eyJwaWQiOjQ1NTU3MzQsInNpZCI6MjA1NTk2...
Produto: CURSOS
Idade: 28
Cidade: GOIÂNIA
Programa de interesse: Cursos de Idiomas
...
```

Parece ser um parâmetro de URL/callback da automação do chatbot (`mcp_token=`)
que está sendo gravado por engano dentro do campo de texto livre, em vez de
ser tratado/descartado pela integração.

**Ação sugerida:** revisar a automação que grava o histórico do lead a partir
do WhatsApp Chatbot para não incluir parâmetros de URL/tokens de sessão no
campo de mensagem — apenas os dados estruturados do lead.

### SGI-3 · 🔴 Campo "Cidade de Interesse" preenchido com a cidade de residência do lead

O campo **"Cidade de Interesse"** está sendo preenchido automaticamente com a
**cidade onde o lead mora** (ex.: "Goiânia"), quando na verdade o lead **não
informou** nenhuma cidade de interesse — só a cidade onde reside. A automação
está interpretando "cidade onde mora" como se fosse "cidade de interesse",
o que está incorreto.

**Ação sugerida:** corrigir o mapeamento da automação para não copiar a
cidade de residência do lead para o campo "Cidade de Interesse". O campo
"Cidade de Interesse" deve ficar vazio quando o lead não informar isso
explicitamente.

### SGI-4 · ⚠️ A esclarecer — Sobreposição de campos quando o mesmo lead é cadastrado mais de uma vez

Quando o mesmo lead entra em contato novamente e já existe no SGI/DataCrazy,
o fluxo de automação registra as etapas:

```
✅ WhatsApp — Número confirmado no WhatsApp
✅ DataCrazy — Lead já existe no DataCrazy
✅ DataCrazy — Gravado no DataCrazy: origem + 11 campo(s)
⚠️ Conversas — Conversa ativa em: prevenda (última msg 20/07, 19:35) — email de alerta enviado
```

**Dúvida em aberto:** quando o passo "Gravado no DataCrazy: origem + 11
campo(s)" roda para um lead que **já existe**, os campos são:
- **sobrescritos** (o novo dado apaga o antigo, mesmo que o antigo estivesse
  correto e o novo venha vazio/errado)? ou
- **mesclados/preservados** (só atualiza campos vazios, sem apagar dado já
  confirmado)?

Isso é relevante porque, se houver sobreposição indevida, um segundo contato
mal interpretado pela automação (ver itens 2 e 3) pode **sobrescrever dados
corretos** já registrados no primeiro contato.

**Ação sugerida:** time de dev confirmar o comportamento real da integração
com o DataCrazy nesse cenário de lead duplicado e documentar aqui.

### SGI-5 · 🔴 Falha ao enviar mensagem via Evolution (WhatsApp) — link de agendamento não entregue

Numa conversa (atendente **Leonardo**, lead **Ana Clara**), a mensagem com o
**link do Calendly** e a mensagem seguinte **falharam no envio** (ícone vermelho
de erro), enquanto a mensagem de texto anterior foi entregue normalmente. Erro
exibido pelo sistema:

```
Não foi possível enviar a mensagem para o Evolution. Verifique a conexão com a
instância e se a mensagem é uma mensagem válida.
```

**Causas prováveis (a investigar pelo dev):**
1. **Instância do Evolution desconectada/instável** — a sessão do WhatsApp caiu
   ou perdeu conexão no meio da conversa (precisa reconectar / re-scan do QR).
2. **Payload inválido** — a mensagem que falhou era **só o link** (URL isolada);
   dependendo da configuração, o Evolution pode rejeitar corpo vazio/mal formado
   ou falhar na geração de preview do link.
3. **Cadência/rate limit** — várias mensagens disparadas em sequência muito
   rápida podem ser recusadas.

**Ações sugeridas (prevenção):**
- **Monitorar a saúde da instância** do Evolution (status de conexão) e **alertar
  + reconectar automaticamente** quando cair; não deixar seguir enviando "no
  escuro".
- **Fila com retry/backoff** para envios que falharem, em vez de descartar a
  mensagem silenciosamente.
- **Validar o payload** antes de enviar (corpo não vazio, texto válido) e
  **embutir o link dentro de uma frase** (ex.: "Segue o link: <url>") em vez de
  mandar a URL sozinha.
- **Throttle** (pequeno intervalo) entre mensagens consecutivas.
- **Fallback visível ao atendente:** quando um envio falhar, sinalizar
  claramente e permitir **reenvio com 1 clique** — para o link de agendamento
  nunca ficar sem chegar ao lead.

> Evidência: print da conversa com a mensagem do link do Calendly e a seguinte
> marcadas com erro (vermelho) e o balão de erro "Não foi possível enviar a
> mensagem para o Evolution...".

### SGI-6 · 🔴 [Novo módulo] Pós-vendas (aba Turismo) — Lançamento e gestão financeira/fiscal de vendas

**Objetivo:** Na **aba Turismo**, dentro do **pós-vendas**, criar uma área para
**lançar as vendas** e centralizar a gestão financeira e fiscal. O sistema deve
ser alimentado a cada venda e **entregar os resultados de vendas por dia, mês e
ano**, sempre com **valor de custo, valor de venda e comissão**, além de tratar a
**nota fiscal (NF) da comissão**.

> **Importante:** **todos os dados abaixo são essenciais para os relatórios** —
> precisam ser gravados de forma **estruturada** (pesquisável, filtrável e
> somável), não apenas como texto solto.

**a) Lançamento da venda — campos necessários:**
- **Cliente comprador** (com cadastro do comprador — ver "d").
- **Fornecedor** (do cadastro de fornecedores — ver "h").
- **Voucher(s)** — anexo em **PDF ou print/imagem** (fonte para a IA — ver "b").
- **Valor de custo**.
- **Valor de venda**.
- **Markup** — regra **`Venda = Custo + Markup`**. Pode ser **preenchido em
  aberto** ou **calculado pelo sistema** a partir de custo e venda
  (`Markup = Venda − Custo`).
- **Comissão** — **campo de percentual (%) aberto** (varia a cada venda); a
  partir do % o sistema calcula o **valor** da comissão.
- **Forma de pagamento**.
- **Pagamento do cliente** — ver controle de pagamentos em "e".
- **Pagamento do fornecedor** — ver controle de pagamentos em "e".
- **Data** (da venda / do lançamento).
- **Observação** (campo de texto livre).

**b) Preenchimento automático por IA (leitura de vouchers):**
- A IA deve **ler o voucher em PDF ou print (imagem)** e **preencher
  automaticamente** os campos do lançamento, com **conferência/ajuste manual**
  antes de salvar.
- Os vouchers têm **layout parecido**. Campos que a IA deve **extrair**:
  - **Número/localizador da reserva**
  - **Data**
  - **Quantidade de hóspedes**
  - **Nome do prestador de serviço local** (hotel, passeios, transfer etc.)
  - **Cidade**
  - **Endereço**
  - **Nomes dos hóspedes**

**c) Resultados de vendas (relatórios/consolidação):**
- Consolidar e exibir os resultados **do dia, do mês e do ano**.
- Mostrar, por venda e em totais: **custo, venda e comissão**.
- Os relatórios se apoiam em **todos** os campos do lançamento — daí a
  necessidade da captura estruturada (ver nota acima).

**d) Cadastro do cliente comprador:**
- Campo/tela para **cadastrar o cliente comprador** e vinculá-lo à venda.

**e) Controle de pagamentos (cliente e fornecedor):**
- Cada pagamento precisa de **status de controle** (ex.: **pago / pendente /
  parcelado** — confirmar a lista final).
- Registrar **tipo/forma de pagamento** e os **dados do pagamento** (ex.: valor,
  data, parcelas/vencimento).
- Vale tanto para o **pagamento do cliente** quanto para o **pagamento ao
  fornecedor**.

**f) Nota fiscal (NF) da comissão:**
- A NF é emitida **somente sobre o valor da comissão**.
- **A princípio, o sistema apenas lança/registra** a **NF já emitida por fora**
  (não emite a NF nesta primeira fase).
- **Uma NF por fornecedor**, cujo valor é a **somatória das comissões** daquele
  fornecedor. Ou seja, o sistema precisa **agrupar por fornecedor** e **somar as
  comissões** das vendas para compor/registrar a NF.
  - ⚠️ **A confirmar:** critério de **agrupamento/fechamento** (ex.: por mês? por
    seleção manual das vendas incluídas?) para fechar a NF de cada fornecedor.

**g) Edição — tudo editável:**
- **Todos** os itens devem ser **editáveis**: fornecedores, markup, comissão,
  vendas, datas, valores e qualquer outro dado do sistema.

**h) Cadastro de fornecedores:**
- Cadastro de **fornecedores** reaproveitado nos lançamentos.
- O **markup** pode ficar **em aberto** por lançamento **ou** ser **derivado** da
  leitura de **custo × venda** (ver "a").

**i) Listagem e filtros:**
- Todas as vendas lançadas aparecem **por ordem de lançamento**.
- **Filtros de busca** para localizar vendas com **todos os seus detalhes**
  (ex.: cliente, fornecedor, período/data, valor, forma de pagamento, status).

**⚠️ Pontos ainda a esclarecer:**
1. **Fechamento da NF por fornecedor** — critério de agrupamento das comissões
   (mês fechado? seleção manual das vendas incluídas?).
2. **Status de pagamento** — confirmar a lista exata usada no dia a dia (ex.:
   pago, pendente, parcialmente pago, parcelado, estornado).

---

## 🗒️ Changelog

- **2026-07-21** — Criação do documento. Registrados os 3 primeiros itens
  (origem "SITE CI - WHATSAPP" inexistente no CI GO; token técnico vazando no
  histórico/mensagem; cidade de interesse preenchida errado com cidade de
  residência) e a dúvida em aberto sobre sobreposição de campos em leads
  duplicados no DataCrazy.
- **2026-07-24** — Item 5: falha de envio via **Evolution/WhatsApp** (link do
  Calendly não entregue na conversa do Leonardo com a lead Ana Clara). Causas
  prováveis (instância desconectada, payload de URL isolada, cadência) e ações de
  prevenção (monitorar/reconectar instância, fila com retry, validar payload,
  embutir o link em frase, throttle, reenvio fácil pelo atendente).
- **2026-07-28** — Reorganização para facilitar o repasse ao dev: adicionada a
  seção **Resumo (triagem)** com tabela (ID, item, prioridade sugerida, status) e
  atribuídos **IDs estáveis** (SGI-1…SGI-5) aos itens. Conteúdo e descrições dos
  itens mantidos sem alteração.
- **2026-07-28** — Registrado o **SGI-6**: novo módulo de **Pós-vendas (aba
  Turismo)** para lançamento e gestão financeira/fiscal de vendas — vouchers,
  pagamentos de cliente e fornecedor, custo/venda/comissão, notas fiscais de
  comissão, leitura automática por IA (PDF/print), listagem por ordem de
  lançamento com filtros, e edição total dos dados. Inclui pontos a esclarecer.
- **2026-07-28** — SGI-6 detalhado com as definições do cliente: `Venda = Custo +
  Markup` (markup em aberto ou calculado por custo × venda); **comissão em %
  aberto**; **NF apenas lançada** (emitida por fora), **por fornecedor** e igual à
  **somatória das comissões**; **controle de pagamentos** com status/tipo/dados
  (cliente e fornecedor); **campos que a IA extrai do voucher** (localizador,
  data, hóspedes, prestador, cidade, endereço, nomes); **cadastro de
  fornecedores**; reforço de que todos os dados alimentam os relatórios. Restam 2
  pontos a confirmar (fechamento da NF por fornecedor; lista de status de pgto).
