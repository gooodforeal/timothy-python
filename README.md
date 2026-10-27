# timothy-python

Agent skill (формат [Agent Skills](https://agentskills.io): `SKILL.md` + `references/`) — личный стандарт написания Python-кода. Работает с любым агентом, который поддерживает skills: Claude Code, Codex, Cursor, Gemini CLI, GitHub Copilot, OpenCode, Windsurf и др.

## Установка

Через npx (само определит установленных агентов и предложит выбрать):

```bash
npx skills add gooodforeal/timothy-python
```

Или вручную — склонировать в папку skills нужного агента:

```bash
git clone https://github.com/gooodforeal/timothy-python <папка-агента>/timothy-python
```

| Агент | Папка (глобально) | Папка (в проекте) |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Codex | `~/.codex/skills/` | `.agents/skills/` |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` |
| Gemini CLI | `~/.gemini/skills/` | `.gemini/skills/` |
| GitHub Copilot | `~/.copilot/skills/` | `.github/skills/` |
| OpenCode | `~/.config/opencode/skills/` | `.opencode/skills/` |

Если агент не поддерживает skills — подключите `SKILL.md` как инструкцию вручную (`AGENTS.md`, `.cursorrules`, `GEMINI.md` и т. п.) и положите рядом папку `references/`.

> Пути у агентов меняются — при сомнениях сверяйтесь с документацией вашего агента.
