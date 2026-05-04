# SPE001_renderizar_cabecalho_pagina

> **Natureza**: `SPE` — specification  
> **ID**: `SPE001`  
> **Camada**: frontend  
> **Submódulo**: 01 · Gestão da plataforma → Recursos & Funcionalidades  
> **Card**: 1 — Cabeçalho de página  
> **Função canônica referenciada**: —  
> **Versão**: 1.0.0  
> **Data**: 04 mai 2026  
> **Status**: definido  

---

> ### ⚠ Artefatos complementares pendentes
>
> Esta função **não é autossuficiente** até que os artefatos abaixo sejam produzidos.
> Sem eles, o desenvolvedor precisará interpretar documentos narrativos ou consultar
> outros membros do time para implementar.
>
> | ID pendente | Natureza | Nome funcional | O que resolve |
> |---|---|---|---|
> | `TAX002` | `TAX` | `navegacao_menu_admin` | Tabela estruturada (JSON/enum) de grupos → itens → sub-páginas do menu Admin. Hoje essa informação existe apenas no PDF executivo v2.3, que não é consumível por código. O breadcrumb (var 1) depende dela para montar os 3 segmentos. |
> | `CON001` | `CON` | `sessao_contexto_acesso` | Contrato de API (schema de resposta) do endpoint que retorna o prestador logado, o domínio da sessão e as permissões do usuário. O contexto do acesso (var 3) depende dele para resolver `nome_fantasia`, `razao_social` e instância Admin. Deve definir também **onde vive a lógica de fallback** (`nome_fantasia` → `razao_social`): se o backend entrega o campo já resolvido ou se o frontend aplica a regra. |
>
> **Quando produzidos**, atualizar o campo **Status** deste documento para `autossuficiente`
> e remover este quadro.

---

## Convenção de nomenclatura

```
{NNN}{000}_{nome_funcional_sintetico}

NNN  → 3 letras de natureza técnica (maiúsculas)
000  → numeração crescente com 3 dígitos
_    → separador
nome → snake_case descrevendo a função

Naturezas registradas:
  CAN  → função canônica (regra de negócio computável)
  SPE  → specification (regra de exibição, comportamento, variáveis)
  TAX  → taxonomy (vocabulário controlado, enumerações)
  CON  → contract (schema de request/response de API)
  TST  → test (critérios de aceite, cenários de teste)
  DOC  → documentation (documentação narrativa, guias)
  WFR  → wireframe (protótipo visual de referência)

Artefatos já existentes no repositório:
  TAX001  → estado_planta (taxonomia CAN_0001, migrada de CAN_0001_estado_planta.json)
  CAN001  → calcular_estado_operacional_planta
```

---

## 1. Objetivo

Definir as regras autossuficientes de cada variável exibida no Card 1 (Cabeçalho de página) do submódulo Recursos & Funcionalidades. Este documento é a fonte de verdade para implementação frontend e para validação em testes automatizados.

---

## 2. Estrutura visual

```
┌─────────────────────────────────────────────────────────────────────┐
│ CONFIGURAÇÕES GLOBAIS / GESTÃO DA PLATAFORMA / RECURSOS & FUNC... │  ← Var 1
│ Recursos & Funcionalidades                                         │  ← Var 2
│ Planta vinculada ao seu acesso · SABESP · Abastecimento de Água   │  ← Var 3
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Variáveis

### 3.1 Breadcrumb de navegação

| Atributo | Valor |
|---|---|
| **ID** | `card1_breadcrumb` |
| **Tipo de dado** | `string` (concatenação de 3 segmentos) |
| **Fonte de dados** | Tabela de navegação do menu Admin (estática por sessão) |
| **Obrigatório** | Sim |
| **Editável pelo usuário** | Não |
| **Atualização** | Muda apenas quando o usuário navega para outro submódulo |

**Regra de composição:**

O breadcrumb é montado por três segmentos separados pelo delimitador ` / ` (espaço, barra, espaço):

```
{grupo_funcional} / {item_menu} / {submodulo_ativo}
```

**Segmento 1 — Grupo funcional:**  
Derivado da tabela de 5 grupos do documento executivo TITAN v2.3:

| ID do grupo | Nome exibido |
|---|---|
| `config_globais` | CONFIGURAÇÕES GLOBAIS |
| `operacao_ativos` | OPERAÇÃO E ATIVOS |
| `estrutura_prestacao` | ESTRUTURA DE PRESTAÇÃO |
| `normativos_qualidade` | NORMATIVOS E QUALIDADE |
| `conformidade_reporte` | CONFORMIDADE E REPORTE |

A resolução é: dado o item de menu ativo, buscar o grupo ao qual pertence. Os itens 01, 02, 03 pertencem a `config_globais`. Os itens 04, 05, 06, 07, 08 pertencem a `operacao_ativos`. E assim por diante, conforme a tabela do documento executivo seção 2.

**Segmento 2 — Item do menu:**  
Nome do item #01 a #21. Para este submódulo, é sempre `GESTÃO DA PLATAFORMA` (item #01).

**Segmento 3 — Submódulo ativo:**  
Nome da sub-página dentro do item. Para este wireframe, é sempre `RECURSOS & FUNCIONALIDADES`.

**Comportamento de navegação:**
- Se o usuário navegar para outro submódulo dentro do mesmo item (ex: Identidade & acesso), apenas o segmento 3 muda.
- Se mudar de item (ex: de #01 para #02 Gestão de domínio), os segmentos 2 e 3 mudam.
- Se mudar de grupo (ex: de Configurações globais para Estrutura de prestação), todos os 3 segmentos mudam.
- Nenhum segmento é clicável no MVP (Onda 1). Na v2, os segmentos 1 e 2 se tornam links de navegação para a página-índice do grupo e do item, respectivamente.

**Tipografia:**

| Propriedade | Valor |
|---|---|
| Fonte | IBM Plex Mono |
| Tamanho | 11px |
| Peso | 500 |
| Transformação | uppercase |
| Letter-spacing | 0.05em |
| Cor | `#94a3b8` |

---

### 3.2 Título do submódulo

| Atributo | Valor |
|---|---|
| **ID** | `card1_titulo` |
| **Tipo de dado** | `string` |
| **Fonte de dados** | Tabela de sub-páginas do menu Admin (mesma que alimenta o breadcrumb) |
| **Obrigatório** | Sim |
| **Editável pelo usuário** | Não |
| **Atualização** | Muda apenas quando o usuário navega para outro submódulo |

**Regra de composição:**

Exibe o nome legível do submódulo ativo. É o mesmo valor do segmento 3 do breadcrumb, porém em formato título (capitalização normal, não uppercase). Para este wireframe: `Recursos & Funcionalidades`.

O valor é puramente derivado da navegação. Não depende de dados da planta, do prestador, do estado operacional ou de qualquer outra entidade de domínio.

**Posição:** Sempre na segunda linha do card, diretamente abaixo do breadcrumb, com `margin-top: 6px`.

**Tipografia:**

| Propriedade | Valor |
|---|---|
| Fonte | IBM Plex Sans |
| Tamanho | 24px |
| Peso | 600 |
| Cor | `#0f172a` |
| Letter-spacing | -0.01em |

---

### 3.3 Contexto do acesso

| Atributo | Valor |
|---|---|
| **ID** | `card1_contexto` |
| **Tipo de dado** | `string` (concatenação de 3 partes) |
| **Fonte de dados** | Sessão autenticada (Operadora + Domínio) |
| **Obrigatório** | Sim |
| **Editável pelo usuário** | Não |
| **Atualização** | Muda apenas quando o usuário faz login com outro perfil ou troca de domínio |

**Regra de composição:**

A frase é composta por três partes concatenadas com o delimitador ` · ` (espaço, ponto médio, espaço):

```
{texto_fixo} · {nome_prestador} · {dominio}
```

**Parte 1 — Texto fixo:**  
Sempre `Planta vinculada ao seu acesso`. Não varia.

**Parte 2 — Nome do prestador:**  
Origem: campo `nome_fantasia` da entidade `Operadora` vinculada ao usuário autenticado. A resolução é:

```
sessao.usuario → sessao.operadora_id → Operadora.nome_fantasia
```

Se o campo `nome_fantasia` for nulo ou vazio, usar `Operadora.razao_social` como fallback. Se ambos forem nulos (situação de erro cadastral), exibir `—` e registrar warning no log.

**Parte 3 — Domínio:**  
Origem: contexto de login da sessão. O TITAN opera com dois domínios independentes (Águas e Efluentes), e o Admin de cada domínio é uma instância separada conforme o documento executivo (seção 1, pressupostos arquiteturais: "Admin Águas e Admin Efluentes são independentes"). O domínio é determinado pela instância Admin em que o usuário se autenticou:

| Instância | Domínio exibido |
|---|---|
| Admin Águas | Abastecimento de Água |
| Admin Efluentes | Tratamento de Efluentes |

Se, em versão futura, o usuário tiver acesso cross-domain, exibe apenas o domínio da sessão ativa (não concatena os dois).

**Formato resultante:**

```
Planta vinculada ao seu acesso · SABESP · Abastecimento de Água
```

**Tipografia:**

| Propriedade | Valor — texto geral | Valor — nome do prestador |
|---|---|---|
| Fonte | IBM Plex Sans | IBM Plex Sans |
| Tamanho | 14px | 14px |
| Peso | 400 | 600 |
| Cor | `#64748b` | `#0f172a` |

---

## 4. Dependências

| Dependência | Módulo de origem | Obrigatória |
|---|---|---|
| `TAX002_navegacao_menu_admin` | Configuração de rotas do frontend | Sim |
| `CON001_sessao_contexto_acesso` | Sistema de autenticação (JWT + RBAC) | Sim |
| `Operadora.nome_fantasia` | #09 Operadoras (Estrutura de prestação) | Sim |

---

## 5. Estados possíveis

| Estado | Comportamento |
|---|---|
| Carregamento | Breadcrumb e título renderizam imediatamente (estáticos). Contexto exibe skeleton enquanto a sessão é resolvida. |
| Sessão válida | Todas as 3 variáveis renderizadas normalmente. |
| Operadora sem nome_fantasia | Fallback para razao_social. Se ambos nulos, exibe `—`. |
| Erro de sessão | Redireciona para login. Card 1 não renderiza. |

---

## 6. Referências cruzadas

| Artefato | Natureza | Relação |
|---|---|---|
| `TAX001_estado_planta` | TAX | Existente · Não referenciado por este card |
| `CAN001_calcular_estado_operacional_planta` | CAN | Existente · Não referenciado por este card |
| `TAX002_navegacao_menu_admin` | TAX | **Pendente** · Alimenta var 1 e var 2 |
| `CON001_sessao_contexto_acesso` | CON | **Pendente** · Alimenta var 3 |
| `SPE002_renderizar_situacao_planta` | SPE | Pendente · Card seguinte na mesma página |

---

## 7. Critérios de aceite

- [ ] Breadcrumb exibe exatamente 3 segmentos separados por ` / `.
- [ ] Segmentos do breadcrumb estão em uppercase, fonte IBM Plex Mono 11px.
- [ ] Título do submódulo está em formato título (não uppercase), fonte IBM Plex Sans 24px weight 600.
- [ ] Nome do prestador no contexto está em weight 600 e cor `#0f172a`, restante em weight 400 e cor `#64748b`.
- [ ] Segmentos do breadcrumb NÃO são clicáveis no MVP.
- [ ] Se `nome_fantasia` for nulo, exibe `razao_social`.
- [ ] O domínio exibido corresponde à instância Admin da sessão ativa.
