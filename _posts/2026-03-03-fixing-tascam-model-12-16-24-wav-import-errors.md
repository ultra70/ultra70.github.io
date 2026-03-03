---
layout: post
title: "Fixing Tascam Model 12 / 16 / 24 WAV Import Errors"
date: 2026-03-03
categories: [audio, linux]
tags: [tascam, wav, ffmpeg, sox, model-12, model-16, model-24, pioneer, akai, roland]
pin: false
toc: true
math: false
mermaid: false
---

If importing a 24-bit WAV file into a Tascam Model 12, Model 16, or Model 24 has resulted in "File Error" with no further explanation, the problem could be in the WAV header, not the audio data.

## Table of Contents

- [Symptoms](#symptoms)
- [Root Cause](#root-cause)
- [Why Your Tools Produce This Header](#why-your-tools-produce-this-header)
  - [ffmpeg](#ffmpeg)
  - [SoX](#sox)
- [The Fix: wav2tascam.py](#the-fix-wav2tascampy)
- [Import Workflow](#import-workflow)
- [Other Affected Hardware](#other-affected-hardware)


## Symptoms

You have a 24-bit, 48 kHz or 44.1 kHz WAV file.  The Tascam owner's manual says the device supports this format ([Tascam Model 12 manual, p. 46](https://www.manualslib.com/manual/2088623/Tascam-12.html?page=46)).  You copy it to the SD card's `MUSIC` folder, navigate to MENU > MTR > TRACK EDIT > IMPORT, and the display shows "File Error."  The manual defines this only as "the file format is not supported" ([Tascam Model 12 manual, p. 59](https://www.manualslib.com/manual/1797080/Tascam-12.html?page=59)).

The file plays correctly in every other application.  Re-exporting at different sample rates does not help.  The SD card is formatted correctly.  The Song sample rate matches the WAV file.

## Root Cause

The WAV file format uses a `fmt` chunk to describe its audio encoding.  There are two relevant variants of this chunk.

WAVE\_FORMAT\_PCM (format tag 0x0001) uses a 16-byte `fmt` chunk.  WAVE\_FORMAT\_EXTENSIBLE (format tag 0xFFFE) extends the `fmt` chunk to 40 bytes by appending a 22-byte extension containing `cbSize`, `wValidBitsPerSample`, `dwChannelMask`, and a 16-byte SubFormat GUID.  Microsoft introduced WAVE\_FORMAT\_EXTENSIBLE around Windows 98 SE to resolve ambiguity between container size and actual sample precision at higher bit depths ([Microsoft WAVEFORMATEXTENSIBLE](https://learn.microsoft.com/en-us/windows/win32/api/mmreg/ns-mmreg-waveformatextensible)).

According to the Microsoft "Multiple Channel Audio Data and WAVE Files" white paper, WAVE\_FORMAT\_EXTENSIBLE should be used when PCM data exceeds 16 bits per sample, when the channel count exceeds 2, when valid bits per sample differs from container size, or when channel-to-speaker mapping is needed ([Microsoft white paper](https://learn.microsoft.com/en-us/previous-versions/windows/hardware/design/dn653308(v=vs.85))).  The McGill University WAV format reference documents the full structure of both variants ([McGill WAV reference](https://www.mmsp.ece.mcgill.ca/Documents/AudioFormats/WAVE/WAVE.html)).

The Tascam firmware only parses the 16-byte `fmt` chunk with format tag 0x0001.  When it encounters 0xFFFE, it rejects the file.  The underlying PCM audio data is identical in both cases; only the header differs.

This was first diagnosed at the hex level by user Sam Trenholme on the Tascam forums in 2015, who compared working and failing WAV files on a Tascam DP-32SD ([tascamforums.com, 2015](https://www.tascamforums.com/threads/dp-32sd-can-not-%E2%80%9Cgrok%E2%80%9D-sox-generated-wav-file.3565/)).  The same behavior has been reported on the Model 24 ([Model 24 thread](https://www.tascamforums.com/threads/model-24-song-file-error-cannot-record.9402/)) and DP-24SD ([DP-24SD thread](https://www.tascamforums.com/threads/problem-importing-wav-file.4172/)).  No Tascam firmware update has addressed this as of Model 12 firmware v1.50.

## Why Your Tools Produce This Header

Both ffmpeg and SoX follow the Microsoft specification and default to WAVE\_FORMAT\_EXTENSIBLE for 24-bit audio.  This is technically correct per the spec, but it produces files that Tascam hardware rejects.

16-bit WAV files are not affected.  ffmpeg and SoX use format tag 0x0001 for 16-bit audio and those files import without issue.

### ffmpeg

The format tag decision is in `libavformat/riffenc.c` in the `ff_put_wav_header()` function.  The code uses WAVE\_FORMAT\_EXTENSIBLE when any of these conditions is true: more than 2 channels with a channel layout set, sample rate exceeding 48,000 Hz, EAC3 codec, or bits per sample exceeding 16 ([ffmpeg source](https://github.com/FFmpeg/FFmpeg/blob/master/libavformat/riffenc.c)).

There is no ffmpeg option to override this.  The WAV muxer exposes options for `write_bext`, `write_peak`, `rf64`, and peak envelope settings, but nothing controls format tag selection ([ffmpeg WAV muxer source](https://github.com/FFmpeg/FFmpeg/blob/master/libavformat/wavenc.c)).

ffmpeg ticket #4426 requested support for writing 24-bit WAV with format tag 0x0001.  The ticket was closed as not a bug, with an ffmpeg developer stating that Microsoft's WAVEFORMATEX documentation restricts bits per sample to 8 or 16 with WAVE\_FORMAT\_PCM and that ffmpeg should not create out-of-spec files ([ffmpeg ticket #4426](https://trac.ffmpeg.org/ticket/4426)).

### SoX

SoX follows the same logic, switching to WAVE\_FORMAT\_EXTENSIBLE when `wBitsPerSample > 16` or `wChannels > 2` ([SoX source wav.c](https://github.com/chirlu/sox/blob/master/src/wav.c)).

SoX developer Mans Rullgard confirmed on the sox-devel mailing list that SoX is correct in creating a WAVEFORMATEXTENSIBLE header for 24-bit data, but that the Tascam firmware cannot parse it ([sox-devel](https://public-inbox.org/sox-devel/yw1xtwqosehb.fsf@unicorn.mansr.com/t/)).

Unlike ffmpeg, SoX provides a workaround: the `-t wavpcm` output type forces the standard 16-byte WAVE\_FORMAT\_PCM header even for 24-bit audio ([soxformat(7)](https://manpages.ubuntu.com/manpages/trusty/man7/soxformat.7.html)).  This feature was added in SoX 14.0.1 (January 2008), prompted by SourceForge Bug #88, where a user reported that GoldWave and Sequoia/Samplitude could not open SoX 24-bit WAV files ([SoX Bug #88](https://sourceforge.net/p/sox/bugs/88/)).

```bash
sox input.flac -b24 -t wavpcm output.wav
```

This works, but it requires SoX to be installed and the user to know about the `-t wavpcm` flag.

## The Fix: wav2tascam.py

[wav2tascam](https://github.com/ultra70/media/wav2tascam), reads a WAV file, extracts the raw PCM data, and writes a new WAV file with a standard 16-byte `fmt` chunk and format tag 0x0001.  Sample rate, bit depth, channel layout and audio data are preserved from the source file.  It handles both WAVE\_FORMAT\_PCM and WAVE\_FORMAT\_EXTENSIBLE input, so running it on a file that already has the correct header produces an identical output.

```bash
./wav2tascam.py input.wav output.wav
```

If the output filename is omitted, it writes to `<input_basename>_tascam.wav` in the current directory.

### Import Workflow

1. Run `wav2tascam.py` on your WAV file.
2. Connect the Model 12 via USB and enter Storage mode, or eject the SD card.
3. Copy the converted file to the `MUSIC` folder at the root of the SD card.
4. Safely unmount/eject.
5. On the Model 12: MENU > MTR > TRACK EDIT > IMPORT.
6. Select the file and assign it to an empty track.

The Song sample rate on the Model 12 must match the file sample rate.  Target tracks must be empty before import (use TRACK CLEAR if needed).  Stereo files import to a track pair and require two empty tracks.

## Other Affected Hardware

This is not specific to Tascam.  The same WAVE\_FORMAT\_EXTENSIBLE rejection occurs on other hardware.

Pioneer CDJ-2000NXS/NXS2, XDJ-1000, and CDJ-900 throw error E-8305 "Unsupported File Format" with 0xFFFE headers.  A Pioneer staff member confirmed the issue and recommended hex-editing bytes 20-21 from `FE FF` to `01 00` ([Pioneer DJ forum](https://forums.pioneerdj.com/hc/en-us/community/posts/4410137950233-Changing-the-WAV-Header-to-meet-CDJs-standard)).

Akai MPC Live, One, and X refuse to load WAVE\_FORMAT\_EXTENSIBLE files ([Akai MPC Forums](https://www.mpc-forums.com/viewtopic.php?t=211783)).

Roland TM-2, SPD-SX, and RC-series loop stations report "Unsupported Format" with non-standard WAV headers ([Roland TM-2 support](https://support.roland.com/hc/en-us/articles/217375646-TM-2-Unsupported-Format-Message), [Roland SPD-SX support](https://support.roland.com/hc/en-us/articles/201936049-SPD-SX-Why-does-my-SPD-SX-display-UNSUPPORTED-FORMAT-when-I-try-to-import-an-audio-file)).

`wav2tascam.py` will work for all of these devices since the fix is the same: rewrite the format tag from 0xFFFE to 0x0001.
