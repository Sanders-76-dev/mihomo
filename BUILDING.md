# Building the fleet mihomo binary

This fork adds one feature on top of upstream mihomo (branch `Meta`):
`PUT /configs/file` — validate + atomically persist + apply a full config
sent inline over the external-controller API (see `hub/route/configs.go`).
The fleet panel (`mihomo-fleet`) uses it to push configs to nodes.

## Why the `with_gvisor` tag matters

The production build uses `-tags with_gvisor`. **Do not drop it** if any node
uses a WireGuard outbound (WARP): the userspace WireGuard stack requires
gVisor. The Makefile targets already include it.

## Quick build (local, same OS/arch)

```bash
go build -tags with_gvisor -trimpath -o mihomo .
```

## Versioned release build (recommended)

The Makefile embeds `Version` / `BuildTime` via ldflags and gzips the result:

```bash
make VERSION=v1.19.29-fleet linux-amd64     # → bin/mihomo-linux-amd64-v1.19.29-fleet.gz
make VERSION=v1.19.29-fleet linux-arm64
```

`VERSION` is derived from `git describe --tags` when omitted (push the tag
`v1.19.29` first, or pass `VERSION=` explicitly as shown).

Verify the endpoint exists in the built binary (run anywhere):

```bash
gzip -dc bin/mihomo-linux-amd64-*.gz > /tmp/mihomo && chmod +x /tmp/mihomo
/tmp/mihomo -v
# then, against a running instance:
curl -X PUT http://127.0.0.1:9090/configs/file \
  -H 'Authorization: Bearer <secret>' -H 'Content-Type: application/json' \
  -d '{"payload": "...full config.yaml..."}'
# 204 = persisted + applied; 404 on stock builds (endpoint absent)
```

## Deploying to a node

Replace the stock binary, keep the config and service untouched:

```bash
scp bin/mihomo-linux-amd64-*.gz root@node:/tmp/
ssh root@node 'systemctl stop mihomo && gzip -dc /tmp/mihomo-linux-amd64-*.gz > /usr/local/bin/mihomo && chmod +x /usr/local/bin/mihomo && systemctl start mihomo'
```

The config format is unchanged — a stock config works as-is.

## CI

`.gitlab-ci.yml` (GitLab only, not mirrored to GitHub) builds
`linux-amd64` / `linux-arm64` on every push to `main` and exposes the gzipped
binaries as job artifacts.

## Syncing with upstream

```bash
git remote add upstream https://github.com/MetaCubeX/mihomo.git  # once
git fetch upstream Meta
git rebase upstream/Meta feat/configs-file-persist   # the patch is 1 file, ~75 lines
```
