[![ci](https://github.com/setyamickala/airdrop-agent/actions/workflows/ci.yml/badge.svg)](https://github.com/setyamickala/airdrop-agent/actions/workflows/ci.yml)

# airdrop-agent v4.1

Local airdrop operator: quests, testnets, waitlists, social farming, one policy engine, one JSON
envelope, MCP for other agents. Python 3.11+, macOS/Linux, SQLite + encrypted vault, no server.

New in 4.1: durable Cloudflare/Turnstile clearance, one browser profile per identity that keeps
logins alive across mining and check-ins, encrypted session restore, quest login pre-flight, and a
tested catalog of quest/testnet/whitelist targets.

Read [docs/INTRO.md](docs/INTRO.md) first if this is new. Full manual: [docs/GUIDE.md](docs/GUIDE.md).

## Install

```bash
unzip airdrop-agent-v4.zip && cd airdrop-agent
./install.sh                 # venv + deps + browser
# or: ./install.sh --full    # also Foundry, for contract deploys
aa init                      # generates the vault password, refuses weak defaults
$EDITOR ~/.airdrop-agent/.env
aa doctor                    # must be green before funded work
```

`install.sh` accepts `--no-browser` and `--service`. On Linux without a desktop, set
`BROWSER_HEADLESS=1` and optionally `FLARESOLVERR_URL` for the Cloudflare layer.

## First identity

```bash
aa identity new main --wallets 2          # HD EVM + Solana, encrypted seed
aa email add --identity main --address you@yourdomain.tld --password '…'
aa social login x --identity main --user you --password '…' --totp 'JBSWY…'
aa session warm --identity main
```

Wallets per identity derive from one 24-word seed; `aa wallet new --count 10` adds more
addresses, `aa wallet export --identity main` prints addresses and ciphertext only.

## Quests

```bash
aa quest probe https://example.app/tasks       # headful DOM dump, no guessing
aa quest import config/quests/dow-points.yaml
aa quest validate dow-points
aa quest run dow-points --identity main --dry
aa quest run dow-points --identity main
```

A quest that needs X, Telegram or Discord checks the login first and parks with
`needs_login` instead of failing halfway. Captcha, wallet-connect gates and 2FA also park.

Nineteen ready quests ship in `config/quests/`: points programs, testnet faucet/trade flows and
waitlist/whitelist forms. See [docs/TARGETS.md](docs/TARGETS.md) for what each one does and where
it stops.

Step vocabulary and schema: [docs/QUESTS.md](docs/QUESTS.md).

## Testnets

```bash
aa testnet status
aa testnet faucets --chain base-sepolia
aa testnet claim --chain base-sepolia --identity main
aa testnet deploy --chain sepolia --identity main --kind erc20
aa wallet send --chain sepolia --identity main --to 0x… --amount 0.01   # dry
aa wallet send --chain sepolia --identity main --to 0x… --amount 0.01 --broadcast
```

Every send simulates, caps gas, and only then signs. Dry run is the default.

## Cloudflare and sessions

```bash
aa cf status
aa cf assist --identity main --url https://app.example.com/points   # one visible click, stored
aa session status --identity main
aa session warm --identity main
aa session save --identity main      # encrypted mirror
aa session restore --identity main
```

Clearance is bound to host + user agent, cached for `CF_TTL_MINUTES` (45), and re-injected before
navigation. Native turnstile interaction, provider tokens and FlareSolverr are tried in that order
inside `CF_WAIT_SECONDS`. Details: [docs/CLOUDFLARE.md](docs/CLOUDFLARE.md),
[docs/SESSIONS.md](docs/SESSIONS.md).

## Autopilot

```bash
aa autopilot plan
aa autopilot tick              # radar → social → warm → daily → quests → faucet
aa autopilot daemon            # foreground loop, respects active hours
```

Modules and intervals live in `~/.airdrop-agent/autopilot.yaml`. Discord chat and funded
campaigns are off by default.

## Modes and approvals

```bash
aa mode safe | balanced | turbo
```

- `safe` — everything gates, nothing fires alone.
- `balanced` — testnet inside caps is automatic; mainnet, permits and unknown contracts gate.
- `turbo` — wide caps, auto-allowlist clean domains, hard blocks still apply.

Gates are payload-hash bound and single-use. Approve with the 6-character code from your Telegram:

```bash
aa gate list
aa gate approve 3 K7F2Q9
```

Then re-run the identical command. Changing any field changes the hash and the approval dies.
KILL: `aa kill "reason"` → exit 99 everywhere until `aa unkill`.

## Other surfaces

```bash
aa tg login --identity main --phone +62…      # MTProto, not a bot token
aa tg join --identity main @channel
aa x follow --identity main @project
aa dc login --identity main --user … --password …
aa dc grind --identity main --server project --minutes 30
aa radar add tg @alpha --trust 2
aa radar scan
aa browser --identity main --url https://app.example.com --headful
aa mcp serve                                  # stdio JSON-RPC, 21 tools
```

## Safety model in one paragraph

The library re-checks policy even when the CLI already did. Domain allowlist is separate from
contract allowlist. `eth_sign` always gates; unlimited approvals are detected and can be blocked;
drainer verdicts stop auto-connect. Secrets are redacted in logs, the vault is AES-256-GCM with
scrypt and AAD, `.env` is 0600, and exports are ciphertext plus a SHA-256. More:
[docs/SAFETY.md](docs/SAFETY.md).

## Layout

```
aa/            engine (cli, policy, vault, browser, quest, social, testnet, radar, mcp)
config/        policy, env, faucets, quest examples, personas, radar sources
docs/          guide, safety, sybil, cloudflare, sessions, targets, troubleshooting
agents/        Hermes / Muse / OpenClaw / Claude adapters
scripts/       v3-compatible shims
tests/         offline suite
```

State lives in `~/.airdrop-agent` (override with `AIRDROP_HOME`): DB, vault, profiles, proofs.

## Troubleshooting

`aa doctor` first. Then [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md). Common causes:
missing `VAULT_PASSWORD`, no browser build, RPC 429, a stale profile lock (cleared automatically),
or a selector that the site renamed.

## License

MIT. No code vendored from other airdrop bots; the Cloudflare layer is an independent
implementation of public browser + provider APIs.
