# Credential Store

**A Rust library for secure credential management** that provides the foundational structure for storing and retrieving sensitive authentication data — API keys, passwords, tokens, and certificates — in a containerized runtime environment.

## Why It Matters

Every non-trivial application handles secrets: database passwords, API tokens, TLS certificates, OAuth client secrets. Hardcoding them in source code or environment variables is insecure and inflexible. A credential store centralizes secret management, enabling rotation, audit logging, and access control.

This library provides the skeleton for a credential management system within the SuperInstance container runtime. It establishes the patterns for secure secret storage that production systems like HashiCorp Vault, AWS Secrets Manager, and Kubernetes Secrets implement at scale.

## How It Works

The library is currently a foundational scaffold with a `main` entry point, designed to be extended with credential storage backends. The intended architecture follows the standard pattern for secret management:

- **Encryption at rest** — Credentials are never stored in plaintext; they're encrypted using a master key (typically managed via a KMS or hardware security module)
- **Namespaced access** — Secrets are organized by namespace/path (e.g., `database/prod/password`), enabling fine-grained access control
- **Lease-based retrieval** — Secrets are checked out with a TTL, after which they must be renewed or are automatically revoked

In production, the credential store integrates with the container runtime to inject secrets at container creation time — similar to how Kubernetes mounts secrets as files or environment variables in pods.

## Quick Start

```rust
fn main() {
    // The credential store runtime entry point
    println!("credential-store starting...");
    
    // Future: initialize secure backend, register credential providers,
    // and expose a typed API for secret retrieval
}
```

## API

Currently in scaffolding phase. Planned API surface:

- `CredentialStore::new() -> Self` — Initialize with encryption backend
- `store(namespace: &str, key: &str, value: &[u8]) -> Result<(), Error>` — Store an encrypted secret
- `retrieve(namespace: &str, key: &str) -> Result<Vec<u8>, Error>` — Decrypt and return a secret
- `rotate(namespace: &str, key: &str) -> Result<(), Error>` — Trigger key rotation

## Architecture Notes

This library provides credential management for the SuperInstance container runtime. It sits alongside `container-runtime` to provide secrets injection into containers at creation time, following the same pattern as Kubernetes Secrets and Docker Secrets.

See the full architecture: [ARCHITECTURE.md](https://github.com/SuperInstance/SuperInstance/blob/main/ARCHITECTURE.md)

## License

MIT
