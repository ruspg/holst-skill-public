# holst-skill

Skill для [opencode](https://opencode.ai) для работы с корпоративным
вайтбордом **Holst** (<HOLST_INSTANCE>) через встроенный MCP desktop-приложения.

Быстрый старт:
1. Установить Holst desktop, залогиниться (SSO)
2. Добавить в opencode.json: `"mcp": {"holst": {"type": "remote", "url": "http://localhost:47641/mcp", "enabled": true}}`
3. Подключить скилл: `"skills": {"paths": ["<path>/holst-skill/skills"]}`
4. Перезапустить opencode, проверить: попроси агента «покажи мои holst-workspace'ы»

Подробности — [AGENTS.md](AGENTS.md) и [skills/holst/SKILL.md](skills/holst/SKILL.md).

> Generic build: corporate host names replaced with placeholders.
