# Inventário do legado — M1

> Documento vivo, preenchido por domínio. Autora: Mary (Analista) · Validação: Eduardo
> Fontes: `../sgi`, `../sgi.nv`, banco `sgi.producao` (dump 2023-06-19). Somente leitura. Contagens são agregadas; nenhum dado de negócio foi copiado.

| Domínio | Status |
|---|---|
| **Compras** | 1ª passada concluída (abaixo); falta cruzar telas × uso e validar a fronteira com Eduardo |
| Demais | pendentes |

---

## 1. Compras

### 1.1 Onde o código mora

O módulo "Compras" não se limita à pasta `sgi/compras`. Ele está espalhado em camadas:

| Camada | Local | Arquivos .java | Conteúdo |
|---|---|---|---|
| UI (JSF/Seam) | `sgi/compras/src/main/java/.../compras/faces` | 44 | 32 FacesBeans + 8 detail + 2 support + 2 jobs |
| Telas | `sgi/compras/src/main/webapp` | 142 `.xhtml` | requisição, OC (cotação, classificação, autorização, recebimento), notas fiscais, relação de NF, contratos de fornecedor, avaliação, lote de validação de NF |
| Serviços/regra | `sgi/core/.../compras/{service,lang,job}` | 40 | `ComprasServiceImpl`, `NotaFiscalServiceImpl`, `ValidacaoNotaFiscalServiceImpl`, `AvaliacaoFornecedorServiceImpl`, clientes NF-e, mailers, exportação Excel |
| Entidades | `sgi/framework/.../compras/domain` | 58 | modelo JPA (Hibernate 3) |
| Relatórios | `sgi/**/compra*` | 23 `.rptdesign`/`.jrxml` (a classificar) | BIRT |
| Testes E2E | `sgi/testes` (caminho com "compra") | 57 | Selenium/TestNG: documentação executável |
| Entidades (nv) | `sgi.nv/domain/.../compras/{domain,enums,interf}` | 54 | mesmas tabelas, reescritas |
| Repositórios (nv) | `sgi.nv/domain/.../repository/{compras,ordemcompra,requisicaocompra}` | 4 | NotaDebito, NotaFiscal, OrdemCompra, RequisicaoCompra |
| Serviços (nv) | `sgi.nv/services/.../impl/compras` | 3 | `NotaPagamento`, `NotaSaida`, `ParcelaNota` |

### 1.2 Subprocessos identificados (cadeia)

`Requisição de compra → Cotação/Proposta → Classificação (melhor proposta/negociação) → Autorização da OC → Pedido → Recebimento (+ rateio) → Nota fiscal de entrada (+ validação RFB/NF-e, relação de NF) → Pagamento`

Satélites: contratos de fornecedor, avaliação de fornecedor, catálogo (item/categoria/NCM/CFOP), lote de validação de NF, notas avulsas, nota de saída (parcelas, retenções, motivos de desconto/devolução).

### 1.3 Jobs

| Job | Repo | Observação |
|---|---|---|
| `ValidacaoNotaFiscalJob` / `JobSchedulerCompras` | sgi | Consulta NF-e de fornecedor na SEFAZ em lotes (padrão 01h30–03h30, ter–sáb; parâmetro `compras.notafiscal.validacao_rfb.crontab`), grava resultado, e-mail ao perfil `admin-nf_e`. **Nunca entrou em produção; funcionalidade a manter** (SEFAZ de homologação se existir; senão simulada) |
| `PagamentoScheduleProcessor` | sgi (core) | a confirmar escopo |
| `NotaFiscalMaterialJob`, `NotasEmailJob` | sgi.nv | a confirmar relação com Compras |

### 1.4 Dados (tabelas de Compras e volume)

Contagem por `count(*)` em 2026-10-22. Janela de datas: **2010-05 a 2023-06-19** (há registros até o último dia do dump).

| Grupo | Tabela: linhas |
|---|---|
| Requisição | `requisicao_compra` 265 mil · `requisicao_compra_refinada` 435 mil · `item_requisicao_compra` 1,26 mi |
| Cotação | `proposta` 521 mil · `item_proposta` 1,67 mi · `fornecedor_habilitado` 532 mil |
| OC/Pedido | `ordem_compra` 496 mil · `pedido` 388 mil · `item_pedido` 1,08 mi |
| Recebimento | `recebimento` 338 mil · `item_recebimento` 996 mil · `rateio_item_recebimento` 647 mil |
| Nota de entrada | `nota` 799 mil · `nota_fiscal` 623 mil · `item_nota_fiscal` 750 mil · `rateio_nota_fiscal` 1,15 mi · `parcela_nota` 702 mil · `nota_pagamento` 176 mil · `relacao_nf` 83 mil · `relacao_nf_nf` 362 mil |
| Nota de saída | `nota_saida` 138 mil · `nota_saida_parcela` 138 mil · `item_nota_saida` 261 |
| Catálogo | `item_compra` 59 mil · `categoria_item_compra` 1,8 mil · `item_compra_conta_contabil` 181 mil · `ncm` 17 mil · `cfop` 552 · `unidade_medida` 30 · `natureza_aquisicao` 3 · `tipo_cobranca` 7 |
| Contratos/avaliação | `contrato*`, `avaliacao`, `item_avaliacao`, `faixa_avaliacao` (volumes a medir) |
| Vazias (funcionalidade não entrou em produção) | `lote_validacao_nf`, `lote_validacao_nf_registro` = 0 linhas. **Decisão do Eduardo:** o job automático foi implementado e nunca entrou em produção; o código pode estar obsoleto, mas a funcionalidade deve existir no sistema novo (hoje a validação de NF é manual) |
| Quase vazias | `erro_validacao_nf` 44 · `item_nota_saida` 261 · `base_calculo_credito` 18 · `documento_fiscal` 16 |

OCs por ano: pico em 2017 (65 mil); 2020–2022 ≈ 29 mil/ano; 2023 até junho = 16,5 mil. O módulo estava em **uso ativo até a data do dump**.

**Tipos de nota (`nota_fiscal.tipo_nota_fiscal`)** — dois grupos, confirmados por Eduardo:
- **NFs recebidas** (NF de fornecedor), tipos 0–10 exceto faturamento: ~485 mil. Ciclo de vida longo, atravessa vários módulos.
- **NFs emitidas** pela própria empresa (Faturamento), tipos 11 e 12: ~138 mil; bate com `nota_saida` (137.838).

Observação: `nota_fiscal.codigo_erro_validacao_nf` está preenchido em 107.509 notas, apesar de o job nunca ter rodado em produção. A origem (validação manual?) está **a investigar**.

Observação: `nota.data_criacao` só vai de 2010-05 a 2011-04, enquanto `nota_fiscal` segue até 2023. Provável herança JOINED em que a data de `nota` deixou de ser preenchida: **a investigar** antes de usar essa coluna como indicador de uso.

---

## Pendências desta passada

1. Cruzar as 142 telas com o uso real, por tabela (lote de validação já aparece como morto).
2. Classificar os 23 relatórios.
3. Confirmar o escopo de `PagamentoScheduleProcessor` e dos jobs do `sgi.nv`.
4. Dicionário de dados de Compras (FKs, colunas), em `dicionario-de-dados/compras.md`.
