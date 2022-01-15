
- video
  - srs livego
  - rtmp rtsp hls rtp/rtcp/rtsp http-flv
  - quic
  - webrtc rtc
  - pcm acc
  - ijkplayer flv.js
  - h.264 h.265 hevc
- interactive
  - tcp
  - protobuf
  - quic
  - websocket
  - ipV6 p4 
  - netty
  - instant messaging system
- storage
  - blob store
  - playback video/comments

We should design a storage fo state messages. This storage should save all of the states of each live room, and client could pull all of the states of just the updated states.

```
GET /state/{roomId}?localVersion=123
```

When some latestVersion version is newer the the localVersion from the client, server should return all of the latest state of this live room.
```
{
  "latestVersion": 125,
  "states": {
    "onlineCount": 65
    "topUsers": 68
  }
}
```

But when the latestVersion is equal to the localVersion, server should return the `HTTP 304 Not Modified` or just return emtpy states to prevent the client fetch the same states again.

```
{
  "latestVersion": 123
}
```

When the client pulled the lastest states, it should find out all the "outdated" states. The "outdated" states means that the versions from server are different to the states from local.

For example, people use
