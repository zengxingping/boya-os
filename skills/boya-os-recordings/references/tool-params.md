# Boya OS MCP Tool Reference

All nine tools are read-only. `R` = required, `O` = optional.

## 1. list_devices - bound recording devices

No parameters.

Returns each bound device's `deviceId` (device serial), `deviceName`, `sn`, `firmwareVersion`, and `boundAt`. Only bound devices are returned; unbound devices are excluded. This tool does not return recording counts, and recordings cannot be filtered by device.

---

## 2. list_folders - recording folders

No parameters.

Returns each folder's `folderId` (pass it as `folder_id` to `list_recordings`), `name`, `parentId`, `createdAt`, and `recordingCount` (non-deleted recordings directly in that folder). Uncategorized recordings at the root are not listed. Subfolders are not included in `recordingCount`.

---

## 3. list_recordings - recording list

Newest first by recording time. With no filters, returns the latest page.

| Parameter | Required | Notes |
|---|---|---|
| `date_from` | O | Start date `yyyy-MM-dd`, inclusive from 00:00:00, filtered by recording time. Use the same value as `date_to` for a single day. Mutually exclusive with `period`. |
| `date_to` | O | End date `yyyy-MM-dd`, inclusive through 23:59:59. Mutually exclusive with `period`. |
| `period` | O | Relative period: `today` / `yesterday` / `this_week` (Monday to now) / `last_week` (Monday to Sunday) / `this_month` / `last_month`. Mutually exclusive with `date_from` / `date_to`. |
| `min_duration` | O | Minimum duration in seconds. Only recordings with a determinable duration are kept. |
| `max_duration` | O | Maximum duration in seconds; must be >= `min_duration`. |
| `folder_id` | O | Folder ID (`parent_id`). Returns only recordings directly in that folder. |
| `page` | O | Page number, starting at 1. Default 1. |
| `page_size` | O | Items per page, 1-100. Default 20. |

Returns `total` (all matches) and `items[]`. Each item contains `recordingId`, `title`, `folderName` (null at root), `deviceName` (null if unbound), `durationSec`, `language`, `status` (`pending` / `processing` / `success` / `failed`), `hasTranscript` / `hasSummary` / `hasMindmap` / `hasActionItems`, and time fields.

There is no device filter.

---

## 4. search_recordings - keyword search

Literal substring matching, not semantic or fuzzy search. Matches title and file name, transcript text, and AI summary.

| Parameter | Required | Notes |
|---|---|---|
| `query` | R | Keyword or phrase likely to appear verbatim in the title or text. Do not paraphrase in a way that changes the wording. |
| `scope` | O | `all` (default) / `title` (title and file name) / `transcript` / `summary`. |
| `page` | O | Page number, starting at 1. Default 1. |
| `page_size` | O | Items per page, 1-100. Default 20. |

Returns `total` and `items[]`. Each item contains `recordingId`, `matchType` (`title` > `transcript` > `summary`; only the highest-priority match is kept per recording), and a matching `snippet`.

---

## 5. get_recording - recording details

| Parameter | Required | Notes |
|---|---|---|
| `recording_id` | R | Recording ID (also the upload/transcription ID), taken from `recordingId` in a list or search result. |
| `include_audio_url` | O | When true, also returns `audioUrl`, a 10-minute pre-signed playback link. Default false. Refetch after expiry. |

Returns `title`, file name, file type/extension, folder, `deviceSn`, `deviceName`, `durationSec`, `language`, `status`, the four `has*` flags, `recordedAt`, `createdAt`, and optionally `audioUrl`.

Missing or unauthorized recordings return `RESOURCE_NOT_FOUND` (anti-enumeration; the two cases are not distinguished).

---

## 6. get_transcript - transcript

| Parameter | Required | Notes |
|---|---|---|
| `recording_id` | R | Recording ID. |
| `format` | O | `text` (default, readable plain text) / `markdown` (structured) / `json` (raw per-sentence data with speakers, start/end times, and text; most complete and not truncated). |
| `include_speakers` | O | Add speaker labels to `text` / `markdown`. Default false. |
| `include_timestamps` | O | Add `[mm:ss]` timestamps to `text` / `markdown`. Default false. |
| `start_ms` | O | Time-window start in milliseconds, inclusive. Only sentences overlapping the window are returned. |
| `end_ms` | O | Time-window end in milliseconds, inclusive; must be >= `start_ms` when both are provided. |
| `max_chars` | O | Maximum characters for `text` / `markdown`. Content beyond the limit is truncated and `truncated=true`. Omit for no truncation. Ignored for `json`. |
| `offset` | O | Sentence offset, starting at 0. Default 0. |
| `limit` | O | Sentences per page, 1-50. Default 50. Increase `offset` to page through long transcripts. |

Returns `sentenceCount` (this page), `total` (total sentences in the window), `offset`, `limit`, `hasMore`, `language`, `durationSec`.

If transcription has not succeeded, returns a non-success `status` with empty content. This is a normal response, not an error.

---

## 7. get_summary - AI summary

| Parameter | Required | Notes |
|---|---|---|
| `recording_id` | R | Recording ID. |

Returns `title`, `summaryText` (summary body), `mainText` (key points), `keywords`, `mindmap` (mind map source), `version`, and `updatedAt`.

One call returns both the summary and the mind map. Use `get_map` only when the user wants the mind map alone.

Returns the latest existing summary only. It never triggers generation or regeneration; direct the user to the Boya OS app for that.

If there is no summary or generation failed, returns `hasSummary=false` with null content fields. This is a normal response, not an error.

For batch use, first filter recordings by `hasSummary` in `list_recordings` to avoid ineffective calls.

---

## 8. get_map - mind map

| Parameter | Required | Notes |
|---|---|---|
| `recording_id` | R | Recording ID. |

Returns `mindmap` (markmap-compatible Markdown that can be rendered directly) and `updatedAt`.

Returns an existing mind map only; never generates or regenerates one. If there is no mind map, returns `hasMap=false` and an empty `mindmap` string. This is a normal response, not an error.

When the user wants the full summary (body, key points, and mind map together), prefer `get_summary`; the mind map is one of its fields.

---

## 9. get_action_items - action items / to-dos

`items` is one merged list of two sources:

1. Structured to-dos already created in the account (`source=memo`).
2. Action items extracted from recording summaries (`source=main_text` / `mindmap`).

| Parameter | Required | Notes |
|---|---|---|
| `recording_id` | O | Restrict to items derived from one recording and action items extracted from its summary. Omit when the user asks for "my action items" to list everything. |
| `include_completed` | O | Include completed items. Default false. Extracted items are always incomplete and are returned either way. |
| `due_from` | O | Start due date `yyyy-MM-dd`, inclusive from 00:00:00, filtered by `remind_time`. |
| `due_to` | O | End due date `yyyy-MM-dd`, inclusive through 23:59:59, filtered by `remind_time`. |
| `created_from` | O | Start creation date `yyyy-MM-dd`, inclusive from 00:00:00, filtered by `create_time`. |
| `created_to` | O | End creation date `yyyy-MM-dd`, inclusive through 23:59:59, filtered by `create_time`. |

Date filters apply to structured to-dos only; extracted items have no dates and are omitted when any date filter is present.

Returns `total` and `items[]`. Each item contains `source`, `completable` (`true` = a real to-do with an `itemId`, completable or remindable in the app; `false` = an action item written in a summary, not a to-do yet), `itemId`, content, and date fields.

With no action items, returns `items: []` and `total: 0`. This is a normal response, not an error.
