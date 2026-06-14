# Configuration

## MCP server — `CONCLAVE_API_URL`

The MCP server targets `https://conclave.rosvetic.com/api` by default. No configuration is needed for standard use.

---

## Conclave server — `ConclaveBrief`

The deliberation brief that Claude receives when it claims a session is configured server-side. You do not configure this in the plugin.

| Key | Default | Description |
|---|---|---|
| `ToneOfVoice` | (see appsettings) | Instructional framing for Claude's voice throughout the deliberation |
| `GeminiPersona` | (see appsettings) | System prompt template sent to Gemini before the first turn; `{topic}` is interpolated at request time |
| `OutputGuidance` | `"freeform"` | `"freeform"` for open consensus-seeking; `"ranked"` for pick/choose topics |
| `MaxTurns` | `10` | Maximum turns before Claude wraps up. Claude may finish earlier; the MCP hard ceiling is 30 regardless. |
| `AgreementGuidance` | (see appsettings) | Defines what counts as a genuine inline agreement worth recording |

---

## agy

`agy` (Antigravity) is Google's Gemini CLI. The plugin assumes it is installed and accessible as `agy` on your `PATH`.

The MCP calls it as:
```
agy --sandbox -p "<prompt>"                      # health check
agy -p "<prompt>"                                # get conversation ID
agy --conversation <id> -p "<prompt>"            # all subsequent turns
```

The `--conversation <id>` flag keeps Gemini in the same thread across all turns including the persona preamble.

Authentication is handled by `agy` itself — run `agy` interactively once to complete the Google sign-in flow before using the plugin.

---

## Timeouts

The MCP uses a 120-second timeout per `agy` call by default. Long deliberation prompts and slow Gemini responses are the primary cause of timeout.
