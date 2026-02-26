# Patrones aprendidos de get-shit-done (GSD)

> Fuente: https://github.com/gsd-build/get-shit-done
> Estrellas: ~20.5k (feb 2026)
> Fecha de analisis: 2026-02-26

## Estructura de Agentes (.claude/agents/*.md)

GSD define agentes como archivos Markdown con YAML frontmatter:

```yaml
---
name: gsd-executor
description: Executes GSD plans with atomic commits...
tools: Read, Write, Edit, Bash, Grep, Glob
color: yellow
---
```

### Campos del frontmatter

| Campo | Obligatorio | Descripcion |
|-------|-------------|-------------|
| `name` | Si | Nombre del agente (usado en Task tool `subagent_type`) |
| `description` | Si | Descripcion corta de lo que hace |
| `tools` | Si | Lista de tools que el agente puede usar |
| `color` | No | Color del agente en el statusline |

### Cuerpo del agente

GSD usa XML tags para estructurar secciones:
- `<role>` — Quien eres, que haces, responsabilidades core
- `<project_context>` — Como descubrir contexto del proyecto (CLAUDE.md, skills)
- `<execution_flow>` con `<step name="..." priority="...">` — Pasos de ejecucion
- `<deviation_rules>` — Reglas para cuando algo sale del plan
- `<success_criteria>` — Cuando considerar el trabajo completo
- `<output_format>` — Formato exacto de la salida esperada

### Patrones clave de agentes GSD

1. **Mandatory Initial Read**: Si el prompt tiene `<files_to_read>`, cargar esos archivos primero
2. **Structured Returns**: Siempre retornar formato estructurado (markdown con headers fijos)
3. **Checkpoint Protocol**: Para tareas largas, checkpoints donde el agente PARA y retorna
4. **Deviation Rules**: Reglas claras de que auto-fixear vs que preguntar
5. **Model Resolution**: El orquestador resuelve el modelo del agente dinamicamente
6. **Files to read pattern**: El prompt incluye una lista explicita de archivos para precargar contexto

## Estructura de Commands (.claude/commands/gsd/*.md)

Son slash commands del usuario (equivalente a nuestros skills invocables).

```yaml
---
name: gsd:debug
description: Systematic debugging with persistent state
argument-hint: [issue description]
allowed-tools:
  - Read
  - Bash
  - Task
  - AskUserQuestion
---
```

### Campos del frontmatter

| Campo | Obligatorio | Descripcion |
|-------|-------------|-------------|
| `name` | Si | Nombre del comando (user invoca `/gsd:debug`) |
| `description` | Si | Descripcion |
| `argument-hint` | No | Hint de argumentos |
| `allowed-tools` | No | Tools restringidos |

### Patron clave: Command = Orquestador

Los commands de GSD son **orquestadores** que:
1. Recopilan informacion del usuario (AskUserQuestion)
2. Cargan estado (via CLI tools)
3. Spawean agentes (Task tool)
4. Manejan respuestas/checkpoints del agente
5. Spawean continuaciones si necesario

El agente hace el trabajo pesado; el command maneja el flujo.

## Hooks (.claude/hooks/*.js)

### Formato critico: Node.js, no Bash

GSD usa **Node.js** para todos los hooks. Las razones:
1. JSON parsing nativo (sin depender de jq)
2. Sin bugs de jq (el operador `//` trata `0` como falsy)
3. Acceso a filesystem (fs), path, os
4. Manejo de errores mas robusto

### Hook Input Format (PostToolUse para Bash)

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/Users/project",
  "permission_mode": "default",
  "hook_event_name": "PostToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "git push origin develop",
    "description": "Push to remote",
    "timeout": 120000,
    "run_in_background": false
  },
  "tool_response": {
    "stdout": "...",
    "stderr": "...",
    "exit_code": 0
  },
  "tool_use_id": "toolu_01ABC123..."
}
```

### Hook Output Formats

**Inyectar context (PostToolUse):**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "Mensaje para Claude"
  }
}
```

**System message:**
```json
{
  "systemMessage": "Mensaje que Claude ve como system reminder"
}
```

**Bloquear accion (PreToolUse):**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Razon"
  }
}
```

**Permitir sin preguntar:**
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow"
  }
}
```

### Hooks de GSD (referencia)

1. **gsd-context-monitor.js** (PostToolUse) — Monitorea uso de contexto via bridge file
   - Statusline escribe metricas a `/tmp/claude-ctx-{session_id}.json`
   - Context monitor lee ese archivo y inyecta warnings cuando context < 35%/25%
   - Debounce: 5 tool uses entre warnings

2. **gsd-statusline.js** (Statusline) — Muestra modelo, tarea actual, directorio, contexto
   - Lee todos desde stdin (no shell env vars)
   - Lee estado de `.claude/todos/` para task in_progress

3. **gsd-check-update.js** (SessionStart) — Checa version nueva en npm

### Bug comun: jq y `0` como falsy

```bash
# BUG: jq trata 0 como falsy
exit_code=$(echo "$input" | jq -r '.tool_response.exit_code // "1"')
# Si exit_code es 0 → jq lo descarta → retorna "1"

# FIX opcion 1: usar if/then/else en jq
exit_code=$(echo "$input" | jq -r 'if .tool_response.exit_code != null then (.tool_response.exit_code | tostring) else "1" end')

# FIX opcion 2: usar Node.js (recomendado)
const data = JSON.parse(input);
const exitCode = data.tool_response?.exit_code ?? 1;
```

## State Management

GSD usa Markdown con YAML frontmatter (`.planning/STATE.md`):
- Campos extraidos via regex (`**Field:** value`)
- YAML frontmatter auto-sincronizado al escribir
- CLI tool (`gsd-tools.cjs`) para operaciones atomicas
- Bajo 100 lineas — es un digest, no un archivo

### Patron Bridge (inter-hook communication)

Los hooks NO pueden comunicarse directamente entre si. GSD usa un "bridge file" en `/tmp/`:
- Statusline escribe → Context monitor lee
- Session-scoped: nombre del archivo incluye `session_id`
- Best-effort: si falla la lectura/escritura, continua silencioso

## Patrones generales

1. **Silent fail**: Hooks NUNCA bloquean la ejecucion. Try/catch con exit(0)
2. **stdin/stdout**: Toda comunicacion es via stdin (JSON in) → stdout (JSON out)
3. **No shell state**: Hooks son stateless entre invocaciones (usan filesystem para persistir)
4. **Session awareness**: Usan `session_id` para scope de estado temporal
5. **Agents are disposable**: Cada spawn es fresco (200k context), estado persiste en files
