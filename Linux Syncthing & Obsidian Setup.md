# Linux Syncthing & Obsidian Setup Guide

Use this guide once you boot into Pop!_OS to configure your Obsidian Vault sync.

---

## 1. Install Obsidian
Open a terminal in Pop!_OS and run:
```bash
flatpak install flathub md.obsidian.Obsidian
```
*(Or search and download "Obsidian" directly in the **Pop!_Shop** software center).*

---

## 2. Install & Start Syncthing
In the terminal, run:
```bash
sudo apt update
sudo apt install syncthing -y

# Enable it as a background service for your user account:
systemctl --user enable --now syncthing
```

---

## 3. Link Your Vault
1. Open your browser in Linux and go to: **`http://localhost:8384`** (this is the Syncthing Web UI).
2. Click **Add Remote Device** and enter your home server (or phone) Device ID.
3. Once the server is linked, accept the incoming **`Obsidian Vault`** folder share.
4. Set the local folder path to where you want the vault to live on Linux, for example:
   `/home/dj/Documents/Notes/Home`

---

## 4. Open in Obsidian
1. Open Obsidian on Pop!_OS.
2. Click **"Open folder as vault"**.
3. Select `/home/dj/Documents/Notes/Home`.

Any changes you make on either Windows or Linux will now sync instantly through your always-on home server!


---

## ⚠️ Troubleshooting: Deleted Files Coming Back (Zombie Files)

If you delete a note in Obsidian and it reappears shortly after, this is a common Syncthing synchronization conflict. Syncthing is trying to keep all devices identical. If one device fails to delete the file, it will sync it back to the rest of your devices.

### Step 1: Turn on "Ignore Permissions" (Crucial for Windows/Linux Sync)
Windows (NTFS) and Linux (ext4) handle file permissions differently. If Syncthing tries to sync permission metadata, it will fail to delete files, causing them to sync back.
1. Open the Syncthing Web UI (`http://localhost:8384`).
2. Click **Edit** on your `Obsidian Vault` folder.
3. Go to the **Advanced** tab.
4. Check **Ignore Permissions**.
5. Click **Save**.
6. **Do this on both Windows, Linux, and your Server/Phone.**

### Step 2: Identify the "Zombie" Device
1. Open the Syncthing Web UI on all devices.
2. Look at the folder state. If one device says **"Out of Sync"** or lists **"Failed Items"** under the folder, that device is the culprit.
3. It has locked the file or failed to delete it (often due to Obsidian running and lock-indexing the file on that machine).
4. Close Obsidian on that device, delete the file manually on that device, and let Syncthing scan again.

### Step 3: Check for Cloud Backups
If the folder you are syncing is also inside a cloud service (like OneDrive, Google Drive, or a server auto-backup), that service might be "restoring" deleted files from its recycle bin as soon as they are deleted, which Syncthing detects as a "new file" and pushes to your other devices. Exclude the vault from other background sync tools.
