# 📦 Architecture complète du projet

```
SDV-PROJET/
├── backend/
│   ├── controllers/              (facultatif – logique métier hors routes)
│   ├── models/
│   │   ├── User.js
│   │   └── Message.js
│   ├── routes/
│   │   ├── auth.js
│   │   └── messages.js
│   ├── socket/
│   │   └── chat.js
│   ├── server.js
│   ├── .env.example
│   └── package.json
│
├── frontend/
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   ├── login/page.tsx
│   │   │   ├── register/page.tsx
│   │   │   └── chat/page.tsx
│   │   ├── components/
│   │   │   ├── Header.tsx
│   │   │   └── MessageItem.tsx
│   │   ├── lib/
│   │   │   ├── api.ts
│   │   │   └── auth.ts
│   │   ├── styles/globals.css
│   │   └── types/
│   │       ├── user.d.ts
│   │       └── message.d.ts
│   ├── public/favicon.ico
│   ├── tailwind.config.ts
│   ├── postcss.config.js
│   ├── tsconfig.json
│   ├── .eslintrc.json
│   └── package.json
│
├── docker-compose.yml
├── .gitignore
└── .github/workflows/deploy.yml
```

---

## 🛠️ BACKEND (`backend/`)

### `package.json`

```json
{
  "name": "backend",
  "version": "1.0.0",
  "description": "Backend Node.js pour chat temps réel",
  "type": "module",
  "main": "server.js",
  "scripts": {
    "dev": "nodemon server.js",
    "start": "node server.js",
    "test": "echo \"Pas de tests pour l'instant\" && exit 0"
  },
  "dependencies": {
    "bcrypt": "^6.0.0",
    "cors": "^2.8.5",
    "dotenv": "^17.0.1",
    "express": "^5.1.0",
    "jsonwebtoken": "^9.0.2",
    "mongoose": "^8.16.1",
    "socket.io": "^4.8.1"
  },
  "devDependencies": {
    "nodemon": "^3.1.10"
  }
}
```

### `.env.example`

```
PORT=5000
MONGO_URI=mongodb://localhost:27017/chatapp
JWT_SECRET=SuperSecretToken123
```

### `server.js`

```js
import 'dotenv/config';
import express from 'express';
import http from 'http';
import cors from 'cors';
import mongoose from 'mongoose';
import { Server } from 'socket.io';

import authRoutes from './routes/auth.js';
import messageRoutes from './routes/messages.js';
import socketSetup from './socket/chat.js';

const app = express();
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: '*' } });

app.use(cors());
app.use(express.json());

app.use('/api/auth', authRoutes);
app.use('/api/messages', messageRoutes);

mongoose.connect(process.env.MONGO_URI)
  .then(() => console.log('✅  MongoDB connecté'))
  .catch(err => console.error('❌  MongoDB error', err));

socketSetup(io);

const PORT = process.env.PORT || 5000;
server.listen(PORT, () => console.log(`🚀  Backend prêt sur http://localhost:${PORT}`));
```

### `models/User.js`

```js
import mongoose from 'mongoose';
const schema = new mongoose.Schema({
  username: { type: String, unique: true },
  password: String
});
export default mongoose.model('User', schema);
```

### `models/Message.js`

```js
import mongoose from 'mongoose';
const schema = new mongoose.Schema({
  sender: String,
  recipient: String,              // null si message public
  content: String,
  timestamp: { type: Date, default: Date.now }
});
export default mongoose.model('Message', schema);
```

### `routes/auth.js`

```js
import { Router } from 'express';
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import User from '../models/User.js';

const router = Router();

router.post('/register', async (req, res) => {
  const { username, password } = req.body;
  if (await User.exists({ username })) {
    return res.status(409).json({ message: 'Nom déjà utilisé' });
  }
  const hashed = await bcrypt.hash(password, 10);
  await User.create({ username, password: hashed });
  res.status(201).json({ message: 'Compte créé' });
});

router.post('/login', async (req, res) => {
  const { username, password } = req.body;
  const user = await User.findOne({ username });
  if (!user) return res.status(404).json({ message: 'Utilisateur introuvable' });

  const valid = await bcrypt.compare(password, user.password);
  if (!valid) return res.status(401).json({ message: 'Mot de passe invalide' });

  const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: '1h' });
  res.json({ token });
});

export default router;
```

### `routes/messages.js`

```js
import { Router } from 'express';
import jwt from 'jsonwebtoken';
import Message from '../models/Message.js';

const router = Router();

// Middleware JWT basique
router.use((req, res, next) => {
  const auth = req.headers.authorization?.split(' ')[1];
  if (!auth) return res.sendStatus(401);
  try {
    req.user = jwt.verify(auth, process.env.JWT_SECRET);
    next();
  } catch {
    res.sendStatus(403);
  }
});

router.get('/', async (_req, res) => {
  const msgs = await Message.find({ recipient: null }).sort({ timestamp: 1 });
  res.json(msgs);
});

router.delete('/:id', async (req, res) => {
  await Message.deleteOne({ _id: req.params.id, sender: req.user.id });
  res.sendStatus(204);
});

export default router;
```

### `socket/chat.js`

```js
import Message from '../models/Message.js';

export default (io) => {
  const online = {};

  io.on('connection', (socket) => {
    socket.on('join', (username) => {
      online[socket.id] = username;
      io.emit('notification', `${username} a rejoint le salon`);
    });

    socket.on('message', async ({ username, text }) => {
      const msg = await Message.create({ sender: username, recipient: null, content: text });
      io.emit('message', msg);          // envoie toute la doc pour conserver l'ID
    });

    socket.on('disconnect', () => {
      const name = online[socket.id];
      delete online[socket.id];
      io.emit('notification', `${name} a quitté le salon`);
    });
  });
};
```

---

## 🎨 FRONTEND (`frontend/`)

### `package.json` (généré par Next.js)

*(conserve celui généré, aucun changement supplémentaire nécessaire)*

---

### `tailwind.config.ts`

```ts
import type { Config } from 'tailwindcss';
const config: Config = {
  content: ['./src/**/*.{ts,tsx,js,jsx}'],
  theme: { extend: {} },
  plugins: []
};
export default config;
```

### `postcss.config.js`

```js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

### `src/styles/globals.css`

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### `tsconfig.json` – alias `@/*`

```json
{
  "compilerOptions": {
    "baseUrl": "./",
    "paths": {
      "@/*": ["src/*"]
    },
    ...
  }
}
```

### `src/types/user.d.ts`

```ts
export interface User {
  username: string;
  token: string;
}
```

### `src/types/message.d.ts`

```ts
export interface Message {
  _id: string;
  sender: string;
  recipient: string | null;
  content: string;
  timestamp: string;
  system?: boolean;
}
```

### `src/lib/api.ts`

```ts
const API = process.env.NEXT_PUBLIC_API || 'http://localhost:5000/api';

export async function register(username: string, password: string) {
  const r = await fetch(`${API}/auth/register`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username, password })
  });
  if (!r.ok) throw new Error(await r.text());
}

export async function login(username: string, password: string) {
  const r = await fetch(`${API}/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username, password })
  });
  if (!r.ok) throw new Error(await r.text());
  return (await r.json()).token as string;
}

export async function getPublicMessages(token: string) {
  const r = await fetch(`${API}/messages`, {
    headers: { Authorization: `Bearer ${token}` }
  });
  return await r.json();
}
```

### `src/lib/auth.ts`

```ts
export function saveToken(t: string) {
  localStorage.setItem('token', t);
}
export function getToken() {
  return localStorage.getItem('token');
}
export function logout() {
  localStorage.removeItem('token');
}
```

### `src/components/Header.tsx`

```tsx
'use client';
import Link from 'next/link';
import { logout } from '@/lib/auth';

export default function Header() {
  return (
    <header className="bg-gray-900 text-white p-4 flex justify-between">
      <h1 className="font-bold">Chat App</h1>
      <nav className="space-x-4">
        <Link href="/chat">Salon</Link>
        <button onClick={logout} className="underline">Déconnexion</button>
      </nav>
    </header>
  );
}
```

### `src/components/MessageItem.tsx`

```tsx
import { Message } from '@/types/message';

export default function MessageItem({ msg }: { msg: Message }) {
  return (
    <div className="mb-1">
      {msg.system ? (
        <em>{msg.content}</em>
      ) : (
        <>
          <b>{msg.sender}</b>: {msg.content}
        </>
      )}
    </div>
  );
}
```

### `src/app/layout.tsx`

```tsx
import '@/styles/globals.css';
import { ReactNode } from 'react';

export const metadata = { title: 'Chat App' };

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="fr">
      <body className="min-h-screen bg-gray-100">{children}</body>
    </html>
  );
}
```

### `src/app/page.tsx` (redirection vers login)

```tsx
import { redirect } from 'next/navigation';
export default function Home() {
  redirect('/login');
}
```

### `src/app/login/page.tsx`

```tsx
'use client';
import { useState } from 'react';
import { login } from '@/lib/api';
import { saveToken } from '@/lib/auth';
import { useRouter } from 'next/navigation';

export default function Login() {
  const [username, setU] = useState('');
  const [password, setP] = useState('');
  const [err, setErr] = useState('');
  const router = useRouter();

  async function submit() {
    try {
      const t = await login(username, password);
      saveToken(t);
      router.push('/chat');
    } catch (e: any) {
      setErr(e.message);
    }
  }

  return (
    <div className="flex flex-col items-center mt-20">
      <h2 className="text-2xl mb-4">Connexion</h2>
      {err && <p className="text-red-600">{err}</p>}
      <input
        className="border p-2 mb-2"
        placeholder="Pseudo"
        value={username}
        onChange={e => setU(e.target.value)}
      />
      <input
        type="password"
        className="border p-2 mb-2"
        placeholder="Mot de passe"
        value={password}
        onChange={e => setP(e.target.value)}
      />
      <button className="bg-blue-600 text-white px-4 py-2" onClick={submit}>
        Entrer
      </button>
    </div>
  );
}
```

### `src/app/register/page.tsx`

*(identique à login mais appelant `register` et redirigeant vers login)*

```tsx
'use client';
import { useState } from 'react';
import { register } from '@/lib/api';
import { useRouter } from 'next/navigation';

export default function Register() {
  const [username, setU] = useState('');
  const [password, setP] = useState('');
  const [err, setErr] = useState('');
  const router = useRouter();

  async function submit() {
    try {
      await register(username, password);
      router.push('/login');
    } catch (e: any) {
      setErr(e.message);
    }
  }

  return (
    <div className="flex flex-col items-center mt-20">
      <h2 className="text-2xl mb-4">Inscription</h2>
      {err && <p className="text-red-600">{err}</p>}
      <input
        className="border p-2 mb-2"
        placeholder="Pseudo"
        value={username}
        onChange={e => setU(e.target.value)}
      />
      <input
        type="password"
        className="border p-2 mb-2"
        placeholder="Mot de passe"
        value={password}
        onChange={e => setP(e.target.value)}
      />
      <button className="bg-green-600 text-white px-4 py-2" onClick={submit}>
        Créer le compte
      </button>
    </div>
  );
}
```

### `src/app/chat/page.tsx`

```tsx
'use client';
import { useEffect, useState, useRef } from 'react';
import io from 'socket.io-client';
import { getToken } from '@/lib/auth';
import { getPublicMessages } from '@/lib/api';
import MessageItem from '@/components/MessageItem';
import { Message } from '@/types/message';

const socket = io('http://localhost:5000');

export default function Chat() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [text, setText] = useState('');
  const endRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const username = prompt('Ton pseudo ?') || 'Anonyme';
    socket.emit('join', username);

    socket.on('message', (m: Message) =>
      setMessages(prev => [...prev, m])
    );
    socket.on('notification', (txt: string) =>
      setMessages(prev => [...prev, { _id: crypto.randomUUID(), sender: '', recipient: null, content: txt, timestamp: new Date().toISOString(), system: true }])
    );

    // historique
    (async () => {
      const token = getToken();
      if (token) {
        const histo = await getPublicMessages(token);
        setMessages(histo);
      }
    })();

    return () => socket.disconnect();
  }, []);

  useEffect(() => {
    endRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  function send() {
    socket.emit('message', { text, username: 'Moi' });
    setText('');
  }

  return (
    <div className="max-w-xl mx-auto mt-6">
      <div className="border h-96 overflow-y-scroll p-2 bg-white">
        {messages.map(m => <MessageItem key={m._id} msg={m} />)}
        <div ref={endRef} />
      </div>
      <div className="flex mt-2">
        <input
          className="flex-1 border p-2"
          value={text}
          onChange={e => setText(e.target.value)}
        />
        <button className="bg-blue-600 text-white px-4" onClick={send}>
          Envoyer
        </button>
      </div>
    </div>
  );
}
```

---

## 🐳 Docker (dev uniquement)

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  mongo:
    image: mongo
    container_name: mongo-chat
    ports:
      - '27017:27017'
    volumes:
      - mongo-data:/data/db

volumes:
  mongo-data:
```

---

## ⚙️ GitHub Actions démo (`.github/workflows/deploy.yml`)

```yaml
name: CI

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    services:
      mongo:
        image: mongo
        ports: ['27017:27017']
        options: --health-cmd "mongosh --eval 'db.stats()' --quiet"

    steps:
      - uses: actions/checkout@v3

      # Backend
      - name: Install backend
        run: |
          cd backend
          npm ci
      - name: Lint backend
        run: cd backend && npx eslint .

      # Frontend
      - name: Install frontend
        run: |
          cd frontend
          npm ci
      - name: Build frontend
        run: cd frontend && npm run build
```

*(ajoute un job de déploiement Azure ou Docker selon ta cible – même principe que précédemment)*

---

# 🚀 Lancement local

```bash
# 1. Démarrer MongoDB
docker-compose up -d

# 2. Backend
cd backend
npm install
npm run dev

# 3. Frontend
cd ../frontend
npm install
npm run dev
```

Tu obtiens :

* **API + WebSocket** : `http://localhost:5000`
* **Front** : `http://localhost:3000`
