# Hyprnote v1.0.1 Data Format

## High Level Facts

- **Source of truth**: SQLite database (`db.sqlite`) via the local persister.
- **Filesystem files**: Derived/synced from SQLite for external access (not the primary store).
- **No CONTENT_BASE support**: Data path is fixed based on bundle identifier.
- **Data path on macOS**:
  - Production: `~/Library/Application Support/hyprnote/`
  - Staging: `~/Library/Application Support/com.hyprnote.staging/`
  - Dev: `~/Library/Application Support/com.hyprnote.dev/`

### Persistence Strategy

| Persister | Mode | Description |
|-----------|------|-------------|
| Local (SQLite) | `load()` + `startAutoLoad()` | Primary data store. Loaded on startup, auto-reloads on external changes. Manual save on blur events. |
| Filesystem persisters | `startAutoSave()` only | Sessions, chats, humans, orgs, prompts, templates, events, calendars, chat_shortcuts. Store changes trigger filesystem writes. |
| Folder | `startAutoLoad()` only | Syncs folder structure from filesystem to store. |
| Settings | `startAutoPersisting()` | Both load and save. Separate store schema. |

The local persister handles app data persistence to SQLite. Filesystem-based persisters exist to make data accessible/editable outside the app.

## Directory Structure

```
<DATA_DIR>/
├── db.sqlite
├── sessions/
│   └── <session_id>/
│       ├── _meta.json
│       ├── _transcript.json
│       ├── _memo.md
│       ├── _summary.md
│       └── <Template Name>.md
├── chats/
│   └── <chat_group_id>/
│       └── _messages.json
├── humans/
│   └── <human_id>.md
├── organizations/
│   └── <org_id>.md
├── prompts/
│   └── <prompt_id>.md
├── templates.json
├── events.json
├── calendars.json
├── chat_shortcuts.json
└── settings.json
```

Sessions can also be nested in folders:
```
sessions/
├── <folder_name>/
│   └── <session_id>/
│       ├── _meta.json
│       └── ...
└── <session_id>/
    ├── _meta.json
    └── ...
```

## Sessions

### `_meta.json`

```json
{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "user_id": "local",
  "created_at": "2025-01-15T10:30:00.000Z",
  "title": "Team Standup Meeting",
  "event_id": "calendar-event-id",
  "participants": [
    {
      "id": "participant-uuid",
      "user_id": "local",
      "created_at": "2025-01-15T10:30:00.000Z",
      "session_id": "01234567-89ab-cdef-0123-456789abcdef",
      "human_id": "human-uuid",
      "source": "manual"
    }
  ],
  "tags": ["work", "standup"]
}
```

- `event_id`: Optional. Links to a calendar event.
- `participants[].source`: Either `"manual"` or `"calendar"`.
- `tags`: Optional array of tag names.

### `_transcript.json`

```json
{
  "transcripts": [
    {
      "id": "transcript-uuid",
      "user_id": "local",
      "created_at": "2025-01-15T10:30:00.000Z",
      "session_id": "01234567-89ab-cdef-0123-456789abcdef",
      "started_at": 0,
      "ended_at": 3600000,
      "words": [
        {
          "id": "word-uuid",
          "transcript_id": "transcript-uuid",
          "text": "Hello",
          "start_ms": 0,
          "end_ms": 500,
          "confidence": 0.95,
          "speaker": 0
        }
      ],
      "speaker_hints": [
        {
          "id": "hint-uuid",
          "transcript_id": "transcript-uuid",
          "speaker": 0,
          "human_id": "human-uuid"
        }
      ]
    }
  ]
}
```

- `started_at`, `ended_at`: Milliseconds relative to session start.
- `words[].speaker`: Integer speaker index (0, 1, 2...).
- `speaker_hints`: Maps speaker indices to human IDs.

### `_memo.md`

User's raw notes during the session. Markdown with YAML frontmatter:

```markdown
---
id: "01234567-89ab-cdef-0123-456789abcdef"
session_id: "01234567-89ab-cdef-0123-456789abcdef"
type: "memo"
---

User's notes here...
```

### `_summary.md`

AI-generated summary (enhanced note without a template). Markdown with YAML frontmatter:

```markdown
---
id: "note-uuid"
session_id: "01234567-89ab-cdef-0123-456789abcdef"
type: "enhanced_note"
position: 0
title: "Summary"
---

## Key Points

- Point 1
- Point 2
```

### `<Template Name>.md`

Enhanced notes created from templates. Filename is sanitized template title:

```markdown
---
id: "note-uuid"
session_id: "01234567-89ab-cdef-0123-456789abcdef"
type: "enhanced_note"
template_id: "template-uuid"
position: 1
title: "Meeting Notes"
---

## Action Items

- [ ] Task 1
- [ ] Task 2
```

## Chats

### `_messages.json`

```json
{
  "chat_group": {
    "id": "chat-group-uuid",
    "user_id": "local",
    "created_at": "2025-01-15T10:30:00.000Z",
    "title": "Chat about project"
  },
  "messages": [
    {
      "id": "message-uuid",
      "user_id": "local",
      "created_at": "2025-01-15T10:30:00.000Z",
      "chat_group_id": "chat-group-uuid",
      "role": "user",
      "content": "What are the action items?"
    },
    {
      "id": "message-uuid-2",
      "user_id": "local",
      "created_at": "2025-01-15T10:30:05.000Z",
      "chat_group_id": "chat-group-uuid",
      "role": "assistant",
      "content": "Based on the meeting..."
    }
  ]
}
```

- `messages[].role`: Either `"user"` or `"assistant"`.
- Messages sorted by `created_at`.

## Humans

Stored as `humans/<human_id>.md`:

```markdown
---
user_id: "local"
created_at: "2025-01-15T10:30:00.000Z"
name: "John Doe"
emails:
  - "john@example.com"
  - "john.doe@company.com"
org_id: "org-uuid"
job_title: "Software Engineer"
linkedin_username: "johndoe"
---

Personal notes about this contact...
```

- `emails`: Array of email addresses.
- Body contains memo/notes about the person.

## Organizations

Stored as `organizations/<org_id>.md`:

```markdown
---
user_id: "local"
created_at: "2025-01-15T10:30:00.000Z"
name: "Acme Corp"
---

```

- Body is currently unused (empty).

## Prompts

Stored as `prompts/<prompt_id>.md`:

```markdown
---
user_id: "local"
created_at: "2025-01-15T10:30:00.000Z"
task_type: "summarize"
---

You are a helpful assistant that summarizes meeting transcripts...
```

- `task_type`: The type of AI task this prompt is for.
- Body contains the actual prompt content.

## Root JSON Files

### `templates.json`

```json
{
  "template-uuid": {
    "user_id": "local",
    "created_at": "2025-01-15T10:30:00.000Z",
    "title": "Meeting Notes",
    "content": "{\"type\":\"doc\",\"content\":[...]}"
  }
}
```

- `content`: TipTap JSON document structure as string.

### `events.json`

```json
{
  "event-uuid": {
    "user_id": "local",
    "created_at": "2025-01-15T10:30:00.000Z",
    "calendar_id": "calendar-uuid",
    "title": "Team Meeting",
    "start_time": "2025-01-15T14:00:00.000Z",
    "end_time": "2025-01-15T15:00:00.000Z",
    "attendees": "[{\"name\":\"John\",\"email\":\"john@example.com\"}]"
  }
}
```

- `attendees`: JSON string of attendee array.

### `calendars.json`

```json
{
  "calendar-uuid": {
    "user_id": "local",
    "created_at": "2025-01-15T10:30:00.000Z",
    "name": "Work Calendar",
    "platform": "apple",
    "account_id": "account-uuid"
  }
}
```

### `chat_shortcuts.json`

```json
{
  "shortcut-uuid": {
    "user_id": "local",
    "created_at": "2025-01-15T10:30:00.000Z",
    "title": "Summarize",
    "content": "Summarize this meeting"
  }
}
```

### `settings.json`

```json
{
  "general": {
    "autostart": true,
    "save_recordings": false,
    "quit_intercept": true,
    "telemetry_consent": true
  },
  "notification": {
    "event": true,
    "detect": true,
    "respect_dnd": false,
    "ignored_platforms": []
  },
  "language": {
    "ai_language": "en",
    "spoken_languages": ["en"]
  },
  "ai": {
    "current_llm_provider": "openai",
    "current_llm_model": "gpt-4",
    "current_stt_provider": "deepgram",
    "current_stt_model": "nova-2",
    "llm": {
      "openai": {
        "base_url": "https://api.openai.com/v1",
        "api_key": "sk-..."
      }
    },
    "stt": {
      "deepgram": {
        "base_url": "https://api.deepgram.com",
        "api_key": "..."
      }
    }
  }
}
```

## SQLite Database

The `db.sqlite` file is the primary data store. It uses TinyBase's `createCustomSqlitePersister` with JSON mode. The filesystem files described above are secondary exports that sync from the SQLite store when data changes.

## Notes

- All UUIDs are v4 format.
- Timestamps are ISO 8601 format with timezone (typically UTC with `Z` suffix).
- `user_id` is typically `"local"` for local-only data.
- Filename sanitization replaces `<>:"/\|?*` with `_`.
- Folders are derived from the directory structure under `sessions/`.
