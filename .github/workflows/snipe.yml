#!/usr/bin/env python3
# -*- coding: utf-8 -*-
import json
import os
import re
import sys
import time
import urllib.parse
from datetime import datetime, timezone, timedelta

import requests
from bs4 import BeautifulSoup

SHOP = "https://standoff-2.com/shop/?type=all&rare=all&skin="
GAME = "https://link.standoff2.com/marketpl"
UA = ("Mozilla/5.0 (Linux; Android 12; SM-A525F) AppleWebKit/537.36 "
      "(KHTML, like Gecko) Chrome/124.0.0.0 Mobile Safari/537.36")

TG_TOKEN = os.environ["TG_TOKEN"]
TG_CHAT = os.environ["TG_CHAT"]
STATE_PATH = os.environ.get("STATE_PATH", "state.json")
CFG_PATH = os.environ.get("CFG_PATH", "config.json")
API = f"https://api.telegram.org/bot{TG_TOKEN}"

DEFAULTS = {
    "min_price": 20, "max_price": 500, "min_discount": 40,
    "min_profit_pct": 20, "repeat_hours": 6, "paused": False, "watch": [],
}


def load_json(path, fallback):
    try:
        with open(path, encoding="utf-8") as f:
            return json.load(f)
    except Exception:
        return fallback


def save_json(path, data):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)


cfg = {**DEFAULTS, **load_json(CFG_PATH, {})}
state = load_json(STATE_PATH, {})
state.setdefault("offset", 0)
state.setdefault("last", {})


def tg(method, **payload):
    try:
        r = requests.post(f"{API}/{method}", json=payload, timeout=20)
        return r.json()
    except Exception as e:
        print(f"telegram {method} error: {e}", file=sys.stderr)
        return {}


def say(text, buttons=None):
    payload = {
        "chat_id": TG_CHAT,
        "text": text,
        "parse_mode": "HTML",
        "disable_web_page_preview": True,
    }
    if buttons:
        payload["reply_markup"] = {"inline_keyboard": buttons}
    tg("sendMessage", **payload)


def esc(s):
    return s.replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;")


def fetch_prices(name):
    url = SHOP + urllib.parse.quote(name)
    r = requests.get(url, headers={"User-Agent": UA}, timeout=25)
    if r.status_code != 200:
        return None
    text = BeautifulSoup(r.text, "html.parser").get_text(" ")
    text = re.sub(r"\s+", " ", text)

    def num(pattern):
        m = re.search(pattern, text)
        if not m:
            return None
        try:
            return float(m.group(1).replace(",", "."))
        except ValueError:
            return None

    current = num(r"Цена ([\d]+(?:[.,]\d+)?)G")
    if current is None:
        current = num(r"Продажи ([\d]+(?:[.,]\d+)?)G")
    return {
        "current": current,
        "min": num(r"Мин\.цена: ([\d]+(?:[.,]\d+)?)G"),
        "max": num(r"Макс\.цена: ([\d]+(?:[.,]\d+)?)G"),
        "avg": num(r"Средняя цена: ([\d]+(?:[.,]\d+)?)G"),
        "url": url,
    }


def judge(name, d, now):
    cur, avg = d["current"], d["avg"]
    if not cur or not avg or avg <= 0:
        return None
    if cur < cfg["min_price"] or cur > cfg["max_price"]:
        return None

    discount = (avg - cur) / avg * 100
    if discount < cfg["min_discount"]:
        return None

    profit = avg * 0.8 - cur
    if profit <= 0:
        return None
    profit_pct = profit / cur * 100
    if profit_pct < cfg["min_profit_pct"]:
        return None

    prev = state["last"].get(name)
    if prev:
        prev_ts = datetime.fromisoformat(prev["ts"])
        fresh = now - prev_ts < timedelta(hours=cfg["repeat_hours"])
        if fresh and cur >= prev["price"] * 0.98:
            return None

    return {"discount": discount, "profit": profit, "profit_pct": profit_pct}


HELP = (
    "<b>Команды</b>\n"
    "/watch Название — добавить скин\n"
    "/del Название — убрать\n"
    "/list — что в вотчлисте\n"
    "/budget мин макс — рамки бюджета\n"
    "/discount 30 — минимальная скидка, %\n"
    "/pause, /resume — стоп и старт\n"
    "Название бери ровно как на рынке, например: AKR \"Dragon\""
)


def handle(text):
    text = text.strip()
    low = text.lower()

    if low.startswith("/watch"):
        name = text[6:].strip().strip('"')
        if not name:
            return "Укажи название скина после команды."
        if any(w.lower() == name.lower() for w in cfg["watch"]):
            return "Уже в вотчлисте."
        cfg["watch"].append(name)
        save_json(CFG_PATH, cfg)
        return f"Добавил: {esc(name)}"

    if low.startswith("/del"):
        name = text[4:].strip().strip('"').lower()
        before = len(cfg["watch"])
        cfg["watch"] = [w for w in cfg["watch"] if name not in w.lower()]
        if len(cfg["watch"]) == before:
            return "Не нашёл такой скин."
        save_json(CFG_PATH, cfg)
        return "Убрал."

    if low.startswith("/list"):
        if not cfg["watch"]:
            return "Вотчлист пуст."
        return "<b>Вотчлист</b>\n" + "\n".join(f"— {esc(w)}" for w in cfg["watch"])

    if low.startswith("/budget"):
        parts = text.split()
        if len(parts) != 3:
            return "Формат: /budget 20 500"
        try:
            cfg["min_price"], cfg["max_price"] = float(parts[1]), float(parts[2])
        except ValueError:
            return "Нужны два числа."
        save_json(CFG_PATH, cfg)
        return f"Бюджет: {cfg['min_price']:.0f}–{cfg['max_price']:.0f} G"

    if low.startswith("/discount"):
        parts = text.split()
        try:
            cfg["min_discount"] = float(parts[1])
        except (IndexError, ValueError):
            return "Формат: /discount 30"
        save_json(CFG_PATH, cfg)
        return f"Минимальная скидка: {cfg['min_discount']:.0f}%"

    if low.startswith("/pause"):
        cfg["paused"] = True
        save_json(CFG_PATH, cfg)
        return "На паузе."

    if low.startswith("/resume"):
        cfg["paused"] = False
        save_json(CFG_PATH, cfg)
        return "Работаю."

    if low.startswith(("/start", "/help")):
        return HELP

    return "Не понял. /help"


def poll_commands():
    res = tg("getUpdates", offset=state["offset"], timeout=0)
    for upd in res.get("result", []):
        state["offset"] = upd["update_id"] + 1
        msg = upd.get("message") or {}
        chat_id = str((msg.get("chat") or {}).get("id", ""))
        text = msg.get("text")
        if text and chat_id == str(TG_CHAT):
            say(handle(text))


def scan(now):
    if cfg["paused"]:
        print("paused")
        return
    for name in list(cfg["watch"]):
        d = fetch_prices(name)
        time.sleep(1.2)
        if not d:
            print(f"skip {name}")
            continue
        verdict = judge(name, d, now)
        if not verdict:
            print(f"- {name}: {d['current']} / avg {d['avg']}")
            continue

        state["last"][name] = {
            "price": d["current"], "ts": now.isoformat(timespec="seconds"),
        }
        net = d["current"] * 0.8
        text = (
            f"<b>{esc(name)}</b>\n"
            f"Цена сейчас: <b>{d['current']:.2f}G</b>\n"
            f"Средняя: {d['avg']:.2f}G · мин {d['min'] or 0:.2f}G · макс {d['max'] or 0:.2f}G\n"
            f"Скидка к средней: <b>−{verdict['discount']:.0f}%</b>\n"
            f"Чистыми при продаже по средней: {net:.2f}G "
            f"(профит {verdict['profit']:.2f}G, +{verdict['profit_pct']:.0f}%)"
        )
        buttons = [[
            {"text": "Открыть карточку", "url": d["url"]},
            {"text": "В игру", "url": GAME},
        ]]
        say(text, buttons)
        print(f"ALERT {name} {d['current']} avg {d['avg']}")


def main():
    now = datetime.now(timezone.utc)
    poll_commands()
    scan(now)
    save_json(STATE_PATH, state)
    save_json(CFG_PATH, cfg)


if __name__ == "__main__":
    main()
