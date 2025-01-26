# GameShell Incus Deployment

This repository showcases how to:

1. **Store a Cloud-Init configuration** (`cloud-init.yaml`) in version control.
2. **Create an Incus (LXD) profile** for devices (network, storage, etc.) **without** hardcoding Cloud-Init directly in the profile.
3. **Inject the Cloud-Init file** into the profile at runtime.
4. **Lock down user access** so that only a `gameshell` user can log in.
5. **Run GameShell** with optional language flags.

## Prerequisites

- **Incus** (or LXD) installed and initialized on your host.
  - Refer to the [Incus documentation](https://linuxcontainers.org/incus/) or your distribution’s package manager for installation instructions.
- A storage pool named `default` (adjust names if you use something else).
- A bridged network interface such as `incusbr0` (or `lxdbr0` if you use the default LXD bridge).

## Repository Contents

- **`cloud-init.yaml`**  
  This file contains our Cloud-Init configuration:
  - Sets up the `gameshell` user, locks out root, disables SSH password auth.
  - Creates a wrapper script that launches the GameShell script with a specific language flag.
  - Installs minimal packages required (e.g., `wget`, `curl`, `tar`).
  - Downloads the GameShell script from GitHub.

- **(Optional) `profile.yml`**  
  An example Incus profile definition (devices only). You may or may not store this in Git—some prefer to create/edit profiles on-the-fly.

- **`README.md`**  
  This documentation.

## Setup Steps

Follow these steps to replicate the environment on your machine.

### 1. Clone the Repository

```bash
git clone https://github.com/tsaquet/gameshell-incus.git
cd gameshell-incus
```

### 2. Create a Named Storage Volume

Create a persistent storage volume for game data. This volume will be mounted to `/home/gameshell/game_data` inside the container. This way, data persists even if the container is destroyed or recreated.

```bash
incus storage volume create default gameshell-data
```

### 3. Create an Incus Profile (Devices Only)

You can create a new profile named `gameshell-profile` that **only** includes device definitions (network, storage) — **no** embedded Cloud-Init. There are two main ways to do this:

#### Option A: Command-line device additions

```bash
# Create an empty profile
incus profile create gameshell-profile --empty

# Add a root disk device (pointing to your default pool)
incus profile device add gameshell-profile root disk path=/ pool=default

# Attach the named storage volume to /home/gameshell/game_data
incus profile device add gameshell-profile gameshell-data disk \
  path=/home/gameshell/game_data \
  pool=default \
  source=gameshell-data

# Add a bridged network interface
incus profile device add gameshell-profile eth0 nic \
  nictype=bridged \
  parent=incusbr0
```

#### Option B: YAML editing

If you prefer manual editing:

```bash
incus profile create gameshell-profile
incus profile edit gameshell-profile
```

Paste in something like:
```yaml
config: {}
description: "Profile devices only; cloud-init is applied separately."
devices:
  root:
    type: disk
    path: /
    pool: default
  gameshell-data:
    type: disk
    path: /home/gameshell/game_data
    pool: default
    source: gameshell-data
  eth0:
    type: nic
    nictype: bridged
    parent: incusbr0
name: gameshell-profile
```

Then save and exit the editor.

#### Option C: Use a Yaml file

**Create (or overwrite) the profile using `gameshell-profile.yml`:**
   ```bash
   incus profile create gameshell-profile
   incus profile edit gameshell-profile < gameshell-profile.yml
   ```

### 4. Inject the Cloud-Init File into the Profile

Inside this repository, you will find `cloud-init.yaml`. This includes:

- User creation (`gameshell`).
- Locking the `root` account, disabling SSH password logins.
- Locale/region setup for French (`fr_FR`) and English (`en_US`).
- Download of the GameShell script.
- A **wrapper script** that calls `gameshell.sh` with a language flag (e.g., `-L fr`).

To load that configuration into your profile:

```bash
incus profile set gameshell-profile user.user-data - < cloud-init.yaml
```

**Note**:  
- The `-` indicates to read from standard input.  
- `< cloud-init.yaml` feeds the file into `incus profile set`.

This command overwrites (or sets) **only** the `user.user-data` key in the profile.

### 5. Launch the Container

Pick a **cloud-init-capable** image such as `ubuntu:22.04`:

```bash
incus launch launch images:ubuntu/24.04/cloud gameshell-container -p gameshell-profile
```

Incus creates the container and boots it. Cloud-Init runs at first boot:

- The `gameshell` user is created,  
- The `gameshell.sh` script is downloaded to `/home/gameshell/game_data/`,  
- A wrapper script sets the default language option,  
- `root` is locked.

### 6. Verify and Log In

After the container is up, verify:

```bash
incus list
incus exec gameshell-container -- su - gameshell
```

You will be logged in as the `gameshell` user, and it should launch the wrapper script. If the script calls GameShell with `-L fr`, you’ll see the French version.

#### Changing the Language

If you want to switch to English, you can either:

- **Modify `cloud-init.yaml`** and re-inject it (only applies for new containers or if you reset the existing one).
- **Set an environment variable** in the container or profile. For example:
  ```bash
  incus profile set gameshell-profile environment.GAMESHELL_LANG en
  incus restart gameshell-container
  ```
  Then adjust your wrapper script to read that variable, e.g.:
  ```bash
  exec /home/gameshell/game_data/gameshell.sh -L "${GAMESHELL_LANG:-fr}"
  ```
  (See repository code comments for more info.)

### 7. Confirm Root is Locked

By default, the `root` account has no password, and `ssh_pwauth` is false. This means **only** the `gameshell` user can log in with an interactive shell. The host’s Incus admin can still run commands like `incus exec gameshell-container -- bash`, but that’s normal for container management.

---

## Maintenance and Updates

- If you want to **edit your Cloud-Init** config, simply update `cloud-init.yaml` and run:
  ```bash
  incus profile set gameshell-profile user.user-data - < cloud-init.yaml
  ```
  **Note**: Cloud-Init usually runs once on **first boot**. To re-run it on an existing container, you might need to reset Cloud-Init’s state manually or recreate the container.

- If you want to **modify devices** (e.g., attach different volumes or networks), edit the profile devices using either the CLI or `incus profile edit gameshell-profile`.

- **Game data** is stored on the named volume `gameshell-data`. If you destroy the container, the data remains in that volume until you remove the volume itself.

---

## Contributing

Feel free to open issues or PRs if you notice any improvements or bugs. The [GameShell project](https://github.com/phyver/GameShell) is maintained separately; this repo just provides an Incus-based deployment example.

---

## License

[MIT License](LICENSE).

---

## Acknowledgments

- [Phyver’s GameShell](https://github.com/phyver/GameShell) for the actual GameShell script.  
- [Incus (LXD) Documentation](https://linuxcontainers.org/incus/) for container virtualization tooling.  

---

**Enjoy your GameShell containerized environment!**
