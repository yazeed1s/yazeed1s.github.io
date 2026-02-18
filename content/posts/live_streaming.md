+++
title = "How Live Streaming Actually Works"
date = 2025-12-22
description = "The full pipeline from OBS to your screen, and where it gets expensive."
[taxonomies]
tags = ["Streaming", "Video", "Infrastructure", "Networking"]
+++

I spent a lot of time thinking about this because of Strimo. Most people look at a Twitch stream and think it's just video going from a camera to their browser, but between the streamer's screen and yours there's encoding, protocol negotiation, transcoding into multiple qualities, segmentation into tiny files, CDN distribution, and adaptive playback — each step adding latency and cost.

## the pipeline

A live stream passes through roughly five stages: the streamer's machine captures and encodes the video, then pushes it over RTMP to the platform's ingest server, which transcodes it into several quality levels, chops it into HLS segments, distributes those segments through a CDN, and finally the viewer's player pulls them and stitches them back together.

Add up the latency from each stage and you land somewhere between 5 and 15 seconds of delay on most platforms. Some of that (or most of it) is unavoidable physics, or buffering decisions, or the platform choosing to trade latency for cheaper infrastructure.

## capture and encoding

The streamer runs OBS or something similar. OBS records the screen, webcam, and audio sources, composites them into one feed, and encodes it. Raw 1080p60 video is around 3 Gbps, which nobody can upload, so the encoder compresses it down to maybe 4-8 Mbps using H.264 (most common), or H.265 (better ratio but heavier), or AV1 (best compression but still too expensive for real-time on most hardware).

OBS can use x264 in software (runs on CPU, good quality, eats cores) or hardware encoding through NVENC on Nvidia, QSV on Intel, or AMF on AMD. Most streamers use NVENC because it offloads the work to the GPU's dedicated encoder block and barely affects game performance, even though historically the quality per bitrate was worse than a well-tuned x264. That gap has mostly closed with newer Nvidia architectures.

You also pick a preset (faster = less CPU = worse compression), a bitrate, resolution, and framerate. The encoder has a hard real-time deadline as it must finish each frame before the next one arrives at 16ms intervals for 60fps. If it can't keep up, frames get dropped and you see that in the stream.

## RTMP: getting the stream out

OBS connects to the platform's ingest server over RTMP, which is a protocol Adobe built for Flash in the early 2000s. Flash died but RTMP stuck around because it does one specific thing well: push a continuous audio/video stream to a server over TCP with minimal framing overhead. OBS does a handshake, authenticates with your stream key, and starts sending FLV-wrapped H.264 and AAC.

> RTMP doesn't natively support newer codecs like HEVC or AV1 (there are vendor extensions but nothing standardized), and being TCP-only means any packet loss causes head-of-line blocking. Despite that, it's still the default for ingest everywhere because SRT and RIST just haven't reached the same adoption level.

Upload bandwidth is a real constraint. Pushing 6 Mbps of video plus audio means you need 8-10 Mbps of stable upstream, and fluctuations cause dropped frames that viewers actually notice. Platforms put ingest servers in dozens of cities so streamers connect to one nearby with low latency.

## ingest and transcoding

Once the ingest server has the RTMP stream, the platform transcodes it into multiple quality levels so viewers on different connections can watch. This is called an ABR ladder (adaptive bitrate). A typical setup is source at 1080p60/6Mbps, then 720p60 at 3 Mbps, 480p30 at 1.5 Mbps, and maybe 360p30 at 600 kbps. Each of those is a separate encoder instance running in parallel, so that's four encodes per stream.

FFmpeg is the standard tool for this, you can feed it a single RTMP ingest and it transcodes into multiple renditions simultaneously, each one produces its own HLS playlist. Smaller platforms literally just run FFmpeg behind Nginx-RTMP and call it a day. At scale though, CPU-based encoding gets too expensive, so giants like Twitch and YouTube use custom FPGA-based encoders and ASICs.

> Twitch doesn't give transcoding to every streamer by default, only partners and affiliates get guaranteed transcode. The reason is straightforward: tens of thousands of concurrent streams × four encoder instances each is an enormous amount of compute, and specialized encoding hardware is a major capital expense on top of the electricity to run it.

## HLS: how viewers get the stream

HLS (HTTP Live Streaming) is how the video actually reaches viewers. The idea is simple, break the continuous video into small segment files (2-6 seconds each), and then write a playlist (`.m3u8`) listing the available segments, and let the player download them one by one over plain HTTP.

```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=6000000,RESOLUTION=1920x1080
1080p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720
720p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1500000,RESOLUTION=854x480
480p/playlist.m3u8
```

The player does adaptive bitrate switching, like if bandwidth drops it falls to a lower rendition automatically, if bandwidth recovers it steps back up, and switches happen at segment boundaries to avoid visual artifacts.

The tradeoff tho is latency. The player has to buffer at least a segment or two before it starts playing, so with 4-second segments and 2 buffered, you're at 8 seconds of delay before you even account for anything else in the pipeline. That's the fundamental reason HLS streams have higher latency than something like a WebRTC call.

HLS was created by Apple, I think. 

> Apple introduced Low-Latency HLS which uses partial segments (sub-second chunks) to bring latency closer to 2-3 seconds, and Twitch uses a proprietary low-latency HLS variant. These help but they add real complexity to both server and player implementations.

## CDN: distribution

One server can't push a stream to 50,000 viewers, the bandwidth alone would be 200 Gbps, which is absurd for a single origin. So the origin generates HLS segments and edge servers around the world and cache them close to viewers. A viewer in Tokyo pulls from a Tokyo edge, not from a datacenter in Virginia.

CDN cost is bandwidth. Providers charge per GB transferred. A stream at 4 Mbps to 10,000 viewers for 3 hours works out to roughly 54 TB of data:

```
4 Mbps × 10,000 × 10,800 seconds ≈ 54 TB
```

Even at bulk pricing, that's expensive. Twitch and YouTube operate their own edge networks partly to keep these costs under control. Smaller platforms pay CloudFront or Fastly or Akamai rates, and the bandwidth bill is usually their single largest expense.

## WebRTC: the low-latency option

WebRTC uses UDP, skips segmentation entirely, and achieves sub-second latency. It was designed for video calls though, not broadcast, and the scaling model reflects that. A 4-person video call is peer-to-peer, everyone sends to everyone. Each one sends to the other three, so 4 people × 3 connections = 12 connections total. That doesn't work for thousands of viewers.

You can put an SFU (Selective Forwarding Unit) in the middle: the streamer sends one stream to the server and the server forwards it to every viewer. But each viewer connection is stateful, it needs DTLS handshake, SRTP encryption state, per-connection bandwidth estimation, RTCP feedback. At 10,000 viewers that's 10,000 concurrent stateful connections with active feedback loops, which is a fundamentally different scaling challenge than HLS where viewers just download files over stateless HTTP and CDN caching handles the rest.

> Some platforms use WebRTC for the first mile (streamer to server, low latency) and HLS for the last mile (server to viewers, scales with CDN). A few like Millicast do full WebRTC end-to-end with large SFU clusters, but the per-viewer infrastructure cost is much higher than HLS.

## FFmpeg: the glue

FFmpeg shows up at almost every stage. A minimal ingest-to-HLS pipeline is one command:

```bash
ffmpeg -i rtmp://localhost/live/stream_key \
  -c:v libx264 -preset veryfast -b:v 3000k \
  -c:a aac -b:a 128k \
  -f hls -hls_time 4 -hls_list_size 5 \
  /var/www/stream/playlist.m3u8
```

That takes an RTMP stream, re-encodes it to H.264 at 3 Mbps, outputs HLS segments of 4 seconds each, and writes playlist files to disk. Serve those with nginx and you have a working live streaming backend. For multiple renditions you run multiple outputs or use the `split` filter. In production, platforms wrap FFmpeg's libraries (libavcodec, libavformat) in their own orchestration with health checks and failover, but the encoding core is the same, you just need engineer and build around/on top of it.

## where the money goes

Encoding and transcoding is compute cost that scales linearly with active streams. More streamers means more encoder instances, and specialized hardware or GPU rentals aren't cheap. CDN bandwidth is the biggest variable cost and scales with viewers × bitrate × time, so a viral stream with 100,000 viewers can generate a terrifying bill in a few hours, which is why platforms cap bitrates or limit quality options. Storage for VODs adds up quietly as every saved stream is hours of multi-bitrate video, and Twitch keeps them for 14-60 days while YouTube keeps them forever. Ingest infrastructure is the global network of servers accepting RTMP connections, each location with its own hardware and bandwidth costs.

The economics are hard for anyone competing with Twitch (Amazon infrastructure), YouTube (Google infrastructure), or Kick (deep investment backing). The infrastructure costs are a barrier to entry that has nothing to do with product quality.

## the latency budget

Where the delay actually lives:

| Stage | Typical latency |
|---|---|
| Encoding (frame buffering) | 30-60 ms |
| RTMP to ingest | 50-200 ms |
| Transcoding | 500-2000 ms |
| HLS segmentation | 2000-6000 ms |
| CDN propagation | 50-200 ms |
| Player buffering | 2000-8000 ms |
| **Total** | **~5-15 seconds** |

Almost all of it is HLS segmentation and player buffering (assuming you actually write good performant code). The actual encoding and network transit are fast. Reducing stream latency basically means either making segments smaller (LL-HLS) or abandoning HLS for WebRTC.

Btw, this whole thing is called "glass-to-glass" latency.

## notes

- RTMP uses port 1935 which corporate firewalls often block, so some streamers use RTMPS (TLS on 443) to get around it.
- SRT handles packet loss better than RTMP on unreliable networks. Haivision made it, OBS supports it, but adoption is still limited.
- AV1 software encoding can't do 1080p60 in real-time on current hardware. Hardware AV1 encoding (Intel Arc, Nvidia 40-series) exists but adoption is early.
- Twitch caps ingest at 8500 kbps partly because higher bitrates mean more expensive transcoding. YouTube allows up to 51 Mbps for 4K.
- Chat infrastructure is completely separate, WebSocket-based or whatever, its own scaling problems with millions of connections and message fan-out, shares almost nothing with video.
