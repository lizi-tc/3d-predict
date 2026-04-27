# -*- coding: utf-8 -*-
"""
福彩3D 下期预测 - 云端定时版
- 自动爬取最新 20 期数据
- 用 20期×12注 / 20期×15注 两种方案预测
- 通过 Server酱 推送到微信

环境变量（在 GitHub Secrets 中配置）：
  SERVERCHAN_KEY - Server酱的 SendKey

依赖：
  pip install requests
"""

import itertools
import os
import sys
from collections import Counter
from datetime import datetime, timezone, timedelta

import requests


# ============================================================
# 配置
# ============================================================
WANNENG_POOL = [
    "0126","0134","0159","0178","0239","0247","0258","0357","0368","0456",
    "0489","0679","1237","1245","1289","1358","1369","1468","1479","1567",
    "2348","2356","2469","2579","2678","3459","3467","3789","4578","5689"
]

API_URL = "https://www.cwl.gov.cn/cwl_admin/front/cwlkj/search/kjxx/findDrawNotice"

# 北京时间
BJ_TZ = timezone(timedelta(hours=8))


# ============================================================
# 数据爬取
# ============================================================
def fetch_recent_draws(n=20):
    headers = {
        "User-Agent": ("Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                       "AppleWebKit/537.36 (KHTML, like Gecko) "
                       "Chrome/120.0.0.0 Safari/537.36"),
        "Referer": "https://www.cwl.gov.cn/ygkj/wqkjgg/3d/",
    }
    params = {
        "name": "3d", "issueCount": "", "issueStart": "", "issueEnd": "",
        "dayStart": "", "dayEnd": "", "pageNo": "1",
        "pageSize": str(n), "week": "", "systemType": "PC",
    }
    resp = requests.get(API_URL, params=params, headers=headers, timeout=30)
    resp.raise_for_status()
    data = resp.json()
    if data.get("state") != 0:
        raise RuntimeError(f"API 返回错误：{data.get('message', data)}")
    raw = list(reversed(data.get("result", [])))
    draws = []
    for item in raw:
        code = item["red"].replace(" ", "").replace(",", "")
        if len(code) != 3:
            continue
        b, s, g = int(code[0]), int(code[1]), int(code[2])
        draws.append({
            "period": item["code"],
            "date": item["date"][:10],
            "nums": (b, s, g)
        })
    return draws


# ============================================================
# 形态分析（与之前完全一致）
# ============================================================
def get_form(nums):
    s = set(nums)
    return "豹子" if len(s) == 1 else "组三" if len(s) == 2 else "组六"


def sum_iv(s):
    for lo, hi in [(0,2),(3,5),(6,8),(9,11),(12,14),(15,17),(18,20),(21,23),(24,27)]:
        if lo <= s <= hi:
            return (lo, hi)


def span_iv(sp):
    if 0 <= sp <= 3: return (0, 3)
    elif 4 <= sp <= 6: return (4, 6)
    else: return (7, 9)


def analyze_form(history):
    sums = [sum(d['nums']) for d in history]
    spans = [max(d['nums']) - min(d['nums']) for d in history]
    parities = [sum(1 for n in d['nums'] if n % 2 == 1) for d in history]
    sizes = [sum(1 for n in d['nums'] if n >= 5) for d in history]

    top3_sum = [iv for iv, _ in Counter([sum_iv(s) for s in sums]).most_common(3)]
    top2_span = [iv for iv, _ in Counter([span_iv(sp) for sp in spans]).most_common(2)]
    top1_parity = Counter(parities).most_common(1)[0][0]
    top1_size = Counter(sizes).most_common(1)[0][0]

    digit_counter = Counter()
    for d in history:
        for n in d['nums']:
            digit_counter[n] += 1
    total = sum(digit_counter.values()) + 0.1 * 10
    digit_probs = {d: (digit_counter.get(d, 0) + 0.1) / total for d in range(10)}

    return {
        'top3_sum': top3_sum, 'top2_span': top2_span,
        'top1_parity': top1_parity, 'top1_size': top1_size,
        'digit_counter': digit_counter, 'digit_probs': digit_probs,
        'history': history
    }


def score_wanneng(four_code, form_info):
    digits = [int(c) for c in four_code]
    combos = list(itertools.combinations(digits, 3))
    h_sum = h_span = h_par = h_sz = 0
    for c in combos:
        s, sp = sum(c), max(c) - min(c)
        p = sum(1 for n in c if n % 2 == 1)
        sz = sum(1 for n in c if n >= 5)
        if sum_iv(s) in form_info['top3_sum']: h_sum += 1
        if span_iv(sp) in form_info['top2_span']: h_span += 1
        if p == form_info['top1_parity']: h_par += 1
        if sz == form_info['top1_size']: h_sz += 1
    n = len(combos)
    return (h_sum/n)*0.4 + (h_span/n)*0.3 + (h_par/n)*0.2 + (h_sz/n)*0.1


def select_core_wanneng(form_info):
    scored = [(c, score_wanneng(c, form_info)) for c in WANNENG_POOL]
    scored.sort(key=lambda x: -x[1])
    return scored[0]


def build_pool(core_four, form_info, num_aux):
    core_digits = set(int(c) for c in core_four)
    candidates = list(set(range(10)) - core_digits)
    last_nums = set(form_info['history'][-1]['nums'])

    miss_periods = {}
    for d in range(10):
        miss = 0
        for hist in reversed(form_info['history']):
            if d in hist['nums']: break
            miss += 1
        miss_periods[d] = miss

    aux_scores = []
    for x in candidates:
        five_code = sorted(core_digits | {x})
        combos = list(itertools.combinations(five_code, 3))
        h_sum = h_span = h_par = h_sz = 0
        for c in combos:
            s, sp = sum(c), max(c) - min(c)
            p = sum(1 for n in c if n % 2 == 1)
            sz = sum(1 for n in c if n >= 5)
            if sum_iv(s) in form_info['top3_sum']: h_sum += 1
            if span_iv(sp) in form_info['top2_span']: h_span += 1
            if p == form_info['top1_parity']: h_par += 1
            if sz == form_info['top1_size']: h_sz += 1
        n = len(combos)
        form_score = (h_sum/n)*0.4 + (h_span/n)*0.3 + (h_par/n)*0.2 + (h_sz/n)*0.1
        avg_prob = sum(form_info['digit_probs'][d] for d in five_code) / 5
        cold_hot = 0
        if x in last_nums: cold_hot += 0.2
        if any(abs(x - ln) == 1 for ln in last_nums): cold_hot += 0.1
        if miss_periods[x] >= 8: cold_hot += 0.1
        cold_hot = min(cold_hot, 0.5)
        aux_scores.append({'aux': x, 'form_score': form_score,
                           'avg_prob': avg_prob, 'cold_hot': cold_hot})

    max_p = max(c['avg_prob'] for c in aux_scores)
    for c in aux_scores:
        c['model_score'] = c['avg_prob'] / max_p
        c['total_score'] = 0.6*c['form_score'] + 0.3*c['model_score'] + 0.1*c['cold_hot']
    aux_scores.sort(key=lambda c: -c['total_score'])

    auxs = [a['aux'] for a in aux_scores[:num_aux]]
    pool_digits = sorted(core_digits | set(auxs))
    return {'pool': pool_digits,
            'all_combos': list(itertools.combinations(pool_digits, 3))}


def select_zuxuan_n(pool, form_info, n_combos):
    combos = pool['all_combos']
    scored = []
    for combo in combos:
        s, sp = sum(combo), max(combo) - min(combo)
        p = sum(1 for n in combo if n % 2 == 1)
        sz = sum(1 for n in combo if n >= 5)
        sum_hit = sum_iv(s) in form_info['top3_sum']
        span_hit = span_iv(sp) in form_info['top2_span']
        par_hit = p == form_info['top1_parity']
        sz_hit = sz == form_info['top1_size']
        priority = 0
        if sum_hit and span_hit:
            priority += 1000
        elif sum_hit or span_hit:
            priority += 500
            if par_hit or sz_hit: priority += 200
        elif par_hit or sz_hit:
            priority += 200
        priority += sum(form_info['digit_counter'].get(d, 0) for d in combo) * 5
        scored.append((combo, priority))
    scored.sort(key=lambda x: -x[1])
    return [s[0] for s in scored[:n_combos]]


def predict(history, n_zhu):
    form_info = analyze_form(history)
    core_four, _ = select_core_wanneng(form_info)
    pool = build_pool(core_four, form_info, 2)
    return {
        'core_four': core_four,
        'pool': pool['pool'],
        'zuxuan': select_zuxuan_n(pool, form_info, n_zhu),
        'form_info': form_info,
    }


def format_combo(combo):
    return ''.join(str(d) for d in sorted(combo))


# ============================================================
# 微信推送（Server酱）
# ============================================================
def push_to_wechat(title, content):
    key = os.environ.get('SERVERCHAN_KEY')
    if not key:
        print("⚠️  未设置 SERVERCHAN_KEY 环境变量，跳过推送")
        return False

    url = f"https://sctapi.ftqq.com/{key}.send"
    try:
        resp = requests.post(url, data={
            'title': title,
            'desp': content,
        }, timeout=10)
        result = resp.json()
        if result.get('code') == 0:
            print(f"✅ 推送成功")
            return True
        else:
            print(f"❌ 推送失败：{result}")
            return False
    except Exception as e:
        print(f"❌ 推送异常：{e}")
        return False


def build_markdown_message(predictions, draws):
    """构造 markdown 格式的微信推送内容"""
    today = datetime.now(BJ_TZ).strftime('%Y-%m-%d')
    last = draws[-1]
    try:
        next_period = str(int(last['period']) + 1)
    except:
        next_period = "下期"

    lines = []
    lines.append(f"## 🎰 福彩3D 第 {next_period} 期推荐\n")
    lines.append(f"**日期**：{today}\n")
    lines.append(f"**基于数据**：最近 {len(draws)} 期\n")
    lines.append(f"\n---\n")

    # 最近5期
    lines.append(f"### 📋 最近5期开奖\n")
    for d in draws[-5:]:
        n = d['nums']
        form = get_form(n)
        lines.append(f"- {d['period']} ({d['date']}): **{n[0]}{n[1]}{n[2]}** [{form}]")
    lines.append("")

    # 形态分析
    fi = predictions[12]['form_info']
    lines.append(f"### 📊 形态分析\n")
    sum_label = ' / '.join(f"{lo}-{hi}" for lo, hi in fi['top3_sum'])
    span_label = ' / '.join(f"{lo}-{hi}" for lo, hi in fi['top2_span'])
    lines.append(f"- 高频和值：{sum_label}")
    lines.append(f"- 高频跨度：{span_label}")
    lines.append(f"- 主流奇数：{fi['top1_parity']} 个")
    lines.append(f"- 主流大数：{fi['top1_size']} 个")
    lines.append(f"- 核心万能码：**{predictions[12]['core_four']}**")
    pool_str = ''.join(str(d) for d in predictions[12]['pool'])
    lines.append(f"- 选号池：**{pool_str}**\n")

    # 推荐号码
    for n_zhu in [12, 15]:
        pred = predictions[n_zhu]
        lines.append(f"### 🎯 方案 {n_zhu} 注（投入 {n_zhu*2} 元）\n")
        codes = [format_combo(c) for c in pred['zuxuan']]
        # 三个一行
        for i in range(0, len(codes), 3):
            row = codes[i:i+3]
            lines.append("**" + "**  **".join(row) + "**  ")
        lines.append("")

    # 话术
    lines.append(f"### 💬 彩票员话术\n")
    lines.append(f"```")
    lines.append(f"3D 第 {next_period} 期，组选六，每注2元：")
    for n_zhu in [12, 15]:
        codes = [format_combo(c) for c in predictions[n_zhu]['zuxuan']]
        lines.append(f"")
        lines.append(f"【{n_zhu}注 - {n_zhu*2}元】")
        for i in range(0, len(codes), 6):
            lines.append("  " + "  ".join(codes[i:i+6]))
    lines.append(f"```\n")

    lines.append(f"---\n")
    lines.append(f"_⚠️ 仅供娱乐，理性投注_")

    return "\n".join(lines)


# ============================================================
# 主流程
# ============================================================
def main():
    now = datetime.now(BJ_TZ)
    print("=" * 60)
    print(f"福彩3D 自动预测 | {now.strftime('%Y-%m-%d %H:%M:%S')} (北京时间)")
    print("=" * 60)

    # 1. 爬取
    try:
        draws = fetch_recent_draws(n=20)
        print(f"✅ 已获取 {len(draws)} 期数据")
    except Exception as e:
        print(f"❌ 爬取失败：{e}")
        push_to_wechat(
            "❌ 福彩3D爬取失败",
            f"### 爬取数据失败\n\n错误信息：\n```\n{e}\n```\n\n请检查日志"
        )
        sys.exit(1)

    if len(draws) < 10:
        msg = f"数据太少（仅 {len(draws)} 期），无法预测"
        print(f"❌ {msg}")
        push_to_wechat("❌ 福彩3D数据不足", msg)
        sys.exit(1)

    # 2. 预测
    predictions = {}
    for n_zhu in [12, 15]:
        predictions[n_zhu] = predict(draws, n_zhu)
    print(f"✅ 预测完成")

    # 3. 控制台打印
    last = draws[-1]
    try:
        next_period = str(int(last['period']) + 1)
    except:
        next_period = "下期"

    print(f"\n🎯 第 {next_period} 期推荐：")
    for n_zhu in [12, 15]:
        codes = [format_combo(c) for c in predictions[n_zhu]['zuxuan']]
        print(f"\n方案 {n_zhu} 注（{n_zhu*2}元）:")
        for i in range(0, len(codes), 6):
            print("  " + "  ".join(codes[i:i+6]))

    # 4. 推送
    print(f"\n📱 推送到微信...")
    content = build_markdown_message(predictions, draws)
    title = f"🎰 3D第{next_period}期推荐"
    success = push_to_wechat(title, content)

    if not success:
        print("⚠️  推送失败，但脚本运行成功")
    print(f"\n✅ 完成")


if __name__ == '__main__':
    main()
