# tmux

Configuration for [tmux](https://github.com/tmux/tmux/wiki). Derivative work of
[tools/tmux](https://github.com/fultonj/tools/blob/main/tmux.md).

Copy each file to its destination:

| File | Destination |
|---|---|
| [dot_tmux.conf](dot_tmux.conf) | `~/.tmux.conf` |

Plugins are managed with [TPM](https://github.com/tmux-plugins/tpm); install it
per the [TPM installation](https://github.com/tmux-plugins/tpm#installation)
steps, then press `prefix + I` inside tmux to install the plugins listed in
`dot_tmux.conf`.
