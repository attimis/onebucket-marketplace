# OneBucket marketplace

Claude Code / Cowork plugins for [OneBucket](https://www.onebucket.io) storage.

Published by [Attimis Corporation](https://www.onebucket.io/about).

## Install

```
/plugin marketplace add attimis/onebucket-marketplace
/plugin install onebucket@onebucket
```

Accept the default endpoints when prompted, then authorize the two OneBucket connectors:
`onebucket` (object operations) and `onebucket-events` (the storage-event stream).

## Plugins

| Plugin | Description |
|---|---|
| `onebucket` | OneBucket storage and storage-event connectors on single global endpoints, plus a skill for working with objects of any size, including large binary files that exceed inline limits. |

## Prerequisite for large objects

Objects above the inline limit are downloaded by the execution sandbox rather than
returned through the model. This requires sandbox network egress to the storage endpoint.

On Team and Enterprise plans an organization owner sets this under **Organization
settings → Capabilities**: enable *Allow network egress*, and set the domain allowlist to
*All domains* or add the storage hosts explicitly.

See [`plugins/onebucket/README.md`](plugins/onebucket/README.md) for detail.

## Development

Validate before pushing:

```bash
claude plugin validate .
```

Test locally:

```
/plugin marketplace add ./onebucket-marketplace
/plugin install onebucket@onebucket
```

Release process and versioning rules are in [MARKETPLACE.md](MARKETPLACE.md).

## License

[Apache-2.0](LICENSE) — applies to the contents of this repository only: the marketplace
catalog, plugin manifests, skill definitions, and documentation.

It grants no rights in the OneBucket service itself, which is proprietary to Attimis
Corporation and governed by separate terms. See [NOTICE](NOTICE) for the full scope
statement and trademark terms.
