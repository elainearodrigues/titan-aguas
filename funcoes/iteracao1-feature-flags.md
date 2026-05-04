# Iteração 1 — Capacidade: Feature Flags

> Este documento é a **iteração 1** da especificação da capacidade Feature Flags, primeira de 6 capacidades da sub-página `01-01 Recursos & Funcionalidades`. Serve para validar o template de capacidade antes de aplicar às demais 5.
>
> **Não é o documento final.** É um teste do formato.

---

## 3.1 CAPACIDADE: Feature Flags

### 3.1.1 Identificação

| Campo | Valor |
|---|---|
| ID canônico | `CAP-AGUAS-01-01-FEATURE-FLAGS` |
| Sub-página | `ADM-AGUAS-01-01` (Recursos & Funcionalidades) |
| Versão da especificação | 1.0-iter1 |
| Status | rascunho |
| Owner técnico | Equipe Núcleo TITAN Águas |
| Owner de produto | Diretoria de Águas |
| Data | 2026-05-02 |

### 3.1.2 Intenção

**Qual problema concreto resolve.** Diferentes prestadores-cliente do TITAN têm maturidades diferentes, contratos diferentes, diretorias com prioridades diferentes. Funcionalidades novas (ex: integração SISAGUA refinada, modelo IA de coagulação Bayesiana, módulo de fase 3 de distribuição) precisam ser **disponibilizadas seletivamente**: para alguns prestadores antes de outros, em algumas plantas antes de outras, com rollout gradual para reduzir risco de regressão. Sem feature flags, lançar funcionalidade nova significa lançar para todos simultaneamente — risco operacional inaceitável em sistema regulatório de saneamento.

**Quem usa, com que frequência.** Equipe TITAN (papel meta-administrador para flags globais; analista TITAN para flags de prestador específico). Frequência típica: ~5-15 mudanças por mês por prestador maduro; ~1-3 por mês após estabilização. Picos durante onboarding de prestador novo ou ativação de fase evolutiva.

**O que aconteceria se não existisse.** Sem feature flags, qualquer mudança no produto vai para 100% dos prestadores no momento do deploy. Bug em modelo Bayesiano de coagulação afetaria todas as ETAs Sabesp simultaneamente — risco operacional (sub-dosagem ou super-dosagem de coagulante) com impacto em qualidade de água potável de milhões de pessoas. Feature flag desacopla deploy de release, permite rollback granular sem rollback de código.

**Drivers normativos específicos.** Não há driver normativo direto. É decisão de arquitetura de produto motivada por gestão de risco operacional. Indiretamente, sustenta cumprimento de SLA contratual (TR §15.1) ao reduzir incidência de incidentes em produção.

### 3.1.3 Glossário local

| Termo | Definição |
|---|---|
| **Feature flag** | Aggregate que representa uma chave de controle binário (ativa/inativa) de uma funcionalidade específica do TITAN, com escopo, rollout percentual e estado próprios. |
| **Chave** | Identificador textual único e estável da flag (ex: `gemeo_digital_v2`, `sinisa_export_404`). Imutável após criação. |
| **Escopo** | Universo de aplicação da flag: `global` (toda a plataforma), `prestador` (apenas um prestador específico), `planta` (apenas uma planta específica), `domínio` (Águas ou Efluentes). |
| **Rollout percentual** | Inteiro 0–100 que define a fração de avaliações que retornam estado `ativo`. Permite ativação gradual (ex: 10% → 30% → 60% → 100%). |
| **Avaliação** | Operação que, dado um contexto (prestador, planta, domínio), retorna o estado efetivo da flag (ativa ou inativa) considerando escopo e rollout. |
| **Owner** | Equipe ou área responsável pela flag (ex: `Diretoria de Águas`, `Núcleo Águas`, `Equipe regulatória`, `P&D`). Não confundir com papel TITAN; é categorização organizacional. |
| **Aplicada a** | Texto descritivo do alcance efetivo da flag baseado em escopo e rollout (ex: "Todas", "4 plantas", "1 planta", "Sabesp +4"). É derivado, não persistido. |

### 3.1.4 Aggregate e invariantes

#### Aggregate root: `FeatureFlag`

```python
from typing import Annotated, Literal, Optional
from pydantic import BaseModel, Field, StringConstraints
from datetime import datetime
from uuid import UUID

ChaveFlag = Annotated[
    str,
    StringConstraints(min_length=3, max_length=80, pattern=r'^[a-z][a-z0-9_]*$')
]

EscopoTipo = Literal['global', 'prestador', 'planta', 'dominio']
EstadoFlag = Literal['rascunho', 'ativa', 'inativa', 'arquivada']
DominioAlvo = Literal['aguas', 'efluentes', 'ambos']

class FeatureFlag(BaseModel):
    id: UUID
    chave: ChaveFlag
    descricao: Annotated[str, StringConstraints(min_length=10, max_length=500)]
    escopo_tipo: EscopoTipo
    escopo_valor: Optional[UUID] = None
    dominio_alvo: DominioAlvo
    rollout_percentual: Annotated[int, Field(ge=0, le=100)]
    estado: EstadoFlag
    owner: Annotated[str, StringConstraints(min_length=3, max_length=100)]
    criado_em: datetime
    criado_por_usuario_id: UUID
    atualizado_em: datetime
    atualizado_por_usuario_id: UUID
    versao: Annotated[int, Field(ge=1)]
```

#### Invariantes (sempre verdadeiras)

| ID | Invariante | Verificação |
|---|---|---|
| `INV-FF-01` | `chave` é única globalmente | UNIQUE constraint em SQL |
| `INV-FF-02` | `chave` é imutável após criação | Trigger de banco rejeita UPDATE de coluna `chave` |
| `INV-FF-03` | `escopo_tipo = 'global'` ⟹ `escopo_valor IS NULL` | CHECK constraint em SQL |
| `INV-FF-04` | `escopo_tipo ∈ {'prestador', 'planta'}` ⟹ `escopo_valor IS NOT NULL` | CHECK constraint em SQL |
| `INV-FF-05` | `escopo_tipo = 'dominio'` ⟹ `escopo_valor IS NULL ∧ dominio_alvo ∈ {'aguas', 'efluentes'}` | CHECK constraint em SQL |
| `INV-FF-06` | `0 ≤ rollout_percentual ≤ 100` | Domínio do tipo + CHECK |
| `INV-FF-07` | `estado = 'inativa'` ⟹ `rollout_percentual` pode ser qualquer valor (preservado mas não consultado) | Lógica de aplicação |
| `INV-FF-08` | `versao` é estritamente crescente | Trigger incrementa em UPDATE |
| `INV-FF-09` | `escopo_valor` (quando UUID) referencia entidade existente e ativa | Validação aplicacional + FK quando aplicável |
| `INV-FF-10` | `chave` segue regex `^[a-z][a-z0-9_]*$` (snake_case minúsculo, começa com letra) | Tipo Pydantic + CHECK |

#### Relacionamentos com outros aggregates

| Aggregate relacionado | Tipo de relação | Onde mora |
|---|---|---|
| `Usuario` | `criado_por_usuario_id` e `atualizado_por_usuario_id` referenciam Usuario | `01-02 Identidade & acesso` |
| `Prestador` | `escopo_valor` quando `escopo_tipo = 'prestador'` | `09 Prestadores` |
| `Planta` | `escopo_valor` quando `escopo_tipo = 'planta'` | `12 Plantas` |
| `EventoAuditoria` | Toda mutação publica evento de domínio que vira entrada de auditoria | `01-05 Auditoria & retenção` |
| `Permissao` | Operações sobre FeatureFlag são governadas por permissões em recurso `feature_flag` | `01-02 Identidade & acesso` |

### 3.1.5 Máquina de estados

```
                 (criar)
                    ↓
              ┌─ rascunho ─┐
              │            │
    (publicar)│            │(arquivar antes de publicar)
              ↓            ↓
            ativa        arquivada (terminal)
              │
              │← → inativa (toggle bidirecional)
              │
              ↓ (arquivar)
           arquivada (terminal)
```

#### Estados

| Estado | Descrição | Permite avaliação? |
|---|---|---|
| `rascunho` | Flag criada mas não publicada. Não afeta sistema. | Não |
| `ativa` | Flag publicada e ligada. Avaliação respeita rollout. | Sim |
| `inativa` | Flag publicada mas desligada. Avaliação retorna sempre `false`. | Sim (sempre false) |
| `arquivada` | Flag descomissionada. Estado terminal. Avaliações retornam erro `FF_ARCHIVED`. | Não |

#### Transições válidas

| Origem | Destino | Operação | Pré-condição |
|---|---|---|---|
| (não existe) | `rascunho` | `criar_feature_flag` | Permissão `criar` em `feature_flag`; chave única |
| `rascunho` | `ativa` | `publicar_feature_flag` | Permissão `aprovar` em `feature_flag`; meta-admin se escopo=global |
| `rascunho` | `arquivada` | `arquivar_feature_flag` | Permissão `excluir` em `feature_flag` |
| `ativa` | `inativa` | `desativar_feature_flag` | Permissão `atualizar` em `feature_flag` |
| `inativa` | `ativa` | `reativar_feature_flag` | Permissão `atualizar` em `feature_flag` |
| `ativa` | `arquivada` | `arquivar_feature_flag` | Permissão `excluir` + flag não consumida em runtime há ≥30 dias |
| `inativa` | `arquivada` | `arquivar_feature_flag` | Permissão `excluir` |

#### Transições inválidas (rejeitadas com erro `FF_TRANSICAO_INVALIDA`)

`arquivada → *` (qualquer destino) · `ativa → rascunho` · `inativa → rascunho` · `(não existe) → ativa` (atalhos não permitidos).

### 3.1.6 Operações e tabela de decisão

Esta é a tabela de decisão exaustiva da capacidade. **Para cada operação, todas as combinações relevantes de input × pré-condição estão tabuladas.** Ambiguidade aqui é defeito da especificação.

#### Op `OP-FF-CRIAR` — criar feature flag

| Cenário | Pré-condição | Input válido? | Comportamento | Estado resultante | Eventos | Persistência | Erro retornado | HTTP |
|---|---|---|---|---|---|---|---|---|
| Caminho feliz | usuário tem permissão `criar` em `feature_flag` ∧ chave não existe | sim | cria registro | `rascunho`, `versao=1` | `FeatureFlagCriada` | INSERT em `feature_flag` | — | 201 |
| Sem permissão | usuário não tem permissão | irrelevante | rejeita imediatamente | inalterado | `AcessoNegado` em audit_log | nenhum INSERT | `FF_PERMISSAO_NEGADA` | 403 |
| Chave duplicada | chave já existe (em qualquer estado, inclusive arquivada) | input válido mas chave duplicada | rejeita | inalterado | `OperacaoRejeitada` em audit_log | nenhum INSERT | `FF_CHAVE_DUPLICADA` | 409 |
| Chave malformada | chave não casa regex `^[a-z][a-z0-9_]*$` | não | rejeita pré-INSERT (validação Pydantic) | inalterado | nenhum | nenhum | `FF_CHAVE_MALFORMADA` | 422 |
| Escopo `prestador` sem `escopo_valor` | input violou INV-FF-04 | não | rejeita pré-INSERT | inalterado | nenhum | nenhum | `FF_ESCOPO_INVALIDO` | 422 |
| Escopo `prestador` com `escopo_valor` apontando para prestador inexistente | INV-FF-09 viola | input formal válido | rejeita após verificação aplicacional | inalterado | nenhum | nenhum | `FF_ESCOPO_VALOR_INEXISTENTE` | 422 |
| Escopo `global` com permissão `criar` mas usuário não é meta-admin | usuário tem `criar` mas papel ≠ meta-admin | input válido | rejeita | inalterado | `OperacaoRejeitada` em audit_log | nenhum INSERT | `FF_GLOBAL_REQUER_METAADMIN` | 403 |
| Rollout fora de [0,100] | rollout = -5 ou 150 | não | rejeita pré-INSERT | inalterado | nenhum | nenhum | `FF_ROLLOUT_INVALIDO` | 422 |
| Descrição < 10 chars | descrição muito curta | não | rejeita | inalterado | nenhum | nenhum | `FF_DESCRICAO_INSUFICIENTE` | 422 |

#### Op `OP-FF-PUBLICAR` — transição rascunho → ativa

| Cenário | Pré-condição | Comportamento | Estado resultante | Eventos | Persistência | Erro | HTTP |
|---|---|---|---|---|---|---|---|
| Caminho feliz | flag em `rascunho` ∧ usuário tem `aprovar` ∧ (escopo ≠ global ∨ usuário é meta-admin) | publica | `ativa`, `versao+=1` | `FeatureFlagPublicada` | UPDATE em `feature_flag` | — | 200 |
| Flag não está em rascunho | flag em `ativa`, `inativa` ou `arquivada` | rejeita | inalterado | `OperacaoRejeitada` | nenhum | `FF_TRANSICAO_INVALIDA` | 409 |
| Sem permissão `aprovar` | usuário não tem | rejeita | inalterado | `AcessoNegado` | nenhum | `FF_PERMISSAO_NEGADA` | 403 |
| Escopo global e não é meta-admin | escopo=`global` ∧ papel ≠ meta-admin | rejeita | inalterado | `AcessoNegado` | nenhum | `FF_GLOBAL_REQUER_METAADMIN` | 403 |
| Concorrência: outra UPDATE concorrente | versao no banco ≠ versao informada no request | rejeita | inalterado | `ConflitoVersao` | nenhum | `FF_VERSAO_OBSOLETA` | 409 |

#### Op `OP-FF-ATUALIZAR` — atualizar campos editáveis (descrição, rollout, owner, dominio_alvo)

Campos editáveis: `descricao`, `rollout_percentual`, `owner`, `dominio_alvo`.
Campos NÃO editáveis: `chave` (INV-FF-02), `escopo_tipo`, `escopo_valor`.

| Cenário | Pré-condição | Comportamento | Estado resultante | Eventos | Persistência | Erro | HTTP |
|---|---|---|---|---|---|---|---|
| Caminho feliz | flag em `ativa` ou `inativa` ∨ `rascunho` ∧ permissão `atualizar` | atualiza | mesma fase, `versao+=1`, `atualizado_em=now()` | `FeatureFlagAtualizada` | UPDATE | — | 200 |
| Flag arquivada | estado=`arquivada` | rejeita | inalterado | nenhum | nenhum | `FF_OPERACAO_EM_ARQUIVADA_PROIBIDA` | 409 |
| Tenta editar `chave` | input contém `chave` diferente | rejeita | inalterado | nenhum | nenhum | `FF_CAMPO_IMUTAVEL` | 422 |
| Tenta editar `escopo_tipo` ou `escopo_valor` | input contém | rejeita | inalterado | nenhum | nenhum | `FF_CAMPO_IMUTAVEL` | 422 |
| Versão obsoleta | versao concorrente | rejeita | inalterado | `ConflitoVersao` | nenhum | `FF_VERSAO_OBSOLETA` | 409 |
| Mudança de rollout para 0 | rollout=0 (atalho de "desligar gradualmente") | aceita; comportamento idêntico a desativar mas mantém estado `ativa` | `versao+=1` | `FeatureFlagAtualizada` + `FeatureFlagRolloutZerado` | UPDATE | — | 200 |

> **Decisão arquitetural**: rollout=0 ≠ estado `inativa`. Rollout=0 é "ativa porém sem entregar para ninguém"; `inativa` é desligada explicitamente. Diferença observável em auditoria e em painéis. Justificativa em ADR-FF-01 (apêndice).

#### Op `OP-FF-DESATIVAR` — transição ativa → inativa

| Cenário | Pré-condição | Comportamento | Estado resultante | Eventos | Persistência | Erro | HTTP |
|---|---|---|---|---|---|---|---|
| Caminho feliz | flag em `ativa` ∧ permissão `atualizar` | desativa | `inativa`, `versao+=1` | `FeatureFlagDesativada` | UPDATE | — | 200 |
| Não está ativa | estado ≠ `ativa` | rejeita | inalterado | nenhum | nenhum | `FF_TRANSICAO_INVALIDA` | 409 |
| Sem permissão | sem `atualizar` | rejeita | inalterado | nenhum | nenhum | `FF_PERMISSAO_NEGADA` | 403 |

#### Op `OP-FF-REATIVAR` — transição inativa → ativa

| Cenário | Pré-condição | Comportamento | Estado resultante | Eventos | Persistência | Erro | HTTP |
|---|---|---|---|---|---|---|---|
| Caminho feliz | flag em `inativa` ∧ permissão `atualizar` | reativa | `ativa`, `versao+=1` | `FeatureFlagReativada` | UPDATE | — | 200 |
| Não está inativa | estado ≠ `inativa` | rejeita | inalterado | nenhum | nenhum | `FF_TRANSICAO_INVALIDA` | 409 |

#### Op `OP-FF-ARQUIVAR` — transição → arquivada (terminal)

| Cenário | Pré-condição | Comportamento | Estado resultante | Eventos | Persistência | Erro | HTTP |
|---|---|---|---|---|---|---|---|
| Caminho feliz (de rascunho/inativa) | flag em `rascunho`∨`inativa` ∧ permissão `excluir` | arquiva | `arquivada`, `versao+=1` | `FeatureFlagArquivada` | UPDATE | — | 200 |
| Caminho feliz (de ativa) | flag em `ativa` ∧ não consumida em runtime há ≥ 30 dias ∧ permissão `excluir` ∧ confirmação textual da chave | arquiva com warning | `arquivada`, `versao+=1` | `FeatureFlagArquivada` | UPDATE | — | 200 |
| De ativa, consumida recentemente | flag `ativa` consumida em runtime nos últimos 30 dias | rejeita | inalterado | nenhum | nenhum | `FF_ARQUIVAR_ATIVA_USADA` | 409 |
| Já arquivada | estado=`arquivada` | rejeita | inalterado | nenhum | nenhum | `FF_TRANSICAO_INVALIDA` | 409 |
| Sem confirmação da chave (de ativa) | input não inclui campo `confirmacao_chave` ou está incorreto | rejeita | inalterado | nenhum | nenhum | `FF_CONFIRMACAO_AUSENTE` | 422 |

#### Op `OP-FF-AVALIAR` — avaliação em runtime (chamada por outras capacidades)

Chamada de leitura crítica. Não modifica estado. Tem caching agressivo.

**Input**: `chave`, `contexto` ({prestador_id?, planta_id?, dominio?, sticky_key?}).

**Output**: `{ativa: bool, versao_avaliada: int, motivo: string}`.

| Cenário | Comportamento | Output | Cache | Erro | HTTP |
|---|---|---|---|---|---|
| Flag não existe | retorna inativa com motivo "flag inexistente" | `{ativa: false, motivo: "FLAG_INEXISTENTE"}` | sim, 5min | — | 200 |
| Flag em `arquivada` | retorna inativa com motivo `arquivada` | `{ativa: false, motivo: "FLAG_ARQUIVADA"}` | sim, 5min | — | 200 |
| Flag em `inativa` | retorna inativa | `{ativa: false, motivo: "FLAG_INATIVA"}` | sim, 30s | — | 200 |
| Flag `ativa` global, rollout=100 | retorna ativa | `{ativa: true, motivo: "GLOBAL_FULL"}` | sim, 30s | — | 200 |
| Flag `ativa` global, rollout < 100, sem `sticky_key` | aleatorização sem aderência: rola dado uniforme [0,100); ativa se < rollout | `{ativa: bool, motivo: "ROLLOUT_RANDOM"}` | não (estocástico) | — | 200 |
| Flag `ativa` global, rollout < 100, com `sticky_key` | hash determinístico de `sticky_key` mod 100 ; ativa se < rollout | `{ativa: bool, motivo: "ROLLOUT_STICKY"}` | sim, 30s | — | 200 |
| Flag escopo `prestador`, contexto sem `prestador_id` | retorna inativa | `{ativa: false, motivo: "CONTEXTO_INSUFICIENTE"}` | não | — | 200 |
| Flag escopo `prestador`, contexto.prestador_id ≠ flag.escopo_valor | retorna inativa | `{ativa: false, motivo: "FORA_DO_ESCOPO"}` | sim, 30s | — | 200 |
| Flag escopo `prestador`, contexto.prestador_id = flag.escopo_valor, rollout=100 | retorna ativa | `{ativa: true, motivo: "PRESTADOR_FULL"}` | sim, 30s | — | 200 |
| Flag escopo `dominio` | mesma lógica, comparando `contexto.dominio` com `flag.dominio_alvo` | conforme | sim, 30s | — | 200 |

> **Sticky key**: para mesma `sticky_key` (ex: prestador_id), a avaliação retorna sempre o mesmo resultado, garantindo experiência consistente. Sem `sticky_key`, avaliação pode oscilar (uso aceitável apenas em flags de chaos engineering ou testes A/B genuinamente aleatórios).

#### Op `OP-FF-LISTAR` — listar feature flags com filtros

Operação de leitura apenas, paginada. Sem efeito colateral. Acesso governado por permissão `ler` em `feature_flag`. Filtros: `estado`, `escopo_tipo`, `dominio_alvo`, `owner`, `chave_contém`. Resposta paginada com `total`, `pagina_atual`, `total_paginas`, `items[]`.

#### Op `OP-FF-EXPORTAR-CONFIGURACAO` — exportar todas as flags em formato YAML/JSON

Operação que gera snapshot completo da configuração de feature flags do prestador (ou global se meta-admin). Output: arquivo YAML/JSON assinado. Hash SHA-256 do conteúdo registrado em auditoria.

| Cenário | Pré-condição | Output | Persistência | HTTP |
|---|---|---|---|---|
| Caminho feliz | permissão `exportar` em `feature_flag` | URL S3 pré-assinada (TTL 1h) | registro em audit_log | 200 |
| Sem permissão | sem `exportar` | erro | nenhum | 403 |

### 3.1.7 Regras de negócio locais

Regras que não são invariantes do aggregate (que valem sempre) mas dependem de contexto.

| ID | Regra | Severidade | Quando avaliada | Comportamento ao violar |
|---|---|---|---|---|
| `R-FF-01` | Chave deve ter ≥ 3 caracteres e ≤ 80 caracteres, snake_case minúsculo | Bloqueante | Síncrono ao digitar (validação inline) e ao salvar | Erro `FF_CHAVE_MALFORMADA` |
| `R-FF-02` | Descrição deve ter ≥ 10 caracteres | Bloqueante | Síncrono ao salvar | Erro `FF_DESCRICAO_INSUFICIENTE` |
| `R-FF-03` | Flags com `escopo=global` só podem ser criadas/publicadas/arquivadas por meta-administrador | Bloqueante | Síncrono | Erro `FF_GLOBAL_REQUER_METAADMIN` |
| `R-FF-04` | Confirmação textual da chave é obrigatória ao arquivar flag em estado `ativa` | Bloqueante | Síncrono ao confirmar | Erro `FF_CONFIRMACAO_AUSENTE` |
| `R-FF-05` | Flag em estado `ativa` consumida em runtime nos últimos 30 dias não pode ser arquivada (proteção contra arquivamento descuidado de flag em uso) | Bloqueante | Síncrono ao arquivar | Erro `FF_ARQUIVAR_ATIVA_USADA` |
| `R-FF-06` | Mudança de `rollout_percentual` em flag `ativa` deve ser monotonicamente crescente para flags com tag `regulatorio` (ex: `sinisa_export_404`) — não permite reduzir rollout uma vez aumentado, exceto via desativação e reativação com justificativa | Obrigatória | Síncrono ao salvar | Aviso modal + bloqueio se confirmação ausente |
| `R-FF-07` | Toda alteração em flag `ativa` é auditada com diff completo dos campos alterados | Bloqueante | Backend (sempre) | Falha de auditoria bloqueia escrita (commit ACID) |
| `R-FF-08` | Após arquivada, flag não pode ser recriada com mesma chave por 90 dias (proteção contra confusão de identidade) | Bloqueante | Síncrono ao criar | Erro `FF_CHAVE_QUARENTENA` |

### 3.1.8 Elementos de UI

Mapeamento direto wireframe → elementos. Cada elemento da imagem do wireframe `01_PlataformaProduto.png` (apresentado separadamente) tem ID, comportamento e função invocada.

#### Métricas de header (parte superior da página, mas a métrica `Feature flags ativas` é desta capacidade)

| ID | Tipo | Posição | Conteúdo no wireframe | Função invocada | Atualização |
|---|---|---|---|---|---|
| `UI-FF-01` | métrica numérica | header esquerdo | "FEATURE FLAGS ATIVAS · 14 · de 23 cadastradas" | `OP-FF-LISTAR` com filtro `estado=ativa` + count total | refresh on demand + auto a cada 5min |

#### Tabela principal "Feature flags"

| ID | Tipo | Posição | Função invocada | Comportamento |
|---|---|---|---|---|
| `UI-FF-02` | título de seção | acima da tabela | — | label "Feature flags" |
| `UI-FF-03` | contador descritivo | direita do título | `OP-FF-LISTAR` agregado | exibe "14 ativas · 9 desabilitadas" |
| `UI-FF-04` | tabela | corpo | `OP-FF-LISTAR` | colunas: chave, escopo, estado, rollout, aplicada a, owner, atualizada |
| `UI-FF-04-COL-CHAVE` | coluna textual | tabela | — | exibe `feature_flag.chave` em fonte mono |
| `UI-FF-04-COL-ESCOPO` | coluna chip | tabela | — | exibe pill com `escopo_tipo` (`global`, `prestador`, `planta`, `fase 2`/`fase 3` quando aplicável) |
| `UI-FF-04-COL-ESTADO` | coluna chip | tabela | — | pill verde "ativa" ou cinza "inativa"; arquivadas não aparecem aqui (filtro padrão) |
| `UI-FF-04-COL-ROLLOUT` | coluna numérica | tabela | — | exibe "X%" |
| `UI-FF-04-COL-APLICADA-A` | coluna textual derivada | tabela | cálculo aplicacional | exibe "Todas", "N plantas", "1 planta", "Sabesp +N" conforme escopo e rollout |
| `UI-FF-04-COL-OWNER` | coluna textual | tabela | — | exibe `feature_flag.owner` |
| `UI-FF-04-COL-ATUALIZADA` | coluna data | tabela | — | exibe `atualizado_em` formatada "DD MMM AAAA" |
| `UI-FF-04-LINHA` | linha clicável | tabela | abre modal de detalhes (`UI-FF-DETALHES`) | clique no row → drill-down |

#### Botões de ação

| ID | Tipo | Posição | Conteúdo | Função invocada | Permissão |
|---|---|---|---|---|---|
| `UI-FF-05` | botão primário | header direito | "Nova feature flag" | abre `UI-FF-MODAL-CRIAR` | `criar` em `feature_flag` |
| `UI-FF-06` | botão secundário | header direito | "Exportar configuração" | invoca `OP-FF-EXPORTAR-CONFIGURACAO` | `exportar` em `feature_flag` |

#### Modal de criação

| ID | Tipo | Conteúdo | Validação | Comportamento |
|---|---|---|---|---|
| `UI-FF-MODAL-CRIAR` | modal | título "Nova feature flag" + form | — | aberto por `UI-FF-05` |
| `UI-FF-MODAL-CRIAR-FLD-CHAVE` | input texto | label "Chave" | regex `^[a-z][a-z0-9_]*$`, 3-80 chars; validação inline com debounce 500ms; verifica unicidade via API | exibe sugestão se chave já existe ou está em quarentena |
| `UI-FF-MODAL-CRIAR-FLD-DESCRICAO` | textarea | label "Descrição" | min 10 chars | contador de caracteres |
| `UI-FF-MODAL-CRIAR-FLD-ESCOPO-TIPO` | select | label "Escopo" | opções: global, prestador, planta, domínio | mudança revela campos contextuais |
| `UI-FF-MODAL-CRIAR-FLD-ESCOPO-VALOR` | autocomplete | label "Aplicada a" | obrigatório se escopo ∈ {prestador, planta} | autocomplete com lista de prestadores ou plantas |
| `UI-FF-MODAL-CRIAR-FLD-DOMINIO` | select | label "Domínio" | opções: aguas, efluentes, ambos | obrigatório se escopo=domínio |
| `UI-FF-MODAL-CRIAR-FLD-ROLLOUT` | slider + input | label "Rollout %" | int 0-100 | slider e input sincronizados |
| `UI-FF-MODAL-CRIAR-FLD-OWNER` | autocomplete | label "Owner" | min 3 chars | autocomplete de owners conhecidos + permite novo |
| `UI-FF-MODAL-CRIAR-BTN-SALVAR` | botão primário | "Criar como rascunho" | desabilitado até form válido | invoca `OP-FF-CRIAR` |
| `UI-FF-MODAL-CRIAR-BTN-CANCELAR` | botão ghost | "Cancelar" | sempre habilitado | fecha modal sem ação |

#### Modal de detalhes / edição

| ID | Tipo | Conteúdo | Comportamento |
|---|---|---|---|
| `UI-FF-MODAL-DETALHES` | modal | metadados completos da flag + histórico de auditoria | aberto ao clicar em linha da tabela |
| `UI-FF-MODAL-DETALHES-BTN-PUBLICAR` | botão primário | "Publicar" (visível só se estado=rascunho) | invoca `OP-FF-PUBLICAR`; exige meta-admin se escopo=global |
| `UI-FF-MODAL-DETALHES-BTN-DESATIVAR` | botão | "Desativar" (visível só se estado=ativa) | invoca `OP-FF-DESATIVAR` |
| `UI-FF-MODAL-DETALHES-BTN-REATIVAR` | botão | "Reativar" (visível só se estado=inativa) | invoca `OP-FF-REATIVAR` |
| `UI-FF-MODAL-DETALHES-BTN-EDITAR` | botão | "Editar" | abre form de edição com campos editáveis (descricao, rollout, owner) |
| `UI-FF-MODAL-DETALHES-BTN-ARQUIVAR` | botão destrutivo | "Arquivar" | abre modal de confirmação |
| `UI-FF-MODAL-CONFIRMAR-ARQUIVAR` | modal modal | exige digitar chave + justificativa se ativa | invoca `OP-FF-ARQUIVAR` após validação |

### 3.1.9 Contrato de API

Endpoints REST. Schemas detalhados ficam em `apêndice 7.1` do documento consolidado da sub-página; aqui cada endpoint é descrito com função, autenticação e códigos.

| Método | Path | Operação | Auth | Códigos sucesso | Códigos erro |
|---|---|---|---|---|---|
| `POST` | `/v1/aguas/admin/01-01/feature-flags` | `OP-FF-CRIAR` | OAuth2 + JWT + permissão `criar` | 201 | 400, 403, 409, 422 |
| `GET` | `/v1/aguas/admin/01-01/feature-flags` | `OP-FF-LISTAR` | + permissão `ler` | 200 | 400, 403 |
| `GET` | `/v1/aguas/admin/01-01/feature-flags/{id}` | leitura unitária | + permissão `ler` | 200 | 403, 404 |
| `PATCH` | `/v1/aguas/admin/01-01/feature-flags/{id}` | `OP-FF-ATUALIZAR` | + permissão `atualizar` | 200 | 400, 403, 409, 422 |
| `POST` | `/v1/aguas/admin/01-01/feature-flags/{id}/publicar` | `OP-FF-PUBLICAR` | + permissão `aprovar` | 200 | 403, 409 |
| `POST` | `/v1/aguas/admin/01-01/feature-flags/{id}/desativar` | `OP-FF-DESATIVAR` | + permissão `atualizar` | 200 | 403, 409 |
| `POST` | `/v1/aguas/admin/01-01/feature-flags/{id}/reativar` | `OP-FF-REATIVAR` | + permissão `atualizar` | 200 | 403, 409 |
| `POST` | `/v1/aguas/admin/01-01/feature-flags/{id}/arquivar` | `OP-FF-ARQUIVAR` | + permissão `excluir` | 200 | 403, 409, 422 |
| `POST` | `/v1/aguas/admin/01-01/feature-flags/{chave}/avaliar` | `OP-FF-AVALIAR` | OAuth2 + JWT (escopo `runtime`) | 200 | 400 |
| `POST` | `/v1/aguas/admin/01-01/feature-flags/exportar` | `OP-FF-EXPORTAR-CONFIGURACAO` | + permissão `exportar` | 200 | 403 |

#### Catálogo de erros

Todo erro retorna corpo no formato:

```json
{
  "error": {
    "code": "FF_CHAVE_DUPLICADA",
    "message": "Mensagem em pt-BR para apresentação ao usuário.",
    "details": { "chave": "gemeo_digital_v2", "estado_existente": "ativa" },
    "trace_id": "uuid",
    "retryable": false,
    "documentation_url": "https://docs.titan/erros/FF_CHAVE_DUPLICADA"
  }
}
```

| Code | HTTP | Retryable | Significado | Mensagem ao usuário |
|---|---|---|---|---|
| `FF_PERMISSAO_NEGADA` | 403 | não | papel sem permissão necessária | "Você não tem permissão para esta operação." |
| `FF_GLOBAL_REQUER_METAADMIN` | 403 | não | escopo=global e papel ≠ meta-admin | "Flags globais só podem ser geridas pelo meta-administrador." |
| `FF_CHAVE_DUPLICADA` | 409 | não | chave já existe | "Já existe feature flag com esta chave." |
| `FF_CHAVE_MALFORMADA` | 422 | não | chave fora do regex | "Use snake_case minúsculo, 3-80 caracteres, começando com letra." |
| `FF_CHAVE_QUARENTENA` | 409 | sim (após 90d) | chave foi arquivada há < 90d | "Chave em quarentena por 90 dias após arquivamento. Disponível em {data}." |
| `FF_DESCRICAO_INSUFICIENTE` | 422 | não | descrição < 10 chars | "Descrição precisa de pelo menos 10 caracteres." |
| `FF_ESCOPO_INVALIDO` | 422 | não | escopo não respeita INV-FF-03/04/05 | "Configuração de escopo inválida." |
| `FF_ESCOPO_VALOR_INEXISTENTE` | 422 | não | escopo_valor aponta para entidade inexistente | "Prestador/planta não encontrado." |
| `FF_ROLLOUT_INVALIDO` | 422 | não | rollout fora de [0,100] | "Rollout deve estar entre 0 e 100." |
| `FF_TRANSICAO_INVALIDA` | 409 | não | tentativa de transição não permitida | "Operação não permitida para flag em estado {estado}." |
| `FF_VERSAO_OBSOLETA` | 409 | sim | versão concorrente desatualizada | "Flag foi alterada por outro usuário. Recarregue e tente novamente." |
| `FF_CONFIRMACAO_AUSENTE` | 422 | não | falta confirmação textual ao arquivar ativa | "Digite a chave para confirmar arquivamento." |
| `FF_ARQUIVAR_ATIVA_USADA` | 409 | sim (após 30d) | flag ativa consumida em runtime <30d | "Flag em uso recente. Desative e aguarde 30 dias antes de arquivar." |
| `FF_CAMPO_IMUTAVEL` | 422 | não | tentou editar campo imutável | "Este campo não pode ser alterado após criação." |

### 3.1.10 Persistência

#### Tabela `feature_flag`

```sql
CREATE TABLE feature_flag (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chave VARCHAR(80) NOT NULL,
    descricao VARCHAR(500) NOT NULL,
    escopo_tipo VARCHAR(20) NOT NULL,
    escopo_valor UUID NULL,
    dominio_alvo VARCHAR(20) NOT NULL,
    rollout_percentual SMALLINT NOT NULL,
    estado VARCHAR(20) NOT NULL DEFAULT 'rascunho',
    owner VARCHAR(100) NOT NULL,
    criado_em TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    criado_por_usuario_id UUID NOT NULL,
    atualizado_em TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    atualizado_por_usuario_id UUID NOT NULL,
    versao INT NOT NULL DEFAULT 1,
    ultimo_consumo_runtime TIMESTAMPTZ NULL,

    CONSTRAINT chk_chave_formato
        CHECK (chave ~ '^[a-z][a-z0-9_]*$'),
    CONSTRAINT chk_chave_tamanho
        CHECK (char_length(chave) BETWEEN 3 AND 80),
    CONSTRAINT chk_descricao_min
        CHECK (char_length(descricao) >= 10),
    CONSTRAINT chk_estado_valido
        CHECK (estado IN ('rascunho', 'ativa', 'inativa', 'arquivada')),
    CONSTRAINT chk_escopo_tipo_valido
        CHECK (escopo_tipo IN ('global', 'prestador', 'planta', 'dominio')),
    CONSTRAINT chk_dominio_alvo_valido
        CHECK (dominio_alvo IN ('aguas', 'efluentes', 'ambos')),
    CONSTRAINT chk_rollout_range
        CHECK (rollout_percentual BETWEEN 0 AND 100),
    CONSTRAINT chk_escopo_global_sem_valor
        CHECK ((escopo_tipo = 'global' AND escopo_valor IS NULL)
               OR escopo_tipo != 'global'),
    CONSTRAINT chk_escopo_entidade_com_valor
        CHECK ((escopo_tipo IN ('prestador', 'planta') AND escopo_valor IS NOT NULL)
               OR escopo_tipo NOT IN ('prestador', 'planta')),
    CONSTRAINT chk_escopo_dominio
        CHECK ((escopo_tipo = 'dominio' AND escopo_valor IS NULL AND dominio_alvo IN ('aguas', 'efluentes'))
               OR escopo_tipo != 'dominio')
);

CREATE UNIQUE INDEX uq_feature_flag_chave_ativa
    ON feature_flag(chave) WHERE estado != 'arquivada';

CREATE INDEX idx_feature_flag_chave_arquivada
    ON feature_flag(chave, atualizado_em) WHERE estado = 'arquivada';

CREATE INDEX idx_feature_flag_estado ON feature_flag(estado)
    WHERE estado IN ('ativa', 'inativa');

CREATE INDEX idx_feature_flag_escopo
    ON feature_flag(escopo_tipo, escopo_valor) WHERE estado != 'arquivada';

CREATE INDEX idx_feature_flag_owner ON feature_flag(owner) WHERE estado != 'arquivada';
```

#### Trigger de auditoria e proteção de imutabilidade

```sql
CREATE OR REPLACE FUNCTION feature_flag_proteger_chave_imutavel()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.chave IS DISTINCT FROM OLD.chave THEN
        RAISE EXCEPTION 'FF_CAMPO_IMUTAVEL: chave da feature flag não pode ser alterada';
    END IF;
    IF NEW.escopo_tipo IS DISTINCT FROM OLD.escopo_tipo THEN
        RAISE EXCEPTION 'FF_CAMPO_IMUTAVEL: escopo_tipo não pode ser alterado';
    END IF;
    IF NEW.escopo_valor IS DISTINCT FROM OLD.escopo_valor THEN
        RAISE EXCEPTION 'FF_CAMPO_IMUTAVEL: escopo_valor não pode ser alterado';
    END IF;
    NEW.versao := OLD.versao + 1;
    NEW.atualizado_em := NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_feature_flag_proteger
    BEFORE UPDATE ON feature_flag
    FOR EACH ROW
    EXECUTE FUNCTION feature_flag_proteger_chave_imutavel();
```

#### Estratégia de retenção

- Flags em estado `rascunho`, `ativa`, `inativa`: retenção indefinida.
- Flags em estado `arquivada`: retenção de 7 anos a partir de `atualizado_em` para conformidade com auditoria regulatória (Lei 14.026 art. 23 §2º). Após 7 anos, tabela auxiliar `feature_flag_historico_arquivado` recebe o registro e a linha original é purgada.
- Eventos de domínio relacionados (em `auditoria_log`): retenção de 7 anos conforme política da capacidade de Auditoria.

#### Particionamento

Não é necessário particionar `feature_flag` — volume estimado total ≤ 5.000 registros mesmo com 10 anos de operação. Acesso a hot path é via cache (Redis) e índice de chave; tabela é pequena e fica integralmente em buffer pool.

### 3.1.11 Eventos de domínio publicados

Cada evento é publicado no Outbox Pattern: gravação atômica com mutação do aggregate, leitura assíncrona por publisher que entrega a SQS/EventBridge.

| Evento | Quando publicado | Schema | Garantia | Consumidores conhecidos |
|---|---|---|---|---|
| `FeatureFlagCriada` | após `OP-FF-CRIAR` bem-sucedida | `{flag_id, chave, escopo_tipo, escopo_valor, criado_por_usuario_id, criado_em}` | at-least-once, ordenado por chave | `01-05 Auditoria` (registro auditável); cache de avaliação (preload) |
| `FeatureFlagPublicada` | após `OP-FF-PUBLICAR` | `{flag_id, chave, versao, publicado_por_usuario_id, publicado_em}` | at-least-once | `01-05 Auditoria`; runtime evaluators (invalidate cache) |
| `FeatureFlagAtualizada` | após `OP-FF-ATUALIZAR` | `{flag_id, chave, versao, campos_alterados[], valor_anterior, valor_posterior, atualizado_por_usuario_id}` | at-least-once | `01-05 Auditoria`; runtime evaluators |
| `FeatureFlagRolloutZerado` | após `OP-FF-ATUALIZAR` que muda rollout para 0 (caso especial) | `{flag_id, chave, versao, rollout_anterior, atualizado_por_usuario_id}` | at-least-once | painel de operação; alertas |
| `FeatureFlagDesativada` | após `OP-FF-DESATIVAR` | `{flag_id, chave, versao, desativado_por_usuario_id}` | at-least-once | `01-05 Auditoria`; runtime evaluators |
| `FeatureFlagReativada` | após `OP-FF-REATIVAR` | idem | at-least-once | idem |
| `FeatureFlagArquivada` | após `OP-FF-ARQUIVAR` | `{flag_id, chave, versao, motivo_textual, arquivado_por_usuario_id}` | at-least-once | `01-05 Auditoria`; runtime evaluators (purge cache) |
| `FeatureFlagAvaliada` | após `OP-FF-AVALIAR` (só em sampling 1%) | `{chave, contexto_anonimizado, resultado, motivo, latencia_ms}` | best-effort, amostral | observabilidade (#06); analytics |

> **Esquema dos eventos é versionado**. Mudança não-retrocompatível exige novo nome de evento (`FeatureFlagAtualizadaV2`). Consumidores podem migrar gradualmente.

### 3.1.12 Concorrência e idempotência

| Operação | Idempotente? | Concorrência otimista? | Lock? | Notas |
|---|---|---|---|---|
| `OP-FF-CRIAR` | sim, via header `Idempotency-Key` (TTL 24h) | n/a | n/a | request com mesma key retorna mesmo resultado |
| `OP-FF-ATUALIZAR` | sim, via key + concorrência otimista (`versao` no payload) | sim | n/a | conflito retorna `FF_VERSAO_OBSOLETA` |
| `OP-FF-PUBLICAR` | sim (publicar 2x não é erro, retorna mesmo resultado) | sim | n/a | tentativa em flag não-rascunho retorna `FF_TRANSICAO_INVALIDA` |
| `OP-FF-DESATIVAR` | sim | sim | n/a | tentativa em flag não-ativa retorna `FF_TRANSICAO_INVALIDA` |
| `OP-FF-REATIVAR` | sim | sim | n/a | idem |
| `OP-FF-ARQUIVAR` | sim | sim | n/a | tentativa em flag arquivada retorna `FF_TRANSICAO_INVALIDA` |
| `OP-FF-AVALIAR` | sim trivialmente (read-only) | n/a | n/a | resultado pode mudar entre chamadas se rollout não-sticky |
| `OP-FF-EXPORTAR-CONFIGURACAO` | sim | n/a | n/a | gera novo arquivo a cada chamada (URL diferente) |

**Sobre `Idempotency-Key`**: clientes podem (e devem, em automação) enviar header `Idempotency-Key: <UUID>` em POSTs. Servidor armazena tupla (chave, request_hash, response) por 24h em cache (Redis); requests subsequentes com mesma chave retornam mesmo resultado sem reexecutar.

### 3.1.13 Critérios de aceitação BDD

Todos os critérios são verificáveis manualmente ou em pipeline de testes E2E (pytest + httpx + testcontainers PostgreSQL).

```gherkin
Funcionalidade: Criação de feature flag

Cenário: criação bem-sucedida de flag global por meta-administrador
  Dado um usuário autenticado com papel "meta-administrador TITAN"
    E nenhuma feature flag com chave "novo_modelo_ia" existe
  Quando ele POST /feature-flags com {chave: "novo_modelo_ia", descricao: "Modelo IA Bayesiano de coagulação", escopo_tipo: "global", dominio_alvo: "aguas", rollout_percentual: 0, owner: "Núcleo Águas"}
  Então a resposta é 201 com {id: <UUID>, chave: "novo_modelo_ia", estado: "rascunho", versao: 1}
    E uma linha é inserida em feature_flag com estado "rascunho"
    E um evento "FeatureFlagCriada" é publicado em outbox
    E uma entrada é criada em auditoria_log com acao="criar" recurso="feature_flag"

Cenário: criação rejeitada por chave duplicada
  Dado uma feature flag com chave "gemeo_digital_v2" em estado "ativa"
    E um usuário autenticado com papel "meta-administrador TITAN"
  Quando ele POST /feature-flags com {chave: "gemeo_digital_v2", descricao: "Tentativa duplicada", escopo_tipo: "global", dominio_alvo: "aguas", rollout_percentual: 0, owner: "Núcleo"}
  Então a resposta é 409 com error.code = "FF_CHAVE_DUPLICADA"
    E nenhuma linha nova é criada em feature_flag
    E uma entrada é criada em auditoria_log com acao="rejeitada"

Cenário: criação de flag global por papel não-meta-admin é rejeitada
  Dado um usuário autenticado com papel "Analista TITAN" (sem privilégio de meta-admin)
  Quando ele POST /feature-flags com {chave: "outro_recurso", escopo_tipo: "global", ...}
  Então a resposta é 403 com error.code = "FF_GLOBAL_REQUER_METAADMIN"

Cenário: arquivamento de flag ativa exige confirmação textual
  Dado uma flag "ia_coagulacao_bayes" em estado "ativa", último consumo há 45 dias
    E um usuário com permissão "excluir" em "feature_flag"
  Quando ele POST /feature-flags/{id}/arquivar com {versao: 3}, sem campo confirmacao_chave
  Então a resposta é 422 com error.code = "FF_CONFIRMACAO_AUSENTE"
  Quando ele repete a chamada com confirmacao_chave="ia_coagulacao_bayes" e justificativa válida
  Então a resposta é 200 e a flag muda para "arquivada"

Cenário: avaliação retorna resultado consistente para mesma sticky_key
  Dado uma flag "ia_coagulacao_bayes" em estado "ativa" com escopo "global" e rollout 60
  Quando avalio 1000 vezes com sticky_key="prestador-sabesp-v1"
  Então o resultado é o mesmo nas 1000 chamadas
    E o motivo é "ROLLOUT_STICKY"

Cenário: tentativa de editar chave é rejeitada
  Dado uma flag "gemeo_digital_v2" em estado "ativa"
  Quando faço PATCH /feature-flags/{id} com {chave: "gemeo_digital_v3", versao: 5}
  Então a resposta é 422 com error.code = "FF_CAMPO_IMUTAVEL"
    E nenhuma alteração é persistida
```

### 3.1.14 Permissões

Matriz papel × ação para esta capacidade.

| Papel | ler | criar | atualizar | aprovar (publicar) | excluir (arquivar) | exportar |
|---|---|---|---|---|---|---|
| Meta-administrador TITAN | ✓ | ✓ (qualquer escopo) | ✓ | ✓ (qualquer escopo) | ✓ | ✓ |
| Analista TITAN | ✓ | ✓ (escopo ≠ global) | ✓ | ✓ (escopo ≠ global) | ✓ (escopo ≠ global) | ✓ |
| Gestor de analistas TITAN | ✓ | ✓ (escopo ≠ global) | ✓ | ✓ (escopo ≠ global) | ✓ | ✓ |
| Qualquer papel Cliente | apenas leitura de flags publicadas e que afetam seu escopo (acesso indireto via runtime evaluators) | — | — | — | — | — |

Permissão `configurar` em `feature_flag` não é usada — capacidade não tem aspecto configurável global; cada flag tem ciclo de vida próprio.

### 3.1.15 Consumidores

Capacidades e itens que consomem `Feature Flags`:

| Consumidor | Como consome | Frequência |
|---|---|---|
| **Runtime evaluators** (todas as capacidades de todos os itens que têm comportamento condicional) | `OP-FF-AVALIAR` síncrono com cache 30s | dezenas de milhares por segundo agregado |
| **Capacidade `Fases evolutivas` desta sub-página** | leitura direta para identificar quais fases estão habilitadas | a cada navegação no painel de fases |
| **`01-05 Auditoria & retenção`** | consumo de eventos `FeatureFlag*` | streaming via Outbox |
| **`01-06 Observabilidade & SLA`** | consumo de evento amostrado `FeatureFlagAvaliada` | streaming, sample 1% |
| **`02 Gestão de domínio`** | leitura de flags relacionadas a habilitação de catálogos de domínio | sob demanda |

### 3.1.16 Fora de escopo

Esta capacidade **não** cobre:

| Fora de escopo | Por quê | Onde mora |
|---|---|---|
| Targeting fino por atributo de usuário (ex: "ativa para usuários com role X em estado Y") | TITAN não tem caso de uso para granularidade tão fina; rollout percentual + escopo bastam | n/a (não previsto) |
| Experimentação A/B com métricas de conversão | TITAN não é produto de growth; não há funil de conversão | n/a |
| Versionamento de valores não-binários (ex: flag que assume múltiplos valores) | mantemos contrato simples ativo/inativo + rollout | n/a |
| Configuração de fases evolutivas (Onda 2, Fase 2, Fase 3) | é capacidade própria nesta mesma sub-página com semântica distinta | `Capacidade Fases Evolutivas` (3.5) |
| Configuração de jobs agendados | é capacidade própria | `Capacidade Jobs Agendados` (3.2) |
| Política de retenção | é capacidade própria | `Capacidade Política de Retenção` (3.6) |

### 3.1.17 Dependências

Esta capacidade depende das seguintes para operar:

| Dependência | Tipo | Sub-página/item | Como é usada | Impacto se indisponível |
|---|---|---|---|---|
| `Usuario` aggregate | dado | `01-02` | referenciado em `criado_por_usuario_id` e `atualizado_por_usuario_id` | criação falha (FK violation) |
| `Permissao` engine | serviço síncrono | `01-02` | autorização de operações | todas as operações falham com 403 |
| `Auditoria` engine | serviço assíncrono | `01-05` | toda mutação publica evento auditável | mutações ainda funcionam mas auditoria fica degradada (alerta operacional) |
| `Prestador` aggregate | dado | `09` | referenciado em `escopo_valor` quando `escopo_tipo='prestador'` | criação de flag de prestador falha se prestador inexistente |
| `Planta` aggregate | dado | `12` | referenciado em `escopo_valor` quando `escopo_tipo='planta'` | criação de flag de planta falha se planta inexistente |
| AWS Secrets Manager | infra | externa | n/a (esta capacidade não usa segredos diretamente) | n/a |
| AWS RDS PostgreSQL 15 | infra | externa | persistência | indisponibilidade total da capacidade |
| Cache Redis | infra | externa | cache de avaliação (30s TTL) | degradação de performance, não erro funcional |

---

## Notas para validação desta iteração 1

Antes de prosseguir com as outras 5 capacidades (`Jobs Agendados`, `Vault de Segredos`, `Operações em Lote`, `Fases Evolutivas`, `Política de Retenção`) na iteração 2, preciso da sua avaliação dos seguintes pontos:

### Pontos onde quero confirmação

**(Q1)** A estrutura de **17 sub-seções por capacidade** está bem dimensionada? Algo é redundante? Falta algo?

**(Q2)** O nível de detalhe na **tabela de decisão (3.X.6)** é suficiente para 3 desenvolvedores produzirem código equivalente? Ou precisa mais granularidade ainda?

**(Q3)** O **catálogo de erros (3.X.9)** com código + HTTP + retryable + mensagem padronizada — esse formato de erro deve ser **canônico em toda a plataforma**? Se sim, eu o promovo para a verdade canônica.

**(Q4)** Os **eventos de domínio (3.X.11)** com schema + garantias + consumidores — formato OK?

**(Q5)** As **invariantes do aggregate (3.X.4)** marcadas como `INV-FF-NN` — você quer esse esquema de IDs (`INV-{recurso}-{NN}`) virando padrão?

**(Q6)** A separação **invariante × regra de negócio** (3.X.4 vs 3.X.7) — está clara? Invariante é o que vale sempre (constraint do banco); regra é contextual (validação aplicacional).

**(Q7)** Os critérios BDD (3.X.13) — quantidade adequada (6 cenários cobrindo principais combinações), ou quer mais?

### Decisões arquiteturais embutidas que merecem nota explícita

Aproveitando a iteração, registrei **decisões implícitas** que tomei e que me parecem certas mas merecem confirmação:

**(D1)** **Rollout 0% ≠ inativa** — flag pode estar `ativa` com rollout 0%, ou pode estar `inativa`. São situações distintas. Justificativa: auditoria precisa diferenciar "desligamos pelo botão" de "deixamos no zero como kill switch gradual". **OK?**

**(D2)** **Quarentena de 90 dias para chaves arquivadas** — após arquivar, mesma chave não pode ser recriada por 90 dias. Justificativa: evita confusão se alguém recriar com semântica diferente e cache antigo retornar resultado errado. **OK?**

**(D3)** **Sticky key obrigatória para rollout < 100%** em flags com tag regulatória — para flags que afetam compliance (ex: `sinisa_export_404`), não permitimos aleatorização sem aderência. **OK?**

**(D4)** **Idempotency-Key como header padrão** em todas as escritas — TTL 24h em Redis. Padrão da plataforma. **OK?**

**(D5)** **Rollout monotonicamente crescente** para flags regulatórias — uma vez subido, só desce via desativação. **OK?**

**(D6)** **Evento `FeatureFlagAvaliada` com sampling 1%** — registramos só amostra para não saturar pipeline. **OK?**

---

[Não vou produzir as outras 5 capacidades nem o sumário executivo até você responder as perguntas Q1–Q7 e validar D1–D6. Cada confirmação aqui economiza horas de retrabalho nos 17 itens.]
