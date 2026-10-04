# ShekAuth V2

A small anti-cheat for Unity games on Quest/Android that catches native mod-menu libraries (`.so` files) and handles the punishment on the server, so players can't just edit their way out of it.

It's made of two files:

- `LibGuard.cs` is the Unity script. It scans for banned libs, deletes them, and reports to PlayFab.
- `ReportBannedLib.cs` is a PlayFab Azure Function. It keeps the strike count, issues the ban, and posts to Discord.

## How it works

On launch (and every few seconds after) LibGuard checks two places:

1. `/proc/self/maps`, which shows what's actually loaded in the running process. This still catches a lib even if someone hid or moved the file afterwards.
2. The app's persistent data folder, cache folder and native library folder, looking for `.so` files whose names are on the banned list.

If it finds one, it deletes the file and calls the `ReportBannedLib` function. The server adds a strike and replies with what to do:

| Strike | What happens |
| --- | --- |
| 1 | Lib is deleted |
| 2 and 3 | Lib is deleted and the player is kicked, and kicked again any time it's detected |
| 4 | PlayFab ban for 1 week, plus a Discord log |
| 5 | Ban for 2 weeks |
| 6 | Ban for 30 days |
| 7 and up | Permanent ban |

A few details worth knowing:

- Only one strike can be added per launch, so one bad session doesn't stack up instantly.
- If a banned lib is already loaded on strike 1, deleting the file won't unload it, so the game closes (you can turn this off with `quitOnFirstStrikeIfLoaded`).
- If the server can't be reached after a few retries, the player gets kicked locally rather than let through.
- If the Discord webhook fails, the ban still goes through.

## Setup

### Unity

1. Make sure the PlayFab SDK is installed in your project.
2. Put `LibGuard.cs` anywhere in `Assets/`.
3. Add it to a GameObject in your first scene, before login or the lobby. It marks itself `DontDestroyOnLoad`.
4. Open the `Banned` list near the top of the script and replace the example names with the real library names you want to block. Use exact file names and only names you're certain belong to mod menus, since a false positive will get a legit player punished.
5. Optional: hook the `onEnforced` event up to something that shows a message in-game. It gets the text that explains why the player is being kicked.

If you use Photon, the script disconnects the player from the room automatically (it compiles that part only when `PHOTON_UNITY_NETWORKING` is defined).

### PlayFab and Azure

1. Create an Azure Function App and add `ReportBannedLib.cs` to it.
2. In the Function App's Configuration, add these application settings:
   - `PLAYFAB_TITLE_ID`: your title ID
   - `PLAYFAB_DEV_SECRET`: your title's secret key
   - `DISCORD_WEBHOOK_URL`: the webhook for your ban-log channel
3. In the PlayFab Game Manager, go to Automation, then Functions, and register the function as an HTTP function using the function's URL. The name has to match `reportFunctionName` in LibGuard (it's `ReportBannedLib` by default).
4. Make sure players log in to PlayFab before the game can report anything. LibGuard waits up to 60 seconds for a login before giving up.

## Configuration

These are all public fields on the LibGuard component:

| Field | Default | What it does |
| --- | --- | --- |
| `scanInterval` | 5 | Seconds between scans after startup |
| `quitOnFirstStrikeIfLoaded` | on | Close the game on strike 1 if the lib was loaded |
| `reportFunctionName` | ReportBannedLib | Name of the PlayFab function to call |
| `maxReportAttempts` | 3 | Retries before it fails closed |
| `waitForLoginSeconds` | 60 | How long to wait for PlayFab login |

## What the Discord log shows

When someone is banned, the embed includes their PlayFab ID and display name, the strike number, ban length, which libs were found, whether they were deleted, whether the lib was loaded in the process, and the device, OS and app version.

## Limitations

This only checks file names, so someone who renames a lib will get past the filename check on disk. It's a deterrent and a clean way to run strikes and bans, not a complete solution. Keeping your banned list updated, and pairing it with server-side checks on player behavior, will do more than any single client-side scan.

## License

MIT
