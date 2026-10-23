# tailgate

Tailscale for UniFi gateways. Installs and persists `tailscaled` on the UDM, then keeps your tailnet in step with the UniFi controller: networks become subnet routes, fixed clients become DNS, firewall zones become access rules.

**Status:** design stage. No release yet.

## What it does

- Installs the official Tailscale static binaries onto the gateway and supervises `tailscaled` as a child process. No packages, no containers, nothing outside `/data`.
- Advertises your UniFi networks as subnet routes and gets them approved, either through `autoApprovers` in your policy or through the Tailscale API.
- Serves a DNS zone (default `home`) from the gateway's tailnet address, populated from UniFi fixed-IP clients, and registers it as split DNS for the tailnet.
- Writes access rules into your Tailscale policy file from UniFi firewall zones, inside a region it owns and never outside it.
- Runs as a reconcile loop. It reads UniFi and Tailscale state, computes a diff, applies it, and repeats on a timer and on UniFi events.

One static Go binary. No agents on other devices. No second source of truth.

Each reconciler implements the same two-step interface: `Plan` produces a diff from a snapshot of observed state, `Apply` executes it. `tailgate plan` prints the diff without applying. `tailgate sync` applies once. `tailgate run` loops.

## Requirements

- A UniFi gateway running UniFi OS 3.x or newer with SSH enabled: UDM, UDM Pro, UDM Pro Max, UDM SE, UDR, UCG Ultra, UCG Max.
- UniFi Network 8.x or newer. Zone-based firewall mapping needs 9.x; on 8.x the ACL reconciler falls back to per-network rules.
- A Tailscale account with admin access to the tailnet.
- Outbound HTTPS from the gateway to `pkgs.tailscale.com` and `api.tailscale.com`.

## Install

### 1. Prepare the Tailscale side

Add a tag and owner to your policy file. tailgate's node carries this tag.

```jsonc
"tagOwners": {
  "tag:tailgate": ["autogroup:admin"],
},
```

Create an OAuth client at https://login.tailscale.com/admin/settings/oauth with:

- Scopes: `devices:core`, `devices:routes`, `dns`, `policy_file`
- Tags: `tag:tailgate`

Keep the client secret. tailgate uses it both as the node's auth key (tagged, preauthorized) and for API calls.

### 2. Prepare the UniFi side

UniFi OS 9.x and newer: create a local API key under Settings → Control Plane → Integrations.

Older UniFi OS: create a local admin account with the Network application scoped to it. tailgate will log in with username and password.

### 3. Install on the gateway

```sh
ssh root@192.168.1.1

mkdir -p /data/tailgate
curl -fsSL -o /data/tailgate/tailgate \
  https://github.com/michealjroberts/tailgate/releases/latest/download/tailgate_linux_arm64
chmod +x /data/tailgate/tailgate

/data/tailgate/tailgate install
```

`install` does the following and is safe to run again at any time:

1. Detects the gateway model, architecture and UniFi OS version.
2. Downloads the matching Tailscale static tarball from `pkgs.tailscale.com`, verifies the published SHA-256, and unpacks `tailscaled` and `tailscale` into `/data/tailgate/bin`.
3. Creates `/data/tailgate/state` for the tailscaled state file and node key.
4. Writes `/etc/systemd/system/tailgate.service` and enables it.
5. Adds `/data/tailgate` to `PATH` for the root shell.

### 4. Configure

```sh
tailgate config init   # writes /data/tailgate/tailgate.yaml with comments
```

Minimum working config:

```yaml
unifi:
  url: https://127.0.0.1
  api_key: ${UNIFI_API_KEY}
  site: default

tailscale:
  tailnet: example.com
  oauth_client_id: kXXXXXXX
  oauth_client_secret: ${TS_OAUTH_SECRET}
  hostname: udm-pro-max

routes:
  include: ["*"]
  exclude: ["Guest"]

dns:
  zone: home

acl:
  manage: true
  zones:
    Internal:
      src: ["autogroup:member"]
```

Secrets can be inline, `${ENV_VAR}` references, or `file:/data/tailgate/secrets/name`.

### 5. Bring it up

```sh
tailgate up        # joins the tailnet as a tagged, preauthorized node
tailgate plan      # shows what sync would change
tailgate sync      # applies it once
systemctl start tailgate   # or: tailgate run, in the foreground
```

### 6. Verify

```sh
tailgate status
tailgate doctor
```

`status` shows the node's tailnet address, advertised and approved routes, DNS records being served, and whether the policy's managed region matches the current plan.

`doctor` checks persistence of `/data/tailgate`, the systemd unit, `tailscaled` health, IP forwarding, UniFi API reachability and credentials, Tailscale API scopes, and whether the DNS port is bound on the tailnet address.

### Firmware updates

`/data` survives UniFi OS firmware updates. The tailgate binary, config, Tailscale binaries, tailscaled state and node key all live there, so the node keeps its identity and nothing needs re-authorising.

`/etc` does not survive. After a firmware update the systemd unit is gone and tailgate will not start on boot. Fix it with one command over SSH:

```sh
/data/tailgate/tailgate install
```

This re-registers the unit and starts the service. It does not re-download anything already present and does not touch the state directory.

UniFi OS does not offer a supported persistent boot hook. tailgate does not pretend otherwise. A boot hook that survives updates is on the roadmap if a reliable mechanism exists for a given firmware line.

## Configuration reference

```yaml
unifi:
  url: https://127.0.0.1        # local controller; use the LAN address if running off-gateway
  api_key: ""                   # UniFi OS 9+ local API key
  username: ""                  # alternative to api_key
  password: ""
  site: default
  insecure_skip_verify: true    # the local controller uses a self-signed certificate

tailscale:
  tailnet: ""                   # "example.com" or "-" for the default tailnet
  oauth_client_id: ""
  oauth_client_secret: ""
  hostname: ""                  # defaults to the gateway's UniFi device name
  tags: ["tag:tailgate"]
  exit_node: false              # also advertise this node as an exit node
  accept_dns: false             # the gateway keeps its own resolver by default

routes:
  include: ["*"]                # UniFi network names or VLAN IDs; globs allowed
  exclude: []
  ipv6: false
  approve: auto                 # auto | api | manual
                                # auto: write autoApprovers for tag:tailgate (needs acl.manage)
                                # api: approve each route through the API
                                # manual: advertise only

dns:
  enabled: true
  zone: home
  listen: tailnet               # tailnet | all | <address>
  port: 53
  ttl: 60
  include_fixed_clients: true   # clients with a fixed IP in UniFi
  include_dhcp_leases: false    # current leases, by DHCP hostname
  include_gateway: true         # <hostname>.<zone> -> gateway LAN address per network
  register_split_dns: true      # point the tailnet's split DNS for <zone> at this node

acl:
  manage: false
  zones:                        # UniFi firewall zone name -> grant
    Internal:
      src: ["autogroup:member"]
      ip: ["*"]
    Cameras:
      src: ["group:admins"]
      ip: ["tcp:443", "tcp:7441", "tcp:554"]
  networks: {}                  # same shape, keyed by network name; used on Network 8.x
                                # or to override a zone rule for one network

reconcile:
  interval: 5m
  watch_events: true            # subscribe to the UniFi event stream and sync on change
  debounce: 10s

log:
  level: info
  format: text                  # text | json
```

## CLI reference

| Command | Description |
|---|---|
| `tailgate install` | Install or repair Tailscale binaries, state directory and systemd unit. Idempotent. |
| `tailgate config init` | Write a commented starter config. |
| `tailgate config validate` | Parse the config and check credentials against both APIs without changing anything. |
| `tailgate up` | Start `tailscaled` if needed and join the tailnet using the OAuth secret as a tagged auth key. |
| `tailgate down` | Disconnect from the tailnet. State is kept. |
| `tailgate plan` | Snapshot both sides, print the diff for every reconciler, change nothing. |
| `tailgate sync` | Plan and apply once. `--only routes,dns,acl` limits scope. |
| `tailgate run` | Supervise `tailscaled` and reconcile on an interval and on UniFi events. This is what the systemd unit runs. |
| `tailgate status` | Node, routes, DNS records, policy region state. `--json` for machine output. |
| `tailgate doctor` | Diagnostic checks with suggested fixes. |
| `tailgate uninstall` | Stop the service, remove the unit, optionally remove `/data/tailgate`. Leaves the policy file's managed region unless `--purge-acl`. |
| `tailgate version` | Build version, Tailscale binary version, UniFi OS version. |

Global flags: `--config`, `--log-level`, `--dry-run` (on `sync` and `run`).

## Design

### Package layout

```
cmd/tailgate/           CLI entry point and subcommands
internal/config/        config schema, loading, secret resolution, validation
internal/installer/     download, verify, unpack, systemd unit, repair
internal/supervisor/    tailscaled lifecycle, restart on exit, upgrade in place
internal/unifi/         minimal client for the local Network API and event stream
internal/tailnet/       LocalAPI client (this node) and control API client (tailnet)
internal/snapshot/      observed state from both sides, taken once per reconcile
internal/reconcile/     Reconciler interface, planner, applier, diff printing
internal/reconcile/routes/
internal/reconcile/dns/
internal/reconcile/acl/
internal/dnsserver/     authoritative responder for the configured zone
internal/naming/        UniFi name -> DNS label normalisation and collision handling
```

The UniFi client is hand-written against the handful of endpoints tailgate needs: networks, known clients, active clients, firewall zones, device info, and the websocket event stream. It is not a general-purpose UniFi library and does not aim to be one.

### Reconciler interface

```go
package reconcile

type Snapshot struct {
	TakenAt   time.Time
	UniFi     unifi.State
	Tailnet   tailnet.State
	LocalNode tailnet.NodeState
}

type Change struct {
	Kind        ChangeKind // Add, Update, Remove
	Resource    string     // "route", "dns-record", "policy-host", "policy-grant", ...
	Key         string
	Before, After any
}

type Plan struct {
	Reconciler string
	Changes    []Change
}

func (plan Plan) IsEmpty() bool

type Reconciler interface {
	Name() string
	Plan(ctx context.Context, snapshot Snapshot) (Plan, error)
	Apply(ctx context.Context, plan Plan) error
}
```

The planner runs every enabled reconciler against the same snapshot, so a single sync sees one consistent view of both sides. Apply order is fixed: routes, then ACL, then DNS. Routes first because the ACL reconciler may write `autoApprovers` for them; DNS last because its split-DNS registration needs the node's tailnet address, which is stable only once the node is up.

### Routes

Desired state is the set of UniFi networks matching `routes.include` minus `routes.exclude`, mapped to their subnets (IPv4 by default, IPv6 if enabled). Observed state is the node's `AdvertiseRoutes` from the LocalAPI and the approved routes from the control API.

`Apply` sets `AdvertiseRoutes` through the LocalAPI, which is the same mechanism as `tailscale set --advertise-routes`, and then approves according to `routes.approve`. With `approve: auto` the ACL reconciler writes an `autoApprovers.routes` entry for each subnet granting `tag:tailgate`, so approval happens in the control plane with no API call per route. With `approve: api` tailgate approves each route through the devices API. With `approve: manual` it only advertises.

Networks removed in UniFi are withdrawn on the next sync. A network renamed in UniFi keeps its route if the subnet is unchanged.

### DNS

tailgate runs a small authoritative responder for one zone, bound to the node's tailnet address. Records come from the snapshot:

- `<client>.<zone>` → fixed IP, for every UniFi client with a fixed IP assignment.
- `<client>.<zone>` → leased IP, for active DHCP clients, if `include_dhcp_leases` is on. These change more often and carry a shorter TTL.
- `<hostname>.<zone>` → the gateway's address on each network, if `include_gateway` is on.

Names are normalised to DNS labels: lowercase, spaces and underscores to hyphens, everything outside `[a-z0-9-]` dropped, leading and trailing hyphens trimmed. Collisions are resolved deterministically by MAC order and logged at warn level so you can rename the client in UniFi.

With `register_split_dns` on, tailgate sets the tailnet's split DNS for `<zone>` to its own tailnet address through the DNS API. Every device on the tailnet then resolves `nas.home` without MagicDNS extra records and without any per-device config. Devices on the LAN keep using the gateway's normal resolver and are unaffected.

The responder is UDP and TCP, answers only for the configured zone, and returns REFUSED for everything else. It is not a recursive resolver and never forwards.

### ACL

tailgate edits the policy file through the control API with `If-Match` on the ETag, so a concurrent edit in the admin console is never overwritten; the sync fails, logs the conflict, and retries from a fresh read.

It owns exactly three things in the file and nothing else:

1. `hosts` entries named `unifi-<network>`, one per included network, with the subnet as the value.
2. `grants` entries whose `dst` list contains only `unifi-*` hosts. Each configured zone produces one grant: `src` from config, `dst` as the hosts for every network in that zone, `ip` from config.
3. The `autoApprovers.routes` entries for `tag:tailgate`, when `routes.approve` is `auto`.

Everything else in the file is preserved byte for byte, comments included, using HuJSON patching rather than unmarshal and re-marshal. `tailgate plan` prints the policy changes as a unified diff.

Zones come from the UniFi zone-based firewall. On Network 8.x, or for any zone you leave out of config, nothing is written. `acl.networks` lets you target a single network directly, which also overrides the zone rule for that network.

### Supervision and persistence

`tailgate run` starts `tailscaled` as a child with `--state=/data/tailgate/state/tailscaled.state --socket=/data/tailgate/tailscaled.sock --tun=tailscale0`, restarts it with backoff if it exits, and forwards SIGTERM for clean shutdown. There is one systemd unit, for tailgate, and it owns both processes.

`tailgate install` with a newer Tailscale available downloads it alongside the current version, verifies it, and swaps on the next `tailscaled` restart. `tailgate run --upgrade-tailscale` forces that restart.

All mutable state is under `/data/tailgate`:

```
/data/tailgate/
  tailgate                  the binary
  tailgate.yaml             config
  bin/tailscaled            Tailscale static binaries
  bin/tailscale
  state/tailscaled.state    node key and preferences
  state/tailgate.json       last applied plan, for status and drift detection
  secrets/                  optional, 0600, referenced as file: in config
```

### Event-driven sync

With `reconcile.watch_events` on, tailgate subscribes to the UniFi event stream and triggers a sync, debounced, on network configuration changes, client fixed-IP changes, and firewall zone changes. The interval timer still runs as a backstop. A sync is never run concurrently with another; events during a sync queue one follow-up.

## Non-goals

- A UniFi controller replacement, or any UI.
- Installing Tailscale on devices other than the gateway.
- Managing the parts of your Tailscale policy file that tailgate did not write.
- Site-to-site or mesh between multiple gateways. That is a later layer on top of this one.
- Running anywhere other than UniFi OS. It may work on a stock Debian router but that is not tested or supported.

## Roadmap

In rough order, after the core above is solid:

- `tailgate status --watch` and a Prometheus `/metrics` endpoint on the tailnet address.
- IPv6 routes and AAAA records.
- A persistent boot hook for firmware lines where one can be made reliable.
- Multi-gateway: one config describing several UniFi sites, with route and DNS namespaces per site.
- UniFi alerts on tailnet events (node approved, route rejected, policy conflict).

## Uninstall

```sh
tailgate uninstall            # stops and removes the service, keeps /data/tailgate
tailgate uninstall --purge    # also removes /data/tailgate and the node's state
tailgate uninstall --purge-acl   # also removes tailgate's managed region from the policy
```

The node remains in the tailnet admin console until you delete it there.

## License

MIT