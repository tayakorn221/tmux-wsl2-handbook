# tmux + WSL2 Handbook

คู่มืออ้างอิงส่วนตัวสำหรับการใช้ tmux บน Windows 11 + WSL2 (Ubuntu) สำหรับ workflow data analyst + freelance web dev

**Stack เป้าหมาย:** WSL2 (Ubuntu) · Claude Code · dbt Core · PostgreSQL · Next.js 14 · Docker · Power BI
**ทดสอบกับ:** Ubuntu 22.04 / 24.04, tmux 3.2a / 3.4
**Last updated:** 2026-05-23

---

## Table of Contents

1. [Section 1: Concept พื้นฐาน](#section-1-concept-พื้นฐาน)
2. [Section 2: การติดตั้ง](#section-2-การติดตั้ง)
3. [Section 3: tmux Basics](#section-3-tmux-basics)
4. [Section 4: Configuration](#section-4-configuration)
5. [Section 5: Workflow Templates](#section-5-workflow-templates)
6. [Section 6: WSL2 ↔ Windows Integration](#section-6-wsl2--windows-integration)
7. [Section 7: Cheat Sheet](#section-7-cheat-sheet)
8. [Section 8: Troubleshooting](#section-8-troubleshooting)
9. [Section 9: ขั้นต่อไป (Advanced)](#section-9-ขั้นต่อไป-advanced)

---

## Section 1: Concept พื้นฐาน

### WSL2 คืออะไร

WSL2 (Windows Subsystem for Linux v2) คือ Linux kernel จริง ที่รันใน lightweight virtual machine ของ Hyper-V บน Windows 11 ทำให้สามารถรัน distro อย่าง Ubuntu, Debian, Fedora ฯลฯ ได้แบบ native ภายใน Windows โดยไม่ต้อง dual boot และไม่ต้องเปิด VirtualBox/VMware แยก

**WSL1 vs WSL2 — ต่างกันที่:**

| ประเด็น | WSL1 | WSL2 |
| --- | --- | --- |
| สถาปัตยกรรม | แปลง syscall ของ Linux เป็น Windows NT syscall (translation layer) | Linux kernel จริงใน Hyper-V VM |
| ความเข้ากันได้ | บาง syscall ไม่รองรับ (Docker daemon รันไม่ได้) | รองรับ Linux เกือบ 100% — Docker, systemd ใช้ได้ |
| Disk I/O (ใน Linux filesystem) | ช้า | เร็วกว่า WSL1 หลายเท่า |
| Disk I/O (ข้าม `/mnt/c/`) | เร็วกว่า WSL2 | ช้ามาก (อ่าน/เขียน Windows drive ผ่าน 9P protocol) |
| RAM | ใช้ตามจริง | จองล่วงหน้า แต่ปรับ cap ได้ใน `.wslconfig` |
| Network | ใช้ network stack ของ Windows ตรงๆ | virtual NIC แยก, มี port forwarding อัตโนมัติให้ localhost |

**สรุป:** WSL2 เป็นค่า default ปัจจุบันและเป็นตัวเลือกที่ถูกต้องสำหรับ dev workflow ทั้งหมด ยกเว้นกรณีพิเศษมากๆ ที่ต้องการ I/O cross-OS เร็วๆ (เกือบไม่มีในงานจริง)

### tmux คืออะไร

tmux (terminal multiplexer) คือโปรแกรมที่ให้รัน terminal session หลายอันใน window เดียว และที่สำคัญที่สุดคือ session ทำงานต่อใน background ได้แม้ปิด terminal ไปแล้ว — กลับมา `tmux attach` ก็ได้ทุก process ที่รันอยู่กลับคืน

### tmux ต่างจาก Windows Terminal tab อย่างไร

| ประเด็น | Windows Terminal tab/pane | tmux |
| --- | --- | --- |
| Persistence | ปิด terminal = process ตายหมด | detach แล้ว process ยังรันใน background |
| Remote/SSH | tab อยู่ที่ฝั่ง Windows เท่านั้น | session อยู่ที่ฝั่ง server — SSH ขาดแล้ว reconnect กลับมาทำงานต่อได้ |
| Layout templates | ทำไม่ได้ | scriptable ผ่าน shell หรือ tmuxinator/tmuxp |
| Keybinding | จำกัด | rebind ได้ทั้งหมด |
| Cross-platform | Windows only | ทุก Unix-like (Linux/macOS/WSL) ใช้ config เดียวกันได้ |

**ใช้คู่กัน:** Windows Terminal เป็น "หน้าต่าง" Windows Terminal เปิด WSL2 → ใน WSL2 รัน tmux เพื่อจัดการ session/window/pane

### Mental Model 4 ชั้น

```text
tmux server  (1 ตัวต่อ user — daemon)
   └── session  (เหมือน "โปรเจกต์" — KMUTT, freelance, ฯลฯ)
          └── window  (เหมือน "tab" — frontend, backend, db, ฯลฯ)
                 └── pane  (เหมือน "split" — code | logs)
```

| ชั้น | ทำหน้าที่อะไร | analogy |
| --- | --- | --- |
| **server** | daemon ที่ดูแลทุก session ของคุณ — เกิดอัตโนมัติเมื่อสั่ง `tmux` ครั้งแรก ดับเมื่อทุก session หาย | บริการระบบ |
| **session** | หน่วยใหญ่สุด — ตั้งชื่อได้ (`kmutt`, `nextjs`) detach/attach อิสระต่อกัน | "โปรเจกต์" |
| **window** | เหมือน tab ใน browser — มีหมายเลข (1, 2, 3) ตั้งชื่อได้ — แต่ละ window มี layout pane เป็นของตัวเอง | "tab" |
| **pane** | terminal จริง — รัน shell หนึ่งตัว split window เป็นหลายส่วน vertical/horizontal | "split" |

### ทำไมต้องใช้ tmux + WSL2 คู่กัน

| ปัญหาที่แก้ | tmux ช่วยอะไร |
| --- | --- |
| Windows Terminal crash, Windows restart, รีสตาร์ทเครื่อง | session ใน tmux ยังอยู่ (ตราบใดที่ WSL2 ยังไม่ shutdown) |
| ต้องเปิด terminal หลาย tab สำหรับโปรเจกต์เดียว | จัด pane ใน window เดียว เห็นทุกอย่างพร้อมกัน |
| ลืมว่าทำอะไรไว้เมื่อวาน | attach กลับ ได้ทุก process + history + layout เดิม |
| Long-running job (dbt run, npm build) | detach ทิ้งไว้ — ไม่ต้องคา terminal เปิดทั้งวัน |
| ต้องโชว์งาน 2 หน้าจอ | window สำหรับ code, อีก window สำหรับ logs/db — สลับด้วย keystroke |

---

## Section 2: การติดตั้ง

### 2.1 เช็คก่อนว่ามี WSL2 อยู่หรือยัง

เปิด **PowerShell** (ไม่ต้อง admin):

```powershell
wsl -l -v
```

ผลลัพธ์ที่เป็นไปได้:

- ขึ้น list distro พร้อม `VERSION 2` → มี WSL2 แล้ว ข้ามไป 2.3
- ขึ้น list แต่ `VERSION 1` → ดู 2.5 (migrate)
- error `Windows Subsystem for Linux has no installed distributions` → ทำ 2.2
- error อื่นๆ เกี่ยวกับ feature → ดู 2.4 (troubleshoot virtualization)

### 2.2 ติดตั้ง WSL2 + Ubuntu

เปิด PowerShell แบบ **Run as Administrator** แล้ว:

```powershell
wsl --install
```

คำสั่งเดียวนี้จะ (อัตโนมัติ):

1. เปิด Windows feature `VirtualMachinePlatform` และ `Microsoft-Windows-Subsystem-Linux`
2. ตั้ง default WSL version = 2
3. download + install Ubuntu (default distro)
4. ขอ reboot — กด reboot

หลัง reboot, Ubuntu จะเปิดขึ้นมาเองและขอตั้ง username/password (UNIX account ไม่ใช่ Microsoft account)

ถ้าอยากเลือก distro อื่น:

```powershell
wsl --list --online                # ดู distro ที่ install ได้
wsl --install -d Ubuntu-24.04
```

### 2.3 ตั้ง default version + update

```powershell
wsl --set-default-version 2
wsl --update
wsl -l -v
```

ควรเห็น `STATE = Running` หรือ `Stopped` และ `VERSION = 2`

### 2.4 Troubleshoot: virtualization ไม่เปิดใน BIOS

ถ้า `wsl --install` แจ้ง error เกี่ยวกับ Hyper-V หรือ virtualization:

1. เช็คใน Task Manager → Performance → CPU → ดูบรรทัด **"Virtualization"** ต้องเป็น `Enabled`
2. ถ้า `Disabled`: รีสตาร์ทเข้า BIOS/UEFI (กด F2/Del/F10 ตอนบูต)
3. หาเมนู `Intel VT-x` / `AMD-V` / `SVM Mode` → enable → save → exit
4. กลับ Windows รัน `wsl --install` ใหม่

ถ้ายังไม่ได้ เปิด feature ด้วยมือ (PowerShell as admin):

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

แล้ว reboot

### 2.5 Migrate WSL1 → WSL2

```powershell
wsl --set-version Ubuntu 2
```

ใช้เวลา 5–30 นาทีขึ้นกับขนาด disk image รอจน `Conversion complete.` แล้วเช็คด้วย `wsl -l -v`

### 2.6 ลง tmux ใน Ubuntu

เปิด Ubuntu (พิมพ์ `ubuntu` ใน Start menu หรือเปิด profile ใน Windows Terminal):

```bash
sudo apt update
sudo apt install -y tmux
tmux -V
```

ควรเห็น `tmux 3.x.x` ขึ้นไป ถ้าเก่ากว่านี้ feature ในคู่มือนี้บางอย่างจะใช้ไม่ได้ — แต่ Ubuntu 22.04 ขึ้นไปจะได้ 3.2a เป็นค่า default ซึ่งพอใช้

### 2.7 เริ่มใช้ครั้งแรก

```bash
tmux                          # เปิด session ไม่ตั้งชื่อ
tmux new -s test              # เปิด session ชื่อ "test"
```

ถ้าเห็นแถบสถานะสีเขียวด้านล่าง = ทำงานแล้ว ออกด้วย `Ctrl+d` หรือพิมพ์ `exit`

---

## Section 3: tmux Basics

### 3.1 Prefix Key Concept

ทุก keybinding ใน tmux จะต้องกด **prefix key** ก่อน แล้วตามด้วย key สั่ง คล้ายๆ การกด leader key ใน Vim หรือ `Ctrl+x` ใน Emacs

| Prefix | ที่มา | ข้อดี | ข้อเสีย |
| --- | --- | --- | --- |
| **`Ctrl+b`** | default ของ tmux | ไม่ชน Emacs convention | กดยาก (ใช้สองมือ), ไกลจากเหย้า |
| **`Ctrl+a`** | จาก GNU Screen | กดง่าย, ergonomic | ชนกับ bash "ไปต้นบรรทัด" (รับได้, นานๆ ใช้ที) |

ในคู่มือนี้และ `tmux.conf` ที่แนบมา ใช้ **`Ctrl+a`** เป็น prefix

> **Why Ctrl+a:** กดด้วยมือซ้ายเดียว (นิ้วก้อย + นิ้วชี้) ลดอาการล้า keystroke ที่ใช้บ่อยที่สุดในชีวิตหลังลง tmux

ตลอดคู่มือนี้ `prefix` = `Ctrl+a` และเขียนเป็น `prefix + X` หมายถึงกด `Ctrl+a` แล้วปล่อย แล้วกด `X`

### 3.2 Session

| Action | Command | Keybinding ใน tmux |
| --- | --- | --- |
| สร้าง session ใหม่ (ไม่ตั้งชื่อ) | `tmux` | — |
| สร้าง session ใหม่ พร้อมตั้งชื่อ | `tmux new -s NAME` | `prefix + S` (custom) |
| Detach (ออกแต่ session ยังรัน) | — | `prefix + d` |
| List session | `tmux ls` | `prefix + s` (interactive picker) |
| Attach session ล่าสุด | `tmux attach` (= `tmux a`) | — |
| Attach session ที่ระบุ | `tmux a -t NAME` | — |
| Switch session ภายใน tmux | — | `prefix + s` แล้วเลือก |
| Rename session ปัจจุบัน | — | `prefix + $` |
| Kill session ที่ระบุ | `tmux kill-session -t NAME` | `prefix + K` (custom, มี confirm) |
| Kill **ทุก** session | `tmux kill-server` | — |

**Workflow ปกติ:**

```bash
tmux new -s kmutt             # เริ่มงาน
# ... ทำงาน ...
# กด prefix + d เพื่อ detach (session ยังรัน)
# ปิด terminal ได้เลย

# กลับมาทีหลัง:
tmux ls                       # ดูว่ามี session อะไรอยู่
tmux a -t kmutt               # attach กลับ — ทุกอย่างยังอยู่
```

### 3.3 Window

Window คือ "tab" ภายใน session — แต่ละ window มีหมายเลข (เริ่มที่ 1 ตาม config) และตั้งชื่อได้

| Action | Keybinding |
| --- | --- |
| สร้าง window ใหม่ | `prefix + c` |
| Switch ไป window N (N = 1..9) | `prefix + N` |
| Window ถัดไป | `prefix + n` |
| Window ก่อนหน้า | `prefix + p` |
| Window ล่าสุด (toggle 2 อันสลับ) | `prefix + Tab` (custom) |
| List window แบบ picker | `prefix + w` |
| Rename window ปัจจุบัน | `prefix + ,` |
| Find window ตามชื่อ | `prefix + f` |
| Kill window ปัจจุบัน | `prefix + &` (มี confirm) |
| ย้าย window ไปตำแหน่งอื่น | `prefix + .` แล้วใส่หมายเลข |

### 3.4 Pane

Pane คือการ split window เป็นหลายช่อง terminal

| Action | Keybinding |
| --- | --- |
| Split vertical (ขวา) | `prefix + |` (custom) |
| Split horizontal (ล่าง) | `prefix + -` (custom) |
| ย้ายไป pane ซ้าย | `prefix + h` (custom) |
| ย้ายไป pane ล่าง | `prefix + j` (custom) |
| ย้ายไป pane บน | `prefix + k` (custom) |
| ย้ายไป pane ขวา | `prefix + l` (custom) |
| สลับ pane ล่าสุด | `prefix + ;` |
| Zoom pane ปัจจุบัน (toggle เต็มจอ) | `prefix + z` |
| Resize: ขยายไปทางซ้าย/ล่าง/บน/ขวา 5 ช่อง | `prefix + H/J/K/L` (custom, repeatable) |
| Cycle layout (even, main-h, ฯลฯ) | `prefix + Space` |
| แสดงหมายเลข pane (ชั่วคราว) | `prefix + q` |
| Swap pane กับ pane ถัดไป | `prefix + }` |
| Convert pane เป็น window | `prefix + !` |
| Kill pane ปัจจุบัน | `prefix + X` (custom, มี confirm) |
| Kill pane ปัจจุบัน (default, ไม่มี confirm) | `prefix + x` |

> **Why split ด้วย `|` กับ `-`:** mnemonic จำง่ายมาก — `|` คือเส้นแบ่งแนวตั้ง, `-` คือเส้นแบ่งแนวนอน ต่างจาก default `%` กับ `"` ที่ไม่สื่อทิศทาง

### 3.5 Copy Mode

Copy mode คือโหมดอ่าน — เลื่อนดู scrollback, ค้นหา, คัดลอกข้อความ ออกจาก mode ปกติ

| Action | Keybinding |
| --- | --- |
| เข้า copy mode | `prefix + [` |
| ออก copy mode | `q` หรือ `Esc` |
| เลื่อนขึ้น/ลง 1 บรรทัด | `k` / `j` (vi mode) |
| เลื่อนซ้าย/ขวา 1 ตัวอักษร | `h` / `l` |
| เลื่อน page up/down | `Ctrl+u` / `Ctrl+d` |
| ไปบนสุด/ล่างสุด | `g` / `G` |
| ค้นหาไปข้างหน้า | `/PATTERN` แล้ว Enter |
| ค้นหาไปข้างหลัง | `?PATTERN` แล้ว Enter |
| ผลค้นหาถัดไป / ก่อนหน้า | `n` / `N` |
| เริ่ม selection | `v` (custom — vi visual) |
| Copy ที่เลือกไป Windows clipboard | `y` (custom — pipe ไป `clip.exe`) |
| Paste จาก tmux buffer | `prefix + ]` |

> เปิด `mouse on` แล้ว (ดู config) — scroll wheel จะพา auto-enter copy mode และเลื่อน history ได้เลย ปล่อย mouse drag = พ่นเข้า Windows clipboard อัตโนมัติ

---

## Section 4: Configuration

ดูไฟล์ [`tmux.conf`](./tmux.conf) ที่แนบมา — มี comment อธิบายทุกบรรทัด

### 4.1 ติดตั้ง config

```bash
# จาก WSL2 (สมมติ clone repo นี้มาที่ ~/tmux-wsl2-handbook/)
cp ~/tmux-wsl2-handbook/tmux.conf ~/.tmux.conf

# หรือ symlink (อัพเดต config ผ่าน git pull ได้ทันที)
ln -s ~/tmux-wsl2-handbook/tmux.conf ~/.tmux.conf
```

### 4.2 Reload config

ถ้าอยู่ใน tmux session อยู่แล้ว ไม่ต้องปิด ใช้ keybinding ที่ตั้งไว้:

```text
prefix + r
```

จะ source `~/.tmux.conf` ใหม่ + แสดง `tmux.conf reloaded` ใน status bar

ถ้าอยู่นอก tmux (หรือ binding ยังไม่ทำงานเพราะเพิ่ง install):

```bash
tmux source-file ~/.tmux.conf
```

### 4.3 สรุปสิ่งที่ config นี้เปลี่ยน

| หมวด | ค่า | เหตุผล |
| --- | --- | --- |
| Prefix | `Ctrl+a` (จาก `Ctrl+b`) | กดง่ายกว่า, มือเดียว |
| Window/pane numbering | เริ่มที่ 1 | ตรงกับลำดับปุ่ม `1..9` บนคีย์บอร์ด |
| Renumber on close | on | ไม่มีรู gap หลังลบ window |
| Mouse | on | scroll, resize, click pane ได้ |
| Scrollback | 50,000 บรรทัด | logs dbt/docker ยาวๆ ไม่หาย |
| Escape time | 10ms | Neovim ไม่ค้างหลังกด `Esc` |
| Split bindings | `|` และ `-` | mnemonic |
| Navigate panes | `h/j/k/l` | ตรงกับ Vim |
| Resize panes | `H/J/K/L` (repeat) | ปรับขนาดเร็วๆ |
| Reload | `prefix + r` | แก้ config ทดสอบทันที |
| Copy mode | vi keys, yank → `clip.exe` | คัดลอกตรงเข้า Windows clipboard |
| Default terminal | `tmux-256color` + RGB override | สีของ Neovim/bat ถูกต้อง |

---

## Section 5: Workflow Templates

ทุก template ออกแบบให้สร้าง session พร้อม pane layout ครบในคำสั่งเดียว ใช้ shell command ปกติ (ไม่ต้องลง plugin) — ก๊อปคำสั่งวางใน WSL2 ได้เลย

> ทุก keybinding ในส่วนนี้ตรงกับ [`tmux.conf`](./tmux.conf)

### 5.1 Template: KMUTT DataPlatform session

**เป้าหมาย:** 1 session, 2 window
- Window 1 `dbt`: 3 pane — Claude Code (ซ้ายเต็มความสูง) | dbt runner (ขวาบน) / psql (ขวาล่าง)
- Window 2 `logs`: 1 pane — สำหรับดู docker logs

```bash
# ปรับ path ตามจริง
PROJECT_DIR="$HOME/projects/kmutt-dataplatform"
PG_USER="postgres"
PG_DB="dataplatform"

# สร้าง session + window แรก ที่ pane เริ่มต้นรัน claude
tmux new-session -d -s kmutt -n dbt -c "$PROJECT_DIR"
tmux send-keys -t kmutt:dbt 'claude' C-m

# split ขวา (pane 2) สำหรับ dbt
tmux split-window -h -t kmutt:dbt -c "$PROJECT_DIR"
tmux send-keys -t kmutt:dbt.2 'source .venv/bin/activate && dbt debug' C-m

# split ขวาล่าง (pane 3) สำหรับ psql
tmux split-window -v -t kmutt:dbt.2 -c "$PROJECT_DIR"
tmux send-keys -t kmutt:dbt.3 "psql -h localhost -U $PG_USER -d $PG_DB" C-m

# window 2 — logs
tmux new-window -t kmutt -n logs -c "$PROJECT_DIR"
tmux send-keys -t kmutt:logs 'docker compose logs -f' C-m

# focus กลับมาที่ window แรก pane Claude
tmux select-window -t kmutt:dbt
tmux select-pane -t kmutt:dbt.1

# attach
tmux attach -t kmutt
```

เก็บเป็น script `~/bin/kmutt-up.sh` แล้ว `chmod +x` — เรียกครั้งเดียวจบ

**Layout ที่ได้:**

```text
+--------------------+--------------------+
|                    |   dbt commands     |
|                    |   (pane 2)         |
|   Claude Code      +--------------------+
|   (pane 1)         |   psql shell       |
|                    |   (pane 3)         |
+--------------------+--------------------+
[window 1: dbt]  [window 2: logs]
```

**Keybinding ที่ใช้บ่อยใน session นี้:**

| Action | Keys |
| --- | --- |
| สลับไป pane Claude | `prefix + h` |
| สลับไป pane dbt | `prefix + l` แล้ว `prefix + k` |
| Zoom pane Claude เต็มจอ ตอนคุย | `prefix + z` (กดอีกครั้งเพื่อย่อ) |
| ไป window logs | `prefix + 2` |
| กลับ window dbt | `prefix + 1` |
| Detach ก่อนปิดคอม | `prefix + d` |

### 5.2 Template: Next.js Freelance Project

**เป้าหมาย:** 1 session, 2 window
- Window 1 `dev`: 4 pane — Claude / `npm run dev` / git / db client
- Window 2 `prisma`: 1 pane — prisma studio หรือ migration commands

```bash
PROJECT_DIR="$HOME/projects/freelance-app"

tmux new-session -d -s nextjs -n dev -c "$PROJECT_DIR"

# pane 1 (ซ้ายบน) — Claude
tmux send-keys -t nextjs:dev 'claude' C-m

# pane 2 (ขวาบน) — dev server
tmux split-window -h -t nextjs:dev -c "$PROJECT_DIR"
tmux send-keys -t nextjs:dev.2 'npm run dev' C-m

# pane 3 (ซ้ายล่าง) — git
tmux split-window -v -t nextjs:dev.1 -c "$PROJECT_DIR"
tmux send-keys -t nextjs:dev.3 'git status' C-m

# pane 4 (ขวาล่าง) — db client (เช่น pgcli)
tmux split-window -v -t nextjs:dev.2 -c "$PROJECT_DIR"
tmux send-keys -t nextjs:dev.4 'pgcli postgres://app:app@localhost:5432/app' C-m

# window 2 — prisma
tmux new-window -t nextjs -n prisma -c "$PROJECT_DIR"
tmux send-keys -t nextjs:prisma 'npx prisma studio' C-m

tmux select-window -t nextjs:dev
tmux select-pane -t nextjs:dev.1
tmux attach -t nextjs
```

**Layout:**

```text
+----------------+----------------+
| Claude (1)     | npm dev (2)    |
+----------------+----------------+
| git (3)        | pgcli (4)      |
+----------------+----------------+
[window 1: dev]  [window 2: prisma]
```

หลัง `npm run dev` ขึ้นมาแล้ว เปิด Chrome บน Windows ที่ `http://localhost:3000` — WSL2 forward port ให้อัตโนมัติ (ดู Section 6.4)

### 5.3 Template: Daily Standup Workflow

ใช้ tmux ให้ครบ cycle ของวัน — เลิกงานแล้ว detach, วันถัดไป attach กลับ

**ตอนเริ่มเช้า (วันแรก ยังไม่มี session):**

```bash
# เปิด WSL2 → รัน template ของวันนี้
~/bin/kmutt-up.sh             # ถ้าวันนี้ทำ dataplatform
# หรือ
~/bin/nextjs-up.sh            # ถ้าวันนี้ทำ freelance
```

**ระหว่างวัน — ไปทานข้าว / ประชุม:**

```text
prefix + d                     # detach — ทุกอย่างยังรันต่อ
```

ปิด Windows Terminal ได้เลย แม้ปิดฝา laptop ก็ได้ (WSL2 จะ pause)

**กลับมาทำงานต่อ:**

```bash
tmux ls
# kmutt: 2 windows (created Mon May 23 09:12:00 2026)
# nextjs: 2 windows (created Mon May 23 13:45:21 2026)

tmux a -t kmutt                # หรือ tmux a -t nextjs
```

**ตอนเลิกงาน:**

```text
prefix + d                     # detach ทุก session ที่กำลัง attach อยู่
```

อย่าปิด WSL2 (อย่ารัน `wsl --shutdown`) — process จะตายหมด session จะหาย

**สิ้นสัปดาห์ ถ้าจะ clean:**

```bash
tmux kill-session -t nextjs    # ลบเฉพาะที่ไม่ใช้
tmux kill-server               # ลบทุก session
```

> **Why ไม่ใช้แค่ Windows Terminal tab:** tab ของ Windows Terminal ผูกกับ process Windows ปิดหน้าต่าง = ทุก process ใน WSL ตาย แต่ tmux session อยู่ฝั่ง Linux — ตราบใดที่ WSL2 ไม่ shutdown session ยังอยู่ครบ

---

## Section 6: WSL2 ↔ Windows Integration

### 6.1 Filesystem: `/home/` vs `/mnt/c/`

| Path ใน WSL | คืออะไร | I/O speed | ใช้ทำอะไร |
| --- | --- | --- | --- |
| `/home/<user>/` (ext4 ใน VHDX ของ Linux) | Linux native filesystem | **เร็วมาก** | **เก็บ project ทั้งหมด** — git repo, node_modules, .venv, Docker volumes |
| `/mnt/c/`, `/mnt/d/` | Windows drive mount ผ่าน 9P protocol | **ช้ามาก** (10–20x ช้ากว่า) | แค่ "เข้าถึง" ไฟล์ Windows ชั่วคราว เช่น copy ไฟล์ Excel/PDF |

> **กฎทอง:** **ห้ามเก็บโค้ดที่ทำงานทุกวันใน `/mnt/c/`** เพราะ `npm install`, `git status`, `dbt run` จะช้าจนใช้ไม่ได้ ทุก project ให้ `cd ~` แล้ว `mkdir projects && cd projects && git clone ...`

ถ้าโปรเจกต์เดิมอยู่ใน `/mnt/c/Users/STUDENT/Documents/...`:

```bash
mkdir -p ~/projects
mv /mnt/c/Users/STUDENT/Documents/myproject ~/projects/
```

### 6.2 เข้าถึงไฟล์ WSL จาก Windows

ใน File Explorer พิมพ์ใน address bar:

```text
\\wsl$\Ubuntu\home\<your-linux-username>\
```

หรือบน Windows 11:

```text
\\wsl.localhost\Ubuntu\home\<your-linux-username>\
```

จะเปิด root home ของ Ubuntu ใน Explorer แชร์ไฟล์, copy ไฟล์ระหว่าง Windows app ↔ WSL ได้ปกติ

ทางลัด: ใน WSL terminal พิมพ์

```bash
explorer.exe .          # เปิด Explorer ที่ folder ปัจจุบัน
```

### 6.3 VS Code Remote-WSL

**Install (ครั้งเดียว):**

1. ลง VS Code บน Windows (จาก https://code.visualstudio.com)
2. ติดตั้ง extension ชื่อ **"WSL"** (publisher: Microsoft) ผ่าน Extensions panel
3. รีสตาร์ท VS Code

**ใช้งาน — เปิด project ใน WSL จาก WSL terminal:**

```bash
cd ~/projects/kmutt-dataplatform
code .
```

ครั้งแรก VS Code จะ install vscode-server ใน WSL อัตโนมัติ ใช้ทุก extension ที่ติด "Install in WSL" ไว้ — Python interpreter, Node, debugger ฯลฯ ทุกอย่างรันฝั่ง Linux ส่วน UI อยู่ฝั่ง Windows

**เช็คว่ากำลังใช้ WSL mode:** มุมล่างซ้ายของ VS Code จะขึ้น `WSL: Ubuntu`

### 6.4 Network: localhost Port Forwarding

WSL2 รันใน VM แยก แต่ Windows 11 + WSL2 รุ่นใหม่มี **localhost relay** อัตโนมัติ — เปิด server ใน WSL ที่ `0.0.0.0:3000` หรือ `localhost:3000` แล้วเปิด Chrome ฝั่ง Windows ที่ `http://localhost:3000` จะเห็นทันที

```bash
# ใน WSL — รัน Next.js
npm run dev                            # default bind 0.0.0.0:3000

# บน Windows Chrome:
# http://localhost:3000  — ใช้ได้เลย
```

**ถ้าใช้ไม่ได้** (ดู Section 8.5 ด้วย):

- เช็คว่า server bind ไป `0.0.0.0` ไม่ใช่ `127.0.0.1` (บาง framework default 127.0.0.1)
- รีสตาร์ท WSL: `wsl --shutdown` แล้วเปิดใหม่
- ปิด Windows Defender Firewall ทดสอบเป็นช่วงสั้นๆ

### 6.5 เปิด Windows app จาก WSL

| คำสั่งใน WSL | ทำอะไร |
| --- | --- |
| `explorer.exe .` | เปิด File Explorer ที่ folder ปัจจุบัน |
| `code .` | เปิด VS Code Remote-WSL |
| `clip.exe < file.txt` | copy เนื้อหาไฟล์เข้า Windows clipboard |
| `cmd.exe /c start https://...` | เปิด URL ด้วย default browser |
| `wslview <file>` | เปิดด้วย default Windows app (เหมือน `xdg-open`) |

---

## Section 7: Cheat Sheet

> หน้าเดียวจบ ครอบคลุม 80% ของการใช้งานประจำวัน พิมพ์แปะข้างจอได้

**Prefix = `Ctrl+a`** (กดแล้วปล่อย ตามด้วย key สั่ง)

### Top 10 Commands (จากนอก tmux)

| # | คำสั่ง | ทำอะไร |
| --- | --- | --- |
| 1 | `tmux new -s NAME` | สร้าง session ใหม่พร้อมตั้งชื่อ |
| 2 | `tmux ls` | ดูว่ามี session อะไรอยู่ |
| 3 | `tmux a` | attach session ล่าสุด |
| 4 | `tmux a -t NAME` | attach session ที่ระบุ |
| 5 | `tmux kill-session -t NAME` | ลบเฉพาะ session |
| 6 | `tmux kill-server` | ลบ tmux ทั้งหมด |
| 7 | `tmux source ~/.tmux.conf` | reload config จากนอก tmux |
| 8 | `tmux -V` | เช็คเวอร์ชัน |
| 9 | `wsl -l -v` | (PowerShell) เช็คสถานะ WSL distro |
| 10 | `wsl --shutdown` | (PowerShell) shutdown WSL ทั้งหมด — ใช้ตอน RAM บวม |

### Quick Reference (ใน tmux — กด prefix ก่อน)

**Session**

| Keys | Action |
| --- | --- |
| `d` | detach (session ยังรัน) |
| `s` | list/switch session |
| `$` | rename session |
| `S` | สร้าง session ใหม่ (custom) |
| `K` | kill session ปัจจุบัน (custom) |

**Window (= tab)**

| Keys | Action |
| --- | --- |
| `c` | สร้าง window ใหม่ |
| `1..9` | ไป window N |
| `n` / `p` | window ถัดไป / ก่อนหน้า |
| `Tab` | window ล่าสุด (toggle) |
| `,` | rename window |
| `w` | list/switch window |
| `&` | kill window |

**Pane (= split)**

| Keys | Action |
| --- | --- |
| `|` | split vertical (ขวา) |
| `-` | split horizontal (ล่าง) |
| `h j k l` | นาวิเกตซ้าย/ล่าง/บน/ขวา |
| `H J K L` | resize (กดค้างได้) |
| `z` | zoom (toggle pane เต็มจอ) |
| `Space` | cycle layout |
| `q` | แสดงเลข pane |
| `;` | สลับ pane ล่าสุด |
| `X` | kill pane (custom) |

**Copy mode**

| Keys | Action |
| --- | --- |
| `[` | เข้า copy mode |
| `q` หรือ `Esc` | ออก |
| `/` , `?` | ค้นหา fwd / bwd |
| `n` / `N` | ผลถัดไป / ก่อนหน้า |
| `v` | start selection |
| `y` | yank → Windows clipboard |
| `]` | paste จาก tmux buffer |

**Misc**

| Keys | Action |
| --- | --- |
| `r` | reload `~/.tmux.conf` (custom) |
| `?` | แสดงรายการ keybinding ทั้งหมด |
| `:` | command prompt (พิมพ์ tmux command ตรงๆ) |
| `t` | แสดงนาฬิกาบนหน้าจอ |

---

## Section 8: Troubleshooting

### 8.1 tmux session ค้าง, ต้อง kill ทุก session

```bash
tmux kill-server
```

ถ้า `tmux ls` หรือ `tmux a` ค้าง:

```bash
pkill -9 tmux                            # ฆ่า process ทั้งหมด
rm -rf /tmp/tmux-$(id -u)                # ลบ socket directory
tmux ls                                  # ควรเห็น "no server running"
```

### 8.2 Config ไม่ reload หลังแก้

ลำดับเช็ค:

```bash
# 1. ทดสอบ syntax (ถ้า error จะ print ออกมา)
tmux source-file ~/.tmux.conf

# 2. เช็คว่า binding กด prefix + r ถูก register หรือยัง
tmux list-keys | grep -i "source-file"

# 3. อยู่ใน session ที่ start ก่อนเพิ่ม binding หรือเปล่า?
#    ถ้าใช่ — ใช้คำสั่งใน 1 หรือ kill-server แล้วเปิดใหม่
```

### 8.3 WSL2 กิน RAM เยอะเกินไป

WSL2 จอง RAM แบบ dynamic แต่ไม่คืนคืนให้ Windows ทันที — สร้าง `C:\Users\<YourWindowsUser>\.wslconfig`:

```ini
[wsl2]
memory=8GB              # cap RAM ที่ WSL2 ใช้ได้ (เลือกตาม RAM เครื่อง — เครื่อง 16GB ใช้ 8GB, เครื่อง 32GB ใช้ 16GB)
processors=4            # cap จำนวน vCPU
swap=4GB                # swap file ของ WSL2
localhostForwarding=true
```

แล้วบังคับ apply:

```powershell
wsl --shutdown
```

เปิด WSL ใหม่ คาเอย — `.wslconfig` มีผลทุก distro ภายใต้ WSL2

### 8.4 ใช้ `wsl --shutdown` ตอนไหน

ใช้เมื่อ:

- เพิ่งแก้ `.wslconfig` แล้วต้องการ apply
- RAM ของ WSL2 บวมมาก ต้องการคืนให้ Windows ทันที
- WSL2 hang ไป network เพี้ยน เปิด distro ก็ค้าง

**ห้ามใช้เมื่อ:**

- มี tmux session ที่ยังไม่อยากให้ตาย → ทุก process ใน WSL จะถูก SIGKILL ทันที session หาย
- กำลังมี long-running job (dbt seed, npm build) → งานพัง

### 8.5 Port ใน WSL ไม่ถึง Windows browser

เช็คตามลำดับ:

1. Server bind ที่อะไร?
   ```bash
   # ใน WSL
   ss -tlnp | grep 3000
   # ต้องเห็น 0.0.0.0:3000 หรือ *:3000
   # ถ้าเห็น 127.0.0.1:3000 — แก้ config ของ framework ให้ bind 0.0.0.0
   ```
2. Firewall Windows block?
   ```powershell
   # Test as admin ชั่วคราว
   New-NetFirewallRule -DisplayName "WSL2 dev" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 3000
   ```
3. รีสตาร์ท WSL:
   ```powershell
   wsl --shutdown
   ```
4. Windows 11 บางเวอร์ชันมี bug — เพิ่ม manual port proxy (PowerShell as admin):
   ```powershell
   $wslIp = (wsl hostname -I).Trim().Split()[0]
   netsh interface portproxy add v4tov4 listenport=3000 listenaddress=0.0.0.0 connectport=3000 connectaddress=$wslIp
   ```

### 8.6 Clock skew หลัง laptop sleep

อาการ: หลัง wake from sleep ใช้ `git`, `npm`, `apt`, JWT auth พังเพราะเวลาใน WSL2 ห่างจาก Windows หลายชั่วโมง

แก้ทันที:

```bash
sudo hwclock -s          # อ่านเวลาจาก hardware clock (= Windows)
# หรือ
sudo ntpdate pool.ntp.org
```

แก้ถาวร — ลง systemd-timesyncd (รุ่นใหม่ของ WSL2 + Ubuntu 22.04 ขึ้นไปมีให้แล้ว):

```bash
sudo apt install -y systemd-timesyncd
sudo systemctl enable --now systemd-timesyncd
timedatectl status
```

ถ้ายังอยู่ ตั้ง cronjob hourly:

```bash
echo "0 * * * * /usr/sbin/hwclock -s" | sudo tee /etc/cron.d/wsl-clock-sync
```

---

## Section 9: ขั้นต่อไป (Advanced)

เมื่อใช้ tmux จนคล่อง ค่อยลองเพิ่ม:

### 9.1 tpm (tmux Plugin Manager)

จัดการ plugin tmux ด้วย git ผ่าน 1 บรรทัดใน `.tmux.conf`

```bash
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

เพิ่มใน `~/.tmux.conf` ท้ายไฟล์:

```bash
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
run '~/.tmux/plugins/tpm/tpm'
```

Install plugin: `prefix + I` (capital i)

### 9.2 tmux-resurrect + tmux-continuum

ปัญหา: `wsl --shutdown` แล้ว session หายหมด ต้อง bootstrap ใหม่
แก้: 2 plugin นี้ — save/restore session อัตโนมัติทุก 15 นาที กลับมาเปิด tmux แล้วทุก window/pane กลับมาเลย

```bash
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'
set -g @continuum-restore 'on'
```

`prefix + Ctrl+s` save, `prefix + Ctrl+r` restore (manual)

### 9.3 tmuxinator / tmuxp

แทนที่ shell script ใน Section 5 ด้วย YAML declarative — config อ่านง่ายและ portable

- **tmuxinator** (Ruby): `~/.config/tmuxinator/kmutt.yml` → `mux kmutt`
- **tmuxp** (Python): `~/.tmuxp/nextjs.yaml` → `tmuxp load nextjs`

แนะนำ **tmuxp** ถ้าคุณใช้ Python ใน data stack อยู่แล้ว — ไม่ต้องลง Ruby เพิ่ม

ตัวอย่าง `kmutt.yaml`:

```yaml
session_name: kmutt
start_directory: ~/projects/kmutt-dataplatform
windows:
  - window_name: dbt
    layout: main-vertical
    panes:
      - claude
      - source .venv/bin/activate && dbt debug
      - psql -h localhost -U postgres -d dataplatform
  - window_name: logs
    panes:
      - docker compose logs -f
```

### 9.4 Integration กับ Neovim

ใน `init.lua` (Neovim) เพิ่ม plugin `christoomey/vim-tmux-navigator` — กด `Ctrl+h/j/k/l` จะ navigate ข้าม Neovim split ↔ tmux pane ได้แบบไร้รอยต่อ (ไม่ต้องกด prefix)

ใน `.tmux.conf` ต้องเพิ่ม smart-detect (ดูเอกสาร plugin) — ถือเป็น power-user setup ที่คุ้มสุดๆ ถ้าคุณใช้ Neovim เป็นหลัก
