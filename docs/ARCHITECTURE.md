# Architecture Notes

Personal notes on design decisions and customizations made to this NanoClaw fork.

## Container Runtime Abstraction

All runtime-specific logic is isolated in `src/container-runtime.ts`: the binary
name, mount syntax, status/startup check, and orphan detection. The rest of the
codebase calls only the abstractions (`readonlyMountArgs`, `stopContainer`, etc.)
and has no knowledge of the underlying runtime.

Switching runtimes (e.g. Apple Container ↔ Docker) means changing only three files:
- `src/container-runtime.ts`
- `container/Dockerfile`
- `container/build.sh`

This fork uses Apple Container on macOS. Docker remains the option for Linux or if
Apple Container becomes unavailable.

### .env Shadowing (Apple Container)

The project root is mounted into the container at `/workspace/project` so the agent
can read its `CLAUDE.md` and IPC files. But `.env` lives in that same directory and
contains the API key. To prevent the agent from reading it directly, the entrypoint
overlays `/dev/null` on top of `.env` via `mount --bind` before the agent starts,
making it unreadable from inside the container. Secrets are passed securely via stdin
as JSON instead.

Docker supports file-level mounts (`-v /dev/null:/workspace/project/.env`), so this
could be done from the host side. Apple Container only supports directory mounts
(VirtioFS), so the bind mount must happen inside the container — which requires root.
Main-group containers therefore start as root, perform the bind mount, then drop
privileges via `setpriv` before the agent runs. Non-main containers are started with
`--user` and never run as root.
