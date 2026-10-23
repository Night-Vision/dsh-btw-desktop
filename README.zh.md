<p align="right"><a href="README.md">English</a> · <a href="README.zh.md">简体中文</a></p>

# dsh-btw

为 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（dsh）提供旁路提问和临时聊天。两者都不写入主会话历史。

- **`/btw`**：主任务继续跑的时候提一个问题。答案出现在输入框上方的临时面板，Esc 关闭。也可以点发送按钮旁的 `btw` 胶囊。
- **临时聊天**：会话右上角入口。不继承当前上下文，工作目录是当前项目，关闭即焚。

## 安装

```sh
dsh plugin --profile desktop add github:Night-Vision/dsh-btw-desktop
dsh plugin --profile desktop add .   # 插件目录下执行（相对路径相对当前执行位置）
dsh plugin --profile desktop add file:/absolute/path/to/plugin
```

重启 DSH Desktop。卸载：`dsh plugin --profile desktop remove dsh-btw`。

## 许可

MIT
