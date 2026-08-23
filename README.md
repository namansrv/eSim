**Author:** namansrv
**Branch:** [`installers`](https://github.com/namansrv/eSim/tree/installers)[cite: 1]
**Environment:** Ubuntu 25.04 (Plucky Puffin), tested in a virtual machine( [`Boxes`](https://flathub.org/en/apps/org.gnome.Boxes) )
**Task:** eSim Semester Long Internship — Autumn 2026, Task 4 (eSim Upgradation)

**Number of issues reported:** 6
**Number of issues fixed:** 6


## Summary Table (Impact-Ordered)


| **Severity** | **Issue**              | **Component** | **Fix Implemented**                                                           |
| ------------ | ---------------------- | ------------- | ----------------------------------------------------------------------------- |
| **Critical** | GUI Fails to Launch    | Core GUI      | Swapped `PyQt5` to `PyQt6` in dependencies to match `Application.py` imports. |
| **High**     | KiCad PPA 404 Error    | Dependencies  | Migrated to `kicad-9.0-releases` native Ubuntu 25.04 repository.              |
| **Medium**   | GHDL Rejects LLVM 20   | NGHDL         | Dropped LLVM flag from configure, defaulting to `mcode` backend.              |
| **Medium**   | `exit 0` Kills Script  | Installer     | Replaced `exit 0` with `return 0` in KiCad check to prevent silent abort.     |
| **Low**      | Obsolete Audio Library | NGHDL         | Replaced deprecated `libcanberra-gtk-module` with `libcanberra-gtk3-module`.  |
| **Low**      | Unsupported OS Version | Installer     | Added `25.04` dispatcher case block to allow installation.                    |


  

  



## How to read this report

This document is written in the order I actually hit each problem, fixed it, and moved on to the next one. Ubuntu 25.04 has no official support in eSim's installer scripts.


---

## Problem 1 — "Unsupported Ubuntu version"

The very first thing the script does is detect the
Ubuntu version and hand off to a version-specific installer script. On
25.04 it immediately failed:

```
Unsupported Ubuntu version: 25.04 ()
```


There was simply no `"25.04")` branch so the script refused to run at all.

**Fix:** Added a `25.04` case, pointing at a new
`install-eSim-25.04.sh` script (based on the closest existing version,
24.04, as a starting point):

```bash
"25.04")
    SCRIPT="$SCRIPT_DIR/install-eSim-25.04.sh"
    ;;
```

With this in place, the script now actually started running on 25.04
## Problem 2 — KiCad wouldn't install 

Once the dispatcher worked, the script reached the KiCad install step and failed with a 404 while adding the PPA:

  

![[Pasted image 20260822230409.png]]

  **What I tried:** I worked backwards from the newest KiCad release and landed on **KiCad 9** (`kicad-9.0-releases`) . It's the newest KiCad line that actually has a published Ubuntu 25.04 build.

  

**One important thing when switching PPAs on a machine where you already tried a different version:** apt will keep the old, broken PPA registered unless you explicitly remove it first, which can cause confusing conflicts. Before switching to a new version, run

```bash
sudo add-apt-repository --remove ppa:kicad/kicad-8.0-releases
sudo apt-get update
```





**Fix:** Set the script to use KiCad 9 specifically:

```bash
kicadppa="kicad/kicad-9.0-releases"
```

Since I changed the actual KiCad version being installed, I also had to update every other place in the script that hardcoded the old version number

  ---

  

  

## Problem 3 — `libcanberra-gtk-module`has no installation candidate

With KiCad finally working, the script moved on to NGHDL's dependencies and immediately hit:

  

![[Pasted image 20260822230917.png]]

  

**Why:** `libcanberra-gtk-module` is the **GTK2** build of this sound-theme integration library. Ubuntu dropped it from the repositories entirely as of 25.04 — only the GTK3 build, `libcanberra-gtk3-module`, still exists. Since the script runs under `set -e`/`trap error_exit ERR`, this single missing package name was enough to abort the whole NGHDL dependency stage.

  

**Fix:** The install line originally requested both:

```bash
sudo apt install -y libcanberra-gtk-module libcanberra-gtk3-module
```

I removed `libcanberra-gtk-module` from the list entirely:

```bash
sudo apt install -y libcanberra-gtk3-module
```

This package only controls whether GTK apps play little UI sound effects — it has no effect on whether GHDL, Verilator, or Ngspice actually build, so dropping it is safe. 



## Problem 4 — GHDL's configure script rejects LLVM 20


Past the dependency stage, the script started extracting and building GHDL from source — and this is where I hit the biggest wall of text in the whole process (pages of file extraction output), ending in:

  

![[Pasted image 20260822231207.png]]

  

**Fix:** GHDL doesn't actually need the LLVM backend for what NGHDL uses it for. Its default backend, `mcode`, doesn't touch LLVM at all and isn't affected by this version check. The original configure call was:

```bash
./configure --with-llvm-config=/usr/bin/llvm-config
```

I removed the LLVM flag entirely:


```bash
./configure
```

This switches the build to `mcode`, which built successfully with no further changes needed.

  

![[Pasted image 20260822232600.png]]

  


## Problem 5 — KiCad "already installed" check silently killed the whole script

 On a re-run (after KiCad 9 was already installed), the script printed:

  

```
KiCad 9.0 is already installed.
```

It then just stopped. No error, no continuation — the terminal returned to the prompt as if the whole install had finished, but NGHDL, the SKY130 PDK, and the desktop shortcut never ran.

  

**Why:** The relevant code was:


```bash
else
    echo "KiCad 9.0 is already installed."
    exit 0
```

Bash functions share the same process as the script that calls them — they don't run in a separate subshell. `exit 0` here doesn't return from just the `installKicad` function, it **terminates the entire script** immediately. Every step after `installKicad` in the main install sequence was silently skipped, with no error to indicate anything had gone wrong.

**Fix:**


```bash
else
    echo "KiCad 9.0 is already installed."
    return 0
```

`return 0` exits only the function, letting the rest of the script continue normally.

---

  

## Problem 6 — eSim GUI Fails to Launch (PyQt5 vs PyQt6)

**What happened:** After a seemingly successful installation, typing `esim` in the terminal failed to actually launch the graphical interface.

  
**Why:** The main application code (`src/frontEnd/Application.py`) was written to import `PyQt6`. However, the `installDependency` function in the wrapper script was still hardcoded to install `python3-pyqt5`. Without the correct Qt framework, the GUI was entirely blocked from launching.

  

**Fix:** Updated both the `apt` and `pip` installation lines to pull the correct framework:


```bash
sudo apt-get install -y python3-pyqt6
pip3 install PyQt6  
```

  
---
  

## Final result

  

With all six problems above resolved, a full, clean run completed

end-to-end:

  

```
-----------------eSim Installed Successfully-----------------
Type "esim" in Terminal to launch it
or double click on "eSim" icon placed on Desktop
```

`esim` launched the application correctly, and a follow-up `--uninstall` run completed cleanly 
  


---

## Files Changed


- `Ubuntu/install-eSim.sh`
    
      
    
    
    
      
    
- `Ubuntu/install-eSim-scripts/install-eSim-25.04.sh`
    
      
    
    
    
      
    
- `nghdl/install-nghdl.sh` (added)
    
      
    
- `nghdl/install-nghdl-scripts/install-nghdl-25.04.sh` (added)

  

  

## Reproduction Steps
  

  

```bash
# 1. Get eSim source (master branch)
git clone [https://github.com/namansrv/eSim.git](https://github.com/namansrv/eSim.git) eSim-test
cd eSim-test
git checkout master

# 2. Pull in the fixed installer scripts from the installers branch
git fetch origin installers
git checkout origin/installers -- Ubuntu
git checkout origin/installers -- nghdl

# 3. Move the installer script to the top level
cp Ubuntu/install-eSim.sh ./install-eSim.sh
cp -r Ubuntu/install-eSim-scripts ./install-eSim-scripts
chmod +x install-eSim.sh

# 4. Build nghdl.zip 
zip -r nghdl.zip nghdl/

# 5. Run the installer
./install-eSim.sh --install
# answer 'n' to the proxy prompt unless applicable

# 6. Verify
esim

# 7. Test uninstall
./install-eSim.sh --uninstall
```

## Disclaimer for Future Installations

If anyone is attempting to port or install this in the future (e.g., for Ubuntu 26.04+), please remember to **completely delete all leftover files made by previous eSim installation attempts** before re-running the installer. Leftover directories and artifacts (such as `~/.esim`, `~/nghdl-simulator`, `~/.config/kicad`, or partially extracted folders) will cause false-positive "Directory not empty" or "Cannot open file" errors that halt the scripts. Always start with a completely scrubbed system environment!
