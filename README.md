# Crehana Claude Marketplace

Claude Code plugins para los flujos de desarrollo de Crehana.

## Plugins disponibles

| Plugin | Contenido | Para quién |
|---|---|---|
| `centralized-users-dev` | 6 skills de scaffolding + 2 agents | Devs del microservicio `centralized-users` |
| `general-skills` | Skill `explain-code` | Cualquier dev del equipo |
| `browser-testing` | MCP de Playwright | Quien necesite automatización de browser |
| `atlassian` | MCP de Jira + Confluence (OAuth) | Todo el equipo |

---

## Instalación

### Requisitos previos

- [Claude Code](https://claude.ai/code) instalado y con sesión activa
- Para `browser-testing`: **Node.js v18+** instalado (el MCP usa `npx`)

#### Instalar Node.js según tu sistema operativo

| OS | Método recomendado |
|---|---|
| **macOS** | `brew install node` o descargar desde [nodejs.org](https://nodejs.org) |
| **Linux** | `sudo apt install nodejs npm` (Ubuntu/Debian) · `sudo dnf install nodejs` (Fedora) · o usar [nvm](https://github.com/nvm-sh/nvm) |
| **Windows** | Descargar el instalador `.msi` desde [nodejs.org](https://nodejs.org) (incluye `npx` automáticamente en el PATH) |

Verifica que Node.js está instalado correctamente:
```bash
node --version   # debe mostrar v18 o superior
npx --version
```

> El plugin `atlassian` no requiere Node.js — usa HTTP directamente.

### Paso 1 — Registrar el marketplace

Ejecuta este comando **una sola vez** en Claude Code. Abre Claude Code y escribe:

```
/plugin marketplace add <github-org>/crehana-claude-marketplace
```

> Reemplaza `<github-org>` con la organización o usuario de GitHub donde está el repo.
> Ejemplo: `/plugin marketplace add crehana/crehana-claude-marketplace`

Esto descarga el catálogo y lo deja disponible para instalar plugins desde él.

### Paso 2 — Instalar los plugins que necesitas

Instala solo los que vas a usar:

```
/plugin install centralized-users-dev@crehana-claude-marketplace
```
```
/plugin install general-skills@crehana-claude-marketplace
```
```
/plugin install browser-testing@crehana-claude-marketplace
```
```
/plugin install atlassian@crehana-claude-marketplace
```

Los plugins se instalan a nivel de **usuario** (disponibles en todos tus proyectos).

### Verificar la instalación

Para confirmar que los plugins quedaron activos:

```
/plugin list
```

Deberías ver `centralized-users-dev`, `general-skills` y/o `browser-testing` en la lista.

---

## Qué incluye cada plugin

### `centralized-users-dev`

Skills de scaffolding para el microservicio `crehana-centralized-users`. Se activan automáticamente cuando describes lo que necesitas, o puedes invocarlas manualmente con `/`.

| Skill | Se activa cuando... |
|---|---|
| `/new-mutation` | Quieres crear una nueva GraphQL mutation |
| `/new-query` | Quieres crear una nueva GraphQL query |
| `/new-pubsub-handler` | Quieres agregar un handler de PubSub |
| `/new-use-case` | Necesitas implementar un nuevo use case |
| `/new-adapter` | Necesitas llamar a Learning o Talent API |
| `/test-conventions` | Quieres escribir tests para cualquier componente |

Agents incluidos (se activan automáticamente):

- **`code-reviewer`** — revisa código, PRs y archivos contra las convenciones del proyecto. Se activa cuando pides review o después de escribir código significativo.
- **`python-test-engineer`** — genera suites de tests completas. Se activa cuando se escribe código Python que necesita cobertura.

### `general-skills`

| Skill | Se activa cuando... |
|---|---|
| `/explain-code` | Preguntas cómo funciona un bloque de código |

### `browser-testing`

Instala el [MCP de Playwright](https://github.com/microsoft/playwright-mcp) de Microsoft. Permite a Claude navegar páginas, hacer clic, llenar formularios y tomar screenshots directamente.

Una vez instalado, Claude tendrá acceso a herramientas como `browser_navigate`, `browser_click`, `browser_screenshot`, etc.

### `atlassian`

Instala el [MCP oficial de Atlassian](https://mcp.atlassian.com). Conecta Claude directamente con Jira y Confluence (31 herramientas disponibles).

Lo que puedes hacer:
- **Jira**: crear, editar y buscar issues, agregar comentarios y worklogs, transicionar tickets, vincular issues
- **Confluence**: leer y crear páginas, buscar con CQL, agregar comentarios inline y de pie de página

> **Requiere autenticación OAuth.** Al instalar el plugin, Claude Code abrirá una ventana en el browser para autenticarte con tu cuenta de Atlassian. Una vez autenticado, la sesión persiste automáticamente.

---

## Actualizar plugins

Cuando haya nuevas versiones disponibles:

```
/plugin update centralized-users-dev@crehana-claude-marketplace
/plugin update general-skills@crehana-claude-marketplace
/plugin update browser-testing@crehana-claude-marketplace
/plugin update atlassian@crehana-claude-marketplace
```

O para actualizar todos a la vez:

```
/plugin update --all
```

---

## Desinstalar

```
/plugin uninstall centralized-users-dev@crehana-claude-marketplace
```

---

## Estructura del repositorio

```
crehana-claude-marketplace/
├── .claude-plugin/
│   └── marketplace.json        # Catálogo de plugins
├── skills/                     # Skills para centralized-users-dev y general-skills
│   ├── new-mutation/
│   ├── new-query/
│   ├── new-pubsub-handler/
│   ├── new-use-case/
│   ├── new-adapter/
│   ├── test-conventions/
│   └── explain-code/
├── agents/                     # Agents incluidos en centralized-users-dev
│   ├── code-reviewer.md
│   └── python-test-engineer.md
└── plugins/
    ├── browser-testing/        # Plugin MCP: Playwright
    │   ├── .claude-plugin/plugin.json
    │   └── .mcp.json
    └── atlassian/              # Plugin MCP: Jira + Confluence (OAuth)
        ├── .claude-plugin/plugin.json
        └── .mcp.json
```

---

## Contribuir

1. Crea una rama: `git checkout -b feature/mi-skill`
2. Agrega o modifica archivos en `skills/` o `agents/`
3. Si es una skill nueva, agrégala al array `skills` correspondiente en `.claude-plugin/marketplace.json`
4. Abre un PR hacia `main`
