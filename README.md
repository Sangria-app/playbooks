# Sangria Playbooks

Official YAML installer recipes for [Sangria](https://github.com/Sangria-app/Sangria), a native macOS launcher for Windows apps and games.

Recipes are installed through a signed catalog. Sangria also supports trusted third-party repositories and local YAML files.

## Catalog layout

- `applications/` — desktop applications such as Notepad++
- `stores/` — Steam, Epic Games Launcher, GOG Galaxy, EA app and HoYoPlay
- `games/` — future game-specific recipes
- `index.json` — generated catalog index
- `index.json.sig` — generated catalog signature

Do not edit the generated index or signature manually.

## Install methods

A recipe can offer several setup methods:

```yaml
installMethods: [download, localInstaller, existingInstallation]
```

- **download** uses the vendor's installer and runs the full recipe automatically.
- **localInstaller** lets the user choose an existing EXE/MSI, then runs the same recipe. Other dependencies still run.
- **existingInstallation** creates a fresh Wine prefix and a shortcut to an existing executable without moving its files.

Single-method recipes stay one-click. Multiple methods open a native chooser.

## Pinned and rolling versions

Pinned assets include SHA-256:

```yaml
version: 8.9.9.1
assets:
  installer:
    url: https://github.com/notepad-plus-plus/notepad-plus-plus/releases/download/v8.9.9.1/npp.8.9.9.1.Installer.x64.msi
    sha256: e3b0607a092ce2604fbdac027db2c94915995e8e568ebd226f8c5493c2f61a80
```

Store recipes use `version: rolling`. A rolling asset is downloaded on every installation and may use a stable HTTPS URL:

```yaml
version: rolling
assets:
  installer:
    url: https://cdn.akamai.steamstatic.com/client/installer/SteamSetup.exe
```

Or resolve the latest URL from small JSON metadata:

```yaml
assets:
  installer:
    resolve:
      metadataURL: https://remote-config.gog.com/components/webinstaller?component_version=2.0.0
      jsonPath: content.windows.downloadLink
    fileName: setup_galaxy.exe
```

Remote metadata and downloads must be credential-free HTTPS. Pinned verification behavior is unchanged.

## Composite store installation

A game recipe can install its store into the same Wine prefix, wait for the user to install the game through that store, stop the store and continue:

```yaml
steps:
  - prefix.create:
      name: Example Game
  - store.install:
      store: com.valvesoftware.steam
  - wait.forInstallation:
      message: Install Example Game through Steam, then choose Continue in Sangria
  - wine.kill: {}
  - registry.set:
      path: HKCU\Software\Wine
      name: ExampleSetting
      value: enabled
```

`store.install` skips the store's own `prefix.create`, so no second prefix is created. `wait.forInstallation` displays a Continue button. `wine.kill` is a separate action and only stops the current prefix.

## Other actions

- `download`, `verify`, `extract`
- `wine.run` with `args`, `environment`, `timeout`, `expectedExitCodes`
- `wine.tool` for utilities such as `winecfg -v win10`
- `msi.exec` for silent MSI installation
- `dependency.install` for supported Winetricks dependencies
- `registry.set` and `dll.override`
- `shortcut.create` for shortcuts without another installation step

`wine.run` also accepts `workingDirectory`. Windows versions are changed through `winecfg -v`, after dependencies that could change them again.

## Compatibility status

| Recipe | Current status |
|---|---|
| Notepad++ | MSI installation passed |
| Steam | Silent installation passed |
| EA app | Installation passed; versioned launcher paths are resolved at launch |
| HoYoPlay | Silent installation passed; actual games require separate compatibility review |
| GOG Galaxy | Offline installation completes; `GalaxyClientService` fails with 1053 and the renderer crashes |
| Epic Games Launcher | MSI stalls on the current runtime and is not ready |
| NVIDIA PhysX | Vendor installer exits with status 4; not part of the base EA recipe |

A completed installer does not necessarily mean the installed client is fully compatible. Check the table before treating a recipe as ready.

## Validation and publication

From this repository, using the Sangria CLI:

```bash
for recipe in $(git ls-files --cached --others --exclude-standard '*.yaml'); do
  "/Users/kweiss/Projects/unnamed-app-launcher/.build/out/Products/Debug/sangria" playbook validate "$recipe"
done

"/Users/kweiss/Projects/unnamed-app-launcher/.build/out/Products/Debug/sangria" catalog build /Users/kweiss/Projects/Sangria-playbooks --sign
git diff --check
```

Build Sangria first if the binary is missing. Use an isolated `SANGRIA_HOME` for real installation tests and verify the installed executable and shortcut.

Recipes and the catalog are MIT licensed. Downloaded applications, runtimes, installers, stores and Winetricks dependencies have their own licenses and terms.
