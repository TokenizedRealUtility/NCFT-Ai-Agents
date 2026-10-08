# Create a TRU NCFT agent identity

**Stop point: before the first AI evolution.** This creates an agent identity and local files. It does not install an AI model, launch a persistent agent, or anchor its memory on-chain.

These menu numbers and RPC examples match the reviewed TRU Core 0.07.7. Other builds may use different menus. This is a Linux/Bash guide for an already configured mainnet node and wallet; it is not a node installation guide.

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
| ai_engine | `nemotron` if you intend to use the existing Nemotron integration. A label does not install a model. |
| style_descriptor | Your desired style, for example `helpful, concise, provenance-aware`. |
| dynamic_morph | `issuer-authorized evolution; persistent runtime in development` |
| update_interval | `manual reviewed milestones` |
| creator_signature | Leave blank; do not invent a cryptographic signature. |
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

## Stop here: ready for a separate first-evolution test

You have completed:

- One confirmed quantity-one NCFT representing your new agent identity.
- A hosted avatar and local byte hash.
- Recorded issuance evidence.
- A private manifest and empty memory workspace.

You have **not** completed AI evolution, installed a persistent-agent runtime, or anchored a memory checkpoint.

The next separate test uses **17 — Token / AI Tools → 2 — AI Evolution → 1 — Preview AI Evolution**. Review the generated content before using **Commit Exact Preview**. This README intentionally stops before preview or commit.

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
