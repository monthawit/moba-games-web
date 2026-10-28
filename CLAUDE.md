# CLAUDE.md

คำแนะนำสำหรับ Claude Code (และผู้ร่วมพัฒนา) เมื่อทำงานในโปรเจกต์นี้

## ภาพรวมโปรเจกต์

**moba-games-web** คือเกม MOBA (Multiplayer Online Battle Arena) ที่เล่นผ่านเว็บเบราว์เซอร์
โดยอ้างอิงสไตล์การเล่นและมุมกล้องจาก **Dota 2**:

- มุมกล้องแบบ **isometric/top-down คงที่** (fixed camera angle มองจากด้านบนเฉียงลงมา ไม่ใช่
  third-person/first-person ตาม character) ควบคุมการเคลื่อนที่ด้วยการคลิก (click-to-move) และ
  สกิลด้วยปุ่มลัด (QWER + คลิกตำแหน่ง/เป้าหมาย) เหมือน Dota 2
- แผนที่แบบ 2 ฝั่ง (เช่น Radiant/Dire concept) มี lane, jungle, tower, base ตามแนวทาง MOBA
  มาตรฐาน — รายละเอียดแผนที่ (จำนวน lane, จำนวนทาวเวอร์ ฯลฯ) จะ design เพิ่มใน Phase ถัดไป
  ไม่ fix ไว้ล่วงหน้าในเอกสารนี้

โปรเจกต์นี้ยังอยู่ในช่วงเริ่มต้น (มีแค่ README.md) — เอกสารนี้กำหนด scope, สถาปัตยกรรม และ
phase การพัฒนาไว้ล่วงหน้าเพื่อให้การพัฒนาต่อเนื่องกันสอดคล้องกัน

## Scope ของ repo นี้

Repo นี้ครอบคลุม**ทั้งเกมและระบบรอบเกม**:
- Game client (เล่นในเบราว์เซอร์ ไม่ต้องติดตั้งอะไรเพิ่ม)
- Game server (คำนวณ game state, sync ผู้เล่นแบบ real-time)
- ระบบ account: สมัคร/login, โปรไฟล์, สถิติการเล่น
- ระบบ admin สำหรับปรับ balance ฮีโร่ (buff/nerf) โดยไม่ต้อง deploy ใหม่ทุกครั้ง

**ไม่อยู่ใน scope** (จนกว่าจะมีการระบุเพิ่ม): mobile native app, matchmaking แบบ ranked/MMR
ที่ซับซ้อน, in-game purchase/economy, replay system, anti-cheat ขั้นสูง, voice chat — ให้ถือเป็น
Phase หลัง ๆ หรือนอกขอบเขตจนกว่าผู้ใช้จะระบุ

## Core Game Design

### Hero roster — 4 สาย (เริ่มต้น)

เริ่ม Phase แรกด้วย **4 ฮีโร่ ฮีโร่ละ 1 สาย** ครอบคลุม 4 attribute หลักตามแนวทาง Dota 2:

| สาย (Attribute) | บทบาท (concept) | หมายเหตุ |
|---|---|---|
| **Strength (Str)** | Tanky melee / แนวหน้า, HP สูง | scale ด้วย STR → HP/HP regen |
| **Agility (Agi)** | DPS/carry ระยะประชิดหรือไกล, attack speed สูง | scale ด้วย AGI → Armor/Attack Speed |
| **Intelligence (Int)** | Caster/นักเวทย์, ใช้ mana ปั๊มสกิล | scale ด้วย INT → Mana/Mana regen |
| **Universal** | ฮีโร่ที่ scale จากทุก attribute รวมกัน (ตามระบบ Universal attribute ของ Dota 2 ยุคใหม่) | ออกแบบให้ยืดหยุ่น เล่นได้หลายบทบาท |

- แต่ละฮีโร่มี: base stats (HP, Mana, Armor, Attack Damage, Attack Speed, Movement Speed),
  stat growth ต่อเลเวล, และ **สกิล 4 ช่อง (Q/W/E + Ultimate R)** ตามแบบ Dota 2
- เพิ่มฮีโร่ใหม่ในอนาคตได้เรื่อย ๆ หลัง Phase แรกเสถียรแล้ว — **อย่าออกแบบระบบฮีโร่แบบ
  hardcode ต่อตัว** ให้ยึดแนวทาง **data-driven** (hero stats/skill เก็บเป็น config/JSON หรือ
  DB record ไม่ใช่ branch โค้ดแยกราย hero) เพื่อให้ admin ปรับ balance และเพิ่มฮีโร่ใหม่ได้
  โดยไม่ต้องแก้โค้ด core

### Admin — ปรับ Balance (Buff/Nerf) ได้ตลอดเวลา

- หน้า Admin Panel (เฉพาะ role admin) ให้แก้ค่าของฮีโร่แบบ **real-time โดยไม่ต้อง deploy ใหม่**:
  base stats, stat growth/level, ค่าสกิล (damage, cooldown, mana cost, cast range, duration ฯลฯ)
- การแก้ไขต้อง **มีผลกับเกมที่กำลังจะเริ่มใหม่ทันที** (balance data โหลดจาก DB/config service
  ตอนเริ่มแมตช์ ไม่ hardcode/compile เข้า client หรือ server binary)
- เก็บ **ประวัติการแก้ไข (change history/version)** ของ balance ไว้เสมอ — ใครแก้ ตอนไหน
  แก้ field ไหนจากค่าอะไรเป็นค่าอะไร (คล้าย audit log) เผื่อต้อง rollback หรือตรวจสอบ
- Balance data เป็นแหล่งเดียว (single source of truth) ที่ทั้ง client (แสดงค่าสกิล/tooltip)
  และ server (คำนวณ game logic จริง) ดึงจากที่เดียวกัน เพื่อไม่ให้ client/server ค่าไม่ตรงกัน

### ระบบ Username / Profile / สถิติ / Login

- สมัครสมาชิกด้วย username + password (hash ด้วย bcrypt/argon2), ไม่เก็บ plaintext
- Login ด้วย JWT หรือ session cookie httpOnly + refresh token
- หน้า Profile แสดง: ฮีโร่ที่เล่น, สถิติสะสม (จำนวนแมตช์, แพ้/ชนะ, K/D/A รวม, เวลาเล่นรวม),
  สถิติต่อฮีโร่ (เผื่อ Phase หลังอยากทำ per-hero stats หน้าแยก)
- ทุกแมตช์ที่เล่นจบ ต้องบันทึกผลลง DB ผูกกับ user ไว้เสมอ (win/loss, K/D/A, ฮีโร่ที่เล่น,
  เวลาเริ่ม/จบแมตช์) เพื่อใช้คำนวณสถิติหน้า Profile

## หลักการออกแบบสถาปัตยกรรม

### แยก 3 ส่วนหลัก: Game Client, Game Server, Platform API

- **Game Client** (เบราว์เซอร์): render เกม, รับ input ผู้เล่น, ส่ง action ไปที่ Game Server,
  แสดงผลจาก game state ที่ sync กลับมา — **ห้ามคำนวณผลลัพธ์เกมฝั่ง client เป็น authority**
  (เช่น ความเสียหาย, การตาย, cooldown จริง) ต้องให้ server เป็นคนตัดสิน (server-authoritative)
  เพื่อป้องกันการโกง — client ทำได้แค่ prediction/interpolation เพื่อความลื่นไหล
- **Game Server**: รับผิดชอบ real-time game loop ต่อแมตช์ (tick-based หรือ loop แบบที่เหมาะกับ
  real-time multiplayer), คำนวณ physics/combat/skill logic ทั้งหมด, sync state ให้ผู้เล่นทุกคน
  ในแมตช์ผ่าน WebSocket (หรือ WebRTC ถ้าต้องการ latency ต่ำกว่าในอนาคต — เริ่มด้วย WebSocket
  ก่อนเพราะ implement ง่ายกว่าและพอสำหรับ Phase แรก)
- **Platform API** (REST): auth, user/profile, สถิติ, matchmaking คิวเข้าเกม, admin balance
  panel — เป็นคนละ service จาก Game Server (แยก concern: ระบบรอบเกม vs ตัวเกมสด ๆ) แต่เรียก
  ใช้ DB ร่วมกันได้ (หรือแยก DB ก็ได้ ขึ้นกับ Phase ถัดไป)

### แนะนำ stack (ปรับได้ตามความถนัดทีม แต่ให้ยึดแนวทางนี้เป็นฐาน)
- **Game Client**: TypeScript + **HTML5 Canvas/WebGL renderer** ผ่าน game engine สำเร็จรูปที่
  เหมาะกับ 2.5D/isometric real-time multiplayer บนเว็บ เช่น **Phaser 3** (2D, เรียบง่าย เริ่มเร็ว)
  หรือ **PixiJS + ของทำ custom scene graph เอง** ถ้าต้องการคุมเองมากกว่า — เลือกอย่างใดอย่างหนึ่ง
  แล้วคงเส้นคงวา ไม่สลับ engine กลางทาง
- **Game Server**: Node.js/TypeScript (จัดการ WebSocket connection ง่าย, ecosystem เดียวกับ
  client ลด context switch) หรือ Go ถ้าต้องการ performance สูงกว่าสำหรับ game loop จำนวนมาก —
  เริ่ม Node.js/TypeScript ก่อนสำหรับ Phase แรกเพื่อความเร็วในการพัฒนา แล้วประเมินย้ายเป็น Go
  ถ้า profiling พบว่า Node เป็นคอขวดจริง
- **Platform API**: Node.js/TypeScript (NestJS หรือ Express) ใช้ ecosystem เดียวกับ Game Server
- **Realtime transport**: WebSocket (เช่น `socket.io` หรือ `ws` ล้วน ๆ)
- **Database**: PostgreSQL (users, profiles, match history, hero/skill balance data, audit log
  ของการแก้ balance)
- **Cache/Session/Matchmaking queue**: Redis (เก็บ session, สถานะคิวจับคู่, pub/sub ระหว่าง
  instance ของ Game Server ถ้า scale หลาย instance)
- **Frontend (เว็บรอบเกม — login, profile, lobby, admin panel)**: React + TypeScript แยกจาก
  game client renderer (คนละหน้า/คนละ bundle แต่ใช้ React component wrap game canvas ได้ตอน
  เข้าสู่หน้าเล่นเกมจริง)

## Feature Requirements (สรุปจาก requirement เดิม, สะสมเพิ่มเรื่อย ๆ)

> หัวข้อนี้คือ requirement สะสม — เมื่อได้รับ feature request ใหม่ ให้ **เพิ่มเป็นข้อถัดไปในลิสต์
> นี้** (อย่าลบของเดิม เว้นแต่ผู้ใช้บอกให้ลบ/แทนที่) แล้วอัปเดต Development Phases /
> "สถานะปัจจุบัน" ให้สะท้อนว่า feature ไหน implement แล้วบ้าง

1. **Gameplay core (Dota 2-style)**: มุมกล้อง isometric คงที่, click-to-move, สกิล 4 ช่อง
   (Q/W/E/R) ต่อฮีโร่, แผนที่ 2 ฝั่งพร้อม lane/tower/base พื้นฐาน, combat ระบบ server-authoritative
2. **Hero roster**: เริ่มต้น 4 ฮีโร่ (Str/Agi/Int/Universal อย่างละ 1), data-driven design
   เพื่อเพิ่มฮีโร่ใหม่และปรับ balance ได้โดยไม่แก้โค้ด core
3. **Admin Balance Panel**: แก้ stats/skill ของฮีโร่แบบ real-time, มี version/change history,
   มีผลกับแมตช์ใหม่ทันทีโดยไม่ต้อง deploy
4. **Account system**: สมัคร/login (JWT หรือ session httpOnly), password hash ด้วย
   bcrypt/argon2
5. **Profile & สถิติ**: หน้าโปรไฟล์แสดงสถิติสะสมและต่อฮีโร่ (แมตช์, แพ้/ชนะ, K/D/A, เวลาเล่น)
6. **Deployment target**: Deploy บน **private Kubernetes cluster ที่มี IP Public ออก internet
   ได้** และใช้ **Cloudflare เป็นผู้จัดการ DNS/Domain** (เช่าโดเมนผ่าน Cloudflare Registrar
   หรือเพิ่ม nameserver ของโดเมนที่มีอยู่แล้วเข้า Cloudflare) — ดูหัวข้อ Deployment Target
   ด้านล่าง

## Deployment Target

- Environment ที่รันจริง: **private Kubernetes cluster ของผู้ใช้เอง** (ไม่ใช่ managed K8s อย่าง
  EKS/GKE/AKS) ที่มี **IP Public ออก internet ได้โดยตรง** (ไม่ได้อยู่หลัง NAT/บ้านล้วน ๆ)
- **Domain/DNS ผ่าน Cloudflare**: จดโดเมนหรือเพิ่มโดเมนที่มีอยู่เข้า Cloudflare แล้วตั้ง DNS record
  (A/AAAA) ชี้ไปที่ IP Public ของ cluster — เปิด Cloudflare proxy (orange cloud) ได้สำหรับ HTTP(S)
  traffic ปกติ (REST API, static asset ของ client) แต่**ต้องตรวจสอบ WebSocket ผ่าน Cloudflare
  proxy ให้ทำงานได้จริงก่อน** (Cloudflare รองรับ WebSocket ผ่าน proxy ได้ แต่มี connection timeout
  ของ idle connection — เกม real-time ต้องมี keepalive/ping เพื่อไม่ให้ connection หลุดกลางเกม)
  ถ้าต้องการ raw TCP/UDP สำหรับ transport อื่นในอนาคต (เช่น ย้ายไป WebRTC) ให้พิจารณา bypass
  proxy เฉพาะ record นั้น (grey cloud) หรือใช้ Cloudflare Spectrum
- สถาปัตยกรรมบน cluster (แนวทางเริ่มต้น):
  - **Game Client** (static build) + **Platform API** + **Admin Panel**: Deployment + Service
    ธรรมดา อยู่หลัง Ingress (TLS ผ่าน cert-manager + Let's Encrypt หรือ Cloudflare Origin
    Certificate)
  - **Game Server**: Deployment ที่รองรับ WebSocket แบบ sticky — ถ้า scale มากกว่า 1 replica
    ต้องมี **session affinity** (เช่น Ingress-NGINX `nginx.ingress.kubernetes.io/affinity: cookie`
    หรือ route ผ่าน Service โดยตรงแยกจาก Platform API) เพราะผู้เล่นในแมตช์เดียวกันต้องเชื่อมต่อ
    Game Server instance เดียวกันตลอดแมตช์ — Phase แรกเริ่มที่ **1 replica ต่อ region/cluster
    ก่อน** (ง่ายสุด ไม่ต้องจัดการ affinity/matchmaking ข้าม instance) แล้วค่อยออกแบบ
    matchmaking ข้าม instance ใน Phase หลัง
  - Namespace แยกของตัวเอง เช่น `moba-game-system`
- **Image ที่จะ push ไปใช้งานจริงต้อง build เป็น `linux/amd64`** เสมอ (เว้นแต่ cluster จริงเป็น
  arch อื่น — ยืนยัน arch ของ node ในคลัสเตอร์ก่อน build จริงถ้าไม่ชัวร์):
  ```bash
  docker buildx build --platform linux/amd64 -t <registry>/<image>:<tag> --push .
  ```
- Local dev ใช้ Docker Compose (native arch ของเครื่อง dev เพื่อความเร็ว) — แยกจาก production
  build โดยสิ้นเชิงเหมือนกับแนวทาง `kubeadm-operation-ui` ห้ามนำ image จาก compose build ไป
  deploy จริง

## Development Phases

พัฒนาแบบค่อยเป็นค่อยไป ให้เห็นภาพเล่นได้จริงเร็วที่สุด แล้วค่อยเพิ่มความลึก — **อย่ากระโดดข้าม
phase หรือ implement ของ phase หลังก่อนที่ phase ก่อนหน้าจะใช้งานได้จริง**

### Phase 0 — Foundation
- Scaffold: game client project (Phaser/PixiJS + TS), game server (WS + TS), platform API
  (REST + TS), Dockerfile ทั้งหมด, docker-compose สำหรับ dev local, README แผนผังสถาปัตยกรรม
- Auth พื้นฐาน: สมัคร/login, เก็บ user ลง PostgreSQL, password hash
- DB schema เริ่มต้น: users, heroes (data-driven), hero_skills, matches, match_participants

### Phase 1 — Playable MOBA core (MVP)
- แผนที่พื้นฐาน 1 แผนที่ (2 ฝั่ง, lane เดียวพอสำหรับ MVP, จุด spawn, 1 tower/ฝั่งพอก่อน)
- 4 ฮีโร่แรก (Str/Agi/Int/Universal) เล่นได้จริง: เคลื่อนที่แบบ click-to-move, attack ปกติ,
  สกิล Q/W/E/R ครบ, ตาย/เกิดใหม่ (respawn)
- Game loop server-authoritative, sync state ผ่าน WebSocket ให้ผู้เล่น 2+ คนเห็นกันในแมตช์เดียว
- จบแมตช์แบบง่าย (เช่น ทำลาย base ฝั่งตรงข้าม หรือจำกัดเวลา) แล้วบันทึกผลลง DB
- หน้า lobby พื้นฐาน: สร้าง/เข้าแมตช์ (ยังไม่ต้องมี matchmaking อัตโนมัติ — join ด้วย room code
  พอสำหรับ Phase นี้)
- หน้า Profile แสดงสถิติจากแมตช์ที่เล่นจริง

### Phase 2 — Admin Balance Panel
- Admin role + Admin Panel: แก้ stats/skill ของฮีโร่แบบ real-time พร้อม change history
- Game Server โหลด balance data จาก DB ตอนเริ่มแมตช์ใหม่ทุกครั้ง (ไม่ cache ค้างข้าม deploy)
- ทดสอบว่าแก้ค่าแล้วมีผลกับแมตช์ที่ **เริ่มใหม่** จริง โดยไม่ต้อง restart service

### Phase 3 — Matchmaking & More Heroes
- Matchmaking อัตโนมัติ (คิวเข้าเกม, จับคู่ผู้เล่นแบบง่าย ๆ ก่อน ยังไม่ต้องมี MMR/ranked)
- เพิ่มฮีโร่ใหม่ (ต่อจาก 4 ตัวแรก) โดยใช้ data-driven system เดิม ไม่แก้โค้ด core
- แผนที่เพิ่มความสมบูรณ์ (หลาย lane, jungle, ทาวเวอร์หลายชั้น) ตามแนวทาง MOBA เต็มรูปแบบ

### Phase 4 — Production Deployment
- Deploy ขึ้น private K8s cluster จริง, ตั้งค่า Ingress/TLS, เชื่อม Cloudflare DNS
- ทดสอบ WebSocket ผ่าน Cloudflare proxy ให้มั่นใจว่าแมตช์ไม่หลุดกลางเกม
- Monitoring พื้นฐานของ Game Server (จำนวนแมตช์ active, latency ผู้เล่น)

### Phase 5 — Hardening / Advanced
- Scale Game Server หลาย instance + session affinity หรือ matchmaking ข้าม instance
- Anti-cheat พื้นฐานฝั่ง server (validate action ที่ผิดปกติ)
- Replay system, ranked/MMR, in-game economy (ถ้าผู้ใช้ต้องการในอนาคต)

## Security Guidelines (บังคับใช้ทุก phase)

- ห้าม hardcode credential, DB password, JWT signing key ใด ๆ ในโค้ดหรือ image — ต้องมาจาก
  Kubernetes Secret / environment variable เท่านั้น
- Password ต้อง hash (bcrypt/argon2) ห้ามเก็บ plaintext
- **Game logic ต้องเป็น server-authoritative เสมอ** — ห้ามเชื่อผลคำนวณที่ client ส่งมาตรง ๆ
  (เช่น "ฉันตีโดนแล้ว ลด HP ให้หน่อย") server ต้องคำนวณ hit/damage/cooldown เองจาก state ที่ตัว
  server เก็บไว้ ไม่ใช่เชื่อค่าจาก client — ป้องกันการโกง
- Admin balance panel ต้องมี authorization check (role admin เท่านั้น) และเขียน audit log
  ทุกครั้งที่แก้ไข (แม้ action จะ fail ก็ต้อง log)
- Rate-limit endpoint auth/login และ WebSocket connection เพื่อลดความเสี่ยง brute-force/DoS
  ระดับ application
- Container image ต้องรันด้วย non-root user, ตั้ง resource limits, และใช้ ServiceAccount ที่มี
  RBAC แบบ least-privilege

## Build & Deploy Conventions

- **Production image ต้องเป็น `linux/amd64` เสมอ** (ยืนยัน arch ของ cluster จริงก่อน ถ้าไม่ชัวร์):
  ```bash
  docker buildx build --platform linux/amd64 -t <registry>/<image>:<tag> --push .
  ```
- Local dev/testing ใช้ `docker compose build`/`docker compose watch` แทน (native arch, เร็วกว่า)
  — สอง path นี้แยกกันเด็ดขาด ห้ามสลับใช้
- Manifest/Helm chart สำหรับ deploy ทั้ง 3 service (game client, game server, platform API) ให้
  เก็บไว้ใน `deploy/` หรือ `charts/`
- แยก config ของ dev/staging/prod ผ่าน env vars หรือ Helm values ไม่ hardcode ในโค้ด

## สถานะปัจจุบัน / หมายเหตุสำหรับ Claude Code

- โปรเจกต์ยังไม่มีโค้ด มีแค่ README.md — ยังไม่เริ่ม Phase 0
- ทุกครั้งที่โครงสร้างไฟล์/คำสั่งจริงเปลี่ยนไปจากที่เขียนไว้ในเอกสารนี้ ให้อัปเดตเอกสารนี้ตาม
  ด้วย (โดยเฉพาะ path, ชื่อ package, คำสั่ง build/run)
- อย่ากระโดดข้าม phase — เริ่ม Phase 0 (scaffold + auth พื้นฐาน) ก่อนเสมอ แม้ผู้ใช้จะขอฟีเจอร์ของ
  phase หลังมาก่อนก็ตาม ให้แจ้งผู้ใช้ว่าต้องทำ phase ก่อนหน้าให้เสร็จก่อน เว้นแต่ผู้ใช้ระบุ scope
  ใหม่อย่างชัดเจน
