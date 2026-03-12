# Agent Skills

Colección de skills para Claude Code y Cursor.

## Skills disponibles

| Skill | Descripción |
|-------|-------------|
| [jira-analyst-skill](skills/jira-analyst-skill/SKILL.md) | Analiza tickets de Jira y produce propuestas funcionales o técnicas |

## Instalación

```bash
npx skills add facuperezbri/jira-analyst-skill
```

El instalador pregunta en qué editores instalar (Claude Code, Cursor, o ambos).

## Configuración MCP

Las skills que usan herramientas externas requieren configurar un servidor MCP.
Ejecutar el script de setup después de instalar las skills:

```bash
# Mac / Linux
chmod +x setup/mcp-setup.sh && ./setup/mcp-setup.sh

# Windows
.\setup\mcp-setup.ps1
```

Los archivos generados (`.mcp.json`, `.cursor/mcp.json`) están en `.gitignore`.

## Agregar nuevas skills

Crear una carpeta en `skills/` con el nombre de la skill y un `SKILL.md`:

```
skills/
└── nombre-skill/
    └── SKILL.md
```

El `SKILL.md` debe seguir el formato [agentskills.io](https://agentskills.io/specification).
