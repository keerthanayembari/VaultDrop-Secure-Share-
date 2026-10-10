https://expiring-file-box.lovable.app
{
  "name": "expiring-file-box",
  "version": "1.0.0",
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start"
  },
  "dependencies": {
    "next": "14.2.5",
    "react": "18.3.1",
    "react-dom": "18.3.1",
    "uuid": "^10.0.0"
  },
  "devDependencies": {
    "@types/uuid": "^10.0.0",
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0",
    "@types/react": "^18.0.0",
    "tailwindcss": "^3.4.0",
    "autoprefixer": "^10.4.0",
    "postcss": "^8.4.0"
  }
}
"use client"
import { useState } from "react"

export default function Home() {
  const [file, setFile] = useState<File | null>(null)
  const [expiry, setExpiry] = useState("24h")
  const [link, setLink] = useState("")
  const [loading, setLoading] = useState(false)

  const handleUpload = async () => {
    if (!file) return
    setLoading(true)
    const fd = new FormData()
    fd.append("file", file)
    fd.append("expiry", expiry)
    const res = await fetch("/api/upload", { method: "POST", body: fd })
    const data = await res.json()
    setLink(`${window.location.origin}/f/${data.id}`)
    setLoading(false)
  }

  return (
    <main className="min-h-screen bg-[#fafafa] flex items-center justify-center p-4">
      <div className="w-full max-w-[520px] bg-white rounded-[24px] shadow-[0_20px_60px_rgba(0,0,0,0.08)] p-8 border">
        <h1 className="text-3xl font-black tracking-tight">Expiring File Box 📦</h1>
        <p className="text-zinc-500 mt-2">Upload, share, self-destructs automatically.</p>

        <div className="mt-8 border-2 border-dashed rounded-2xl p-6 bg-zinc-50">
          <input type="file" onChange={e=>setFile(e.target.files?.[0]||null)} className="w-full text-sm" />

          <div className="flex gap-2 mt-4">
            {[
              {k:"1h", l:"1 Hour"},
              {k:"24h", l:"24 Hours"},
              {k:"7d", l:"7 Days"},
              {k:"1dl", l:"After 1 Download"},
            ].map(o=>(
              <button key={o.k} onClick={()=>setExpiry(o.k)}
                className={`px-3 py-2 rounded-full text-xs font-semibold border ${expiry===o.k? "bg-black text-white border-black" : "bg-white"}`}>
                {o.l}
              </button>
            ))}
          </div>

          <button onClick={handleUpload} disabled={!file || loading}
            className="w-full mt-6 bg-black text-white py-3 rounded-xl font-bold disabled:opacity-50">
            {loading? "Uploading..." : "Create Expiring Link"}
          </button>
        </div>

        {link && (
          <div className="mt-6 bg-green-50 border border-green-200 p-4 rounded-xl">
            <p className="text-xs text-green-700 font-semibold">YOUR SECURE LINK</p>
            <a href={link} className="text-sm break-all font-mono underline">{link}</a>
            <p className="text-xs mt-2 text-zinc-500">This file will be auto-deleted after {expiry}. No one can recover it.</p>
            <button onClick={()=>navigator.clipboard.writeText(link)} className="mt-3 text-xs bg-black text-white px-3 py-1.5 rounded-full">Copy Link</button>
          </div>
        )}
      </div>
    </main>
  )
}
import { NextRequest, NextResponse } from "next/server"
import { v4 as uuid } from "uuid"
import fs from "fs"
import path from "path"

const UPLOAD_DIR = path.join(process.cwd(), "uploads")
const DB_PATH = path.join(process.cwd(), "uploads/db.json")

if (!fs.existsSync(UPLOAD_DIR)) fs.mkdirSync(UPLOAD_DIR, { recursive: true })
if (!fs.existsSync(DB_PATH)) fs.writeFileSync(DB_PATH, "{}")

export async function POST(req: NextRequest) {
  const form = await req.formData()
  const file = form.get("file") as File
  const expiry = form.get("expiry") as string

  const id = uuid().slice(0, 8)
  const bytes = await file.arrayBuffer()
  const filePath = path.join(UPLOAD_DIR, id + "_" + file.name)
  fs.writeFileSync(filePath, Buffer.from(bytes))

  const expiryMs: any = { "1h": 60*60*1000, "24h": 24*60*60*1000, "7d": 7*24*60*60*1000 }

  const db = JSON.parse(fs.readFileSync(DB_PATH, "utf-8"))
  db[id] = {
    path: filePath,
    name: file.name,
    expiresAt: expiryMs[expiry]? Date.now() + expiryMs[expiry] : null,
    oneTime: expiry === "1dl",
    createdAt: Date.now()
  }
  fs.writeFileSync(DB_PATH, JSON.stringify(db, null, 2))

  return NextResponse.json({ id })
}
import { NextResponse } from "next/server"
import fs from "fs"
import path from "path"
const DB_PATH = path.join(process.cwd(), "uploads/db.json")

export async function GET() {
  const db = JSON.parse(fs.readFileSync(DB_PATH, "utf-8"))
  let deleted = 0
  for (const [id, f] of Object.entries(db as any)) {
    if ((f as any).expiresAt && Date.now() > (f as any).expiresAt) {
      try { fs.unlinkSync((f as any).path) } catch {}
      delete db[id]
      deleted++
    }
  }
  fs.writeFileSync(DB_PATH, JSON.stringify(db, null, 2))
  return NextResponse.json({ deleted })
}
import { NextResponse } from "next/server"
import fs from "fs"
import path from "path"
const DB_PATH = path.join(process.cwd(), "uploads/db.json")

export async function GET() {
  const db = JSON.parse(fs.readFileSync(DB_PATH, "utf-8"))
  let deleted = 0
  for (const [id, f] of Object.entries(db as any)) {
    if ((f as any).expiresAt && Date.now() > (f as any).expiresAt) {
      try { fs.unlinkSync((f as any).path) } catch {}
      delete db[id]
      deleted++
    }
  }
  fs.writeFileSync(DB_PATH, JSON.stringify(db, null, 2))
  return NextResponse.json({ deleted })
}
{
  "crons": [{ "path": "/api/cleanup", "schedule": "*/10 * * * *" }]
}
