# Install the corpus Binary on the Dagster LXC

Install and verify the pinned `corpus` release (Rust binary + dataset configs) on
the [Dagster LXC](../projects/eve-industry-corpus.md), then prove it with an
end-to-end ingest smoke test. This is the runtime step of phase 3 in the
[eve-industry-corpus build order](../projects/eve-industry-corpus.md#build-order),
following [Deploy the Dagster Orchestration LXC](deploy-dagster-lxc.md).

The binary is a fully static `x86_64-unknown-linux-musl` build — it runs on the
LXC, never on the Windows workstation. All commands run inside the container as
`root` (the LXC has no `sudo`); the smoke test runs as the `corpus` service
account, the same identity Dagster uses.

Pinned to **v0.1.3** throughout. Bump the `VERSION` value to install a newer release.

## Prerequisites

- The Dagster LXC stood up per [Deploy the Dagster Orchestration LXC](deploy-dagster-lxc.md),
  including the `corpus` service account (UID 1000, primary group 988)
- Outbound HTTPS from the LXC to `github.com` (release pull) and `data.everef.net`
  (ingest source) — pull-only, no inbound
- `rclone` on PATH: `apt-get update && apt-get install -y rclone`. The binary
  shells out to `rclone lsjson` (via the config-free `:http:` backend) to list EVE
  Ref for `everef missing-partitions` / `everef list` — the sensor depends on it.
  `ingest` fetches over plain HTTPS and does not need it, so a missing `rclone`
  surfaces only when the availability sensor first runs.
- A GitHub fine-grained PAT, generated once (see below)

### Create the fine-grained PAT

The release repository is private, so `gh` needs a token. Use a fine-grained PAT
scoped to the minimum, not a classic `repo`-scope token.

- **GitHub → Settings → Developer settings → Fine-grained tokens → Generate new token**
- **Resource owner**: the repository owner · **Repository access**: *Only select
  repositories* → `eve-industry-corpus`
- **Permissions → Repository → Contents: Read-only** — nothing else
- Name `dagster-lxc-corpus-release-ro`, with a description recording host, purpose,
  and scope

## 1. Install gh and authenticate

Debian 13 has no `gh` by default. Install it from GitHub's official apt repository:

```bash
apt-get update
apt-get install -y wget
mkdir -p -m 755 /etc/apt/keyrings
wget -nv -O- https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  | tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null
chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
  | tee /etc/apt/sources.list.d/github-cli.list > /dev/null
apt-get update
apt-get install -y gh
```

Authenticate with the fine-grained PAT via `--with-token`:

```bash
PAT='github_pat_...'                              # the fine-grained PAT
printf '%s' "$PAT" | gh auth login --with-token
unset PAT
gh auth status
```

> [!NOTE]
> If `GH_TOKEN` (or `GITHUB_TOKEN`) is set in the environment, `gh` uses that
> value directly and refuses both `gh auth logout` and `--with-token` storage.
> Clear it first (`unset GH_TOKEN`) so `gh` manages stored credentials. A prior
> `gh auth login` via browser device-flow stores a broad `repo`-scope token — run
> `gh auth logout --hostname github.com` (with `GH_TOKEN` unset) to remove it
> before logging in with the fine-grained PAT.

### Verify the token scope

```bash
gh release view v0.1.3 --repo jasperholtjer/eve-industry-corpus --json tagName --jq .tagName
```

`gh auth status` shows the active token as `github_pat_...` with `Token scopes:
none` — a fine-grained PAT reports no classic scopes, which is correct. The
release view prints `v0.1.3`. A fine-grained PAT retains read-only access to
*public* repositories regardless of selection, so reachability of an unrelated
public repo is expected and not a scope leak; only access to an unrelated
*private* repo would indicate an over-broad token.

## 2. Download and install the pinned release

Download the binary, dataset bundle, and checksums; verify before installing:

```bash
VERSION=v0.1.3

gh release download "$VERSION" --repo jasperholtjer/eve-industry-corpus \
  --pattern 'corpus-*-linux-musl' --pattern 'corpus-datasets-*.tar.gz' \
  --pattern SHA256SUMS --dir /tmp/corpus-dl --clobber

( cd /tmp/corpus-dl && sha256sum -c SHA256SUMS )   # every line must report OK

install -m 0755 /tmp/corpus-dl/corpus-*-linux-musl /usr/local/bin/corpus

mkdir -p /usr/local/share/corpus
tar -xzf /tmp/corpus-dl/corpus-datasets-*.tar.gz -C /usr/local/share/corpus

rm -rf /tmp/corpus-dl
```

The binary lands at `/usr/local/bin/corpus` (root-owned, world-executable) and the
dataset YAML configs at `/usr/local/share/corpus/datasets/` (e.g.
`market-history.yaml`). Both are readable by the `corpus` account.

## 3. Configure the dataset directory and PATH

`corpus` resolves its dataset configs from the `--datasets-dir` flag, then
`CORPUS_DATASETS_DIR`, then a workspace `datasets/` dir (dev builds only). An
installed binary must set the flag or the env var. Persist the env var and add
`/usr/local/bin` to `PATH` for login shells:

```bash
cat > /etc/profile.d/corpus.sh <<'EOF'
export CORPUS_DATASETS_DIR=/usr/local/share/corpus/datasets
case ":$PATH:" in *:/usr/local/bin:*) ;; *) export PATH="/usr/local/bin:$PATH";; esac
EOF
chmod 0644 /etc/profile.d/corpus.sh
```

> [!IMPORTANT]
> `/etc/profile.d/*.sh` is sourced by login shells only. A non-login shell (such
> as `pct enter`) inherits neither the env var nor the extended `PATH` — invoke
> the binary by absolute path there. The Dagster systemd units do not source it
> either; they must set `Environment=CORPUS_DATASETS_DIR=...` and call `corpus` by
> absolute path. That wiring belongs to the `eve-industry-orchestration` repository.

## 4. Verify the install

```bash
/usr/local/bin/corpus --version     # corpus v0.1.3  (== the release tag)
ldd /usr/local/bin/corpus           # statically linked / not a dynamic executable
/usr/local/bin/corpus --help
```

The version matches the pinned tag, and `ldd` confirms the static musl build (no
runtime library dependencies).

## 5. End-to-end ingest smoke test

Run one snapshot through ingest → verify → state query as the `corpus` account,
writing to a throwaway local sink (not the NFS share). The login shell (`su -`)
sources `/etc/profile.d/corpus.sh`, so `PATH` and `CORPUS_DATASETS_DIR` are set.

```bash
su - corpus -c '
  corpus ingest --dataset market-history --date 2024-01-15 --sink-path /tmp/corpus-test
  corpus verify --dataset market-history --date 2024-01-15 --tier silver --sink-path /tmp/corpus-test
  find /tmp/corpus-test -type f | sort
  corpus state query --sink-path /tmp/corpus-test \
    --sql "SELECT dataset,tier,partition_key,rows,retention_class FROM partitions"
'
```

Expected result:

- Ingest fetches `market-history-2024-01-15.csv.bz2` from `data.everef.net` and
  writes **52037 rows** to `silver/market-history/year=2024/month=01/day=15`
- The partition holds the full contract: `data.parquet`, `_INDEX.json`, `_DONE`
- Verify reports `ok` (sha256 cross-check) for the date
- The state DB returns one row: `market-history`, tier `silver`, partition_key
  `date=2024-01-15`, `rows` 52037, retention_class `validated`

Remove the throwaway sink once satisfied:

```bash
rm -rf /tmp/corpus-test
```
