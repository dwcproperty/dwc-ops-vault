# Quick Commands

## Launch Claude Code on the Linode box

From your local machine (Mac/laptop):

```bash
ssh root@96.126.118.135
cd /root/dwc-coding
claude
```

### One-liner alias (recommended)

Add to `~/.zshrc` (Mac default) or `~/.bashrc` on your local machine:

```bash
alias dwc='ssh -t root@96.126.118.135 "cd /root/dwc-coding && claude"'
```

Then just type `dwc` from any terminal. The `-t` flag forces a TTY so Claude Code runs interactively.

Reload shell after adding: `source ~/.zshrc`

### Cleaner alternative — SSH config

Add to `~/.ssh/config` on your local machine:

```
Host dwc
    HostName 96.126.118.135
    User root
    RequestTTY yes
    RemoteCommand cd /root/dwc-coding && claude
```

Then `ssh dwc` launches Claude Code in the right directory.

---

## Linode box info

- IP: `96.126.118.135`
- Working directory for Claude sessions: `/root/dwc-coding`
- CLAUDE.md context file: `/root/dwc-coding/CLAUDE.md` (auto-loads on session start)

## Health check the webhook

```bash
curl https://webhook.collincountyrent.com/webhook/health
```

## Restart webhook after edits

```bash
pm2 restart webhook && pm2 logs webhook --lines 5
```
