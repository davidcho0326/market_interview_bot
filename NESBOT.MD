#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
여신금융협회 면접 대비 뉴스 모니터링 텔레그램 봇
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
구조: PC(Windows)에서 상시 실행 → 모바일 텔레그램에서 /news 명령 → 뉴스 전송

사전 준비:
1) 네이버 개발자센터 → 검색 API → Client ID / Secret 발급
2) 텔레그램 봇 토큰 & Chat ID

실행:
  python news_bot.py

텔레그램 명령어:
  /news       → 즉시 뉴스 수집·전송
  /help       → 도움말
"""

import urllib.request
import urllib.parse
import json
import os
import re
import hashlib
import time
from datetime import datetime

# ============================================================
# ★ 여기에 본인 키 입력 ★
# ============================================================
NAVER_CLIENT_ID     = "YOUR_NAVER_CLIENT_ID"
NAVER_CLIENT_SECRET = "YOUR_NAVER_CLIENT_SECRET"
TELEGRAM_BOT_TOKEN  = "YOUR_TELEGRAM_BOT_TOKEN"
TELEGRAM_CHAT_ID    = "YOUR_TELEGRAM_CHAT_ID"

# ============================================================
# 검색 키워드 설정 (4개 영역)
# ============================================================
SEARCH_KEYWORDS = [
    # 🏛️ 여신금융협회 관련
    "여신금융협회",
    "여신협회",
    # 💳 카드/캐피탈/리스 업권
    "카드사 규제",
    "캐피탈 대출",
    "리스 금융",
    "할부금융",
    "신용카드 시장",
    # 📋 금융위/금감원 정책·규제
    "여신전문금융업법",
    "금융위 카드",
    "금감원 여신",
    "가맹점 수수료",
    # 🔥 핀테크/BNPL 경쟁
    "BNPL",
    "선불충전금",
    "간편결제 규제",
    "빅테크 금융",
]

KEYWORD_CATEGORY = {
    "여신금융협회":     "🏛️ 협회",
    "여신협회":         "🏛️ 협회",
    "카드사 규제":      "💳 업권",
    "캐피탈 대출":      "💳 업권",
    "리스 금융":        "💳 업권",
    "할부금융":         "💳 업권",
    "신용카드 시장":    "💳 업권",
    "여신전문금융업법": "📋 정책",
    "금융위 카드":      "📋 정책",
    "금감원 여신":      "📋 정책",
    "가맹점 수수료":    "📋 정책",
    "BNPL":             "🔥 경쟁",
    "선불충전금":       "🔥 경쟁",
    "간편결제 규제":    "🔥 경쟁",
    "빅테크 금융":      "🔥 경쟁",
}

# 설정값
MAX_NEWS_PER_KEYWORD = 5
MAX_NEWS_PER_MESSAGE = 20
SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
SENT_NEWS_FILE = os.path.join(SCRIPT_DIR, "sent_news.json")
SENT_NEWS_MAX_KEEP = 2000


# ============================================================
# 유틸리티
# ============================================================

def clean_html(text):
    text = re.sub(r'<[^>]+>', '', text)
    for old, new in [('&quot;','"'),('&amp;','&'),('&lt;','<'),('&gt;','>'),('&apos;',"'")]:
        text = text.replace(old, new)
    return text.strip()


def make_hash(title, link):
    return hashlib.md5(f"{title}|{link}".encode('utf-8')).hexdigest()


def load_sent_news():
    if os.path.exists(SENT_NEWS_FILE):
        try:
            with open(SENT_NEWS_FILE, 'r', encoding='utf-8') as f:
                return set(json.load(f))
        except:
            return set()
    return set()


def save_sent_news(sent_set):
    sent_list = list(sent_set)
    if len(sent_list) > SENT_NEWS_MAX_KEEP:
        sent_list = sent_list[-SENT_NEWS_MAX_KEEP:]
    with open(SENT_NEWS_FILE, 'w', encoding='utf-8') as f:
        json.dump(sent_list, f, ensure_ascii=False)


def log(msg):
    print(f"[{datetime.now().strftime('%H:%M:%S')}] {msg}")


# ============================================================
# 네이버 뉴스 검색 API
# ============================================================

def search_naver_news(keyword, display=10, sort="date"):
    base_url = "https://openapi.naver.com/v1/search/news.json"
    params = urllib.parse.urlencode({
        "query": keyword,
        "display": display,
        "start": 1,
        "sort": sort,
    })
    url = f"{base_url}?{params}"

    req = urllib.request.Request(url)
    req.add_header("X-Naver-Client-Id", NAVER_CLIENT_ID)
    req.add_header("X-Naver-Client-Secret", NAVER_CLIENT_SECRET)

    try:
        response = urllib.request.urlopen(req, timeout=10)
        if response.getcode() == 200:
            return json.loads(response.read().decode('utf-8'))
    except Exception as e:
        log(f"⚠️ 네이버 API 오류 ({keyword}): {e}")
    return None


def collect_news():
    sent_hashes = load_sent_news()
    all_news = []
    seen_hashes = set()

    for keyword in SEARCH_KEYWORDS:
        result = search_naver_news(keyword, display=MAX_NEWS_PER_KEYWORD, sort="date")
        if not result or 'items' not in result:
            continue

        category = KEYWORD_CATEGORY.get(keyword, "📰")

        for item in result['items']:
            title = clean_html(item.get('title', ''))
            description = clean_html(item.get('description', ''))
            link = item.get('originallink') or item.get('link', '')
            pub_date = item.get('pubDate', '')

            news_hash = make_hash(title, link)
            if news_hash in sent_hashes or news_hash in seen_hashes:
                continue

            seen_hashes.add(news_hash)

            try:
                dt = datetime.strptime(pub_date, "%a, %d %b %Y %H:%M:%S %z")
                pub_str = dt.strftime("%m/%d %H:%M")
            except:
                pub_str = pub_date[:16] if pub_date else ""

            all_news.append({
                "title": title[:50] + "…" if len(title) > 50 else title,
                "description": description[:80] + "..." if len(description) > 80 else description,
                "link": link,
                "pub_date": pub_str,
                "category": category,
                "keyword": keyword,
                "hash": news_hash,
            })

    all_news.sort(key=lambda x: x['pub_date'], reverse=True)
    if len(all_news) > MAX_NEWS_PER_MESSAGE:
        all_news = all_news[:MAX_NEWS_PER_MESSAGE]

    return all_news, sent_hashes


# ============================================================
# 텔레그램 API
# ============================================================

def tg_request(method, data=None):
    url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/{method}"
    if data:
        payload = urllib.parse.urlencode(data).encode('utf-8')
    else:
        payload = None

    try:
        req = urllib.request.Request(url, data=payload)
        response = urllib.request.urlopen(req, timeout=30)
        return json.loads(response.read().decode('utf-8'))
    except Exception as e:
        log(f"⚠️ 텔레그램 API 오류 ({method}): {e}")
    return None


def send_message(text, chat_id=None):
    if chat_id is None:
        chat_id = TELEGRAM_CHAT_ID

    parts = split_message(text)
    for part in parts:
        result = tg_request("sendMessage", {
            "chat_id": chat_id,
            "text": part,
            "parse_mode": "HTML",
            "disable_web_page_preview": "true",
        })
        if result and result.get('ok'):
            log(f"✅ 메시지 전송 ({len(part)}자)")
        else:
            log(f"❌ 메시지 전송 실패")
            return False
    return True


def split_message(text, max_len=4000):
    if len(text) <= max_len:
        return [text]
    parts = []
    while text:
        if len(text) <= max_len:
            parts.append(text)
            break
        cut = text[:max_len].rfind('\n')
        if cut == -1:
            cut = max_len
        parts.append(text[:cut])
        text = text[cut:].lstrip('\n')
    return parts


def format_news_message(news_list):
    now = datetime.now().strftime("%Y.%m.%d %H:%M")
    lines = [
        f"📋 <b>여신금융 면접 뉴스브리핑</b>",
        f"🕐 {now} 기준\n",
    ]

    categories = {}
    for news in news_list:
        cat = news['category']
        if cat not in categories:
            categories[cat] = []
        categories[cat].append(news)

    for cat, items in categories.items():
        lines.append(f"\n<b>{cat}</b>")
        for news in items:
            lines.append(
                f"• <a href=\"{news['link']}\">{news['title']}</a>"
                f"\n  └ {news['pub_date']} | {news['keyword']}"
            )

    lines.append(f"\n─────────────────")
    lines.append(f"총 {len(news_list)}건")

    return "\n".join(lines)


# ============================================================
# 명령 처리
# ============================================================

def handle_news_command(chat_id):
    log("📡 뉴스 수집 시작...")
    send_message("🔍 뉴스 수집 중...", chat_id)

    news_list, sent_hashes = collect_news()

    if not news_list:
        send_message("📭 새로운 뉴스가 없습니다.\n이전에 보낸 뉴스와 중복된 기사만 검색되었어요.", chat_id)
        return

    log(f"📰 새 뉴스 {len(news_list)}건 발견")
    message = format_news_message(news_list)
    success = send_message(message, chat_id)

    if success:
        for news in news_list:
            sent_hashes.add(news['hash'])
        save_sent_news(sent_hashes)
        log(f"✅ {len(news_list)}건 전송 완료")


def handle_help_command(chat_id):
    help_text = (
        "📋 <b>여신금융 뉴스봇 명령어</b>\n\n"
        "/news  →  즉시 뉴스 수집·전송\n"
        "/help  →  도움말\n\n"
        "<b>모니터링 키워드 (4개 영역)</b>\n"
        "🏛️ 협회: 여신금융협회, 여신협회\n"
        "💳 업권: 카드사, 캐피탈, 리스, 할부금융\n"
        "📋 정책: 여신전문금융업법, 금융위, 금감원\n"
        "🔥 경쟁: BNPL, 선불충전금, 간편결제, 빅테크"
    )
    send_message(help_text, chat_id)


# ============================================================
# 텔레그램 폴링 (메시지 수신 대기)
# ============================================================

def start_polling():
    log("🤖 봇 시작! 텔레그램에서 /news 를 입력하세요.")
    log("─" * 40)

    last_update_id = 0

    while True:
        try:
            result = tg_request("getUpdates", {
                "offset": last_update_id + 1,
                "timeout": 30,
                "allowed_updates": '["message"]',
            })

            if result and result.get('ok') and result.get('result'):
                for update in result['result']:
                    last_update_id = update['update_id']

                    message = update.get('message', {})
                    text = message.get('text', '').strip()
                    chat_id = message.get('chat', {}).get('id')

                    if not text or not chat_id:
                        continue

                    # 본인 Chat ID만 응답 (보안)
                    if str(chat_id) != str(TELEGRAM_CHAT_ID):
                        log(f"⛔ 권한 없는 접근: chat_id={chat_id}")
                        continue

                    log(f"📩 명령 수신: {text}")

                    if text.startswith("/news"):
                        handle_news_command(chat_id)
                    elif text.startswith("/help") or text.startswith("/start"):
                        handle_help_command(chat_id)
                    else:
                        send_message("❓ 알 수 없는 명령어입니다.\n/help 를 입력해보세요.", chat_id)

        except KeyboardInterrupt:
            log("\n🛑 봇 종료")
            break
        except Exception as e:
            log(f"⚠️ 폴링 오류: {e}")
            time.sleep(5)


# ============================================================
# 메인
# ============================================================

def main():
    print("=" * 50)
    print("  📋 여신금융 면접 뉴스봇")
    print("  텔레그램에서 /news 로 뉴스를 받아보세요")
    print("=" * 50)

    if "YOUR_" in NAVER_CLIENT_ID or "YOUR_" in TELEGRAM_BOT_TOKEN:
        print("\n⚠️  API 키가 설정되지 않았습니다!")
        print("   스크립트 상단의 4개 변수를 수정해주세요:")
        print("   - NAVER_CLIENT_ID / NAVER_CLIENT_SECRET")
        print("   - TELEGRAM_BOT_TOKEN / TELEGRAM_CHAT_ID")
        return

    # 봇 연결 테스트
    me = tg_request("getMe")
    if me and me.get('ok'):
        bot_name = me['result'].get('username', '?')
        log(f"✅ 봇 연결 성공: @{bot_name}")
    else:
        log("❌ 봇 연결 실패! 토큰을 확인해주세요.")
        return

    start_polling()


if __name__ == "__main__":
    main()