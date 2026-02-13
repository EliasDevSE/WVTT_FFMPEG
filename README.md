# WebVTT ISOBMFF Support for FFmpeg 7.0

Patches to add support for WebVTT subtitles stored in ISOBMFF boxes (codec tag `wvtt`) inside MP4/MOV containers.

## License

Use this however the hell you want to be honest, I just thought it was a stupid problem and was surprised no one came over it in 20 years. 

## Problem

FFmpeg 7.0 doesn't recognize wvtt subtitle tracks in MP4 files. They show up as:
```
Stream #0:2: Data: none (wvtt / 0x74747677)
```

## Solution

Two patches that add full wvtt read/write support:

| Patch | Description |
|-------|-------------|
| `wvtt-isobmff.patch` | Demuxer + decoder. Recognizes wvtt as subtitle, decodes ISOBMFF binary samples. |
| `wvtt-muxer.patch` | Muxer. Writes wvtt tracks to MP4 with proper vttC config box. |

## Applying the Patches

```bash
# Extract FFmpeg 7.0
tar -xjf ffmpeg-7.0.tar.bz2
cd ffmpeg-7.0

# Apply patches
patch -p1 < wvtt-isobmff.patch
patch -p1 < wvtt-muxer.patch
```

## Building

```bash
./configure --enable-gpl
make -j$(nproc)
```

## Usage

After patching, wvtt tracks show correctly:
```
Stream #0:2: Subtitle: webvtt (wvtt / 0x74747677)
```

### Extract subtitles from MP4
```bash
ffmpeg -i input.mp4 -map 0:s -c:s webvtt output.vtt
```

### Copy wvtt subtitles to another video (no re-encode)
```bash
ffmpeg -i video.mp4 -i source_with_wvtt.mp4 \
  -map 0:v -map 0:a -map 1:s \
  -c copy output.mp4
```

### Convert wvtt to mov_text (tx3g)
```bash
ffmpeg -i input.mp4 -map 0:v -map 0:a -map 0:s \
  -c:v copy -c:a copy -c:s mov_text output.mp4
```

## Files Modified

**wvtt-isobmff.patch:**
- `libavformat/isom.c` - adds wvtt to subtitle codec tags
- `libavformat/mov.c` - parses vttC config box in stsd
- `libavcodec/webvttdec.c` - decodes ISOBMFF vttc/vtte boxes

**wvtt-muxer.patch:**
- `libavformat/movenc.c` - writes vttC box, adds wvtt to MP4 codec tags

## Technical Details

wvtt samples in MP4 are binary ISOBMFF boxes per [ISO 14496-30](https://www.iso.org/standard/63107.html?__cf_chl_tk=1.QT0s2d75.re0rWEV258cyWyVP59nN7URWsxNXf8jE-1771012949-1.0.1.1-MhXLRXcp2gJcAAF8x.DrS3fciD_PyyBqoy1htqIQU00):
- `vttc` box contains cue data with `payl` (payload text), `sttg` (settings), `iden` (id)
- `vtte` box is an empty cue

The decoder detects binary format by checking if data starts with a valid vttc/vtte box header, then extracts the payload text. Plain-text WebVTT (from HLS/Matroska) still works unchanged.
