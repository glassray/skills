# Glassray skills

Agent skills for [Glassray](https://glassray.ai), the evaluation layer that makes AI agents
self-improve. A skill is a folder of instructions a coding agent loads on demand; these follow
the open [Agent Skills](https://agentskills.io) format and work in Claude Code, Cursor, Codex,
GitHub Copilot, VS Code, and any other agent that supports it.

[![skills.sh](https://skills.sh/b/glassray/skills)](https://skills.sh/glassray/skills)

## Install

Any agent that supports Agent Skills, via the `skills` CLI:

```bash
npx skills add glassray/skills --skill glassray-otel
```

Claude Code, as a plugin:

```bash
claude plugin marketplace add glassray/skills
claude plugin install glassray-otel@glassray-skills
```

Or hand an agent the single file: https://glassray.ai/SKILL.md

## Skills

### glassray-otel

Send an AI agent's traces to Glassray from whatever OpenTelemetry setup is already in the repo,
and confirm they land. Reads the repo first, routes by what it finds, tags
every trace with `glassray.customer` / `agent` / `flow`, and refuses to call it done until a
trace has arrived. [Read the skill.](skills/glassray-otel/SKILL.md)

**Use when:**

- "Send our traces to Glassray" / "connect Glassray" - with an existing OpenTelemetry setup in
  Python, Node, Ruby, Go, Java or .NET
- Pointing the Vercel AI SDK, Strands Agents, Google ADK, Pydantic AI, OpenLLMetry,
  OpenInference, OpenLIT or an OpenTelemetry Collector at Glassray
- Tagging traces so Glassray can slice them by customer, agent, flow, user and session
- Traces aren't arriving - `401`, `403`, `415`, or an empty Traces view

**Not for** querying an organization's existing traces, flows or deviations - that is the
[Glassray MCP server](https://glassray.ai/docs/mcp-server).

**Docs:** [Send traces with OpenTelemetry](https://glassray.ai/docs/otlp-ingestion)

## Contributing

See [AGENTS.md](AGENTS.md) for the layout and the rules a skill here has to follow. Every push
runs the Agent Skills reference validator and checks every link in every skill.

MIT - see [LICENSE](LICENSE).
