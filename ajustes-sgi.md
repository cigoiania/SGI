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

**a) Lançamento da venda — campos necessários:**
- **Cliente comprador** (com cadastro do comprador — ver item "d").
- **Fornecedor**.
- **Voucher(s)** — anexo em **PDF ou print/imagem**.
- **Valor de custo**.
- **Valor de venda**.
- **Markup**.
- **Comissão**.
- **Forma de pagamento**.
- **Pagamento do cliente** (registro do que o cliente pagou).
- **Pagamento do fornecedor** (registro do que foi pago ao fornecedor).
- **Data** (da venda / do lançamento).
- **Observação** (campo de texto livre).

**b) Preenchimento automático por IA (leitura de documentos):**
- A IA deve **ler o voucher em PDF ou print (imagem)** e **preencher
  automaticamente** todos os dados possíveis do lançamento.
- Deve permitir **conferência e ajuste manual** antes de confirmar (ver "f").

**c) Resultados de vendas (relatórios/consolidação):**
- Consolidar e exibir os resultados **do dia, do mês e do ano**.
- Mostrar, por venda e em totais: **custo, venda e comissão**.

**d) Cadastro do cliente comprador:**
- Campo/tela para **cadastrar o cliente comprador** e vinculá-lo à venda.

**e) Nota fiscal (NF) da comissão:**
- **Emissão da NF da comissão de cada fornecedor.**
- **Lançamento da NF da comissão de cada venda realizada.**

**f) Edição — tudo editável:**
- **Todos** os itens devem ser **editáveis**: fornecedores, markup, vendas,
  datas, valores e qualquer outro dado do sistema.

**g) Listagem e filtros:**
- Todas as vendas lançadas aparecem **por ordem de lançamento**.
- **Filtros de busca** para localizar vendas com **todos os seus detalhes**
  (ex.: cliente, fornecedor, período/data, valor, forma de pagamento).

**⚠️ Pontos a esclarecer (produto/dev):**
1. **Modelo de comissão** — a comissão é o valor que a **CI recebe do
   fornecedor** (motivo da NF de comissão) ou algo pago a um vendedor? Como ela
   se calcula em relação a custo/venda/markup (ex.: `venda = custo + markup`;
   comissão = % de quê)?
2. **Nota fiscal** — "NF da comissão de **cada fornecedor**" e "NF da comissão de
   **cada venda**" são o mesmo fluxo ou dois? A NF é **emitida pelo próprio
   sistema** (integração com emissor NFS-e/NF-e) ou apenas **lançada/registrada**
   manualmente depois de emitida por fora?
3. **Pagamentos** — precisa de **status** (pago / pendente / parcelado), datas de
   vencimento e baixa, tanto para o cliente quanto para o fornecedor?
4. **IA de leitura** — quais campos têm prioridade na extração? Os vouchers têm
   **layout padronizado** por fornecedor?
5. **Cadastro base** — haverá um cadastro de **fornecedores** (com **markup
   padrão**) reaproveitado nos lançamentos?

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
