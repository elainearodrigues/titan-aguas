# SPEC_01_01_card1_cabecalho — Cabeçalho de página

> **Tipo**: specification  
> **Camada**: frontend  
> **Submódulo**: 01 · Gestão da plataforma → Recursos & Funcionalidades  
> **Função canônica referenciada**: —  
> **Versão**: 1.0.0  
> **Data**: 03 mai 2026  
> **Status**: definido  

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
| Tabela de grupos funcionais (5 grupos) | Documento executivo TITAN v2.3, seção 2 | Sim |
| Tabela de itens do menu (21 itens) | Documento executivo TITAN v2.3, seção 2 | Sim |
| Tabela de sub-páginas por item | Configuração de rotas do frontend | Sim |
| `Operadora.nome_fantasia` | #09 Operadoras (Estrutura de prestação) | Sim |
| Contexto de domínio da sessão | Sistema de autenticação (JWT + RBAC) | Sim |

---

## 5. Estados possíveis

| Estado | Comportamento |
|---|---|
| Carregamento | Breadcrumb e título renderizam imediatamente (estáticos). Contexto exibe skeleton enquanto a sessão é resolvida. |
| Sessão válida | Todas as 3 variáveis renderizadas normalmente. |
| Operadora sem nome_fantasia | Fallback para razao_social. Se ambos nulos, exibe `—`. |
| Erro de sessão | Redireciona para login. Card 1 não renderiza. |

---

## 6. Critérios de aceite

- [ ] Breadcrumb exibe exatamente 3 segmentos separados por ` / `.
- [ ] Segmentos do breadcrumb estão em uppercase, fonte IBM Plex Mono 11px.
- [ ] Título do submódulo está em formato título (não uppercase), fonte IBM Plex Sans 24px weight 600.
- [ ] Nome do prestador no contexto está em weight 600 e cor `#0f172a`, restante em weight 400 e cor `#64748b`.
- [ ] Segmentos do breadcrumb NÃO são clicáveis no MVP.
- [ ] Se `nome_fantasia` for nulo, exibe `razao_social`.
- [ ] O domínio exibido corresponde à instância Admin da sessão ativa.
