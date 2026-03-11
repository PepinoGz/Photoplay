# PhotoPlay .HIS High Score Format Specification

This document details the binary structure of the high score files (`.HIS`) used in PhotoPlay by Funworld systems.

## Overview
The `.HIS` files are fixed-length binary databases used to store the Top 10 rankings for individual games. Each file is exactly **434 bytes** in size and uses a **parallel array** structure rather than interleaved records.

### File Locations & Naming
- **Directory**: `\FOTO\HISCORE`
- **Naming Convention**: `[GAMENAME].HIS` (e.g., `SHANGHAI.HIS`, `FINDIT.HIS`)

## Ranking Behavior (Service Manual)
- **Automatic Deletion**: The system can be configured to clear high scores automatically after a set period or manually from the Setup menu.
- **Top Scorer Title**: Only players reaching a specific score threshold (relative to current rankings) are prompted to enter their name.
- **All-Time List**: Accessible via Info (button)-> ALL TIME HI-SCORE (button) which displays the absolute Top 10 across the system's history without name entry in the preview.

> [!IMPORTANT]
> This documentation is based on data extracted from a **Version 2008 (ES) NSB IGO8 VM007** system image. While other IGO versions are assumed to follow a similar structure, this has not been verified for older or different regional variants.

## Data Layout Table

| Block Name | Start Offset (Hex) | End Offset (Hex) | Length (Bytes) | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Header/Padding** | `0x000` | `0x067` | 104 | System-specific header and identification strings. |
| **Score Array** | `0x068` | `0x08F` | 40 | 10 x 4-byte Little Endian integers. |
| **Name Array** | `0x090` | `0x157` | 200 | 10 x 20-byte chunks. **Limit: 14 chars**. |
| **Date Array** | `0x158` | `0x17F` | 40 | 10 x 4-byte Little Endian integers (Days since epoch). |
| **Language Tags** | `0x180` | `0x1B1` | 50 | 10 x 5-byte fixed-length ASCII strings (e.g., "SPA", "ENG"). |
| **Footer** | `0x1B2` | `0x1B1` | 158 | Miscellaneous settings or checksum padding. |

---

## Detailed Data Types

### 1. Scores (`0x68`)
Stored as 32-bit unsigned integers in Little Endian format. 
- Example: `12373` is stored as `5D 30 00 00`.

### 2. Names (`0x90`)
Physically, each record is **20 bytes** long, but the PhotoPlay engine enforces an **effective limit of 14 characters** in the UI. 
- The 20-byte allocation allows for 19 characters + null, but the last 5-6 bytes are typically unused (`0x00`).
- If a name is shorter than 20 bytes, it is padded with null bytes.
- Default names (e.g., "ALBERTO", "MARIA") are physically written into this block during a factory reset.

### 3. Dates (`0x158`)
PhotoPlay uses a custom epoch for its date system to optimize storage space. Instead of storing full strings or complex structures, the system stores a single 4-byte integer representing the number of days elapsed since a fixed "Starting Point" or **Day 0**.

- **Epoch**: `1990-01-01`
- **Purpose**: Storage efficiency. Using a 4-byte integer (`uint32`) allows the system to store dates for over 11,000 years using only 4 bytes per record.
- **System Clock**: The machine's internal clock (BIOS) can be set to any modern date (e.g., 2026). When saving a score, the game engine calculates the difference: `Stored_Value = (Current_Date - 1990-01-01) + 1`. This is because PhotoPlay systems treat the Epoch as Day 1.
- **Calculation**: 
  - `Actual_Date = datetime(1990, 1, 1) + timedelta(days=value - 1)`
- **Example**: 
  - Stored Hex Value: `A2 33 00 00` (Decimal: `13218`)
  - Calculation: `1990-01-01 + (13218 - 1) days` = **2026-03-10**.

### 4. Language Tags (`0x180`)
These tags indicate the system language setting at the time the record was created.
- `SPA`: Spanish
- `ENG`: English
- `GER`: German
- `FRA`: French
- The tags are 5 bytes long, null-terminated.

---

## Technical Notes
- **Endianness**: Little Endian (Intel x86 standard).
- **Encoding**: Standard ASCII.
- **Checksums**: There are no apparent header/footer checksums; the game engine appears to read the raw offsets directly.
- **Record Count**: The system is strictly limited to 10 records per file.

## Implementation (Python Example)
```python
import struct
import os
import sys
from datetime import datetime, timedelta

# PhotoPlay High Score Extractor (HIS Format)
# Epoch confirmed for PhotoPlay systems (1990-01-01 = Day 1)
EPOCH = datetime(1990, 1, 1)

def parse_his_file(path):
    if not os.path.exists(path):
        return None
    
    with open(path, 'rb') as f:
        data = f.read()
    
    if len(data) < 434:
        return None

    ranking = []
    # PhotoPlay .HIS files use parallel arrays for the Top 10 records
    for i in range(10):
        # 1. Scores (4-byte Little Endian integers starting at 0x68)
        score_off = 0x68 + (i * 4)
        score, = struct.unpack('<I', data[score_off:score_off+4])
        
        # 2. Names (20-byte ASCII strings starting at 0x90)
        name_off = 0x90 + (i * 20)
        name_bytes = data[name_off:name_off+20].split(b'\x00')[0]
        name = name_bytes.decode('ascii', errors='ignore').strip()
        
        # 3. Dates (4-byte integers representing days since 1990-01-01 at 0x158)
        date_off = 0x158 + (i * 4)
        days, = struct.unpack('<I', data[date_off:date_off+4])
        # Legacy systems often treat Day 1 as the Epoch itself; Python adds to Epoch.
        date_obj = EPOCH + timedelta(days=days-1)
        
        # 4. Language/Locale (5-byte ASCII tags starting at 0x180)
        lang_off = 0x180 + (i * 5)
        lang = data[lang_off:lang_off+5].split(b'\x00')[0].decode('ascii', errors='ignore')

        if name or score > 0:
            ranking.append({
                'rank': i + 1,
                'name': name,
                'score': score,
                'date': date_obj.strftime('%d/%m/%Y'),
                'lang': lang
            })
            
    return ranking

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: python extract_scores.py <file.HIS>")
        sys.exit(1)
        
    path = sys.argv[1]
    res = parse_his_file(path)
    if res:
        game_name = os.path.basename(path).replace('.HIS', '').upper()
        print("-" * 65)
        print(f" RANKING: {game_name}")
        print("-" * 65)
        print(f"{'POS':4} | {'NAME':20} | {'SCORE':10} | {'DATE':12} | {'LANG'}")
        print("-" * 65)
        for r in res:
            print(f"{r['rank']:<4} | {r['name']:20} | {r['score']:<10} | {r['date']:12} | {r['lang']}")
        print("-" * 65)
    else:
        print(f"Error: Could not parse {path}")
```

## Usage
To extract rankings from any PhotoPlay game, run the script passing the path to the `.HIS` file:

```bash
python extract_scores.py SHANGHAI.HIS
```

### Expected Output
```text
-----------------------------------------------------------------
 RANKING: SHANGHAI
-----------------------------------------------------------------
POS  | NAME                 | SCORE      | DATE         | LANG
-----------------------------------------------------------------
1    | ABCD                 | 12373      | 10/03/2026   | SPA
2    | EFGH                 | 9595       | 10/03/2026   | SPA
...
-----------------------------------------------------------------
```
