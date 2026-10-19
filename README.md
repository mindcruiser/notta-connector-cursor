# Notta - AI Meeting Notes for Cursor

![Notta](assets/logo.png)

Connect your Notta workspace to Cursor to search meeting records, read transcripts and existing AI summaries, browse folders, and export meeting content.

- **Version:** 1.0.0
- **Publisher:** Notta PTE. LTD.
- **MCP endpoint:** `https://mcp.notta.io/mcp`
- **Authentication:** OAuth Authorization Code with PKCE S256

The plugin connects to Notta's hosted MCP server. No local server, API key, or client secret is required.

## Requirements

- A Cursor installation that supports plugins and remote MCP OAuth authentication.
- A Notta account with access to the relevant workspace and meeting records.
- Completed transcription to retrieve a transcript, and an existing Notta AI summary to retrieve that summary.
- The appropriate Notta permissions for export and record-management actions.

## Local installation

1. Copy the plugin files, including the hidden `.cursor-plugin` directory, into `~/.cursor/plugins/local/notta/`.
2. Confirm that the manifest is at `~/.cursor/plugins/local/notta/.cursor-plugin/plugin.json` and that `mcp.json` is at the plugin root.
3. Restart Cursor or run **Developer: Reload Window**.
4. Open **Customize**, find the Notta plugin, and start the Notta MCP connection.
5. Sign in to Notta and authorize the intended workspace.
6. Ask **Which Notta account is connected?** and confirm the account before accessing meeting content.

For organization-managed installations, local imports may require administrator approval. See [Cursor's local plugin guide](https://cursor.com/docs/plugins#test-plugins-locally).

## Example prompts

- Find my meeting notes about the product launch.
- Show the transcript and existing AI summary for this meeting.
- Export this meeting as a PDF and provide the download link.
- What did the client say about pricing in our last sales call? Cite the relevant meeting excerpts.
- What action items came out of this week's project meetings? Identify the source meetings.

When several meetings match, select the intended record before exporting or modifying it. Cursor can summarize retrieved content in a response; this connector does not create or save new AI summaries in Notta.

## Available tools

| Purpose | Tools |
| --- | --- |
| Identify the connected account | `auth_user_info` |
| Find and read meeting records | `record_list`, `record_search`, `record_status`, `record_view` |
| Read sharing settings | `record_access` |
| Browse folders | `folder_list` |
| Read existing AI summaries | `summary_batch` |
| Create and track exports | `export_create`, `export_status`, `export_status_list` |
| Manage records | `record_trash`, `record_restore`, `record_delete` |

Exports support PDF, Word, TXT, subtitles, Excel, and original media, subject to record access and export permissions. Export creation returns a task; a download link is available after that task succeeds. Preserve the task ID and check its status before retrying an export.

Moving records to trash and restoring them change Notta data. Permanent deletion applies to an already-trashed record, cannot be undone, and requires explicit approval for the specific record.

The connector does not upload recordings, start new transcriptions, or change sharing permissions. `record_access` only reads sharing settings.

## Authentication and data access

`mcp.json` supplies a public OAuth client ID. Users sign in with their own Notta accounts; no client secret is included or required. The flow uses PKCE S256. The same redirect URI must be used for authorization and code exchange. Cursor desktop uses `http://localhost:8787/callback`.

Meeting data and export links requested through the tools are returned to Cursor. Access is limited by the connected Notta account and workspace permissions. Do not include passwords, tokens, signed download links, or private meeting content in public issue reports.

See the [integration reference](INTEGRATION.md) for endpoints, tool arguments, errors, and operation limits, and [Cursor's OAuth documentation](https://cursor.com/docs/mcp#static-oauth-for-remote-servers) for client configuration.

## Documentation and support

- [Notta website](https://www.notta.ai/en)
- [Notta Remote MCP Integration Guide](https://support.notta.ai/hc/en-us/articles/57137620216219)
- [Notta Help Center](https://support.notta.ai/hc/en-us)
- [Privacy Policy](https://www.notta.ai/en/privacy)
- [Terms of Service](https://www.notta.ai/en/terms)

## License and scope

The plugin configuration and documentation in this repository are licensed
under the [MIT License](LICENSE), copyright (c) 2026 Notta PTE. LTD.

The following are excluded from that license:

- **Remote MCP server and hosted services.** The Notta MCP server
  implementation, backend business logic, infrastructure, and other hosted
  services are not distributed in this repository and are not licensed under
  this repository's MIT License. The endpoint and API documentation describe
  how to connect to the service; they do not license its implementation or
  grant access to the service or user data. Service use remains subject to
  Notta's [Terms of Service](https://www.notta.ai/en/terms), authentication,
  and account permissions.
- **Notta brand assets.** Notta's names, trademarks, logos, and other brand
  assets, including [`assets/logo.png`](assets/logo.png), are excluded from
  the MIT License. No trademark or brand-asset rights are granted by that
  license. See the [brand asset notice](assets/NOTICE).
