- [Interaction between vscode plugin and leaf on the Leaf view](#interaction-between-vscode-and-leaf-on-the-leaf-view)
  - [Load and display the Leaf view](#load-and-display-the-leaf-view)
  - [Handle actions on the Leaf view](#handle-actions-on-the-leaf-view)
  - [Watch and keep the Leaf view up to date](#watch-and-keep-the-leaf-view-up-to-date)
- [Interaction between vscode plugin and language server on the Legato view](#interaction-between-vscode-plugin-and-language-server-on-the-legato-view)
  - [Load and display the Legato view](#load-and-display-the-legato-view)
  - [Handle actions on the Legato view](#handle-actions-on-the-legato-view)
  - [Watch and keep the Legato view up to date](#watch-and-keep-the-legato-view-up-to-date)

# Interaction between vscode plugin and leaf on the Leaf view

The Leaf view is implemented in the folder `leaf` of vscode pluggin

```sh
$ tree
.
├── api
│   ├── bridge.ts         // Contain leaf bridge commands that interacting with leaf-bridge.py 
│   ├── core.ts           
│   ├── filewatcher.ts    // Keep the Leaf view up to date by watching the leaf files
│   ├── interface.ts      // Create an interface to work with Leaf
│   └── process.ts
└── extension
    ├── packages.ts       // Define Packages view and commands
    ├── profiles.ts       // Define Profiles view and commands
    ├── remotes.ts        // Define Remotes view and commands
    ├── terminal.ts
    └── uiComponents.ts
```

## Load and display the Leaf view
The following example steps will outline how the Packages view (a smaller view inside the Leaf View) is loaded and displayed in vscode pluggin:
1) The class `LeafInterface` (interface.ts) will create an interface to work with Leaf (Set a breakpoint at the function `requestPackages`)
2) The function `send` (bridge.ts) is executed to send a request command to leaf-bridge.py to collect the package information
3) The function `onBridgeResponse` (bridge.ts) is executed to receive the package information from leaf-bridge.py
4) After collecting all packages data, the function `getRootElements` (packages.ts) is executed to display the Packages view

## Handle command on the Leaf view
The following example steps will outline how the `Add to profile...` command (an action on the Leaf view) is handled in vscode pluggin:
1) The function `addToProfile` (packages.ts) is called to execute the `Add to profile...` command
2) The function `createProfile` (core.ts) function is called to execute the `leaf setup` command 
3) When the leaf folder changed, the function `watchLeafFiles` (filewatcher.ts) will detect it and update the Leaf view (following the steps 2, 3, 4 in "Load and display the Leaf view" section)

## Watch and keep the Leaf view up to date
Vscode pluggin uses the function `watchLeafFiles` (filewatcher.ts) to watch and keep the Leaf view up to date.

```Typescript
/**
 * Watch all Leaf files
 */
private async watchLeafFiles() {
    try {
        // Watch leaf workspace folder
        // Emit fileChanged event when something change in 'leaf-data' folder or 'leaf-workspace.json' file
        this.watch({
            path: getWorkspaceFolderPath(),
            callbacks: [this.notifyFilesChanged],
            depth: 2,
            filters: [LEAF_FILES.DATA_FOLDER, LEAF_FILES.WORKSPACE_FILE]
        });

        // Watch leaf config folder
        // Emit fileChanged and packageChanged event when something change
        this.watch({
            path: await this.configFolder,
            callbacks: [this.notifyFilesChanged, this.notifyPackagesChanged],
            filters: [LEAF_FILES.CONFIG_FILE]
        });

        // Watch leaf cache folder
        // Emit packageChanged event when something change in 'remotes' folder,
        this.watch({
            path: await this.cacheFolder,
            callbacks: [this.notifyPackagesChanged],
            depth: 2,
            filters: [LEAF_FILES.REMOTE_CACHE_FOLDER]
        });

        // Watch leaf package folder
        // Emit packageChanged event when something change
        this.watch({
            path: await this.packageFolder,
            callbacks: [this.notifyPackagesChanged],
            depth: 0
        });
    } catch (reason) {
        // Catch and log because this method is never awaited
        console.error(reason);
    }
}
```

# Interaction between vscode plugin and language server on the Legato view

The Legato view is implemented in the folder `legato` of vscode pluggin.

```sh
$ tree
.
├── api
│   ├── core.ts
│   ├── files.ts
│   ├── filewatcher.ts
│   ├── language.ts        // Configure language server
│   ├── mkBuild.ts
│   ├── mkEdit.ts          // Interact with mkedit commands
│   └── toolchain.ts
└── extension
    ├── buildtasks.ts
    ├── debug.ts
    ├── snippets.ts
    ├── statusBar.ts
    └── system.ts          // Define Legato view and commands
```

## Load and display the Legato view
The following steps will outline how the Legato view is loaded and displayed in vscode pluggin:
1) The function `stopAndStartLegatoLanguageServer` will start language server. Then, collecting Legato system information into the variable `data` (DefinitionObject type) to create Legato view 
2) After collecting all Legato system information, the function `getRootElements` (system.ts) is executed to display the Legato view

## Handle actions on the Legato view
The following example steps will outline how the `New app` command (an action on the Legato view) is handled in vscode pluggin:
1) The function `newApplication` (packages.ts) is called to execute the `New app` command
2) The function `newApplication` (mkEdit.ts) function is called to execute the `mkedit create app` command 
3) When the legato files changed, the function `watchFiles` in `legatoLangServ/src/lspClient.ts` (vscode-support) will detect it and update the Legato view (following the steps in "Load and display the Legato view" section)

## Watch and keep the Legato view up to date
The Legato view is updated by langguage server (vscode-support). The function `watchFiles` in `legatoLangServ/src/lspClient.ts` is used to watch and keep the Legato view up to date.

```Typescript
/**
 * Watch all Legato files
 * @param watchPaths: Contain all Legato file will be watched
 */
private watchFiles(watchPaths: string[])
{
    this.clearModifyTimeout();

    if (watchPaths.length > 0)
    {
        let theThis = this;

        this.fileWatcher = fs_watcher.watch(watchPaths);

        this.fileWatcher.on('change',
            function (_path, _stats)
            {
                theThis.clearModifyTimeout();
                theThis.reloadTimer = setTimeout(
                    function ()
                    {
                        theThis.reloadActiveModel();
                        theThis.notifyModelUpdate();
                        theThis.reloadTimer = undefined;
                    },
                    400);
            });
    }
}
```
