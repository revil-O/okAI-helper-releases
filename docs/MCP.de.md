# okAI-helper über MCP nutzen

[English](MCP.md) · [Deutsch](MCP.de.md)

MCP (das [Model Context Protocol](https://modelcontextprotocol.io)) lässt Assistenz- und Agentenprogramme Werkzeuge
benutzen. Mit eingeschaltetem MCP-Server nutzt so ein Programm die Sprachmodelle auf deinem Mac als Werkzeuge: Es listet
sie auf, stellt einem eine Frage oder macht aus Texten Vektoren. Alles bleibt auf deinem Mac oder in deinem eigenen Netz.

okAI-helper spricht beide MCP-Generationen unter derselben Adresse: **2025-11-25** (mit Handshake, das nutzen heute die
meisten Programme) und **2026-07-28** (zustandslos, die aktuelle Fassung).

## In fünf Schritten einrichten

1. **Server.** In okAI-helper **Server** öffnen und **Lokaler Server** einschalten. Der Port steht dort (1235 als
   Grundeinstellung). Für andere Geräte **Im Netzwerk erreichbar** einschalten und bei Bedarf die **Erlaubten Geräte**
   eintragen.
2. **MCP.** Auf derselben Seite **MCP-Server** einschalten (als Grundeinstellung aus). Die Seite zeigt dann die
   MCP-Adresse und ein fertiges Beispiel.
3. **Token.** Das **Zugangstoken** kopieren (Zeigen, Kopieren). Jede Anfrage braucht es; behandle es wie ein Passwort.
   **Neues Token** sperrt jedes Programm aus, das noch das alte benutzt.
4. **Dein Programm.** Einen MCP-Server vom Typ **HTTP** (auch *Streamable HTTP*) anlegen:
   - auf diesem Mac: `http://127.0.0.1:1235/mcp`
   - von einem anderen Gerät: `http://<Adresse des Macs>:1235/mcp` (die Server-Seite zeigt sie unter *Im Netzwerk*)
   - Kopfzeile: `Authorization: Bearer <Token>`

   Die genaue Form hängt vom Programm ab, siehe unten.
5. **Prüfen** mit dem offiziellen MCP Inspector (braucht Node.js 22.23.4, 24.22.1, 26.12.1 oder neuer):

   ```bash
   npx @modelcontextprotocol/inspector --cli http://127.0.0.1:1235/mcp --transport http \
     --header "Authorization: Bearer <Token>" --method tools/list
   ```

   Er sollte `list_models`, `ask_model` und `embed` auflisten. `--protocol-era modern` prüft die Generation 2026-07-28.

## Was ein Programm bekommt

| Werkzeug | Was es tut | Felder |
| --- | --- | --- |
| `list_models` | die Modelle der Bibliothek: ID, Art (Chat oder Embedding), Format, Größe, ob geladen, und die passenden Prompt-Profile | keine |
| `ask_model` | fragt ein Chat-Modell und gibt seine Antwort als Text zurück; ein nicht geladenes Modell wird zuerst geladen | `prompt` (Pflicht), `model` (eine ID aus `list_models`; ohne antwortet das geladene Modell), `profile` (braucht `model`), `system`, `max_tokens` |
| `embed` | macht aus Texten Vektoren mit einem Embedding-Modell | `model` (Pflicht), `input` (eine Liste von Texten) |

Die **Prompt-Profile** der App gibt es auch als MCP-*Prompts*: Ein Programm kann eins auswählen und bekommt dessen
Systemprompt (als erste Nachricht) und dessen Beispiele.

Jedes Werkzeug läuft über den eigenen Server der App: Laden bei Bedarf, Profile, die Sampling-Werte des Modells, der
Wächter-Monitor und das **Server-Log** gelten wie für jedes andere Programm. Jede Frage erscheint im Log zweimal: als
`/mcp` vom Gerät des Programms und als `/v1/…` von 127.0.0.1 (das Werkzeug selbst).

## Programme

Von uns getestet: **MCP Inspector** und **Claude Code**. Die anderen folgen der Dokumentation ihrer Hersteller
(verlinkt) und sind hier nicht getestet; ihre Formate ändern sich ab und zu.

### Claude Code

```bash
claude mcp add --transport http okai http://127.0.0.1:1235/mcp \
  --header "Authorization: Bearer <Token>"
```

`--scope user` macht ihn in allen Projekten verfügbar, `--scope project` schreibt eine `.mcp.json` für das Projekt. In
einer `.mcp.json` kann das Token aus einer Umgebungsvariable kommen statt in der Datei zu stehen:

```json
{ "mcpServers": { "okai": { "type": "http", "url": "http://127.0.0.1:1235/mcp",
  "headers": { "Authorization": "Bearer ${OKAI_TOKEN}" } } } }
```

`"type": "http"` ist Pflicht. Quelle: [Claude Code, MCP](https://code.claude.com/docs/en/mcp).

### VS Code (GitHub Copilot)

Die aktuelle Form ist eine `~/.copilot/mcp-config.json` (für alle Projekte) oder eine `.mcp.json` im Workspace:

```json
{ "mcpServers": { "okai": { "type": "http", "url": "http://127.0.0.1:1235/mcp",
  "headers": { "Authorization": "Bearer ${OKAI_TOKEN}" }, "tools": ["*"] } } }
```

Die ältere `.vscode/mcp.json` (mit `servers` und `inputs`) ist abgekündigt und fällt laut Dokumentation im Februar 2027
weg. Quelle: [VS Code, MCP-Konfiguration](https://code.visualstudio.com/docs/agents/reference/mcp-configuration).

### Cursor

`~/.cursor/mcp.json` (alle Projekte) oder `.cursor/mcp.json` (ein Projekt):

```json
{ "mcpServers": { "okai": { "url": "http://127.0.0.1:1235/mcp",
  "headers": { "Authorization": "Bearer ${env:OKAI_TOKEN}" } } } }
```

Quelle: [Cursor, MCP](https://cursor.com/docs/context/mcp).

### Continue

Eine Datei in `.continue/mcpServers/` des Workspace, z. B. `okai.yaml`:

```yaml
name: okAI-helper
version: 0.0.1
schema: v1
mcpServers:
  - name: okAI-helper
    type: streamable-http
    url: http://127.0.0.1:1235/mcp
    requestOptions:
      headers:
        Authorization: "Bearer <Token>"
```

Quelle: [Continue, MCP](https://docs.continue.dev/customize/deep-dives/mcp).

### LM Studio (0.3.17 oder neuer)

Reiter Program → Install → `mcp.json` bearbeiten:

```json
{ "mcpServers": { "okai": { "url": "http://127.0.0.1:1235/mcp",
  "headers": { "Authorization": "Bearer <Token>" } } } }
```

Quelle: [LM Studio, MCP](https://lmstudio.ai/docs/app/mcp).

### Open WebUI (0.6.31 oder neuer)

Als Admin: Settings → Integrations → External Tool Servers → **+ Add Connection**, Typ **MCP (Streamable HTTP)**, die
Adresse, Auth **Bearer** und das Token als *Key*. Open WebUI verbindet sich von seinem Server aus: In Docker lautet die
Adresse `http://host.docker.internal:1235/mcp`. Quelle: [Open WebUI, MCP](https://docs.openwebui.com/features/extensibility/mcp).

### Claude Desktop

Die *eigenen Konnektoren* von Claude Desktop werden aus Anthropics Cloud erreicht: Sie brauchen eine Adresse im
öffentlichen Internet über HTTPS und erreichen deshalb keinen Server auf deinem Mac oder in deinem Netz. Für einen
lokalen HTTP-Server gibt es die Brücke [mcp-remote](https://github.com/geelen/mcp-remote) eines Drittanbieters (nicht
von Anthropic, hier nicht getestet). Quellen:
[eigene Konnektoren](https://support.claude.com/en/articles/11175166-getting-started-with-custom-connectors-using-remote-mcp),
[lokale MCP-Server](https://support.claude.com/en/articles/10949351-getting-started-with-local-mcp-servers-on-claude-desktop).

## Wenn etwas nicht klappt

| Antwort | Warum | Was tun |
| --- | --- | --- |
| 401 | Token fehlt oder ist falsch | Token neu kopieren; nach **Neues Token** braucht jedes Programm das neue |
| 403 | das Gerät steht nicht unter **Erlaubte Geräte**, oder die Anfrage kam von einer Webseite im Browser | Adresse oder Netz des Geräts eintragen; Browser werden immer abgewiesen |
| 404 | **MCP-Server** ist aus, oder die Adresse endet nicht auf `/mcp` | unter Server einschalten; Adresse prüfen |
| keine Verbindung | Server aus, anderer Port, oder **Im Netzwerk erreichbar** ist für ein anderes Gerät aus | Server-Seite prüfen; eine Firewall auf dem Mac kann den Port sperren |
| die erste Antwort dauert | das Modell wird zuerst geladen | `list_models` zeigt, welche Modelle geladen sind |
| `a profile needs a model` | `profile` ohne `model` | beides angeben |

Im Netz spricht der Server einfaches HTTP: nur im eigenen Netz verwenden (oder über ein VPN).
