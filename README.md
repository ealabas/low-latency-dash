# Low-Latency DASH Streaming

A low-latency live video streaming pipeline that delivers real-time video from
a capture device to a browser player with sub-5-second end-to-end latency
(CS418 course project).

OBS captures and encodes the stream and pushes it over RTMP to a patched
FFmpeg, which packages it into chunked LL-DASH (CMAF) segments. NGINX serves
the manifest while a modified node-gpac-dash pushes each fragment to a dash.js
player over HTTP/1.1 chunked transfer, so the player renders fragments as
they're produced instead of waiting for whole segments. Latency is measured by
burning a QR-coded timestamp into the video with OBS and decoding it on the
player side.

## Setup

FFmpeg is built from source with a one-line patch to `libavformat/dashenc.c`
that drops the `.tmp` rename, so node-gpac-dash can read segments while they're
still being written.

Run in order — OBS first (start streaming), then:

```bash
# NGINX serves the player, MPD and init segments
sudo nginx

# node-gpac-dash pushes fragments to the player (chunks-per-segment = seg/frag)
node node-gpac-dash/gpac-dash.js -chunk-media-segments -cors -chunks-per-segment 4

# patched FFmpeg listens for OBS's RTMP stream and writes LL-DASH
~/bin/ffmpeg -f flv -listen 1 -i rtmp://127.0.0.1:1935/live/app \
  -c:v copy -an \
  -ldash 1 -streaming 1 -use_template 1 -use_timeline 0 \
  -seg_duration 4 -frag_duration 1 -frag_type duration \
  -utc_timing_url "https://time.akamai.com/?iso" \
  -window_size 15 -extra_window_size 15 -remove_at_exit 1 \
  -f dash /var/www/html/ldash/1.mpd
```

Then open the player at `http://localhost:8080` — latency should settle under
5 seconds.

> `-chunks-per-segment` is `seg_duration / frag_duration` (4 here). Adjust it
> whenever you change the segment or fragment duration.

## Results

Two sweeps measured how latency responds to segment and fragment duration
(QR method: system clock − timestamp decoded from the frame).

Segment duration barely matters past ~3s — chunked transfer delivers each
fragment as it's written, so the bottleneck is fragment duration, not the
segment boundary. Shrinking fragments lowers latency all the way to 3.15s at
0.1s, but below 0.1s node-gpac-dash can't keep up with FFmpeg and the player
stalls.

| frag_duration (s) | End-to-end latency (s) |
|------------------:|-----------------------:|
| 0.1  | 3.15 |
| 0.2  | 3.78 |
| 0.5  | 3.91 |
| 1.0  | 4.51 |
| 2.0  | 5.36 |
| 4.0  | 7.81 |

*(seg_duration fixed at 4s; below 0.1s the pipeline overloads and stalls.)*

Full analysis, both experiments, and figures:
[CS418_Project_3.pdf](./CS418_Project_3.pdf)
