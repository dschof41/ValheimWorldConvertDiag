# ValheimWorldConvertDiag

Diagnostic BepInEx plugin for tracking down legacy inventory data that causes Valheim 1.0 world conversion to crash.

This was created after a previously healthy vanilla world failed to load after updating to Valheim 1.0 with:

```text
System.NullReferenceException: Object reference not set to an instance of an object

at ItemDrop+ItemData.IsSameType
at Inventory.AddItem
at Inventory.AddTempItem
at Inventory.LoadOld
at ZDOMan.ConvertInventories

World load failed mid-file. Exiting without save. Check backups!
```

The same world loaded correctly on the previous Valheim version, including backups going back about a year.

The problem turned out to be one malformed legacy container inventory.

Since 1.3.0 the plugin also covers a second conversion failure with a different trace:

```text
System.FormatException: The input is not a valid Base-64 string

at System.Convert.FromBase64String
at ZPackage..ctor
at ZDOMan.ConvertContainers

World load failed mid-file. Exiting without save. Check backups!
```

Both failures come from one bad legacy container, but they break in different places. See
[Two separate failure paths](#two-separate-failure-paths).

## What this plugin does

ValheimWorldConvertDiag patches the legacy world inventory conversion process and logs the inventory and ZDO being processed when conversion fails.

It records details such as:

- inventory size
- loaded item count
- last item name
- prefab hash
- stack size
- durability
- inventory grid position
- crafter data
- `ItemData.IsSameType()` state
- `m_shared` null state
- recent items processed in the same inventory
- source ZDO UID
- source ZDO world position
- source prefab hash

It also checks every container's stored `items` string before `ZDOMan.ConvertContainers` runs, and
reports each one that is not valid Base-64 along with its ZDO UID, prefab hash and world position.

The most useful part is the ZDO position.

Instead of manually opening hundreds of chests, you can use the logged coordinates to find the exact world object causing the conversion failure.

## Two separate failure paths

Valheim 1.0 converts legacy containers in two stages, and a malformed container can break either one.

| Stage | Exception | Covered by |
|---|---|---|
| `ZDOMan.ConvertInventories` rebuilding an inventory | `NullReferenceException` in `ItemDrop+ItemData.IsSameType` | the inventory patches, since 1.0.0 |
| `ZDOMan.ConvertContainers` Base-64 decoding an `items` string | `FormatException` in `ZPackage..ctor` | the container pre-scan, since 1.3.0 |

The second one aborts the load before any inventory is rebuilt, so the inventory patches never run and
log nothing useful. The pre-scan walks the same ZDO list first, so the bad container is reported either way.

Check which exception your log shows before reading the rest of this document. The recovery steps are the
same for both.

## Important

This plugin is diagnostic only.

It does not:

- modify world data
- delete items
- skip broken inventories
- suppress the original exception
- automatically repair the world

Always work from a backup.

## Confirmed real-world case

The world used to develop this plugin contained more than 1.4 million ZDOs and had never been modded.

Valheim 1.0 failed while converting this legacy inventory:

```text
Inventory size: 4x4
Loaded items before failure: 6
```

Recent items included:

```text
Wood x50        pos 1,0
Stone x7        pos 3,0
Stone x50       pos 0,1
CharredBone x50 pos 2,1
CharredBone x50 pos 4,1
CharredBone x14 pos 1,1
Wood x38        pos 4,1
```

The suspicious part was that a 4-wide inventory contained items at:

```text
pos=4,1
```

and multiple items appeared to overlap at that same position.

The crash occurred inside:

```text
ItemDrop+ItemData.IsSameType
```

while both `ItemData` objects had:

```text
m_shared=<null>
```

The plugin identified the source ZDO as:

```text
UID: 1:183157
Position: (-866.75, 31.46, -8632.98)
Prefab hash: 328745978
```

That location turned out to be an unimportant chest at an old Ashlands landing camp.

The world was loaded using the previous Valheim version, the offending chests were destroyed, dropped items were removed, and the world was saved again.

After that, Valheim 1.0 converted the world successfully.

## Recommended recovery workflow

Do not try to repair the world directly in Valheim 1.0.

The safest process is:

1. Keep an untouched backup of the original pre-1.0 world.
2. Run Valheim 1.0 with this diagnostic plugin.
3. Let conversion fail normally.
4. Read the logged ZDO coordinates.
5. Switch back to the previous Valheim build.
6. Load the same world on that older version.
7. Travel to the logged coordinates.
8. Remove or empty the offending container/object.
9. Save the world normally.
10. Switch back to Valheim 1.0.
11. Try conversion again.
12. Repeat if another malformed inventory is found.

This avoids editing the save file directly.

## Using the previous Valheim build

Valheim keeps an older stable branch available through Steam.

For the regular game client:

1. Open Steam.
2. Right-click **Valheim**.
3. Select **Properties**.
4. Open the **Betas** tab.
5. Select the previous stable branch, commonly named:

```text
default_old
```

6. Let Steam download the older build.
7. Start Valheim.
8. Load the affected world.
9. Fix the object at the coordinates reported by the plugin.
10. Save and exit cleanly.

When finished:

1. Return to **Properties > Betas**.
2. Switch back to the normal/default branch.
3. Let Steam update to Valheim 1.0 again.
4. Retry the world conversion.

Branch names can change over time, so if `default_old` is no longer present, use whichever Steam beta branch Iron Gate provides for the previous stable build.

## Dedicated server rollback

For a dedicated server, the exact process depends on how the server is installed.

If using SteamCMD, install or update the server using the previous stable branch before loading the world.

Example pattern:

```text
app_update 896660 -beta default_old validate
```

Do not blindly run this against your production server without backing up the server files and world first.

If using Docker, check how your image exposes Steam beta branches. Many Valheim Docker images allow a Steam branch or beta environment variable.

The important part is that the world must be loaded using a Valheim build from before the failing 1.0 conversion code.

## Do not mix converted and unconverted saves

Once Valheim 1.0 successfully converts a world, keep that converted copy separate from the old-format version.

Recommended backup layout:

```text
MyWorld-pre-1.0/
MyWorld-fixed-old-build/
MyWorld-converted-1.0/
```

Do not repeatedly move the same working copy back and forth between old and new versions.

Use copies.

## Dedicated server use

This was developed and tested on a dedicated Valheim server running BepInEx.

Typical plugin location:

```text
/config/bepinex/plugins/ValheimWorldConvertDiag.dll
```

or, depending on your setup:

```text
BepInEx/plugins/ValheimWorldConvertDiag.dll
```

The diagnostic log is written to the BepInEx config directory.

On the Docker setup used during development:

```text
/config/bepinex/ValheimWorldConvertDiag.log
```

## Single-player / local world use

The plugin patches shared Valheim world-conversion classes, including:

```text
ZDOMan
Inventory
ItemDrop.ItemData
```

These are not dedicated-server-only classes.

Because of that, the plugin should also work for:

- single-player worlds
- locally hosted worlds
- player-hosted multiplayer worlds

as long as Valheim is running with BepInEx.

Typical client installation:

```text
Valheim/
└── BepInEx/
    └── plugins/
        └── ValheimWorldConvertDiag.dll
```

The diagnostic log should appear under the normal BepInEx config directory, for example:

```text
Valheim/BepInEx/config/ValheimWorldConvertDiag.log
```

Client-side use has not yet been personally tested, so please open an issue if you confirm it works or find anything different.

## How to use

1. Back up the affected world.
2. Install BepInEx.
3. Place the plugin DLL in the BepInEx plugins folder.
4. Start Valheim using the version that crashes during conversion.
5. Let the world conversion fail normally.
6. Open:

```text
ValheimWorldConvertDiag.log
```

7. Find:

```text
---------------- CONVERSION FAILURE REPORT ----------------
```

8. Look for:

```text
Source ZDO candidate:
```

Example:

```text
Source ZDO candidate:
type=ZDO,
m_uid=1:183157,
m_position=(-866.75, 31.46, -8632.98),
m_prefab=328745978
```

9. Record the X and Z coordinates.
10. Switch to the previous Valheim build.
11. Load the same world.
12. Travel to the reported location.
13. Identify the container or object there.
14. Empty or remove it.
15. Save the world normally.
16. Return to Valheim 1.0.
17. Retry conversion.

If conversion fails again, repeat the process. There may be more than one malformed legacy inventory.

## Teleporting to the bad object

With developer commands enabled, the logged X and Z coordinates can be used with:

```text
goto X Z
```

For example:

```text
goto -867 -8633
```

Using rounded whole numbers first may be easier on older game versions.

If developer commands are not enabled, launch Valheim with:

```text
-console
```

Then open the console in-game and run:

```text
devcommands
```

After that, `goto` can be used.

## If the object is disposable

If the reported location is just an old temporary camp, abandoned storage area, boat, cart, or junk chest, the simplest repair may be to destroy it entirely.

In the confirmed case:

1. The old version was loaded.
2. The offending chests were broken.
3. `removedrops` was run.
4. The world was saved.
5. Valheim 1.0 conversion was retried.
6. Conversion succeeded.

Only do this if you are comfortable losing the contents of that object.

## Building

This project intentionally does not include Valheim, Unity, Harmony, or BepInEx DLLs.

Place the required references in:

```text
lib/
```

Required files:

```text
BepInEx.dll
0Harmony.dll
UnityEngine.dll
UnityEngine.CoreModule.dll
```

Then build with:

```powershell
.\build.ps1
```

or:

```powershell
dotnet build -c Release
```

The output DLL will be under:

```text
bin/Release/netstandard2.1/
```

## Why not automatically repair the inventory?

The plugin intentionally does not attempt automatic repair.

There are too many unknowns around malformed legacy inventory data, and silently rewriting world data could cause more damage than the original problem.

The safer workflow is:

```text
diagnose
locate
repair manually on the old game version
retry conversion
```

## When this plugin may help

This is specifically useful when:

- the world worked before Valheim 1.0
- the same world still loads on the previous Valheim version
- Valheim 1.0 crashes during world conversion
- the stack trace contains some combination of:

```text
ZDOMan.ConvertInventories
Inventory.LoadOld
Inventory.AddTempItem
Inventory.AddItem
ItemDrop+ItemData.IsSameType
```

If the world does not load on the previous Valheim version either, the problem may be unrelated corruption and this plugin may not help.

## Suggested backup strategy

Before doing anything, back up both world files:

```text
world.db
world.fwl
```

Keep at least:

```text
original untouched backup
old-build repair copy
1.0 conversion test copy
```

Do not test on your only copy.

## Notes on the underlying failure

The confirmed case appears to have involved malformed legacy inventory slot data rather than a generally corrupted world.

The container was reported as a 4x4 inventory, but some serialized items used an X coordinate of `4`, which is outside the normal `0-3` range for a 4-wide inventory.

The older Valheim build was still able to load and use that container.

Valheim 1.0's conversion path attempted to reconstruct the legacy inventory, encountered invalid or conflicting slot state, and eventually hit a null reference inside:

```text
ItemDrop+ItemData.IsSameType
```

This is why one bad container can prevent an otherwise healthy world from converting.

The exact original cause of the malformed inventory is unknown.

Possible causes include:

- an old inventory serialization bug
- a save occurring during a stack move or merge
- inventory slot metadata becoming out of sync
- a one-off container state that older versions tolerated

The neighboring containers in the confirmed case were built at the same time and were unaffected, so this does not appear to be tied simply to container age or the game version in which the chest was created.

In the Base-64 variant the damage is in the serialized string itself rather than in the slot data. The
confirmed case was a 312 character `items` string holding a replacement character (`U+FFFD`) at index 164,
which suggests a byte was corrupted and re-encoded as invalid UTF-8 at some earlier point. The old build
decoded what it could and loaded the chest anyway; 1.0 rejects the whole string, and with it the world.

## Troubleshooting

### The plugin does not create a log

Confirm that:

- BepInEx is installed and loading
- the DLL is in the correct `BepInEx/plugins` directory
- the plugin appears in the BepInEx startup log
- the world is actually reaching `ZDOMan.ConvertInventories`

### The log does not contain a `Source ZDO candidate`

Make sure you are using the current plugin version that includes ZDO inventory getter diagnostics.

Older versions of the plugin could identify the failing inventory contents but not its world coordinates.

### The reported coordinates contain nothing

Possible explanations include:

- the object is below terrain
- the object is inside another structure
- the inventory belongs to a cart
- the inventory belongs to a ship
- the object has been partially destroyed
- the object is a temporary or unusual container

Search the immediate area rather than assuming it must be a normal chest.

### The reported Y coordinate is around 5000

The object is inside a dungeon interior. Valheim builds cave and crypt interiors several thousand metres
above the world, at the same X and Z as their entrance, so `goto X Z` lands you on the surface next to the
entrance rather than at the object.

Walk in through the entrance and look for the container inside. Flying up to the reported altitude puts you
outside the interior, where the only thing you will find is its exit door.

`goto` takes two arguments, X and Z. It ignores a third, so a Y value cannot be passed to it.

### Conversion succeeds after fixing one object but fails again

That means there may be multiple malformed inventories.

Run the diagnostic plugin again, get the next ZDO coordinates, switch back to the old build, fix the next object, save, and retry.

## Tested environment

Initial development and successful recovery were performed on:

- Valheim 1.0
- legacy world created before 1.0
- vanilla world
- dedicated server
- BepInEx 5.x
- Docker-hosted Valheim server
- Windows Docker host

The `ConvertContainers` coverage in 1.3.0 was developed and verified on:

- Valheim 1.0.17 dedicated server
- legacy world created before 1.0, 1.15 million ZDOs
- vanilla world
- BepInEx 5.4.23.5 (BepInExPack Valheim 5.4.2350)
- `lloesche/valheim-server` Docker image
- Linux Docker host (Ubuntu)

The same diagnostic approach should apply anywhere the same Valheim world conversion code is used.

## Contributing

If this plugin helps recover your world, please open an issue or discussion with:

- whether you were using single-player or dedicated server
- whether the world was modded
- the relevant stack trace
- the diagnostic failure report
- what kind of object was found at the reported coordinates
- whether removing or emptying it allowed conversion to succeed

Do not upload your entire world save unless you are comfortable sharing it.

## Disclaimer

This is an unofficial diagnostic tool and is not affiliated with Iron Gate AB.

Back up your world before testing anything.

Use at your own risk.
