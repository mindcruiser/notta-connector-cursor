# Notta MCP integration guide

Search Notta meeting records, read transcripts and existing summaries, browse folders, export content, and manage trashed records. A Notta account with access to the relevant workspace is required.

Public setup and usage documentation: [Notta Remote MCP — Integration Guide](https://support.notta.ai/hc/en-us/articles/57137620216219).

## 1. Connection

**Environment:** Production. **Transport:** stateless Streamable HTTP. Every `/mcp` request, including initialization, tool discovery, ping, notifications, and tool calls, requires `Authorization: Bearer <access_token>`; `OPTIONS` preflights are exempt. Requests without a Bearer token receive HTTP 401 with a `WWW-Authenticate` challenge pointing to protected resource metadata.

| Endpoint | URL |
| --- | --- |
| MCP / resource / issuer | `https://mcp.notta.io/mcp` |
| Protected resource metadata | `https://mcp.notta.io/.well-known/oauth-protected-resource/mcp` |
| Authorization server metadata | `https://mcp.notta.io/.well-known/oauth-authorization-server/mcp` |
| Authorization | `https://mcp.notta.io/mcp/oauth/authorize` |
| Token | `https://mcp.notta.io/mcp/oauth/token` |
| Revocation | `https://mcp.notta.io/mcp/oauth/revoke` |

| OAuth setting | Value |
| --- | --- |
| public client ID | `c_j7uksddnma8z61b7` |
| Flow | Authorization Code with PKCE S256 |
| Token / revocation client authentication | `none` or `client_secret_post`; the public client above does not require `client_secret`. |
| Scopes | Controlled by Notta authorization; MCP metadata does not publish scope names. |

Open the authorization endpoint in a browser (GET). It returns a 302 redirect to the Notta sign-in and authorization page, preserving all authorization request parameters.

Send the same actual `redirect_uri` in both authorization and code exchange. It must be an absolute URI without whitespace, embedded credentials, or a fragment; HTTP callbacks are limited to loopback addresses.

1. Open the authorization endpoint with `response_type=code`, `client_id`, `redirect_uri`, `state`, `code_challenge`, and `code_challenge_method=S256`. Validate `state` on callback.
2. Exchange the code using the form below. Retain the returned tokens and refresh them when needed; revoke on disconnect.
3. Initialize the MCP connection, send `notifications/initialized`, then use `tools/list` and `tools/call`. Send `Content-Type: application/json`, `Accept: application/json, text/event-stream`, and the negotiated `MCP-Protocol-Version` on subsequent requests.

Token and revocation endpoints accept POST forms with `Content-Type: application/x-www-form-urlencoded`:

| Operation | Required fields | Optional fields |
| --- | --- | --- |
| Code exchange | `grant_type=authorization_code`, `client_id`, `code`, `code_verifier`, `redirect_uri` | `resource`, `client_secret` |
| Refresh | `grant_type=refresh_token`, `client_id`, `refresh_token` | `resource`, `scope`, `client_secret` |
| Revoke | `client_id`, `token` | `client_secret`; `token_type_hint`: `access_token` or `refresh_token` |

`client_secret` is an optional form field of up to 128 characters. When supplied, it is forwarded unchanged to Notta. The client's registration determines whether it is required and how it is validated.

If supplied, `resource` must equal `https://mcp.notta.io/mcp`. Token responses contain `access_token`, `refresh_token`, `token_type`, `expires_in` (seconds), `workspace_id`, and optional `scope`. Revocation succeeds with HTTP 200 and `{}`.

## 2. Tools

All tools enforce the connected account's workspace and operation permissions. Required arguments are listed first; optional arguments follow “optional”. Outputs below are key fields within `data`.

| Tool | Class | Arguments | Output, permission or effect |
| --- | --- | --- | --- |
| `auth_user_info` | Read | `{}` | Account `uid` and `nickname`. |
| `record_list` | Read | Optional `folder_id`, `cursor`, `page_size` | Visible `record_list`, `page_size`, `next_cursor`. Omit folder for all visible folders; `"0"` selects root. |
| `record_search` | Read | `keyword`; optional `cursor`, `page_size` | Indexed name/transcript matches with `record_id`, `record_name`, `next_cursor`. An unavailable index can yield no matches. |
| `record_status` | Read | `record_id` | Transcription status. |
| `record_view` | Read | `record_id` | Transcript `content`: `text`, `attr.speaker_name`, `attr.start_time`, `attr.stop_time` (seconds). `speaker_name` is the speaker display name, or an empty string when unavailable. Requires completed transcription. |
| `record_access` | Read | `record_ids` | `access_result`: sharing settings for records the user can manage. Omitted records may still be readable. |
| `folder_list` | Read | Optional `folder_id`, `cursor`, `page_size` | `folder_list` with IDs, names and parent IDs; `page_size`, `next_cursor`. Omit parent for visible folders; `"0"` selects root; other IDs select children. |
| `summary_batch` | Read | `record_ids` | `summary_result`: `record_id` and Markdown `result`. Reads existing summaries; empty means none available. |
| `export_status` | Read | `task_id` | Task status; `Success` can include `download_url`, `download_url_expire_time`, `file_name`. |
| `export_status_list` | Read | Optional `page_size`, `cursor`, `status` | User's workspace exports: `task_list` with `task_id`, `record_id`, `status`; `next_cursor`. |
| `record_trash` | Write | `record_id` | Moves the record to trash; requires trash permission. Reversible with restore. |
| `record_restore` | Write | `record_id`; optional `folder_id` | Restores the record; requires restore/destination permissions. Omitted folder or `"0"` restores to root. |
| `export_create` | Write | `record_id`, `file_type`; optional `export_options`, `extra_info` | Creates an export task and returns `task_id`; requires export permission. |
| `record_delete` | **Sensitive write** | `record_id`, `confirm: true` | Permanently deletes an already-trashed record; requires deletion permission. Irreversible. |

**Arguments:** IDs are strings; `folder_id` is decimal. `keyword` is a nonempty string. `record_access.record_ids` must contain 1–10 unique nonempty IDs after empty strings and exact duplicates are removed. `summary_batch.record_ids` accepts 1–10 nonempty input entries, which may contain comma-separated IDs; duplicates count toward this limit. `page_size` is an integer from 1–100, default 20. Pass `next_cursor` unchanged as `cursor` with the same filters. Omit unused optional fields; unknown fields and `null` are rejected. Full input schemas are available through `tools/list`.

**Results:** Successful tools return `{error_code: 0, message: string, data: JSON value}` in both `structuredContent` and JSON-encoded `content[].text`. Trash, restore and delete return operation results without a guaranteed `success` field. Check JSON-RPC `error` and tool `isError`, not HTTP status alone.

### Export arguments

`file_type`: `txt`, `docx`, `pdf`, `srt`, `excel`, or `media` (original audio/video). For `media`, omit `export_options` and `extra_info`.

All `export_options` fields are optional numeric `0`/`1` flags:

| Field | Content included when 1 |
| --- | --- |
| `show_time` | Timestamps |
| `include_transcription` | Transcript |
| `include_notes` | Existing AI summary |
| `show_original` | Original-language text |
| `show_translation` | Existing translation |

Original and translation cannot both be disabled; validation defaults are 1/0. SRT applies these defaults; other omitted settings use Notta defaults. Optional `extra_info` fields: `language` (string, e.g. `en-US`) and `timezone` (IANA time zone, e.g. `Asia/Tokyo`).

## 3. Status and write handling

**Permanent deletion requires explicit user approval for each record before setting `confirm: true`.** Other writes must follow the user's authorized request.

| Operation | Status handling |
| --- | --- |
| Transcription | `NotStart` / `Transcribing`: pending. `Transcribed`: ready to read. `Failed`: report failure. |
| Export | `NotStart` / `Processing`: retain `task_id`; recommended polling interval ≥2 seconds. `Success`: use the returned link and expiry. `Failed` / `Expired` / `NotExist`: stop polling. |

Export-list `status` accepts the export states above except `NotExist`. Lists may be stale; use `export_status` for current status or a refreshed signed link. After a write timeout, check the record or task before retrying. Repeating `export_create` can create duplicate tasks; failed calls are not automatically retried.

## 4. Errors

Tool failures set `isError: true`; `content[].text` may contain `error_code`, `message`, and `data` with `request_id`, `required_scope`, or `retry_after` (seconds). Validation errors may be plain text. OAuth errors use `error` and `error_description`.

| HTTP status | Action |
| --- | --- |
| 400 / 404 / 405 / 413 / 415 | Correct the parameters, route, method, body size or content type. |
| 401 / 403 | Refresh or reconnect, or resolve missing scope/access. |
| 429 | Honor `Retry-After` and back off. |
| 500 / 502 / 503 / 504 | Service failure or timeout; check write outcomes before retrying. |

Common business codes: `100001` authorization, `100002` scope, `100003` / `120001` rate limit, `100013` parameters, `110002` incomplete transcription, `130001`–`130003` record/folder access or existence. Preserve the message and request ID for other errors.

## 5. Limits

Rate limits are enforced by the upstream OpenAPI. The size and timeout defaults below may be overridden by deployment settings.

| Limit | Default |
| --- | --- |
| Request body | MCP: 64 KiB; OAuth form: 16 KiB, at most 16 parameters |
| Single upstream response | 8 MiB; oversized responses fail |
| Upstream timeout | 15 seconds per call; a tool request can make multiple calls |

Respect `Retry-After` or tool `retry_after` when provided. Transcript retrieval has no chunking option.

## 6. Privacy and support

Requested meeting data and export links are returned to the connected client. See the [Privacy Policy](https://www.notta.ai/en/privacy), [Terms of Service](https://www.notta.ai/en/terms), and [Help Center](https://support.notta.ai/hc/en-us). Include the request ID in support reports; omit tokens and private meeting content.
