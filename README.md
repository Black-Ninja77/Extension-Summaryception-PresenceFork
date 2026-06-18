# Summaryception (Presence Fork)

- **[Summaryception](https://github.com/Lodactio/Extension-Summaryception)** (originally by Lodactio) — layered, recursive chat summarization that keeps thousands of turns of history in a few thousand tokens.
- **[Extension-Summaryception](https://github.com/Dogoo9/Extension-Summaryception)** (fork by dogoo9) — adds per-character-card memory banks and a direct KoboldCPP connection on top of the original.
- **[SillyTavern-Presence](https://github.com/leandrojofre/SillyTavern-Presence)** (by leandrojofre) — lets you manually (or automatically) control which characters are "present" in a group chat scene, hiding messages from characters who weren't there.

This fork sits on top of dogoo9's per-character branch and teaches it to read Presence's actual presence data instead of guessing. In a group chat, each character's memory bank now only ever contains summaries of the messages that character was genuinely present for.

## Why this fork exists

The base Summaryception extension is excellent at compressing conversation history into layered summaries. This fork adds a Presence-focused behavior so that, when Presence metadata is available, the extension can better track which turns belong to which character and reduce the impact of raw conversation noise in character-specific memory.

In practice, that means:

- group chats can keep more useful per-character context
- older turns can be summarized without losing the important narrative state
- summaries can be injected back into the prompt using configurable strategies
- the extension can work with multiple backend providers for summarization

## Features

| Area | Capability |
| --- | --- |
| Recursive summarization | Builds layered summaries so older context is compressed instead of being kept verbatim forever |
| Non-destructive memory handling | Uses SillyTavern's hide/unhide flow so summarized content stays readable while being removed from active context |
| Presence-aware filtering | Uses Presence metadata to better associate summarized content with the active character when available |
| Per-character memory scope | Can keep separate memory banks per character card within a chat |
| Multiple model backends | Supports default SillyTavern generation, connection profiles, Ollama, OpenAI-compatible endpoints, and KoboldCPP |
| Prompt injection modes | Lets you inject summaries into the prompt, into the chat, or before the prompt |
| Advanced controls | Includes pause/resume behavior, force summarize, repair helpers, and memory browsing tools |

## Requirements

Before using this extension, make sure you have:

- a working installation of SillyTavern
- the Presence extension installed and enabled if you want Presence-based filtering
- a summarizer model/provider configured (the main API can be used, or a separate cheaper/faster model)
- a browser/network setup that allows your chosen backend to respond correctly

## Installation

1. Download or clone this repository.
1a. Download or clone the **[SillyTavern-Presence](https://github.com/leandrojofre/SillyTavern-Presence)** extension or else the integration will not work.
2. Place the extension folder into your SillyTavern extensions directory.
3. Restart SillyTavern.
4. Open the extension settings panel and enable Summaryception.

If you are using local backends such as Ollama, LM Studio, or KoboldCPP, make sure the relevant endpoint is reachable and any required CORS configuration is set correctly.

## Usage

### Basic workflow

1. Enable the extension.
2. Choose a summarizer backend.
3. Configure how many turns should stay verbatim before summarization begins.
4. Set how many snippets should be summarized together and how deep the recursion should go.
5. Let the extension run while you chat.

### Presence integration

If you want the fork's character-specific behavior:

- install and enable Presence
- turn on the option labeled "Use Presence extension integration"
- enable "Separate memory per character card" to generate independent memory banks for different character cards in the same chat

This helps keep the summary logic aligned with the active character context in group scenes.
Note that this only applies to group chats.

## Configuration guide

### Core settings

- **Enable Summaryception**: turns the extension on or off
- **Pause Summarization**: stops new summarization work while keeping current injection logic active
- **Separate memory per character card**: keeps individual memory banks for active character cards
- **Use Presence extension integration**: enables Presence-based filtering for summarized turns

### Turn settings

- **Verbatim Assistant Turns to Keep**: controls how many recent assistant turns are preserved exactly
- **Turns per Summary Batch**: controls how many older turns are grouped into one summary batch

### Layer settings

- **Max Snippets per Layer**: controls when older snippets are promoted upward
- **Snippets per Promotion**: controls how many snippets merge into the next layer
- **Maximum Layer Depth**: controls how many recursive summarization layers are allowed

### Summarizer prompt settings

You can customize:

- the summarizer system prompt
- the summarizer user prompt template
- the injection template
- injection position and role

The extension includes presets for narrative or game-state style prompting, and you can also supply a custom prompt.

## Supported connection modes

The extension can send summarization requests through several backends:

- **Default**: uses SillyTavern's active generation pipeline
- **Connection Profile**: routes through a saved SillyTavern connection profile
- **Ollama**: local model support with proxy handling
- **OpenAI-compatible**: works with compatible endpoints and cloud services
- **KoboldCPP**: direct endpoint support for compatible setups

## Troubleshooting

### Summaries are not updating

- confirm the extension is enabled
- check that your summarizer backend is reachable
- try using "Force Summarize Now"
- verify that the prompt injection settings are not accidentally disabled

### Local model connections fail

- confirm the endpoint URL is correct
- verify CORS settings if you are using a browser-based local endpoint
- try a direct endpoint or proxy setup depending on your provider

### Presence integration seems inconsistent

- confirm Presence is installed and running
- verify that the chat messages actually include Presence metadata
- ensure the integration toggle is enabled in the settings

## Notes

- This project is a fork and is intended for users who want Summaryception-style memory compression together with Presence-aware character filtering.
- The extension is designed to keep the chat readable while reducing prompt bloat.
- The repository includes settings UI and connection logic needed to support different provider configurations.

## License

This project is licensed under the AGPL-3.0 license.

## Credits

- **Lodactio** — created the original Summaryception and its layered summarization design.
- **dogoo9** — forked it to add per-character-card memory banks and a direct KoboldCPP connection.
- **leandrojofre** — created SillyTavern-Presence, the source of truth for per-message character presence that this fork reads from.

This fork only adds the integration layer between the two; all credit for the underlying summarization engine and presence-tracking system goes to the projects above. If you find this useful, consider starring the upstream repos too.
