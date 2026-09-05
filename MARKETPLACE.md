# Running the OneBucket marketplace

## Layout

```
onebucket-marketplace/            ← git repo root
├── .claude-plugin/
│   └── marketplace.json          ← the catalog
└── plugins/
    └── onebucket/                ← the plugin
        ├── .claude-plugin/plugin.json
        ├── .mcp.json
        ├── skills/onebucket-objects/SKILL.md
        └── README.md
```

Relative sources must start with `./` and resolve from the **marketplace root**, not from
`.claude-plugin/`. Never use `../` — paths outside the marketplace root are rejected.

## Test locally before pushing

```bash
claude plugin validate .
```

Pointed at a marketplace directory this checks the catalog for schema errors, duplicate
plugin names, and path traversal, and validates the `plugin.json` of every local-path
entry. It also warns when an entry's `version` disagrees with the plugin's own.

Then install from the local directory and exercise it:

```
/plugin marketplace add ./onebucket-marketplace
/plugin install onebucket@onebucket
```

## Publish

Push the repo to GitHub. Clients add it once:

```
/plugin marketplace add attimis/onebucket-marketplace
/plugin install onebucket@onebucket
```

Pin to a tag if you want clients on a release rather than `main`:

```
claude plugin marketplace add attimis/onebucket-marketplace@v1.0
```

## Shipping updates

**The trap:** `version` in `plugin.json` pins the plugin. Push new commits without
changing that string and existing users keep their cached copy — Claude sees the same
version and skips them. The plugin currently declares `"version": "0.3.0"`.

Two workable policies. Pick one and hold to it.

| Policy | How | Suits |
|---|---|---|
| **Explicit versions** | Bump `version` in `plugin.json` on every release | Client-facing releases with clear boundaries |
| **Track the commit** | Delete `version` from `plugin.json` | Internal or fast-moving development — every push is an update |

Do **not** set `version` in both `plugin.json` and the marketplace entry. The
`plugin.json` value silently wins, so a stale manifest can mask what you put in the
catalog. The entry in this marketplace deliberately omits it.

Release flow with explicit versions:

```bash
# 1. bump plugins/onebucket/.claude-plugin/plugin.json  →  "version": "0.4.0"
# 2. validate
claude plugin validate .
# 3. commit and push
git commit -am "onebucket 0.4.0" && git push
```

Clients then refresh and update:

```
/plugin marketplace update onebucket
/plugin update onebucket@onebucket
```

`marketplace update` refreshes the catalog; `plugin update` installs the new version.
There is also a background refresh that pulls the marketplace shortly after session
start, but a version bump is still what triggers the actual plugin update.

## Enterprise distribution

Team/Enterprise administrators can also install this marketplace org-wide under
`claude.ai/admin-settings/plugins`, either by connecting the repo for automatic sync or by
uploading a plugin bundle directly. Note that the GitHub-sync route requires the
marketplace repo to be private or internal, so it does not apply to this public repo;
administrators wanting sync should mirror it internally.

## Naming rules that will bite you

- Marketplace `name` is public and users type it: `/plugin install onebucket@onebucket`.
  Each user can register only **one marketplace per name** — adding a second with the same
  name replaces the first.
- A plugin's `name` is its stable identifier. Changing it breaks every existing install.
  To change the display label, add `displayName` and leave `name` alone. If you truly must
  rename, add a `renames` entry mapping old name → new, and treat that map as append-only
  history.
- Names must be kebab-case for the claude.ai marketplace sync, even though the CLI is more
  permissive.

## Adding more plugins later

One marketplace per name, so additional plugins go in the same `marketplace.json`
`plugins` array rather than a second marketplace.

## Useful commands

```bash
claude plugin marketplace list --json      # what's registered, and pinned refs
claude plugin marketplace update           # refresh all catalogs
claude plugin validate ./plugins/onebucket # validate one plugin deeply
claude plugin update onebucket@onebucket --yes   # non-interactive, for scripts
```

Removing a marketplace from its last scope **uninstalls plugins installed from it**. To
refresh without that risk, use `marketplace update`, not `remove` and re-`add`.
