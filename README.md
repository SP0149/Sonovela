# Sonovela — Firmware Overview

Sonovela is a clip-on book companion device that plays author-curated music synced to your current page in a book. Each book's soundtrack is associated with a physical mini vinyl record containing an NFC tag. The reader sets their starting page via a dial, and Sonovela autonomously plays the correct song based on estimated reading progress.

---

## How it works

### 1. Vinyl detection (NFC)
The PN532 NFC reader polls continuously for a vinyl record placed in the dock. When a vinyl is detected, its unique tag ID is read and used to load the corresponding book's page map and audio files from the microSD card. When the vinyl is removed, playback pauses and the device waits for a new vinyl.

### 2. Page map
Each book in Sonovela's catalog has a page map file stored on the microSD card. The file is a CSV structured as follows:

```
total_pages: 287
1,47,song1.wav
48,93,song2.wav
94,141,song3.wav
...
```

Each row defines a page range and the corresponding audio file to play. The `total_pages` field is included for future use (reading progress tracking, companion app integration).

### 3. Reading pace calibration (soft, continuous)
At the start of a reading session, the reader sets the dial to their current page. Sonovela records this as the baseline. Each time the reader nudges the dial to a new page, Sonovela calculates the pace for that interval:

```
currentPace = pagesMoved / timeElapsed
averagePace = (averagePace * 0.7) + (currentPace * 0.3)
```

The weighted average ensures a single slow or fast session does not dramatically skew the overall estimate. No formal calibration step is required — the estimate gets more accurate the more the reader uses the device.

### 4. Page estimation
Between manual dial nudges, Sonovela estimates the reader's current page using elapsed time and their calculated pace:

```
estimatedPage = lastSetPage + (elapsedMinutes * averagePace)
```

When the reader nudges the dial, the baseline resets:
```
lastSetPage = newDialPage
lastDialUpdateTime = now
```

### 5. Song selection
On every loop iteration, the estimated current page is looked up against the page map to determine which song should be playing:

```
function getSongForPage(page):
  for each row in pageMap:
    if page >= row.startPage AND page <= row.endPage:
      return row.filename
  return null
```

If the result differs from the currently playing song, the device fades out the current track and begins playing the new one.

### 6. Audio playback
Audio files (WAV format) are streamed from the microSD card over I2S to the PCM5102 DAC, then amplified by the MAX98357 and output via the 3.5mm headphone jack. Playback uses the ESP32-audioI2S library. The `audio.loop()` function runs continuously to ensure uninterrupted streaming.

### 7. Dual core architecture
To prevent NFC polling and encoder reading from blocking the audio stream, the ESP32's two cores are used independently:

- **Core 0** — continuous audio streaming (`audio.loop()`)
- **Core 1** — NFC polling, encoder reading, page estimation, song selection logic

Cores communicate via a shared variable (current song filename) protected by a mutex to avoid race conditions.

---

## Core loop pseudocode

```cpp
// Core 1 — logic
void logicTask() {
  while (true) {
    // NFC polling
    nfcTag = nfc.readTag();
    if (nfcTag != currentVinyl) {
      if (nfcTag == null) {
        pausePlayback();
      } else {
        loadPageMap(nfcTag.id);
        currentVinyl = nfcTag.id;
      }
    }

    // Encoder reading
    newPage = readEncoderPosition();
    if (newPage != lastSetPage) {
      updatePace(newPage, lastSetPage, timeSinceLastUpdate);
      lastSetPage = newPage;
      lastDialUpdateTime = millis();
    }

    // Page estimation
    elapsedMinutes = (millis() - lastDialUpdateTime) / 60000.0;
    estimatedPage = lastSetPage + (elapsedMinutes * averagePace);

    // Song selection
    newSong = getSongForPage(estimatedPage);
    if (newSong != currentSong) {
      currentSong = newSong;
      playSong(newSong);  // triggers audio on Core 0
    }

    delay(500);  // poll every 500ms
  }
}

// Core 0 — audio
void audioTask() {
  while (true) {
    audio.loop();
  }
}
```

---

## File structure on microSD

```
/sonovela
  /books
    /if_you_could_see_the_sun
      pagemap.csv
      song1.wav
      song2.wav
      song3.wav
    /another_book
      pagemap.csv
      song1.wav
      ...
  /nfc_index.csv   ← maps NFC tag IDs to book folder names
```

### nfc_index.csv format
```
tag_id,book_folder
04A3F2,if_you_could_see_the_sun
09B1C4,another_book
```

---

## Hardware summary

| Component | Role |
|---|---|
| ESP32-WROOM-32D | Main MCU, dual core processing |
| PN532 | NFC reader, vinyl detection |
| PCM5102 | DAC, digital to analog audio conversion |
| MAX98357 | Amplifier, drives headphone jack |
| MicroSD card | Stores page maps and WAV audio files |
| EC11 rotary encoder | Manual page dial input |
| DRV8833 | Motor driver for vinyl spin motor |
| DC pancake motor | Spins vinyl for aesthetics |
| TP4056 | USB-C battery charging |
| MT3608 | Boost converter, 3.7V → 5V |
| 18650 Li-ion cell | Power source |

---

## Pin assignments (ESP32)

| Pin | Function |
|---|---|
| IO25 | I2S BCLK (DAC + Amplifier) |
| IO26 | I2S LRCLK (DAC + Amplifier) |
| IO27 | I2S DIN (DAC + Amplifier) |
| IO21 | I2C SDA (NFC reader) |
| IO22 | I2C SCL (NFC reader) |
| IO18 | SPI CLK (microSD) |
| IO23 | SPI MOSI (microSD) |
| IO13 | SPI MISO (microSD) |
| IO5 | SPI CS (microSD) |
| IO14 | Motor PWM (DRV8833 AIN1) |
| IO33 | Motor direction (DRV8833 AIN2) |
| IO34 | Encoder A |
| IO35 | Encoder B |
| IO19 | Encoder switch |

---

## Future considerations
- Companion app integration for reading pace analytics and progress tracking
- Reading streaks and estimated time to finish
- Expanded book catalog with author-curated playlists
- Tonearm mechanism with friction wheel encoder for automatic page tracking (v2)
- Song fade transitions between page ranges
