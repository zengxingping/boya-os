---
name: boya-os-recordings
description: "Use when the user wants to view their own Boya OS recordings, find a recording by something said in it, read a transcript, AI summary, mind map, or action items, or check their bound recording devices and folders. Trigger words: my recordings, recording list, what did I record, find a recording, search recordings, transcript, summary, meeting notes, mind map, action items, to-dos, devices, folders."
---

# Boya OS Recordings

Read-only access to the signed-in user's own recordings and derived content: transcripts, AI summaries, mind maps, action items, folders, and bound devices. This skill never records, edits, deletes, generates, or regenerates anything.

All data is scoped to the currently authorized Boya OS account. Never use it to look for another user's data.

## 1. Choose the right tool

| User intent | Tool |
|---|---|
| "What did I record this week / today?" | `list_recordings` |
| "I remember X was said in a recording" | `search_recordings` |
| "What folders do I have, and how many recordings?" | `list_folders` |
| "Which recording devices are bound?" | `list_devices` |
| "Show details / duration / transcription status" | `get_recording` |
| "Give me the transcript / full text" | `get_transcript` |
| "Give me the summary / meeting notes" | `get_summary` |
| "Show me the mind map" | `get_map` (mind map only) or `get_summary` (full summary) |
| "What are my action items / to-dos?" | `get_action_items` |

Browsing versus searching: use `list_recordings` to browse by date, folder, or duration; use `search_recordings` when the user remembers a keyword but not which recording it was.

## 2. Chain through `recording_id`

Every tool except `list_recordings`, `search_recordings`, `list_folders`, and `list_devices` requires a `recording_id`.

```
list_recordings / search_recordings
        -> take recordingId from each returned item
        -> pass it as recording_id to get_recording / get_transcript / get_summary / get_map / get_action_items
```

- `recordingId` is the stable recording/upload identifier. Never use a title as an ID; titles can repeat.
- Never invent a `recording_id`. If the user has not identified the recording, search or list first.

## 3. Page through all results

`list_recordings` and `search_recordings` return `total` (all matches) and the current page in `items`. If fewer items than `total` have been returned and the user needs a complete answer, increase `page` until all matches are read.

`page_size` is 1-100 (default 20). Do not conclude "that is everything" from the first page unless `items` covers `total`.

## 4. Check derived flags before calling downstream tools

`list_recordings` and `get_recording` return flags that tell you whether readable content exists:

| Flag | Meaning | Call only when true |
|---|---|---|
| `hasTranscript` | A readable transcript exists | `get_transcript` |
| `hasSummary` | An AI summary exists | `get_summary` |
| `hasMindmap` | A mind map exists | `get_map` |
| `hasActionItems` | On/off flag only; details are not included | Always call `get_action_items` for details |
| `status` | `pending` / `processing` / `success` / `failed` | Only `success` can have content |

This avoids ineffective calls when a recording is still processing or has no derived content.

## 5. Empty results are normal responses, not errors

| Situation | Response | How to reply |
|---|---|---|
| Not transcribed, or transcription failed | `get_transcript` returns `status != success` and empty content | "This recording does not have a transcript yet." |
| No summary, or summary generation failed | `get_summary` returns `hasSummary=false` and null content fields | "This recording does not have a summary yet." |
| No mind map | `get_map` returns `hasMap=false` and an empty string | "This recording does not have a mind map yet." |
| No action items | `get_action_items` returns `items: []`, `total: 0` | "There are no action items right now." |

If the user wants to generate or regenerate a summary or mind map, direct them to the Boya OS app. This skill is read-only and never triggers generation.

## 6. Action items come from two sources

`get_action_items` returns one merged list. Use `source` to distinguish:

- `source = memo`: a structured to-do already created in the account.
- `source = main_text` / `mindmap`: an action item extracted from a recording summary.

Use `completable` to distinguish capability, and always say which kind you are listing:

| `completable` | Meaning | How to reply |
|---|---|---|
| `true` | A real to-do with an `itemId`; it can be completed or given a reminder in the app | List normally |
| `false` | Only an action item written in a summary; no to-do was created, so it cannot be completed or reminded | Say explicitly: "These are mentioned in the summary; they are not to-dos yet." |

Other rules:

- Do not pass `recording_id` when the user asks for "my action items"; omitting it lists everything.
- By default only open items are returned; `include_completed=true` also returns completed ones.
- Any date filter (`due_from` / `due_to` / `created_from` / `created_to`) returns structured to-dos only, because extracted items have no dates.

## 7. Common call sequences

| User says | Call sequence |
|---|---|
| "What did I record this week?" | `list_recordings(period=this_week, page_size=100)` then page until `total` is reached |
| "Which meetings last week have summaries?" | `list_recordings(period=last_week)` then filter by `hasSummary` and call `get_summary` for each |
| "Find the recording where I talked about the budget" | `search_recordings(query="budget")` then use the returned `recordingId` |
| "What was said in that recording?" | Locate the `recordingId`, then `get_transcript(format=text, include_speakers=true)` |
| "How long is it, and is it transcribed?" | `get_recording(recording_id=...)` |
| "What are my action items?" | `get_action_items()` with no `recording_id` |
| "What were the action items from this meeting?" | `get_action_items(recording_id=...)` |
| "How are my recordings organized?" | `list_folders()` then `list_recordings(folder_id=...)` |
| "Which devices do I have?" | `list_devices()` |

## 8. Tool parameters and field definitions

See `references/tool-params.md` for every parameter, default, returned field, and empty-state behavior.

## 9. Notes

- **Read-only.** Do not attempt any create, update, delete, or generation action. Direct those requests to the Boya OS app.
- **No device filter.** `list_devices` returns bound devices but `list_recordings` cannot filter recordings by device.
- `list_folders.recordingCount` counts only non-deleted recordings directly in that folder; it does not include subfolders or uncategorized recordings.
- Audio playback is off by default. `get_recording(include_audio_url=true)` returns a 10-minute pre-signed playback link; fetch the recording again after it expires.
- Missing or unauthorized recordings return `RESOURCE_NOT_FOUND` without distinguishing the two cases. Do not infer whether a recording exists.
