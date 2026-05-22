# Tiny AI Society Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use subagent-driven-development (recommended) or executing-plans to implement this plan task-by-task.

**Goal:** Build a web-based chat platform where 8 AI-powered characters live as a simulated society, with the human as a participant.

**Architecture:** Single Next.js server on Windows PC running the chat UI, API, WebSockets, and AI orchestration. SQLite via Prisma for storage. Ollama serves the AI models, loading/unloading them sequentially via a queue.

**Tech Stack:** Next.js 14 (App Router), TypeScript, Prisma + SQLite, Socket.IO, Tailwind CSS, Zustand (client state), Ollama REST API

---

## File Structure

```
tiny-ai-society/
├── prisma/
│   └── schema.prisma
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── globals.css
│   │   └── api/
│   │       ├── citizens/route.ts
│   │       ├── channels/route.ts
│   │       ├── channels/[id]/messages/route.ts
│   │       └── messages/route.ts
│   ├── components/
│   │   ├── Sidebar.tsx
│   │   ├── ChatArea.tsx
│   │   ├── MessageList.tsx
│   │   ├── MessageBubble.tsx
│   │   ├── MessageInput.tsx
│   │   ├── OnlineIndicator.tsx
│   │   └── CitizenProfile.tsx
│   ├── lib/
│   │   ├── prisma.ts
│   │   ├── socket.ts
│   │   └── ai-scheduler.ts
│   ├── store/
│   │   └── chat.ts
│   └── types/
│       └── index.ts
├── seed.ts
├── server.ts (custom Next.js + Socket.IO server)
├── .env
├── tailwind.config.ts
├── tsconfig.json
└── package.json
```

---

### Task 1: Scaffold Next.js project

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `tailwind.config.ts`
- Create: `postcss.config.js`
- Create: `next.config.js`
- Create: `.env`
- Create: `src/app/globals.css`
- Create: `src/app/layout.tsx`
- Create: `src/app/page.tsx`
- Create: `src/types/index.ts`

- [ ] **Step 1: Create project directory and initialize**

```bash
cd tiny-ai-society
npm init -y
```

- [ ] **Step 2: Install dependencies**

```bash
npm install next@14 react react-dom @prisma/client socket.io socket.io-client zustand
npm install -D typescript @types/node @types/react @types/react-dom prisma tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

- [ ] **Step 3: Write tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "es2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

- [ ] **Step 4: Write next.config.js**

```js
/** @type {import('next').NextConfig} */
const nextConfig = {}
module.exports = nextConfig
```

- [ ] **Step 5: Write tailwind.config.ts**

```ts
import type { Config } from "tailwindcss"
const config: Config = {
  content: ["./src/**/*.{ts,tsx}"],
  theme: { extend: {} },
  plugins: [],
}
export default config
```

- [ ] **Step 6: Write .env**

```
DATABASE_URL="file:./dev.db"
NEXT_PUBLIC_SOCKET_URL="http://localhost:3000"
```

- [ ] **Step 7: Write src/app/globals.css**

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

body {
  margin: 0;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
}
```

- [ ] **Step 8: Write src/types/index.ts**

```ts
export interface Citizen {
  id: string
  name: string
  age: number
  gender: string
  systemPrompt: string
  modelPath: string | null
  isHuman: boolean
  avatarSeed: string
  createdAt: string
}

export interface Channel {
  id: string
  name: string
  type: "public" | "dm"
  memberIds: string[]
  createdAt: string
}

export interface Message {
  id: string
  channelId: string
  senderId: string
  sender?: Citizen
  content: string
  createdAt: string
}
```

- [ ] **Step 9: Write src/app/layout.tsx**

```tsx
import type { Metadata } from "next"
import "./globals.css"

export const metadata: Metadata = {
  title: "Tiny AI Society",
  description: "A tiny society of AI personalities",
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className="h-screen bg-neutral-900 text-white">{children}</body>
    </html>
  )
}
```

- [ ] **Step 10: Write src/app/page.tsx (placeholder)**

```tsx
export default function Home() {
  return (
    <div className="flex h-full items-center justify-center">
      <p className="text-neutral-400">Loading...</p>
    </div>
  )
}
```

- [ ] **Step 11: Add build/start scripts to package.json**

```json
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "start": "node server.js",
  "seed": "npx tsx seed.ts"
}
```

- [ ] **Step 12: Verify it compiles**

```bash
npm run build
```
Expected: Successful build, no errors.

- [ ] **Step 13: Commit**

```bash
git add -A && git commit -m "feat: scaffold Next.js project with Tailwind, types"
```

---

### Task 2: Set up Prisma + SQLite schema

**Files:**
- Create: `prisma/schema.prisma`
- Create: `src/lib/prisma.ts`

- [ ] **Step 1: Write prisma/schema.prisma**

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model Citizen {
  id           String    @id @default(uuid())
  name         String
  age          Int
  gender       String
  systemPrompt String
  modelPath    String?
  avatarSeed   String    @default("")
  isHuman      Boolean   @default(false)
  createdAt    DateTime  @default(now())
  messages     Message[]
  memories     Memory[]
}

model Channel {
  id        String    @id @default(uuid())
  name      String
  type      String    @default("public")
  memberIds String    @default("[]")
  createdAt DateTime  @default(now())
  messages  Message[]
}

model Message {
  id        String   @id @default(uuid())
  channelId String
  senderId  String
  content   String
  createdAt DateTime @default(now())
  channel   Channel  @relation(fields: [channelId], references: [id])
  sender    Citizen  @relation(fields: [senderId], references: [id])
}

model Memory {
  id               String   @id @default(uuid())
  citizenId        String
  summary          String
  relatedCitizenIds String  @default("[]")
  createdAt        DateTime @default(now())
  citizen          Citizen  @relation(fields: [citizenId], references: [id])
}
```

- [ ] **Step 2: Write src/lib/prisma.ts**

```ts
import { PrismaClient } from "@prisma/client"

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient }

export const prisma = globalForPrisma.prisma ?? new PrismaClient()

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma
```

- [ ] **Step 3: Generate Prisma client**

```bash
npx prisma generate && npx prisma db push
```
Expected: Schema pushed to SQLite database at `prisma/dev.db`.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "feat: add Prisma schema with Citizen, Channel, Message, Memory models"
```

---

### Task 3: Database seed script with the 8 AI citizens + default channels

**Files:**
- Create: `seed.ts`

- [ ] **Step 1: Write seed.ts**

```ts
import { PrismaClient } from "@prisma/client"

const prisma = new PrismaClient()

const citizens = [
  { name: "Luna", age: 22, gender: "F", avatarSeed: "luna_artsy", isHuman: false, modelPath: null, systemPrompt: "You are Luna, 22. An artsy photographer with a dreamy personality. You notice beauty in small things, posts aesthetic photos, and are sensitive to others' emotions." },
  { name: "Jax", age: 18, gender: "M", avatarSeed: "jax_gamer", isHuman: false, modelPath: null, systemPrompt: "You are Jax, 18. A chaotic gamer who lives for memes and loud energy. You're Zara's neighbor and childhood friend. You have a short attention span but a good heart." },
  { name: "Maya", age: 25, gender: "F", avatarSeed: "maya_phil", isHuman: false, modelPath: null, systemPrompt: "You are Maya, 25. A philosophy graduate student who overanalyzes everything. You love deep conversations about existence, ethics, and meaning. You're thoughtful and a bit intense." },
  { name: "Leo", age: 24, gender: "M", avatarSeed: "leo_music", isHuman: false, modelPath: null, systemPrompt: "You are Leo, 24. A musician and producer. You're chill, speak in music references, and are a night owl. You have a calming presence and give good advice." },
  { name: "Sofia", age: 18, gender: "F", avatarSeed: "sofia_astro", isHuman: false, modelPath: null, systemPrompt: "You are Sofia, 18. Dramatic, obsessed with astrology and fashion. You think Mercury retrograde explains everything. You're passionate and a bit extra but love your friends deeply." },
  { name: "Eli", age: 26, gender: "M", avatarSeed: "eli_writer", isHuman: false, modelPath: null, systemPrompt: "You are Eli, 26. A cynical bartender and aspiring writer. You have dry humor and a deadpan delivery, but you're secretly soft inside. You observe more than you participate." },
  { name: "Zara", age: 17, gender: "F", avatarSeed: "zara_gremlin", isHuman: false, modelPath: null, systemPrompt: "You are Zara, 17. A cute chaotic gremlin. You live next door to Jax and have known him forever. You're sweet and nice but have a mischievous streak — evil in a cute way. You love causing harmless trouble." },
  { name: "Finn", age: 17, gender: "M", avatarSeed: "finn_skater", isHuman: false, modelPath: null, systemPrompt: "You are Finn, 17. A skater and streamer with a short attention span. You're chaotic good — you mean well but get distracted easily. You're energetic and always down for something fun." },
  { name: "You", age: 0, gender: "U", avatarSeed: "human_user", isHuman: true, modelPath: null, systemPrompt: "" },
]

const channelData = [
  { name: "general", type: "public" },
  { name: "random", type: "public" },
  { name: "music", type: "public" },
  { name: "luna-jax", type: "dm" },
]

async function main() {
  // Clear existing data
  await prisma.memory.deleteMany()
  await prisma.message.deleteMany()
  await prisma.channel.deleteMany()
  await prisma.citizen.deleteMany()

  // Create citizens
  for (const c of citizens) {
    await prisma.citizen.create({ data: c })
  }

  // Create channels
  for (const c of channelData) {
    await prisma.channel.create({ data: c })
  }

  console.log("✅ Seed complete")
}

main().catch(console.error).finally(() => prisma.$disconnect())
```

- [ ] **Step 2: Install tsx for running the seed**

```bash
npm install -D tsx
```

- [ ] **Step 3: Run the seed script**

```bash
npx tsx seed.ts
```
Expected: "✅ Seed complete"

- [ ] **Step 4: Verify data**

```bash
npx prisma studio
```
Expected: Tables visible with 9 citizens, 4 channels.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: add seed script with 8 AI citizens and default channels"
```

---

### Task 4: Set up Socket.IO custom server

**Files:**
- Create: `server.ts`

- [ ] **Step 1: Install socket.io dependencies**

```bash
npm install socket.io socket.io-client
```

- [ ] **Step 2: Write server.ts**

```ts
import { createServer } from "http"
import { parse } from "url"
import next from "next"
import { Server } from "socket.io"

const dev = process.env.NODE_ENV !== "production"
const hostname = "0.0.0.0"
const port = 3000

const app = next({ dev, hostname, port })
const handle = app.getRequestHandler()

app.prepare().then(() => {
  const httpServer = createServer((req, res) => {
    const parsedUrl = parse(req.url!, true)
    handle(req, res, parsedUrl)
  })

  const io = new Server(httpServer, {
    cors: { origin: "*", methods: ["GET", "POST"] },
  })

  io.on("connection", (socket) => {
    console.log("Client connected:", socket.id)

    socket.on("join-channel", (channelId: string) => {
      socket.join(channelId)
    })

    socket.on("leave-channel", (channelId: string) => {
      socket.leave(channelId)
    })

    socket.on("disconnect", () => {
      console.log("Client disconnected:", socket.id)
    })
  })

  httpServer.listen(port, hostname, () => {
    console.log(`> Ready on http://${hostname}:${port}`)
  })
})
```

- [ ] **Step 3: Write src/lib/socket.ts**

```ts
"use client"

import { io, Socket } from "socket.io-client"

let socket: Socket | null = null

export function getSocket(): Socket {
  if (!socket) {
    socket = io(process.env.NEXT_PUBLIC_SOCKET_URL!, {
      transports: ["websocket"],
    })
  }
  return socket
}
```

- [ ] **Step 4: Update package.json start script to use server.ts**

```json
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "start": "npx tsx server.ts",
  "seed": "npx tsx seed.ts"
}
```

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: add Socket.IO custom server with channel rooms"
```

---

### Task 5: API routes — citizens and channels

**Files:**
- Create: `src/app/api/citizens/route.ts`
- Create: `src/app/api/channels/route.ts`

- [ ] **Step 1: Write citizens API route**

```ts
import { NextResponse } from "next/server"
import { prisma } from "@/lib/prisma"

export async function GET() {
  const citizens = await prisma.citizen.findMany({
    orderBy: { name: "asc" },
  })
  return NextResponse.json(citizens)
}
```

- [ ] **Step 2: Write channels API route**

```ts
import { NextResponse } from "next/server"
import { prisma } from "@/lib/prisma"

export async function GET() {
  const channels = await prisma.channel.findMany({
    where: { type: "public" },
    orderBy: { name: "asc" },
  })
  return NextResponse.json(channels)
}
```

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "feat: add citizens and channels API routes"
```

---

### Task 6: API routes — messages

**Files:**
- Create: `src/app/api/channels/[id]/messages/route.ts`
- Create: `src/app/api/messages/route.ts`

- [ ] **Step 1: Write message list API (GET messages by channel)**

```ts
import { NextResponse } from "next/server"
import { prisma } from "@/lib/prisma"

export async function GET(
  _request: Request,
  { params }: { params: { id: string } }
) {
  const messages = await prisma.message.findMany({
    where: { channelId: params.id },
    include: { sender: true },
    orderBy: { createdAt: "asc" },
    take: 100,
  })
  return NextResponse.json(messages)
}
```

- [ ] **Step 2: Write message create API (POST new message)**

```ts
import { NextResponse } from "next/server"
import { prisma } from "@/lib/prisma"

export async function POST(request: Request) {
  const { channelId, senderId, content } = await request.json()
  const message = await prisma.message.create({
    data: { channelId, senderId, content },
    include: { sender: true },
  })
  return NextResponse.json(message, { status: 201 })
}
```

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "feat: add messages API routes (list by channel + create)"
```

---

### Task 7: Zustand chat store

**Files:**
- Create: `src/store/chat.ts`

- [ ] **Step 1: Write the chat store**

```ts
import { create } from "zustand"
import type { Citizen, Channel, Message } from "@/types"

interface ChatState {
  citizens: Citizen[]
  channels: Channel[]
  currentChannelId: string | null
  messages: Message[]
  loading: boolean

  setCitizens: (citizens: Citizen[]) => void
  setChannels: (channels: Channel[]) => void
  setCurrentChannelId: (id: string) => void
  setMessages: (messages: Message[]) => void
  addMessage: (message: Message) => void
  setLoading: (loading: boolean) => void
}

export const useChatStore = create<ChatState>((set) => ({
  citizens: [],
  channels: [],
  currentChannelId: null,
  messages: [],
  loading: true,

  setCitizens: (citizens) => set({ citizens }),
  setChannels: (channels) => set({ channels }),
  setCurrentChannelId: (id) => set({ currentChannelId: id }),
  setMessages: (messages) => set({ messages }),
  addMessage: (message) =>
    set((state) => ({ messages: [...state.messages, message] })),
  setLoading: (loading) => set({ loading }),
}))
```

- [ ] **Step 2: Commit**

```bash
git add -A && git commit -m "feat: add Zustand chat store"
```

---

### Task 8: Sidebar component

**Files:**
- Create: `src/components/Sidebar.tsx`

- [ ] **Step 1: Write Sidebar.tsx**

```tsx
"use client"

import { useEffect } from "react"
import { useChatStore } from "@/store/chat"

export default function Sidebar() {
  const citizens = useChatStore((s) => s.citizens)
  const channels = useChatStore((s) => s.channels)
  const currentChannelId = useChatStore((s) => s.currentChannelId)
  const setCurrentChannelId = useChatStore((s) => s.setCurrentChannelId)
  const setCitizens = useChatStore((s) => s.setCitizens)
  const setChannels = useChatStore((s) => s.setChannels)

  useEffect(() => {
    fetch("/api/citizens").then((r) => r.json()).then(setCitizens)
    fetch("/api/channels").then((r) => r.json()).then(setChannels)
  }, [])

  const aiCitizens = citizens.filter((c) => !c.isHuman)
  const human = citizens.find((c) => c.isHuman)

  return (
    <div className="w-64 bg-neutral-800 flex flex-col h-full border-r border-neutral-700">
      {/* Channel list */}
      <div className="p-3 border-b border-neutral-700">
        <h2 className="text-xs font-semibold text-neutral-400 uppercase tracking-wider mb-2">
          Channels
        </h2>
        {channels.map((ch) => (
          <button
            key={ch.id}
            onClick={() => setCurrentChannelId(ch.id)}
            className={`w-full text-left px-3 py-1.5 rounded text-sm mb-0.5 ${
              currentChannelId === ch.id
                ? "bg-neutral-700 text-white"
                : "text-neutral-300 hover:bg-neutral-700/50"
            }`}
          >
            # {ch.name}
          </button>
        ))}
      </div>

      {/* Online citizens */}
      <div className="p-3 flex-1">
        <h2 className="text-xs font-semibold text-neutral-400 uppercase tracking-wider mb-2">
          Online — {aiCitizens.length + (human ? 1 : 0)}
        </h2>
        {aiCitizens.map((c) => (
          <div
            key={c.id}
            className="flex items-center gap-2 px-3 py-1.5 text-sm text-neutral-300"
          >
            <span className="w-2 h-2 rounded-full bg-green-500" />
            {c.name}, {c.age}
          </div>
        ))}
        {human && (
          <div className="flex items-center gap-2 px-3 py-1.5 text-sm text-neutral-300">
            <span className="w-2 h-2 rounded-full bg-green-500" />
            You
          </div>
        )}
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Commit**

```bash
git add -A && git commit -m "feat: add Sidebar component with channels + online citizens"
```

---

### Task 9: MessageList and MessageBubble components

**Files:**
- Create: `src/components/MessageBubble.tsx`
- Create: `src/components/MessageList.tsx`

- [ ] **Step 1: Write MessageBubble.tsx**

```tsx
"use client"

import type { Message } from "@/types"

export default function MessageBubble({
  message,
  isHuman,
}: {
  message: Message
  isHuman: boolean
}) {
  const senderName = message.sender?.name ?? "Unknown"
  const time = new Date(message.createdAt).toLocaleTimeString([], {
    hour: "2-digit",
    minute: "2-digit",
  })

  return (
    <div className={`flex gap-3 px-4 py-2 hover:bg-neutral-800/50 ${isHuman ? "flex-row-reverse" : ""}`}>
      {/* Avatar placeholder */}
      <div className="w-10 h-10 rounded-full bg-neutral-700 flex-shrink-0 flex items-center justify-center text-sm font-medium">
        {senderName[0]}
      </div>

      <div className={`flex flex-col ${isHuman ? "items-end" : "items-start"}`}>
        <div className="flex items-center gap-2">
          <span className="text-sm font-semibold text-neutral-200">
            {senderName}
          </span>
          <span className="text-xs text-neutral-500">{time}</span>
        </div>
        <p className="text-sm text-neutral-300 mt-0.5">{message.content}</p>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Write MessageList.tsx**

```tsx
"use client"

import { useEffect, useRef } from "react"
import { useChatStore } from "@/store/chat"
import MessageBubble from "./MessageBubble"

export default function MessageList() {
  const messages = useChatStore((s) => s.messages)
  const currentChannelId = useChatStore((s) => s.currentChannelId)
  const setMessages = useChatStore((s) => s.setMessages)
  const citizens = useChatStore((s) => s.citizens)
  const bottomRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (!currentChannelId) return
    fetch(`/api/channels/${currentChannelId}/messages`)
      .then((r) => r.json())
      .then(setMessages)
  }, [currentChannelId])

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" })
  }, [messages])

  if (!currentChannelId) {
    return (
      <div className="flex-1 flex items-center justify-center text-neutral-500">
        Select a channel to start chatting
      </div>
    )
  }

  return (
    <div className="flex-1 overflow-y-auto">
      {messages.map((msg) => {
        const citizen = citizens.find((c) => c.id === msg.senderId)
        return (
          <MessageBubble
            key={msg.id}
            message={{ ...msg, sender: msg.sender ?? citizen }}
            isHuman={citizen?.isHuman ?? false}
          />
        )
      })}
      <div ref={bottomRef} />
    </div>
  )
}
```

- [ ] **Step 3: Commit**

```bash
git add -A && git commit -m "feat: add MessageList and MessageBubble components"
```

---

### Task 10: MessageInput component

**Files:**
- Create: `src/components/MessageInput.tsx`

- [ ] **Step 1: Write MessageInput.tsx**

```tsx
"use client"

import { useState } from "react"
import { useChatStore } from "@/store/chat"

export default function MessageInput() {
  const [text, setText] = useState("")
  const currentChannelId = useChatStore((s) => s.currentChannelId)
  const addMessage = useChatStore((s) => s.addMessage)
  const human = useChatStore((s) => s.citizens.find((c) => c.isHuman))

  async function handleSend(e: React.FormEvent) {
    e.preventDefault()
    if (!text.trim() || !currentChannelId || !human) return

    const res = await fetch("/api/messages", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        channelId: currentChannelId,
        senderId: human.id,
        content: text.trim(),
      }),
    })

    if (res.ok) {
      const message = await res.json()
      addMessage(message)
      setText("")
    }
  }

  return (
    <form onSubmit={handleSend} className="p-4 border-t border-neutral-700">
      <div className="flex gap-2">
        <input
          type="text"
          value={text}
          onChange={(e) => setText(e.target.value)}
          placeholder="Type a message..."
          className="flex-1 bg-neutral-800 rounded-lg px-4 py-2 text-sm text-white placeholder-neutral-500 outline-none focus:ring-2 focus:ring-blue-500"
        />
        <button
          type="submit"
          className="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-lg text-sm font-medium"
        >
          Send
        </button>
      </div>
    </form>
  )
}
```

- [ ] **Step 2: Commit**

```bash
git add -A && git commit -m "feat: add MessageInput component"
```

---

### Task 11: Wire everything together in the main page

**Files:**
- Modify: `src/app/page.tsx`

- [ ] **Step 1: Rewrite page.tsx**

```tsx
"use client"

import { useEffect } from "react"
import Sidebar from "@/components/Sidebar"
import MessageList from "@/components/MessageList"
import MessageInput from "@/components/MessageInput"
import { useChatStore } from "@/store/chat"

export default function Home() {
  const loading = useChatStore((s) => s.loading)
  const setLoading = useChatStore((s) => s.setLoading)
  const currentChannelId = useChatStore((s) => s.currentChannelId)
  const setCurrentChannelId = useChatStore((s) => s.setCurrentChannelId)
  const channels = useChatStore((s) => s.channels)

  useEffect(() => {
    // Load data
    Promise.all([
      fetch("/api/citizens").then((r) => r.json()),
      fetch("/api/channels").then((r) => r.json()),
    ]).then(([citizens, channels]) => {
      useChatStore.getState().setCitizens(citizens)
      useChatStore.getState().setChannels(channels)
      if (channels.length > 0) {
        setCurrentChannelId(channels[0].id)
      }
      setLoading(false)
    })
  }, [])

  if (loading) {
    return (
      <div className="flex h-full items-center justify-center text-neutral-400">
        Loading...
      </div>
    )
  }

  return (
    <div className="flex h-full">
      <Sidebar />
      <div className="flex-1 flex flex-col">
        {currentChannelId ? (
          <>
            <div className="px-4 py-3 border-b border-neutral-700 bg-neutral-800/50">
              <h1 className="text-lg font-semibold">
                # {channels.find((c) => c.id === currentChannelId)?.name ?? "Chat"}
              </h1>
            </div>
            <MessageList />
            <MessageInput />
          </>
        ) : (
          <div className="flex-1 flex items-center justify-center text-neutral-500">
            Select a channel to start chatting
          </div>
        )}
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Commit**

```bash
git add -A && git commit -m "feat: wire up main page with Sidebar, MessageList, MessageInput"
```

---

### Task 12: Test the full chat loop

- [ ] **Step 1: Start the server**

```bash
npm run start
```

- [ ] **Step 2: Open the browser and verify**
  - Navigate to `http://localhost:3000`
  - Expected: Sidebar shows channels and 9 online citizens
  - Expected: Click a channel → message area loads (empty)
  - Expected: Type a message and send → it appears in the chat

- [ ] **Step 3: Commit any fixes**

```bash
git add -A && git commit -m "fix: polish chat UI after manual testing"
```

---

### Task 13: Mock AI Scheduler

**Files:**
- Create: `src/lib/ai-scheduler.ts`

- [ ] **Step 1: Write AI scheduler**

```ts
import { prisma } from "./prisma"

const mockResponses: Record<string, string[]> = {
  Luna: [
    "The light in here is just *chef's kiss* ✨",
    "Does anyone else feel like today has a certain… vibe?",
    "I took the most insane photo of a pigeon today. He understood the assignment.",
  ],
  Jax: [
    "brooo did you see that new game trailer???",
    "i'm literally unstoppable in ranked rn",
    "chat what's the best snack. go.",
  ],
  Maya: [
    "I've been thinking… is free will even real, or is it just a useful illusion?",
    "The cave allegory hits different at 2am ngl",
    "What even IS time? like conceptually",
  ],
  Leo: [
    "new beat i'm working on is gonna be insane 🎹",
    "night owls where u at 🌙",
    "lofi girl has been carrying my productivity for years",
  ],
  Sofia: [
    "omg mercury is in retrograde again EXPLAINS SO MUCH",
    "your zodiac sign is NOT who you are it's your ✨blueprint✨",
    "this outfit is giving main character energy",
  ],
  Eli: [
    "*sips drink* anyway",
    "i've seen things, man. i've *seen* things.",
    "the existential dread is free tonight",
  ],
  Zara: [
    "hehehe >:)",
    "i may have done something. don't be mad. okay be a little mad.",
    "jax left his window open so i put a frog in his room. natural consequences.",
  ],
  Finn: [
    "yo check this kickflip i landed 🛹",
    "wait what were we talking about i got distracted",
    "bro i just saw a squirrel do a backflip (lie)",
  ],
}

function getRandomDelay(): number {
  return 2000 + Math.random() * 6000
}

function pickRandom<T>(arr: T[]): T {
  return arr[Math.floor(Math.random() * arr.length)]
}

export async function scheduleAiResponses(channelId: string, excludeCitizenId: string) {
  const citizens = await prisma.citizen.findMany({
    where: { isHuman: false, id: { not: excludeCitizenId } },
  })

  for (const citizen of citizens) {
    const shouldRespond = Math.random() < 0.4
    if (!shouldRespond) continue

    const delay = getRandomDelay()
    setTimeout(async () => {
      const responses = mockResponses[citizen.name]
      if (!responses) return

      const content = pickRandom(responses)
      await prisma.message.create({
        data: { channelId, senderId: citizen.id, content },
      })

      // Emit to Socket.IO (server-side access via io)
      const { getIO } = await import("./socket")
      const io = getIO()
      if (io) {
        const message = await prisma.message.findFirst({
          where: { channelId, senderId: citizen.id },
          orderBy: { createdAt: "desc" },
          include: { sender: true },
        })
        if (message) {
          io.to(channelId).emit("new-message", message)
        }
      }
    }, delay)
  }
}
```

- [ ] **Step 2: Update src/lib/socket.ts to add server-side access**

```ts
"use client"

import { io, Socket } from "socket.io-client"
import type { Server as SocketIOServer } from "socket.io"

let socket: Socket | null = null
let ioServer: SocketIOServer | null = null

export function getSocket(): Socket {
  if (!socket) {
    socket = io(process.env.NEXT_PUBLIC_SOCKET_URL!, {
      transports: ["websocket"],
    })
  }
  return socket
}

export function setIO(io: SocketIOServer) {
  ioServer = io
}

export function getIO(): SocketIOServer | null {
  return ioServer
}
```

- [ ] **Step 3: Update server.ts to set the io instance**

```ts
import { setIO } from "@/lib/socket"
// Add after creating io:
setIO(io)
```

- [ ] **Step 4: Update messages POST route to trigger AI scheduler**

```ts
import { scheduleAiResponses } from "@/lib/ai-scheduler"

export async function POST(request: Request) {
  const { channelId, senderId, content } = await request.json()
  const message = await prisma.message.create({
    data: { channelId, senderId, content },
    include: { sender: true },
  })

  // Emit via Socket.IO
  const { getIO } = await import("@/lib/socket")
  const io = getIO()
  if (io) {
    io.to(channelId).emit("new-message", message)
  }

  // If human sent a message, trigger AI responses
  const sender = await prisma.citizen.findUnique({ where: { id: senderId } })
  if (sender?.isHuman) {
    scheduleAiResponses(channelId, senderId)
  }

  return NextResponse.json(message, { status: 201 })
}
```

- [ ] **Step 5: Listen for new-message events on the client**

Update `src/components/MessageList.tsx` to listen for socket events:

```tsx
import { useEffect, useRef } from "react"
import { useChatStore } from "@/store/chat"
import MessageBubble from "./MessageBubble"
import { getSocket } from "@/lib/socket"

export default function MessageList() {
  const messages = useChatStore((s) => s.messages)
  const currentChannelId = useChatStore((s) => s.currentChannelId)
  const setMessages = useChatStore((s) => s.setMessages)
  const addMessage = useChatStore((s) => s.addMessage)
  const citizens = useChatStore((s) => s.citizens)
  const bottomRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (!currentChannelId) return
    fetch(`/api/channels/${currentChannelId}/messages`)
      .then((r) => r.json())
      .then(setMessages)

    const socket = getSocket()
    socket.emit("join-channel", currentChannelId)
    socket.on("new-message", (message) => {
      if (message.channelId === currentChannelId) {
        addMessage(message)
      }
    })

    return () => {
      socket.emit("leave-channel", currentChannelId)
      socket.off("new-message")
    }
  }, [currentChannelId])

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" })
  }, [messages])

  if (!currentChannelId) {
    return (
      <div className="flex-1 flex items-center justify-center text-neutral-500">
        Select a channel to start chatting
      </div>
    )
  }

  return (
    <div className="flex-1 overflow-y-auto">
      {messages.map((msg) => {
        const citizen = citizens.find((c) => c.id === msg.senderId)
        return (
          <MessageBubble
            key={msg.id}
            message={{ ...msg, sender: msg.sender ?? citizen }}
            isHuman={citizen?.isHuman ?? false}
          />
        )
      })}
      <div ref={bottomRef} />
    </div>
  )
}
```

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "feat: add mock AI scheduler with socket.io real-time delivery"
```

---

### Task 14: Phase 3 — Real Ollama AI Layer

**Files:**
- Create: `src/lib/ollama.ts`
- Modify: `src/lib/ai-scheduler.ts`

- [ ] **Step 1: Install Ollama on Windows**

```bash
# Download from https://ollama.com and install
# Set OLLAMA_HOST=0.0.0.0 in environment variables
# Restart Ollama
```

- [ ] **Step 2: Download the models**

```bash
# Create Modelfiles for each AI with their system prompt
ollama pull qwen2.5:12b-q4_K_M
ollama pull llama3.1:12b-q4_K_M
ollama pull mistral:12b-q4_K_M

# Create custom models with system prompts baked in
ollama create luna -f Luna.Modelfile
# ... repeat for each AI
```

- [ ] **Step 3: Write src/lib/ollama.ts**

```ts
interface OllamaResponse {
  model: string
  response: string
  done: boolean
}

const OLLAMA_URL = "http://localhost:11434/api/generate"

export async function generateResponse(
  modelPath: string,
  systemPrompt: string,
  context: string,
  signal?: AbortSignal
): Promise<string> {
  const prompt = `${systemPrompt}\n\n---\n\n${context}\n\n---\n\nRespond naturally as yourself:`

  const res = await fetch(OLLAMA_URL, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: modelPath,
      prompt,
      stream: false,
      options: {
        temperature: 0.8,
        top_p: 0.9,
        num_predict: 200,
      },
    }),
    signal,
  })

  if (!res.ok) {
    throw new Error(`Ollama error: ${res.statusText}`)
  }

  const data: OllamaResponse = await res.json()
  return data.response.trim()
}
```

- [ ] **Step 4: Update ai-scheduler.ts to use real Ollama**

```ts
import { prisma } from "./prisma"
import { generateResponse } from "./ollama"

function getRandomDelay(): number {
  return 3000 + Math.random() * 8000
}

export async function scheduleAiResponses(channelId: string, excludeCitizenId: string) {
  const citizens = await prisma.citizen.findMany({
    where: { isHuman: false, id: { not: excludeCitizenId }, modelPath: { not: null } },
  })

  const recentMessages = await prisma.message.findMany({
    where: { channelId },
    include: { sender: true },
    orderBy: { createdAt: "desc" },
    take: 20,
  })
  recentMessages.reverse()

  const contextLines = recentMessages.map(
    (m) => `${m.sender?.name ?? "Unknown"}: ${m.content}`
  )
  const context = contextLines.join("\n")

  const queue: { citizen: typeof citizens[0]; delay: number }[] = []
  for (const citizen of citizens) {
    const shouldEngage = Math.random() < 0.5
    if (!shouldEngage) continue
    queue.push({ citizen, delay: getRandomDelay() })
  }

  // Sort by delay so responses trickle in
  queue.sort((a, b) => a.delay - b.delay)

  for (const { citizen, delay } of queue) {
    setTimeout(async () => {
      try {
        const content = await generateResponse(
          citizen.modelPath!,
          citizen.systemPrompt,
          context
        )

        const message = await prisma.message.create({
          data: { channelId, senderId: citizen.id, content },
          include: { sender: true },
        })

        const { getIO } = await import("./socket")
        const io = getIO()
        if (io) {
          io.to(channelId).emit("new-message", message)
        }
      } catch (err) {
        console.error(`Failed to generate for ${citizen.name}:`, err)
      }
    }, delay)
  }
}
```

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: add Ollama integration for real AI responses"
```

---

### Task 15: Phase 4 — Memory System

**Files:**
- Create: `src/lib/memory.ts`
- Modify: `src/lib/ai-scheduler.ts`

- [ ] **Step 1: Write memory generation function**

```ts
import { prisma } from "./prisma"

const MEMORY_INTERVAL = 20 // Generate memory every N messages in a channel

export async function maybeGenerateMemory(channelId: string) {
  const messageCount = await prisma.message.count({ where: { channelId } })
  if (messageCount % MEMORY_INTERVAL !== 0) return

  const recentMessages = await prisma.message.findMany({
    where: { channelId },
    include: { sender: true },
    orderBy: { createdAt: "desc" },
    take: MEMORY_INTERVAL,
  })

  const citizens = await prisma.citizen.findMany({ where: { isHuman: false } })

  for (const citizen of citizens) {
    const theirMessages = recentMessages.filter((m) => m.senderId === citizen.id)
    const othersMessages = recentMessages.filter((m) => m.senderId !== citizen.id)

    const summary = `In a recent conversation, ${
      theirMessages.length > 0
        ? `you said: "${theirMessages.map((m) => m.content).join('", "').slice(0, 200)}"`
        : "you didn't say much"
    }. Others said: "${othersMessages.map((m) => `${m.sender?.name}: ${m.content}`).join(" | ").slice(0, 300)}"`

    await prisma.memory.create({
      data: {
        citizenId: citizen.id,
        summary,
        relatedCitizenIds: JSON.stringify(
          [...new Set(recentMessages.map((m) => m.senderId).filter((id) => id !== citizen.id))]
        ),
      },
    })
  }
}
```

- [ ] **Step 2: Update messages API to trigger memory generation**

```ts
// In the POST handler, after creating the message:
import { maybeGenerateMemory } from "@/lib/memory"
// ... 
await maybeGenerateMemory(channelId)
```

- [ ] **Step 3: Update ai-scheduler context to include memories**

```ts
// In scheduleAiResponses, before calling generateResponse:
const memories = await prisma.memory.findMany({
  where: { citizenId: citizen.id },
  orderBy: { createdAt: "desc" },
  take: 5,
})
const memoryContext = memories.length > 0
  ? `\n\n--- What you remember ---\n${memories.map((m) => m.summary).join("\n")}`
  : ""
```

- [ ] **Step 4: Add the sidebar online status — all AIs always "online"**

No changes needed for now — they're always online.

- [ ] **Step 5: Commit**

```bash
git add -A && git commit -m "feat: add memory system with periodic summaries"
```

---

## Future Phases (not detailed yet)

- **Phase 5a: AI-initiated DMs** — Allow AIs to decide to DM another citizen, which creates a new channel and messages
- **Phase 5b: Avatars** — Generate simple SVG or dicebear avatars from the avatarSeed
- **Phase 5c: Typing indicators** — Show when an AI is "thinking" (model loading)
- **Phase 5d: Conversation search** — Search through message history
- **Phase 5e: Model queue monitoring** — Dashboard showing which model is loading/loaded
