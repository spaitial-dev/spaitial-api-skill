# Spaitial API Agent Skill

Official Agent Skill for integrating with the Spaitial v1 API.

This skill gives coding agents a compact reference for creating and editing 3D worlds through `api.spaitial.ai`, including authentication, world generation, panorama edits, iterative `pano_...` artifacts, file uploads, polling, downloads, mesh exports, webhooks, idempotency, errors, rate limits, and model discovery.

## Install

Install the latest published skill:

```bash
gh skill install spaitial-dev/spaitial-api-skill
```

Pin a specific version:

```bash
gh skill install spaitial-dev/spaitial-api-skill spaitial-api --pin v1.3.2
```

## Example Prompts

```text
Use the spaitial-api skill to build a TypeScript client for /v1/worlds.
```

```text
Use the spaitial-api skill to add webhook signature verification in Node.
```

```text
Use the spaitial-api skill to upload an image, create a world, poll until complete, and download the .spz.
```

```text
Use the spaitial-api skill to edit an existing world panorama, iterate with a reference image, then create a new world from the final pano_ id.
```

## Skill Layout

```text
skills/
  spaitial-api/
    SKILL.md
    content/
    templates/
    resources/
```

## License

MIT
