# SPE001_renderizar_cabecalho_pagina

> **Natureza**: `SPE` — specification  
> **ID**: `SPE001`  
> **Camada**: frontend  
> **Escopo**: global — aplica-se a todas as páginas de todos os 21 itens do menu Admin  
> **Função canônica referenciada**: —  
> **Versão**: 1.1.0  
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
> | `TAX002` | `TAX` | `navegacao_menu_admin` | Tabela estruturada (JSON/enum) de grupos → itens → sub-páginas do menu Admin. Hoje essa informação existe apenas no PDF executivo v2.3, que não é consumível por código. O breadcrumb (var 1) e o título (var 2) dependem dela para resolver os valores em qualquer página. |
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
  TAX001  → estado_planta
  CAN001  → calcular_estado_operacional_planta
```

---

## 1. Objetivo

Definir as regras autossuficientes do componente **Cabeçalho de página**, que é renderizado como primeiro elemento visual de toda página do menu Admin TITAN. Este componente é **global**: a mesma estrutura, tipografia e lógica se aplicam aos 21 itens do menu e a todas as suas sub-páginas. Apenas os valores exibidos mudam conforme a navegação.

Este documento é a fonte de verdade única para implementação frontend do cabeçalho. Nenhuma spec de card ou módulo individual precisa redefinir este componente — basta referenciá-lo como `SPE001`.

---

## 2. Estrutura visual

```
┌─────────────────────────────────────────────────────────────────────┐
│ {GRUPO FUNCIONAL} / {ITEM DO MENU} / {SUBMÓDULO ATIVO}            │  ← Var 1
│ {Nome do submódulo}                                                 │  ← Var 2
│ {Contexto do acesso}                                                │  ← Var 3
└─────────────────────────────────────────────────────────────────────┘
```

**Exemplos de instâncias reais:**

```
Item #01, sub-página Recursos & Funcionalidades:
  CONFIGURAÇÕES GLOBAIS / GESTÃO DA PLATAFORMA / RECURSOS & FUNCIONALIDADES
  Recursos & Funcionalidades
  Planta vinculada ao seu acesso · SABESP · Abastecimento de Água

Item #01, sub-página Identidade & acesso:
  CONFIGURAÇÕES GLOBAIS / GESTÃO DA PLATAFORMA / IDENTIDADE & ACESSO
  Identidade & acesso
  Planta vinculada ao seu acesso · SABESP · Abastecimento de Água

Item #05, sub-página Tags SCADA:
  OPERAÇÃO E ATIVOS / CONFIGURAÇÃO SCADA & IOT / TAGS SCADA
  Tags SCADA
  Planta vinculada ao seu acesso · SABESP · Abastecimento de Água

Item #12, sub-página Conformidade snapshot:
  ESTRUTURA DE PRESTAÇÃO / PLANTAS / CONFORMIDADE SNAPSHOT
  Conformidade snapshot
  Planta vinculada ao seu acesso · SABESP · Abastecimento de Água

Item #15, sub-página Renovações & vencimentos:
  NORMATIVOS E QUALIDADE / OUTORGAS / RENOVAÇÕES & VENCIMENTOS
  Renovações & vencimentos
  Planta vinculada ao seu acesso · SABESP · Abastecimento de Água
```

---

## 3. Variáveis

### 3.1 Breadcrumb de navegação

| Atributo | Valor |
|---|---|
| **ID** | `cabecalho_breadcrumb` |
| **Tipo de dado** | `string` (concatenação de 3 segmentos) |
| **Fonte de dados** | `TAX002_navegacao_menu_admin` (pendente) |
| **Obrigatório** | Sim |
| **Editável pelo usuário** | Não |
| **Atualização** | Muda quando o usuário navega para outra página |

**Regra de composição:**

O breadcrumb é montado por três segmentos separados pelo delimitador ` / ` (espaço, barra, espaço):

```
{grupo_funcional} / {item_menu} / {submodulo_ativo}
```

**Segmento 1 — Grupo funcional:**  
Derivado da tabela de 5 grupos definida em `TAX002`. Resolução: dado o item de menu ativo, buscar o grupo ao qual pertence.

| ID do grupo | Nome exibido | Itens pertencentes |
|---|---|---|
| `config_globais` | CONFIGURAÇÕES GLOBAIS | #01, #02, #03 |
| `operacao_ativos` | OPERAÇÃO E ATIVOS | #04, #05, #06, #07, #08 |
| `estrutura_prestacao` | ESTRUTURA DE PRESTAÇÃO | #09, #10, #11, #12, #13 |
| `normativos_qualidade` | NORMATIVOS E QUALIDADE | #14, #15, #16, #17 |
| `conformidade_reporte` | CONFORMIDADE E REPORTE | #18, #19, #20, #21 |

**Segmento 2 — Item do menu:**  
Nome do item #01 a #21 conforme `TAX002`. Exemplos: `GESTÃO DA PLATAFORMA`, `CONFIGURAÇÃO SCADA & IOT`, `PLANTAS`, `OUTORGAS`, `PACOTES REGULATÓRIOS`.

**Segmento 3 — Submódulo ativo:**  
Nome da sub-página dentro do item conforme `TAX002`. Cada item tem N sub-páginas (ex: item #01 tem Recursos & Funcionalidades, Identidade & acesso, Privacidade & LGPD, etc.; item #12 tem Lista de plantas, Trens de tratamento, Vínculos institucionais, Outorga & captação, Conformidade snapshot).

**Comportamento de navegação:**
- Mudança de sub-página dentro do mesmo item: apenas segmento 3 muda.
- Mudança de item dentro do mesmo grupo: segmentos 2 e 3 mudam.
- Mudança de grupo: todos os 3 segmentos mudam.
- Nenhum segmento é clicável no MVP (Onda 1). Na v2, os segmentos 1 e 2 se tornam links de navegação.

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
| **ID** | `cabecalho_titulo` |
| **Tipo de dado** | `string` |
| **Fonte de dados** | `TAX002_navegacao_menu_admin` (pendente) |
| **Obrigatório** | Sim |
| **Editável pelo usuário** | Não |
| **Atualização** | Muda quando o usuário navega para outra página |

**Regra de composição:**

Exibe o nome legível do submódulo ativo. É o mesmo valor do segmento 3 do breadcrumb, porém em formato título (capitalização normal, não uppercase).

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
| **ID** | `cabecalho_contexto` |
| **Tipo de dado** | `string` (concatenação de 3 partes) |
| **Fonte de dados** | `CON001_sessao_contexto_acesso` (pendente) |
| **Obrigatório** | Sim |
| **Editável pelo usuário** | Não |
| **Atualização** | Muda apenas quando o usuário faz login com outro perfil ou troca de domínio. Não muda com navegação entre páginas. |

**Regra de composição:**

A frase é composta por três partes concatenadas com o delimitador ` · ` (espaço, ponto médio, espaço):

```
{texto_fixo} · {nome_prestador} · {dominio}
```

**Parte 1 — Texto fixo:**  
Sempre `Planta vinculada ao seu acesso`. Não varia. Não depende de navegação.

**Parte 2 — Nome do prestador:**  
Origem: campo `nome_fantasia` da entidade `Operadora` vinculada ao usuário autenticado, conforme `CON001`. A resolução é:

```
sessao.usuario → sessao.operadora_id → Operadora.nome_fantasia
```

Se o campo `nome_fantasia` for nulo ou vazio, usar `Operadora.razao_social` como fallback. Se ambos forem nulos (situação de erro cadastral), exibir `—` e registrar warning no log.

**Parte 3 — Domínio:**  
Origem: contexto de login da sessão conforme `CON001`. O TITAN opera com dois domínios independentes (Águas e Efluentes), e o Admin de cada domínio é uma instância separada conforme o documento executivo (seção 1, pressupostos arquiteturais).

| Instância | Domínio exibido |
|---|---|
| Admin Águas | Abastecimento de Água |
| Admin Efluentes | Tratamento de Efluentes |

Se, em versão futura, o usuário tiver acesso cross-domain, exibe apenas o domínio da sessão ativa.

**Formato resultante (mesmo em qualquer página):**

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

| Dependência | Artefato | Status |
|---|---|---|
| Tabela de navegação (grupos, itens, sub-páginas) | `TAX002_navegacao_menu_admin` | **Pendente** |
| Contrato de sessão (prestador, domínio) | `CON001_sessao_contexto_acesso` | **Pendente** |
| Campo `nome_fantasia` da operadora | #09 Operadoras | Existente no modelo de dados |

---

## 5. Estados possíveis

| Estado | Comportamento |
|---|---|
| Carregamento | Breadcrumb e título renderizam imediatamente (derivados da rota, sem chamada de API). Contexto exibe skeleton enquanto a sessão é resolvida. |
| Sessão válida | Todas as 3 variáveis renderizadas normalmente. |
| Operadora sem nome_fantasia | Fallback para razao_social. Se ambos nulos, exibe `—`. |
| Erro de sessão | Redireciona para login. Cabeçalho não renderiza. |
| Item desabilitado (fase 2/3) | Cabeçalho renderiza normalmente. O conteúdo abaixo do cabeçalho é que exibe estado de desabilitado, não o cabeçalho. |

---

## 6. Referências cruzadas

| Artefato | Natureza | Relação |
|---|---|---|
| `TAX001_estado_planta` | TAX | Existente · Não referenciado por este componente |
| `CAN001_calcular_estado_operacional_planta` | CAN | Existente · Não referenciado por este componente |
| `TAX002_navegacao_menu_admin` | TAX | **Pendente** · Alimenta var 1 e var 2 |
| `CON001_sessao_contexto_acesso` | CON | **Pendente** · Alimenta var 3 |

**Quem referencia este componente:**  
Toda spec de card ou página que tenha um cabeçalho (ou seja, todas as páginas do Admin) deve declarar `SPE001` como dependência e não redefinir breadcrumb, título ou contexto.

---

## 7. Critérios de aceite

- [ ] Componente renderiza em todas as páginas dos 21 itens do menu Admin.
- [ ] Breadcrumb exibe exatamente 3 segmentos separados por ` / `.
- [ ] Segmentos do breadcrumb estão em uppercase, fonte IBM Plex Mono 11px.
- [ ] Título do submódulo está em formato título (não uppercase), fonte IBM Plex Sans 24px weight 600.
- [ ] Nome do prestador no contexto está em weight 600 e cor `#0f172a`, restante em weight 400 e cor `#64748b`.
- [ ] Segmentos do breadcrumb NÃO são clicáveis no MVP.
- [ ] Se `nome_fantasia` for nulo, exibe `razao_social`.
- [ ] O domínio exibido corresponde à instância Admin da sessão ativa.
- [ ] Contexto do acesso (var 3) não muda ao navegar entre páginas — permanece fixo durante a sessão.
- [ ] Breadcrumb e título (var 1 e var 2) atualizam corretamente ao navegar entre sub-páginas, itens e grupos.
