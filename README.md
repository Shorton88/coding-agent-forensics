# coding-agent-forensics

**Offline, single-file forensic viewer for AI coding-agent transcripts.**

Drop in sessions from Claude Code, Codex, Copilot, Cursor, Gemini, Cline, Goose, Aider, OpenCode, Hermes, or DeepSeek Harness and get a searchable timeline, reconstructed file diffs, and rule-based flags for credentials, exfiltration, and git history rewrites. Nothing leaves the browser.

---

## Why

Coding agents now read files, run shell commands, edit source, and talk to the network on a developer's behalf. Every one of them writes a complete record of what it did to local disk. That record is an underused evidence source: it is often the only place showing what an agent was *told*, what it *saw*, and what it *ran* — including the injected context the developer never typed and may never have seen.

The existing tooling for these files is aimed at developers debugging their own sessions. This is aimed at whoever has to review someone else's, after the fact, on a machine that may not be allowed to talk to the internet.

## What it is

One HTML file. About 210 KB. No build step, no dependencies, no CDN, no network calls of any kind.

Open it from `file://`, drag transcripts onto the window, and read. Everything is parsed in the browser and nothing is uploaded — which is the point, because these files routinely contain credentials, proprietary source, and customer data.

```
git clone https://github.com/<you>/coding-agent-forensics
open coding-agent-forensics/agent-session-review.html
```

Or download the single file from Releases and open it. That is the whole install.

## Screenshots

<!-- Replace with real captures before publishing. -->
| Timeline and risk rail | Findings across sessions | File edit reconstruction |
| --- | --- | --- |
| `docs/timeline.png` | `docs/findings.png` |

## Supported agents

| Agent | Storage | Notes |
| --- | --- | --- |
| Claude Code | JSONL per session | subagent sidechains preserved |
| Codex CLI | JSONL per session | records its originating timezone |
| Cursor | SQLite key-value | composer sessions plus the flat prompt history |
| Gemini CLI | JSONL per session | rewound turns kept and tagged |
| GitHub Copilot Chat | JSON / JSONL | VS Code storage, or the repo under Visual Studio |
| Cline | JSON per task | Roo Code and Kilo Code share the format |
| Goose | JSONL per session | `toolRequest` / `toolResponse` parts |
| Aider | Markdown | **lives in the repository, not the profile** |
| OpenCode | SQLite | needs the `-wal` sidecar |
| Hermes Agent | SQLite | one database shared by all processes |
| DeepSeek Harness (dsh) | JSONL event stream, or SQLite | zstd-compressed; decompress first |

Unrecognised JSONL and SQLite stores fall back to a generic adapter that matches on column and field names, so an agent not listed here will usually still render something.

## Features

**Timeline.** Every entry classified by speaker (user / assistant / tool / system) and kind (message, reasoning, tool call, tool result, file edit, injected context, metadata). Collapsed by default with a two-line preview; expand for full text, arguments, output, and raw JSON.

**Injected context is separated from prompts.** Machine-generated blocks that wear a user role — environment context, plugin lists, memory injections — get their own entry kind, so the prompt count reflects only what a human actually typed.

**File edits reconstructed as diffs.** `Edit`, `MultiEdit`, `Write`, `str_replace`, `apply_patch` (including inside shell heredocs), `write_file` and friends are normalised to before/after pairs and rendered as real line diffs with `+N -N` stats. `File edit` is its own filter, so you can reduce a session to just the mutations it made to disk.

**Detection rules.** Nineteen built-in heuristics at three severities: credential shapes, credential file access, sandbox and approval bypass, outbound network from a shell, obfuscated execution, destructive commands, privilege changes, build and CI surface changes, prompt-injection-shaped text in tool output, and a seven-rule git suite covering history rewrites, hook and transport weakening, embedded credentials in remote URLs, working-tree discards, identity overrides, and remote retargeting.

**Rules are editable.** The Rules sheet exposes the whole set as JSON — id, label, severity, regex pattern, flags, and an optional `kinds` array. Load a file, edit inline, apply, download. Applying re-scores every loaded session immediately. Bad JSON, an unknown severity, an invalid regex, or a duplicate id each produce a specific error and leave the active rules untouched.

```json
[
  {
    "id": "crown_jewels",
    "label": "Crown jewel path touched",
    "severity": "high",
    "pattern": "services/billing/|infra/terraform/prod",
    "flags": "i",
    "kinds": ["tool_call", "edit", "message"]
  }
]
```

**Cross-session correlation.** Load a whole collection and get every finding in one table — severity, flag, agent, session, UTC time, tool, excerpt. Filter, then click through to the exact entry in its session. CSV export covers the whole set.

**Two-tier filtering.** Scope by speaker and kind with live counts, then narrow by search text, flag, or images, with `grep -C` style context (±0/2/5/10/all) and clickable gap markers for what is hidden.

**Timezones done properly.** Local, UTC, or — where the transcript records it — the zone the session was actually recorded in. That last one matters: "local" is *your* analysis machine, not the developer's. Hover any timestamp for all three renderings. Exports are always UTC regardless of the display setting.

**Playback.** Replay a session paced by the real gaps between timestamps, at 1x through 600x. Watching an agent burst through thirty tool calls in four seconds and then sit idle for seven minutes tells you something a static list does not.

**Inline images.** Screenshots pasted into a session are extracted and rendered, both from known structural locations and by a brute-force scan for base64 image data anywhere in the record. SVG attachments are deliberately not rendered.

**Collection guide built in.** A sixteen-section reference covering storage paths per agent on Windows, macOS and Linux, a copyable glob list, EDR live-response notes, and a two-phase Velociraptor playbook with a packaged custom artifact.

**Themes.** Neon, Galaxy, Graphite, Light, Dark, and an indulgent animated Nature theme with generated ambient sound, for when the transcript is the only scenery you have had all week. Role and severity hues stay constant across all of them, because colour carries meaning here.

## Collecting the files

Full detail is in the app's Collection guide. The three things that cause most failed collections:

1. **Collect the `-wal` sidecar with any SQLite store.** Four of the eleven agents use SQLite and all of them use write-ahead logging. In testing, a freshly written Hermes database was 4 KB on disk with 111 KB of conversation still in the WAL — collecting `state.db` alone would have recovered nothing.
2. **Two agents do not live in the user profile.** Aider writes into the working tree and Copilot under Visual Studio writes into `.vs\` in the solution directory. A sweep of home directories misses both, and because those files sit in the repo they can be committed and pushed.
3. **Gemini deletes its own history** on a retention policy by count and age. It needs scheduled collection rather than on-demand acquisition.

Do not collect `auth.json`, `.credentials.json`, or config files holding API keys. They add nothing to an investigation and expand what you are now holding.

## Evidentiary caveats

Read these before putting findings in a report.

- **Transcripts are unsigned and user-writable.** They are evidence of what the tool recorded, not proof of what a person did. Treat them as you would application logs on a workstation the subject controlled.
- **Flags are heuristics, not verdicts.** Every one needs a human read of the surrounding turn. A `curl | bash` in a tool call may be the agent following a legitimate install doc.
- **A discard destroys the record of what came before it.** When you see `git restore` or `reset --hard`, the diffs you read earlier in that session may no longer match disk.
- **Sessions can fork.** Gemini rewinds, Hermes and DeepSeek Harness fork, and a log that appears to start mid-task may be a branch whose earlier turns live elsewhere. The viewer surfaces the lineage where the format records it.
- **What the model saw is not always what was stored.** Hermes records an `api_content` sidecar when the wire bytes differ from the displayed content; that mismatch is flagged, and it is where ephemeral injections show up.

## Known limitations

- **zstd-compressed logs cannot be read in-browser.** DeepSeek Harness compresses its session logs. The viewer detects the magic bytes and tells you to run `zstd -d` rather than failing obscurely. Shipping a decompressor would mean bundling a library, and an evidence tool should not fetch or execute third-party code at runtime.
- **Aider edits do not appear under the File edit filter.** Its transcript is markdown with search-and-replace blocks inside model responses, not structured tool calls, so there is nothing to normalise.
- **No persistence between loads.** Browser storage is unreliable from `file://` and silently failing state is worse than none. Themes and rules reset on reload; rules can be downloaded and re-loaded.
- **Adapters for some agents were built from documentation and fixtures rather than captured real-world files.** Where a store opens but yields nothing, the status line reports the tables and columns it actually found, so a bug report can be specific.

## Contributing

Adapters are the most useful contribution. Every one normalises to a single entry shape, so adding a format means writing one function and a detection clause — the renderer, filters, rules, diffing, and correlation all come for free.

A sample transcript with anything sensitive removed is just as useful as code. If a store opens but produces nothing, paste the status-line output; it names the tables and columns found.

## Related work

[**numbat**](https://github.com/perplexityai/numbat) (Perplexity, Apache-2.0) is the closest tool in this space and solves an adjacent problem: an endpoint agent with hooks, a CEL rule engine, on-device detection, optional pre-action blocking, and forensic reconstruction across 15+ agents. If you want fleet-wide visibility and prevention, use numbat. This project is the offline reader you open afterwards, on a collection you already have, with no daemon to deploy.

There is also a healthy set of developer-facing viewers for reading your own sessions, including [claude-code-log](https://github.com/daaain/claude-code-log) and [claude-code-transcripts](https://github.com/simonw/claude-code-transcripts).

## Credit

The original inspiration for this project was **[Simon Willison](https://simonwillison.net/)**'s `codex-timeline.html` — a single-file, drop-a-transcript-in, read-it-in-your-browser tool from his [`simonw/tools`](https://github.com/simonw/tools) collection. The core idea here is his: that a transcript viewer should be one file you can open from disk with nothing installed and nothing uploaded. This project takes that shape and points it at a different job — multiple agents, evidence handling, and detection — but the shape is the good part, and it was his first.

No code from that project is used here; everything was written from scratch, including the read-only SQLite reader.

## License

MIT. See [LICENSE](LICENSE).
