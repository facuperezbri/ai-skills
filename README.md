# Agent Skills

Colección de skills para Claude Code y Cursor.

## Instalación

```bash
npx skills add facuperezbri/jira-analyst-skill
```

Pregunta en qué editores instalar → Claude Code, Cursor, o ambos.

## Skills

| Skill | Descripción | MCPs requeridos |
|-------|-------------|-----------------|
| [jira-analyst-skill](skills/jira-analyst-skill/SKILL.md) | Analiza tickets de Jira y produce propuestas funcionales o técnicas | `mcp-atlassian` · `bitbucket` (opcional) |

---

## Configuración de MCP servers

Las skills de este repo requieren MCP servers para conectarse a Jira, Confluence y (opcionalmente) Bitbucket. Hay dos formas de configurarlos:

### Opción A: Script interactivo (recomendado)

El script te pide las credenciales y genera los archivos de configuración automáticamente.

**Prerequisito — instalar `uv`:**

```bash
# Mac / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

**Ejecutar el setup:**

```bash
# Mac / Linux
chmod +x setup/mcp-setup.sh && ./setup/mcp-setup.sh

# Windows
.\setup\mcp-setup.ps1
```

El script genera `.mcp.json` (Claude Code) y `.cursor/mcp.json` (Cursor) en la raíz del proyecto. Ambos archivos están en `.gitignore` — tus credenciales no se suben al repo.

---

### Opción B: Configuración manual

Si preferís configurar los MCPs a mano, podés editar directamente los archivos de configuración.

#### Archivos de configuración

| Editor | Archivo | Scope |
|--------|---------|-------|
| Claude Code | `.mcp.json` (raíz del proyecto) | Proyecto (compartido vía git, sin credenciales) |
| Claude Code | `~/.claude.json` | Global (todas tus sesiones) |
| Cursor | `.cursor/mcp.json` (raíz del proyecto) | Proyecto |

#### MCP requerido: `mcp-atlassian`

Conecta con Jira y Confluence. Usa [`mcp-atlassian`](https://github.com/sooperset/mcp-atlassian) vía `uvx`.

```json
{
  "mcpServers": {
    "mcp-atlassian": {
      "type": "stdio",
      "command": "uvx",
      "args": ["mcp-atlassian"],
      "env": {
        "JIRA_URL": "https://tuempresa.atlassian.net/",
        "JIRA_USERNAME": "tu@email.com",
        "JIRA_API_TOKEN": "tu-api-token",
        "CONFLUENCE_URL": "https://tuempresa.atlassian.net/",
        "CONFLUENCE_USERNAME": "tu@email.com",
        "CONFLUENCE_API_TOKEN": "tu-api-token"
      }
    }
  }
}
```

> Generá tu API token en: [id.atlassian.com/manage-api-tokens](https://id.atlassian.com/manage-api-tokens)

#### MCP opcional: `bitbucket`

Solo necesario si el código fuente vive en Bitbucket Server/Data Center (no en local).

```json
{
  "mcpServers": {
    "bitbucket": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "bitbucket-server-mcp"],
      "env": {
        "BITBUCKET_URL": "https://bitbucket.tuempresa.com",
        "BITBUCKET_TOKEN": "tu-personal-access-token",
        "BITBUCKET_DEFAULT_PROJECT": "MI_PROYECTO"
      }
    }
  }
}
```

#### Ejemplo: `.mcp.json` completo

```json
{
  "mcpServers": {
    "mcp-atlassian": {
      "type": "stdio",
      "command": "uvx",
      "args": ["mcp-atlassian"],
      "env": {
        "JIRA_URL": "https://tuempresa.atlassian.net/",
        "JIRA_USERNAME": "tu@email.com",
        "JIRA_API_TOKEN": "tu-api-token",
        "CONFLUENCE_URL": "https://tuempresa.atlassian.net/",
        "CONFLUENCE_USERNAME": "tu@email.com",
        "CONFLUENCE_API_TOKEN": "tu-api-token"
      }
    },
    "bitbucket": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "bitbucket-server-mcp"],
      "env": {
        "BITBUCKET_URL": "https://bitbucket.tuempresa.com",
        "BITBUCKET_TOKEN": "tu-personal-access-token",
        "BITBUCKET_DEFAULT_PROJECT": "MI_PROYECTO"
      }
    }
  }
}
```

#### Verificar la configuración

```bash
# Listar los MCP servers activos en Claude Code
claude mcp list

# Diagnosticar problemas
claude doctor
```

---

## Uso

**Claude Code**
```
/jira-analyst-skill PROJ-123
```

**Cursor**
```
Analiza el ticket PROJ-123
```