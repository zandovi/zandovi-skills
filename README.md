# Zandovi skills

Authoring knowledge for designing [Zandovi](https://zandovi.com) templates (coupons, gift cards, invitations, certificates, tickets, social graphics) as an agent skill: the element schema, canvas size presets, coordinate origins, fonts, icons, variables, z-order and every validation code. The skill's validate → preview → create workflow uses the MCP tools (`validate_design`, `preview_design`, `create_template`) or the REST endpoints (`POST /api/v1/designs/validate`, `POST /api/v1/designs/render`, `POST /api/v1/templates`); without either it ends with canvasData JSON that a person saves from the Zandovi editor. `edit_template` changes one element of a saved template without resending the whole design; `list_images`/`upload_image` manage the organization's uploads and `search_stock_images` (Unsplash, Pexels, Pixabay) is available in the Claude.ai and ChatGPT connectors only, not in `npx @zandovi/mcp`.

## `zandovi-template-authoring`

The one skill in this repository. See [`zandovi-template-authoring/SKILL.md`](./zandovi-template-authoring/SKILL.md) for what it covers; the `references/` files underneath are generated from the same schema, font and icon registries the Zandovi API and the [`@zandovi/mcp`](https://www.npmjs.com/package/@zandovi/mcp) server use, so the skill never drifts from what the API actually accepts.

## Installing

### Claude Code

Copy the skill directly into your project:

```bash
mkdir -p .claude/skills
cp -r zandovi-template-authoring /path/to/your/project/.claude/skills/
```

Claude Code picks up any `SKILL.md` under `.claude/skills/` automatically; no restart needed beyond starting a new session.

### Claude.ai (Claude skills)

Download or clone this repository, then upload the `zandovi-template-authoring/` folder as a skill from **Settings → Capabilities → Skills → Upload skill** (zip the folder first if the uploader asks for an archive). Claude.ai only reads `SKILL.md` and the files under `references/`.

### Codex

Copy `zandovi-template-authoring/` into the skills directory Codex reads on your machine (for example `~/.codex/skills/` or your project's own skills folder, depending on your Codex configuration), the same way as the Claude Code instructions above. Codex uses the same `SKILL.md` format.

## Staying up to date

`capabilitiesVersion` (in the Zandovi design schema) advances whenever the element schema, fonts or icons change; the skill's generated reference files always match the version named in its commit history. If you use the MCP server instead of a static skill, call `get_account` — it reports `packStale: true` when your installed `@zandovi/mcp` predates the API's `capabilitiesVersion`.
