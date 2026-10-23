<!-- bmad:context -->
<!-- Verified 2026-10-22 against 599d377. Managed by bmad-project-context; edits inside this block are replaced on refresh. Keep anything you want preserved outside the markers. -->

## sgi-bmad

Extração de regras de negócio de um ERP legado descontinuado (SGI) para reconstrução greenfield; o código antigo não é reaproveitado. Fontes únicas: os repositórios `../sgi` (contas a pagar) e `../sgi.nv` (faturamento), complementares, e o banco `sgi.producao` (dump de 2023-06-19). Roadmap em `_bmad-output/planning-artifacts/roadmap.md`; Eduardo valida todas as regras.

## Policy

- Nunca altere `../sgi`, `../sgi.nv` nem `../dumps`: são cópias do legado, somente leitura. Registre achados em `_bmad-output/`.
- Nunca grave credenciais nem dados sensíveis (senhas, tokens, dados pessoais ou de negócio reais do dump) em documentação, código ou artefatos versionados. Use amostras mascaradas. Segredos ficam em arquivos de configuração não versionados; só a estrutura (`.sample`) é versionada.
- Leia a conexão do banco apenas do `.env` na raiz; nunca copie seus valores para outro arquivo, log ou resposta.
- O banco é só para análise (consultas de leitura). Pergunte a Eduardo antes de criar, alterar ou apagar qualquer objeto ou dado nele; se for necessário, ele pode ser restaurado do dump em `../dumps/prod-20230619.dmp.gz`.
- A regra do legado vem só do código e do banco (não há Wiki, Jira etc.). Pesquisa externa é permitida como enriquecimento, marcada como tal e com a fonte citada. Dúvida de negócio vai para Eduardo, não é suposta.
- Não defina escopo de MVP antes de o catálogo de regras estar validado (marco M5b do roadmap).

## Where things are

- Banco: container Docker `postgres17`, base `sgi.producao`, schemas `public` e `sgc`; use `docker exec` com os dados do `.env`.
- Marcos e decisões: `_bmad-output/planning-artifacts/roadmap.md`.

## Conventions that differ from defaults

- Documentos em português; regras com ID `RN-<DOMÍNIO>-nnn` e evidência no formato `arquivo:linha` ou `tabela`.
- Integrações externas (SEFAZ/NF-e, bancos, WhatsApp, impressão de cheques) são simuladas no sistema novo; SMS foi substituído por WhatsApp.

## Known pitfalls

- A regra de negócio está no código Java; o banco tem só 2 funções utilitárias e nenhum trigger. Não procure regras em triggers.
- As estatísticas de `pg_stat_user_tables` estão zeradas (dump sem `ANALYZE`); conte linhas com `count(*)`.

<!-- /bmad:context -->
