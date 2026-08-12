# Tools Berry MCP server

An [MCP](https://modelcontextprotocol.io) server that answers US paycheck and payroll-tax
questions for 2026 with real arithmetic instead of a guess: take-home pay, bonus withholding,
state comparisons, and per-state rate schedules for all 50 states and DC.

It is already hosted and free to use. You do not need to install or run anything.

**Endpoint:** `https://mcp.tools-berry.com` — Streamable HTTP, JSON-RPC 2.0 over POST.
Alias: `https://tools-berry-mcp.edydaherz.workers.dev`.

## Connect

**Claude Code**

```sh
claude mcp add --transport http tools-berry https://mcp.tools-berry.com
```

**Claude.ai / Claude Desktop** — Settings → Connectors → Add custom connector, and paste:

```
https://mcp.tools-berry.com
```

**Cursor** — in `~/.cursor/mcp.json` (or `.cursor/mcp.json` in a project):

```json
{
  "mcpServers": {
    "tools-berry": {
      "url": "https://mcp.tools-berry.com"
    }
  }
}
```

**Anything else** — it is plain JSON-RPC, so curl works:

```sh
curl -s https://mcp.tools-berry.com \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{
        "name":"compute_take_home",
        "arguments":{"state":"ohio","salary":80000}}}'
```

That call returns a net of **$63,294.38** on an $80,000 Ohio salary — the same number the
matching page on the site shows, to the cent.

## Tools

| Tool | Answers | Required arguments |
|---|---|---|
| `compute_take_home` | Full paycheck breakdown for a salary in one state: federal income tax, Social Security, Medicare, state income tax, state payroll programs, net annual/monthly/biweekly, effective rate | `state`, `salary` |
| `compute_bonus_withholding` | What is actually withheld from a bonus: the federal 22% supplemental rate (37% above $1,000,000), the state's bonus treatment, and FICA | `state`, `bonusAmount` |
| `compare_states` | Net pay on the same salary across several states, ranked best-first with each state's gap to the winner | `states`, `salary` |
| `get_state_rates` | One state's 2026 schedule: flat rate or bracket ladder, standard deduction, employee payroll programs, supplemental method, and the statute the figures come from | `state` |

`state` accepts a full name (`Ohio`), a two-letter code (`OH`), or a slug (`ohio`,
`district-of-columbia`). `filingStatus` is optional on every tool and defaults to `single`;
it accepts `single`, `married`, `head_of_household`, and the usual aliases (`mfj`, `mfs`, `hoh`).
Every response carries both readable text and a `structuredContent` object of raw numbers.

## Probing it in a browser

The endpoint answers GETs so you can check it without a client:

| Request | Returns |
|---|---|
| `GET /` | A JSON self-description: server info, protocol version, instructions, tool list |
| `GET /health` | `{"ok":true}` |
| `GET /?rpc=<url-encoded JSON-RPC request>` | Executes a single read-only request and returns the JSON-RPC response |

The `?rpc=` form is a convenience for smoke tests, not part of MCP. It exists because some
networks block `*.workers.dev` or drop POST probes, and a server nobody can reach is
indistinguishable from a server that is down.

```sh
curl -sG https://mcp.tools-berry.com \
  --data-urlencode 'rpc={"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

## Design

- **No forked math.** `tools.js` imports `src/engine/paycheck-engine.js` and
  `src/engine/bonus-tax.js` directly and feeds them the same data files the public pages are
  built from. Nothing is re-derived for the MCP layer; that layer only shapes input, picks a
  link, and formats text. `test/test-mcp-server.js` asserts the server's answers equal the
  engine called directly, to the cent, across all 51 jurisdictions — so the server cannot
  quietly drift away from the site.
- **No dependencies.** The protocol surface needed here is `initialize`, `tools/list`,
  `tools/call`, plus `ping` and the empty `resources/list` / `prompts/list` probes, so it is
  implemented by hand in `mcp.js` rather than pulling in an SDK. `npm install` is a no-op;
  `npm test` runs on plain Node.
- **Stateless and read-only.** No auth, no session, no storage, no bindings, no writes.
  Nothing you send is retained.
- **Rate limit:** 120 requests per minute per IP. Every call is a few microseconds of
  arithmetic, so the ceiling only has to stop a runaway loop.
- **Attribution built in.** Every tool response ends with a source line and a deep link to the
  page on tools-berry.com that shows the same figure.

## The numbers

Tax year **2026**, all 50 states plus the District of Columbia. Covered: federal income tax,
Social Security and Medicare (including the additional 0.9%), state income tax, and state
employee payroll programs such as SDI and PFML. Figures are withholding-style estimates for a
single job with no pre-tax deductions and no credits beyond the standard deduction. Local and
city income taxes are not included. This is not tax advice and it is not a filed return.

`get_state_rates` returns a `source` field naming the statute or department schedule each
state's figures come from, so any number here can be traced back.

**Provenance and refresh.** The engine and the two data files are maintained on
[tools-berry.com](https://tools-berry.com) and mirrored into this repository. The hosted server
is deployed from the site's own repository, so this repo is the readable, reusable copy rather
than the deployment source. When rates change on the site, the mirror is refreshed here.
If you need a guarantee that you are on current figures, call the hosted endpoint.

## Citation

If figures or code from here feed something you publish, please cite **tools-berry.com**.
That request is the whole business model behind giving the engine away.

## Deploy your own

You do not need to — the hosted instance is free and open. But if you want your own copy:

```sh
npx wrangler deploy
```

It is a single Cloudflare Worker with no bindings and no secrets. `wrangler.toml` has the
custom-domain block commented out; uncomment it and point it at a hostname you control, or
just use the `workers.dev` URL wrangler prints.

## License

MIT — see [LICENSE](LICENSE). Copyright 2026 Edmond Daher.
