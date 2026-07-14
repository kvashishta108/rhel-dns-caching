# Localized DNS Caching for Mixed RHEL 9 / RHEL 10 Environments

A single Ansible playbook (`dns-caching.yml`) that configures per-host DNS
caching across a mixed estate of RHEL 9 and RHEL 10 systems, selecting the
vendor-supported caching mechanism dynamically based on each host's operating
system major version.

## Scoping Assumption

I interpreted "localized DNS caching" as a **per-host caching resolver on
loopback** — each system caching its own DNS queries locally — rather than
deploying dedicated, centralized caching DNS servers for the environment to
point at. This interpretation follows from the word "localized," and it adds
no new infrastructure dependency or failure domain: each host simply gains a
loopback cache in front of whatever upstream DNS it already uses.

If the centralized-resolver design was the intent, the trade-offs (larger
shared cache and fewer upstream queries, versus a new service dependency,
additional failure domain, and loss of per-host cache locality) are a
discussion I am happy to have.

## Design Rationale

The goal is to follow vendor and industry recommended best practices. The
key finding from researching Red Hat's current documentation is that **the
recommended mechanism differs between the two releases**:

| | RHEL 9 | RHEL 10 |
|---|---|---|
| systemd-resolved status | Unsupported Technology Preview | Fully supported since 10.0 (DNSSEC validation remains Technology Preview) |
| Vendor-supported caching method | NetworkManager dnsmasq plugin (`dns=dnsmasq`) | systemd-resolved stub listener |
| Cache listens on | 127.0.0.1:53 | 127.0.0.53:53 |

**RHEL 9.** Red Hat's RHEL 9 networking documentation identifies
systemd-resolved as a Technology Preview, not supported for production, and
directs users to dnsmasq in NetworkManager as the supported solution for
local DNS caching. With `dns=dnsmasq`, NetworkManager launches and supervises
a private dnsmasq instance bound to 127.0.0.1:53, writes
`nameserver 127.0.0.1` into `/etc/resolv.conf`, and dynamically forwards
queries to the DNS servers defined in the active connection profiles.

References:
- RHEL 9 Configuring and Managing Networking — "Using dnsmasq in
  NetworkManager to send DNS requests for a specific domain to a selected
  DNS server" (and the Technology Preview notice on systemd-resolved in the
  same guide)
- RHEL 9 Release Notes — systemd-resolved listed under Technology Previews

**RHEL 10.** The RHEL 10.0 release notes state that systemd-resolved,
previously a Technology Preview, is fully supported — with the exception of
DNSSEC validation, which remains a Technology Preview. The playbook therefore
enables the systemd-resolved stub resolver with caching, integrates it with
NetworkManager (`dns=systemd-resolved`), and symlinks `/etc/resolv.conf` to
`/run/systemd/resolve/stub-resolv.conf`, the mode of operation recommended by
systemd upstream. Because DNSSEC validation is not yet production-supported,
it is disabled by default (`DNSSEC=no`) and exposed as a variable
(`dns_cache_resolved_dnssec`) so it can be set to `allow-downgrade` for
evaluation.

**Deliberate non-goals.** The playbook does not override upstream DNS servers
on either OS. Both mechanisms consume the DNS servers already present in the
NetworkManager connection profiles (DHCP-provided or static), so the playbook
is safe to roll out without touching existing resolver topology. It also
stops and disables competing port-53 services (standalone dnsmasq, unbound,
nscd, and — on RHEL 9 — systemd-resolved) to prevent port conflicts.

## How It Works

- Facts are gathered explicitly (`gather_facts: true`) because all branching
  keys on `ansible_facts['distribution_major_version']` and
  `ansible_facts['os_family']`.
- A `pre_tasks` assertion rejects any host that is not RedHat-family major
  version 9 or 10 before anything is modified. Because the check uses
  `os_family`, the playbook covers RHEL and its rebuilds (Rocky, AlmaLinux)
  identically.
- Each OS version has its own `block:` with a block-level `when:` condition;
  hosts on the other version skip it entirely. Each block has a `rescue:`
  section that fails with a pointer to the relevant journal.
- All service reconfiguration is driven by handlers, notified only when file
  content actually changes — the playbook converges to `changed=0` on repeat
  runs. NetworkManager is **reloaded**, not restarted, so interfaces (and the
  SSH session driving the run) are never disrupted.
- Handlers are flushed (`meta: flush_handlers`) before a shared verification
  stage that waits for the loopback listener (127.0.0.1 on 9, 127.0.0.53 on
  10), asserts `/etc/resolv.conf` points at the loopback cache, and performs
  a live lookup via `getent ahosts` — exercising the same glibc/NSS path real
  applications use, rather than querying the daemon directly with dig.

## Repository Contents

- `dns-caching.yml` — the playbook (single file, `ansible.builtin` modules
  only; no external collections required)
- `inventory.ini` — lab inventory
- `ansible.cfg` — lab configuration (inventory path, privilege escalation)

## Usage

    ansible-playbook -i inventory.ini dns-caching.yml -K

Tunables (see `vars:` at the top of the playbook):

| Variable | Default | Purpose |
|---|---|---|
| `dns_cache_dnsmasq_cache_size` | `10000` | dnsmasq cache entries on RHEL 9 (NM's embedded default is 400) |
| `dns_cache_dnsmasq_negcache` | `true` | Cache NXDOMAIN answers on RHEL 9 |
| `dns_cache_resolved_cache` | `"yes"` | systemd-resolved `Cache=` setting |
| `dns_cache_resolved_dnssec` | `"no"` | DNSSEC validation (Technology Preview on RHEL 10); set `"allow-downgrade"` to evaluate |
| `dns_cache_resolved_dnsovertls` | `"no"` | DNS over TLS to upstreams |
| `dns_cache_test_hostname` | `redhat.com` | Name resolved by the verification stage |

## Test Environment and Results

Tested on a ten-node lab — one control node (which is also a managed node)
and nine further managed nodes — split 4× Rocky Linux 9.8 / 6× Rocky Linux
10.2:

| Host | OS | Role | Branch exercised |
|---|---|---|---|
| ansible-3 | Rocky Linux 9.8 | managed node | NetworkManager + dnsmasq (127.0.0.1) |
| rocky9-1 | Rocky Linux 9.8 | managed node | NetworkManager + dnsmasq (127.0.0.1) |
| rocky9-2 | Rocky Linux 9.8 | managed node | NetworkManager + dnsmasq (127.0.0.1) |
| rocky9-3 | Rocky Linux 9.8 | managed node | NetworkManager + dnsmasq (127.0.0.1) |
| ansible-1 | Rocky Linux 10.2 | managed node | systemd-resolved (127.0.0.53) |
| ansible-2 | Rocky Linux 10.2 | control node + managed node | systemd-resolved (127.0.0.53) |
| rocky10-1 | Rocky Linux 10.2 | managed node | systemd-resolved (127.0.0.53) |
| rocky10-2 | Rocky Linux 10.2 | managed node | systemd-resolved (127.0.0.53) |
| rocky10-3 | Rocky Linux 10.2 | managed node | systemd-resolved (127.0.0.53) |
| rocky10-4 | Rocky Linux 10.2 | managed node | systemd-resolved (127.0.0.53) |

Rocky Linux was used as a bug-for-bug RHEL rebuild; the playbook keys on
`os_family` and the major version fact, so behavior on RHEL 9.x/10.x is
identical.

Run sequence and outcomes:

1. **`--check` dry run** — reports expected spurious failures in each
   branch: check mode simulates the package install without performing it,
   so the subsequent service-management task cannot find the not-yet-installed
   unit and the block's `rescue:` fires. This is a known limitation of check
   mode with install-then-manage dependencies (avoidable with
   `when: not ansible_check_mode` guards, at the cost of the dry run covering
   less). It also demonstrated that the rescue/error-reporting path works.
2. **First real run** — full success; `changed=8` on RHEL 10-family
   managed nodes / `changed=4` on RHEL 9-family managed nodes; handlers
   fired once per host before verification; live lookups succeeded on both
   listener addresses.
3. **Repeat run** — `changed=0` on all hosts, no handlers fired,
   verification still green: the playbook is idempotent.
4. **Fleet rollout** — after the initial three-node validation, seven
   additional managed nodes were bootstrapped (dedicated automation account,
   SSH key, sudoers) and the playbook was applied fleet-wide. Final recap:
   ten hosts, `changed=0`, `failed=0`, with per-branch task counts identical
   within each OS group (`ok=13/skipped=8` on every 10.x node,
   `ok=12/skipped=9` on every 9.x node) — the fingerprint of consistent
   fact-based branching with no per-host drift.

Post-run manual verification:

    # RHEL 9 managed nodes
    cat /etc/resolv.conf            # nameserver 127.0.0.1
    ps aux | grep [d]nsmasq         # NetworkManager-supervised instance
    time dig redhat.com @127.0.0.1  # second query answered from cache

    # RHEL 10 managed nodes
    ls -l /etc/resolv.conf          # symlink -> /run/systemd/resolve/stub-resolv.conf
    resolvectl status               # per-link DNS, cache settings
    resolvectl statistics           # cache hit counters

A notable observation from testing: Rocky 10.2 does **not** ship
systemd-resolved installed by default — the playbook's package task is doing
real work on RHEL 10-family hosts, not a formality.

## Rollback

To revert a host: remove `/etc/NetworkManager/conf.d/00-dns-caching.conf`
(and on RHEL 9, `/etc/NetworkManager/dnsmasq.d/00-cache-tuning.conf`; on
RHEL 10, `/etc/systemd/resolved.conf.d/10-dns-caching.conf`), on RHEL 10
`systemctl disable --now systemd-resolved` and delete the `/etc/resolv.conf`
symlink, then `systemctl reload NetworkManager` to regenerate a standard
resolv.conf from the connection profiles.
