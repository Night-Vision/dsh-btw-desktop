# dsh-btw

Side questions and temporary chat for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (dsh), adapted for DSH Desktop. Neither enters the main session history.

- **`/btw`**: ask one question while the main task keeps running. The answer sits in a temporary panel above the composer. Esc closes it. The `btw` chip next to Send does the same.
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

- Boots on DeepSeek Harness 0.2.1-rc.2. The temp-chat overlay is registered through `slots.inject`; the upstream bare registration fails to boot on this core.
- `/btw` answers and temporary chat are not yet verified end to end on Desktop.

## Requirements

- A default model selected in DSH settings. Without one, `/btw` and temporary chat report "no default model configured".
- The interface text is Chinese only.

## License

MIT
