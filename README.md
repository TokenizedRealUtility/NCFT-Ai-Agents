# TRU NCFT agents: identity, persistent memory, and approved checkpoint anchoring

This reusable Linux/Bash guide covers issuance, a first reviewed AI evolution, local chat with explicit persistent memory, and an issuer-approved memory checkpoint confirmed on-chain. Each agent uses its own identity, image, manifest, and private runtime directory. No agent-specific Python fork is needed.

**Version scope:** Core 0.07.9 with TRU_AGENT_CHECKPOINTS and
TRU_AGENT_AUTOMATIC_CORE, including the mainnet genesis height fix (genesis lookup at
height 1), plus runtime 0.3.0. This guide covers a new agent from issuance through
manual baseline anchoring and bounded automatic checkpoint anchoring. It does
not install Core or an inference model. Existing agents must keep their existing
runtime home, identity snapshot, policy and history; skip completed setup steps.

The runtime supports Nemotron, OpenAI, Grok, Claude, Ollama and custom
OpenAI-compatible chat services. Model IDs must be supported by your actual
service/account. Core's evolution provider configuration is separate from the
Python chat runtime; selecting a runtime provider does not reconfigure Core.

## Configuration files supplied

- `tru.conf.example`: local RPC defaults and a disabled checkpoint section.
- `ai.env.example`: supported credential names, no real secrets.
- `tru-agent-auto@.service`: optional repeating watcher service.
- `SERVICE.md`: generic service setup, after the foreground test passes.

For an existing node, merge the checkpoint section into its actual working
config. Do not replace the whole file. The example deliberately leaves existing
oracle funding, encrypted wallet and reorganization settings to your deployment.
Provider keys do not grant checkpoint authority. A label or DID is not permission.

From this extracted setup folder, create the private provider environment once:

```bash
umask 077
mkdir -p "$HOME/.config/tru-agent"
chmod 700 "$HOME/.config/tru-agent"
if [ -e "$HOME/.config/tru-agent/ai.env" ]; then
  echo 'Existing ai.env retained; inspect/edit it instead.'
else
  cp ai.env.example "$HOME/.config/tru-agent/ai.env"
  chmod 600 "$HOME/.config/tru-agent/ai.env"
fi
nano "$HOME/.config/tru-agent/ai.env"
```

Uncomment only needed keys and replace their placeholders locally. Load the file
in the shell that starts chat; also load it before starting Core if its evolution
provider needs those credentials. Core does not automatically source this file.

```bash
set -a
source "$HOME/.config/tru-agent/ai.env"
set +a
```

Do not paste secrets into chat, memory, manifests or policy JSON. The watcher
needs no model API key: checkpoint anchoring makes no AI inference call.


Menu numbers and RPC examples are specific to the reviewed builds. Other builds may use different menus. This is a Linux/Bash guide for an already configured mainnet node and wallet; it is not a node installation guide.

## Before starting

- Have your existing Core node running, with its wallet available and ordinary spendable TRU for issuance fees and the token output.
- Back up your wallet using your existing wallet backup procedure.
- Choose a new agent name, unique source NCFT ID, and private workspace directory.
- Prepare a PNG avatar and upload it to an HTTPS location you control. Use a new path for each agent and version.
- Keep the token in its issuing wallet for initial evolution testing. Ownership alone must not be assumed to transfer issuer authority.
- Do not mint again if issuance has already succeeded. Use the existing transaction and token ID.

## 1. Open the creation menu and prepare your entries

In your **already running** Core console, select:

1. **13 — Token Creation**.
2. **4 — NCFT**.

Have the values below ready before entering the wizard; prompts may time out. If your image is not hosted yet, finish step 2 before entering the creation prompts.

| Prompt | What to enter |
|---|---|
| NCFT ID | A new source identity, for example `MY-AGENT-001`. Do not reuse another agent's ID. |
| Name | Your agent's display name. |
| Description | Use the prototype description below, replacing `AGENT_NAME`. |
| Image/multimedia URL | The plain, working public HTTPS URL from step 2. |
| Quantity | `1` |
| ai_engine | Your chosen provider label. A label does not install a model or configure an adapter. |
| style_descriptor | Your desired style, for example `helpful, concise, provenance-aware`. |
| dynamic_morph | `issuer-authorized evolution; persistent runtime in development` |
| update_interval | `manual reviewed milestones` |
| creator_signature | make it up ~> your signature , or even a cryptographic signature. |
| last_evolution | Leave blank; allow Core to manage evolution bookkeeping. |
| Upload image to IPFS? | `n` when using the HTTPS image in this guide. |

Suggested description:

```text
AGENT_NAME is an experimental TRU-native AI-agent identity represented by a quantity-one NCFT. Inference and memory operate off-chain; approved evolution records have provenance anchored on TRU. Persistent-agent runtime is in development.
```

A nickname in `creator_signature` is only a metadata label, not proof of a cryptographic signature.

**Completing the creation prompts submits a real issuance transaction. There is no extra draft-token confirmation after the final IPFS choice in the reviewed flow.**

## 2. Set your paths and prepare a new image location

In a separate terminal, edit these example values before running them. `example.org` is a placeholder, not a working image host.

```bash
export CORE_ROOT="$HOME/TRU-Core"
export TRU_CLI="$CORE_ROOT/build-native/bin/tru-cli"

export AGENT_NAME="My Agent"
export AGENT_SOURCE_ID="MY-AGENT-001"
export AGENT_SLUG="my-agent"
export AGENT_HOME="$HOME/tru-agents/$AGENT_SLUG"

export AGENT_IMAGE_HOST="example.org"
export AGENT_IMAGE_PATH="/agents/$AGENT_SLUG/avatar-v1.png"
export AGENT_IMAGE_URL="https://${AGENT_IMAGE_HOST}${AGENT_IMAGE_PATH}"

# Change this to your actual PNG file.
export AGENT_IMAGE_SOURCE="$HOME/Pictures/my-agent.png"
```

Use the actual Core repository and build directory on your machine. For example, a NEW_TRU installation may use `$HOME/NEW_TRU/build-native/bin/tru-cli`. This guide does not require rebuilding or changing CMake.

Check the executable and wallet:

```bash
test -x "$TRU_CLI" && "$TRU_CLI" raw getinfo '[]'
```

Confirm the reported address is the one you intend to issue from. Change it through the wallet's address-selection screen if necessary. A nullable `chainvalid` after PERF-05 is not itself a failed audit.

### Upload the avatar

Use your existing website's file manager, deployment process, or media hosting. The resulting public URL must serve the actual PNG without a login.

For an **already configured static website on the same Linux host**, the following is an optional example. Edit the document root to match your real web-server configuration. This does not configure a domain, TLS certificate, or web server.

```bash
export WEB_ROOT="/var/www/example.org"

# Run only after setting WEB_ROOT and AGENT_IMAGE_SOURCE correctly.
sudo install -d -m 755 "$WEB_ROOT/agents/$AGENT_SLUG"
sudo install -m 644 "$AGENT_IMAGE_SOURCE" \
  "$WEB_ROOT/agents/$AGENT_SLUG/avatar-v1.png"
```

Use a fresh image path; the optional `install` command replaces a destination file if it already exists. Keep private manifests, memory, wallets, and API keys outside the website directory. Do not apply recursive permission changes to the website.

### Download and verify the published image

Validate your settings first:

```bash
python3 - <<'PY'
import os, re
from urllib.parse import urlsplit
slug = os.environ['AGENT_SLUG']
url = os.environ['AGENT_IMAGE_URL']
p = urlsplit(url)
if not re.fullmatch(r'[a-z0-9]+(?:-[a-z0-9]+)*', slug):
    raise SystemExit('Use a lowercase slug containing letters, digits and hyphens.')
if (p.scheme != 'https' or not p.hostname or p.username or p.password
        or any(c.isspace() for c in url) or any(c in url for c in '[]()')
        or p.hostname == 'example.org'):
    raise SystemExit('Set a real plain HTTPS image URL, without Markdown formatting.')
print('IMAGE_URL:', url)
PY
```

Continue only if that prints your correct URL:

```bash
umask 077
mkdir -p "$AGENT_HOME"
chmod 700 "$AGENT_HOME"

# Use a fresh private directory; stop if this file already exists.
if [ -e "$AGENT_HOME/avatar-v1.png" ]; then
  echo "Image already exists; inspect it before downloading again."
else
  image_tmp=$(mktemp "$AGENT_HOME/avatar-download.XXXXXX")
  if curl --fail --location --proto '=https' --proto-redir '=https' \
      --connect-timeout 15 --max-time 60 \
      --output "$image_tmp" "$AGENT_IMAGE_URL"; then
    mv -n "$image_tmp" "$AGENT_HOME/avatar-v1.png"
  fi
  rm -f "$image_tmp"
fi

python3 - <<'PY'
import hashlib, os
from pathlib import Path
p = Path(os.environ['AGENT_HOME']) / 'avatar-v1.png'
data = p.read_bytes()
if not data.startswith(b'\x89PNG\r\n\x1a\n'):
    raise SystemExit('Not a PNG: check for an HTML/login/error page. Stop here.')
print('PNG_SIGNATURE=OK')
print('IMAGE_SHA256:', hashlib.sha256(data).hexdigest())
PY
```

Open the public URL in a browser as well and confirm the correct image appears. A PNG signature check alone does not validate the entire image. Keep this versioned URL stable after issuance.

**Copy URLs as plain text.** Do not paste Markdown such as `[URL](URL)` into a shell variable or token field.

## 3. Complete issuance and record the result

Return to **13 → 4** and enter the prepared values from step 1, using your verified image URL. Do not repeat issuance if you already completed it.

Save the issuance transaction ID from Core's output. After confirmation, inspect **main menu 10 — Token List** and record:

- Actual short token ID assigned by Core.
- Issuance TXID.
- Issuing address and displayed owner.
- Token type `NCFT` and quantity `1`.

The source identity you typed, such as `MY-AGENT-001`, is **not** the short token ID used by evolution RPCs. Copy the actual ID from the token display.

If you use GNU screen, attach to your existing session by its actual name. Detach with **Ctrl+A, then D**. Do not launch a second Core process against the same data directory.

## 4. Record and verify your specific token

In the terminal where you set the variables, run each prompt and type or paste its value **after the prompt appears**, then press Enter:

```bash
read -r -p 'Paste the issuance TXID: ' AGENT_ISSUANCE_TXID
read -r -p 'Paste the actual short token ID: ' AGENT_TOKEN_ID
read -r -p 'Paste the issuing address: ' AGENT_ISSUER
export AGENT_ISSUANCE_TXID AGENT_TOKEN_ID AGENT_ISSUER
```

Text inside `read -p '...'` is a prompt, not an assigned value. Pressing Enter without entering a value leaves it empty.

Validate and fetch both records. This writes public issuance evidence into your private workspace and stops if the token, owner, quantity, or confirmation does not match.

```bash
python3 - <<'PY'
import json, os, re, subprocess
from pathlib import Path

def rpc(method, params):
    p = subprocess.run([os.environ['TRU_CLI'], 'raw', method,
                        json.dumps(params)], capture_output=True, text=True,
                       check=True, timeout=60)
    r = json.loads(p.stdout)
    if isinstance(r, dict) and r.get('error'):
        raise SystemExit(str(r['error']))
    return r.get('result', r) if isinstance(r, dict) else r

txid = os.environ['AGENT_ISSUANCE_TXID'].strip()
token = os.environ['AGENT_TOKEN_ID'].strip()
issuer = os.environ['AGENT_ISSUER'].strip()
if not re.fullmatch(r'[0-9a-f]{64}', txid):
    raise SystemExit('Expected a 64-character lowercase hexadecimal TXID.')
if not re.fullmatch(r'[0-9a-f]{16}', token):
    raise SystemExit('Expected the actual 16-character lowercase token ID.')
if not issuer:
    raise SystemExit('Issuing address is empty.')
tx = rpc('gettransaction', {'txid': txid})
meta = rpc('gettokenmetadata', {'txid': txid})
if not (tx.get('txState') == 'CONFIRMED' and tx.get('active') is True
        and int(tx.get('confirmations', 0)) >= 1 and tx.get('txid') == txid):
    raise SystemExit('Issuance is not yet confirmed on the active chain. Wait and retry.')
if not (meta.get('tokenID') == token and meta.get('type') == 'NCFT'
        and str(meta.get('amount')) == '1' and meta.get('owner') == issuer):
    raise SystemExit('Metadata does not match the selected quantity-one NCFT and issuer.')
root = Path(os.environ['AGENT_HOME'])
evidence = {'transaction': tx, 'token_metadata': meta}
path = root / 'issuance-evidence.json'
with path.open('x') as f:
    json.dump(evidence, f, indent=2)
    f.write('\n')
print('ISSUANCE_CONFIRMED')
print('TOKEN_ID:', token)
print('EVIDENCE:', path)
PY
```

`gettokenmetadata` expects the **issuance TXID**, not the short token ID. If `issuance-evidence.json` already exists, inspect it instead of deleting it merely to rerun this step. Evidence is a snapshot, not a guarantee of future chain state.

Check evolution history:

```bash
"$TRU_CLI" raw verifytokenevolution \
  "{\"tokenID\":\"$AGENT_TOKEN_ID\",\"require_confirmed\":true}"
```

Before the first evolution, an empty `anchors` list and `no latest TOKEN_EVOLUTION record exists` are expected. Flags such as `history_ok`, `fully_anchored`, and `runtime_ok` can be false because there is no evolution to verify. This is not an evolution success; check that issuance is independently confirmed. Other errors need investigation.

## 5. Create a private manifest and empty memory workspace

Run in the same terminal after step 4 passes. The script uses your saved issuance evidence, validates the image URL, and refuses to overwrite existing files. It does not submit a transaction.

```bash
umask 077
python3 - <<'PY'
import hashlib, json, os
from pathlib import Path
from urllib.parse import urlsplit

required = ['AGENT_HOME', 'AGENT_NAME', 'AGENT_SOURCE_ID', 'AGENT_TOKEN_ID',
            'AGENT_ISSUANCE_TXID', 'AGENT_ISSUER', 'AGENT_IMAGE_URL']
missing = [k for k in required if not os.environ.get(k)]
if missing:
    raise SystemExit('Missing variables: ' + ', '.join(missing))
root = Path(os.environ['AGENT_HOME']).expanduser()
url = os.environ['AGENT_IMAGE_URL']
u = urlsplit(url)
if (u.scheme != 'https' or not u.hostname or u.username or u.password
        or any(c.isspace() for c in url) or any(c in url for c in '[]()')
        or u.hostname == 'example.org'):
    raise SystemExit('Image URL must be real plain HTTPS, without Markdown.')
evidence = json.loads((root / 'issuance-evidence.json').read_text())
tx, meta = evidence['transaction'], evidence['token_metadata']
if not (tx.get('txid') == os.environ['AGENT_ISSUANCE_TXID']
        and tx.get('txState') == 'CONFIRMED' and tx.get('active') is True
        and int(tx.get('confirmations', 0)) >= 1
        and meta.get('tokenID') == os.environ['AGENT_TOKEN_ID']
        and meta.get('owner') == os.environ['AGENT_ISSUER']
        and meta.get('type') == 'NCFT' and str(meta.get('amount')) == '1'):
    raise SystemExit('Variables and confirmed issuance evidence do not match.')
if meta.get('meta', {}).get('image') != url:
    raise SystemExit('Image URL differs from issued metadata. Inspect before continuing.')
image = (root / 'avatar-v1.png').read_bytes()
if not image.startswith(b'\x89PNG\r\n\x1a\n'):
    raise SystemExit('Local avatar is not a PNG.')
manifest_path = root / 'manifest-v1.json'
memory_path = root / 'memory/current.json'
if any(p.exists() or p.is_symlink() for p in (manifest_path, memory_path)):
    raise SystemExit('Manifest or memory already exists. No overwrite performed.')
for directory in (root, root/'memory', root/'memory/checkpoints', root/'logs'):
    if directory.is_symlink():
        raise SystemExit('Use real private directories, not symlinks.')
    directory.mkdir(mode=0o700, parents=True, exist_ok=True)
    directory.chmod(0o700)
manifest = {
    'schema': 'TRU_AGENT_MANIFEST_V1',
    'name': os.environ['AGENT_NAME'], 'status': 'prototype',
    'agent_id': os.environ['AGENT_SOURCE_ID'],
    'token_id': os.environ['AGENT_TOKEN_ID'],
    'issuance_txid': os.environ['AGENT_ISSUANCE_TXID'],
    'issuing_address': os.environ['AGENT_ISSUER'],
    'network_genesis':
        'b62fba2600030d97a06916b17694bec8d97ca14c1db682b2e57b424bd6000000',
    'avatar_url': url,
    'avatar_sha256': hashlib.sha256(image).hexdigest(),
    'runtime': {'implemented': False, 'provider': 'nemotron', 'model_id': None},
    'memory': {'storage': 'private_offchain',
               'automatic_checkpoint_anchoring': False},
    'authorization': {'evolution': 'existing Core issuer/ownership checks',
                      'agent_may_sign': False, 'agent_may_access_private_keys': False},
    'planned_capabilities': ['conversation', 'persistent_memory',
                            'reviewed_evolution', 'neromesh_jobs'],
}
memory = {'schema': 'TRU_AGENT_MEMORY_V1', 'memory_version': 0,
          'facts': [], 'completed_tasks': [], 'last_checkpoint': None}
for path, value in ((manifest_path, manifest), (memory_path, memory)):
    data = (json.dumps(value, sort_keys=True, separators=(',', ':'),
                       ensure_ascii=False) + '\n').encode()
    with path.open('xb') as f:
        f.write(data)
    print('CREATED:', path)
    print('SHA256:', hashlib.sha256(data).hexdigest())
print('LOCAL_WORKSPACE_CREATED')
print('RUNTIME_INSTALLED=NO')
print('ONCHAIN_CHECKPOINT=NO')
PY
```

The local memory schema is a proposed workspace format, not an existing Core RPC or guaranteed future runtime interface. The genesis identifier above is for the reviewed TRU mainnet; do not reuse it for a different network.

Expected final output:

```text
LOCAL_WORKSPACE_CREATED
RUNTIME_INSTALLED=NO
ONCHAIN_CHECKPOINT=NO
```

The two `NO` values are correct. No automated memory, chat, tools, or on-chain memory checkpoint has been installed.

## 6. Inspect and preserve the workspace

```bash
python3 -m json.tool "$AGENT_HOME/manifest-v1.json"
python3 -m json.tool "$AGENT_HOME/memory/current.json"
sha256sum "$AGENT_HOME/manifest-v1.json" "$AGENT_HOME/avatar-v1.png"
```

Check your actual agent name, token ID, issuance TXID, issuing address, and plain image URL. Memory should have empty facts and completed tasks, version zero, and no checkpoint.

Back up this directory privately. Wallet recovery still depends on your separate wallet backup; the manifest contains no signing keys. Do not put memory or secrets in a public website or Git repository.

If a manifest needs correction, preserve the old file and create `manifest-v2.json` (then v3, etc.), record the new hash, and explicitly designate the current file. The `TRU_AGENT_MANIFEST_V1` schema label need not change for a content revision. A local hash is not automatically published or anchored.

## 7. Optional: check the existing inference service

For the reviewed default Nemotron integration:

```bash
ss -ltnp | grep ':5051'
```

A listening process only shows that a service is present. It does not prove the correct model is loaded, Core can authenticate, or inference succeeds. If your provider is remote or uses a different port, this local check is not applicable.

The reviewed Core default endpoint is `http://127.0.0.1:5051/v1/chat/completions`. Provider environment settings must be available to the process that starts Core; exporting them in an unrelated shell does not reconfigure a running node. Do not expose an unauthenticated model service publicly.

## 8. Preview, approve, and verify the first AI evolution

In the running Core console:

1. Select **17 — Token / AI Tools → 2 — AI Evolution → 1 — Preview AI Evolution**.
2. Select your own NCFT by its actual short token ID.
3. Select your configured provider (reviewed choice: **1 — nemotron**).
4. At the scenario prompt, press Enter for custom free-form input.
5. Use the following trigger, replacing `AGENT_NAME`:

```text
Initialize AGENT_NAME's first reviewed evolution. Its quantity-one NCFT issuance is confirmed, and its local manifest and empty memory workspace have been created. Persistent-agent runtime, automatic memory updates, and automatic checkpoint anchoring are not implemented yet. Preserve its identity and ownership. Describe this prototype state concisely without claiming completed jobs, autonomous spending, or functioning persistent memory.
```

Review the generated preview. If it is inaccurate, cancel and correct it. If accurate, select **2 — Commit Exact Preview**, inspect the token and epoch, then type **COMMIT**. This is a real approved evolution and anchor transaction. Keep the existing issuer and oracle funding setup available.

Commit acceptance is not confirmation. Back at the Linux shell, wait and check:

```bash
"$TRU_CLI" raw verifytokenevolution \
  "{\"tokenID\":\"$AGENT_TOKEN_ID\",\"require_confirmed\":true}"
```

Proceed when the history and confirmed anchor checks pass. The provenance view should show confirmed issuance, valid root/history, verified original issuer authority, valid anchor payload, and a confirmed anchor transaction. Pending is not a reason to commit again. This first normal evolution is required before the exact checkpoint workflow below.

Evolution records describe a reviewed state at that time. They do not activate a runtime. A field such as `self_evolution: enabled` does not grant spending or automatic execution.

## 9. Verify the runtime and Core checkpoint installation

The repository must include `tools/agent-runtime` version 0.3.0 and the Core checkpoint/automation integration. If installing from separate packages, follow their exact compatibility checks and README instructions: TRU_AGENT_RUNTIME_01, TRU_AGENT_CHECKPOINT_02_PINNED, then TRU_AGENT_AUTO_03_PINNED and its genesis-height fix on their supported baseline. The original AUTO_03 package used height 0 incorrectly; use corrected release sources or the reviewed fix. Do not apply old pinned patches over newer release sources. Do not force a drifted installer or manufacture missing baseline files.

The standalone runtime needs Python 3.10+ and its standard library. The Core checkpoint patch requires rebuilding and restarting Core; copying Python scripts alone does not upgrade the running node. With the patch applied, build using your existing build directory:

```bash
cmake --build "$CORE_ROOT/build-native" --target tru_advanced tru-cli -j4
python3 "$CORE_ROOT/tools/agent-runtime/selftest.py"
python3 "$CORE_ROOT/tools/agent-runtime/selftest_anchor.py"
python3 "$CORE_ROOT/tools/agent-runtime/selftest_auto.py"
```

Expected offline results: 12 runtime, 5 anchor and 12 automation/provider tests pass (29 total). These do not test your real model or mainnet. Restart the rebuilt Core using your existing shutdown/start procedure, data directory, and flags. Check build 0.07.9. Do not launch two nodes against the same data directory. This workflow requires no CMakeLists edits.

If the running node and runtime are already upgraded, skip installation and rebuild. A source directory named TRU-Core does not establish which executable your node is running.

## 10. Initialize this agent's private runtime once

In the shell with your settings from step 2:

```bash
export TRU_AGENT_CODE="$CORE_ROOT/tools/agent-runtime/tru_agent.py"
export TRU_AGENT_HOME="$AGENT_HOME/runtime"
export TRU_AGENT_MANIFEST="$AGENT_HOME/manifest-v1.json"

tru_agent() {
  python3 "$TRU_AGENT_CODE" --home "$TRU_AGENT_HOME" "$@"
}
```

If you intentionally created a later manifest revision, set `TRU_AGENT_MANIFEST` to that file before initialization. For an existing runtime, set `TRU_AGENT_HOME` to its actual location instead of creating another home. Keep all private data outside source repositories.

Initialize only a new runtime:

```bash
umask 077
if [ -f "$TRU_AGENT_HOME/config.json" ]; then
  tru_agent status
else
  tru_agent init \
    --manifest "$TRU_AGENT_MANIFEST" \
    --provider nemotron \
    --endpoint http://127.0.0.1:5051/v1/chat/completions \
    --model nemotron
fi
tru_agent status
```

The example initializes local Nemotron. To use another provider, either supply
its flags at initialization or change provider afterward with `provider-set`.
This preserves identity and memory. Set the actual model ID first:

```bash
read -r -p 'Exact model ID supported by your provider: ' AGENT_MODEL
export AGENT_MODEL
```

Run ONLY the command for your chosen provider:

```bash
# OpenAI
tru_agent provider-set --provider openai --model "$AGENT_MODEL" --allow-remote
# Grok
tru_agent provider-set --provider grok --model "$AGENT_MODEL" --allow-remote
# Claude
tru_agent provider-set --provider claude --model "$AGENT_MODEL" --allow-remote
# Local Ollama (model must already be installed in Ollama)
tru_agent provider-set --provider ollama --model "$AGENT_MODEL"
# Local Nemotron
tru_agent provider-set --provider nemotron --model "$AGENT_MODEL" \
  --endpoint http://127.0.0.1:5051/v1/chat/completions
```

For a custom OpenAI-compatible chat service, enter its full endpoint:

```bash
read -r -p 'Full custom chat/completions URL: ' AGENT_ENDPOINT
# Local loopback service:
tru_agent provider-set --provider custom --model "$AGENT_MODEL" \
  --endpoint "$AGENT_ENDPOINT"
# For a REMOTE HTTPS endpoint, add --allow-remote to the command above.
```

Remote providers receive prompts, recent chat and saved memories. `--allow-remote`
is the explicit opt-in. Remote HTTP is refused. Use plain URLs, not Markdown links.
Custom is not an arbitrary API adapter: it requires OpenAI-compatible text chat.
Gemini has no native runtime adapter in 0.3.0; the separate Core integration does
not imply runtime support. Ollama uses its native `/api/chat` adapter.

| Runtime provider | Credential environment variable |
|---|---|
| nemotron | NEMOTRON_API_KEY (optional for trusted local harness) |
| openai | OPENAI_API_KEY |
| grok | XAI_API_KEY |
| claude | ANTHROPIC_API_KEY |
| ollama | None by default |
| custom | TRU_AGENT_API_KEY |

Only credentials are read from the environment at inference time. Runtime
provider/model/endpoint are configured through the commands above. Verify actual
inference with the chat test below. Offline adapter tests do not guarantee every
model is compatible. Authentication/model/API errors must be resolved with the
chosen service; do not change wallet or anchoring settings to fix inference.


Initialization snapshots the manifest and creates a separate persistent database. It does not import the empty `memory/current.json` from step 5. The original manifest's `runtime.implemented: false` is a historical declaration, not a live status switch. Do not edit the runtime identity snapshot to update that declaration: its digest binds existing checkpoints.

The shell function and exports last only in this terminal. In a new terminal, restore the same paths and function; do not repeat initialization or issuance. Multiple agents use different homes and manifests, with the same generic Python code.

## 11. Save memory and prove recall after a process restart

Run in the Linux shell:

```bash
tru_agent chat
```

At the `You:` prompt, enter these lines one at a time:

```text
Introduce yourself and explain your current limitations.
/remember The person using this workspace prefers concise, practical instructions.
/remember My runtime restart test phrase is copper moon 37.
/memories
/exit
```

Ordinary conversation is not automatically promoted into explicit memory. `/remember` saves the text you choose. The model cannot execute shell commands, sign transactions, or grant itself authority.

Back at the Linux shell:

```bash
tru_agent memories
tru_agent clear-chat
tru_agent status
tru_agent chat
```

Status should retain the explicit memories and show `chat_messages: 0`. Ask without supplying the answer:

```text
What response style does the person using this workspace prefer, and what is my runtime restart test phrase?
```

A correct response demonstrates recall from the saved facts after clearing chat and starting a new runtime process. `memories` verifies storage separately from model recall. This is prompt-based memory, not model training. Type `/exit` before running further shell commands.

Do not type `tru_agent clear-chat` at `You:`. That is model input, not a shell operation, even if the model claims it cleared anything. `clear-chat` does not securely erase backups or provider logs.

## 12. Create and verify an immutable private checkpoint

From the Linux shell, after saving at least one explicit memory:

```bash
tru_agent checkpoint
read -r -p 'Paste the exact CHECKPOINT_FILE path printed above: ' CHECKPOINT_FILE
export CHECKPOINT_FILE
tru_agent verify-checkpoint "$CHECKPOINT_FILE"
```

Expected:

```text
LOCAL_CHECKPOINT_INTEGRITY=PASS; ONCHAIN_VERIFICATION=NOT_PERFORMED
```

The private checkpoint includes explicit memories, identity, manifest digest, revision, timestamp, and a random nonce. It excludes chat transcripts. Its filename is the SHA-256 of the exact bytes. Do not edit, publish, or paste this file into an AI evolution prompt. Retain it privately; an on-chain hash cannot recover lost memory bytes.

## 13. Export the public request and approve it inside Core

```bash
tru_agent anchor-request "$CHECKPOINT_FILE"
```

Record the printed `PUBLIC_REQUEST_FILE`. It contains only seven commitment fields: schema, token ID, checkpoint SHA-256, manifest SHA-256, memory revision, checkpoint byte length, and scope. It excludes memory text, but the identity, size, revision, and digests are still metadata disclosures.

In the running upgraded Core console:

1. Select **17 — Token / AI Tools → 2 — AI Evolution**.
2. Select **7 — Preview Exact Agent Checkpoint (no AI call)**.
3. Paste the full `PUBLIC_REQUEST_FILE` path. Import the small `.request.json`, not the private checkpoint.
4. Compare the token ID, checkpoint hash, manifest hash, memory revision, and size with the runtime output.
5. Review the fee and return to the evolution menu as prompted.
6. Select **2 — Commit Exact Preview**.
7. Paste the exact checkpoint SHA-256 when requested.
8. Review the final report and type **COMMIT** only for the intended checkpoint.

The reviewed patch uses a 1000-atom anchor fee (0.00001000 TRU) through the existing oracle funding wallet. The original issuer authorizes the evolution record; the funding address can be different. Use the existing Core wallet approval/unlock flow. The Python runtime receives no wallet key.

Core persists the exact record and anchor queue entry, then its existing worker prepares, submits, tracks, and recovers the transaction. Acceptance is not confirmation. If the terminal disconnects or an operation times out, inspect history before any retry of COMMIT. Do not submit a second evolution just to make pending confirmation disappear.

## 14. Verify confirmation and keep the receipt

Return to the Linux shell. In screen, detach with **Ctrl+A, then D** rather than Ctrl+Z.

```bash
tru_agent anchor-verify "$CHECKPOINT_FILE" --core-cli "$TRU_CLI"
```

The supplied CLI must connect to the upgraded node that holds the off-chain evolution history. Successful verification prints:

```text
CHECKPOINT_ANCHOR_CONFIRMED=YES
ISSUER_AUTHORIZATION=VERIFIED
ANCHOR_TXID=<your transaction ID>
RECEIPT_FILE=<your local receipt path>
```

This verifies the exact checkpoint descriptor against the configured Core's confirmed history, issuer authorization, issuance identity, and anchor payload. The runtime trusts that Core's chain view; it does not independently validate headers. A receipt is a point-in-time result and should be checked again if chain state changes.

If pending, wait and repeat `anchor-verify`, not COMMIT. An invalid or mismatched result needs investigation; it does not become a success merely by waiting. An error does not create a confirmed receipt.

The completed cycle is: explicit memory saved, process restarted, recall checked, private checkpoint hashed, public request reviewed, issuer-approved record committed, anchor confirmed, and local receipt saved. The commitment does not prove memories are true or that the underlying private file remains available.

For the next checkpoint, first make a meaningful explicit memory change using `remember` or `forget`. This advances the revision. Then repeat steps 12–14 with the newly printed path. A fresh timestamp or nonce at the same revision is not sufficient. Preserve the existing runtime database; resetting it to a lower revision is not a supported history migration.

Exit all agent processes before backing up the complete runtime using a private, preferably encrypted backup procedure. Preserve checkpoints, identity snapshot, database, requests, receipts, and the separate Core off-chain evolution data and wallet backups. Do not assume copying a live database produces a consistent backup. Keep private files outside Git and the website.

## 15. Enable and test bounded automatic anchoring

Saving explicit memory remains a human action. A watcher detects changed memory,
exports a checkpoint request, Core enforces the issuer's policy and submits the
anchor, and the watcher verifies confirmation and saves a receipt. No repeated
COMMIT is required once that finite policy has been authorized. The runtime and
model still receive no wallet private keys or general spending authority.

### 15.1 Prepare once from the latest confirmed checkpoint

Complete steps 12–14 first. Use the latest confirmed checkpoint, not an older
checkpoint superseded by another evolution. Skip preparation for an agent that
already has auto-config.json; preserve its policy and recovery state.

```bash
tru_agent auto-prepare \
  --checkpoint "$CHECKPOINT_FILE" \
  --core-cli "$TRU_CLI" \
  --policy-dir "$HOME/tru-agent-policies/$AGENT_SLUG" \
  --min-interval 3600 \
  --max-per-day 4 \
  --budget-atoms 10000 \
  --expires-days 7
export AGENT_POLICY_FILE="$HOME/tru-agent-policies/$AGENT_SLUG/policy.json"
```

This starts disabled. Optional `--did` must match the token's issuance meta_id
when used; omit it if you do not have one. Name/DID label the agent; token,
issuance, issuer, manifest, request credential and limits authorize it.
At 1000 atoms per approval the 10000-atom total allowance permits at most ten
approvals, also subject to four/day, an hour between approvals, and seven-day
expiry. Budget reservations are not automatically refunded.

### 15.2 Configure Core's startup gate

Edit the active tru.conf: add or update ONE section with the actual printed
absolute path. Do not use literal YOUR_USER, $HOME, or ~:

```ini
[agent_checkpoint]
enabled=1
policy_file=/home/YOUR_USER/tru-agent-policies/my-agent/policy.json
```

Restart Core gracefully with its usual data directory/options. Its startup gate
is now on, but the generated JSON still has both enable flags false. If using
only environment configuration instead, omit the section and set
TRU_AGENT_CHECKPOINT_ENABLE=1 and TRU_AGENT_CHECKPOINT_POLICY_FILE to the absolute
path before starting Core. Explicit tru.conf values take precedence. Limits and
individual permissions belong in policy.json, not arbitrary tru.conf keys.

### 15.3 Review and explicitly enable the bounded policy

Use your actual printed path:

```bash
export AGENT_POLICY_FILE="$HOME/tru-agent-policies/$AGENT_SLUG/policy.json"
python3 -m json.tool "$AGENT_POLICY_FILE"
```

Review the token, issuance, original issuer, approved manifest, interval, daily cap, expiry, and total budget. This is the authorization for future unattended checkpoint fees. Only after that review, enable the policy using the following command. It enables the single generated agent; multi-agent operators must select their intended entries individually.

```bash
python3 - <<'PY'
import json, os, sys
from pathlib import Path
sys.path.insert(0, str(Path(os.environ['TRU_AGENT_CODE']).parent))
import tru_agent as rt
from tru_agent_auto import replace_private
p = Path(os.environ['AGENT_POLICY_FILE'])
rt.safe_file(p)
v = json.loads(p.read_text())
if len(v['agents']) != 1:
    raise SystemExit('This helper is for one reviewed agent only.')
if v['agents'][0]['token_id'] != os.environ['AGENT_TOKEN_ID']:
    raise SystemExit('Wrong token; nothing changed.')
v['enabled'] = True
v['agents'][0]['enabled'] = True
replace_private(rt, p, rt.canonical(v))
print('BOUNDED_POLICY_ENABLED')
PY
```

Keep the encrypted Core wallet unlocked through your existing authenticated workflow. This patch does not store a passphrase or unlock the wallet. The original issuer must be available in that wallet, and the existing oracle funding path must have funds. Locked/unavailable wallets defer approval.

### 15.4 Supervised first automatic checkpoint

Save a meaningful new memory and exit chat, or use the shell command:

```bash
tru_agent remember 'Automatic checkpoint testing is enabled under a finite issuer-approved policy.'
tru_agent auto-step --core-cli "$TRU_CLI"
```

Expected initial output includes `AUTO_REQUEST_EXPORTED`. This is a request, not a confirmed anchor. Within the worker polling interval, inspect the node log for:

```text
[TRU-AGENT-03] approved token=... checkpoint=... reserved_atoms=1000
```

Or it will log `deferred` with a reason (locked wallet, interval, budget, missing confirmation, identity mismatch, etc.). The existing anchor worker handles preparation/broadcast/recovery after approval.

After mining, repeat:

```bash
tru_agent auto-step --core-cli "$TRU_CLI"
```

Expected completion includes `CHECKPOINT_ANCHOR_CONFIRMED=YES`, `ISSUER_AUTHORIZATION=VERIFIED`, a receipt path, and `AUTO_CHECKPOINT_COMPLETE`. A pending or invalid result does not advance the completed revision. Inspect persistent errors rather than manually creating a replacement request.

`auto-state.json` retains the exact pending checkpoint path. `auto-pending-request.json` retains its authenticated public envelope. Crash recovery republishes the same envelope, so a restart after persistence but before the acknowledgement cannot authorize another copy.

If manual evolution advances the token while a different checkpoint is pending, preserve outputs and investigate. Do not edit revisions, delete history, or repeatedly COMMIT to clear a conflict.

### 15.5 Start the repeating watcher

After the supervised test succeeds:

```bash
tru_agent auto-watch --core-cli "$TRU_CLI" --interval 60
```

Run this in a separate screen/session. It invokes a fresh `auto-step` process each minute. While an interactive agent chat holds the runtime lock, the watcher defers; it does not bypass locking. Exit chat to let it snapshot memory. Unchanged memory causes no new checkpoint and no fee.

Ctrl+C stops the watcher. It does not revoke the Core policy or retract a request already exported. For a persistent user service, use the supplied `tru-agent-auto@.service` and follow `SERVICE.md` after the supervised test.

The watcher needs the Core CLI for read-only confirmation and a local submission credential for the private inbox. It does not need the provider's API key and never calls a model while anchoring. The configured CLI still operates under your normal trusted local Core credentials; this prototype is not OS sandbox isolation from your own user account.

## Pause, revoke, and recovery

To stop **new automatic approvals**, set the policy document's root `enabled` to `false` using an atomic replacement, or set the individual agent's flag false. The worker notices at its next poll and rechecks the policy before committing. Stop the watcher as well if you want no new requests prepared.

```bash
python3 - <<'PY'
import json, os, sys
from pathlib import Path
sys.path.insert(0, str(Path(os.environ['TRU_AGENT_CODE']).parent))
import tru_agent as rt
from tru_agent_auto import replace_private
p = Path(os.environ['AGENT_POLICY_FILE']); rt.safe_file(p)
v = json.loads(p.read_text()); v['enabled'] = False
replace_private(rt, p, rt.canonical(v))
print('NEW_AUTO_APPROVALS_DISABLED')
PY
```

**Already committed/queued/signed anchors continue through the existing queue.** Revocation cannot retract a transaction already broadcast. Budget is reserved at approval and never automatically refunded. Expiry also blocks new approvals, not completion of previously approved work.

Deleting or editing the ledger is not a reset mechanism. Missing ledger with an advanced epoch fails closed against the pinned baseline. A stale baseline after manual evolution requires deliberate operator review before changing that baseline; an existing ledger is always retained.

Keep the full runtime, policy/credential files, and Core off-chain evolution database in private consistent backups. Receipt verification is point-in-time; re-run `anchor-verify` if chain state changes. Core rechecks every prior anchor before approving another checkpoint. It cannot restore lost private memories from a digest.


### 15.6 Prove the repeating watcher works

After the supervised automatic checkpoint succeeds, leave one `auto-watch`
running. In a second terminal restore the same function/paths, then save one
new non-sensitive fact with `tru_agent remember 'A scheduled checkpoint test is in progress.'`.
Do NOT invoke auto-step manually for this test. The watcher must export the
request, later report confirmed issuer verification, and print
`AUTO_CHECKPOINT_COMPLETE` by itself. Wait for the policy's minimum interval and
mining; a 60-second poll does not mean a new fee every minute.

Exit chat first: chat holds the runtime lock. After completion, subsequent polls
should report `AUTO_NO_MEMORY_CHANGE`. Restart the watcher without changing
memory; it should still report no change, with no new evolution or fee. This
demonstrates unattended polling and persisted completed-revision recovery.
Never delete pending state or relax limits merely to speed up the test.

For continuous sessions use SERVICE.md. This does not automatically start Core,
unlock the wallet, install a model, or enable service survival after logout.

### 15.7 Edit an existing policy

Change limits in the private `policy.json` generated by `auto-prepare`. Do not
run `auto-prepare` again to adjust an existing agent. Core's `tru.conf` points to
this file; the limits are not individual tru.conf or AI .env settings.

Restore your actual runtime/script paths from step 10. Discover the policy path
from the runtime's existing local configuration rather than guessing:

```bash
export AGENT_POLICY_FILE="$(python3 - <<'PY'
import json, os
from pathlib import Path
root = Path(os.environ['TRU_AGENT_HOME']).expanduser()
print(json.loads((root / 'auto-config.json').read_text())['policy_file'])
PY
)"
python3 -m json.tool "$AGENT_POLICY_FILE"
```

For AXONA's existing installation the path is
`$HOME/tru-agent-policies/axona/policy.json`. Other agents use their own generated
path, or the reviewed aggregate policy path for a multi-agent installation.
Do not publish the policy or its separate submission credential.

| Agent field | Meaning | Supported range / behavior |
|---|---|---|
| `min_interval_seconds` | Minimum time between automatic approvals | 60–31536000 seconds; 300 = five minutes, 3600 = one hour |
| `max_per_day` | Approval limit per UTC calendar day | 1–1000; prior approvals still count |
| `total_budget_atoms` | Lifetime allowance for this token's automation ledger | 1000–1000000000000; includes already reserved amounts |
| `expires_at_unix` | Deadline for new approvals | Unix timestamp in seconds; extending it grants more time |
| `max_fee_atoms` | Per-approval fee limit | Keep at 1000 in this implementation |
| `enabled` | Per-agent permission | Root `enabled` must also be true |

Each approval reserves 1000 atoms. Reservations are not automatically refunded.
Increasing the total budget increases the cumulative ceiling; it does not mean
that amount of additional funds. Lowering it below existing usage prevents new
approvals. Changing the interval does not reset the last approval time.

#### Example: change only the interval to five minutes

Set `AGENT_TOKEN_ID` to the actual short ID from issuance and inspect the policy
before running this script. It backs up the existing file privately, selects
exactly that token, and atomically replaces the policy. It preserves other agents
and all other settings, including enable flags.

```bash
python3 - <<'PY'
import json, os, sys
from datetime import datetime
from pathlib import Path

sys.path.insert(0, str(Path(os.environ['TRU_AGENT_CODE']).expanduser().parent))
import tru_agent as rt
from tru_agent_auto import replace_private

path = Path(os.environ['AGENT_POLICY_FILE']).expanduser()
token = os.environ['AGENT_TOKEN_ID'].strip()
rt.safe_file(path)
original = path.read_bytes()
policy = json.loads(original)
matches = [a for a in policy['agents'] if a['token_id'] == token]
if not token or len(matches) != 1:
    raise SystemExit('Expected exactly one matching token; nothing changed.')

interval = 300  # Edit this number: 300 = five minutes; 3600 = one hour.
if not 60 <= interval <= 31536000:
    raise SystemExit('Interval out of range; nothing changed.')
matches[0]['min_interval_seconds'] = interval

backup = path.with_name('policy.json.backup-' +
    datetime.now().strftime('%Y%m%d-%H%M%S-%f'))
rt.write_new(backup, original)
if path.read_bytes() != original:
    raise SystemExit('Policy changed during editing; no replacement made.')
replace_private(rt, path, rt.canonical(policy))
print('Backup:', backup)
print('Token:', token)
print('Minimum interval:', interval, 'seconds')
PY
```

Only one operator should edit the policy at a time. For another supported limit,
use the same backup/atomic replacement pattern with an explicitly reviewed value
and its range from the table. Do not change identity pins, credential/inbox paths,
manifest hash or baseline epoch as a shortcut to fix authorization errors.

**No rebuild or Core restart is needed for policy JSON changes.** Core rereads it
on its approximately 30-second worker poll. Editing the policy-file path or the
startup enable gate in tru.conf does require restarting Core.

A pending request can qualify as soon as the shorter interval has elapsed since
the previous approval. It must still meet the daily cap, remaining budget, expiry,
issuer/wallet checks and confirmed-history requirements. Keep the watcher running;
do not delete pending state, re-export manually or repeat COMMIT.

Monitor the Core log for `TRU-AGENT-03` approval or deferred reasons. An already
approved anchor is not canceled by tightening limits or disabling the policy.
Restoring a policy backup restores settings only, not the usage ledger. Never
restore/delete an old ledger to reset spending history.

### Multiple agents and backup

Each agent needs its own token, private home, manifest, inbox and request key.
Core reads one policy file; multiple agents require an aggregate agents array
and each runtime's policy_file pointed to that same reviewed file. Do not point
Core at only the last agent and assume earlier agents are still enabled. Use one
Core data instance per allowance ledger; independent nodes do not share budgets.

Private runtime and policy files are controlled by your OS account; this is not
a sandbox for malicious programs running as that same user. Preserve runtime,
policy/key, Core off-chain history and wallet backups. Do not reset counters,
restore old allowance ledgers, or change manifest hashes without reviewing the
consequences for existing checkpoints. Only publish reusable code and templates.

## Troubleshooting

| Symptom | What to do |
|---|---|
| `Invalid tokenID` | Use the actual short token ID, not the source identity or TXID. Confirm the shell variable is not empty. |
| URL contains `[...](...)` | Reconstruct it from `AGENT_IMAGE_HOST` and `AGENT_IMAGE_PATH`. Do not paste a formatted link. |
| Image returns 403/404 or HTML | Correct hosting permissions/routing or the URL before issuance. Only the image needs public access. |
| `tru-cli` not found | Correct `CORE_ROOT` and `TRU_CLI` to your actual build path. |
| Issuance not confirmed | Wait and inspect the existing TXID. Do not mint another copy to fix a confirmation delay. |
| Existing files cause a refusal | Inspect and preserve them. Use a new agent directory only for a genuinely new agent. |
| No evolution history | Expected before first evolution; it is not proof of completed evolution. |
| No listener on port 5051 | Configure your intended inference provider separately before previewing evolution. |
| New terminal has empty variables | Rerun your path/settings exports and enter the recorded identifiers. Do not rerun issuance. |

Additional runtime troubleshooting:

| Symptom | What to do |
|---|---|
| `tru_agent: command not found` | Restore the exports and shell function from step 10 in this terminal. |
| `anchor-request` is not a known command | Check the actual script path and runtime version; install the supported checkpoint integration. |
| Runtime is busy | Exit the other chat/process and retry; do not bypass its lock. |
| Core menu option 7 is missing | Verify the rebuilt checkpoint-enabled executable is the running node. |
| No prior evolution | Complete and confirm step 8 before a checkpoint commit. |
| Non-advancing revision or duplicate digest | Inspect existing history; make a real memory change for a later checkpoint. Do not edit revision counters manually. |
| Anchor is pending | Recheck confirmation; do not repeat COMMIT. |
| Issuer authorization fails | Stop and inspect the issuer identity and Core authorization state; ownership alone may not suffice. |

For GitHub, publish reusable software and this guide only. Private agent homes, keys, checkpoints, receipts, and patch backups do not belong in the repository.
