# Mr.Pet × PetRisk.AI

## 專案概述

Mr.Pet × PetRisk.AI 下一代人寵健康 AI 生態系的展示網站。純靜態 HTML + Tailwind CSS CDN，無需建構工具。

## 結構

- `index.html` — Landing page 宣傳主頁
- `app.html` — Mr.Pet 飼主端 App UI（mobile-first，5 個 tab）
- `dashboard.html` — PetRisk.AI 獸醫端 Dashboard（desktop-first）

## 設計規範

- 色彩：金色 #D4A574（Mr.Pet）、青綠 #4ecdc4（PetRisk.AI）、深藍 #1a1a2e
- 字型：DM Sans + Noto Sans TC（Google Fonts CDN）
- 圖示：Heroicons inline SVG，不使用 emoji
- 風格：Glassmorphism cards、深色/暖色交替區段

## 部署

- 域名：mrpet.mhsu.me
- 主機：Linode (linode-maxmilian)
- 純靜態站，nginx 直接 serve
