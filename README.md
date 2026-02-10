# Knowledge Graph Tool

Self-describing knowledge graph for context resolution in Promethia/OpenClaw.

Each knowledge file declares its dependencies in YAML frontmatter. The graph resolver navigates dependencies automatically, building complete context packages for subagent injection.

## Usage

```bash
# Build/update graph index
python3 knowledge_graph.py index

# Resolve context for entry point
python3 knowledge_graph.py resolve knowledge/specialists/nina.md --task "create GMN post"

# Show graph structure
python3 knowledge_graph.py show knowledge/specialists/nina.md
```

## Architecture

See KNOWLEDGE-GRAPH.md for the full paradigm documentation.
