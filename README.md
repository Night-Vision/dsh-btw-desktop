# dsh-btw

Side questions and temporary chat for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (dsh), adapted for DSH Desktop. Neither enters the main session history.

- **`/btw`**: ask one question while the main task keeps running. The answer sits in a temporary panel above the composer. Esc closes it. The `btw` chip next to Send does the same, and `/btw` is offered in the `/` menu.
- **Temporary chat**: the entry at the top right of the session. No parent context, working directory is the current project, gone on close.

## Install

```sh
dsh plugin --profile desktop add github:Night-Vision/dsh-btw-desktop
dsh plugin --profile desktop add .   # run from the plugin directory; relative paths are anchored to the invoking directory
dsh plugin --profile desktop add file:/absolute/path/to/plugin
```

Restart DSH Desktop. Uninstall: `dsh plugin --profile desktop remove dsh-btw`.

Use the CLI that ships with the app (`/Applications/DeepSeek Harness.app/Contents/Resources/runtime/cli/bin/dsh`), so the plugin is installed against the same core the app runs.

## Status

- Runs on DeepSeek Harness 0.2.1-rc.2, the core the desktop app ships. The host serves its RPC as exact fetch routes under `/api/btw/*`, because that core allows a single RPC interceptor on `/api` (API Gateway's) and a second one crashes the app at startup. The temp-chat overlay is registered through `slots.inject`.
- Checked on Desktop: the app starts, and `/btw` answers from the current conversation using that chat's model. In a test instance, the `/` menu entry and the panel render in English.
- Not yet verified: Chinese display, and temporary chat replies end to end.

## Requirements

- A model: `/btw` and temporary chat use the model selected in the current chat. If the chat has none, they use the default model from DSH settings, and report "no default model configured" when there is neither.
- The interface follows the DSH language: English and Chinese are included, any other language falls back to English.

## License

MIT
