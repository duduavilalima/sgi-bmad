# Roadmap — Reconstrução Greenfield do SGI (extração de regras de negócio com BMad)

> Documento vivo. Revisar ao fechar cada marco (seção 6). Autor: John (PM) · Dono: Eduardo · Criado em 2026-10-22

## 1. Contexto e objetivo

O SGI é um ERP corporativo (Fasolti → grupo Araújo Abreu) com forte carga fiscal/contábil brasileira. Está em dois repositórios legados, e nenhum código será reaproveitado. **O ativo a preservar são as regras de negócio**; o produto novo nasce do zero.

**Objetivo:** extrair, validar e documentar as regras de negócio do legado, e usá-las como insumo (PRD → arquitetura → épicos) de um sistema novo.

**Fora de escopo:** migração de código, port linha a linha, manutenção do legado.

## 2. O que o levantamento inicial mostrou (fatos, não suposição)

| Item | `sgi` (legado 1) | `sgi.nv` (legado 2) |
|---|---|---|
| Stack | Java 7, JSF 1.2, RichFaces 3, Seam 2, Spring 2.5, Hibernate 3.2, Tomcat 7, CAS | Java 8, Spring Boot 2.1.5, Hibernate 5.2, front JS/SCSS, RabbitMQ para jobs/e-mail |
| Tamanho | ~1.255 arquivos `.java` (framework 355, core 327, controlefiscal 114) | ~1.574 arquivos `.java` (services.client 487, domain 467, services 181, rest 107) |
| Papel aparente | ERP original, módulos WAR por área | Reescrita parcial em camadas (domain/services/rest/jobs) |
| Testes | ~231 classes E2E Selenium/TestNG: **documentação executável de regras** | pasta `testes` presente |
| Docs | `OVERVIEW.md` bom, `PENDENCIAS_BUILD.md` | README só de infra |

**Banco** (`sgi.producao`, PostgreSQL 17.2, container `postgres17`, conexão validada pelo `.env`): dump de prod de **2023-06-19**.
- 2 schemas: `public` (246 tabelas) e `sgc` (32), no total 259 tabelas base e 19 views; **240 tabelas com dados**, 19 vazias.
- 518 FKs: o modelo relacional está bem declarado, o que ajuda muito a inferir regras.
- Só 2 funções (`fixsequence`, `resetsequence`) e 0 triggers: **a regra de negócio está no código Java, não no banco**.
- Maiores volumes: `item_lote_contabil` (7,3 mi), `meio_contato` (1,8 mi), `item_proposta` (1,7 mi), `item_requisicao_compra` (1,3 mi), `rateio_nota_fiscal` (1,1 mi), `nota` (799 mil), `pagamento` (743 mil).

**Domínios identificados:** Administrativo (cadastros mestres), Comercial, Compras (requisição → proposta → pedido/OC → recebimento → NF-e), Contas a Pagar, Contabilidade (lotes/interfaces FATC, FATN, ECF, PER/DCOMP, boletos), Controle Fiscal (o maior), Faturamento, Fundo Fixo, Tesouraria (impressão de cheques), Relatórios (BIRT), mais transversais (Segurança/CAS, Jobs/Quartz, E-mail/SMS).

## 2.1 Decisões confirmadas pelo dono do negócio (2026-10-22)

| # | Decisão |
|---|---|
| Q1 | O sistema foi **descontinuado**; os repositórios são a cópia do que rodava em produção. Os dois **conviviam**: `sgi` cobre sobretudo o fluxo de **contas a pagar** e `sgi.nv` o de **faturamento** (em linhas gerais). Os limites exatos por módulo ainda serão mapeados no M1. |
| Q2 | O dump de jun/2023 é o **mais recente** e definitivo. Não há dado mais novo. |
| Q3 | **Eduardo valida todas as regras** e responde às dúvidas ao longo da extração. Não há contador/fiscal à disposição. |
| Q4 | O sistema novo **começa limpo**: sem migração de dados nem cutover. O dump serve só como fonte de análise e como massa de dados de demonstração (mascarada). |
| Q5 | Sem restrição de stack, prazo ou orçamento. **Demonstração em serviços de nuvem gratuitos**; infra de produção reavaliada se for o caso. **Desenvolvimento todo em Docker**, para validar a modularidade. |
| Q6 | Os dois sistemas **compartilham os mesmos conceitos** e são **complementares, não excludentes**. Não há duas visões concorrentes a reconciliar, e sim partes do mesmo modelo. |
| Q8 | **SMS sai**; entra **WhatsApp**. **Todas as integrações externas são simuladas** (SEFAZ/NF-e, bancos/boletos, WhatsApp, impressão de cheques). |
| Q9 | Compras é exclusivo do `sgi`. NFs: *recebidas* (fornecedor) e *emitidas* (empresa/Faturamento). Validação automática de NF na SEFAZ nunca entrou em produção, mas deve existir no sistema novo (homologação SEFAZ se houver, senão simulada). |
| MVP | **Primeiro levantamos as regras de negócio; só depois, com o catálogo pronto, definimos o MVP** de implementação (marco M5b abaixo). Nada de escopo de construção antes disso. |

## 3. Riscos e premissas que orientam o plano

1. **Duas bases complementares.** `sgi` (contas a pagar) e `sgi.nv` (faturamento) compartilham conceitos. O M1 deve mapear a fronteira exata e os pontos de contato (cadastros, banco compartilhado) para que o mesmo conceito não seja extraído duas vezes nem com definições divergentes.
2. **Sistema parado desde 2023.** Como não há produção nem dado novo, o legado é a verdade do que o negócio *fazia*. As regras fiscais podem estar defasadas diante da legislação atual, e sem contador a validação depende de você e de pesquisa em fontes oficiais.
3. **Dados reais de produção** (pessoas, contatos, fornecedores): tratar sob LGPD. Os agentes devem usar amostras e dados mascarados nos artefatos versionados, nunca despejar dados brutos em `_bmad-output`.
4. **Regras fiscais mudam por lei.** Para regras fiscais não basta replicar o legado: é preciso validar contra a legislação vigente (NF-e, SPED, ECF, DIRF, reforma tributária). Isso vira marco próprio.
5. **Regra implícita.** Lógica espalhada em XHTML/Seam, validações em JS, relatórios BIRT e queries nativas: a extração precisa varrer essas camadas, não só os `ServiceImpl`.
6. **Escopo gigante.** São cerca de 2.800 arquivos Java. Extrair por **bounded context**, em fatias verticais, e nunca tudo de uma vez.

## 4. Abordagem

Para cada domínio, cruzar quatro fontes e só então escrever a regra:

```
 código (services/entidades/validações)  ─┐
 dados reais (distribuições, constraints) ─┼─►  Regra de negócio  ─► validada com você (dono do negócio)
 testes E2E / relatórios BIRT / telas     ─┤        (ID, fonte, evidência, exemplo)
 documentação e CHANGELOG                 ─┘
```

**Formato de cada regra:** `RN-<DOMÍNIO>-nnn` · descrição em linguagem de negócio · gatilho · condições · efeito · exceções · evidência (arquivo:linha / tabela / teste) · status (`extraída`, `confirmada`, `obsoleta`, `a decidir`).

**Mapeamento BMad:**

| Fase | Skill/agente |
|---|---|
| Descoberta e inventário | Mary (`bmad-agent-analyst`), `bmad-deep-recon` |
| Visão de produto | `bmad-product-brief`, `bmad-forge-idea` |
| Requisitos | John (`bmad-prd`) |
| UX | Sally (`bmad-ux`) |
| Arquitetura | Winston (`bmad-architecture`) |
| Épicos e histórias | `bmad-create-epics-and-stories` |
| Plano e execução | `bmad-sprint-planning`, Amelia (`bmad-build`) |
| Qualidade | Murat (`bmad-tea`), `bmad-testarch-*` |
| Mudança de rumo | `bmad-correct-course` |

## 5. Roadmap por marcos

Sem datas: o ritmo depende do seu tempo disponível. Cada marco tem **critério de saída verificável**.

### M0 — Fundações *(quase pronto)*
- [x] BMad instalado; repositórios clonados; banco no container acessível via `.env`.
- [x] Levantamento inicial (seção 2).
- [ ] `bmad-project-context` para registrar convenções do projeto de análise.
- [x] Questões Q1–Q5 respondidas (seção 2.1).

**Saída:** este roadmap aprovado por você.

### M1 — Inventário e mapa do legado
- Catálogo de módulos de cada repositório e **matriz de cobertura** `sgi` × `sgi.nv` por domínio (qual repositório implementa qual parte do mesmo conceito; os dois são complementares).
- Dicionário de dados: tabelas agrupadas por domínio, com volume, FKs e tabelas "mortas" (vazias ou sem uso no código).
- Mapa de funcionalidades: telas/endpoints/jobs/relatórios por domínio, cruzados com o uso real (dados recentes no dump).
- Priorização de domínios por valor × risco × volume.

**Saída:** `inventario.md`, `matriz-cobertura.md`, `dicionario-de-dados/` e ordem de ataque dos domínios aprovada por você.

### M2 — Visão de produto (Product Brief)
- Por que reconstruir; quem são os usuários e os processos; o que muda e o que se mantém.
- Dúvida Q7 (escopo do sistema novo): tratada como **análise de apoio à decisão** depois do levantamento, no marco do MVP. Aqui só se registra a visão, sem fixar escopo.
- Decisões sobre pontos estruturais: SSO, impressão de cheques, geração de arquivos fiscais, integração SEFAZ.

**Saída:** `product-brief.md` validado.

### M3 — Extração de regras, domínio piloto
Escolher **um** domínio de tamanho médio, com ciclo completo (sugestão: **Compras**, de requisição a recebimento), e fazer a extração de ponta a ponta para calibrar o método, o formato e o esforço.
- Catálogo `RN-COMPRAS-nnn` com evidências.
- Sessão de validação com você; ajuste do template.
- Medir tempo e custo por regra, para reestimar M4.

**Saída:** catálogo do piloto confirmado + template de regra congelado.

### M4 — Extração de regras, demais domínios (em ondas)
Ordem proposta (revisar após M1): *Cadastros mestres → Contas a Pagar → Faturamento → Tesouraria/Fundo Fixo → Contabilidade → Controle Fiscal (por último e em subfatias: NF-e, apuração, SPED/ECF/DIRF, PER/DCOMP).*
- Uma onda por domínio, cada uma com seu catálogo e sua validação.
- Regras fiscais: marcar as que dependem de legislação, para reconfirmar no M5.
- Backlog de **conflitos** entre os dois repositórios e de **regras obsoletas**.

**Saída:** 100% dos domínios do escopo da v1 com catálogo `confirmado`; lista de decisões pendentes zerada ou aceita.

### M5 — Validação fiscal e de conformidade
- Revisar as regras fiscais/contábeis contra a legislação vigente. Sem contador disponível, usar pesquisa em fontes oficiais (Receita, SEFAZ, portais SPED) e submeter cada conclusão à sua validação, marcando o nível de confiança.
- Definir tratamento de LGPD e retenção de dados.

**Saída:** parecer por regra fiscal (`mantém`, `muda`, `descontinua`).

### M5b — Definição do MVP *(só após M4 e M5)*
- Com o catálogo de regras completo e validado, comparar cenários de recorte (por exemplo: só contas a pagar; contas a pagar + faturamento; fluxo fim a fim reduzido) por valor, dependência entre regras, esforço e risco.
- Responde à Q7 com dados, não com palpite. A decisão é sua, com recomendação minha.
- Define o que entra no MVP, o que fica para depois e o que é descontinuado.

**Saída:** `mvp.md` aprovado: lista de regras (`RN-*`) no MVP, regras adiadas e critérios de sucesso.

### M6 — PRD
- PRD **do MVP definido em M5b**, com requisitos funcionais derivados do catálogo (rastreáveis `RN → RF`), requisitos não funcionais e critérios de aceite.
- Estratégia de dados: **começa limpo**, sem migração. Definir apenas a massa de demonstração, gerada a partir do dump com dados mascarados.

**Saída:** `prd.md` validado (`bmad-prd` em modo validação).

### M7 — UX e arquitetura
- Jornadas e telas-chave (Sally); `DESIGN.md` e `EXPERIENCE.md`.
- Arquitetura greenfield (Winston): stack, módulos, integrações, segurança, observabilidade.
- **Restrições já definidas:** ambiente de desenvolvimento 100% em Docker (um serviço por módulo, para provar a modularidade, com `docker compose`); demo hospedada em nuvem com camada gratuita (a escolha do provedor entra como decisão da arquitetura, com análise de limites e riscos de cada free tier).
- Estratégia de testes (Murat): as regras viram casos de teste de aceite.

**Saída:** documentos de UX e arquitetura aprovados.

### M8 — Épicos, histórias e prontidão
- Épicos por fatia vertical de valor; cada história referencia as `RN-*` que implementa.
- Verificação de prontidão e plano de sprints.

**Saída:** gate de prontidão aprovado; `sprint-status` gerado.

### M9 — Construção incremental
- Primeira fatia ponta a ponta (walking skeleton), depois por épico, com `bmad-build`, revisão e testes.
- Retrospectiva por épico; `bmad-correct-course` quando o rumo mudar.

**Saída:** definida no PRD (MVP em produção paralela com o legado, se aplicável).

## 6. Governança do roadmap

- **Revisão:** ao fim de cada marco, com três perguntas: o escopo mudou? a estimativa mudou? alguma premissa caiu?
- **Registro de mudanças:** anotar abaixo cada revisão.

| Data | Mudança | Motivo |
|---|---|---|
| 2026-10-22 | Versão inicial | Pedido do Eduardo |
| 2026-10-22 | Q6 e Q8 respondidas; Q7 adiada para o novo marco M5b (definição do MVP após o levantamento); `matriz-sobreposicao` renomeada para `matriz-cobertura` | Complementaridade dos sistemas e decisão do Eduardo de definir o MVP só após as regras |
| 2026-10-22 | Q1–Q5 respondidas: legado descontinuado, começa limpo, validação por Eduardo, Docker + demo em nuvem gratuita | Decisões do dono do negócio |

## 7. Questões abertas

Q1–Q6 e Q8 fechadas (ver 2.1). Em aberto:

| # | Pergunta | Marco |
|---|---|---|
| Q7 | O sistema novo cobre todos os domínios ou só um recorte no primeiro corte? Quero ajudar a decidir: **análise no marco M5b**, depois do levantamento das regras. | M5b |
