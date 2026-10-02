# Part 97: WebRTC และ Real-time Media (สื่อแบบ Real-time)

## เป้าหมายการเรียนรู้

หลังจากจบ Part นี้ คุณจะสามารถ:
- Signaling server ด้วย Phoenix Channels
- WebRTC peer connection
- Video/audio call ในแอพ
- Screen sharing

---

## 1. Signaling Server ด้วย Phoenix Channels

```elixir
# Phoenix Channel ทำหน้าที่เป็น signaling server
# WebRTC ต้องการ signaling เพื่อ exchange SDP และ ICE candidates

defmodule MyAppWeb.RoomChannel do
  use Phoenix.Channel

  def join("room:" <> room_id, _params, socket) do
    socket = assign(socket, :room_id, room_id)

    # ประกาศว่าผู้ใช้เข้าร่วม
    broadcast_from!(socket, "user_joined", %{
      user_id: socket.assigns.user_id
    })

    {:ok, socket}
  end

  # WebRTC: ส่ง SDP offer ไปยังผู้รับ
  def handle_in("webrtc_offer", %{"offer" => offer, "to" => to_user_id}, socket) do
    broadcast_to_user(socket, to_user_id, "webrtc_offer", %{
      offer: offer,
      from: socket.assigns.user_id
    })
    {:noreply, socket}
  end

  # WebRTC: ส่ง SDP answer กลับ
  def handle_in("webrtc_answer", %{"answer" => answer, "to" => to_user_id}, socket) do
    broadcast_to_user(socket, to_user_id, "webrtc_answer", %{
      answer: answer,
      from: socket.assigns.user_id
    })
    {:noreply, socket}
  end

  # WebRTC: exchange ICE candidates
  def handle_in("ice_candidate", %{"candidate" => candidate, "to" => to_user_id}, socket) do
    broadcast_to_user(socket, to_user_id, "ice_candidate", %{
      candidate: candidate,
      from: socket.assigns.user_id
    })
    {:noreply, socket}
  end

  # User leaving
  def terminate(_reason, socket) do
    broadcast_from!(socket, "user_left", %{
      user_id: socket.assigns.user_id
    })
  end

  defp broadcast_to_user(socket, user_id, event, payload) do
    topic = "user:#{user_id}"
    MyAppWeb.Endpoint.broadcast(topic, event, payload)
  end
end
```

---

## 2. Client-side WebRTC (JavaScript)

```javascript
class VideoCall {
  constructor(channelSocket, localVideoEl, remoteVideoEl) {
    this.channel = channelSocket
    this.localVideo = localVideoEl
    this.remoteVideo = remoteVideoEl
    this.peers = {}
    this.localStream = null

    this.setupChannelHandlers()
  }

  async startCamera() {
    this.localStream = await navigator.mediaDevices.getUserMedia({
      video: true,
      audio: true
    })
    this.localVideo.srcObject = this.localStream
  }

  async callUser(userId) {
    const peer = await this.createPeerConnection(userId)

    // Add local tracks
    this.localStream.getTracks().forEach(track => {
      peer.addTrack(track, this.localStream)
    })

    // Create and send offer
    const offer = await peer.createOffer()
    await peer.setLocalDescription(offer)

    this.channel.push("webrtc_offer", { offer, to: userId })
  }

  async createPeerConnection(userId) {
    const config = {
      iceServers: [
        { urls: "stun:stun.l.google.com:19302" },
        {
          urls: "turn:turn.myapp.com",
          username: "user",
          credential: "secret"
        }
      ]
    }

    const peer = new RTCPeerConnection(config)
    this.peers[userId] = peer

    // Send ICE candidates to other peer
    peer.onicecandidate = ({ candidate }) => {
      if (candidate) {
        this.channel.push("ice_candidate", { candidate, to: userId })
      }
    }

    // Display remote video
    peer.ontrack = (event) => {
      if (event.streams[0]) {
        this.remoteVideo.srcObject = event.streams[0]
      }
    }

    return peer
  }

  setupChannelHandlers() {
    // Receive call offer
    this.channel.on("webrtc_offer", async ({ offer, from }) => {
      const peer = await this.createPeerConnection(from)

      this.localStream.getTracks().forEach(track => {
        peer.addTrack(track, this.localStream)
      })

      await peer.setRemoteDescription(offer)
      const answer = await peer.createAnswer()
      await peer.setLocalDescription(answer)

      this.channel.push("webrtc_answer", { answer, to: from })
    })

    // Receive answer
    this.channel.on("webrtc_answer", async ({ answer, from }) => {
      const peer = this.peers[from]
      if (peer) {
        await peer.setRemoteDescription(answer)
      }
    })

    // Receive ICE candidate
    this.channel.on("ice_candidate", async ({ candidate, from }) => {
      const peer = this.peers[from]
      if (peer) {
        await peer.addIceCandidate(candidate)
      }
    })

    this.channel.on("user_left", ({ user_id }) => {
      this.closePeerConnection(user_id)
    })
  }

  closePeerConnection(userId) {
    if (this.peers[userId]) {
      this.peers[userId].close()
      delete this.peers[userId]
    }
  }

  async shareScreen() {
    const screenStream = await navigator.mediaDevices.getDisplayMedia({
      video: { cursor: "always" },
      audio: false
    })

    const videoTrack = screenStream.getVideoTracks()[0]

    // Replace video track in all peer connections
    Object.values(this.peers).forEach(peer => {
      const sender = peer.getSenders().find(s => s.track?.kind === "video")
      if (sender) sender.replaceTrack(videoTrack)
    })

    videoTrack.onended = () => this.stopScreenShare()
  }

  async stopScreenShare() {
    const cameraTrack = this.localStream.getVideoTracks()[0]
    Object.values(this.peers).forEach(peer => {
      const sender = peer.getSenders().find(s => s.track?.kind === "video")
      if (sender) sender.replaceTrack(cameraTrack)
    })
  }
}
```

---

## 3. LiveView Integration

```elixir
defmodule MyAppWeb.VideoCallLive do
  use MyAppWeb, :live_view

  def mount(%{"room_id" => room_id}, session, socket) do
    current_user = get_user_from_session(session)

    # Generate secure token for channel auth
    token = Phoenix.Token.sign(
      MyAppWeb.Endpoint,
      "user socket",
      current_user.id
    )

    {:ok, assign(socket,
      room_id: room_id,
      user_token: token,
      participants: []
    )}
  end

  def handle_event("join_call", _params, socket) do
    # Subscribe to presence updates
    MyAppWeb.Presence.track(
      self(),
      "room:#{socket.assigns.room_id}",
      socket.assigns.current_user.id,
      %{name: socket.assigns.current_user.name}
    )

    {:noreply, socket}
  end

  def render(assigns) do
    ~H"""
    <div id="video-call-container"
         phx-hook="VideoCall"
         data-room-id={@room_id}
         data-user-token={@user_token}>

      <video id="local-video" autoplay muted class="w-48 h-36 rounded"></video>
      <div id="remote-videos" class="grid grid-cols-2 gap-2"></div>

      <div class="controls flex gap-4 mt-4">
        <button phx-click="toggle_mic">Mic</button>
        <button phx-click="toggle_video">Camera</button>
        <button id="share-screen-btn">Share Screen</button>
        <button phx-click="leave_call" class="bg-red-500">Leave</button>
      </div>
    </div>
    """
  end
end
```

---

## สรุป

```
WebRTC Architecture:
├── Signaling: Phoenix Channel (SDP, ICE exchange)
├── STUN: discover public IP (Google STUN)
├── TURN: relay if direct connection fails
└── Peer Connection: direct P2P after signaling

WebRTC Flow:
1. User A creates offer (SDP)
2. User A sends offer via Channel
3. User B sets remote description, creates answer
4. User B sends answer via Channel
5. Both exchange ICE candidates
6. Direct P2P connection established

Features:
├── Video/audio call: getUserMedia
├── Screen share: getDisplayMedia
├── Track replacement: sender.replaceTrack()
└── Multi-party: mesh topology (small groups)

Production needs:
├── TURN server (coturn): for NAT traversal
├── SFU (mediasoup, Janus): for many participants
└── Recording: requires media server
```

---

*ก่อนหน้า: [Part 96](part_96.md) | ต่อไป: [Part 98 - PWA และ Offline Support](part_98.md)*
