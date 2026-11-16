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
android: [<package name>]
windows: [<program file name>]
domains: [<domain suffix>]
asn: [<AS number of the service owner itself>]
cidr: [<subnet, when the owner has no AS of its own>]
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
- no subnet wider than /16 for IPv4, no entry without `evidence`.

## Build and signing

CI builds one `.srs` per app and per `kind` and country, and `manifest.json` with a version that only grows and,
per file, SHA-256, size, `kind` and `country`.

Two keys, as the content and timestamp roles in TUF:

- the **lists key** signs `manifest.json` on a release, after reviewed PRs, with a human approving the job. Its
  public keys are built into the Niko client and raycontrol only and change only with their releases;
- the **freshness key** signs a small `timestamp.json` (the manifest's version and SHA-256, and `expires`) on a
  weekly schedule, committed to the unprotected `timestamp` branch. Its public keys are listed in the signed
  manifest. A leak of it can only keep an old version alive, never change the lists.

## Contributing

Every change to the lists is a PR with `evidence`, reviewed by two people. Third-party lists may be used as input
to CI, never shipped as they are.
