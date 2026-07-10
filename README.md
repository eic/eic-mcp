# eic-mcp

Launcher for the EIC [Model Context Protocol](https://modelcontextprotocol.io)
servers inside [eic-shell](https://github.com/eic/eic-shell).

`eic-mcp` runs the EIC MCP servers (`uproot-mcp-server`, `xrootd-mcp-server`,
`rucio-eic-mcp`, `zenodo-mcp-server`, …) as local streamable-HTTP services on
`127.0.0.1:910x/mcp`, so an LLM client running anywhere on your machine —
opencode, GitHub Copilot, Cursor, Claude Code, Gemini CLI, Codex — can drive
EIC tools.

It is **hybrid**: a server already installed in the eic_xl image
(`/opt/local/bin`) is used as-is; one that is not is bootstrapped automatically
on the first `up` (a one-time clone + build into `~/.eic-mcp/servers`). The
same two commands work either way, and the bootstrap disappears by itself once
the Spack-packaged servers ship in the image. Each stdio server is exposed
over streamable HTTP by [`supergateway`](https://github.com/supercorp-ai/supergateway)
(stateful mode, under a restart supervisor).

## Usage

```bash
git clone https://github.com/eic/eic-mcp
export PATH=$PWD/eic-mcp/bin:$PATH
```

Then, inside eic-shell (or from the host — see below):

```bash
eic-mcp up                 # start the enabled servers (http://127.0.0.1:910x/mcp)
eic-mcp status             # which are listening
eic-mcp config opencode    # print client config → redirect into your client
eic-mcp logs uproot        # tail a server log
eic-mcp down               # stop them
eic-mcp setup              # optional pre-build ('up' does this automatically)
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
# register NAME PORT COMMAND REPO KIND [ENV_HOOK]
register indico 9105 indico-mcp-server https://github.com/cohm/indico-mcp node
```

`COMMAND` is the installed stdio server (preferred when on `PATH`); `REPO` is
the upstream git URL used to bootstrap when it is not installed; `KIND`
(`py` | `node`) selects the bootstrap build recipe; `ENV_HOOK` is an optional
shell function that echoes `export …` lines for auth/config.

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
`eic-shell` script.

## License

MIT — see [LICENSE](LICENSE).
