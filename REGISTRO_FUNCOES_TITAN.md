# REGISTRO_FUNCOES_TITAN

Índice centralizado de todos os artefatos funcionais do repositório TITAN. Uma linha por artefato. Fonte de verdade para saber o que existe, o que falta e o que bloqueia o quê. Atualizar a cada commit que crie, altere status ou remova um artefato. Ordenação: por natureza (CAN → TAX → SPE → CON → TST → DOC → WFR), dentro de cada natureza por ID crescente.

| ID | Natureza | Nome funcional | Escopo | Camada | Status | Pendências | Caminho |
|---|---|---|---|---|---|---|---|
| CAN001 | CAN | calcular_estado_operacional_planta | global | backend | definido | — | shared/taxonomies/CAN001_calcular_estado_operacional_planta/ |
| TAX001 | TAX | estado_planta | global | shared | definido | — | shared/taxonomies/TAX001_estado_planta.json |
| SPE001 | SPE | renderizar_cabecalho_pagina | global | frontend | definido | TAX002, CON001 | shared/specifications/SPE001_renderizar_cabecalho_pagina.md |
