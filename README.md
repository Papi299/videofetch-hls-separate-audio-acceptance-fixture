# VideoFetch separate-audio HLS public acceptance fixture

A small, permanent, public HLS source for acceptance-testing VideoFetch's
**separate-audio fMP4 clear-HLS** support over the real internet. It is
synthetic test material only: a colour test pattern and a 440 Hz tone. There
is no user media, no third-party media, no personal data and nothing secret in
this repository.

Submitted page: <https://papi299.github.io/videofetch-hls-separate-audio-acceptance-fixture/>
(use the trailing slash).

## What it is for

It exercises exactly one supported shape end to end: public DNS, HTTPS and CDN
delivery; VideoFetch analysis, including VideoFetch fetching the Master
Playlist itself to prove which audio rendition belongs to the video variant;
playlist, map and fragment acquisition of both halves; and the timestamp-aware
merge of the two halves into one MP4.

The audio starts about 0.48 s after the video, on purpose. A correct merge
must keep that offset; a merge that zeroes each input separately would not.

## Supported shape (and nothing else)

```
index.html        HTML5 <video><source src="master.m3u8" type="application/x-mpegURL">
master.m3u8       one EXT-X-STREAM-INF video variant, AUDIO="aud", and exactly one
                  EXT-X-MEDIA TYPE=AUDIO rendition in GROUP-ID "aud", with URI
video/media.m3u8  video-only fMP4 rendition: finite VOD, one URI-only EXT-X-MAP,
                  4 fragments of 1 s (H.264 High@4.0, 1920x1080, 25 fps, no audio)
video/init_0.mp4  its initialization segment (ftyp, moov)
video/seg-N.m4s   its fragments (styp, sidx, moof, mdat)
audio/media.m3u8  audio-only fMP4 rendition: finite VOD, one URI-only EXT-X-MAP,
                  5 fragments (AAC-LC, mono, 48 kHz, no video)
audio/init_1.mp4  its initialization segment (ftyp, moov)
audio/seg-N.m4s   its fragments (styp, sidx, moof, mdat)
```

`master.m3u8`:

```
#EXTM3U
#EXT-X-VERSION:7
#EXT-X-INDEPENDENT-SEGMENTS
#EXT-X-MEDIA:TYPE=AUDIO,GROUP-ID="aud",NAME="Tone",LANGUAGE="en",DEFAULT=YES,AUTOSELECT=YES,CHANNELS="1",URI="audio/media.m3u8"
#EXT-X-STREAM-INF:BANDWIDTH=900000,AVERAGE-BANDWIDTH=724000,RESOLUTION=1920x1080,FRAME-RATE=25.000,CODECS="avc1.640028,mp4a.40.2",AUDIO="aud"
video/media.m3u8
```

There is one video variant, one audio group, and exactly one URI-bearing
audio rendition in it: no second rendition, no alternate language, no
subtitles, no I-frame playlist and no query strings. Neither media playlist
has `EXT-X-KEY`, `EXT-X-BYTERANGE`, a map `BYTERANGE`, `EXT-X-DISCONTINUITY`
or live/EVENT semantics.

## Claim boundary

Passing VideoFetch acceptance on this fixture shows that this one shape works
over real DNS, HTTPS and this CDN. It does **not** prove:

- arbitrary HLS or CMAF;
- multiple audio renditions, or a choice between them;
- language selection;
- MPEG-TS separate audio;
- mixed MPEG-TS and fMP4;
- encrypted (or DRM) HLS;
- byte-range HLS;
- live or EVENT playlists;
- discontinuities;
- compatibility with arbitrary CDNs or packagers.

## How it was made

The media and both media playlists are the unmodified output of the
`pair-audio-late` recipe in VideoFetch's HLS-12 release-acceptance harness:
`generateHls12Recipe({ recipeName: "pair-audio-late" })` in
[`deploy/acceptance/ytdlp-generic/fixtures/hls12-media.mjs`](https://github.com/Papi299/sail-lilac-comet-mint/blob/34c9567abc12ca9280f7fe9f22d703fa03f5199e/deploy/acceptance/ytdlp-generic/fixtures/hls12-media.mjs)
at commit `34c9567abc12ca9280f7fe9f22d703fa03f5199e`.

- **Generator:** `/usr/bin/ffmpeg` version `5.1.9-0+deb12u1` (Debian bookworm;
  libx264 `0.164.3095`) inside the qualified VideoFetch Worker release
  candidate image
  `sha256:262f5633bc38b28111ea6b14373747c050f198aed846b5b253ac5f0b3f187bf5`
  (tag `videofetch-worker:rc-34c9567abc12-262f5633bc38`). The container had no
  network and no secrets.
- **Command** (`<dir>` is the output directory; `v0` is the video rendition,
  `v1` the audio rendition):

  ```
  ffmpeg -hide_banner -nostdin -loglevel error -y -fflags +bitexact
    -f lavfi -i testsrc2=size=1920x1080:rate=25:duration=4
    -itsoffset 0.5 -f lavfi -i sine=frequency=440:sample_rate=48000:duration=4
    -map 0:v -map 1:a
    -c:v libx264 -preset veryfast -threads 1 -pix_fmt yuv420p -profile:v high -level 4.0
    -g 25 -keyint_min 25 -sc_threshold 0 -b:v 600k -maxrate 600k -bufsize 600k -flags:v +bitexact
    -c:a aac -b:a 64k -ac 1 -flags:a +bitexact
    -map_metadata -1 -f hls -hls_time 1 -hls_playlist_type vod -hls_segment_type fmp4
    -hls_flags independent_segments -hls_fmp4_init_filename init.mp4
    -var_stream_map "v:0,agroup:aud a:0,agroup:aud"
    -hls_segment_filename <dir>/v%v/seg-%d.m4s -master_pl_name packager-master.m3u8 <dir>/v%v/media.m3u8
  ```

- **Bit-exact:** a second run produced byte-identical files. The video and
  audio byte streams (map plus fragments) are identical to the HLS-12
  `pair-audio-late` renditions VideoFetch's release qualification used.
- **Copied verbatim:** `v0/` became `video/` and `v1/` became `audio/`.
  FFmpeg names the two maps `init_0.mp4` and `init_1.mp4` when it writes two
  renditions in one run, and the generated playlists name them that way, so
  they were not renamed. The packager's own master was not published.
- **Hand-written:** `master.m3u8` and `index.html`, in the same shape as the
  harness's `hls12Master(…, "paired")` and `hls12Page`. The harness's private
  test markers, signed queries and loopback addresses were replaced by plain
  public values. `avc1.640028` matches the video map's `avcC` record (High
  profile, level 4.0), and `mp4a.40.2` is AAC-LC. `BANDWIDTH=900000` is above
  the measured peak segment bit rate of the pair (833,097 bit/s).
  `AVERAGE-BANDWIDTH=724000` is the measured average (723,713 bit/s) rounded
  up.

## Measured source timing

Measured from packet timestamps with VideoFetch's shared packet-timing oracle
(`pairSourceTiming` in `deploy/acceptance/ytdlp-generic/lib/merge-timing.mjs`),
on each rendition's map plus fragments:

| | First presented packet | Packets |
| :--- | :--- | :--- |
| Video | 80,000 µs | 100 |
| Audio | 558,000 µs | 189 |
| **Audio minus video** | **+478,000 µs** | |

A correct merge keeps the video as the synchronization reference and the audio
478 ms after it.

## Integrity

`SHA256SUMS` lists every file the acceptance path serves: `index.html`,
`master.m3u8`, both media playlists, both maps and every fragment (15
entries, sorted by path). It does not list itself, `README.md` or
`.nojekyll`.

Acceptance evidence binds to those digests, so **do not modify these files**.
Any change needs a new commit and a new verification, and a different fixture
belongs in a new directory or repository.
