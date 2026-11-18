# claude-skills

Personal [Claude Code](https://docs.claude.com/en/docs/claude-code) skills.

## Skills

- **scaffold**: starts a new project from an interview (purpose, stack, logging); its `references/` hold the coding standards
- **git-cleanup**: makes a project safe to publish (secrets, personal info, deployment details, AI files) and installs a pre-commit guard

## Install

Clone into the personal skills folder:

```bash
git clone https://github.com/brianforan/claude-skills.git ~/.claude/skills
```

Skills load in new Claude Code sessions. Run one with `/scaffold` or `/git-cleanup`, or just describe the task.

## Stack covered by scaffold

- Backend: Python + FastAPI, or Node + TypeScript + Fastify
- Frontend: React + Vite + TypeScript
- Database: Postgres
- Logging: ECS JSON logs, Elasticsearch + Kibana + Filebeat via Docker Compose

## License

[MIT](LICENSE)
