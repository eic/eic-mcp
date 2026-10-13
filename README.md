# eic-mcp

Runs the EIC [MCP](https://modelcontextprotocol.io) servers that the `eic_xl`
image installs as local HTTP services, for an LLM client on the same machine.

| server | port | tools for |
| --- | --- | --- |
| [`uproot`](https://github.com/eic/uproot-mcp-server) | 9101 | reading ROOT files and trees |
| [`xrootd`](https://github.com/eic/xrootd-mcp-server) | 9102 | browsing and reading the EIC data stores |
| [`rucio`](https://github.com/eic/rucio-eic-mcp-server) | 9103 | querying the data catalogue |
| [`zenodo`](https://github.com/eic/zenodo-mcp-server) | 9104 | stdio only — `up` skips it |

## Install

```bash
git clone https://github.com/eic/eic-mcp ~/eic-mcp
export PATH="$HOME/eic-mcp/bin:$PATH"    # add this to your shell profile
```

## Use

Inside eic-shell, or from the host — see [From the host](#from-the-host):

```bash
eic-mcp up              # start them on http://127.0.0.1:910x/mcp
eic-mcp status          # which are listening, and whose
eic-mcp config claude   # write the client config file
eic-mcp logs uproot     # tail one server's log
eic-mcp down            # stop them
```

A port that another user's server holds is refused: `up` and `config` name
the user.

`config` writes the file the client reads, and never overwrites an existing
one. Add `-` to print instead: `eic-mcp config claude -`.

| client | file |
| --- | --- |
| `opencode` | `opencode.jsonc` |
| `claude` | `.mcp.json` |
| `copilot` | `~/.copilot/mcp-config.json` |
| `vscode` | `.vscode/mcp.json` |
| `cursor` | `.cursor/mcp.json` |
| `gemini` | `.gemini/settings.json` |
| `codex` | `~/.codex/config.toml`, appended |

Project-local files apply to a client started in that directory. Inside
eic-shell the stdio entries run `./eic-shell -- eic-mcp run <name>` through
your launcher, so the same file works for a client on the host and for one
inside. This needs a launcher from eic-shell 2026.10 or later: run
`./eic-shell --upgrade` once if yours is older.

## Shared machines

On a host where other users could reach the ports, write commands instead of
URLs: the client spawns the servers over stdio as you, and they exit with it.
No ports, nothing other users can reach, and every registered server counts,
`zenodo` included.

```bash
eic-mcp config claude --stdio
```

## Which servers

Default: `uproot xrootd rucio`.

```bash
EIC_MCP_SERVERS="uproot xrootd rucio zenodo" eic-mcp up
eic-mcp list    # all registered servers
```

## From the host

Outside eic-shell, `eic-mcp` finds the `eic_xl` image and runs the servers
inside it. On Linux and WSL the `127.0.0.1` URLs work inside and outside:

```bash
eic-mcp up                                            # image located automatically
EIC_MCP_SIF=/path/to/eic_xl-nightly.sif eic-mcp up    # or set one (remembered)
```

On macOS, eic-shell is Docker and its ports are published at startup, so run
`eic-mcp` inside it (macOS bash is 3.2; `eic-mcp` needs 4+):

```bash
DOCKER_OPTIONS="$(./eic-shell -- eic-mcp docker-args)" ./eic-shell
eic-mcp up
```

## Other machines

Everything binds `127.0.0.1`. `EIC_MCP_HOST` sets the bind address; the configs
then advertise `EIC_MCP_ADVERTISE_HOST` instead of the unreachable `0.0.0.0`:

```bash
EIC_MCP_HOST=0.0.0.0 EIC_MCP_ADVERTISE_HOST=mybox.lab.org eic-mcp up
```

The servers have no authentication: anything that reaches the 910x ports can
call every tool. To use them from another machine without exposing the ports,
keep the loopback default and tunnel: `ssh -L 9101:127.0.0.1:9101 …`.

## Adding a server

Register it in `~/.config/eic-mcp/servers.d/<name>.conf` instead of editing
the script.

```bash
# register NAME PORT COMMAND REPO HTTP_TEMPLATE [ENV_HOOK]
register indico 9105 indico-mcp-server https://github.com/cohm/indico-mcp \
  '%c --transport http --host %h --port %p'
```

- `COMMAND` — the server command, on `PATH` inside the image
- `HTTP_TEMPLATE` — how to run it as an HTTP service: `%c` command, `%h` bind
  host, `%p` port
- an empty template marks it stdio-only: shown by `list`, skipped by `up`
  and by HTTP configs
- `ENV_HOOK` — optional function echoing `export …` lines for auth or config

To use a locally built server:

```bash
EIC_MCP_EXTRA_PATH=/path/to/my-server/bin eic-mcp up
```

The path is prepended to `PATH` inside the container.

## License

MIT — see [LICENSE](LICENSE).
