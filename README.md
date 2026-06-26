# Guide: Enabling Keyboard Backlight on Acer Aspire Lite (AL15-52) in Arch Linux

## The Problem

The Acer Aspire Lite AL15-52 utilizes an ODM motherboard design (typically Clevo or Tongfang). In these specific chassis architectures, the keyboard backlight lacks direct physical ACPI hardware control.

In Windows, pressing the Fn keys triggers a proprietary WMI (Windows Management Instrumentation) call via software like "Control Center 3.0" or `fnhotkeys.exe` to change the light state. In standard Linux kernels, these proprietary WMI calls are absent. Consequently, the Embedded Controller (EC) receives no signals, causing the backlight to idle and eventually turn off via a firmware sleep timer, requiring a reboot to Windows to reactivate.

## The Solution

To resolve this, we must inject a custom DKMS kernel module that translates these proprietary WMI calls for the Linux kernel, map the hardware interface, and automate the permissions and shortcuts using standard desktop environment tools.

---

### Step 1: Install Kernel Headers

Custom drivers compile directly against your running kernel using DKMS (Dynamic Kernel Module Support). If the correct headers are missing, the build process will fail silently or throw directory errors.

Install the standard Arch Linux headers:

```bash
sudo pacman -S linux-headers

```

*(Ensure you select the package that matches your exact running kernel version, avoiding LTS or Zen variants unless explicitly using them).*

### Step 2: Install the Patched WMI Driver

We require the `clevo-drivers-dkms-git` package from the AUR.

**Why this specific driver?** It is a fork of the official Tuxedo keyboard driver that intentionally patches out the `tuxedo_is_compatible()` vendor checks. This forces the module to load and communicate directly with the underlying Clevo/Tongfang WMI interface on the Acer-branded board.

Install via your AUR helper (e.g., `paru`):

```bash
paru -S clevo-drivers-dkms-git

```

Once installed, verify the modules compiled successfully:

```bash
dkms status

```

### Step 3: Enable Kernel Modules on Boot

To ensure the kernel loads the drivers automatically upon every reboot, add them to `systemd`'s module load list.

Create a configuration file in `modules-load.d`:

```bash
echo -e "tuxedo_keyboard\nclevo_wmi" | sudo tee /etc/modules-load.d/clevo-backlight.conf

```

### Step 4: Automate Hardware Permissions (udev rule)

The kernel exposes the new software control interface at `/sys/class/leds/white:kbd_backlight/brightness`. By default, Linux restricts writing to `/sys/` class files to the `root` user.

To allow a standard user script to toggle the light without prompting for a `sudo` password, create a `udev` rule that sets read/write permissions (`666`) the moment the virtual hardware initializes during boot.

Create the rule:

```bash
echo 'SUBSYSTEM=="leds", ACTION=="add", KERNEL=="white:kbd_backlight", RUN+="/bin/chmod 666 /sys/class/leds/%k/brightness"' | sudo tee /etc/udev/rules.d/99-kbd-backlight.rules

```

Apply the rule immediately for the current session:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
sudo chmod 666 /sys/class/leds/white\:kbd_backlight/brightness

```

### Step 5: Create the Toggle Script

Create a Bash script that evaluates the current state of the backlight file and flips the value (turning it on if off, and vice versa).

1. Create the local binary directory and the script file:
```bash
mkdir -p ~/.local/bin
nano ~/.local/bin/toggle-kbd-light.sh

```


2. Add the following logic:
```bash
#!/bin/bash
BL_FILE="/sys/class/leds/white:kbd_backlight/brightness"

# Read the current brightness state
STATE=$(cat "$BL_FILE")

# Toggle the state (255 is typically maximum brightness, 0 is off)
if [ "$STATE" -eq 0 ]; then
    echo 255 > "$BL_FILE"
else
    echo 0 > "$BL_FILE"
fi

```


3. Make the script executable:
```bash
chmod +x ~/.local/bin/toggle-kbd-light.sh

```



### Step 6: Bind to a GNOME Keyboard Shortcut

To replicate the native OS experience, bind the newly created script to a physical keyboard combination.

1. Open **GNOME Settings** -> **Keyboard**.
2. Select **View and Customize Shortcuts** -> **Custom Shortcuts**.
3. Click **Add Shortcut** and enter:
* **Name:** Toggle Keyboard Backlight
* **Command:** `/home/YOUR_USERNAME/.local/bin/toggle-kbd-light.sh`
* **Shortcut:** Set your preferred keybinding (e.g., `Super + F8`).



You now have complete, automated software control over the ODM keyboard backlight within Arch Linux.
