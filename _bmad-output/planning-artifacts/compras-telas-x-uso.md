# Compras — telas × uso real (M1, passo 1)

> Autora: Mary (Analista) · Validação: Eduardo · Status: **rascunho para validação**
> Fontes: `sgi/compras/src/main/webapp` (142 `.xhtml`), FacesBeans, entidades em `sgi/framework`, banco `sgi.producao` (dump 2023-06-19). Somente leitura; só contagens agregadas.
> Medição em 2026-10-23 com `count(*)`. **"Últimos 12 meses" = 2022-06-19 a 2023-06-19** (janela terminando na data do dump).

## 1. Como ler

As 142 telas se agrupam em 9 diretórios. Muitas são fragmentos (painéis, modais, tooltips, e-mails, excel), não funcionalidades. Por isso o cruzamento é feito por **funcionalidade**, com a tabela principal como indicador de uso.

Classificação de uso:
- **Ativo**: criação de registros nos últimos 12 meses em volume relevante.
- **Residual**: existe histórico, mas quase nenhuma criação nos últimos 12 meses.
- **Morto**: sem linhas.
- **A medir**: ainda sem indicador (listado no §5).

## 2. Resultado por funcionalidade

| Funcionalidade | Telas (`.xhtml`) | Tabela-indicador | Total | Últimos 12m | Uso |
|---|---|---|---|---|---|
| Requisição de compra | 12 (`requisicaocompra/`) | `requisicao_compra` | 265.076 | 22.355 | **Ativo** |
| Ordem de compra: cadastro/listagem | 7 raiz + 7 `support/` + 1 relatório | `ordem_compra` | 495.664 | 32.597 | **Ativo** |
| Cotação / proposta | 9 (`ordemcompra/cotacao/`) | `proposta` | 520.518 | 35.198 | **Ativo** |
| Classificação (melhor proposta/negociação) | 19 (`ordemcompra/classificacao/`) | `pedido` | 387.947 | 29.662 | **Ativo** |
| Autorização da OC | 5 (`ordemcompra/autorizacao/`) | `ordem_compra` (data_envio_autorizacao) | — | a medir | provável Ativo |
| Recebimento | 11 (`ordemcompra/recebimento/`) | `recebimento` | 338.048 | 25.428 | **Ativo** |
| Nota fiscal (fornecedor e demais tipos) | 18 (`notas/fiscais/`) + 18 (`notas/support/`) | `nota_fiscal` (por `data_emissao`) | 623.397 | 49.792 | **Ativo** |
| Nota fiscal avulsa (tipo Fornecimento) | 9 (`notas/fiscais/avulsas/`) | `nota_fiscal` tipo 0 | 249.237 | 12.071 | **Ativo** |
| Relação de NF (envio ao financeiro) | 7 (`relacao/`) | `relacao_nf` | 82.920 | 4.594 | **Ativo** |
| Contratos de fornecedores | 13 (`contratosfornecedores/`) | `contrato` / `contrato_parcela` | 1.084 / 13.135 | 195 / 3.835 | **Ativo** (volume menor) |
| Avaliação de fornecedor (relatório) | 1 (`avaliacao/relatorio/`) | `avaliacao` | 99.458 | 11.063 | **Ativo** |
| Lote de validação de NF | 3 (`cadastros/lotevalidacaonf/`) | `lote_validacao_nf` | **0** | 0 | **Morto** |
| Relatório de item de compra | 1 (`itemcompra/relatorio/`) | `item_compra` | 59 mil | a medir | a medir |
| Movimento geral | 1 (`movimentogeral.xhtml`) | sem tabela própria identificada | — | — | a medir |

Total de telas cobertas: 3 + 1 + 1 + 13 + 1 + 45 + 59 + 7 + 12 = **142**.

## 3. Sub-funcionalidades da nota fiscal (volume nas tabelas filhas)

| Sub-funcionalidade (telas) | Tabela | Total | Últimos 12m* | Uso |
|---|---|---|---|---|
| Itens da NF (`aba_produtos`, `edicao/*`) | `item_nota_fiscal` | 750.369 | 34.145 | **Ativo** |
| Tributos/alíquotas da NF (`detail.tributos`) | `nota_tributo_aliquota` | 518.292 | 71.070 | **Ativo** |
| Anexos da NF (`detail.anexo(s)`) | `nota_anexo` | 39.027 | 25.004 | **Ativo** |
| Notas relacionadas (`detail.notasrelacionadas`) | `relacao_nf_nf` | 362.178 | 25.476 | **Ativo** |
| Relação bancária da NF (`detail.relacaobancaria`) | `relacao_bancaria` | 22.984 | 1.372 | **Ativo** |
| Nota de débito | `nota_debito` | 155 | 61 | **Ativo** (volume pequeno) |
| Outros descontos | `nota_outro_desconto` | 61 | 0 | **Residual** |
| Erro de processamento da NF | `nota_fiscal_erro_processo` | 141 | 0 | **Residual** |
| Tributos na OC | `ordem_compra_tributo` | 10.644 | 4 | **Residual** (praticamente abandonado) |
| Validação automática da NF (`detail.validacaonf`) | `erro_validacao_nf` (catálogo) | 44 | n/a | catálogo existe; ver §4.2 |

\* Para tabelas sem data própria, usa-se a `data_emissao` (ou `data_criacao`) da entidade-mãe.

### 3.1 Tipos de NF em uso

A coluna `nota_fiscal.tipo_nota_fiscal` é gravada pelo **ordinal** do enum `TipoNotaFiscal` (`NotaFiscal.java:66`; não há `@Enumerated`, então vale o padrão JPA). **A confirmar com Eduardo.** Pelo ordinal:

| Ordinal | Tipo | Total | Últimos 12m |
|---|---|---|---|
| 0 | Fornecimento (avulsa) | 249.237 | 12.071 |
| 1 | Ordem de Compra | 186.914 | 13.730 |
| 2 | NF/PC/Fundo Fixo | 42.776 | 3.527 |
| 3 | NF/PC/Reembolso | 6.117 | 742 |
| 4 | Transporte | 104 | 1 |
| 5 | NF/PC/Adiantamento | 77 | 0 |
| 6 | Devolução parcial | 94 | 13 |
| 7 | Devolução total | 72 | 1 |
| 8 | OC/PC/Fundo Fixo | 297 | 30 |
| 9 | OC/PC/Reembolso | 36 | 3 |
| 10 | OC/PC/Adiantamento | 0 | 0 |
| 11 | Faturamento | 136.851 | 19.593 |
| 12 | Faturamento, exclusão lógica | 822 | 81 |

Leitura: o fluxo vivo é Fornecimento, OC e as prestações de contas de Fundo Fixo/Reembolso. Os tipos 4–10 somam menos de 700 notas (em 12m, menos de 50). O tipo 10 (OC/PC/Adiantamento) nunca foi usado. Os tipos 6 e 7 (devolução) têm comentário no código "usado somente para Nota de Saída", mas aparecem em `nota_fiscal`: **a investigar**.

## 4. Achados nas pendências do M1

### 4.1 `nota.data_criacao` / `nota.data_atualizacao`: confirmado que estão obsoletas
`nota.data_atualizacao` vai de 2010-05 a 2011-04, igual a `data_criacao`. Não serve de indicador de uso. Para volume ao longo do tempo, usar a data da tabela-filha (`nota_fiscal.data_emissao`, `item_nota_fiscal.data_criacao`). A causa (herança JOINED) segue **não confirmada**.

### 4.2 `codigo_erro_validacao_nf`: hipótese nova, a validar com Eduardo
- Todas as 107.509 notas preenchidas têm o **mesmo** código, **100**, que o catálogo `erro_validacao_nf` descreve como "Autorizado o Uso".
- Elas são de notas emitidas de **2010 a 2017** (pico 2014: 25.571). **Nenhuma** de 2018 em diante, e nenhuma nos últimos 12 meses.
- O mapeamento JPA é `NotaFiscal.java:615` (`@JoinColumn codigo_erro_validacao_nf`). Os scripts `db/issues/SGINV-478_*` e `SGINV-479-*` criaram o catálogo (códigos 1000–1002 de erro técnico).
- Hipótese: a coluna **não foi gravada pelo job**, e sim por uma validação anterior ou carga única que cobriu 2010–2017. Isso conflita com "a validação hoje é manual", então **preciso da sua confirmação** do que foi feito até 2017 (ver §6).
- Impacto para o sistema novo: a regra "NF validada na SEFAZ ⇒ código 100" tem histórico real, o que ajuda a especificar o resultado esperado da simulação.

## 5. Ainda a medir (não afirmar uso)
- Autorização da OC (5 telas): contar `ordem_compra` por `data_envio_autorizacao`.
- `movimentogeral.xhtml` (`MovimentoGeralFacesBean`): descobrir quais tabelas ele lê.
- Relatório de item de compra (1 tela) e os 23 relatórios BIRT: passo 2.
- Sub-telas sem indicador próprio: `detail.efd`, `detail.nf.saida`, `detail.np.adiantamento`, `moverEntreGerencias`, `modal.cobranca`, `modal.avaliacao`, `desmembramento`, `popup_trocar_cnpj_fornecedor_pedido`.
- `nota_pagamento` (175.647 linhas): a maioria **não** liga a `nota_fiscal` (só 660 têm `codigo_nota_fiscal`, 130 nos últimos 12m). Pode ser do Contas a Pagar; tratar na fronteira entre domínios.

## 6. Perguntas para o Eduardo
1. O que gravou `codigo_erro_validacao_nf = 100` nas notas de 2010–2017? Foi uma validação manual, carga única, ou o job rodou por um período?
2. Por que a gravação parou em 2017/2018 (mudança de processo, de ferramenta, de regra)?
3. Os tipos 4–10 de NF (Transporte, Adiantamento, devoluções, OC/PC/*) ainda fazem parte do negócio, ou podem sair do escopo do sistema novo? (Decisão de escopo só depois do M5b; aqui é só levantamento.)
4. `Contrato` mora em `administracao/domain`, não em `compras/domain`. Confirma que contratos de fornecedores pertencem a Compras na fronteira do sistema novo?
