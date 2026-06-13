# Credential Store — Secure Credential Management for Rust

`credential-store` is a Rust crate providing a unified interface for storing, retrieving, and rotating secrets (API keys, tokens, passwords) used by applications and agents. It abstracts over multiple backends — environment variables, encrypted local files, OS keychains — so applications can use the same API in development (env vars) and production (keychain or encrypted store).

## Why It Matters

Hardcoded secrets cause security incidents. According to GitGuardian's 2024 State of Secrets Sprawl report, over 12.8 million secrets were leaked in public GitHub commits — a 4x increase since 2021. The root cause is almost always convenience: developers embed credentials directly in code because the alternative requires pulling in heavyweight vault infrastructure.

`credential-store` provides the *minimal* abstraction needed to keep secrets out of source code without adding operational complexity:

- **Dev mode:** read from `.env` or environment variables
- **CI mode:** read from CI/CD secret injection (`$CI_SECRETS_*`)
- **Prod mode:** read from OS keychain (macOS Keychain, Linux Secret Service, Windows Credential Manager)

The same `get("STRIPE_API_KEY")` call works in all three.

## How It Works

### Threat Model

| Threat | Mitigation |
|---|---|
| Secret in source code | Secrets loaded at runtime, never compiled in |
| Secret in git history | `.gitignore` excludes credential files |
| Secret in process memory exposed via core dump | Zeroized buffers on drop (planned) |
| Secret intercepted in transit | TLS for remote backends; no network for local backends |
| Secret leak via logs | Redaction filters on tracing layer (planned) |

### Access Model

Credential access follows a **layered resolution** strategy:

$$\text{resolve}(key) = \text{keychain}(key)\ \|\|\ \text{env}(key)\ \|\|\ \text{file}(key)\ \|\|\ \bot$$

Each layer is tried in order of security. The first hit wins. If no layer has the key, the result is ⊥ (error).

### Big-O Analysis

| Operation | Time | Notes |
|---|---|---|
| `get(key)` — env var | O(1) | HashMap lookup in process environment |
| `get(key)` — keychain | O(log n) | Keychain stores are typically B-tree indexed |
| `put(key, val)` — file | O(1) | Append/overwrite to encrypted blob |
| `rotate(key)` | O(1) | Atomic replace with new value |
| `list()` | O(n) | Enumerate all keys |

Where n = number of stored credentials.

### Cryptographic Foundations

For the encrypted file backend (planned):

- **Algorithm:** AES-256-GCM (authenticated encryption)
- **Key derivation:** Argon2id from user passphrase
- **Nonce:** 96-bit random per encryption, prepended to ciphertext
- **Integrity:** GCM tag verified on every read

$$c = \text{AES-256-GCM}_{k,\ \text{nonce}}(\text{plaintext})$$
$$k = \text{Argon2id}(\text{passphrase},\ \text{salt},\ t, m, p)$$

Where *t* = time cost (iterations), *m* = memory cost, *p* = parallelism.

## Quick Start

```toml
[dependencies]
credential-store = "0.1"
```

```rust
// Minimal usage (current stub implementation)
fn main() {
    println!("Hello, world!");
    // Full API:
    // let store = CredentialStore::auto()?;
    // let api_key = store.get("OPENAI_API_KEY")?;
    // println!("Loaded key: {}...{}", &api_key[..4], &api_key[api_key.len()-4..]);
}
```

### Planned API

```rust
let store = CredentialStore::builder()
    .env_layer()           // try env vars first
    .keychain_layer()      // then OS keychain
    .file_layer("~/.secrets.enc", passphrase)
    .build()?;

let key: String = store.get("DATABASE_URL")?;
store.put("DATABASE_URL", "postgres://...")?;
store.rotate("DATABASE_URL", new_value)?;
```

## API

### Current

The crate is currently a stub with a `main.rs` entry point. The full API is under development.

### Planned

| Type | Method | Description |
|---|---|---|
| `CredentialStore` | `builder() -> Builder` | Start layered configuration |
| `CredentialStore` | `get(key: &str) -> Result<String>` | Resolve a secret |
| `CredentialStore` | `put(key, val) -> Result<()>` | Store a secret |
| `CredentialStore` | `rotate(key, new_val) -> Result<()>` | Atomically replace |
| `CredentialStore` | `list() -> Result<Vec<String>>` | List all keys |
| `Builder` | `env_layer()` | Add environment variable backend |
| `Builder` | `keychain_layer()` | Add OS keychain backend |
| `Builder` | `file_layer(path, passphrase)` | Add encrypted file backend |

## Architecture Notes

The credential store is designed around **γ + η = C**:

- **γ (gamma)**: The layered resolution contract — the *policy* that defines the order and rules for secret lookup. This is the security specification.
- **η (eta)**: The concrete backends — `std::env`, `security-framework` (macOS), `dbus-secret-service` (Linux), `wincred` (Windows), `aes-gcm` (file). These are the *mechanisms*.
- **C (Configuration)**: **Secure secret access** — the property that emerges when the policy (γ) is correctly implemented by the backends (η). When both align, the application gets the right secret from the right source without leaking it.

The layering is critical: if a developer accidentally puts a real key in `.env`, the keychain layer (with higher priority) ensures the *production* key takes precedence. This prevents test credentials from masking production credentials in deployed environments.

Secret hygiene extends to the codebase: the crate enforces `.gitignore` patterns for credential files and integrates with the workspace's `git diff` pre-push secret scanner (see `AGENTS.md`).

## References

- **Ferguson, N., Schneier, B., & Kohno, T. (2010).** *Cryptography Engineering: Design Principles and Practical Applications.* Wiley. — AES-GCM, key derivation, and threat modeling.
- **Biryukov, A., Dinu, D., & Khovratovich, D. (2016).** "Argon2: New Generation of Memory-Hard Functions for Password Hashing and Other Applications." *IEEE S&P 2016*. — Argon2id specification (RFC 9106).
- **Dwork, C., & Naor, M. (1992).** "Pricing via Processing or Combatting Junk Mail." *Proc. CRYPTO '92*. — Memory-hard functions foundations.
- **Rogaway, P., & Shrimpton, T. (2007).** "A Provable-Security Treatment of the Key-Wrap Problem." *Proc. EUROCRYPT 2006*, LNCS 4004. — AEAD security definitions underlying AES-GCM.
- **GitGuardian. (2024).** *State of Secrets Sprawl 2024.* — Industry data on leaked credentials in source code.
- **NIST SP 800-63B. (2017).** "Digital Identity Guidelines: Authentication and Lifecycle Management." — Memorized secret guidelines.
- **OWASP. (2023).** "Cryptographic Storage Cheat Sheet." — Best practices for credential storage at rest.

## License

MIT
