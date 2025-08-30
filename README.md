# whatsapp
# =============================
# package.json
# =============================
{
  "name": "whatsapp-like-chat",
  "version": "1.0.0",
  "description": "Minimal WhatsApp-style realtime chat using Node.js, Express, and Socket.IO",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "node server.js"
  },
  "dependencies": {
    "express": "^4.19.2",
    "socket.io": "^4.7.5",
    "uuid": "^9.0.1"
  }
}

# =============================
# server.js
# =============================
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const { v4: uuidv4 } = require('uuid');

const app = express();
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: '*' } });

const PORT = process.env.PORT || 3000;

// Serve static files
app.use(express.static('public'));

// In-memory store (demo only)
// rooms: { [roomId]: { name, users: Map<socketId, {username}>, messages: [{id, roomId, from, text, ts, status}] } }
const rooms = new Map();

function ensureRoom(roomId) {
  if (!rooms.has(roomId)) {
    rooms.set(roomId, {
      id: roomId,
      name: roomId,
      users: new Map(),
      messages: [],
    });
  }
  return rooms.get(roomId);
}

io.on('connection', (socket) => {
  let current = { roomId: null, username: null };

  socket.on('join', ({ roomId, username }) => {
    current = { roomId, username };
    const room = ensureRoom(roomId);

    socket.join(roomId);
    room.users.set(socket.id, { username });

    // Send backlog to the new user
    socket.emit('backlog', room.messages.slice(-200));

    // Notify others
    socket.to(roomId).emit('presence', {
      type: 'join',
      user: { id: socket.id, username },
      users: [...room.users.values()].map((u, i) => ({ id: [...room.users.keys()][i], username: u.username }))
    });
  });

  socket.on('typing', ({ roomId, typing }) => {
    socket.to(roomId).emit('typing', { userId: socket.id, typing });
  });

  socket.on('message', ({ roomId, text }) => {
    if (!text || !text.trim()) return;
    const id = uuidv4();
    const msg = {
      id,
      roomId,
      from: { id: socket.id, username: current.username },
      text: text.trim(),
      ts: Date.now(),
      status: 'sent' // 'sent' | 'delivered' | 'seen'
    };

    const room = ensureRoom(roomId);
    room.messages.push(msg);

    // Acknowledge to sender as 'sent'
    socket.emit('ack', { id, status: 'sent' });

    // Broadcast to others; receivers mark as delivered on receipt
    socket.to(roomId).emit('message', msg);
  });

  // When a client receives a message, it can report delivery
  socket.on('delivered', ({ roomId, id }) => {
    const room = rooms.get(roomId);
    if (!room) return;
    const msg = room.messages.find(m => m.id === id);
    if (msg && msg.status !== 'seen') {
      msg.status = 'delivered';
      io.to(roomId).emit('ack', { id, status: 'delivered' });
    }
  });

  // When a client views the chat, it can report seen
  socket.on('seen', ({ roomId, ids }) => {
    const room = rooms.get(roomId);
    if (!room) return;
    ids.forEach(id => {
      const msg = room.messages.find(m => m.id === id);
      if (msg) msg.status = 'seen';
    });
    io.to(roomId).emit('seen', { ids });
  });

  socket.on('disconnect', () => {
    const { roomId, username } = current;
    if (!roomId) return;
    const room = rooms.get(roomId);
    if (!room) return;
    room.users.delete(socket.id);
    socket.to(roomId).emit('presence', {
      type: 'leave',
      user: { id: socket.id, username },
      users: [...room.users.values()].map((u, i) => ({ id: [...room.users.keys()][i], username: u.username }))
    });
  });
});

server.listen(PORT, () => {
  console.log(`\n✅ WhatsApp-like chat server running on http://localhost:${PORT}\n`);
});

# =============================
# public/index.html
# =============================
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>WhatsApp-like Chat</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;600&display=swap" rel="stylesheet">
  <style>
    :root { --bg:#0b141a; --panel:#111b21; --me:#005c4b; --them:#202c33; --text:#e9edef; --muted:#8696a0; --accent:#25D366; }
    *{ box-sizing:border-box; }
    body { margin:0; font-family:Inter, system-ui, -apple-system, Segoe UI, Roboto, sans-serif; background:var(--bg); color:var(--text); height:100vh; display:flex; }
    .sidebar{ width:320px; background:var(--panel); border-right:1px solid #1f2c33; display:flex; flex-direction:column; }
    .brand{ padding:16px; font-weight:600; border-bottom:1px solid #1f2c33; display:flex; gap:8px; align-items:center; }
    .brand .dot{ width:10px; height:10px; background:var(--accent); border-radius:50%; box-shadow:0 0 16px var(--accent); }
    .controls{ padding:12px; display:grid; gap:8px; }
    input, button{ padding:10px 12px; border-radius:12px; border:1px solid #2a3942; background:#0b141a; color:var(--text); outline:none; }
    button{ background:var(--accent); color:#0b141a; font-weight:600; border:none; cursor:pointer; }
    .users{ padding:8px 12px; font-size:12px; color:var(--muted); }

    .chat{ flex:1; display:flex; flex-direction:column; }
    .header{ padding:12px 16px; background:var(--panel); border-bottom:1px solid #1f2c33; display:flex; align-items:center; justify-content:space-between; }
    .room{ font-weight:600; }
    .typing{ color:var(--muted); font-size:12px; height:18px; }
    .messages{ flex:1; padding:16px; overflow:auto; display:flex; flex-direction:column; gap:8px; background:url('https://i.imgur.com/yK8w2fD.jpeg') center/cover fixed; }
    .bubble{ max-width:70%; padding:10px 12px; border-radius:16px; position:relative; box-shadow:0 4px 16px rgba(0,0,0,0.2); }
    .me{ align-self:flex-end; background:var(--me); border-bottom-right-radius:4px; }
    .them{ align-self:flex-start; background:var(--them); border-bottom-left-radius:4px; }
    .meta{ display:flex; gap:8px; align-items:center; justify-content:flex-end; color:#cfd9de; opacity:.8; font-size:11px; margin-top:4px; }
    .ticks{ font-size:14px; }

    .composer{ display:flex; gap:8px; padding:12px; background:var(--panel); border-top:1px solid #1f2c33; }
    .composer input{ flex:1; }
  </style>
</head>
<body>
  <div class="sidebar">
    <div class="brand"><div class="dot"></div> WhatsApp-like</div>
    <div class="controls">
      <input id="username" placeholder="Your name" />
      <input id="room" placeholder="Room (e.g. general)" />
      <button id="join">Join Room</button>
    </div>
    <div class="users" id="users">No users yet</div>
  </div>

  <div class="chat">
    <div class="header">
      <div>
        <div class="room" id="roomName">(no room)</div>
        <div class="typing" id="typing"></div>
      </div>
      <div id="me">You: -</div>
    </div>
    <div class="messages" id="messages"></div>
    <div class="composer">
      <input id="input" placeholder="Type a message" disabled />
      <button id="send" disabled>Send</button>
    </div>
  </div>

  <script src="/socket.io/socket.io.js"></script>
  <script>
    const $ = (id) => document.getElementById(id);
    const usernameInput = $('username');
    const roomInput = $('room');
    const joinBtn = $('join');
    const usersEl = $('users');
    const roomNameEl = $('roomName');
    const typingEl = $('typing');
    const meEl = $('me');
    const messagesEl = $('messages');
    const inputEl = $('input');
    const sendBtn = $('send');

    const socket = io();

    // State
    let me = { id: null, username: null };
    let roomId = null;
    let typingTimer = null;
    let visible = true;

    document.addEventListener('visibilitychange', () => {
      visible = !document.hidden;
      if (visible) markAllSeen();
    });

    // Restore from URL hash: #room=username
    (function initFromHash(){
      if (location.hash.includes('=')){
        const [r, u] = location.hash.slice(1).split('=');
        if (r) roomInput.value = decodeURIComponent(r);
        if (u) usernameInput.value = decodeURIComponent(u);
      }
    })();

    joinBtn.onclick = () => joinRoom();
    sendBtn.onclick = () => sendMessage();
    inputEl.oninput = () => socket.emit('typing', { roomId, typing: inputEl.value.length > 0 });
    inputEl.onkeyup = (e) => { if (e.key === 'Enter') sendMessage(); };

    function joinRoom(){
      const username = usernameInput.value.trim() || `User-${Math.floor(Math.random()*1000)}`;
      const room = roomInput.value.trim() || 'general';
      me.username = username;
      meEl.textContent = `You: ${username}`;
      roomId = room;
      roomNameEl.textContent = `Room: ${room}`;
      inputEl.disabled = false; sendBtn.disabled = false;
      location.hash = `${encodeURIComponent(room)}=${encodeURIComponent(username)}`;
      socket.emit('join', { roomId: room, username });
    }

    socket.on('connect', () => { me.id = socket.id; });

    socket.on('backlog', (messages) => {
      messagesEl.innerHTML = '';
      messages.forEach(renderMessage);
      scrollToEnd();
      markAllSeen();
    });

    socket.on('message', (msg) => {
      renderMessage(msg);
      scrollToEnd();
      // Report delivered for messages not from me
      if (msg.from.id !== me.id) socket.emit('delivered', { roomId, id: msg.id });
      if (visible) markAllSeen();
    });

    socket.on('ack', ({ id, status }) => {
      const bubble = document.querySelector(`[data-id="${id}"] .ticks`);
      if (bubble) bubble.textContent = statusToTicks(status);
    });

    socket.on('seen', ({ ids }) => {
      ids.forEach(id => {
        const bubble = document.querySelector(`[data-id="${id}"] .ticks`);
        if (bubble) bubble.textContent = statusToTicks('seen');
      });
    });

    socket.on('typing', ({ userId, typing }) => {
      if (typing) {
        typingEl.textContent = 'typing…';
        clearTimeout(typingTimer);
        typingTimer = setTimeout(() => typingEl.textContent = '', 1200);
      } else {
        typingEl.textContent = '';
      }
    });

    socket.on('presence', ({ type, users }) => {
      usersEl.textContent = users.length ? users.map(u => u.username).join(', ') : 'No users yet';
    });

    function sendMessage(){
      const text = inputEl.value;
      if (!text.trim()) return;
      socket.emit('message', { roomId, text });
      // Optimistic render
      const localId = crypto.randomUUID ? crypto.randomUUID() : Math.random().toString(36).slice(2);
      const msg = { id: localId, from: { id: me.id, username: me.username }, text, ts: Date.now(), status: 'sent' };
      renderMessage(msg);
      inputEl.value = '';
      scrollToEnd();
    }

    function renderMessage(msg){
      const mine = msg.from.id === me.id;
      const wrap = document.createElement('div');
      wrap.className = `bubble ${mine ? 'me' : 'them'}`;
      wrap.dataset.id = msg.id;
      wrap.innerHTML = `
        <div>${escapeHtml(msg.text)}</div>
        <div class="meta">
          <span>${formatTime(msg.ts)}</span>
          ${mine ? `<span class="ticks">${statusToTicks(msg.status)}</span>` : ''}
        </div>
      `;
      messagesEl.appendChild(wrap);
    }

    function markAllSeen(){
      // Collect all non-mine message IDs
      const ids = [...document.querySelectorAll('.bubble.them')].map(el => el.dataset.id);
      if (ids.length) socket.emit('seen', { roomId, ids });
    }

    function formatTime(ts){
      const d = new Date(ts);
      return d.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
    }

    function statusToTicks(status){
      // ✓ sent, ✓✓ delivered, ✓✓ (bold) seen
      if (status === 'seen') return '✓✓';
      if (status === 'delivered') return '✓✓';
      return '✓';
    }

    function scrollToEnd(){ messagesEl.scrollTop = messagesEl.scrollHeight; }

    function escapeHtml(s){
      return s.replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;','\'':'&#39;'}[c]));
    }
  </script>
</body>
</html>
