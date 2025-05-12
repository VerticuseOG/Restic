# Using the Minecraft Server Restic Backup System

## 1. Overview of the Backup System

This server is equipped with an automated backup system for the Minecraft world using **Restic**.

* **Automation:** Backups are performed **daily at 6:00 AM IST (Server Time)** by a cron job running the `/home/ubuntu/scripts/minecraft_server_backup_script_automatic.sh` script.
* **Process:** The script automatically stops the Minecraft server (using `mcrcon`), performs the Restic backup, applies a retention policy to old backups, and then restarts the Minecraft server (in a `screen` session).
* **Storage:** Backups are stored locally in an encrypted and deduplicated Restic repository located at `/home/ubuntu/minecraftbackups`.
* **Versioning:** Multiple snapshots (point-in-time backups) are kept, allowing restoration from different times.
* **Retention:** The script automatically manages snapshot retention (typically keeping the last 24 snapshots if run hourly (which it is not, so this would be the last 24 days), 7 daily distinct snapshots, and 4 weekly distinct snapshots).
* **Logging:** Backup operations are logged by the script, in a file `/home/ubuntu/logs/minecraft_backup.log`.

## 2. Essential Prerequisite: Setting Up Your Environment for Restic Commands

To run any `restic` command manually, you first need to tell Restic where your repository is and how to access its password. The automated script does this with internal configuration. For manual admin use:

1.  **Restic Repository Location:** Defined in the backup script (e.g., `/home/ubuntu/minecraftbackups`).
2.  **Restic Password File:** Defined in the backup script (e.g., `/home/ubuntu/.config/restic/.restic_password`). This file contains the encryption password for the repository.

**Before running the Restic commands below, set these environment variables in your terminal session:**

```bash
export RESTIC_REPOSITORY="/home/ubuntu/minecraftbackups"
export RESTIC_PASSWORD_FILE="/home/ubuntu/.config/restic/.restic_password"
```
*Alternatively, you can add `--repo YOUR_REPO_PATH` and `--password-file YOUR_PASSWORD_FILE_PATH` to every `restic` command, but using environment variables is often easier for a session.*

## 3. Common Restic Commands for Server Admins

### 3.1. Listing Backup Snapshots

* **Simple:**
    ```bash
    restic snapshots
    ```
* **Detailed Instructions:**
    This command lists all available backup snapshots (point-in-time backups) stored in the repository.
    The output will show:
    * `ID`: A short unique identifier for each snapshot. You'll use this ID for restores or inspecting specific snapshots.
    * `Time`: The date and time the backup was taken.
    * `Host`: The hostname of the server where the backup was created.
    * `Tags`: Any tags applied to the snapshot (the automated script may or may not apply tags).
    * `Paths`: The original directories that were backed up in that snapshot (e.g., `/home/ubuntu/minecraft`).

### 3.2. Viewing Contents of a Snapshot

* **Simple (Latest Snapshot):**
    ```bash
    restic ls latest
    ```
* **Simple (Specific Snapshot):**
    ```bash
    restic ls <SNAPSHOT_ID>
    ```
* **Detailed Instructions:**
    This command allows you to see the file and directory structure within a specific snapshot without actually restoring any data.
    * Replace `<SNAPSHOT_ID>` with an actual ID from the `restic snapshots` output.
    * You can also list contents of a sub-directory within a snapshot:
        ```bash
        restic ls latest -- /world/playerdata
        ```
    * The paths are relative to the root of what was backed up (e.g., if `/home/ubuntu/minecraft` was backed up, `/world/playerdata` refers to `/home/ubuntu/minecraft/world/playerdata` in the source).

### 3.3. Restoring Backups

**CRITICAL SAFETY PRECAUTIONS FOR RESTORING:**
* **ALWAYS restore to a NEW, EMPTY, TEMPORARY directory first.** Inspect the restored files thoroughly.
* **NEVER restore directly over your live Minecraft server directory unless you are certain and have taken a separate, immediate backup of the live state.**
* **STOP your live Minecraft server** before attempting to replace any of its files with restored data.

* **A. Full Restore (Latest Snapshot) to a Temporary Location:**
    * **Simple:**
        ```bash
        restic restore latest --target /home/ubuntu/minecraft_restore_temp
        ```
    * **Detailed Instructions:**
        This restores the entire content of the most recent snapshot into the `/home/ubuntu/minecraft_restore_temp` directory. Restic will create this target directory if it doesn't exist. After restoration, you can inspect the files and, if correct, manually stop your live server, move/rename the live server directory (as a precaution), and then move the restored content into place.

* **B. Full Restore (Specific Snapshot) to a Temporary Location:**
    * **Simple:**
        ```bash
        restic restore <SNAPSHOT_ID> --target /home/ubuntu/minecraft_restore_temp
        ```
    * **Detailed Instructions:**
        Similar to above, but restores the server state from the snapshot matching the given `<SNAPSHOT_ID>`.

* **C. Partial Restore (Specific Files/Folders) to a Temporary Location:**
    * **Simple (e.g., restoring `server.properties` and the `world/datapacks` folder from latest):**
        ```bash
        restic restore latest --target /home/ubuntu/minecraft_partial_restore --include "/server.properties" --include "/world/datapacks/"
        ```
    * **Detailed Instructions:**
        Use the `--include` flag to specify which files or folders *within the snapshot* you want to restore. You can use multiple `--include` flags.
        * Paths for `--include` are relative to the root of what was backed up in the snapshot.
        * Restic will recreate the necessary parent directory structure for the included items within your `--target` location (e.g., `/home/ubuntu/minecraft_partial_restore/world/datapacks/`).

### 3.4. Checking Backup Repository Integrity

* **Simple (Checks metadata and structure):**
    ```bash
    restic check
    ```
* **Detailed Instructions:**
    This command verifies the consistency and integrity of the Restic repository. It's good practice to run this periodically (e.g., weekly or monthly).
    * For a more thorough check that reads *all* actual backup data (can be very slow and I/O intensive):
        ```bash
        restic check --read-data
        ```
    * To check a subset of data (e.g., 25%):
        ```bash
        restic check --read-data-subset=1/4
        ```

### 3.5. Mounting the Repository (Browse Backups as a Filesystem)

* **Simple:**
    1.  Create a mount point: `mkdir ~/mc_backup_mount`
    2.  Mount the repository: `restic mount ~/mc_backup_mount`
    3.  Browse `~/mc_backup_mount/snapshots/` using your file manager or `ls`, `cd`. Snapshots will appear as dated folders or by ID.
    4.  When done, unmount: `fusermount -u ~/mc_backup_mount`
* **Detailed Instructions:**
    This command mounts all snapshots in your repository as a regular filesystem using FUSE. This is very useful for easily Browse or copying out individual files from various snapshots without a full restore command. The mount is read-only. You'll need `fuse` package installed (`sudo apt install fuse`).

## 4. Usage Restrictions & Best Practices

* **DO NOT manually modify or delete files within the Restic repository directory** (`/home/ubuntu/minecraftbackups` or your configured path). This WILL corrupt your backups.
* **DO NOT manually run `restic forget` or `restic prune` unless you fully understand the system's automated retention policy and coordinate any changes.** The automated backup script already handles this.
* **DO periodically test the restore process** to a temporary location. This ensures your backups are working and you are familiar with recovery procedures.
* **DO ensure the Restic password file** (e.g., `/home/ubuntu/.config/restic/password`) remains secure and has strict permissions (`chmod 600`).
* **DO monitor the backup script's log file** (e.g., `/home/ubuntu/logs/minecraft_backup.log`) for errors or successful completion messages.
* **DO NOT rely solely on automated systems.** Periodically verify functionality.
* **ALWAYS STOP your live server and consider making an additional quick manual backup of its current state *before* replacing any live server files with data from a Restic restore.**

## 5. Basic Troubleshooting for Manual Commands

* **"Fatal: unable to open repository..."**: Ensure `RESTIC_REPOSITORY` is set correctly or you're using the `--repo` flag with the correct path.
* **"Fatal: wrong password or no key found"**: Ensure `RESTIC_PASSWORD_FILE` is set correctly and points to the correct password file, or that you're entering the correct password if prompted (though manual commands should use the file for consistency).
* **"Fatal: repository is already locked by PID ..."**: Another Restic process is currently operating on the repository (e.g., the automated cron job might be running). Wait for it to finish. If you are sure no other process is active, you can (with caution) run `restic unlock`.
* Refer to the script's log file for errors related to the automated backup process.