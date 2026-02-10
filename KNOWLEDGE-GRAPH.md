---
id: knowledge-graph-paradigm
type: paradigm
title: "Progressive Disclosure via Knowledge Graph"
version: 1.0
created: 2026-02-05
---

# Knowledge Graph — Paradigma de Leitura Determinística

> **Princípio**: Progressive Disclosure de leitura determinística baseada em navegação de grafo de conhecimento estruturado em frontmatter YAML por cada peça.

---

## TL;DR

Cada arquivo de conhecimento declara suas próprias dependências no frontmatter YAML. Um script determinístico navega o grafo, resolve dependências, e monta contexto completo — **sem passar pelo orquestrador**.

```
Arquivo → declara dependências → grafo autodescritivo → resolução automática
```

---

## Problema Resolvido

| Antes | Depois |
|-------|--------|
| Script hard-coded com paths | Grafo autodescritivo |
| Nex carrega tudo (queima tokens) | Script resolve fora do LLM |
| Evolução quebra código | Evolução atualiza apenas frontmatter |
| Subagent "esquece" de carregar | Contexto injetado, não instruído |

---

## Arquitetura

```
┌─────────────────────────────────────────────────────────────┐
│                    KNOWLEDGE GRAPH                          │
│                                                             │
│  Workspace (indexed)           GDrive (on-demand)           │
│  ┌──────────────┐             ┌──────────────────┐         │
│  │ nina.md      │──requires──►│ metodo_nina.md   │         │
│  │ ray.md       │             │ K010_rotacao.md  │         │
│  │ mark.md      │             │ exposys.md       │         │
│  │ routines/*   │             │ workflows/*      │         │
│  │ domains/*    │             │ pills/*          │         │
│  └──────────────┘             └──────────────────┘         │
│         │                              │                    │
│         └──────────┬───────────────────┘                    │
│                    ▼                                        │
│          ┌─────────────────┐                               │
│          │  Graph Resolver │  (tools/knowledge_graph.py)   │
│          │  BFS + on-demand│                               │
│          └────────┬────────┘                               │
│                   ▼                                        │
│          ┌─────────────────┐                               │
│          │ Flattened Prompt│  ← Nex NUNCA vê este volume  │
│          │ (ready for spawn)│                              │
│          └─────────────────┘                               │
└─────────────────────────────────────────────────────────────┘
```

---

## Frontmatter Autodescritivo

Cada peça de conhecimento declara:

```yaml
---
id: unique-identifier
type: specialist | routine | domain | workflow | pill
title: "Human-readable title"
requires:
  - path/to/core_dependency.md      # Sempre carregado
optional:
  - path: path/to/conditional.md    # Formato condicional
    when: "regex|pattern|here"      # Carrega se task match
  - path/to/simple.md               # Formato simples (task-aware)
provides:
  - capability_a
  - capability_b
---
```

### Campos

| Campo | Obrigatório | Descrição |
|-------|-------------|-----------|
| `id` | Sim | Identificador único no grafo |
| `type` | Sim | Tipo do nodo (specialist, routine, domain, workflow, pill) |
| `title` | Não | Título legível |
| `requires` | Não | Dependências obrigatórias (sempre carregadas) |
| `optional` | Não | Dependências condicionais (ver formatos abaixo) |
| `provides` | Não | Capacidades que este nodo oferece |

### Formatos de Optional

**Simples** (carrega se qualquer task):
```yaml
optional:
  - ~/gdrive/.../pill.md
```

**Condicional** (carrega se task match regex):
```yaml
optional:
  - path: ~/gdrive/.../K011_formato.md
    when: "post|criar|formato|publicar"
  - path: ~/gdrive/.../guia_reviews.md
    when: "review|avaliação|reputação"
```

O `when` é um regex case-insensitive aplicado à task string.

### Tipos de Nodo

| Tipo | Descrição | Exemplo |
|------|-----------|---------|
| `specialist` | Agente Promethia (L3) | nina.md, ray.md, mark.md |
| `domain` | Contexto de domínio | diadema.md, caicara-marketing.md |
| `routine` | Workflow operacional | diadema-produtividade.md |
| `workflow` | Processo estruturado | WF-NINA-001.md |
| `pill` | Conhecimento atômico | K010_matriz_rotacao.md |

---

## Uso

### 1. Indexar o Grafo

```bash
python3 tools/knowledge_graph.py index
```

Escaneia workspace, parseia frontmatters, salva em `.cache/knowledge_graph.json`.

### 2. Consultar Sumário (Orquestrador)

```bash
# Índice geral
python3 tools/knowledge_graph.py summary

# Capacidades de um specialist
python3 tools/knowledge_graph.py summary nina
```

**Output** (~300 tokens):
```
## "NINA | Guardiã do Perfil Google"
### Required (always loaded)
- metodo_nina_gmn_lean

### Optional (specify to load)
| ID | Path | When |
| 1 | K010_rotacao | planejar|calendário |
| 2 | K011_formato | post|criar|formato |
| 3 | guia_reviews | review|avaliação |
```

### 3. Resolver Contexto

```bash
# Modo explícito (Nex especifica pills)
python3 tools/knowledge_graph.py resolve nina --pills K011 --task "criar post"

# Modo task-aware (regex match automático)
python3 tools/knowledge_graph.py resolve nina --task "criar post GMN"

# Modo mínimo (apenas required)
python3 tools/knowledge_graph.py resolve nina --json
```

### 4. Visualizar Estrutura

```bash
python3 tools/knowledge_graph.py show nina
```

---

## Arquitetura Híbrida

```
┌─────────────────────────────────────────────────────────────┐
│  NEX (Orquestrador)                                         │
│  ┌─────────────────┐                                       │
│  │ summary (~300t) │  ← Índice de capacidades, não conteúdo │
│  └────────┬────────┘                                       │
│           │ decide: nina + K011                             │
│           ▼                                                 │
│  resolve nina --pills K011 --task "criar post"             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│  SUBAGENT                                                   │
│  Recebe: nina.md + método + K011 (~5K, não ~15K)           │
└─────────────────────────────────────────────────────────────┘
```

### Três Modos de Resolução

| Modo | Flag | Comportamento |
|------|------|---------------|
| **Explícito** | `--pills K011 K003` | Carrega apenas pills especificadas |
| **Task-aware** | `--task "criar post"` | Regex match nos patterns `when` |
| **Mínimo** | (sem flags) | Carrega apenas `requires` |

### Economia de Tokens

| Papel | Antes | Agora |
|-------|-------|-------|
| Nex (roteamento) | ~15K (lia tudo) | ~300 (sumário) |
| Subagent (execução) | ~15K (tudo) | ~5K (focado) |

---

## Fluxo de Execução

```
1. Nex recebe tarefa
2. Nex consulta: summary nina (~300 tokens)
3. Nex decide caminho: specialist=nina, pills=[K011]
4. Nex executa: resolve nina --pills K011 --task "..."
5. Script navega grafo, carrega APENAS dependências necessárias
6. Nex faz sessions_spawn com prompt focado
7. Subagent executa com contexto mínimo necessário
```

**Resultado**: Nex vê ~300 tokens (sumário), subagent recebe ~5K tokens (contexto focado).

**Nota técnica**: O output do resolve (~5K) passa pelo contexto de Nex antes do spawn. É overhead aceitável — blob opaco, não processado, não acumula.

---

## Evolução do Grafo

Ao adicionar novo conhecimento:

1. **Criar arquivo** com frontmatter declarando dependências
2. **Re-indexar**: `python3 knowledge_graph.py index`
3. **Testar**: `python3 knowledge_graph.py show <new-id>`

Nenhum código precisa mudar — o grafo se auto-atualiza.

---

## Princípios

1. **Autodescritivo**: Conhecimento declara próprias dependências
2. **Determinístico**: Mesma entrada → mesmo contexto
3. **Lazy Loading**: GDrive carregado on-demand, não indexado
4. **Token Efficient**: Orquestrador não carrega conteúdo
5. **Evolutivo**: Frontmatter atualiza, código permanece

---

## Arquivos Relacionados

| Arquivo | Propósito |
|---------|-----------|
| `tools/knowledge_graph.py` | Graph resolver (index, resolve, show) |
| `tools/invoke_specialist.py` | Wrapper para invocação de specialists |
| `config/specialists.json` | Configuração de specialists (legacy, sendo migrado) |
| `.cache/knowledge_graph.json` | Cache do grafo indexado |

---

*Paradigma estabelecido: 2026-02-05*
*"Progressive Disclosure via Knowledge Graph"*
