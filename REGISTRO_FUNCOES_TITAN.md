# Registro Mestre de Funções — TITAN Águas

Este arquivo registra as funções, capacidades e artefatos compartilhados do TITAN Águas.

| ID | Nome da função | Capacidade funcional | Interface | Local de criação | Local de aplicação | Domínio | Tipo | Natureza técnica | Regras principais | Aggregate | Use case | Endpoint/API | Dependências | Status | Prioridade | Responsável | Criticidade | Observações |
---

---

## Valores padronizados

Os campos da tabela devem seguir estritamente os padrões definidos abaixo, garantindo consistência em todo o ecossistema TITAN.

**Interface:** shared, admin, prestador, governanca_admin, governanca_prestador.

**Tipo:** frontend, backend, specification, taxonomy, contract, test, documentation.

**Natureza técnica:** agregado, projecao, comando, caso_de_uso, funcao_canonica, motor_de_regras, contrato, taxonomia, evento, worker, scheduler, auditoria, cache, api.

**Status:** nao_iniciado, em_desenvolvimento, em_validacao, pronto, descontinuado.

**Prioridade:** alta, media, baixa.

**Criticidade:** alta, media, baixa.

**Domínio funcional:** governanca_plataforma, identidade_seguranca_privacidade, estrutura_planta_estado, dados_operacionais_ativos, inteligencia_operacional, regulatorio_qualidade, workflows_relatorios.

---

## Regras de uso

Cada função, contrato, taxonomia, especificação ou componente técnico relevante deve possuir um identificador único.

Os identificadores devem seguir o padrão FUNC-0001, FUNC-0002, FUNC-0003, garantindo rastreabilidade ao longo de todo o sistema.

Funções canônicas compartilhadas devem ser implementadas exclusivamente no diretório shared/backend/domains/, evitando duplicidade entre interfaces.

Taxonomias oficiais devem ser mantidas em shared/taxonomies/, sendo a única fonte de verdade para estados, classificações e códigos.

Especificações devem ser documentadas em shared/specifications/<dominio>/, descrevendo regras de negócio, critérios e comportamento esperado.

Contratos devem ser definidos em shared/contracts/<dominio>/, garantindo alinhamento entre backend, frontend e integrações externas.

Nenhuma interface deve implementar lógica própria que já esteja definida como função canônica compartilhada.

O campo Aggregate / Projection deve identificar claramente o responsável pela consistência dos dados (modelo de domínio) ou a estrutura de leitura.

O campo Dependências deve listar explicitamente todas as taxonomias, dados ou funções necessárias para execução correta da capacidade.

O status só deve ser definido como "pronto" após validação técnica, testes mínimos e verificação de aderência à especificação.
