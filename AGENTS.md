# Sangria playbooks agent guide

Keep this file and `README.md` current when the schema, installation flow, catalog layout or compatibility status changes.

## Repository purpose

This is the official recipe catalog for Sangria. YAML files describe unattended setup through a native ARM64 Wine/FEX runtime on macOS. The app can also load trusted third-party catalogs and local YAML files.

Categories currently live under:

- `applications/`
- `stores/`
- `games/` when actual game recipes are added

`index.json` and `index.json.sig` are generated artifacts that are committed for clients. Edit YAML recipes, not the index by hand.

## Licensing and restricted components

- License rules apply to copied recipes and scripts as well as application code. Before borrowing copyrightable material, identify the upstream license and preserve all copyright and license notices. GPL/LGPL borrowing is allowed only when every obligation is met, including copyleft, license-text and corresponding-source requirements. Create or update `CREDITS.md` with the upstream URL, exact revision, license, copied or modified files and distribution obligations; attribution alone does not satisfy GPL. If the resulting licensing is incompatible or unclear, stop and contact the project owner/maintainer (`@kira-we1ss` on GitHub) before combining the material.
- A package that requires affirmative EULA or license acceptance must not be installed as if it were fully silent. The user must see the license and explicitly accept or decline it before the relevant step. Never pre-accept, auto-click, hide or bypass a license through silent flags, registry values or playbook metadata. `unattended: true` must not conceal required consent. Implement a native host consent flow first, use the vendor installer's own prompt, or leave the recipe unsupported.
- If redistribution or automatic downloading is prohibited, do not commit the component or expose it as a normal `download` asset, and do not use mirrors to bypass the restriction. Provide the official download page and manual instructions in the README and recipe presentation. Use `localInstaller` or implement a user-provided asset override so Sangria can accept a user-selected download, optionally detect it in an explicitly chosen folder, validate it where possible, copy it into the playbook's isolated working directory and install from that local copy.
- Recheck license terms and redistribution permissions before adding or updating components. Never weaken catalog signatures, source trust or hash verification to work around them.
- Account for every licensing issue across the whole recipe and its dependencies, not only copied code or an individual installer. Do not defer a known licensing problem. If compliance affects anything beyond an isolated code or recipe change, including Sangria's license, distribution model, catalog format, installation flow, runtime bundling or required user interaction, stop and contact the project owner/maintainer (`@kira-we1ss` on GitHub) before implementing it. If another contributor is running the agent, that contributor's approval is not a substitute.

## Contribution policy

- The project is substantially vibe-coded, but that is not a reason to accept more unreviewed generated output. Do not accept `full-slop` contributions (unreviewed, low-effort AI-generated dumps) from anyone except `@kira-we1ss` and `@sawrams`. This exception is account-specific and does not extend to other maintainers or contributors.
- Contributions from people who understand the problem and their changes are welcome. External recipes and changes should be focused, reviewed and understood by their author, explained in plain language and covered by relevant validation. Do not approve an unexamined AI dump or try to rescue an oversized generated change; ask for a smaller, understood contribution instead.
- Follow [No Meat Proxy](https://nomeatproxy.com) as the baseline for contribution and review discussions. Authors must understand, verify and own their work, explain the implementation and its validation, and answer review feedback directly. Do not paste AI-generated questions, answers or review replies instead of engaging with the substance.

## Codebase navigation

- When the `graphify` skill is available, actively use it to understand recipe structure, documentation, dependencies and change impact. If `graphify_out/graph.json` or `graphify-out/graph.json` exists, query the existing graph first for file relationships, cross-recipe dependencies and reviews instead of broad manual searches. Prefer updating an existing graph when it is stale rather than repeatedly rebuilding it.
- Keep generated graphify output untracked. Both `graphify_out/` and the current skill's default `graphify-out/` are intentionally ignored.

## Required checks

Use the Sangria CLI from `/Users/kweiss/Projects/unnamed-app-launcher`:

```bash
for recipe in $(git ls-files --cached --others --exclude-standard '*.yaml'); do
  "/Users/kweiss/Projects/unnamed-app-launcher/.build/out/Products/Debug/sangria" playbook validate "$recipe"
done
"/Users/kweiss/Projects/unnamed-app-launcher/.build/out/Products/Debug/sangria" catalog build /Users/kweiss/Projects/Sangria-playbooks --sign
git diff --check
```

Build Sangria first if the binary is missing. The signing command uses the existing local keychain state. Never print, copy or exfiltrate private signing material.

For local smoke tests, always set an isolated `SANGRIA_HOME` in the app repository's `.artifacts`. Do not uninstall user data. Verify the resulting executable and shortcut, not only YAML syntax.

## Schema rules

Every recipe needs `schema: 1`, a unique ASCII `id`, `name`, `version`, `unattended`, runtime/graphics policies, steps and optional shortcuts. Official recipes must have `unattended: true` and must actually complete without hidden input. Use `wait.forInstallation` when a human intentionally installs a game through the store UI.

Pinned versions need a 64-character SHA-256 for each asset. `version: rolling` allows an omitted hash and downloads on every installation. Do not pin a store to a dated binary merely because it is the current release.

A rolling asset can either use a stable HTTPS `url` or:

```yaml
resolve:
  metadataURL: https://example.com/latest.json
  jsonPath: content.windows.downloadLink
```

Metadata and resolved downloads must remain HTTPS and credential-free. JSON metadata is limited to 1 MiB.

Use `installMethods` when appropriate:

```yaml
installMethods: [download, localInstaller, existingInstallation]
```

Existing installation creates a fresh prefix and a shortcut without moving existing files. A local installer overrides only the `installer` asset; other dependencies still run.

## Actions

- `download`: fetch an asset, resolving metadata first when requested.
- `verify`: check a pinned asset hash.
- `extract`: extract an archive inside the current prefix.
- `prefix.create`: create a new Wine environment.
- `wine.run`: run an executable with optional `args`, `environment`, `timeout` and `expectedExitCodes`.
- `wine.tool`: run a Wine utility such as `winecfg -v win10`.
- `msi.exec`: run an MSI silently with `/quiet /norestart`.
- `dependency.install`: install a supported Winetricks dependency.
- `registry.set`: write a registry value.
- `dll.override`: set a DLL loading order; an empty mode disables the DLL.
- `shortcut.create`: add a program without adding installation steps.
- `store.install`: install a store by ID into the current prefix, skipping its nested `prefix.create`.
- `wait.forInstallation`: pause for the user, who continues after installing through the store UI.
- `wine.kill`: stop this prefix's Wine processes, then continue with later steps.

`prefix.create` supports `profile: hoyoplay` at the playbook root. This applies default metadata, while game-specific work still happens at launch. Do not make users reinstall HoYoPlay to change graphics settings.

## Dependency guidance

Compare against maintained Lutris/Bottles recipes, but test every assumption on the actual runtime. Common dependencies:

- Steam: normally no additional dependency is needed.
- EA app: `corefonts`; optional game-specific dependencies do not belong in the base store recipe.
- Epic: `corefonts`.
- GOG: Visual C++, `corefonts`, `dotnet48`, offline installer and `libglesv2` disabled.
- HoYoPlay: Visual C++ and `corefonts`, Windows 11, DXMT/MSync and launch-time game profiles.

Apply the final Windows version after Winetricks, because verbs such as `dotnet48` can switch the prefix back to Windows 7. Use `winecfg -v`, not only a registry value.

Do not add PhysX to the base EA recipe; its vendor installer currently exits with status 4.

## Git workflow

Work on `dev`. Before every commit, show validation and the changed files and obtain approval. Push only after approval. Updating `main` requires explicit approval as well. Rebuild and sign the index before every recipe commit. Do not commit installers, downloaded assets, private keys, test prefixes or logs.

## Current compatibility status

Keep this section and the README table accurate:

- Steam: silent installation passed.
- EA app: installation passed; `StoreSupport` resolves launchers in versioned folders.
- HoYoPlay: silent installation passed; actual games need compatibility work beyond the base recipe.
- GOG: offline installation completes, but `GalaxyClientService` fails with error 1053 and the renderer crashes.
- Epic: MSI stalls on the current runtime and is not ready.
- PhysX: vendor installer exits with status 4 and needs an alternate extraction path.
