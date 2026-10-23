<p align="right"><a href="README.md">English</a> · <a href="README.zh.md">简体中文</a></p>

# dsh-btw

Side questions and temporary chat for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (dsh). Neither enters the main session history.

- **`/btw`**: ask one question while the main task keeps running. The answer sits in a temporary panel above the composer. Esc closes it. The `btw` chip next to Send does the same.
- **Temporary chat**: the entry at the top right of the session. No parent context, working directory is the current project, gone on close.

## Install

```sh
dsh plugin --profile desktop add github:Night-Vision/dsh-btw-desktop
dsh plugin --profile desktop add .   # run from the plugin directory; relative paths are anchored to the invoking directory
dsh plugin --profile desktop add file:/absolute/path/to/plugin
```

Restart DSH Desktop. Uninstall: `dsh plugin --profile desktop remove dsh-btw`.

## License

MIT
