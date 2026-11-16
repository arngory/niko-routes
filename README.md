# niko-routes

Per-app routing lists for [Niko](https://github.com/arngory/niko): each entry is an app or a service with its
packages, programs, domains and subnets, built into signed sing-box rule sets (`.srs`).

**Status: draft.** The format and the signing model below are a proposal under review with the Niko client and
the raycontrol node. Nothing here is consumed yet. The design and its reasoning live in the Niko repository,
`docs/design/ROUTING-APPS.md`.

## Why apps, not domains

Routing one service by its domains splits it: its own names, CDNs, analytics and address checks never fit one
list, so part of it goes one way and part another, and the service sees two addresses. The unit of routing here
is the app as a whole (by Android package or Windows program). Domains and subnets cover the same service when
another app, such as a browser, reaches it.

## An entry

One file per app or service, `apps/<country>/<id>.yaml`:

```yaml
name: <display name>
country: <ISO 3166-1 alpha-2, lower case>
kind: inside-only        # inside-only | blocked | ads
android:
  - package: <package name>
    cert_sha256: [<SHA-256 of a signing certificate>]   # APK v3 key rotation: list the lineage
windows: [<program file name>]
domains: [<domain suffix>]
asn: [<AS number of the service owner itself>]
cidr: [<subnet, when the owner has no AS of its own>]
cdn: [<CDN provider the service relies on, by name>]
cname: [<allowed CNAME targets of the entry's names>]
evidence: [<link or traffic capture showing the names and addresses>]
```

- `inside-only` — works only from inside the country (it checks the address's country);
- `blocked` — blocked inside the country;
- `ads` — ads and trackers.

## What CI will check

- the schema; conflicts (one name or subnet under two `kind`s) and names shared by many apps;
- AS numbers expanded into subnets from RouteViews (CC BY 4.0, with attribution);
- no subnets of CDNs and shared clouds: an address there serves many sites, so those parts of a service are
  matched by name only;
- no subnet wider than /16 for IPv4 or /32 for IPv6, no entry without `evidence`;
- `cname` lists the CNAME targets an entry's names may point at outside its own `domains`. When a relay resolves
  an entry's name itself, every target in the CNAME chain must be under the entry's `domains` or match `cname`,
  or it refuses; an empty `cname` allows no CNAME outside the entry's own domains. CI keeps `cname` narrow (exact
  targets, not a provider's whole zone where anyone can claim a name) and rejects targets that resolve nowhere or
  point at a resource anyone could claim;
- names are matched on label boundaries, lower case, ASCII only (IDNs as A-labels), without a trailing dot. Sets
  hold no other form. sing-box matches a `domain_suffix` without a leading dot on label boundaries and lowercases
  the host, but does not strip a trailing dot, so the Niko client strips one from a sniffed or FakeIP name before
  its rules;
- a set's tag never ends in `.entries`, `.srs` or `.txt`;
- no suffix whose subdomains third parties control by DNS (shared hosting, cloud zones), and a search for dangling
  CNAMEs pointing at resources anyone could claim: a relay resolves `inside-only` names itself;
- an entry that relies on a CDN names it (`cdn:`), from a closed list of providers kept by CI, with `evidence` and
  review. CI builds those providers' ranges into files of their own (`cdn-<provider>`, `kind: cdn`, no
  `country`), listed in the signed manifest, so a node cannot swap them. Only the relay's check of its own
  resolution reads them; sets, the node's nftables and client rules skip them, and the manifest schema enforces
  it. This does not lift the ban on CDN subnets in sets: a relay lets a resolved address out into a named CDN only
  when it resolved that entry's own name itself;
- every Android package carries its signing certificates (`cert_sha256`). A confirmed client tags a connection with
  its set only when the installed package's signature matches, since a relay lets a confirmed inside-only app out
  to any address: an app with a borrowed package name must not get the tag.

## Build and signing

CI builds one `.srs` per app and per `kind` and country, and `manifest.json` with an `epoch`, a version that only
grows within it (at most 2^53) and, per file, `name`, `sha256`, `size`, `kind`, `country` and `form`. The epoch must
equal the one built into the reader; a release that retires a leaked lists key raises it, and the version starts
again in the new epoch. A release is tagged `m<epoch>.<version>`; the tags are protected from deletion and
rewriting, and a reader checking GitHub for the newest release looks only at its own epoch's tags.

- `kind`: `inside-only` | `blocked` | `ads` | `country` | `cdn`;
- `form`: `srs` (read by the client) | `text` (read by the server).

**Server form of the sets.** Next to each `.srs`, CI builds `<tag>.txt` from the same YAML entries:

- lines `suffix:<domain>` and `cidr:<subnet>`;
- domains in lower case, IDNs as A-labels (punycode), subnets in canonical form (`netip.Prefix.Masked`);
- sorted, no duplicates, UTF-8, LF, no comments.

The server reads only this form and the client only `.srs`. Both come from one commit in one CI run, and both
are listed in the manifest.

**Country sets.** `country-<cc>.txt`, `kind: country`, `form: text` only, with `cidr:` lines alone, merged and
collapsed:

- an address is in a country's set only when **both** sources place it there: the RIR delegations
  (delegated-extended of all five RIRs) **and** IPinfo Lite. An intersection, not a union: a foreign address
  taken as domestic would open a relay to the outside, while the opposite error only sends a connection through
  the node;
- subnets of CDNs and clouds (the same exclusions as for entries), private, special-purpose and bogon ranges are
  removed;
- IPinfo Lite is CC BY-SA 4.0, so a country set is published under CC BY-SA 4.0 with attribution to IPinfo, here
  and in the manifest;
- if a country set changes by more than 5 % of its addresses between versions, CI requires a person's explicit
  mark with a reason in the PR, so a broken source does not slip through.

A relay lets out the destinations of its own country's `inside-only` sets and the addresses in
`country-<its cc>`.

Two keys, as the content and timestamp roles in TUF:

- the **lists key** signs `manifest.json` on a release, after reviewed PRs, with a human approving the job. Its
  public keys are built into the Niko client and raycontrol only and change only with their releases;
- the **freshness key** signs a small `timestamp.json` (the manifest's version and SHA-256, the signing time and
  `expires`, at most 14 days later) every day, committed to the unprotected `timestamp` branch. The key lives in
  a GitHub environment open only to the `main` branch, not in repository secrets; the job itself requires
  `github.event_name == 'schedule'`, and the workflow has no `pull_request_target` or `workflow_dispatch` trigger.
  Its public keys are listed in the signed manifest. A leak of it can only keep an old version alive, never change
  the lists.

The manifest may carry several signatures, one `.minisig` per key id; a reader accepts it if any trusted lists key
verifies. A reader checks the manifest first (the lists key, its epoch, a version not below the one it accepted in
that epoch), then `timestamp.json` under the freshness keys that manifest lists.

## Contributing

The lists are maintained by the Niko team; the lists key is held by the owner. Outside PRs are welcome within these
limits:

- a PR from outside may touch only entries under `apps/`; CI rejects anything else (workflows, keys, scripts,
  country sets, exclusions), and `CODEOWNERS` puts every other path under the maintainers;
- every change is a PR with `evidence`, reviewed by two maintainers;
- `inside-only` entries get the strictest review: they send traffic past the foreign exit, directly or through a
  relay in the user's country;
- third-party lists may be used as input to CI, never shipped as they are.

## Setting up the keys (owner)

- lists key: `minisign -G` on an offline machine; the secret key goes into a GitHub environment that requires the
  owner's approval for each run; the public key goes into `keys/lists.pub` here and into the Niko and raycontrol
  builds;
- freshness key: `minisign -G` the same way; the secret key goes into a separate environment open to `main` only;
  its public key is listed in `manifest.json`.
