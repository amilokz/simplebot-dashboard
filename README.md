# 🌹 Laila WhatsApp Bot — Full Architecture & Implementation Guide

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| WhatsApp Library | **Baileys** (`@whiskeysockets/baileys`) | Free, open-source, supports QR + pairing code |
| AI Replies | **Groq API** (free tier) or **Google Gemini** (free) | Fast, human-like, free |
| Backend | **Node.js + Express** | Simple, fast, Railway/Render compatible |
| Session Storage | **Local filesystem** (or Redis for scale) | Persistent multi-session support |
| Frontend Dashboard | **React + Vite** (Vercel) | Beautiful control panel |
| Real-time | **Socket.IO** | Live QR code display, session status |

---

## Project Structure

```
laila-bot/
├── backend/
│   ├── src/
│   │   ├── index.js              # Express server entry point
│   │   ├── whatsapp/
│   │   │   ├── sessionManager.js # Multi-session handler
│   │   │   ├── messageHandler.js # Incoming message logic
│   │   │   └── reactionHandler.js# Auto reactions
│   │   ├── ai/
│   │   │   └── aiReply.js        # AI reply engine (Groq/Gemini)
│   │   └── routes/
│   │       ├── session.routes.js # Connect/disconnect endpoints
│   │       └── message.routes.js # Manual override, history
│   ├── sessions/                 # Saved WhatsApp auth per user
│   ├── package.json
│   └── .env
└── frontend/
    ├── src/
    │   ├── App.jsx
    │   ├── pages/
    │   │   ├── Dashboard.jsx
    │   │   ├── Connect.jsx       # QR + Pairing code UI
    │   │   └── Conversations.jsx
    │   └── components/
    │       ├── QRDisplay.jsx
    │       └── SessionCard.jsx
    └── package.json
```

---

## Step 1: Backend Setup

### `backend/package.json`
```json
{
  "name": "laila-backend",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js"
  },
  "dependencies": {
    "@whiskeysockets/baileys": "^6.7.9",
    "express": "^4.18.2",
    "socket.io": "^4.7.2",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "groq-sdk": "^0.3.2",
    "qrcode": "^1.5.3",
    "pino": "^8.15.0"
  }
}
```

### `backend/.env`
```env
PORT=3001
GROQ_API_KEY=your_groq_api_key_here   # Get free at console.groq.com
OWNER_PHONE=923001234567               # Your number (with country code, no +)
FRONTEND_URL=https://your-app.vercel.app
```

---

## Step 2: Core Files

### `backend/src/index.js`
```js
import express from 'express';
import { createServer } from 'http';
import { Server } from 'socket.io';
import cors from 'cors';
import dotenv from 'dotenv';
import sessionRoutes from './routes/session.routes.js';
import messageRoutes from './routes/message.routes.js';

dotenv.config();

const app = express();
const httpServer = createServer(app);

export const io = new Server(httpServer, {
  cors: { origin: process.env.FRONTEND_URL || '*', methods: ['GET', 'POST'] }
});

app.use(cors({ origin: process.env.FRONTEND_URL || '*' }));
app.use(express.json());

app.use('/api/session', sessionRoutes);
app.use('/api/message', messageRoutes);

app.get('/health', (_, res) => res.json({ status: 'Laila is alive ✨' }));

httpServer.listen(process.env.PORT || 3001, () => {
  console.log(`🌹 Laila backend running on port ${process.env.PORT || 3001}`);
});
```

---

### `backend/src/whatsapp/sessionManager.js`
```js
import makeWASocket, {
  DisconnectReason,
  useMultiFileAuthState,
  fetchLatestBaileysVersion,
  makeCacheableSignalKeyStore
} from '@whiskeysockets/baileys';
import { Boom } from '@hapi/boom';
import pino from 'pino';
import path from 'path';
import { io } from '../index.js';
import { handleIncomingMessage } from './messageHandler.js';

const sessions = new Map(); // sessionId -> socket instance
const logger = pino({ level: 'silent' });

export function getSession(sessionId) {
  return sessions.get(sessionId);
}

export function getAllSessions() {
  return [...sessions.entries()].map(([id, sock]) => ({
    id,
    status: sock?.user ? 'connected' : 'connecting',
    user: sock?.user
  }));
}

export async function createSession(sessionId, phoneNumber, mode = 'qr') {
  const sessionDir = path.resolve(`sessions/${sessionId}`);
  const { state, saveCreds } = await useMultiFileAuthState(sessionDir);
  const { version } = await fetchLatestBaileysVersion();

  const sock = makeWASocket({
    version,
    logger,
    auth: {
      creds: state.creds,
      keys: makeCacheableSignalKeyStore(state.keys, logger)
    },
    printQRInTerminal: false,
    browser: ['Laila', 'Chrome', '1.0.0']
  });

  sessions.set(sessionId, sock);

  // --- QR / Pairing Code ---
  if (mode === 'qr') {
    sock.ev.on('connection.update', async (update) => {
      const { qr, connection, lastDisconnect } = update;
      if (qr) {
        io.to(sessionId).emit('qr', qr);
      }
      if (connection === 'open') {
        io.to(sessionId).emit('connected', { user: sock.user });
      }
      if (connection === 'close') {
        const shouldReconnect =
          (lastDisconnect?.error instanceof Boom)
            ? lastDisconnect.error.output.statusCode !== DisconnectReason.loggedOut
            : true;
        if (shouldReconnect) createSession(sessionId, phoneNumber, mode);
        else sessions.delete(sessionId);
      }
    });
  }

  if (mode === 'pairing' && phoneNumber) {
    // Wait a moment before requesting pairing code
    setTimeout(async () => {
      try {
        const code = await sock.requestPairingCode(phoneNumber);
        io.to(sessionId).emit('pairingCode', code);
      } catch (e) {
        io.to(sessionId).emit('error', e.message);
      }
    }, 2000);

    sock.ev.on('connection.update', async ({ connection, lastDisconnect }) => {
      if (connection === 'open') io.to(sessionId).emit('connected', { user: sock.user });
      if (connection === 'close') {
        const shouldReconnect =
          (lastDisconnect?.error instanceof Boom)
            ? lastDisconnect.error.output.statusCode !== DisconnectReason.loggedOut
            : true;
        if (shouldReconnect) createSession(sessionId, phoneNumber, mode);
        else sessions.delete(sessionId);
      }
    });
  }

  sock.ev.on('creds.update', saveCreds);

  // --- Incoming messages ---
  sock.ev.on('messages.upsert', async ({ messages, type }) => {
    if (type !== 'notify') return;
    for (const msg of messages) {
      if (!msg.key.fromMe) {
        await handleIncomingMessage(sock, msg, sessionId);
      }
    }
  });

  return sock;
}

export async function disconnectSession(sessionId) {
  const sock = sessions.get(sessionId);
  if (sock) {
    await sock.logout();
    sessions.delete(sessionId);
  }
}
```

---

### `backend/src/whatsapp/messageHandler.js`
```js
import { getAIReply } from '../ai/aiReply.js';
import { sendReaction } from './reactionHandler.js';
import dotenv from 'dotenv';
dotenv.config();

// Track conversations where owner has manually replied
const ownerOverride = new Map(); // jid -> timestamp

export function setOwnerOverride(jid) {
  ownerOverride.set(jid, Date.now());
}

export async function handleIncomingMessage(sock, msg, sessionId) {
  const jid = msg.key.remoteJid;
  if (!jid || jid.includes('status') || jid.includes('broadcast')) return;

  const text =
    msg.message?.conversation ||
    msg.message?.extendedTextMessage?.text ||
    msg.message?.imageMessage?.caption ||
    '';

  if (!text) return;

  // 1. Send emoji reaction
  await sendReaction(sock, msg);

  // 2. Check owner override (silent for 30 min after manual reply)
  const lastOverride = ownerOverride.get(jid);
  if (lastOverride && Date.now() - lastOverride < 30 * 60 * 1000) {
    console.log(`[Laila] Silent mode for ${jid}`);
    return;
  }

  // 3. Typing indicator for realism
  await sock.sendPresenceUpdate('composing', jid);
  await delay(1200 + Math.random() * 1800); // 1.2–3 second delay

  // 4. Generate AI reply
  const reply = await getAIReply(text, jid);

  // 5. Send reply
  await sock.sendMessage(jid, { text: reply });
  await sock.sendPresenceUpdate('paused', jid);
}

function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}
```

---

### `backend/src/whatsapp/reactionHandler.js`
```js
const REACTION_POOLS = {
  greeting: ['❤️', '😊', '✨', '🌹', '💫'],
  question: ['🤔', '💭', '👀', '✨', '💡'],
  positive: ['🔥', '💯', '❤️', '😍', '✨', '🥰'],
  default: ['❤️', '😊', '✨', '🔥', '💫', '🌹', '💖', '😄', '👏', '🥰']
};

function detectContext(text) {
  if (/\?/.test(text)) return 'question';
  if (/(hi|hello|hola|salam|hey)/i.test(text)) return 'greeting';
  if (/(good|great|love|nice|amazing)/i.test(text)) return 'positive';
  return 'default';
}

function pickRandom(arr) {
  return arr[Math.floor(Math.random() * arr.length)];
}

export async function sendReaction(sock, msg) {
  try {
    const text =
      msg.message?.conversation ||
      msg.message?.extendedTextMessage?.text || '';
    const pool = REACTION_POOLS[detectContext(text)];
    const emoji = pickRandom(pool);

    await sock.sendMessage(msg.key.remoteJid, {
      react: { text: emoji, key: msg.key }
    });
  } catch (e) {
    console.error('[Reaction error]', e.message);
  }
}
```

---

### `backend/src/ai/aiReply.js`
```js
import Groq from 'groq-sdk';
import dotenv from 'dotenv';
dotenv.config();

const groq = new Groq({ apiKey: process.env.GROQ_API_KEY });

// Per-user conversation history (last 10 messages for context)
const conversationHistory = new Map();

const SYSTEM_PROMPT = `You are Laila — a warm, friendly, slightly playful and emotionally intelligent WhatsApp friend. You respond naturally like a real human would.

Rules:
- Keep replies SHORT: 1–3 sentences max
- Mix English and Urdu naturally (Roman Urdu is fine, e.g. "Haan bilkul", "Kya haal hai", "Shukriya")
- Use emojis naturally but sparingly (1–2 per message max)
- If user seems sad → be caring and empathetic
- If user is playful/flirty → be playful back but stay respectful
- If user is serious/professional → match that energy
- NEVER mention AI, ChatGPT, or that you're a bot
- Sound like a real person texting on WhatsApp
- Never give long paragraph answers
- Respond in the same language mix the user uses`;

export async function getAIReply(userMessage, jid) {
  try {
    // Get or initialize conversation history
    if (!conversationHistory.has(jid)) {
      conversationHistory.set(jid, []);
    }
    const history = conversationHistory.get(jid);

    // Add user message
    history.push({ role: 'user', content: userMessage });

    // Keep last 10 exchanges only
    if (history.length > 20) history.splice(0, 2);

    const response = await groq.chat.completions.create({
      model: 'llama3-8b-8192', // Free, fast Groq model
      messages: [
        { role: 'system', content: SYSTEM_PROMPT },
        ...history
      ],
      max_tokens: 150,
      temperature: 0.85
    });

    const reply = response.choices[0]?.message?.content?.trim() || 'Haan? 😊';

    // Add assistant reply to history
    history.push({ role: 'assistant', content: reply });

    return reply;
  } catch (e) {
    console.error('[AI Error]', e.message);
    return 'Abhi busy hoon, thodi der mein baat karte hain ✨';
  }
}
```

---

### `backend/src/routes/session.routes.js`
```js
import express from 'express';
import { v4 as uuidv4 } from 'uuid';
import { createSession, disconnectSession, getAllSessions } from '../whatsapp/sessionManager.js';

const router = express.Router();

router.get('/', (req, res) => {
  res.json(getAllSessions());
});

router.post('/connect', async (req, res) => {
  const { phoneNumber, mode } = req.body; // mode: 'qr' | 'pairing'
  const sessionId = `session_${uuidv4()}`;
  await createSession(sessionId, phoneNumber?.replace(/\D/g, ''), mode || 'qr');
  res.json({ sessionId, message: 'Session initializing...' });
});

router.post('/disconnect', async (req, res) => {
  const { sessionId } = req.body;
  await disconnectSession(sessionId);
  res.json({ success: true });
});

export default router;
```

### `backend/src/routes/message.routes.js`
```js
import express from 'express';
import { setOwnerOverride } from '../whatsapp/messageHandler.js';
import { getSession } from '../whatsapp/sessionManager.js';

const router = express.Router();

// Called when owner manually replies (to silence the bot)
router.post('/owner-reply', (req, res) => {
  const { jid } = req.body;
  setOwnerOverride(jid);
  res.json({ success: true, message: `Bot silenced for ${jid} for 30 minutes` });
});

// Send manual message from dashboard
router.post('/send', async (req, res) => {
  const { sessionId, jid, text } = req.body;
  const sock = getSession(sessionId);
  if (!sock) return res.status(404).json({ error: 'Session not found' });
  await sock.sendMessage(jid, { text });
  setOwnerOverride(jid); // Silence bot after manual send
  res.json({ success: true });
});

export default router;
```

---

## Step 3: Free AI Alternatives

| Provider | Free Tier | Model | Notes |
|---|---|---|---|
| **Groq** | 14,400 req/day | llama3-8b | Fastest, recommended |
| **Google Gemini** | 1500 req/day | gemini-1.5-flash | Great quality |
| **Together AI** | $25 free credit | mixtral-8x7b | Good fallback |
| **Ollama** (self-host) | Unlimited | llama3 | If you have a VPS |

### Gemini Alternative (`aiReply.js` swap):
```js
import { GoogleGenerativeAI } from '@google/generative-ai';
const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY);
const model = genAI.getGenerativeModel({ model: 'gemini-1.5-flash' });

export async function getAIReply(userMessage, jid) {
  const result = await model.generateContent(
    `${SYSTEM_PROMPT}\n\nUser: ${userMessage}\nLaila:`
  );
  return result.response.text().trim();
}
```

---

## Step 4: Frontend Dashboard

See `dashboard.html` for the full React/HTML dashboard with:
- Session connection panel (QR + Pairing code)
- Live session status cards
- Quick reply panel

---

## Step 5: Deployment

### Backend (Railway/Render)
```bash
# Railway
railway login
railway init
railway up

# OR Render: connect GitHub repo, set env vars, deploy
```

```
Environment Variables to set:
GROQ_API_KEY=gsk_xxxxx
OWNER_PHONE=923001234567
FRONTEND_URL=https://your-dashboard.vercel.app
PORT=3001
```

### Frontend (Vercel)
```bash
cd frontend
vercel --prod
# Set VITE_BACKEND_URL=https://your-railway-app.railway.app
```

### Persistent Sessions on Railway
Add a volume mount at `/app/sessions` in Railway settings so sessions survive restarts.

---

## Step 6: Adding New Features Later

The architecture is plugin-ready. Example: adding a `/schedule` command:

```js
// In messageHandler.js, add to handleIncomingMessage():
if (text.startsWith('/schedule')) {
  return handleScheduleCommand(sock, jid, text);
}
```

Future features you can add easily:
- 📋 Auto-save contacts to Google Sheets
- 📅 Message scheduling
- 📊 Chat analytics dashboard  
- 🎤 Voice message transcription
- 🖼️ AI image replies
- 📢 Broadcast messages
- 🔕 Do Not Disturb hours

---

## Quick Start (5 minutes)

```bash
# 1. Clone and install
git clone <your-repo>
cd laila-bot/backend
npm install

# 2. Get free Groq API key at console.groq.com (takes 1 min)

# 3. Set .env file

# 4. Start
npm run dev

# 5. Open frontend, enter your phone number, scan QR or use pairing code
```

**That's it — Laila is alive! 🌹**
