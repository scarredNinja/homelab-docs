---
project_id: Homelab-2025
phase: 'Phase 3: Network Config'
tags:
  - network
  - reference
status: Reference
---

## **1. Save your current config before testing**

```bash
save configuration primary
```

This writes the current running config to the **primary** slot (think of it like a save point).  
You can also save to the secondary slot if you want two checkpoints:

```bash
save configuration secondary
```

---

## **2. View and choose which config to boot**

EXOS keeps **primary** and **secondary** configs stored in NVRAM.  
You can check them with:

```bash
show configuration
```

or for more detail:

```bash
show switch
```

---

## **3. Roll back to a known-good config**

You can tell EXOS to load your last saved config without rebooting:

```bash
use configuration primary
```

Or:

```bash
use configuration secondary
```

> This immediately applies the stored config — useful for undoing test changes on the fly.

---

## **4. Restore after a reboot**

If you want your next boot to use a specific saved config:

```bash
configure boot configuration primary
```

or

```bash
configure boot configuration secondary
```

Then:

```bash
reboot
```

---

## **5. Full “snapshot and revert” workflow**

When testing something risky, I’d do:

```bash
save configuration primary      # Save current good setup
<make test changes>
<if broken>
use configuration primary       # Instantly reload good config
```

---

💡 **Extra Tip**  
If you want to back up off-switch so you can restore later even after a factory reset:

```bash
tftp put <server_ip> <filename>.cfg
```

And to restore:

```bash
tftp get <server_ip> <filename>.cfg
```

---

If you want, I can also write you a **one-command “snapshot & restore” helper** for EXOS so you can flip between your test and production configs instantly.  
Do you want me to do that next?


## **Method 1 — Save and reload a config file**

1. **Save current running config to a named file**
    
    ```shell
    save configuration my-stable-config
    ```
    
    This creates a snapshot of your current working setup.
    
2. **Make your changes and test**  
    Adjust VLANs, trunks, bonds, etc.
    
3. **Reload the old config if needed**
    
    ```shell
    load configuration my-stable-config
    ```
    
    This will overwrite the running config with your saved one.
    
4. **Reboot to apply fully** _(optional, but ensures a clean start)_
    
    ```shell
    reboot
    ```
    

---

## **Method 2 — Use primary/secondary configs for quick fallback**

EXOS keeps **two boot config slots** — _primary_ and _secondary_. You can boot from either.

1. **Save your stable config to secondary slot**
    
    ```shell
    save configuration secondary
    ```
    
2. **Do your testing on primary** (or vice versa)
    
3. **Switch boot slot back to stable one**
    
    ```shell
    use configuration secondary
    reboot
    ```
    

---
