# CLAUDE.md — ERP Contábil Multi-Tenant (Brasil)

## Status Atual do Projeto
- **Fase atual:** Estudo de estrutura monolítica modular e consolidação de documentação/glossário (Rodada 10 aberta / "Pronto para construir", com código de negócio aguardando a partida da implementação após o estudo do monolito).
- **Ambiente local ativo:** RHEL (WSL) com Jetty 12.1.15 e JVM 21 servindo o glossário em `http://localhost:8080/glossario/` via `ROOT.war`.
- **Pendências de validação contábil (`[VALIDAR_CONTADOR]`):**
  1. Escopo detalhado de poderes/serviços na `ProcuracaoEletronica`.
  2. Regra de retroatividade para `EnquadramentoTributarioRetificado` quando a `Competencia` já estiver fechada.
  3. Campos mínimos obrigatórios de `CadastrarEmpresa`, antecedência de alertas de vencimento e detalhes operacionais de retenção no encerramento de vínculo.

## Arquitetura e Guardrails (Monolito Modular + DDD)
- **Módulo dono principal:** `/cadastro` (contextos delimitados *Cadastro de Entidades* e *Certificados Digitais*, schema `cadastro`), integrado de forma assíncrona via *Transactional Outbox* com *Contabilidade*, *Apuração Fiscal*, `/fiscal-worker`, `/zimbra-connector` e *Auditoria*.
- **Isolamento modular:**
  - Respeitar fronteiras explícitas entre módulos (`api`/`ports` públicos vs. detalhes internos).
  - **Proibido** fazer `JOIN` ou `FK` cruzados entre schemas de módulos distintos.
- **Multi-tenancy, Segurança e LGPD:**
  - Isolamento estrito por `tenant_id` (`Escritório`) em consultas, comandos, projeções e trilhas de auditoria.
  - Aplicar mascaramento em dados pessoais e documentos fiscais conforme a LGPD.
  - Manter trilha de auditoria *append-only* com `ocorrido_em`, `registrado_em` e `correlation_id`.
  - **Nunca** armazenar arquivos `.pfx`/`.p12` ou senhas de certificado no banco, logs ou eventos — usar apenas `ReferenciaSegredoCertificado` resolvido via Secret Manager.
- **Resiliência e Contratos:**
  - Envelope padrão de eventos com `event_id`, `event_type`, `schema_version`, `tenant_id`, `empresa_id`, `ocorrido_em`, `registrado_em` e `correlation_id`.
  - Usar `Idempotency-Key`, *retry* e *DLQ* (*Dead Letter Queue*) na comunicação assíncrona.
  - Respostas de erro HTTP em `application/problem+json`.

## Glossário da Linguagem Onipresente
- **`TenantId` / `Escritório`:** Identificador do escritório contábil que delimita o escopo multi-tenant.
- **`Empresa` / `EmpresaId`:** Raiz de agregado da entidade contribuinte vinculada ao escritório.
- **`Certificado`:** Raiz de agregado do certificado digital A1; guarda metadados e `ReferenciaSegredoCertificado` (sem material criptográfico bruto ou senhas).
- **`ProcuracaoEletronica`:** Raiz de agregado que representa outorga de poderes/representação dentro de uma `Vigencia`.
- **`Vigencia`:** Objeto de valor de intervalo temporal (`data_inicio_vigencia` e `data_fim_vigencia` opcional) com validação de coerência e operações fechadas.
- **`Competencia`:** Objeto de valor de período contábil/fiscal no formato `AAAA-MM`.
- **`DocumentoFiscal`:** Objeto de valor para `CNPJ` e `CPF` com validação e mascaramento LGPD.
- **`VinculoEmpresaEncerrado`:** Regra tácita/evento que encerra operação ativa e novos alertas no Zimbra, mantendo histórico e auditoria em leitura.
- **`correlation_id` & `Idempotency-Key`:** Chaves de rastreabilidade ponta a ponta e deduplicação de requisições/eventos.
- **`Outbox` & `DLQ`:** Mecanismos de publicação confiável de eventos e isolamento de falhas persistentes.

## Convenções de Nomenclatura e Git
- **Código e Domínio:** Nomes de conceitos contábeis/fiscais na Linguagem Onipresente (`Empresa`, `Certificado`, `ProcuracaoEletronica`, `Vigencia`, `Competencia`, `DocumentoFiscal`, `TenantId`, `EmpresaId`), em PascalCase/camelCase conforme a linguagem e `snake_case` em campos de eventos/JSON/banco quando aplicável.
- **Git no RHEL:** Trabalhar em branches temáticas curtas (ex.: `docs/glossario-claude`, `arch/monolito-modular`), manter `main` estável, usar `git fetch --all --prune` e `git pull --rebase` antes de enviar, e usar tags anotadas para marcos consolidados.
