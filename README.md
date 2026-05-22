# karet-skills

Claude skills that automate common tasks against a running Karet
instance. Each skill is a self-contained directory the user can drop
into `~/.claude/skills/` (or symlink in place during development) and
invoke from a Claude Code session.

## Configuration

All skills authenticate against the Karet web API using:

- `KARET_BASE_URL` -- defaults to `http://localhost:3000`
- `KARET_API_KEY` -- the value of `KARET_API_KEY` from `.env` on the
  running container. Skills send it as `x-api-key`.

The skills assume the user already has a running Karet instance and
data sitting in S3 under their pipeline's `raw/` prefix.
