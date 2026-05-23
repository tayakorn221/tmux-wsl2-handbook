# tmux-wsl2-handbook

คู่มืออ้างอิงส่วนตัวสำหรับการใช้ tmux บน Windows 11 + WSL2 (Ubuntu) — single source of truth ที่ออกแบบให้กลับมาเปิดดูได้โดยไม่ต้อง search Google ใหม่

## Why this exists

- รวมความรู้ tmux + WSL2 + workflow dev/data ที่ใช้ทุกวันไว้ที่เดียว
- ไม่ต้อง search Stack Overflow ซ้ำเมื่อลืม keybinding หรือ syntax
- portable — clone repo บนเครื่องใหม่ คัดลอก `tmux.conf` ใช้ได้ทันที
- เป็น living document — อัพเดตเมื่อ workflow เปลี่ยน

## Quick Start

ถ้ารีบใช้ ข้ามไปที่ [Section 7: Cheat Sheet](./tmux-wsl2-handbook.md#section-7-cheat-sheet) ของ handbook ได้เลย

ติดตั้ง config:

```bash
git clone https://github.com/<your-username>/tmux-wsl2-handbook.git ~/tmux-wsl2-handbook
ln -s ~/tmux-wsl2-handbook/tmux.conf ~/.tmux.conf
tmux source-file ~/.tmux.conf
```

## Contents

| ไฟล์ | เนื้อหา |
| --- | --- |
| [`tmux-wsl2-handbook.md`](./tmux-wsl2-handbook.md) | คู่มือฉบับสมบูรณ์ 9 section |
| [`tmux.conf`](./tmux.conf) | config file พร้อม comment ทุกบรรทัด — copy ไป `~/.tmux.conf` |

## Handbook TOC

1. [Concept พื้นฐาน](./tmux-wsl2-handbook.md#section-1-concept-พื้นฐาน) — WSL2, tmux, mental model
2. [การติดตั้ง](./tmux-wsl2-handbook.md#section-2-การติดตั้ง) — `wsl --install`, apt, troubleshoot BIOS
3. [tmux Basics](./tmux-wsl2-handbook.md#section-3-tmux-basics) — session/window/pane/copy mode
4. [Configuration](./tmux-wsl2-handbook.md#section-4-configuration) — อธิบาย `tmux.conf` แบบบรรทัดต่อบรรทัด
5. [Workflow Templates](./tmux-wsl2-handbook.md#section-5-workflow-templates) — KMUTT DataPlatform, Next.js, daily standup
6. [WSL2 ↔ Windows Integration](./tmux-wsl2-handbook.md#section-6-wsl2--windows-integration) — filesystem, VS Code, port forwarding
7. [Cheat Sheet](./tmux-wsl2-handbook.md#section-7-cheat-sheet) — แปะข้างจอได้
8. [Troubleshooting](./tmux-wsl2-handbook.md#section-8-troubleshooting) — RAM, port, clock skew
9. [ขั้นต่อไป (Advanced)](./tmux-wsl2-handbook.md#section-9-ขั้นต่อไป-advanced) — tpm, resurrect, tmuxp, Neovim

## Environment

- Windows 11
- WSL2 + Ubuntu 22.04 / 24.04
- tmux 3.2a / 3.4
- Stack: Claude Code, dbt Core, PostgreSQL, Next.js 14, Docker, Power BI

## License

Personal use. คู่มือนี้เขียนสำหรับตัวเอง — fork มาปรับใช้ได้ตามต้องการ

---

**Last updated:** 2026-05-23
