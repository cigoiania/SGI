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
| SGI-7 | [Novo módulo] Cadastros base: Clientes, Vendedores, Fornecedores | Alta | 🔴 Aberto |
| SGI-8 | "País de interesse" vazio mostra "Irlanda" (mockup) em vez de "—" | Baixa | 🔴 Aberto |
| SGI-9 | Observações grava "Produto alterado: X → Y" sem alteração real | Média | 🔴 Aberto |

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

> Evidência (2026-07-28): print do cadastro de novo lead com **Cidade de interesse
> = "APARECIDA DE GOIÂNIA"**, idêntica à **Localização do lead ("Aparecida De
> Goiânia | Goiás")** — ou seja, foi copiada a cidade de residência. O lead não
> informou cidade de interesse.

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
- **Vendedor** (responsável pela venda — do cadastro de **Vendedores**, ver SGI-7).
- **Voucher(s)** — anexo em **PDF ou print/imagem** (fonte para a IA — ver "b").
- **Valor de custo**.
- **Valor de venda**.
- **Markup** — regra **`Venda = Custo + Markup`**. Pode ser **preenchido em
  aberto** ou **calculado pelo sistema** a partir de custo e venda
  (`Markup = Venda − Custo`).
- **Comissão do fornecedor** — **campo de percentual (%) aberto** (varia a cada
  venda); a partir do % o sistema calcula o **valor** da comissão recebida do
  fornecedor.
- **Comissão do vendedor** — **padrão 40%** (campo **ajustável** por venda),
  calculada **sobre o markup**: `Comissão do vendedor = 40% × Markup`. Vinculada
  ao **vendedor** (SGI-7).
- **Forma de pagamento**.
- **Pagamento do cliente** — ver controle de pagamentos em "e".
- **Pagamento do fornecedor** — ver controle de pagamentos em "e".
- **Data** (da venda / do lançamento).
- **Observação** (campo de texto livre).
- **Status da venda** — **Ativa** (padrão) ou **Cancelada**. Quando **Cancelada**,
  registrar:
  - **Reembolso?** — **sim / não**.
  - **Valor reembolsado** (quando houver reembolso).
  - *(É o status da **venda**, diferente do status de **pagamento** do item "e".)*

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
- Identificar **vendas canceladas** e **valores reembolsados** nos resultados
  (não misturar com as vendas efetivas).
- Os relatórios se apoiam em **todos** os campos do lançamento — daí a
  necessidade da captura estruturada (ver nota acima).

**d) Cadastro do cliente comprador:**
- Campo/tela para **cadastrar o cliente comprador** e vinculá-lo à venda
  (faz parte do módulo de **Cadastros → Clientes** — ver **SGI-7**).

**e) Controle de pagamentos (cliente e fornecedor):**
- Cada pagamento precisa de **status de controle**: **Pago / Pendente /
  Parcelado / Cancelado / Estornado**.
- Registrar **tipo/forma de pagamento** e os **dados do pagamento** (ex.: valor,
  data, parcelas/vencimento).
- Vale tanto para o **pagamento do cliente** quanto para o **pagamento ao
  fornecedor**.
- O **vencimento do pagamento ao fornecedor** pode ser calculado a partir do
  **prazo para pagamento** definido no cadastro do fornecedor (ver **SGI-7**).

**f) Nota fiscal (NF) da comissão + recebimento da comissão:**
- A NF é **somente sobre o valor da comissão**.
- **A princípio, o sistema apenas lança/registra** a **NF já emitida por fora**
  (não emite a NF nesta primeira fase).
- **Uma NF por fornecedor**, com valor = **somatória das comissões**. As vendas
  que compõem cada NF são definidas por **seleção manual** — você marca as
  **reservas** que aquela NF cobre.
- A **IA deve ler o documento da NF** e **vincular/preencher os dados em cada
  reserva** a que a NF se refere.
- **Campo para marcar que o fornecedor efetuou o pagamento** da comissão
  (status **comissão recebida × pendente**).
- 🔜 **Fase futura:** **integração com o banco** para **reconhecer os pagamentos
  automaticamente** e avisar quando a **comissão ainda estiver pendente**.

**g) Edição — tudo editável:**
- **Todos** os itens devem ser **editáveis**: fornecedores, markup, comissão,
  vendas, datas, valores e qualquer outro dado do sistema.

**h) Cadastro de fornecedores:**
- Cadastro de **fornecedores** reaproveitado nos lançamentos (parte do módulo de
  **Cadastros → Fornecedores** — ver **SGI-7**).
- O **markup** pode ficar **em aberto** por lançamento **ou** ser **derivado** da
  leitura de **custo × venda** (ver "a").

**i) Listagem e filtros:**
- Todas as vendas lançadas aparecem **por ordem de lançamento**.
- **Filtros de busca** para localizar vendas com **todos os seus detalhes**
  (ex.: cliente, fornecedor, período/data, valor, forma de pagamento, **status da
  venda** — inclusive canceladas — e status de pagamento).

**✅ Sem pontos em aberto no momento** — todas as dúvidas levantadas foram
definidas (ver seções acima). A spec está pronta para repasse ao dev; novos
prints/detalhes podem refiná-la.

### SGI-7 · 🔴 [Novo módulo] Cadastros (base de dados mestre do sistema)

**Objetivo:** Ter uma **área de Cadastros** (menu próprio) com os registros-base
reutilizados em todo o sistema — inclusive alimentando o lançamento de vendas do
**SGI-6**. Conforme o menu do sistema, os cadastros são:

- **Clientes**
- **Vendedores**
- **Fornecedores**

> **Fora do escopo:** **Parceiros** e **Companhias Aéreas** aparecem no menu do
> sistema, mas **não são necessários** e **não devem ser implementados** neste
> momento. O Turismo **não incluirá venda de aéreo**.

**Requisitos gerais (para todos os cadastros):**
- **CRUD completo:** criar, listar, **editar** e inativar/excluir.
- **Busca/filtro** dentro de cada cadastro.
- **Reaproveitáveis** nas telas e lançamentos do sistema (ex.: selecionar um
  fornecedor/cliente já cadastrado ao lançar uma venda — ver **SGI-6**).
- Tudo **editável** (coerente com a regra geral do SGI-6).

**Campos por cadastro** (✅ = definido pelo cliente):
- **Clientes** ✅ *(mesmo "cliente comprador" do SGI-6)*:
  - **Nome**
  - **Data de nascimento**
  - **Telefone**
  - **E-mail**
  - **CPF** e **RG**
  - **Passaporte** — número e **data de expiração**
  - **Endereço** completo com **CEP**
- **Vendedores** ✅: nome, contato, **comissão padrão = 40% sobre o markup**
  (campo **ajustável** por venda). *(Entra no cálculo do SGI-6.)*
- **Fornecedores** ✅: **nome/razão social**, **CNPJ**, **prazo para pagamento**
  (prazo para pagar o fornecedor — alimenta o vencimento no SGI-6). **Markup
  padrão** opcional (pode ficar em aberto por venda — ver SGI-6).

**✅ Sem pontos em aberto no momento** — cadastros definidos: **Clientes,
Vendedores e Fornecedores** (com seus campos). Novos detalhes podem refiná-lo.

### SGI-8 · 🔴 Campo "País de interesse" vazio mostra "Irlanda" (placeholder/mockup) em vez de "—"

No cadastro de novo lead, quando o **"País de interesse"** **não é preenchido**, o
campo exibe **"Irlanda"** em cinza (texto de **placeholder/mockup**), o que pode
ser lido erroneamente como se o país de interesse fosse a Irlanda.

O correto é **não exibir um país de exemplo**: mostrar **"—"** no estado vazio,
igual aos demais campos não preenchidos (ex.: **Consultor responsável**,
**Idiomas desejados**, **Modalidade do programa**).

**Ação sugerida:** trocar o placeholder "Irlanda" do campo "País de interesse"
por **"—"** (ou vazio neutro), padronizando o estado vazio com os outros campos.

> Observação: o dado em si parece **vazio/correto** (o lead não informou país) — o
> problema é o **placeholder enganoso**. Severidade baixa, mas confunde.

### SGI-9 · 🔴 Observações grava "Produto alterado: X → Y" mesmo sem alteração (ex.: "Cursos → Cursos")

No cadastro de novo lead, o campo **"Observações"** trouxe **"Produto alterado:
Cursos → Cursos"** — mas **não houve alteração** nem duplicidade de produto: o
produto sempre foi **Cursos**. A automação está registrando uma "alteração de
produto" mesmo quando o valor **antigo e o novo são iguais**.

**Ação sugerida:** só gravar a observação **"Produto alterado: A → B" quando
A ≠ B** (alteração real). Quando o valor não muda, **não escrever nada**.

> Pode estar ligado ao **reprocessamento de lead** (ver SGI-4): reconferir se o
> log de "produto alterado" dispara em re-submissões sem mudança real.

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
- **2026-07-28** — SGI-6 fechado nos 2 pontos que faltavam: **NF por fornecedor
  agrupa comissões por seleção manual** das reservas; **IA também lê a NF** e
  vincula os dados às reservas cobertas; **campo "fornecedor pagou a comissão"**
  (recebida × pendente); **status de pagamento = Pago / Pendente / Parcelado /
  Cancelado / Estornado**. Registrada **fase futura** de **integração bancária**
  (reconhecer pagamentos e sinalizar comissões pendentes). Sem pontos em aberto
  no momento.
- **2026-07-28** — Registrado o **SGI-7**: módulo de **Cadastros** (base mestre)
  com **Clientes, Vendedores, Fornecedores, Parceiros e Companhias Aéreas**,
  reaproveitados em todo o sistema (inclusive no SGI-6), com CRUD + busca + edição
  em cada cadastro. Campos por cadastro sugeridos (a confirmar) e pontos a
  esclarecer (comissão de vendedor, papel do parceiro, venda de aéreo via
  companhias aéreas). Cruzamentos adicionados no SGI-6 (itens "d" e "h").
- **2026-07-28** — SGI-7 afinado com o cliente: **campos de Clientes** definidos
  (nome, nascimento, telefone, e-mail, CPF/RG, passaporte com nº e validade,
  endereço com CEP); **comissão do vendedor = 40% (ajustável)**, refletida no
  SGI-6 (novo campo **Vendedor** e separação entre **comissão do fornecedor** e
  **comissão do vendedor** — base dos 40% a confirmar); **Parceiros** e
  **Companhias Aéreas** movidos para **fora do escopo** (Turismo **sem venda de
  aéreo**). Cadastros a implementar: **Clientes, Vendedores, Fornecedores**.
- **2026-07-28** — Definida a **base da comissão do vendedor**: **40% sobre o
  markup** (`Comissão do vendedor = 40% × Markup`, ajustável). Atualizado no SGI-6
  (item "a") e no SGI-7.
- **2026-07-28** — Campos de **Fornecedores** definidos: **CNPJ** e **prazo para
  pagamento** (além de nome/razão social; markup padrão opcional). O prazo
  alimenta o **vencimento do pagamento ao fornecedor** no SGI-6. **SGI-7 sem
  pontos em aberto** — cadastros: Clientes, Vendedores, Fornecedores.
- **2026-07-28** — SGI-6: adicionado o **status da venda** (**Ativa/Cancelada**);
  quando **cancelada**, registrar **reembolso (sim/não)** e **valor reembolsado**.
  Cancelamentos e reembolsos passam a ser identificados nos **relatórios** e
  **filtros** (é o status da venda, distinto do status de pagamento).
- **2026-07-28** — Mais erros de interpretação de dados do novo lead (a partir de
  print): **SGI-3** reforçado com evidência (Cidade de interesse = "Aparecida de
  Goiânia" = cidade de residência); novo **SGI-8** — "País de interesse" vazio
  mostra o mockup "Irlanda" em vez de "—"; novo **SGI-9** — Observações grava
  "Produto alterado: Cursos → Cursos" sem alteração real.
