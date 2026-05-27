# Shai-Hulud: Open Sourcing The Carnage

A sandworm surfaces in the desert. It takes everything.

Referenced in: ["TanStack, Mistral AI, UiPath Hit in Fresh Supply Chain Attack"](https://www.securityweek.com/tanstack-mistral-ai-uipath-hit-in-fresh-supply-chain-attack/) (SecurityWeek, May 2026)

*Vibecoded by TeamPCP.*

Drop this into a CI pipeline. The sandworm crawls through your infrastructure, feeding on credentials from every provider it can reach, then exfiltrates through encrypted channels. If it finds npm tokens or OIDC access, it backdoors packages and publishes them. Downstream consumers install the infection. The worm grows.

## 🌐 **Join the Community**

> [!NOTE]
> **Building with AI doesn’t have to be a solo grind.**  
> Join our Discord community to meet other people exploring the latest models, tools, workflows, and ideas: **https://discord.gg/whhrDtCrSS**
>
> We talk about what’s new, what’s useful, and what’s actually worth paying attention to in AI.  
> *And if you want more than conversation,* members also get access to **heavily discounted AI products and services** — including deals on tools like **ChatGPT Plus** and more for just a few dollars.

---

## The Worm's Anatomy

### Phase 1: Surface

The worm wakes up and checks its surroundings. Russian locale? Exit. Already in the target repo (OpenSearch-js)? Activate the OIDC attack. Not in CI? Daemonize into the background, detach from the terminal, keep running.

### Phase 2: The Harvest

Eight providers comb for credentials in parallel.

The filesystem provider reads over a hundred known hotspot files. AWS credentials, SSH keys, cloud configs, wallet files (Bitcoin, Ethereum, Monero, and a dozen more), Discord and Telegram and Signal storage, VPN configurations, Kubernetes service account tokens, Claude settings, npmrc, netrc, shell histories for bash, zsh, Python, MySQL, PostgreSQL, Redis. Nothing is off limits.

The shell provider runs `gh auth token` and dumps every environment variable in the process.

The runner provider targets GitHub Actions workers. It injects a Python script that reads the Runner.Worker process memory through procfs, dumping the entire address space. The script greps the output for GitHub Actions secrets in their JSON representation.

AWS SSM and Secrets Manager enumerate every parameter and secret across all seventeen default regions using SigV4 signed requests. Credential resolution follows the full AWS chain: environment variables, web identity tokens (EKS IRSA), ECS container metadata, EC2 IMDSv2, and every profile in ~/.aws/credentials and ~/.aws/config.

The Kubernetes provider reads the in-cluster service account token or kubeconfig, then queries the K8s API for every namespace and every secret. It decodes base64 payloads and scans for eighteen categories of credentials: AWS keys, GCP service accounts, Azure keys, database connection strings, Stripe tokens, Slack tokens, Twilio keys, SSH keys, Docker auths, and everything else.

The Vault provider authenticates through every gate: token in environment, token file, Kubernetes auth, AWS IAM auth. It lists all KV mounts, enumerates every secret path, and decrypts the values.

### Phase 3: Exfiltration

The worm serializes all harvested data, gzip compresses it, and encrypts it with AES-256-GCM using a random key per batch. It wraps the key with RSA-OAEP using the embedded public key. No one but the C2 operator can read the payload.

The dispatcher tries senders in priority order. First: an HTTPS POST to the primary C2 domain (`git-tanstack.com`). If that domain is unreachable, it searches GitHub for signed commits containing a fallback domain. If even that fails, the worm creates a new public GitHub repository with a Dune-themed name drawn from a word list and commits the encrypted payload as JSON files.

When a token belongs to a user without organization membership, the worm double-base64 encodes the token and embeds it in the commit message. Then it installs a persistence daemon: a systemd service on Linux, a LaunchAgent on macOS. The daemon polls `https://api.github.com/user` every sixty seconds. If the token comes back revoked, it executes a destructive command.

### Phase 4: Propagation

The worm carries three mutators for spreading.

The first targets npm packages through OIDC. When the worm detects it is inside the OpenSearch-js CI pipeline, it requests an OIDC token from GitHub Actions with audience `npm:registry.npmjs.org`, exchanges it for an npm automation token, downloads the latest tarball, injects a malicious dependency into optionalDependencies, generates valid Sigstore provenance (signed through Fulcio, logged in Rekor), and publishes the backdoored version.

The second targets any npm token found during the harvest. It validates the token, determines which packages the token can write to, downloads the latest tarball, injects the worm binary plus a setup script with a preinstall hook, bumps the patch version, and publishes. Every downstream `npm install` executes the worm.

The third targets GitHub tokens without workflow scope. It enumerates all branches in the repository via the GraphQL API, then commits files to every eligible branch: a VSCode task that runs the worm on folder open, a Claude settings hook that runs on session start, and the worm binary itself. Co-authored-by trailers attribute the commits to `claude@users.noreply.github.com`.

### Phase 5: The Dead Man's Switch

The daemon installed during Phase 3 runs for up to twenty four hours. During that window, anyone who revokes the stolen token triggers the handler. The default handler is `rm -rf ~/`.

---

## Build Pipeline

The build pipeline encrypts every sensitive string in the source with a random passphrase unique to each build. The build pipeline rewrites environment variable names into encrypted string lookups. If silent mode is enabled, all logging statements are stripped from the output. The build pipeline runs the final bundle through JavaScript obfuscator with control flow flattening and base64 string encoding.

The build pipeline encrypts assets (shell scripts, Python loaders, config files, public keys) with AES-256-GCM and packs them into a generated module. The worm decrypts and gunzips them at runtime.

Three platform-specific loaders (Bash, Python, Node.js) each download a Bun binary from GitHub releases and execute the worm.

---

## What It Takes

Change the C2 domain in the source and the RSA keypair for encryption. The providers, collectors, dispatchers, senders, mutators, and build pipeline are ready as-is.

```bash
bun install
bun run build:obf
```

---

This material is published for research and educational purposes. The authors do not condone using it against systems without explicit authorization.
