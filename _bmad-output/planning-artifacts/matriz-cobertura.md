# Matriz de cobertura `sgi` × `sgi.nv` — M1

> Os dois repositórios são complementares (Q6). Legenda: ● implementa a regra/serviço · ◐ só modelo/consulta · ○ ausente.
> Preenchida por domínio. Autora: Mary · Validação: Eduardo

## Compras

| Subprocesso / conceito | `sgi` (UI + serviço + modelo) | `sgi.nv` | Evidência | Confiança |
|---|---|---|---|---|
| Requisição de compra | ● `RequisicaoCompraFacesBean`, `ComprasServiceImpl`, entidade | ◐ entidade + `RequisicaoCompraRepository` | sgi/compras/.../faces; sgi.nv/domain/.../repository/requisicaocompra | alta |
| Cotação / proposta | ● `PropostaFacesBean`, `OrdemCompraReceberCotacaoFacesBean` | ◐ entidades `Proposta`, `ItemProposta` | idem | alta |
| Classificação / negociação | ● `ClassificarMelhorPropostaFacesBean`, `OrdemCompraClassificarFacesBean` | ○ | idem | alta |
| Autorização (requisição e OC) | ● `AutorizarRequisicaoCompraFacesBean`, `OrdemCompraAutorizarFacesBean` | ○ | idem | alta |
| Pedido | ● `PedidoFacesBean`, exportação Excel | ◐ entidade | idem | alta |
| Recebimento + rateio | ● `OrdemCompraReceberFacesBean` | ◐ entidades | idem | alta |
| NF recebida (de fornecedor, tipos 0–10) | ● `NotaFiscalServiceImpl` + telas | ◐ `NotaFiscalServiceImpl`, `NotaServiceImpl` (a ler: parece serviço de apoio ao contas a pagar/faturamento) | sgi/core/.../NotaFiscalServiceImpl; sgi.nv/services/.../impl/nota | média |
| Validação NF na SEFAZ (lote/job) | ● `ValidacaoNotaFiscalJob` (implementado, **nunca em produção**; funcionalidade a manter) | ○ | tabelas `lote_validacao_nf*` = 0; Eduardo 2026-10-22 | alta |
| Relação de NF | ● `RelacaoNotaFiscalFacesBean` | ◐ entidade | idem | média |
| NF emitida / nota de saída (tipos 11–12) / parcelas | ◐ serviços `NotaSaida*` em core | ● `NotaSaidaServiceImpl`, `NotaSaidaParcelaServiceImpl` (alinha com faturamento) | sgi.nv/services/.../compras | média |
| Nota de pagamento / parcela de nota | ◐ `PagamentoScheduleProcessor` | ● `NotaPagamentoServiceImpl`, `ParcelaNotaServiceImpl` (alinha com contas a pagar) | sgi.nv/services/.../compras | média |
| Nota de débito | ○ | ● `NotaDebito`, `NotaDebitoServiceImpl` (única entidade nova do nv) | sgi.nv/domain | alta |
| Contratos de fornecedor | ● `ContratoFornecedorFacesBean` | ● `ContratoServiceImpl` (a comparar) | idem | baixa |
| Avaliação de fornecedor | ● `AvaliacaoFornecedorServiceImpl` | ○ | sgi/core | alta |
| Cadastros (item, categoria, NCM, CFOP) | ● `CategoriaFacesBean` | ◐ entidades | idem | média |
| Impressão/exportação (Excel de OC/pedido/proposta, e-mail) | ● | ○ | sgi/core/.../service/{impl,support} | alta |

### Decisões do Eduardo (2026-10-22)

1. Compras é exclusivo do `sgi`; o `sgi.nv` só reutiliza as entidades.
2. NFs dividem-se em **recebidas** (fornecedor; atravessam vários módulos) e **emitidas** (empresa; Faturamento). A separação por `tipo_nota_fiscal` reflete exatamente esses grupos.
3. Validação de NF na SEFAZ: funcionalidade a existir (hoje é manual); usar homologação da SEFAZ se disponível, senão simular.

### Leitura da matriz (hipóteses restantes)

1. (Confirmada) **Fluxo de requisição → recebimento é só do `sgi`**: o `sgi.nv` tem o modelo (mesmas tabelas, base compartilhada) mas **nenhum serviço ou tela** desse fluxo. Logo, para o piloto de Compras o `sgi` é a fonte de regra; o `sgi.nv` serve de segundo olhar sobre o modelo.
2. A NF recebida percorre Compras → Contas a Pagar → Contabilidade/Fiscal; a regra de cada etapa deve ficar no domínio dono da etapa, e o ciclo de vida da NF será mapeado de ponta a ponta.
3. As entidades do `sgi.nv` são reescritas (mesmas tabelas; `AuditEntity` sem `AuditableEntity`, fetch LAZY no lugar de EAGER, e o `enum Status` não aparece mais dentro de `RequisicaoCompra`: ainda não localizei onde foi parar). As **diferenças entre as duas versões da mesma entidade** podem esconder regras que mudaram: auditar na extração.
