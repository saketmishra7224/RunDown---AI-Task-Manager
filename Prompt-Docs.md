# Prompt Documentation

This document describes the AI prompt templates used in the RunDown project and where to find them in the codebase.

## 1. Core Prompt Template (Chat Assistant)

**Location**

- [`routes.chat_routes.chat`](routes/chat_routes.py)

**Description**

Main system-style prompt for the AI assistant. It:

- Describes RunDown and available features.
- Injects either recent calendar events or emails as `Relevant Data`.
- Documents supported commands (`@add`, `@remove`, `@list`, `@check`, `@when`, `@suggest`, `@help`).
- Instructs the model to answer concisely and suggest using commands when appropriate.

This is the primary “prompt template” used whenever the user sends a free-form chat message that is **not** one of the explicit command prefixes.

---

## 2. Command-Specific Prompts

### 2.1 `@add` – Natural-Language Event Creation

**Location**

- [`routes.chat_routes.add_event_command`](routes/chat_routes.py)

**Description**

Prompt asks the model to:

- Extract a concise `title`, normalized `date` (`YYYY-MM-DD HH:MM`), `location`, and `details` from a free-text event description.
- Apply rules for missing years (use current year or next year depending on month).
- Default to “tomorrow at 9am” if no date is specified.
- Return **only** JSON with the specified schema.

Used to turn `@add ...` chat commands into calendar events.

---

### 2.2 `@remove` – Event Removal

**Location**

- [`routes.chat_routes.remove_event_command`](routes/chat_routes.py)

**Description**

Prompt (inside this function) asks the model to:

- Interpret the user’s description of an event to remove.
- Match that description against existing events.
- Return structured guidance about which event best matches (ID/summary) or that none match.

---

### 2.3 `@check` / `@when` – Availability & Free Slots

**Locations**

- [`routes.chat_routes.check_availability_command`](routes/chat_routes.py)
- [`routes.chat_routes.parse_date_with_ai`](routes/chat_routes.py)

**Descriptions**

- `parse_date_with_ai` uses a prompt that:
  - Receives a natural-language date phrase (e.g. “next Monday”, “tomorrow”).
  - Knows “today” via a formatted current date in the prompt.
  - Must return only a date in `$YYYY\text{-}MM\text{-}DD$` format.

- `check_availability_command`:
  - Uses `parse_date_with_ai` for the target date.
  - Then formats the list of free and busy slots into a natural-language message (no extra prompt beyond `parse_date_with_ai`).

`@when` reuses the same logic.

---

### 2.4 `@suggest` – Time Suggestion Prompt

**Location**

- [`routes.chat_routes.suggest_time_command`](routes/chat_routes.py)

**Description**

Prompt asks the model to:

- Extract from the user text:
  - `title` (event title),
  - `target_date` (could be absolute or relative like “tomorrow”),
  - `duration` in minutes (default 60),
  - `preference` (time of day such as “morning”, “afternoon”).
- Return **only** JSON with these fields.

The function then:

- Uses `parse_date_with_ai` on `target_date`.
- Computes candidate free slots from the user’s calendar.
- Returns a formatted Markdown response with the suggested time and also a structured `event_suggestion` object for the frontend follow-up flow.

---

### 2.5 `@help` – Static Help Text

**Location**

- [`routes.chat_routes.show_help_command`](routes/chat_routes.py)

**Description**

No AI call. Returns a static Markdown help message listing all supported commands and examples.

---

## 3. Email & Task Extraction Prompts

### 3.1 Background Email Event Extraction (Scheduler)

**Location**

- [`app.process_emails`](app.py)

**Description**

Prompt asks the model to:

- Read `Email Subject` and `Email Content`.
- Extract:
  - Exact `event_date` (`YYYY-MM-DD HH:MM`), no relative dates.
  - `location`.
  - Short `description`.
- Return JSON:

```json
{
  "event_date": "YYYY-MM-DD HH:MM or \"none\"",
  "location": "location string or \"none\"",
  "description": "brief description"
}
```

Used by the background job to auto-create calendar events from emails.

---

### 3.2 Email → Suggested Task Prompt

**Location**

- [`routes.chat_routes.add_suggestion`](routes/chat_routes.py)

**Description**

Prompt (for each email) instructs the model to:

- Interpret the email subject and body.
- Extract:
  - `task` (summary of what the user should do / event to attend),
  - `event_date` (normalized `YYYY-MM-DD HH:MM` or `none`),
  - `location`,
  - `is_time_sensitive` (boolean).
- If no actionable task exists, return an “FYI” style `task` with `event_date: "none"` and `is_time_sensitive: false`.
- Return **only** JSON with this structure.

Results are turned into UI suggestions and sorted by time-sensitivity.

---

### 3.3 Natural-Language Task → Calendar Event

**Location**

- [`routes.chat_routes.add_task`](routes/chat_routes.py)

**Two prompt paths**:

1. **With explicit `event_date` present**  
   If a structured `event_date` comes from the frontend (e.g., from a suggestion), AI is not used for date extraction. The app only needs the time converted; the prompt is skipped.

2. **AI-based extraction**  
   When no explicit date is available, the function uses a prompt that:

   - Receives the raw text the user typed as `task_desc`.
   - Asks the model to extract:
     - `title` (5–10-word concise task title),
     - `date` (`YYYY-MM-DD HH:MM` or `null`),
     - `location` (or `null`),
     - `details` (or `null`).
   - Includes explicit date-handling rules (current year, next year, default tomorrow 9am).
   - Requires **JSON-only** output with the specified keys.

The backend uses this output to create Google Calendar events and send back formatted deadlines to the frontend.

---

## 4. Date Parsing Helper Prompt

**Location**

- [`routes.chat_routes.parse_date_with_ai`](routes/chat_routes.py)

**Description**

Standalone helper prompt:

- Input: a free-text date phrase and the current date.
- Instructs: “Respond with ONLY a date in `YYYY-MM-DD` format.”
- Implementation post-processes the model text with a regex and `datetime.strptime`.

Used by `check_availability_command` and `suggest_time_command`.

---

## 5. Model Configuration

**Locations**

- [`app`](app.py) – global configuration via `genai.configure(api_key=GOOGLE_API_KEY)`
- [`routes.chat_routes`](routes/chat_routes.py) – `model = genai.GenerativeModel("gemini-1.5-flash")`
- Additional on-demand `GenerativeModel` instances in:
  - [`routes.chat_routes.chat`](routes/chat_routes.py)
  - [`routes.chat_routes.check_availability_command`](routes/chat_routes.py)
  - [`routes.chat_routes.suggest_time_command`](routes/chat_routes.py)

All prompts in this document are designed for Google’s Gemini 1.5 Flash model.

---

## 6. Summary of Prompts Used

| Area                             | Function / Entry Point                                           | Purpose                                                        |
|----------------------------------|------------------------------------------------------------------|----------------------------------------------------------------|
| Main chat assistant              | [`routes.chat_routes.chat`](routes/chat_routes.py)              | General Q&A + pointing users to commands                      |
| `@add` command                   | [`routes.chat_routes.add_event_command`](routes/chat_routes.py) | Turn event description into structured event JSON             |
| `@remove` command                | [`routes.chat_routes.remove_event_command`](routes/chat_routes.py) | Interpret which event to delete                            |
| `@check`, `@when` (date only)   | [`routes.chat_routes.parse_date_with_ai`](routes/chat_routes.py) | Normalize natural-language dates                          |
| `@suggest` command               | [`routes.chat_routes.suggest_time_command`](routes/chat_routes.py) | Extract event meta for time-suggestion flow              |
| Background email processing      | [`app.process_emails`](app.py)                                  | Extract event details from raw emails                         |
| Email → suggestion generation    | [`routes.chat_routes.add_suggestion`](routes/chat_routes.py)    | Turn emails into actionable task suggestions                  |
| Manual task → calendar event     | [`routes.chat_routes.add_task`](routes/chat_routes.py)          | Turn free-text task descriptions into calendar events         |
| Date-only helper                 | [`routes.chat_routes.parse_date_with_ai`](routes/chat_routes.py) | Shared date-normalization helper                           |

This file serves as the central reference (“prompt documentation”) for all AI prompts used in RunDown.