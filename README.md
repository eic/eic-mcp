# eic-mcp

Launcher for the EIC [Model Context Protocol](https://modelcontextprotocol.io)
servers inside [eic-shell](https://github.com/eic/eic-shell).

`eic-mcp` runs the EIC MCP servers (`uproot-mcp-server`, `xrootd-mcp-server`,
`rucio-eic-mcp`, `zenodo-mcp-server`, …) as local streamable-HTTP services on
`127.0.0.1:910x/mcp`, so an LLM client running anywhere on your machine —
opencode, GitHub Copilot, Cursor, Claude Code, Gemini CLI, Codex — can drive
EIC tools.

It is **service management only**: it starts each server the eic_xl image
ships (`/opt/local/bin`) in its own **native** streamable-HTTP mode
(`--transport http` for the Python servers, `MCP_TRANSPORT=http` for the Node
one), under a small restart supervisor. There is no bridge process and
nothing is cloned, built, or fetched at run time. If a server command is
missing, the image is too old — upgrade it with `./eic-shell --upgrade`.

## Usage

```bash
git clone https://github.com/eic/eic-mcp ~/eic-mcp
export PATH="$HOME/eic-mcp/bin:$PATH"    # add to your shell profile
```

Then, inside eic-shell (or from the host — see below):

```bash
eic-mcp up                 # start the enabled servers (http://127.0.0.1:910x/mcp)
eic-mcp status             # which are listening
eic-mcp config opencode    # print client config → redirect into your client
eic-mcp logs uproot        # tail a server log
eic-mcp down               # stop them
```

Connect any MCP client:

```bash
eic-mcp config opencode > opencode.jsonc        # opencode
eic-mcp config claude   > .mcp.json             # Claude Code / Desktop
eic-mcp config copilot  > .vscode/mcp.json      # VS Code / Copilot
eic-mcp config cursor   > .cursor/mcp.json      # Cursor
eic-mcp config gemini   > .gemini/settings.json # Gemini CLI
eic-mcp config codex   >> ~/.codex/config.toml  # Codex (TOML, appended)
```

## Choosing servers

Default enabled set is `uproot xrootd rucio`. Override per invocation:

```bash
EIC_MCP_SERVERS="uproot xrootd rucio zenodo" eic-mcp up
```

## Adding servers

New / future EIC servers are added **without editing the script** — drop a file
in `~/.config/eic-mcp/servers.d/<name>.conf` that calls `register`:

```bash
# register NAME PORT COMMAND REPO HTTP_TEMPLATE [ENV_HOOK]
register indico 9105 indico-mcp-server https://github.com/cohm/indico-mcp \
  '%c --transport http --host %h --port %p'
```

`COMMAND` is the installed server command (on `PATH` inside the image);
`REPO` is the upstream project URL (informational, shown by `list`);
`HTTP_TEMPLATE` says how to run the command as a native streamable-HTTP
service — `%c` expands to the resolved command, `%h` to the bind host, `%p`
to the port. An **empty** template marks the server stdio-only: it stays in
the registry but `up` skips it and `config` never emits its URL (that is how
`zenodo` is registered until it gains an HTTP transport). `ENV_HOOK` is an
optional shell function that echoes `export …` lines for auth/config.

### Testing an unreleased server

`eic-mcp` never builds anything, but you can point it at a local build:

```bash
EIC_MCP_EXTRA_PATH=/path/to/my-server/bin eic-mcp up
```

The path is prepended to `PATH` *inside* the container, so `command -v` finds
your build before (or instead of) the image-shipped one.

## Serving clients on other machines

By default everything binds `127.0.0.1`. `EIC_MCP_HOST` is a **real bind
address**: set it to `0.0.0.0` to serve other machines, and the client
configs then advertise `EIC_MCP_ADVERTISE_HOST` (default: this host's first
address) instead of the unconnectable `0.0.0.0`.

```bash
EIC_MCP_HOST=0.0.0.0 EIC_MCP_ADVERTISE_HOST=mybox.lab.org eic-mcp up
EIC_MCP_HOST=0.0.0.0 EIC_MCP_ADVERTISE_HOST=mybox.lab.org eic-mcp config opencode -
```

**The servers have no authentication** — anything that can reach the 910x
ports gets the full tool surface. Use only on a trusted network, or keep the
loopback default and tunnel: `ssh -L 9101:127.0.0.1:9101 …`.

## From the host

Outside eic-shell, `eic-mcp` finds your `eic_xl` image automatically (or set
`EIC_MCP_SIF`) and execs into the container for you:

```bash
EIC_MCP_SIF=/path/to/eic_xl-nightly.sif eic-mcp up
```

On Linux/WSL, Apptainer shares the host network, so the `127.0.0.1` URLs work
from inside and outside the container alike. On macOS, eic-shell runs via
Docker with no published ports — either run your client inside eic-shell, or
add `-p 127.0.0.1:9101-9104:9101-9104` to the `docker run` line of your
`eic-shell` script (`eic-mcp` itself ships in the image).

## License

MIT — see [LICENSE](LICENSE).
