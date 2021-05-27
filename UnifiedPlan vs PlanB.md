# UnifiedPlan vs PlanB

kUnifiedPlan 和 kPlanB 是SDP协商中，多路媒体流的协商方式， 用于支持在一个PeerConnection上支持多路 Media Stream。
- 在 kPlanB 中， 音频只有一个m=, 视频只有一个m= 。 当有多路媒体流时， 根据ssrc 区分；
- 在 kUnifiedPlan 中， 每个媒体源都有一个m= ；
- 当只有一个路视频流，一路音频流时，两者格式兼容。

示例：
UnifiedPlan
```
type: offer, sdp: v=0
o=mediasoup 76760093 1 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-lite
a=fingerprint:sha-256 75:E9:8A:3C:96:49:89:97:18:A3:0D:73:E9:9A:59:D1:8A:B4:15:A9:8F:C4:DE:91:8F:5D:E3:73:0D:E2:00:4E
a=msid-semantic: WMS *
a=group:BUNDLE audio-1 video-1 j6gtnlva p0o0fho7 i860vo1m xwa9siz1
m=audio 7 RTP/SAVPF 100
c=IN IP4 127.0.0.1
a=rtpmap:100 opus/48000/2
a=fmtp:100 minptime=10;useinbandfec=1
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=extmap:5 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=setup:actpass
a=mid:audio-1
a=recvonly
a=ice-ufrag:3lw1e76eix7edjcm
a=ice-pwd:cidqjcjfv28re4t7djaf410gkzpoblq4
a=candidate:udpcandidate 1 udp 1078862079 10.1.133.155 19211 typ host
a=end-of-candidates
a=ice-options:renomination
a=rtcp-mux
a=rtcp-rsize
m=video 7 RTP/SAVPF 123 101
c=IN IP4 127.0.0.1
a=rtpmap:123 VP8/90000
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=123
a=rtcp-fb:123 goog-remb
a=rtcp-fb:123 ccm fir
a=rtcp-fb:123 nack
a=rtcp-fb:123 nack pli
a=extmap:2 urn:ietf:params:rtp-hdrext:toffset
a=extmap:3 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:4 urn:3gpp:video-orientation
a=extmap:5 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=setup:actpass
a=mid:video-1
a=recvonly
a=ice-ufrag:3lw1e76eix7edjcm
a=ice-pwd:cidqjcjfv28re4t7djaf410gkzpoblq4
a=candidate:udpcandidate 1 udp 1078862079 10.1.133.155 19211 typ host
a=end-of-candidates
a=ice-options:renomination
a=rtcp-mux
a=rtcp-rsize
m=video 7 RTP/SAVPF 123 101
c=IN IP4 127.0.0.1
a=rtpmap:123 VP8/90000
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=123
a=rtcp-fb:123 goog-remb
a=rtcp-fb:123 ccm fir
a=rtcp-fb:123 nack
a=rtcp-fb:123 nack pli
a=extmap:2 urn:ietf:params:rtp-hdrext:toffset
a=extmap:3 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:4 urn:3gpp:video-orientation
a=extmap:5 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=setup:actpass
a=mid:j6gtnlva
a=msid:kREh1yv7S0I5syCOntCAIUBeMoesO6v5RlkL 28c40df0-83b8-46c7-bd06-34e7e6fc8fa0
a=sendonly
a=ice-ufrag:3lw1e76eix7edjcm
a=ice-pwd:cidqjcjfv28re4t7djaf410gkzpoblq4
a=candidate:udpcandidate 1 udp 1078862079 10.1.133.155 19211 typ host
a=end-of-candidates
a=ice-options:renomination
a=ssrc:4131643841 cname:pYEUk1YEO5F9x9T/
a=ssrc:1251779610 cname:pYEUk1YEO5F9x9T/
a=ssrc-group:FID 4131643841 1251779610
a=rtcp-mux
a=rtcp-rsize
m=audio 7 RTP/SAVPF 100
c=IN IP4 127.0.0.1
a=rtpmap:100 opus/48000/2
a=fmtp:100 minptime=10;useinbandfec=1
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=extmap:5 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=setup:actpass
a=mid:p0o0fho7
a=msid:kREh1yv7S0I5syCOntCAIUBeMoesO6v5RlkL 1271c25f-e4e8-4763-8770-2f043f426445
a=sendonly
a=ice-ufrag:3lw1e76eix7edjcm
a=ice-pwd:cidqjcjfv28re4t7djaf410gkzpoblq4
a=candidate:udpcandidate 1 udp 1078862079 10.1.133.155 19211 typ host
a=end-of-candidates
a=ice-options:renomination
a=ssrc:4125350497 cname:pYEUk1YEO5F9x9T/
a=rtcp-mux
a=rtcp-rsize
m=video 7 RTP/SAVPF 123 101
c=IN IP4 127.0.0.1
a=rtpmap:123 VP8/90000
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=123
a=rtcp-fb:123 goog-remb
a=rtcp-fb:123 ccm fir
a=rtcp-fb:123 nack
a=rtcp-fb:123 nack pli
a=fmtp:123 x-google-min-bitrate=140;x-google-max-bitrate=180;x-google-start-bitrate=160;x-google-min-bitrate=140;x-google-max-bitrate=180;x-google-start-bitrate=160;x-google-min-bitrate=140;x-google-max-bitrate=180;x-google-start-bitrate=160
a=extmap:2 urn:ietf:params:rtp-hdrext:toffset
a=extmap:3 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:4 urn:3gpp:video-orientation
a=extmap:5 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=setup:actpass
a=mid:i860vo1m
a=msid:nbwkpaLOrbf3snCxWpLgPCi01iwx6ghw9xu2 1098359b-2cc9-4b96-b7f4-6aa2b7c40031
a=sendonly
a=ice-ufrag:3lw1e76eix7edjcm
a=ice-pwd:cidqjcjfv28re4t7djaf410gkzpoblq4
a=candidate:udpcandidate 1 udp 1078862079 10.1.133.155 19211 typ host
a=end-of-candidates
a=ice-options:renomination
a=ssrc:2368086230 cname:QVg07Awk9yf5LedF
a=ssrc:2368086230 cname:QVg07Awk9yf5LedF
a=ssrc-group:FID 2368086230 2368086230
a=rtcp-mux
a=rtcp-rsize
m=audio 7 RTP/SAVPF 100
c=IN IP4 127.0.0.1
a=rtpmap:100 opus/48000/2
a=fmtp:100 minptime=10;useinbandfec=1
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=extmap:5 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=setup:actpass
a=mid:xwa9siz1
a=msid:nbwkpaLOrbf3snCxWpLgPCi01iwx6ghw9xu2 357db268-a530-4b09-98e9-8ad599921249
a=sendonly
a=ice-ufrag:3lw1e76eix7edjcm
a=ice-pwd:cidqjcjfv28re4t7djaf410gkzpoblq4
a=candidate:udpcandidate 1 udp 1078862079 10.1.133.155 19211 typ host
a=end-of-candidates
a=ice-options:renomination
a=ssrc:1666321053 cname:QVg07Awk9yf5LedF
a=rtcp-mux
a=rtcp-rsize
```
PlanB
```
type: offer, sdp: v=0
o=mediasoup 24017464 1 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-lite
a=fingerprint:sha-256 75:E9:8A:3C:96:49:89:97:18:A3:0D:73:E9:9A:59:D1:8A:B4:15:A9:8F:C4:DE:91:8F:5D:E3:73:0D:E2:00:4E
a=msid-semantic: WMS *
a=group:BUNDLE audio-tracks video-tracks
m=audio 7 RTP/SAVPF 100
c=IN IP4 127.0.0.1
a=rtpmap:100 opus/48000/2
a=fmtp:100 minptime=10;useinbandfec=1
a=extmap:1 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=setup:actpass
a=mid:audio-tracks
a=sendrecv
a=ice-ufrag:vdb6b4984o7dkf16
a=ice-pwd:4agfo1y9pl4ylg2zxrx4lufimetfuxzc
a=candidate:udpcandidate 1 udp 1078862079 10.1.133.155 51891 typ host
a=end-of-candidates
a=ice-options:renomination
a=ssrc:1689928828 cname:tw9LY2QY98p9spEm
a=ssrc:1689928828 msid:NSyf3jpXBCyhNnds6Shejy3vfxXmQRpVkcpQ 9f9624db-1c35-49a0-9898-2d578606961d
a=ssrc:2064048689 cname:0SQ2ImcTH1runIhp
a=ssrc:2064048689 msid:FWqQatBcsifPcujegGiIXt2BgBNSS89G20VC 06c7586a-c83e-4a5a-a06c-d126c49a810d
a=rtcp-mux
a=rtcp-rsize
m=video 7 RTP/SAVPF 123 101
c=IN IP4 127.0.0.1
a=rtpmap:123 VP8/90000
a=rtpmap:101 rtx/90000
a=fmtp:101 apt=123
a=rtcp-fb:123 goog-remb
a=rtcp-fb:123 ccm fir
a=rtcp-fb:123 nack
a=rtcp-fb:123 nack pli
a=fmtp:123 x-google-min-bitrate=140;x-google-max-bitrate=180;x-google-start-bitrate=160
a=extmap:2 urn:ietf:params:rtp-hdrext:toffset
a=extmap:3 http://www.webrtc.org/experiments/rtp-hdrext/abs-send-time
a=extmap:4 urn:3gpp:video-orientation
a=setup:actpass
a=mid:video-tracks
a=sendrecv
a=ice-ufrag:vdb6b4984o7dkf16
a=ice-pwd:4agfo1y9pl4ylg2zxrx4lufimetfuxzc
a=candidate:udpcandidate 1 udp 1078862079 10.1.133.155 51891 typ host
a=end-of-candidates
a=ice-options:renomination
a=ssrc:1028448819 cname:tw9LY2QY98p9spEm
a=ssrc:1028448819 msid:NSyf3jpXBCyhNnds6Shejy3vfxXmQRpVkcpQ e59b243c-f944-471b-b410-45a80db0797e
a=ssrc:788703880 cname:tw9LY2QY98p9spEm
a=ssrc:788703880 msid:NSyf3jpXBCyhNnds6Shejy3vfxXmQRpVkcpQ e59b243c-f944-471b-b410-45a80db0797e
a=ssrc:2889626611 cname:0SQ2ImcTH1runIhp
a=ssrc:2889626611 msid:FWqQatBcsifPcujegGiIXt2BgBNSS89G20VC c73ef760-4a91-4077-9bf1-fbdda5c10c71
a=ssrc:4061195486 cname:0SQ2ImcTH1runIhp
a=ssrc:4061195486 msid:FWqQatBcsifPcujegGiIXt2BgBNSS89G20VC c73ef760-4a91-4077-9bf1-fbdda5c10c71
a=ssrc-group:FID 1028448819 788703880
a=ssrc-group:FID 2889626611 4061195486
a=rtcp-mux
a=rtcp-rsize
```