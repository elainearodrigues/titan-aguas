# TITAN Águas — Função Canônica de Estado da Planta

**Arquivo:** `shared/specifications/plant_status_calculator.md`  
**Status:** especificação canônica para todo o TITAN Águas  
**Escopo:** admin, prestador, governanca_admin e governanca_prestador

## Finalidade

Esta especificação define a regra oficial para cálculo do estado de uma planta no TITAN Águas. Nenhuma interface deve calcular estado da planta por conta própria.

## Separação obrigatória

A planta possui duas dimensões:

```text
estado_implantacao
estado_operacional
```

Porque:

```text
Homologada = apta a operar
Operacional = operando de fato com dados confiáveis
```

## Estados de implantação

### EM_CADASTRAMENTO
Planta em criação no TITAN, com dados cadastrais obrigatórios ainda pendentes.

### EM_CONFERENCIA_CADASTRAL
Todos os dados cadastrais obrigatórios foram preenchidos, inclusive dados importados, mas ainda precisam ser conferidos tecnicamente.

### EM_CALIBRACAO_VALIDACAO_DADOS
A estrutura cadastral foi conferida, mas instrumentos, tags, integrações e dados ainda precisam ser calibrados ou validados.

### AGUARDANDO_HOMOLOGACAO
A planta está tecnicamente pronta, mas ainda depende de aprovação formal.

### HOMOLOGADA
A planta foi formalmente aceita pela equipe autorizada do TITAN e está apta a entrar em operação.

**Homologada não significa operacional automaticamente.**

## Estados operacionais

### NAO_OPERACIONAL
Planta ainda não opera no TITAN.

### OPERACIONAL
Planta homologada, ativa, com ingestão regular de dados válidos e sem falha crítica.

### OPERACIONAL_RESTRITO
Planta homologada e ativa, com limitações conhecidas que reduzem o escopo ou a completude das análises, mas ainda permitem uso parcial confiável.

### DEGRADADA
Planta homologada cuja confiabilidade no TITAN está comprometida por falha relevante de dados, comunicação, integração ou calibração.

### INATIVA
Planta mantida no cadastro, mas fora da operação ativa.

### ENCERRADA
Planta definitivamente descontinuada no TITAN. Estado terminal.

## Ordem de precedência

```text
1. ENCERRADA
2. INATIVA
3. EM_CADASTRAMENTO
4. EM_CONFERENCIA_CADASTRAL
5. EM_CALIBRACAO_VALIDACAO_DADOS
6. AGUARDANDO_HOMOLOGACAO
7. HOMOLOGADA
8. DEGRADADA
9. OPERACIONAL_RESTRITO
10. OPERACIONAL
```

## Saída padrão

```json
{
  "estado_implantacao": "HOMOLOGADA",
  "estado_operacional": "OPERACIONAL_RESTRITO",
  "grupo_dashboard": "operacao_com_restricao",
  "justificativas": [
    "Planta opera com limitações conhecidas, mas possui dados mínimos confiáveis."
  ]
}
```
