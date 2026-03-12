# PhotoPlay System Architecture & File System Map

This document provides a comprehensive technical breakdown of the PhotoPlay environment (Version 2008 (ES) NSB IGO8 VM007), detailing its operating system, boot process, directory structure, and execution logic.

## 1. Operating System: PTS-DOS
**PTS-DOS** is a high-performance **MS-DOS clone** developed in Russia by **PhysTechSoft** and Paragon Technology Systems (first released in 1993). 

The use of PTS-DOS in the PhotoPlay platform is presumed to be due to its superior conventional memory management and streamlined kernel, which generally offered better stability for embedded systems than MS-DOS 6.22 or early versions of FreeDOS. These technical advantages likely ensured optimal performance for the touchscreen interface and game software while maintaining compatibility within the FAT16 file system standards of the era.

### Core Kernel Files (Root Directory)
- **PTSBIO.SYS**: Hardware abstraction layer.
- **PTSDOS.SYS**: Kernel services.
- **COMMAND.COM**: Command-line interpreter.

---

## 2. The Boot Chain & Initialization
The system boot process is a multi-stage sequence designed to initialize hardware and launch the game shell with maximum protection.

### Stage 1: CONFIG.PTS (The System Config)
The `CONFIG.PTS` (equivalent to `CONFIG.SYS`) handles the early environment:
- **`install = \foto\logo.com`**: Immediately displays the "PhotoPlay" splash screen before the CLI appears.
- **`device = \SOUND\himem.sys`**: Extended memory manager (driver variant for specific sound support).
- **`device = \ptsdos\ramdisk.sys 12M /E`**: Creates a **12MB RAM Drive**. This is crucial for temporary game data and logs to avoid HDD wear and increase speed.
- **`shell = \PTSDOS\command.com \PTSDOS /p \AUTOPTS.BAT`**: Defines the final shell script.

### Stage 2: AUTOPTS.BAT (The Master Script)
The initialization sequence in `AUTOPTS.BAT`:
1.  **Update Check**: `call \update\check.bat`. Verifies if there's a CD-ROM or USB with updates.
2.  **Touch Init**: `call \touch\tch_init.bat`. Detects and loads Elo or MicroTouch drivers.
3.  **Sound Init**: `call \sound\snd_init.bat`. Auto-detects the motherboard type and loads the appropriate driver (e.g., ESS1868, C-Media).
4.  **Main Loop**: Enters the game execution loop.

---

## 3. The Application Loop (Execution Logic)
PhotoPlay uses a persistent loop to handle system state transitions without rebooting.

```batch
:restart
cd\menu
main.com
if errorlevel 100 goto transmit
if errorlevel 42 goto 25p42
...
goto restart
```

### Logical State Machine (Error Codes)
The main shell (`main.com`) communicates its exit reason via DOS error levels:
- **100 (Transmit)**: The system attempts to sync statistics/scores with **Fun.net** (calls `transmit.bat`).
  > [!NOTE]
  > This feature is **optional** and only functions if the system has a modem correctly configured and connected to a phone line.
- **42/40/34/33 (Calibrate)**: Triggers a **25-point touch calibration**.
- **Action on Calibration**: The loop deletes `\menu\touch.ini`, runs the calibration tool (`monitor.exe`), and returns to `:restart`.

---

## 4. File System Map (HardDisk Structure)

The disk is organized into functional modules. Most games are stored alphabetically in the root.

### Core System Folders
| Folder | Purpose | Key Files |
| :--- | :--- | :--- |
| **`\FOTO`** | Main Application Assets | `FOTOPLAY.WAD` (asset bundle), `LOGO.COM`. |
| **`\FOTO\HISCORE`**| Game Rankings | Contains all `.HIS` files for all games. |
| **`\FOTO\SETTINGS`**| Persistance | Global machine settings (Language, Volume). |
| **`\MENU`** | System Shell | `MAIN.COM` (logic), `MENU.EXE` (GUI), `STAT.DAT`. |
| **`\PTSDOS`** | OS binaries | Command-line utilities (format, fdisk, etc). |
| **`\SOUND`** | Audio drivers | Detection scripts and vendor BIOS drivers. |
| **`\TOUCH`** | Input Drivers | `ELO` (EloTouch), `MT` (MicroTouch). |
| **`\UPDATE`** | Maintenance | Update logic and CD-ROM drivers (`ATAPI.SYS`). |

### Game Directories & Executables
The system separates game logic from game assets to facilitate updates:

- **`\EXE`**: This is the "brain" folder. It contains all the main game executables (`.EXE`) and their metadata files (`.INF`).
  - *Example*: `\EXE\SHANGHAI.EXE`, `\EXE\FINDIT.EXE`.
- **`\[GAMENAME]` folders**: These folders are **asset-only**. They do not contain the executable but store the specific resources the game needs to run.
  - *Example*: `\SHANGHAI\TILES`, `\SHANGHAI\SFX`.

This centralization allows the `main.com` menu system to launch all games from a single path (`\EXE`) while keeping assets organized alphabetically.

---

## 5. Multi-Language & Localization System
PhotoPlay's engine supports an extensive range of international localizations.

### Language Mapping (`\FOTO\LOCALIZE`)
The system contains definitions for **33 different languages**:
- **`LANGUAGE.CSV`**: Lists 33 entries (including `SPA`, `ENG`, `GER`, `RUS`, `JPA`, `CHI`, etc.) and maps them to their respective codepages for character rendering.
- **`LNGNAME.CSV`**: Provides the native name for all 33 supported languages.

### Installed vs. Supported Languages (`\FOTO\SETTINGS`)
While the *engine* supports 33 languages, this specific disk installation only contains the data files for a subset:
- **String Tables (.DBF)**: On this disk, we find translation databases for **SPA** (Español), **ENG** (English), **GER** (Deutsch), **ITA** (Italiano), **FRE** (Français), and **POR** (Português).
- **`GAMES.DBF`**: Stores the localized game descriptions for these specific languages.

### Selection Constraints
- **UI Limit**: Although multiple language packs can be installed, the operator menu and `SYSTEM.INI` configuration typically limit the machine to **3 active languages** at a time for the user-facing "Language Toggle" button.

### Active Selection
The machine's current language preference is stored in:
1.  **`Main.set`**: Contains the binary index of the selected language.
2.  **`SYSTEM.INI`** (in `\MENU`): Defines the available language set for the user to toggle (e.g., `SPA ENG,POR,SPA`).

---

## 6. Security & Hardware Protection
The PhotoPlay platform employs a hardware-based security system designed to prevent unauthorized distribution, enforce licensing, and uniquely identify machine hardware.

### Smart Card Dongle Mechanism
The system utilizes a **Smart Card Reader** (ISO 7816) typically connected to a serial interface (COM port).
- **The Card**: A security-hardened smart card (often in SIM form factor) acts as a **Secure Access Module (SAM)**. It contains protected memory for license keys and a cryptographic co-processor.
- **Protocol**: The software communicates via a **Challenge-Response** protocol. The application sends a unique "challenge" to the card, which calculates a signed response using an internal, non-exportable private key.
- **Identity Enforcement**: The card stores the cabinet's unique serial number and its authorized "Machine Class" (e.g., Smart, Masters).

### Legacy Protection (PTS-DOS Versions)
Earlier PTS-DOS versions used a combination of:
- **Dallas Button Dongle**: A 1-Wire iButton device for unique identification.
- **Parallel Dongle**: A hardware key connected to the **LPT port**.

### Multi-Level Software Integration
The protection is deeply integrated into the entire application stack:
1. **The Shell Handler (`MENU.EXE`)**: Performs the initial hardware handshake. If the device is unresponsive or the signature is invalid, it triggers an "illegal version" or "dongle not found" halt before the main GUI loads.
2. **Individual Game Binaries (`\EXE\*.EXE`)**: Every game executable contains its own verification routines. This ensures that hardware must be present for any gameplay to occur, even if the menu is bypassed.
3. **Dynamic Resource Decryption**: System resources (such as specific bitmaps) may require hardware-derived keys to be properly parsed by the engine.

### System Bypass (Bypass Architecture)
Hardware protection can theoretically be bypassed through targeted binary modification of the system executables.
- **Instruction Patching**: This involves locating the specific memory offsets for conditional jump instructions (e.g., `JZ` - Jump if Zero) that follow a security check. By replacing these with unconditional jumps (`JMP`) or `NOP` (No Operation) bytes, the software can be "forced" to ignore the absence of hardware.
- **Layered Bypass**: Because the protection is distributed, a complete system bypass requires patching both the main shell (`MENU.EXE`) and every individual game executable in the `\EXE` directory to ensure seamless operation without hardware.

---

## 7. Platform Generations & Evolution
The PhotoPlay platform evolved across several operating systems and hardware generations:

| Generation | Operating System | Hardware / Notes |
| :--- | :--- | :--- |
| **Original / Masters** | **PTS-DOS** | Initial versions (like IGO8). Used Dallas/Parallel dongles. |
| **NG1** | **Windows 98** | Next Generation 1. Transition to Windows-based GUI. |
| **NG2** | **Windows XP** | Next Generation 2. Modernized platform for later cabinets. |

---

## 8. Hardware Compatibility & Motherboard Detection
PhotoPlay includes an automated hardware discovery system used to load appropriate sound and memory drivers.

### Detection Mechanism (`\SOUND`)
Initialization is handled by `SND_INIT.BAT`, which triggers a signature scan:
- **`CHECKMB.EXE \SOUND\MOBO.CSV`**: Scans the system BIOS for a specific 8-character string.
- **`CHECKMB.BAT`**: Generated dynamically, it sets the `%MB%` environment variable with the detected motherboard ID.

### Supported Motherboards
The system explicitly identifies and handles the following hardware:
- **Zida Tomato II 486**: The primary Socket 3 hardware for late-DOS iterations.
    - **Specs**: 3 PCI Slots, 3 ISA Slots, 4MB-128MB SIMM RAM support.
    - **Sound**: Typically paired with an **ESS 1868** or **C-Media** chipset.
    - **Video**: **PCI VGA 1MB** dedicated card.
- **Chaintech 6WEV** (`EV69MC0C`): A Socket 370 motherboard using the **C-Media** sound chipset and a specialized memory manager (`HIMEM.6WE`).
- **Chaintech 5SFV** (`2A5IIC3A`): A Socket 7 motherboard using the **ESS 1868** sound chipset.
- **Zida Tomato 4DPS**: Older Socket 3 hardware, identified by labels `TOM1` and `TOM2`.
- **Lucky Star LS-486**: A common alternative for older cabinets, functionally very similar to the Zida 4DPS.
- **Legacy 386**: Includes support for an unknown 386 motherboard found in the original 1997 PhotoPlay units.
- **OEM Variants**: Includes support for other labels like `INFM`.

### Hardware-Specific Tasks
Depending on the detected board, the system executes custom scripts (`MB6WEV.BAT`, `MB5SFV.BAT`):
- **Sound Configuration**: Loads specific DOS drivers (e.g., `esscfg.exe`, `setaudio.exe`) and sets the `BLASTER` variable.
- **Quirk Fixes**: For the Chaintech 6WEV, the boot process explicitly runs `CKUSB.EXE` to disable USB interference with DOS resources.

---

## 9. Touch Screen Support
PhotoPlay depends entirely on touch input, supporting the two industry standards of the 1990s and early 2000s.

### Supported Controllers
- **SMT3 (Serial Touchscreen Controller)**: The official standard for PhotoPlay Smart cabinets.
    - **Mapping**: **COM 3** (Base: `3E8h`, IRQ: **3**).
- **Elo TouchSystems**: Historically supported via the `ELODEV.EXE` driver.
- **MicroTouch (3M)**: Supported via the `DOSTOUCH.EXE` driver.

### Calibration Logic (`\TOUCH`)
- **Initialization**: `TCH_INIT.BAT` loads the specific driver for the hardware installed in the cabinet.
- **Calibration Tool**: `MONITOR.EXE` (found in `\TOUCH\MT`) is the primary tool used by the system for touch alignment. When the main menu returns error level 42, the system triggers a 25-point calibration using these tools.

---

## 10. Modem & Remote Connectivity
PhotoPlay cabinets used a dial-up modem for terminal registration and remote statistics reporting via the **Photo Play net** (Fun.net) infrastructure.

### Hardware Interface
According to official PhotoPlay Smart documentation, the I/O mapping is:
- **COM 1 (fun.link)**: Base `3F8h`, IRQ **4**.
- **COM 2 (Data Print)**: Base `2F8h`, IRQ **3**.
- **COM 3 (TS-SERIELL)**: Base `3E8h`, IRQ **3**. Used for the **SMT3** Touchscreen Controller.
- **COM 4 (MODEM)**: Base `2E8h`, IRQ **10**. Used for remote communication.
- **LPT**: Used for the security **Dongle**.
- **FW 1**: 25-pin connection for **Coincontrol & Notereader**.
- **FW 2**: 15-pin connection for **Bookkeeping & Counter**.

### Physical & Power Specs
- **Power Supply**: 230V / 50Hz (90 VA).
- **Output Voltages (Pin/Color Code)**:
    - **+5V (20A)**: Red
    - **Power Good**: Orange
    - **-5V (0.5A)**: White
    - **+12V (8A)**: Yellow
    - **-12V (0.5A)**: Blue
- **Weight**: Approx. 65 kg (Stationary PhotoPlay 2000).
- **Audio**: 8 Ohm, 3-5 Watt Speakers.

### Technical Configuration (`\FN_SYS\DFU\NET.CFG`)
Remote connectivity is governed by a **PPP Link Driver** with the following parameters:
- **Baud Rate**: 57,600 bps.
- **Initialization String**: `ate0m1l3S8=5S7=20S10=40x3`.
- **Protocol**: Standard Point-to-Point Protocol (PPP).
- **Authentication**: Uses a pre-configured user profile (e.g., `funny150`) with encrypted credentials.

### Machine Registration (`SETTINGS.TAB`)
License status and machine identification are stored in `\FN_SYS\DATABASE\USER\SETTINGS.TAB`.
- **`machlic`**: A 12-to-16 character unique license key.
- **Registration Flags**: Boolean flags (`Registered`, `GENLIC`) determine if the machine can access networked features or if it remains in a "Wrong License" state.
- **`ppserialnumber`**: Usually follows a format like `2550150-13-xx`.

---

## 11. Persistence & Logging
- **`LOGGING.OUT`**: Found in the root directory. Used by the shell to trace errors and peripheral responses.
- **`.INI` files**: Used extensively for configuration (e.g., `SYSTEM.INI`, `BONUS.INI`).
- **RAM Drive**: Active logs and temporary buffers are often redirected to the 12MB RAM drive (Stage 1) to minimize physical disk writes.

---

## 12. Hardware Troubleshooting & Specific Quirks
Derived from original maintenance manuals (07-1998):

### Critical IC Check (U16)
- **Problem**: Frame or PCX loading errors.
- **Check**: The **IC U16** on the motherboard must be a **74LS244N**. If it is a **74F244N**, the motherboard may cause system instability or graphics errors.
- **Cable length**: The IDE/Interface ribbon cable for the hard drive must not exceed **35cm** in PhotoPlay 2000 units.

### Common Boot Failures
- **"Stuck at World Map"**: Usually indicates a conflict between plug-in cards. Disconnect all expansion cards except VGA and reconnect one by one to identify the faulty card.
- **Performance degradation**: Check for the message **"Eck funworld Bios"** at boot. If present, the BIOS must be configured as **"ENERGY BIOS"** to prevent CPU throttling.
- **Key Logic**: The system uses a mechanical key switch next to the red LED to "lock" the hard drive enclosure. If the key is not "Closed", the BIOS may report **"DRIVE NOT READY"**.
- **Touch Sensitivity**: Excessive "ghost clicks" or high sensitivity are often caused by a missing ground connection. Verify the **green ground wire** attached to the touch screen glass.

### Coin & Note Reader Logic
- **Multifunction Card**: Jumpers 3 and 8 must be set to **ON** for proper communication with the coin reader; all others should be **OFF**.
- **Printer**: Statistics can be printed via the **Dataprint 3000 S** connected to the parallel port.
