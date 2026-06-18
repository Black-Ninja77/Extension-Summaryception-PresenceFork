# 🧠 Summaryception — Presence Fork

A compatibility fork that connects two SillyTavern extensions so they work *with* each other instead of past each other:

- **[Summaryception](https://github.com/Lodactio/Extension-Summaryception)** (originally by Lodactio) — layered, recursive chat summarization that keeps thousands of turns of history in a few thousand tokens.
- **[Extension-Summaryception](https://github.com/Dogoo9/Extension-Summaryception)** (fork by dogoo9) — adds per-character-card memory banks and a direct KoboldCPP connection on top of the original.
- **[SillyTavern-Presence](https://github.com/leandrojofre/SillyTavern-Presence)** (by leandrojofre) — lets you manually (or automatically) control which characters are "present" in a group chat scene, hiding messages from characters who weren't there.

This fork sits on top of dogoo9's per-character branch and teaches it to read Presence's actual presence data instead of guessing. In a group chat, each character's memory bank now only ever contains summaries of the messages that character was genuinely present for.

---

## The Problem This Solves

Per-character memory banks are easy to get right in a 1-on-1 chat — there's only one character, so every message belongs to their bank. Group chats are different: every assistant message in the chat is technically "visible" to Summaryception's turn counter, regardless of which character spoke or who else was in the room at the time. Without Presence, the per-character fork has no way to know that Character A wasn't around for the conversation Character B and the player had three scenes ago, so it either lumps everything into one shared bank or attributes history to the wrong character.

Presence already solves the *visibility* half of this problem — it tracks, per message, who was present — but it only acts on that data during live generation (hiding/unhiding messages). It has no concept of long-term compressed memory. This fork bridges the two: Presence supplies the "who was there," Summaryception supplies the "what do they remember about it."

---

## Requirements

- SillyTavern 1.16.0 or newer (same requirement as upstream Summaryception)
- [SillyTavern-Presence](https://github.com/leandrojofre/SillyTavern-Presence) installed and enabled — **optional**, but required for any of the behavior described in this README. Without it, this fork behaves exactly like dogoo9's original.
- An active group chat. Presence integration is inert in 1-on-1 chats (Presence itself disables in solo chats too).

---

## Installation

1. Remove or rename any existing `Extension-Summaryception` folder under `data/default-user/extensions/third-party/`.
2. Copy this fork's files into a folder named `Extension-Summaryception` in that same location: `index.js`, `connectionutil.js`, `manifest.json`, `settings.html`, `style.css`.
3. Make sure SillyTavern-Presence is installed separately — this fork does not bundle or replace it.
4. Restart SillyTavern.
5. Open the Summaryception panel in Extensions and confirm a new **Presence Integration** section appears.

If you're cloning this from your own GitHub fork instead of copying files by hand:

```bash
cd SillyTavern/data/default-user/extensions/third-party/
git clone <your-fork-url> Extension-Summaryception
```

---

## How It Works

### Memory bank keys

Summaryception's memory banks live in chat metadata, keyed by a string. This fork extends the key scheme:

| Situation | Key | Notes |
|---|---|---|
| Solo chat with character card | `character:<id>` | Unchanged from upstream |
| Group chat, Presence resolved a specific character | `character:<id>` | Same key format, but selected via Presence data rather than SillyTavern's "active character," which doesn't reliably exist in groups |
| Group chat, no specific character in context (e.g. manual Force Summarize) | `group:<groupId>` | New fallback. Upstream falls back to a single `character:unknown` bank here, which silently orphans data every time it's used |

### Live generation flow

Three SillyTavern events drive the integration:

1. **`GROUP_MEMBER_DRAFTED`** fires when SillyTavern (with Presence's input) decides which character will generate next, *before* the prompt is assembled. This fork uses that moment to swap the injected summary block to that character's own bank, so the LLM call goes out with the right memory already in context — no guessing involved.
2. **`MESSAGE_RECEIVED`** fires once that character's reply lands in the chat. This fork reads the message's avatar, resolves it to that character's memory bank, and runs the usual turn-counting / summarization logic — but scoped only to turns Presence says that character witnessed.
3. **`GENERATION_STOPPED`** / **`GENERATION_ENDED`** clear the forced character context afterward, so unrelated operations (like a chat-changed refresh) don't accidentally inherit the last-used character's bank.

### Reading Presence's presence data

Presence stores witness data directly on each chat message as `mes.present`, an array. The tricky part — and the source of a real bug during development — is that this array isn't consistently one data type. It can contain:

- **Numeric character indices**, e.g. `[0, 31, 41]` — this is the normal case, copied from the group's member list when a message is sent.
- **Avatar filename strings**, e.g. `"May.png"` — used by Presence's "see last message" short-term memory feature and by the manual present/absent toggle.
- The literal string `"presence_universal_tracker"` — set when a character has the "all-seeing narrator" toggle enabled in the character panel; that character is treated as present for every message regardless of the rest of the array.

This fork's presence check handles all three: it tries a direct string match first, then resolves any numeric entries against the live character list before giving up. Getting this wrong (i.e. only checking for avatar strings) silently breaks turn counting for nearly every message, since the common case is numeric indices.

---

## New Settings

Found in the Summaryception panel under **Presence Integration**:

| Setting | Options | Behavior |
|---|---|---|
| Presence Integration | `Auto` / `Enabled` / `Disabled` | **Auto** (default) turns on automatically when Presence is detected as installed and enabled, and you're in a group chat. **Enabled** forces it on (and shows a warning badge if Presence isn't actually available). **Disabled** restores stock per-character-card behavior, ignoring Presence entirely. |
| Status badge | — | Live indicator under the dropdown showing whether integration is currently active, and why not if it isn't (Presence not installed, Presence disabled, or not in a group chat). |
| Backfill Character Memories | button | See below. Only visible while integration is active. |

---

## Backfilling an Existing Chat

If you're adding this to a chat that already has history, per-character banks start empty — they only fill in going forward, from live generation. To retroactively build them from everything that's already happened, use **Backfill Character Memories**.

For each member of the current group, it walks the *entire* chat history (including messages already hidden/ghosted by an earlier Force Summarize run), filters it down to what that character was present for, and summarizes it into their own bank in normal batch sizes. It deliberately does **not** touch message visibility — whichever bank originally hid those messages keeps ownership of that; backfill only adds snippet coverage to each character's own bank, working entirely in the background of what's already on screen.

A typical first-time workflow on an existing chat looks like:

1. **Force Summarize** to clear any backlog into the shared group bank, so old messages get hidden and your context window goes back to normal size.
2. **Backfill Character Memories** to give every character their own retroactive memory of that same history.

**This is safe to interrupt and re-run.** Backfill tracks `summarizedUpTo` per character exactly the same way live summarization does, and skips any batch already covered. If your summarizer backend rate-limits you partway through (common when backfilling several characters back-to-back triggers a burst of calls), just click the button again — it resumes from where it stopped rather than restarting or duplicating snippets. If you're on a backend with tight rate limits, consider backfilling during a quiet moment, or be ready to click the button two or three times in succession.

---

## Inherited Features

Everything from dogoo9's fork carries over unchanged when Presence integration is off, and stays available alongside it when integration is on: layered recursive summarization (turns → Layer 0 → Layer 1 → …, each layer compressing the one below it), non-destructive ghosting (hidden messages stay readable in the chat UI, never deleted), prompt isolation during summarizer calls, configurable summarizer connection (default/OpenAI-compatible/Ollama/connection profile, plus the KoboldCPP direct option dogoo9 added), and the built-in tools — Layer Stats, Injection Preview, Snippet Browser, Export/Import, Force Summarize, Stop, and Repair. Full documentation for these lives in the upstream READMEs linked at the top of this file; this fork doesn't change how any of them behave outside of group-chat character attribution.

---

## Known Limitations / Troubleshooting

This integration has been tested primarily in one multi-character group-chat setup and is still rough around a few edges:

- **Group-level message hiding is still chat-wide.** Presence and Summaryception both hide messages, but only one "owner" of a given hidden range really exists at a time. Per-character banks intentionally leave `ghostedIndices` empty — they track *what they've summarized*, not *what's hidden* — so don't read an empty ghost list on a character bank as a sign that nothing happened.
- **Character ID resolution depends on the character list being loaded.** If you run Backfill immediately after switching chats, before SillyTavern has fully populated its character list, you may see banks with mismatched or duplicated labels (e.g. several entries all showing the group's name instead of a character's name). If this happens, wait for the chat to finish loading and re-run Backfill.
- **Backfilling many characters back-to-back can trigger API rate limits** on the summarizer backend. See the resumability note above — re-running is the fix, not a sign of corruption.
- **This is a personal/community integration, not an official release of either upstream project.** Back up your chat file (or at least export the memory database from the panel) before running Force Summarize or Backfill on a chat you care about, the same way you'd be cautious with any tool that rewrites chat metadata.

If you hit something that doesn't match this document, the Memory Database export (under the panel's DB tools) is the fastest way to see exactly what state each bank is in — check `presenceForcedCharacterKey`-driven fields like `key`, `characterId`, and `characterAvatar` line up with what you expect before filing an issue.

---

## Slash Commands

Inherited from upstream, unaffected by Presence integration: `/sc-status`, `/sc-preview`, `/sc-db`, `/sc-clear`.

---

## Credits

- **Lodactio** — created the original Summaryception and its layered summarization design.
- **dogoo9** — forked it to add per-character-card memory banks and a direct KoboldCPP connection.
- **leandrojofre** — created SillyTavern-Presence, the source of truth for per-message character presence that this fork reads from.

This fork only adds the integration layer between the two; all credit for the underlying summarization engine and presence-tracking system goes to the projects above. If you find this useful, consider starring the upstream repos too.

---

## License

AGPL-3.0, inherited from both upstream projects. See `LICENSE`.
