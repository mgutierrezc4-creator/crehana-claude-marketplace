# Crehana Claude Marketplace

Claude Code plugins for Crehana development workflows.

## Available Plugins

| Plugin | Contenido | Para quién |
|---|---|---|
| `centralized-users-dev` | Skills: new-mutation, new-query, new-pubsub-handler, new-use-case, new-adapter, test-conventions. Agents: code-reviewer, python-test-engineer | Devs del microservicio centralized-users |
| `general-skills` | Skill: explain-code | Cualquier dev del equipo |
| `browser-testing` | MCP de Playwright para automatización de browser | Quien necesite testing de UI |

## Instalación

### 1. Registrar el marketplace (una sola vez)

```
/plugin marketplace add <github-org>/crehana-claude-marketplace
```

### 2. Instalar los plugins que necesitas

```
/plugin install centralized-users-dev@crehana-claude-marketplace
/plugin install general-skills@crehana-claude-marketplace
/plugin install browser-testing@crehana-claude-marketplace
```

## Uso

### Skills disponibles (invocar con `/`)

| Skill | Cuándo se activa |
|---|---|
| `/new-mutation` | Crear una nueva GraphQL mutation en centralized-users |
| `/new-query` | Crear una nueva GraphQL query en centralized-users |
| `/new-pubsub-handler` | Crear un nuevo handler de PubSub |
| `/new-use-case` | Crear un nuevo use case con patrón de compensación |
| `/new-adapter` | Crear un adapter para Learning o Talent |
| `/test-conventions` | Escribir tests siguiendo las convenciones del proyecto |
| `/explain-code` | Explicar cómo funciona un bloque de código |

### Agents disponibles

- **code-reviewer**: Se activa automáticamente al pedir review de código, PRs o archivos. También puedes invocarlo explícitamente.
- **python-test-engineer**: Se activa automáticamente cuando se escribe código Python significativo que necesita tests.

## Actualizar plugins

```
/plugin update centralized-users-dev@crehana-claude-marketplace
```

## Contribuir

1. Hacer cambios en `plugins/<plugin>/`
2. Actualizar `version` en el `plugin.json` correspondiente
3. Crear un PR con los cambios
