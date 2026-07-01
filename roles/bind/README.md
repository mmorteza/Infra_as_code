# ansible-bind-dns

Deploys a BIND9 **primary/secondary** (master/slave) DNS pair with:

- Forward zones (A/NS records) driven entirely from variables
- Reverse zones (PTR records) driven entirely from variables
- Zone transfers secured with a TSIG key (not open AXFR)
- Automatic NOTIFY from master -> slave
- Works on Debian/Ubuntu (`bind9`) and RHEL/CentOS/Rocky (`bind`)

## Layout

```
ansible-bind-dns/
├── ansible.cfg
├── site.yml                     # entry playbook
├── inventory/
│   └── hosts.ini                # define your master + slave here
├── group_vars/
│   ├── all.yml                  # shared settings (TSIG key, ACLs)
│   └── bind_servers.yml         # zone data (forward + reverse)
└── roles/
    └── bind/
        ├── defaults/main.yml
        ├── vars/{debian,redhat}.yml
        ├── tasks/{main,install,tsigkey,master,slave}.yml
        ├── handlers/main.yml
        └── templates/
            ├── named.conf.options.j2
            ├── named.conf.local.j2
            ├── tsig.key.j2
            ├── zone.db.j2        # forward zone file
            └── zone.rev.j2       # reverse zone file
```

## Quick start

1. Edit `inventory/hosts.ini` — put your primary in `[bind_masters]`,
   your secondary(ies) in `[bind_slaves]`.
2. Generate a real TSIG key and put it in `group_vars/all.yml`
   (see "TSIG key" below) — **do not ship the example key to production**.
3. Edit `group_vars/bind_servers.yml` — define your forward zone(s) and
   matching reverse zone(s).
4. Run it:

```bash
ansible-galaxy install -r requirements.yml   # none needed currently, placeholder
ansible-playbook -i inventory/hosts.ini site.yml
```

## How master/slave is decided

Group membership decides the role, not a manually-set variable:
- Hosts in `bind_masters` get `type master;` zone stanzas and serve as the
  authoritative source.
- Hosts in `bind_slaves` get `type slave;` stanzas, pull from the master's
  IP over TCP/53, and are kept in sync automatically via AXFR/IXFR +
  NOTIFY (the master pushes a NOTIFY on every change, so slaves update in
  seconds rather than waiting out the SOA refresh timer).

You can add more than one slave just by adding more hosts to
`[bind_slaves]` — the master's `allow-transfer` / `also-notify` lists loop
over all of them automatically.

## Authoritative-only by default (no recursion)

`bind_recursion: false` in `group_vars/all.yml` is the default. This pair
will **only** answer for zones it's authoritative for (your forward zone
and, in particular, your reverse/`in-addr.arpa` zones) and will refuse to
act as a recursive resolver or cache for anything else — `recursion no;`,
`allow-recursion { none; };`, `allow-query-cache { none; };` are all set
explicitly, not just left at BIND's defaults.

`bind_allow_query` controls who may ask for your authoritative data —
default `any`, since that's the point of running public-facing reverse
DNS. `bind_allow_recursion` is a separate ACL that only takes effect if
you flip `bind_recursion` back to `true`; keep it scoped to your own
networks if you ever do, so you don't end up running an open resolver.

## TSIG key

Generate one on your control node:

```bash
tsig-keygen -a hmac-sha256 sync-key
```

That prints something like:

```
key "sync-key" {
    algorithm hmac-sha256;
    secret "base64stringhere==";
};
```

Copy the `secret` value into `group_vars/all.yml` as `bind_tsig_secret`,
then **encrypt the file with ansible-vault** before committing:

```bash
ansible-vault encrypt group_vars/all.yml
```

## Adding/changing zones

All zone content — forward records and PTR records — lives in
`group_vars/bind_servers.yml`. Bump the `serial` for any zone you edit
(the role does NOT auto-increment serials, on purpose, so you stay in
control of when a change actually gets pushed to the slave). Re-run the
playbook and the master will reload + NOTIFY the slave automatically.

## Firewall

Slave needs to reach the master on TCP/53 and UDP/53. This role does not
manage firewall rules — add your own `ufw`/`firewalld` tasks, or handle it
via a separate role/security group.
