## Tabela principal

| ID | Nome da função | Capacidade funcional | Domínio | Interface | Local de criação | Local de aplicação | Área funcional | Tipo | Natureza técnica | Regras principais | Aggregate / Projection | Use case | Endpoint/API | Dependências | Status | Prioridade | Responsável | Criticidade | Observações |
|----|----------------|----------------------|---------|-----------|------------------|--------------------|----------------|------|------------------|-------------------|-------------------------|-----------|--------------|--------------|--------|------------|-------------|-------------|-------------|
| CAN-0001 | calcular_estado_planta | Determinar estado de implantação e operação da planta | AGU, EFL | ADM, PRT, GOV | 01_01 | aplicacao | estrutura_planta_estado | backend | funcao_canonica | Homologada != operacional; depende de dados válidos | Planta / Projection | CalcularEstadoPlanta | GET /plants/{id}/status | cadastro_planta, dados_operacionais | em_desenvolvimento | alta | Elaine | alta | Função base |
| FUNC-0002 |  |  |  |  |  |  |  |  |  |  |  |  |  |  | nao_iniciado |  |  |  |  |
| FUNC-0003 |  |  |  |  |  |  |  |  |  |  |  |  |  |  | nao_iniciado |  |  |  |  |
| FUNC-0004 |  |  |  |  |  |  |  |  |  |  |  |  |  |  | nao_iniciado |  |  |  |  |
| FUNC-0005 |  |  |  |  |  |  |  |  |  |  |  |  |  |  | nao_iniciado |  |  |  |  |
| FUNC-0006 |  |  |  |  |  |  |  |  |  |  |  |  |  |  | nao_iniciado |  |  |  |  |
| FUNC-0007 |  |  |  |  |  |  |  |  |  |  |  |  |  |  | nao_iniciado |  |  |  |  |
| FUNC-0008 |  |  |  |  |  |  |  |  |  |  |  |  |  |  | nao_iniciado |  |  |  |  |
| FUNC-0009 |  |  |  |  |  |  |  |  |  |  |  |  |  |  | nao_iniciado |  |  |  |  |
| FUNC-0010 |  |  |  |  |  |  |  |  |  |  |  |  |  |  | nao_iniciado |  |  |  |  |
| FUNC-0011 |  |  |  |  |  |  |  |  |  |  |  |  |  |  | nao_iniciado |  |  |  |  |

## Valores padronizados

Os campos da tabela devem seguir estritamente os padrões definidos abaixo, garantindo consistência em todo o ecossistema TITAN.

**Interface:** shared, admin, prestador, governanca_admin, governanca_prestador.

**Tipo:** frontend, backend, specification, taxonomy, contract, test, documentation.

**Status:** nao_iniciado, em_desenvolvimento, em_validacao, pronto, descontinuado.

**Prioridade:** alta, media, baixa.

**Criticidade:** alta, media, baixa.

**Área funcional:** governanca_plataforma, identidade_seguranca_privacidade, estrutura_planta_estado, dados_operacionais_ativos, inteligencia_operacional, regulatorio_qualidade, workflows_relatorios.

---

## Regras de uso

Cada função, contrato, taxonomia, especificação ou componente técnico relevante deve possuir um identificador único.

Funções canônicas compartilhadas devem ser implementadas exclusivamente no diretório shared/backend/domains/, evitando duplicidade entre interfaces.

Taxonomias oficiais devem ser mantidas em shared/taxonomies/, sendo a única fonte de verdade para estados, classificações e códigos.

Especificações devem ser documentadas em shared/specifications/<dominio>/, descrevendo regras de negócio, critérios e comportamento esperado.

Contratos devem ser definidos em shared/contracts/<dominio>/, garantindo alinhamento entre backend, frontend e integrações externas.

Nenhuma interface deve implementar lógica própria que já esteja definida como função canônica compartilhada.

O campo Aggregate / Projection deve identificar claramente o responsável pela consistência dos dados (modelo de domínio) ou a estrutura de leitura.

O campo Dependências deve listar explicitamente todas as taxonomias, dados ou funções necessárias para execução correta da capacidade.

O status só deve ser definido como "pronto" após validação técnica, testes mínimos e verificação de aderência à especificação.


## Naturezas técnicas padronizadas

| Código | Nome da natureza | Descrição técnica | Quando usar | Exemplo |
|--------|-----------------|------------------|-------------|---------|
| CAN | Função canônica | Regra central, única fonte de verdade, reutilizável por todo o sistema | Quando a lógica não pode ser duplicada e define comportamento global | calcular_estado_planta |
| RUL | Motor de regras | Avaliação de múltiplas condições com lógica dinâmica | Quando há várias regras configuráveis ou avaliáveis | validar_qualidade_dados |
| PRJ | Projeção | Consolidação de dados para leitura, sem alterar estado | Quando a função alimenta dashboards ou consultas | listar_funcionalidades_ativas |
| USE | Caso de uso | Ação orquestradora que executa fluxo de negócio | Quando coordena múltiplos componentes | processar_homologacao |
| CMD | Comando | Operação que altera estado do sistema | Quando há mudança de estado persistida | ativar_funcionalidade |
| API | API | Interface externa de acesso ao sistema | Quando expõe dados para frontend ou integração | GET /plants/{id}/status |
| EVT | Evento | Notificação de mudança de estado | Quando algo relevante ocorre no sistema | status_planta_atualizado |
| WRK | Worker | Processamento assíncrono ou em lote | Quando roda em background ou fila | recalcular_indicadores |
| SCH | Scheduler | Execução programada recorrente | Quando há execução periódica automática | job_sinisa_mensal |
| AGG | Agregado | Entidade raiz que controla consistência (DDD) | Quando representa o dono dos dados | Planta |
| PRO | Projeção de leitura | Modelo derivado para consulta otimizada | Quando representa leitura estruturada | PlantStatusProjection |
| CTR | Contrato | Estrutura de dados formal (API/evento) | Quando define entrada/saída padronizada | plant_status_response.json |
| TAX | Taxonomia | Lista de valores padronizados | Quando define estados, tipos ou categorias | plant_status.json |
| SPC | Especificação | Documento de regra técnica/negócio | Quando descreve comportamento esperado | plant_status_calculator.md |
| TST | Teste | Validação automatizada de comportamento | Quando garante funcionamento correto | test_calcular_estado_planta |
| AUD | Auditoria | Registro rastreável de ações | Quando há necessidade de compliance | log_alteracao_estado |
| CCH | Cache | Otimização de acesso a dados | Quando evita recomputação | cache_status_planta |
| VAL | Validador | Validação de consistência de dados | Quando valida entrada ou integridade | validar_cadastro_planta |
| ORQ | Orquestrador | Coordenação de múltiplos processos | Quando integra vários domínios | orquestrar_implantacao_planta |

