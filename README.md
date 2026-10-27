# proton-rclone-sync

One-way sync from selected local folders to Proton Drive on Linux, using `rclone` and `systemd` user units. No daemon of its own: a path unit and a timer start a `oneshot` service, the service runs `rclone sync` and exits.

## How it works

- `proton-sync.path` watches `~/Documents`, `~/Pictures` and `~/Videos` and starts a sync when something changes there.
- `proton-sync.timer` starts a sync 5 minutes after the user manager starts and then every 30 minutes.
- `proton-sync.service` runs `rclone sync` with the rules from `proton.filter`.

The path unit uses inotify, which is not recursive. It only sees changes to files directly inside the three folders, not inside their subfolders. Changes in subfolders are picked up by the timer.

This is a mirror, not an archive. `rclone sync` makes the Proton side match the local side for everything the filter includes, so a file deleted locally is deleted on Proton too, and files that exist only on Proton inside the included folders (for example uploaded from the web or phone) are removed. `--max-delete 50` stops a run after 50 deletions, it does not prevent them. If you want a backup that never deletes, change `sync` to `copy` in the service file.

## Requirements

- [`rclone`](https://rclone.org/install/) installed
- A Proton Drive remote created with `rclone config` (see the [rclone Proton Drive docs](https://rclone.org/protondrive/))

The service calls `/usr/bin/rclone`. If `which rclone` prints something else, edit `ExecStart` in `proton-sync.service`.

## Installation

1. Clone the repo:

   ```bash
   git clone https://github.com/l0wa1r/proton-rclone-sync ~/.local/share/proton-sync
   cd ~/.local/share/proton-sync
   ```

2. Copy the unit files:

   ```bash
   mkdir -p ~/.config/systemd/user/
   cp proton-sync.service proton-sync.path proton-sync.timer ~/.config/systemd/user/
   ```

3. Copy the filter file and edit it to match your folders:

   ```bash
   mkdir -p ~/.config/rclone/
   cp proton.filter ~/.config/rclone/proton.filter
   ```

4. Enable and start:

   ```bash
   systemctl --user daemon-reload
   systemctl --user enable --now proton-sync.path proton-sync.timer
   ```

User units only run while you have a session. If you want syncs to keep running while logged out, run `loginctl enable-linger $USER`.

## Usage

Follow the logs:

```bash
journalctl --user -u proton-sync.service -f
```

Run a sync by hand:

```bash
systemctl --user start proton-sync.service &
journalctl --user -u proton-sync.service -f
```

## Configuration

### Remote name

The service assumes the remote is called `proton`. Check yours with:

```bash
rclone listremotes
```

If it has a different name, replace `proton:` in `~/.config/systemd/user/proton-sync.service`, then run `systemctl --user daemon-reload`.

### Filter file

`~/.config/rclone/proton.filter` decides what gets synced:

```
- **/.git/**
- **/node_modules/**
- **/.env
- **/.env.*
- **/.cache/**
+ /Documents/**
+ /Pictures/**
+ /Videos/**
- *
```

Rules are checked from top to bottom and the first match wins, so the excludes have to stay above the includes. The last line excludes everything that was not included. Hidden files inside the included folders are synced unless a rule excludes them, which is why `.git`, `.env` and similar are listed at the top.

If you add or remove a folder here, make the same change in `proton-sync.path` and run `systemctl --user daemon-reload`.

### Comparison method

The service uses `--update --size-only`. Proton Drive does not let rclone change a file's modification time after upload, so files are compared by size only. A file that was edited but kept exactly the same size will not be re-uploaded. If that matters to you, replace `--size-only` with `--checksum`. The Proton backend supports SHA1, but every run will then hash all the local files in the filter, which is slow on large folders.

### Rate limiting

`--transfers 1`, `--tpslimit 1` and `--tpslimit-burst 1` keep the request rate low on purpose. Uploading a large `Videos` folder will take a while.

## Troubleshooting

**422 errors from Proton Drive:** usually a corrupted draft stuck on Proton's side. Find the failing file in the logs, delete it from Proton Drive on the web, and retry. The service is tuned to avoid these (`--transfers 1`, `--tpslimit 1`, `--protondrive-replace-existing-draft=true`).

**`--max-delete threshold reached`:** a run found more than 50 files to delete on the Proton side. Nothing further is deleted. Check what changed before running again, since the same limit will hit on every following run until the difference is resolved.

**Sync not triggering:** check that the watchers are active:

```bash
systemctl --user status proton-sync.path proton-sync.timer
```

If a change in a subfolder did not trigger anything, that is expected. Wait for the next timer run or start the service by hand.
