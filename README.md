# dsh-btw

Side questions and temporary chat for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (dsh), adapted for DSH Desktop. Neither enters the main session history.

Fork of [iluluyu/dsh-btw](https://github.com/iluluyu/dsh-btw), the DSH Web plugin, adapted for Desktop. See [Credits](#credits).

## Features

- **`/btw` side question**: ask one question while the main task keeps running. The answer sits in a temporary panel above the composer, and Esc closes it. Open it with the `btw` chip next to Send, with `/btw` in the `/` menu, or by typing `/btw your question` in the composer.
  - It sees the recent conversation as plain text: the last 14 messages, each cut to 1,600 characters. It does not see tool output or file contents.
  - That text and your question go to the model selected in the current chat. If a request fails, the error names the provider and model that were used.
- **Temporary chat**: a separate chat in a tab in the right side window, opened from the entry at the top right of the session or from the Start page. It has no parent context, its working directory is the current project, and it is burned when you close the tab. A "Thinking…" bubble shows between sending a message and the reply.
  - **It can act on your project.** The child agent has shell and file tools in the project folder, with the workspace-write sandbox. Approvals are off, so anything that would need approval is refused automatically. Its instructions tell it to stay read-only unless you ask otherwise, but that is an instruction, not a lock.
- **Follows the DSH language**: English and Chinese, switching live with Settings, General, Language. Any other language falls back to English.

## Install

1. Launch DSH Desktop once so its profile exists, then quit it completely.
2. Run the CLI that ships with the app, so the plugin is installed against the same core the app runs. On macOS:

```sh
"/Applications/DeepSeek Harness.app/Contents/Resources/runtime/cli/bin/dsh" plugin --profile desktop add github:Night-Vision/dsh-btw-desktop
```

3. Reopen DSH Desktop.

On Windows the same folder holds `dsh.cmd` (`resources\runtime\cli\bin\dsh.cmd`) inside the app's install directory.

To install from a local clone, use `add file:/absolute/path/to/plugin`, or `add .` from the plugin directory (relative paths are anchored to the invoking directory). To update a local install, run `remove` first and then `add` again, otherwise the old copy stays.

Uninstall: `dsh plugin --profile desktop remove dsh-btw` (with the app quit).

## Status

| Feature | In DSH Desktop (macOS, by hand) | In a test instance | Not checked |
|---|---|---|---|
| `/btw` side question | Answers from the current conversation using that chat's model | `/` menu entry and panel in English and Chinese | `/` menu entry in Chinese |
| Temporary chat | Side tab opens, question gets the Thinking bubble, reply comes back | Mock model: both entry points, reply with persona and tools, reopening, error for an unknown model, session cleanup on close | A temporary chat that edits files |
| Language follow | | Header button, side tab, Start-page card and `/btw` panel switch live between English and Chinese | |
| Windows | | | Everything. A [test plan](https://github.com/Night-Vision/dsh-btw-desktop/blob/windows-compat/docs/superpowers/plans/2026-10-24-windows-linux-compat.md) is on the `windows-compat` branch. |
| Linux | | | Everything |

Runs on DeepSeek Harness 0.2.1-rc.2, the core the desktop app ships. The host serves its RPC as exact fetch routes under `/api/btw/*`, because that core allows a single RPC interceptor on `/api` (API Gateway's) and a second one crashes the app at startup. The temporary chat is a right-sidebar page type (`sidebarRightTabs`), and every slot goes through `slots.inject`.

## Requirements

- A model: `/btw` and temporary chat use the model selected in the current chat. If the chat has none, they use the default model from DSH settings, and report "no default model configured" when there is neither.
- DSH Desktop 0.2.1-rc.2. Other versions are untested.

## Known limitations

- Answers are plain text; markdown is not rendered.
- Temporary chat replies appear when the turn finishes, not token by token.
- One temporary chat at a time, shared across sessions.
- The plugin adds an agent preset named `btw-temp`, which shows up under Settings, Agent presets.
- If you close the temporary chat tab while it is still opening, its session folder is removed on the next open or when DSH exits.

## Development

- `npm run check` runs a syntax check on both halves.
- `lib/index.js` is the host half (RPC routes, side-question calls, the temporary chat's child agent). `lib/client.js` is the browser half, hand-written with no build step. `cordis.yml` loads the plugin and declares the `btw-temp` preset.
- To test without touching your real profile, point `DSH_HOME` at a scratch folder, install the plugin there with the app's CLI, serve a web profile on a local port, and use a local OpenAI-compatible mock as the model.

## Credits

- [iluluyu/dsh-btw](https://github.com/iluluyu/dsh-btw): the original DSH Web plugin this is forked from (MIT).
- `dsh-incognito`: the pattern for the disposable temporary chat, adapted here to use the current project folder and the session-cleanup rules described above.

## License

MIT
