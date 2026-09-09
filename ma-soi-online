const express = require('express');
const http = require('http');
const { Server } = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: "*" } });

let roomsMap = new Map();

// --- XỬ LÝ MÁY CHỦ (BACKEND) ---
io.on('connection', (socket) => {
  socket.emit('rooms_updated', Array.from(roomsMap.values()));

  socket.on('create_room', (newRoom) => {
    roomsMap.set(newRoom.room_code, newRoom);
    socket.join(newRoom.room_code);
    io.emit('rooms_updated', Array.from(roomsMap.values()));
  });

  socket.on('join_room', ({ roomCode, updatedRoom }) => {
    roomsMap.set(roomCode, updatedRoom);
    socket.join(roomCode);
    io.to(roomCode).emit('room_data_changed', updatedRoom);
    io.emit('rooms_updated', Array.from(roomsMap.values()));
  });

  socket.on('update_room', (updatedRoom) => {
    if (updatedRoom && updatedRoom.room_code) {
      roomsMap.set(updatedRoom.room_code, updatedRoom);
      io.to(updatedRoom.room_code).emit('room_data_changed', updatedRoom);
      io.emit('rooms_updated', Array.from(roomsMap.values()));
    }
  });

  socket.on('leave_room', ({ roomCode, updatedRoom }) => {
    socket.leave(roomCode);
    if (updatedRoom) {
      roomsMap.set(roomCode, updatedRoom);
      io.to(roomCode).emit('room_data_changed', updatedRoom);
    }
    io.emit('rooms_updated', Array.from(roomsMap.values()));
  });
});

// --- PHỤC VỤ GIAO DIỆN & LOGIC VOICE CHAT (FRONTEND GỘP CHUNG) ---
app.get('/', (req, res) => {
  res.send(`
<!DOCTYPE html>
<html lang="vi">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Ma Sói Online - Voice Chat</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <script src="/socket.io/socket.io.js"></script>
  <script src="https://unpkg.com/peerjs@1.5.4/dist/peerjs.min.js"></script>
  <style>
    body { background-color: #121212; color: #f1f1f1; font-family: sans-serif; }
    .screen { display: none; }
    .screen.active { display: block; }
    .is-hidden { display: none !important; }
  </style>
</head>
<body class="p-4 max-w-md mx-auto">
  <div id="landing-screen" class="screen active text-center py-10">
    <h1 class="text-3xl font-bold text-red-500 mb-6">🐺 MA SÓI ONLINE</h1>
    <button id="open-create" class="w-full bg-red-600 text-white py-3 rounded-lg font-bold mb-3">Tạo Phòng Mới</button>
    <button id="open-join" class="w-full bg-gray-800 text-white py-3 rounded-lg font-bold">Tham Gia Phòng</button>
  </div>

  <div id="create-screen" class="screen py-6">
    <h2 class="text-xl font-bold mb-4">Tạo Phòng Chơi</h2>
    <form id="create-form" class="space-y-4">
      <input id="room-name" type="text" placeholder="Tên phòng..." required class="w-full p-3 bg-gray-800 rounded text-white">
      <input id="host-name" type="text" placeholder="Tên của bạn..." required class="w-full p-3 bg-gray-800 rounded text-white">
      <input id="max-players" type="number" value="5" min="4" max="15" class="w-full p-3 bg-gray-800 rounded text-white">
      <button type="submit" class="w-full bg-red-600 py-3 rounded font-bold">Xác Nhận Tạo</button>
    </form>
    <button data-back="landing-screen" class="w-full mt-3 bg-gray-700 py-2 rounded">Quay lại</button>
  </div>

  <div id="join-screen" class="screen py-6">
    <h2 class="text-xl font-bold mb-4">Vào Phòng Game</h2>
    <form id="join-form" class="space-y-4">
      <input id="room-code" type="text" placeholder="Mã phòng (VD: WOLF-ABCD)..." required class="w-full p-3 bg-gray-800 rounded text-white uppercase">
      <input id="player-name" type="text" placeholder="Tên của bạn..." required class="w-full p-3 bg-gray-800 rounded text-white">
      <button type="submit" class="w-full bg-red-600 py-3 rounded font-bold">Vào Phòng</button>
    </form>
    <p id="join-error" class="text-red-400 mt-2 text-sm is-hidden"></p>
    <button data-back="landing-screen" class="w-full mt-3 bg-gray-700 py-2 rounded">Quay lại</button>
  </div>

  <div id="room-screen" class="screen py-4">
    <div class="bg-gray-800 p-4 rounded-lg mb-4">
      <h2 id="room-header" class="text-xl font-bold text-red-400">---</h2>
      <p id="room-meta" class="text-sm text-gray-400"></p>
      <p id="sync-status" class="text-xs text-emerald-400 mt-1"></p>
    </div>

    <div class="bg-gray-900 p-3 rounded-lg mb-4">
      <h3 class="font-bold text-sm mb-2">🎙️ Danh Sách Người Chơi (Voice Chat):</h3>
      <div id="player-list" class="space-y-2"></div>
    </div>

    <div class="bg-gray-900 p-3 rounded-lg mb-4">
      <h3 class="font-bold text-sm mb-2">💬 Trò Chuyện Trong Làng:</h3>
      <div id="chat-list" class="h-40 overflow-y-auto space-y-2 mb-2 p-2 bg-black/30 rounded"></div>
      <p id="chat-note" class="text-xs text-gray-400 mb-2"></p>
      <form id="chat-form" class="flex gap-2">
        <input id="chat-input" type="text" placeholder="Nhập tin nhắn..." class="flex-1 p-2 bg-gray-800 rounded text-sm text-white">
        <button id="send-message" type="submit" class="bg-red-600 px-4 rounded font-bold text-sm">Gửi</button>
      </form>
    </div>

    <button id="leave-room" class="w-full bg-gray-700 py-2 rounded text-sm">Rời Phòng</button>
  </div>

  <script>
    const socket = io();
    const $ = (id) => document.getElementById(id);

    let rooms = [];
    let activeRoom = null;
    let myPlayerId = 'user_' + Math.random().toString(36).substring(2, 9);

    // VOICE CHAT (PEERJS)
    let peer = new Peer();
    let myPeerId = null;
    let localStream = null;

    peer.on('open', (id) => { myPeerId = id; });

    navigator.mediaDevices.getUserMedia({ audio: true, video: false })
      .then((stream) => { localStream = stream; })
      .catch((err) => console.log('Micro chưa sẵn sàng:', err));

    peer.on('call', (call) => {
      call.answer(localStream);
      call.on('stream', (remoteStream) => addAudioElement(call.peer, remoteStream));
    });

    function connectVoiceToAll(players) {
      players.forEach((p) => {
        if (p.peerId && p.peerId !== myPeerId && localStream) {
          const call = peer.call(p.peerId, localStream);
          call.on('stream', (remoteStream) => addAudioElement(p.peerId, remoteStream));
        }
      });
    }

    function addAudioElement(peerId, stream) {
      if ($('audio-' + peerId)) return;
      const audio = document.createElement('audio');
      audio.id = 'audio-' + peerId;
      audio.srcObject = stream;
      audio.autoplay = true;
      document.body.appendChild(audio);
    }

    // LOGIC DỮ LIỆU
    function parseJSON(v, f) { try { return JSON.parse(v || ""); } catch { return f; } }
    function getPlayers(room) { return parseJSON(room.players_data, []); }
    function getMessages(room) { return parseJSON(room.messages, []); }
    function getGameState(room) { return parseJSON(room.game_state, {}); }
    function getRoom(code) { return rooms.find((r) => r.room_code === String(code).toUpperCase()); }

    function setScreen(id) {
      document.querySelectorAll('.screen').forEach((s) => s.classList.remove('active'));
      if ($(id)) $(id).classList.add('active');
    }

    function renderPlayers(room) {
      const list = $("player-list");
      if (!list) return;
      const players = getPlayers(room);
      list.innerHTML = '';
      players.forEach((p) => {
        const div = document.createElement('div');
        div.className = 'flex justify-between items-center p-2 bg-gray-800 rounded text-sm';
        div.innerHTML = \`<span>\${p.name} \${p.id === room.host_id ? '👑' : ''} \${p.id === myPlayerId ? '(Bạn)' : ''}</span><span class="text-xs text-emerald-400">🎙️ Đã Bật Voice</span>\`;
        list.appendChild(div);
      });
      connectVoiceToAll(players);
    }

    function renderChat(room) {
      const list = $("chat-list");
      if (!list) return;
      const messages = getMessages(room);
      list.innerHTML = '';
      messages.forEach((m) => {
        const p = document.createElement('p');
        p.className = 'text-xs mb-1';
        p.innerHTML = \`<strong class="text-yellow-500">\${m.name}:</strong> \${m.text}\`;
        list.appendChild(p);
      });
      list.scrollTop = list.scrollHeight;
    }

    function renderRoom() {
      if (!activeRoom) return;
      const room = activeRoom;
      const players = getPlayers(room);
      const max = Number(getGameState(room).max_players || 5);

      if ($("room-header")) $("room-header").textContent = room.room_name + " (" + room.room_code + ")";
      if ($("room-meta")) $("room-meta").textContent = "👥 " + players.length + "/" + max + " Người chơi";

      renderPlayers(room);
      renderChat(room);
    }

    // SỰ KIỆN
    $("open-create").addEventListener("click", () => setScreen("create-screen"));
    $("open-join").addEventListener("click", () => setScreen("join-screen"));
    document.querySelectorAll("[data-back]").forEach((b) => b.addEventListener("click", () => setScreen(b.dataset.back)));

    $("create-form").addEventListener("submit", (e) => {
      e.preventDefault();
      const code = "WOLF-" + Math.random().toString(36).substring(2, 6).toUpperCase();
      const newRoom = {
        room_code: code,
        room_name: $("room-name").value.trim(),
        host_id: myPlayerId,
        game_state: JSON.stringify({ max_players: $("max-players").value }),
        players_data: JSON.stringify([{ id: myPlayerId, name: $("host-name").value.trim(), peerId: myPeerId }]),
        messages: "[]"
      };
      socket.emit('create_room', newRoom);
      activeRoom = newRoom;
      setScreen("room-screen");
      renderRoom();
    });

    $("join-form").addEventListener("submit", (e) => {
      e.preventDefault();
      const code = $("room-code").value.trim().toUpperCase();
      const room = getRoom(code);
      if (!room) return alert("Không tìm thấy phòng!");

      const players = getPlayers(room);
      const updatedRoom = {
        ...room,
        players_data: JSON.stringify([...players, { id: myPlayerId, name: $("player-name").value.trim(), peerId: myPeerId }])
      };
      activeRoom = updatedRoom;
      socket.emit('join_room', { roomCode: code, updatedRoom });
      setScreen("room-screen");
      renderRoom();
    });

    $("chat-form").addEventListener("submit", (e) => {
      e.preventDefault();
      if (!activeRoom) return;
      const input = $("chat-input");
      const me = getPlayers(activeRoom).find((p) => p.id === myPlayerId);
      if (!input.value.trim() || !me) return;

      const msgs = [...getMessages(activeRoom), { name: me.name, text: input.value.trim() }];
      activeRoom = { ...activeRoom, messages: JSON.stringify(msgs) };
      socket.emit('update_room', activeRoom);
      input.value = "";
    });

    $("leave-room").addEventListener("click", () => {
      if (!activeRoom) return;
      const players = getPlayers(activeRoom).filter((p) => p.id !== myPlayerId);
      socket.emit('leave_room', { roomCode: activeRoom.room_code, updatedRoom: { ...activeRoom, players_data: JSON.stringify(players) } });
      activeRoom = null;
      setScreen("landing-screen");
    });

    socket.on('rooms_updated', (data) => { rooms = data; });
    socket.on('room_data_changed', (updated) => {
      if (activeRoom && activeRoom.room_code === updated.room_code) {
        activeRoom = updated;
        renderRoom();
      }
    });
  </script>
</body>
</html>
  `);
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
  console.log(`Server Ma Sói đang chạy tại port ${PORT}`);
});

{
  "name": "ma-soi",
  "scripts": { "start": "node server.js" },
  "dependencies": { "express": "^4.18.2", "socket.io": "^4.7.2" }
}
