# Optional Linux user service

Use only after the supervised automatic checkpoint test succeeds. This starts the watcher, not Core or an AI model. The encrypted Core issuer wallet must already be unlocked through its normal authenticated workflow.

From the extracted package directory:

```bash
mkdir -p "$HOME/.config/systemd/user" "$HOME/.config/tru-agent"
chmod 700 "$HOME/.config/tru-agent"
install -m 644 tru-agent-auto@.service "$HOME/.config/systemd/user/"
```

Use the exported AGENT_SLUG, TRU_AGENT_CODE, TRU_AGENT_HOME and TRU_CLI from README. For your agent, create a private service environment file with your actual paths. It contains paths, not provider keys or wallet passwords. Refuse to overwrite an existing file:

```bash
umask 077
python3 - <<'PY'
from pathlib import Path
import os, re
slug = os.environ["AGENT_SLUG"]
if not re.fullmatch(r"[a-z0-9]+(?:-[a-z0-9]+)*", slug):
    raise SystemExit("Invalid agent slug")
home = Path.home()
p = home / '.config/tru-agent' / (slug + '.env')
values = {
    'TRU_AGENT_CODE': os.environ['TRU_AGENT_CODE'],
    'TRU_AGENT_HOME': os.environ['TRU_AGENT_HOME'],
    'TRU_CORE_CLI': os.environ['TRU_CLI'],
}
if any('\n' in v or '"' in v or '\\' in v for v in values.values()):
    raise SystemExit('Configure unusual paths manually with systemd environment quoting.')
with p.open('x') as f:
    for key, value in values.items():
        f.write(f'{key}="{value}"\n')
print(p)
PY

systemctl --user daemon-reload
systemctl --user enable --now "tru-agent-auto@${AGENT_SLUG}.service"
journalctl --user -u "tru-agent-auto@${AGENT_SLUG}.service" -f
```

Other agents use their own instance name, `.env` file, runtime home, and separately authorized policy. Do not run the foreground watcher and service simultaneously. Runtime locking defers overlap but duplicate watchers add unnecessary load.

Stopping the watcher:

```bash
systemctl --user disable --now "tru-agent-auto@${AGENT_SLUG}.service"
```

Also disable the Core policy as documented in README.md to prevent approval of already exported requests. Already approved anchors remain queued. Starting the user service is not a guarantee it runs after logout or boot: that depends on the host's existing user-session/lingering configuration. No lingering setting, Core startup, or wallet unlock is changed by this package.
