# Analysis and Reverse Engineering of the PhotoPlay WAD Format

## 1. Overview of the `FOTOPLAY.WAD` file
In PhotoPlay arcade systems (based on PTS-DOS), game assets, sounds, and graphics are stored in archive files named `FOTOPLAY.WAD`. These files act as containers for multiple resources. While they might look like standard archives, they use a custom **XOR-based encryption** and obfuscation layer to prevent direct extraction.

## 2. GWAD File Structure

The file is divided into two main sections:
1. **Directory Table (Table of Contents)**: Located at the beginning of the file.
2. **Data Blocks (The actual files)**: Stored consecutively following the directory.

### 2.1 Directory Decryption
The entire directory/index section of the `.WAD` file is obscured using a simple **binary XOR operation with the static key `0x55`** (decimal 85).

Applying XOR `0x55` to the first few kilobytes (usually up to 64KB) reveals the following structure:
- **Magic Number** (4 bytes): `GWAD`
- **File Count** (4 bytes, Little Endian): Number of files contained in the archive.
- **File Entries** (21 bytes each):
  - **Filename** (13 bytes): Null-terminated ASCII string (e.g., `SOUND.WAV\x00`).
  - **Offset** (4 bytes, Little Endian): The starting position of the file within the `.WAD`.
  - **Size** (4 bytes, Little Endian): The exact byte size of the file.

### 2.3 Animation and Video Files (.FLC / .FLI)
Unlike graphics and audio, animation files in **Autodesk Animator** format (`.FLC` and `.FLI`) are typically **not encrypted** inside the GWAD containers. These formats were widely used in the 90s for intros, demos, and interactive tutorials.

Modern players like **VLC Media Player** or tools like **FFmpeg** can open and convert these files directly once extracted from the archive.

### 2.4 Typical File Locations
The most common filename for these archives is `FOTOPLAY.WAD`. They are found inside the category and game folders of the system disk.

**Example from the game "SHANGHAI":**
- `HardDisk/SHANGHAI/SFX/FOTOPLAY.WAD` (Sound effects)
- `HardDisk/SHANGHAI/GFX/FOTOPLAY.WAD` (General graphics)
- `HardDisk/SHANGHAI/GFX/FIELDS/FOTOPLAY.WAD` (Game board tiles)
- `HardDisk/SHANGHAI/BACKGR/FOTOPLAY.WAD` (Background images)

The system is highly modular; each subdirectory usually has its own `FOTOPLAY.WAD` containing the specific assets for that module.

## 3. Asset Decryption Strategy
Not every file in a GWAD archive is encrypted. Formats like animations (`.FLC`, `.FLI`) and raw data files (`.BIN`) are typically stored in plain text at their respective offsets.

However, the primary game assets—**Images (`.PCX`)** and **Audio (`.WAV`)**—use a targeted obfuscation method. To break standard file viewers, the system **encrypts only the file header**, leaving the actual data payload (compressed pixels or raw audio samples) intact.

#### Decryption Keys
By performing a *Known-Plaintext Attack* (comparing encrypted files against the expected standard headers for PCX and WAV formats), the following XOR keys were recovered:

**Standard Audio Header (`.WAV`):**
The first **44 bytes** (standard RIFF/WAVE PCM header) are encrypted.
*XOR Key for WAV (44 bytes):*
```hex
D8 43 3B 20 C7 F9 62 D6 C8 4C DD 75 61 DE 61 F7 
87 4F A7 B1 2B A0 08 CB DE 3D 61 14 18 59 8D F6 
2D 8A 28 98 17 D2 62 70 C1 7B 07 EC
```
*Note: This specific key preserves the original SampleRates of the audio files.*

**Standard Image Header (`.PCX`):**
The first **128 bytes** (standard ZSoft PCX header) are encrypted.
*XOR Key for PCX (128 bytes):*
```hex
D8 43 3B 20 CF F9 62 D6 C8 4C DD 75 29 DF 29 F6 
87 47 BF 98 33 80 28 D3 CE 6F 58 25 31 68 B4 FE 
0D 92 30 80 07 93 72 68 84 42 36 29 74 EF 0B CA 
4E D4 1C C2 3A 02 1C 06 B3 29 CC 2D F7 49 94 82 
AC 9B BF 0C 99 3A 55 8F FC E6 34 92 FB F7 E7 CB 
01 8C FA 4A 75 09 03 E4 1D F0 9C BD E6 D3 CF AF 
21 45 74 9C 4E F4 6B FF 18 5C C8 3A 1C BF 53 7F 
93 46 EE 4E 06 86 CD E0 93 C5 86 08 7B B4 6B 55
```

## 3. Extraction and Conversion
To extract the files, one must:
1. Load the archive and apply XOR `0x55` to the directory to get the file list.
2. For each file, seek to the `Offset`.
3. If the extension is `.PCX` or `.WAV`, apply the respective XOR key to the first 128 or 44 bytes.
4. Keep the remaining part of the file as-is.
5. Save the resulting data.

The decrypted outputs are standard files that can be opened by any modern media player or image editor. PCX files can be easily converted to PNG once the header is restored.
