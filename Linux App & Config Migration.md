# Linux App & Config Migration Guide

This guide details how to handle migrations, configurations, and directory mounting for your development tools, IDEs, and games when moving between Windows and Linux.

---

## 💾 Mounting vs. Syncing: The Rules of Thumb

When dual-booting, it is tempting to mount folders from your Windows drive directly into Linux. Here is what you should and shouldn't mount:

| Category | Mount directly? | Recommended Strategy |
| :--- | :---: | :--- |
| **Active Project Folders (Code)** | **Yes (with caveats)** | You can mount your `D:` drive and edit code files directly. However, filesystem-heavy commands like `npm install` are slower on NTFS mounts in Linux. |
| **App Settings / Configs (VS Code, etc)** | **❌ No** | Do not mount settings directories. Linux configurations require native Linux filesystems (`ext4`) for file permissions, sockets, and symlinks. Use built-in cloud sync or Syncthing. |
| **Steam Game Libraries** | **❌ No** | Steam on Linux runs games via Proton. Proton requires native Linux permissions and symlinks. Storing Proton games on NTFS drives causes launch failures and file lock errors. |

---

## 🛠️ Tool-by-Tool Migration Strategy

### 1. Antigravity History
Antigravity stores your conversation logs and data in the application data directory.
* **On Windows**: `C:\Users\DJ\.gemini\antigravity`
* **On Linux**: `~/.gemini/antigravity` (or `/home/dj/.gemini/antigravity`)
* **Migration**: 
  1. Once Antigravity is installed on Linux, simply copy the `brain` folder from your Windows drive to your Linux user directory:
     ```bash
     cp -r /mnt/win_c/Users/DJ/.gemini/antigravity/brain ~/.gemini/antigravity/
     ```
  2. This will restore all conversation history and transcripts instantly on Linux.

### 2. VS Code (Settings & Extensions)
Do not manually copy the configuration files. 
* **The Best Way**: Open VS Code on Windows, click the **Gear icon (bottom left) > Turn on Settings Sync...** and sign in with your GitHub or Microsoft account.
* **On Linux**: Open VS Code, sign in with the same account. All your settings, keybindings, theme configurations, and extensions will install automatically in the background.

### 3. Claude Desktop
* **The Catch**: Anthropic does not publish an official native client for Linux.
* **The Solution**: 
  * Use the web interface as a **PWA (Progressive Web App)** in Chrome or Brave (click the "Install" icon in the URL bar). It runs in an isolated window and behaves exactly like a desktop app.
  * Alternatively, use community-built desktop wrappers like `claude-desktop-linux` or install it via unofficial Flatpaks available in the software manager.

### 4. Git & Project Folders (NTFS Mounts)
If you decide to open and edit code projects located on your NTFS `D:` drive from Linux, Git might mark every file as modified because NTFS does not support Linux execute (`chmod +x`) permissions.
* **The Fix**: Open a terminal in Linux and run this command once globally:
  ```bash
  git config --global core.filemode false
  ```
  This tells Git to ignore file permission changes when comparing files, preventing spam modifications in your git status.

---

## ⚙️ How to Auto-Mount Your D: Drive on Linux Boot
To make sure your `D:` drive (Seagate 2TB HDD) is always accessible under the same path in Linux, configure it to auto-mount on boot:

1. Open the **Disks** application in Pop!_OS.
2. Select your 2TB Seagate HDD.
3. Click the partition, then click the **Gears icon (Additional partition options)** below it.
4. Select **Edit Mount Options...**.
5. Toggle **Session Default Options** to **Off**.
6. Check **Mount at system startup**.
7. Set the **Mount Point** to a clean path like `/media/data` or `/media/d-drive`.
8. Click **OK** and enter your password. The drive will now load automatically every time you boot Linux.


---

## 🔄 Using Syncthing for App History (Yes vs. No)

If you want to maintain your active history between Windows and Linux using Syncthing:

### 1. Antigravity: ✅ Yes (Highly Recommended)
* **What to sync**: The `brain` folder (`C:\Users\DJ\.gemini\antigravity\brain` on Windows and `~/.gemini/antigravity/brain` on Linux).
* **Why**: The conversation transcripts, indexing data, and agent states are stored in files (like SQLite DBs and JSONL). Since you are dual-booting (so the two systems never run concurrently), Syncthing will seamlessly update this folder every time you boot back and forth.
* **Benefit**: Your agent chat history is perfectly preserved across both OSes.

### 2. VS Code: ❌ No (Do Not Sync settings folders)
* **Why**: The raw `%APPDATA%\Code` and `~/.config/Code` directories contain platform-specific absolute paths, system caches, binary files, and extensions compiled specifically for Windows or Linux. Syncing this folder will corrupt VS Code.
* **Instead**: Enable **Settings Sync** inside VS Code (under Settings > Sign In). This only syncs your JSON configurations and extension lists, and VS Code automatically downloads the correct binaries for each operating system.

### 3. Claude Desktop / Web: ℹ️ Not Needed
* **Why**: Claude's history is stored on Anthropic's cloud servers, not locally. As long as you sign in with the same account in your Linux browser/PWA, all your chat history will load automatically.


---

## 🔍 Your Windows System Audit: What Else to Migrate

Based on an audit of your Windows machine, here are the critical files and setups you need to migrate:

### 1. SSH Keys 🔑 (High Priority)
You have active SSH keys under `C:\Users\DJ\.ssh\`:
* `homelab_ed25519` & `homelab_id_rsa` (likely used for your home server / Swarm / Portainer access)
* `id_ed25519` & `id_rsa` (likely used for GitHub / GitLab)
* **Migration**:
  1. Boot into Linux.
  2. Create the directory: `mkdir -p ~/.ssh`
  3. Copy your SSH key files from your Windows partition (or a temporary USB/Syncthing folder) into `~/.ssh/`.
  4. **CRITICAL: Set the correct file permissions** (Linux will refuse to use SSH keys if their permissions are too open):
     ```bash
     chmod 700 ~/.ssh
     chmod 600 ~/.ssh/*
     ```

### 2. Git & Git-LFS Configuration
Your Windows Git config shows that you use **Git Large File Storage (LFS)**:
* **Migration**:
  1. Install Git-LFS on Pop!_OS:
     ```bash
     sudo apt install git-lfs -y
     ```
  2. Set up your global Git identity:
     ```bash
     git config --global user.name "DJ Purvis"
     git config --global user.email "scarredninja360@gmail.com"
     git config --global filter.lfs.required true
     git lfs install
     ```

### 3. WSL2 Distros (Ubuntu)
You have a WSL2 **Ubuntu** distro currently stopped on Windows.
* **Warning**: If you have any projects, databases, or keys stored *inside* your WSL2 Ubuntu environment, they are stored in a virtual disk (`ext4.vhdx`) on your Windows `C:` drive.
* **Migration**:
  * On Windows, open File Explorer and navigate to `\\wsl.localhost\Ubuntu\home\`.
  * Copy your projects and folders from there onto your shared `D:` drive or back them up.

### 4. MS SQL Server (`MSSQLSERVER`)
You have Microsoft SQL Server running as a local database service on Windows.
* **Linux Alternative**: MS SQL Server does not run natively on Linux desktop, but you can run it perfectly inside a **Docker container**:
  ```bash
  docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=YourStrongPassword" -p 1433:1433 -d mcr.microsoft.com/mssql/server:2022-latest
  ```
  You can connect to it using **DBeaver** or **Azure Data Studio** (both have native Linux builds).

### 5. PuTTY and WinSCP Alternatives
* **On Linux**: You do not need PuTTY or WinSCP. 
  * Open the terminal and type `ssh user@ip` to connect to servers (using the keys we migrated in Step 1).
  * To transfer files (WinSCP replacement), open the default Files manager in Pop!_OS, click **+ Other Locations**, and under **Connect to Server** at the bottom, type:
    `sftp://user@ip_address`
    This mounts the remote server directly in your file explorer, letting you drag and drop files natively!


---

## 📦 Exact Migration Commands & Logistics

Since you are dual-booting, **Pop!_OS can read and write to your Windows C: drive directly**. You do not need to upload files to the cloud or use a USB drive to move things. 

Here is the exact step-by-step procedure for migrating each item:

### 1. Preparation: Mount your Windows C: Drive in Pop!_OS
1. Boot into Pop!_OS.
2. Open the **Files** manager.
3. Click **+ Other Locations** in the left sidebar.
4. Click on your Windows OS drive (Samsung 980 Pro 1TB, usually labeled "980 PRO" or "Windows"). This automatically mounts it.
5. In your terminal, find where it was mounted by running:
   ```bash
   ls /media/$USER/
   ```
   You will see a folder named with a long alphanumeric ID or your Windows partition label. We will refer to this mount path as `/media/$USER/Windows/` in the commands below.

---

### 2. Migrate SSH Keys
Run these commands in the Pop!_OS terminal:
```bash
# 1. Create your Linux SSH directory
mkdir -p ~/.ssh

# 2. Copy the keys from your Windows C: drive
cp -r /media/$USER/Windows/Users/DJ/.ssh/* ~/.ssh/

# 3. Secure the permissions (vital for SSH to work)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/*
```

---

### 3. Migrate Antigravity History
Run these commands in the Pop!_OS terminal:
```bash
# 1. Create the destination directory
mkdir -p ~/.gemini/antigravity

# 2. Copy your brain database and transcript history
cp -r /media/$USER/Windows/Users/DJ/.gemini/antigravity/brain ~/.gemini/antigravity/
```

---

### 4. Migrate WSL2 Ubuntu Files (DO THIS ON WINDOWS FIRST)
Since WSL2 stores files inside a virtual disk (`.vhdx` file), mounting it inside Linux is highly complex. It is much easier to do this **on Windows before you log off**:
1. While booted into Windows, open **File Explorer**.
2. Go to the address bar and type: `\\wsl.localhost\Ubuntu\home\`
3. Navigate to your user folder and copy any projects, scripts, or files you need.
4. Paste them onto your **shared D: drive** (e.g. `D:\WSL_Backup`).
5. Once you boot into Linux, auto-mount your D: drive (as detailed above) to access these files directly.

---

### 5. Migrate Microsoft SQL Server Databases
To move local SQL Server databases to your Linux Docker container:
1. **On Windows (Before logging off)**:
   * Open SQL Server Management Studio (SSMS).
   * Right-click your database -> **Tasks** -> **Back Up...**
   * Back up the database as a `.bak` file.
   * Save this `.bak` file onto your shared **D: drive** (e.g. `D:\SQL_Backups\mydb.bak`).
2. **On Linux**:
   * Start your MS SQL Server Docker container:
     ```bash
     docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=YourStrongPassword123" -p 1433:1433 -v /media/d-drive/SQL_Backups:/var/opt/mssql/backup -d mcr.microsoft.com/mssql/server:2022-latest
     ```
     *(Note: The `-v` flag mounts your shared D: drive backup folder directly inside the container).*
   * Open **DBeaver** or **Azure Data Studio** on Linux, connect to the container (`localhost,1433` with user `sa` and password `YourStrongPassword123`), and run a standard T-SQL query to restore your database from `/var/opt/mssql/backup/mydb.bak`.


---

## 📁 Documents, Photos, and Media Files

Your personal media libraries (Documents, Pictures, Downloads, Music, Videos) can be migrated in two ways depending on your space requirements:

### Option A: Local Copy (Best if library size is small)
Copy the directories directly from your mounted Windows drive to the equivalent folders in your Linux home directory:
```bash
cp -r /media/$USER/Windows/Users/DJ/Documents/* ~/Documents/
cp -r /media/$USER/Windows/Users/DJ/Pictures/* ~/Pictures/
cp -r /media/$USER/Windows/Users/DJ/Music/* ~/Music/
cp -r /media/$USER/Windows/Users/DJ/Videos/* ~/Videos/
```

### Option B: Shared D: Drive + Symlinks (Highly Recommended for Large Libraries)
Since your Linux OS drive is 250GB and your shared `D:` drive (HDD) is 2TB, keeping massive photo, video, or document libraries on your Linux OS SSD will fill it up quickly. 

Instead, you can store your libraries on the `D:` drive and create **symbolic links** in Linux so they appear natively in your home folder:

1. **Move your Windows folders to the shared D: drive**:
   On Windows, move your Documents and Pictures folders to `D:\Documents` and `D:\Pictures` if they aren't already there.
2. **Delete the default empty folders in Linux**:
   ```bash
   rm -rf ~/Documents ~/Pictures ~/Music ~/Videos
   ```
3. **Create Symbolic Links pointing to the D: drive**:
   *(Assuming your D: drive is auto-mounted at `/media/d-drive` as detailed in the mounting section above)*:
   ```bash
   ln -s /media/d-drive/Documents ~/Documents
   ln -s /media/d-drive/Pictures ~/Pictures
   ln -s /media/d-drive/Music ~/Music
   ln -s /media/d-drive/Videos ~/Videos
   ```
   Now, clicking on "Pictures" or "Documents" in your Pop!_OS sidebar will instantly open the files on your 2TB drive without consuming any space on your Linux SSD.
