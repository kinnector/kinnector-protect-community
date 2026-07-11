# Kinnector Protect Community

Kinnector Protect Community is the central repository for threat signatures, security policies, allowlists, and behavioral rulesets used across the Kinnector ecosystem.

---

## Why this exists

Threat signatures and application exceptions must change dynamically without recompiling EDR daemon binaries. However, managing rules in unstructured text formats makes it difficult to audit changes and introduces parsing latency.

Kinnector Protect Community solves this by providing a version-controlled repository of policies, YARA signatures, and Sigma correlation rules. These configurations are validated, compiled into unified binary FlatBuffers databases, and signed cryptographically for distribution.

---

## Directory Structure

* **`configs/`**: Approved vendors and process execution capabilities:
  * `trusted_vendors.json`: Approved code signers (Team IDs, Authenticode publisher names).
  * `trusted_cli_tools.json`: Scoped sensitive category maps for command-line utilities.
  * `app_overrides.json`: Local exceptions and application overrides.
* **`policies/`**: TOML configuration files defining host-level enforcement actions.
* **`rules/`**: Threat intelligence rulesets:
  * `yara/`: YARA rules for memory scanning and binary analysis.
  * `sigma/`: Sigma correlation logic for Windows ETW and Linux eBPF telemetry.

---

## Configuration Formats

### 1. Trusted Code Signers (`configs/trusted_vendors.json`)
Defines publisher signatures that bypass standard heuristic checks:

```json
{
  "vendors": [
    {
      "name": "Google LLC",
      "team_id": "EQHXZ8M8AV",
      "platforms": ["windows", "macos"]
    },
    {
      "name": "Mozilla Corporation",
      "team_id": "959G39P522",
      "platforms": ["windows", "macos", "linux"]
    }
  ]
}
```

### 2. Scoped CLI Rules (`configs/trusted_cli_tools.json`)
Maps command-line utilities to specific telemetry categories they are authorized to access (such as reading credentials or establishing network connections):

```json
{
  "cli_tools": [
    {
      "name": "ssh",
      "path": "/usr/bin/ssh",
      "allowed_categories": ["CAT_SSH_KEYS"]
    },
    {
      "name": "git",
      "path": "/usr/bin/git",
      "allowed_categories": ["CAT_SSH_KEYS", "CAT_CLOUD_CREDS"]
    }
  ]
}
```

### 3. Policy Structures (`policies/default_policy.toml`)
Configures threshold levels and enforcement reactions:

```toml
[policy]
name = "block-unauthorized-dns"
action = "Block"
severity = "High"

[[rules]]
type = "network"
port = 53
allow_list = ["1.1.1.1", "8.8.8.8"]
```

---

## Validation & Local Verification

Verify rule and policy syntax locally prior to submission:

```bash
antitheft-cli rules validate --path ./policies/
```

Trigger policy reload on the active daemon instance:

```bash
antitheft-cli rules reload
```

---

## Distribution Pipeline

```
[ Git Pull Request ] ──> [ Schema Validation CI ] ──> [ FlatBuffers Compiler ] ──> [ Ed25519 Signing ] ──> [ Agent Update API ]
```

1. **Validation**: CI checks incoming pull requests for schema compliance and syntax correctness.
2. **Compilation**: Validated rules are compiled into a unified binary rules database (`rules.db`) utilizing the FlatBuffers schema.
3. **Signing**: The database is signed with the organization's private key.
4. **Ingestion**: Kinnector Agent verifies the Ed25519 signature of `rules.db` prior to memory hot-swapping.
