# dsh-btw

Side questions and temporary chat for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (dsh), adapted for DSH Desktop. Neither enters the main session history.

Fork of [iluluyu/dsh-btw](https://github.com/iluluyu/dsh-btw), the DSH Web plugin, adapted for Desktop.

- **`/btw`**: ask one question while the main task keeps running. The answer sits in a temporary panel above the composer. Esc closes it. The `btw` chip next to Send does the same, and `/btw` is offered in the `/` menu.
- **Temporary chat**: opens as a tab in the right side window, from the entry at the top right of the session or the Start page. No parent context, working directory is the current project, burned when the tab is closed.

## Install

```sh
dsh plugin --profile desktop add github:Night-Vision/dsh-btw-desktop
dsh plugin --profile desktop add .   # run from the plugin directory; relative paths are anchored to the invoking directory
dsh plugin --profile desktop add file:/absolute/path/to/plugin
```

Restart DSH Desktop. Uninstall: `dsh plugin --profile desktop remove dsh-btw`.

Use the CLI that ships with the app (`/Applications/DeepSeek Harness.app/Contents/Resources/runtime/cli/bin/dsh`), so the plugin is installed against the same core the app runs.

## Status

- Runs on DeepSeek Harness 0.2.1-rc.2, the core the desktop app ships. The host serves its RPC as exact fetch routes under `/api/btw/*`, because that core allows a single RPC interceptor on `/api` (API Gateway's) and a second one crashes the app at startup. The temp-chat tab is a right-sidebar page type (`sidebarRightTabs`), and every slot goes through `slots.inject`.
- Checked on Desktop: the app starts, and `/btw` answers from the current conversation using that chat's model. In a test instance, the `/` menu entry and the panel render in English.
- Tested on macOS only. The CLI path above is the macOS bundle path; Windows and Linux are untested.
- Answers appear as plain text; markdown is not rendered.
- Temporary chat: its persona and read-leaning tools come from the `btw-temp` agent preset that the plugin declares in `cordis.yml`; the preset also shows up under Agent presets in Settings. Checked in a test instance with a local mock model: the side tab from both entry points, send (reply, persona and tools applied), reopening, error message for an unknown model, and closing the tab with session cleanup. Replies appear when the turn finishes, not token by token; a "Thinking…" bubble shows until then. Checked by hand in DSH Desktop on macOS: the side tab opens, the question gets the Thinking bubble, and the reply comes back.
- Language: checked in a test instance by switching DSH between English and Chinese (the two languages this app ships). The header button, side tab, Start-page card and `/btw` panel follow the switch without a reload. The `/` menu entry was not checked in Chinese.

## Requirements

- A model: `/btw` and temporary chat use the model selected in the current chat. If the chat has none, they use the default model from DSH settings, and report "no default model configured" when there is neither.
- The interface follows the DSH language (Settings, General, Language). The plugin includes English and Chinese, the two languages this DSH ships; any other language falls back to English.

## License

MIT
