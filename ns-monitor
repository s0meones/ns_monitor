#!/usr/bin/env bash
# =========================================================
# setup_ns_monitor.sh
# 在 Debian 12 上一键部署 “RSS ➜ Telegram” 监控工具
#   - 项目目录：/root/ns_monitor
#   - Docker 服务名：ns-monitor
# =========================================================
set -euo pipefail

# ------------ 交互式收集配置 ------------------------------
prompt() {
  local __var=$1 __msg=$2 __def=${3-}
  local __tmp
  if command -v whiptail >/dev/null 2>&1; then
    __tmp=$(whiptail --inputbox "$__msg" 10 70 "$__def" --title "NS-Monitor 设置" 3>&1 1>&2 2>&3) || true
  else
    read -rp "$__msg [默认: ${__def:-空}] : " __tmp
  fi
  printf -v "$__var" '%s' "${__tmp:-$__def}"
}

echo "========================================================="
echo "      NS-Monitor (RSS ➜ Telegram) 安装向导"
echo "========================================================="

prompt BOT_TOKEN  "请输入 Telegram Bot Token"            ""
prompt CHAT_ID    "请输入目标 chat_id（可个人或群）"      ""
prompt DEFAULT_FEEDS "默认订阅源，逗号分隔" \
       "https://planetpython.org/rss20.xml,https://www.debian.org/News/news"
prompt DEFAULT_KEYS  "默认关键词，逗号分隔（留空=全部推送）" "debian,docker"
prompt FETCH_INTERVAL "拉取频率（秒）"                    "300"

# ------------ 安装 Docker & 依赖 ---------------------------
echo "==> 安装 Docker 及依赖 ..."
sudo apt update -y
sudo apt install -y ca-certificates curl gnupg lsb-release jq whiptail

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | \
  sudo tee /etc/apt/keyrings/docker.gpg >/dev/null
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/debian $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list >/dev/null

sudo apt update -y
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER" || true   # 重新登陆后生效

# ------------ 生成项目文件 -------------------------------
PROJECT_DIR="/root/ns_monitor"
echo "==> 创建项目目录 $PROJECT_DIR"
sudo mkdir -p "$PROJECT_DIR"/data
sudo chown -R "$USER":"$USER" "$PROJECT_DIR"
cd "$PROJECT_DIR"

# Dockerfile
cat > Dockerfile <<'DOCKERFILE'
FROM python:3.12-slim
ENV TZ=Asia/Shanghai
RUN apt-get update && apt-get install -y tzdata && rm -rf /var/lib/apt/lists/*
RUN pip install --no-cache-dir feedparser python-telegram-bot==20.7
WORKDIR /app
COPY rss_monitor.py /app/rss_monitor.py
VOLUME ["/data"]
ENTRYPOINT ["python", "/app/rss_monitor.py"]
DOCKERFILE

# docker-compose.yml
cat > docker-compose.yml <<'COMPOSE'
version: "3.9"
services:
  ns-monitor:
    build: .
    env_file: .env
    volumes:
      - ./data:/data
    restart: unless-stopped
COMPOSE

# 主程序
cat > rss_monitor.py <<'PY'
import os, json, asyncio, feedparser
from pathlib import Path
from datetime import datetime, timezone
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes, AIORateLimiter

BOT_TOKEN  = os.getenv("BOT_TOKEN")
CHAT_ID    = int(os.getenv("CHAT_ID"))
RAW_FEEDS  = os.getenv("FEEDS", "")
RAW_KEYS   = os.getenv("KEYWORDS", "")
INTERVAL   = int(os.getenv("INTERVAL", 300))
STATE_FILE = Path("/data/state.json")

def load_state():
    if STATE_FILE.exists():
        return json.loads(STATE_FILE.read_text("utf-8"))
    return {
        "feeds": [u.strip() for u in RAW_FEEDS.split(",") if u.strip()],
        "keywords": [k.strip().lower() for k in RAW_KEYS.split(",") if k.strip()],
        "sent_ids": []
    }

def save_state(s):
    STATE_FILE.parent.mkdir(parents=True, exist_ok=True)
    STATE_FILE.write_text(json.dumps(s, ensure_ascii=False, indent=2), "utf-8")

state = load_state()

def uid(e): return e.get("id") or e.get("link")
def hit(e, keys):
    if not keys: return True
    txt = f"{e.get('title','')} {e.get('summary','')}".lower()
    return any(k in txt for k in keys)

async def cmd_keywords(u:Update,c:ContextTypes.DEFAULT_TYPE):
    if u.effective_chat.id!=CHAT_ID: return
    if c.args:
        state["keywords"]=[w.lower() for w in c.args if w.strip()]
        save_state(state)
        await u.message.reply_text("✅ 关键词已更新: "+(", ".join(state["keywords"]) or "（空）"))
    else:
        await u.message.reply_text("当前关键词: "+(", ".join(state["keywords"]) or "（空）"))

async def cmd_addkw(u:Update,c:ContextTypes.DEFAULT_TYPE):
    if u.effective_chat.id!=CHAT_ID or not c.args: return
    added=[w.lower() for w in c.args if w.strip() and w.lower() not in state["keywords"]]
    if added:
        state["keywords"].extend(added); save_state(state)
    await u.message.reply_text("✅ 新增关键词: "+(", ".join(added) or "无"))

async def cmd_rmkw(u:Update,c:ContextTypes.DEFAULT_TYPE):
    if u.effective_chat.id!=CHAT_ID or not c.args: return
    removed=[w.lower() for w in c.args if w.lower() in state["keywords"]]
    for w in removed: state["keywords"].remove(w)
    if removed: save_state(state)
    await u.message.reply_text("✅ 移除关键词: "+(", ".join(removed) or "无"))

async def cmd_feeds(u:Update,c:ContextTypes.DEFAULT_TYPE):
    if u.effective_chat.id!=CHAT_ID: return
    if c.args:
        state["feeds"]=[x for x in c.args if x.strip()]
        save_state(state)
        await u.message.reply_text(f"✅ 订阅源已更新，共 {len(state['feeds'])} 条")
    else:
        await u.message.reply_text("当前订阅:\n"+("\n".join(state["feeds"]) or "（空）"))

async def poll(app):
    while True:
        for url in state["feeds"]:
            try:
                d=feedparser.parse(url,agent="ns-monitor/1.0")
                for e in d.entries:
                    k=uid(e)
                    if not k or k in state["sent_ids"] or not hit(e,state["keywords"]): continue
                    txt=f"📡 <b>{e.get('title','(无标题)')}</b>\n{e.get('link','')}"
                    await app.bot.send_message(CHAT_ID,txt,parse_mode="HTML",disable_web_page_preview=True)
                    state["sent_ids"].append(k)
            except Exception as ex:
                print(f"[{datetime.now(timezone.utc)}] {ex}")
        save_state(state)
        await asyncio.sleep(INTERVAL)

def main():
    if not BOT_TOKEN or not CHAT_ID:
        raise RuntimeError("必须设置 BOT_TOKEN 和 CHAT_ID")
    app=ApplicationBuilder().token(BOT_TOKEN).rate_limiter(AIORateLimiter()).build()
    for cmd,fn in [("keywords",cmd_keywords),("addkw",cmd_addkw),
                   ("rmkw",cmd_rmkw),("feeds",cmd_feeds)]:
        app.add_handler(CommandHandler(cmd,fn))
    app.job_queue.run_once(lambda *_: asyncio.create_task(poll(app)),0)
    print("Bot started ..."); app.run_polling()

if __name__=="__main__": main()
PY

# .env
cat > .env <<EOF
BOT_TOKEN=${BOT_TOKEN}
CHAT_ID=${CHAT_ID}
FEEDS=${DEFAULT_FEEDS}
KEYWORDS=${DEFAULT_KEYS}
INTERVAL=${FETCH_INTERVAL}
EOF

# ------------ 构建并启动 -------------------------------
echo "==> 构建镜像并启动 ..."
docker compose build
docker compose up -d

echo "========================================================="
echo "  NS-Monitor 部署完成！容器日志如下（Ctrl+C 退出查看）："
echo "========================================================="
docker compose logs -f
