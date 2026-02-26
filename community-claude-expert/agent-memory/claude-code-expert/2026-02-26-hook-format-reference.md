# Claude Code Hook Format Reference

> Fecha: 2026-02-26
> Fuente: Documentacion oficial + GSD repo (verificacion cruzada)

## Hook Events Disponibles

| Event | Cuando se dispara | Puede bloquear |
|-------|-------------------|----------------|
| `PreToolUse` | Antes de ejecutar una tool | Si (deny/allow/ask) |
| `PostToolUse` | Despues de ejecutar una tool | Si (decision: block) |
| `PostToolUseFailure` | Despues de un error en una tool | Si |
| `Notification` | Cuando Claude quiere notificar algo | No |
| `Stop` | Cuando Claude termina su turno | Si |
| `SubagentStop` | Cuando un subagente termina | Si |
| `UserPromptSubmit` | Cuando el usuario envia un mensaje | Si |
| `SessionStart` | Al iniciar sesion | No |
| `ConfigChange` | Al cambiar configuracion | Si |

## Input JSON por Tool

### Bash (PostToolUse)

```json
{
  "session_id": "string",
  "transcript_path": "string (absolute path)",
  "cwd": "string (absolute path)",
  "permission_mode": "default|allowAll|...",
  "hook_event_name": "PostToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "string",
    "description": "string | undefined",
    "timeout": "number (ms)",
    "run_in_background": "boolean"
  },
  "tool_response": {
    "stdout": "string",
    "stderr": "string",
    "exit_code": "number (integer, NOT string)"
  },
  "tool_use_id": "string"
}
```

### Read (PostToolUse)

```json
{
  "tool_name": "Read",
  "tool_input": {
    "file_path": "string (absolute path)",
    "offset": "number | undefined",
    "limit": "number | undefined"
  },
  "tool_response": {
    "content": "string (file content)"
  }
}
```

### Edit (PreToolUse)

```json
{
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "string (absolute path)",
    "old_string": "string",
    "new_string": "string",
    "replace_all": "boolean"
  }
}
```

### Write (PreToolUse)

```json
{
  "tool_name": "Write",
  "tool_input": {
    "file_path": "string (absolute path)",
    "content": "string"
  }
}
```

### Task (PostToolUse)

```json
{
  "tool_name": "Task",
  "tool_input": {
    "prompt": "string",
    "subagent_type": "string",
    "description": "string"
  }
}
```

## Output JSON Patterns

### 1. Inyectar contexto adicional

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "Texto que Claude ve en su contexto"
  }
}
```

### 2. System message (visible como system-reminder)

```json
{
  "systemMessage": "Texto del sistema"
}
```

### 3. Combinado (context + system message)

```json
{
  "systemMessage": "Mensaje corto",
  "hookSpecificOutput": {
    "hookEventName": "PostToolUse",
    "additionalContext": "Contexto extendido para Claude"
  }
}
```

### 4. PreToolUse - Controlar permisos

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow|deny|ask",
    "permissionDecisionReason": "Explicacion",
    "updatedInput": { "command": "comando modificado" },
    "additionalContext": "Contexto extra"
  }
}
```

### 5. Bloquear una accion

```json
{
  "decision": "block",
  "reason": "Razon por la que se bloquea"
}
```

### 6. Detener a Claude

```json
{
  "continue": false,
  "stopReason": "Razon para parar"
}
```

## settings.json - Configuracion de Hooks

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "path/to/hook.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "path/to/hook.js"
          }
        ]
      }
    ]
  }
}
```

### Matcher patterns

- `"Bash"` — solo Bash tool
- `"Edit|Write"` — Edit o Write
- `"Read"` — solo Read
- `"Task"` — solo Task (subagentes)
- No soporta regex avanzado, solo nombres de tools separados por `|`

## Gotchas Importantes

1. **exit_code es integer**: En tool_response, exit_code viene como number, NO string
2. **jq `//` bug**: El operador alternativo de jq trata `0`, `false`, `null` como falsy
3. **Hooks deben ser rapidos**: Hooks lentos bloquean la UX de Claude Code
4. **Silent fail**: Si un hook crashea, Claude Code lo ignora silenciosamente
5. **No interactive**: Hooks no pueden pedir input al usuario
6. **stdout = output**: Todo lo que se imprime a stdout es el output del hook
7. **stderr = logs**: Stderr va a logs de Claude Code, no afecta el output
8. **exit 0 = normal**: Siempre salir con 0. Exit 2 = legacy block.
9. **$CLAUDE_PROJECT_DIR**: Variable de entorno disponible en hooks
10. **$HOME/.claude/**: Directorio global de Claude Code

## Recomendacion: Node.js sobre Bash para hooks

| Aspecto | Bash + jq | Node.js |
|---------|-----------|---------|
| JSON parsing | Fragil (jq bugs) | Nativo |
| Error handling | Limitado | Try/catch |
| Filesystem | Basico | fs completo |
| Strings | Fragil con espacios | Robusto |
| Portabilidad | Depende de jq instalado | Node siempre presente (Claude Code lo requiere) |
| Debugging | Dificil | console.error va a logs |

GSD usa Node.js exclusivamente para hooks. La ventaja principal es evitar el bug de jq con `0` como falsy.
