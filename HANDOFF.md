# HANDOFF — ICT Super Engine V5 — حالة المشروع الكاملة
# آخر تحديث: 2026-07-22

---

## 1. ما هو المشروع؟

محرك تداول ICT (Inner Circle Trader) للكريبتو سبوت — شراء فقط (Spot Buy Only).
المحرك مبني على مفاهيم:
- ICT (Inner Circle Trader) — Michael J. Huddleston
- Order Flow / CVD (Cumulative Volume Delta)
- Liquidity Sweeps / FVG / Order Blocks / Breaker Blocks
- Effort vs Result / Displacement / Market Structure Shift

**المبدأ الأساسي:** المحرك يشتري سبوت فقط. لا شورت. لا فيوتشرز. لا رافعة.

---

## 2. تاريخ الشات — ماذا فعلنا من الأول لهلق

### المرحلة 1: فهم المشكلة الأصلية
- المحرك القديم (V4) كان "حافظ مش فاهم" — يطبق شروط ثابتة بدون فهم سياق السوق
- كان يدخل بعدد صفقات قليل جدًا أو لا يدخل إطلاقًا
- كان يخسر بسبب stops ثابتة و targets غير واقعية

### المرحلة 2: بناء V5 — النواة الجديدة
بنينا `engine_v5/` من الصفر بمعمارية مختلفة تمامًا:

| الملف | الوظيفة |
|---|---|
| `engine_v5/core.py` | الميكروستركشر: CVD, absorption, sweeps, displacements, FVGs |
| `engine_v5/strategy.py` | الاستراتيجية: entry logic, stop, targets, session filter |
| `engine_v5/backtest.py` | الباكتست bar-by-bar بدون lookahead |
| `engine_v5/run_paper_signals.py` | Paper trading / إشارات بدون تنفيذ |

### المرحلة 3: اختبار على بيانات حقيقية
- اختبرنا على BTCUSDT 1m من Coinbase (شهر كامل)
- اختبرنا على تيستات تيكات مخصصة من المستخدم (user_ticks2.csv)
- اكتشفنا مشاكل مجهرية وفصلناها في تقارير

### المرحلة 4: الإصلاحات المجهرية
- أصلحنا Sweep Detection (كان يفوّت micro sweeps)
- أصلحنا Premium/Discount filter (كان يرفض depth=0.00 دائمًا)
- أصلحنا Target logic (كان لا يجد أهداف فوق الدخول)
- أصلحنا Stop logic (كان tight جدًا أو بعيد جدًا)

---

## 3. المعمارية الحالية — كيف يعمل المحرك

### 3.1 Session Filter
```python
if session not in ("LONDON_AM", "NY_AM", "NY_PM"):
    return  # لا يتداول خارج جلسات لندن/نيويورك
```

### 3.2 Sweep Detection (بعد الإصلاح)
```python
significant = (
    (prior_low - b.low) >= max(0.02 * atr, 1e-9)   # penetration صغير كافٍ
    or reclaim_strength >= 0.60                      # أو reclaim قوي
    or b.volume >= 1.15 * recent_vol                 # أو حجم أعلى من المتوسط
)
```

### 3.3 Premium/Discount (بعد الإصلاح)
```python
if disp_strength >= 6.0 and disp_cvd_norm >= 0.15:
    min_depth = 0.00   # displacement قوي جدًا → يدخل حتى من القمة
elif disp_strength >= 4.0:
    min_depth = 0.05
elif disp_strength >= 2.0:
    min_depth = 0.12
```

### 3.4 Stop Logic (بعد الإصلاح)
```python
flexible_stop = max(
    fvg["low"] - 0.20 * atr,       # تحت FVG
    ref_entry - 0.50 * atr,         # أو تحت الدخول بـ 0.5 ATR
    stop,                            # أو الستوب الأصلي
)
```

### 3.5 Target Logic (بعد الإصلاح)
```python
targets = [
    displacement_high,              # قمة الـ displacement
    displacement_high + 0.27*range, # projection 0.27
    price + 0.50*range,             # compressed projection
    BSL_pools,                      # سيولة فوق
    bearish_FVGs,                   # FVG هابطة
]
# fallback: أفضل هدف بـ net RR >= session_rr - 1.0
```

### 3.6 Entry Modes
```python
MODES:
- DEEP_DEFENSIVE: edge=0%, ce=35%, breaker=65%
- NORMAL: edge=35%, ce=65%, breaker=0%
- URGENT: edge=70%, ce=30%, breaker=0%  (displacement قوي جدًا)
```

---

## 4. المشاكل الحالية — ما الذي لم يُحل بعد

### مشكلة 1: "no targets above" ما تزال الأكثر شيوعًا
- حتى بعد الإصلاح، 672–724 setup يُرفض بسبب "no targets above"
- السبب: التيستات المخصصة فيها أهداف قريبة جدًا أو غير موجودة
- **الحل المقترح:** إضافة target fallback أوسع (displacement high + projections)

### مشكلة 2: Stops ما تزال tight
- الصفقات تضرب ستوب بسرعة (BAR 1.0: 7 خطط، 0 wins)
- السبب: التيستات فيها pullbacks عنيفة والمحرك يخرج قبل أن تأخذ الحركة وقتها
- **الحل المقترح:** stop أبعد قليلاً + trailing stop بعد أول هدف

### مشكلة 3: CVD ما يزال proxy
- CVD محسوب من `(close - low) / (high - low)` وليس من tick data حقيقي
- **الحل المقترح:** ربط aggTrades من Binance/KuCoin لحساب CVD حقيقي

### مشكلة 4: Volume Bars ما تزال ثابتة الحجم
- حاليًا FixedVolumeBarAggregator (1/2/5 BTC)
- **الحل المقترح:** DynamicVolumeBarAggregator (موجود في الكود لكن غير مفعّل)

### مشكلة 5: لا يوجد Paper Trading فعلي
- المحرك يعمل backtest فقط
- **الحل المقترح:** تفعيل `run_paper_signals.py` مع WebSocket حقيقي

---

## 5. البيانات المتوفرة

| الملف | الوصف |
|---|---|
| `BTCUSD_1m_last_month_coinbase.csv` | BTC 1m شهر كامل من Coinbase |
| `BTCUSD_1m_last_month_kraken.csv` | BTC 1m من Kraken |
| `BTCUSD_real_ticks_latest_coinbase.csv` | تيكات حقيقية من Coinbase |
| `user_ticks2.csv` | تيست مخصص من المستخدم (198 tick) |
| `user_test_ticks.csv` | تيست أقصر من المستخدم |
| `BTCUSDT_15m_1month_detailed.json` | نتائج باكتست BTC 15m |

---

## 6. التقارير المكتوبة

| الملف | المحتوى |
|---|---|
| `Engine_V5_Build_Report_AR.md` | تقرير بناء V5 |
| `Engine_V5_Diagnosis_and_Flexible_Plan_AR.md` | تشخيص + خطة |
| `BTC_1Month_V5_Test_Report_AR.md` | اختبار BTC شهر |
| `User_Ticks_Test_Report_AR.md` | اختبار تيست المستخدم الأول |
| `User_Ticks2_Microscopic_Errors_AR.md` | تشريح مجهري لأخطاء التيست الثاني |

---

## 7. ما الذي يجب فعله في الشات الجديد

### الأولوية 1: إصلاح "no targets above"
- إضافة displacement high كـ target طبيعي
- إضافة projections (0.27, 0.50) كـ targets
- fallback: أفضل هدف بـ net RR >= 1.0

### الأولوية 2: إصلاح Stops
- stop أبعد قليلاً (0.5 ATR بدل 0.35)
- trailing stop بعد أول هدف
- لا stop ثابت — دائمًا نسبة من ATR

### الأولوية 3: تفعيل CVD حقيقي
- ربط aggTrades من Binance/KuCoin
- حساب CVD من tick data وليس proxy

### الأولوية 4: تفعيل Paper Trading
- WebSocket حقيقي
- إشارات بدون تنفيذ
- journal لكل إشارة

### الأولوية 5: Dynamic Volume Bars
- تفعيل DynamicVolumeBarAggregator
- حجم البار يتكيف مع السوق

---

## 8. كيف تفتح الشات الجديد وتكمل

انسخ هذا الملف (`HANDOFF.md`) + ملف `Engine_V5_Latest.zip` إلى الشات الجديد.

في الشات الجديد، اكتب:

> "عندي مشروع ICT Super Engine V5. هاد ملف الـ HANDOFF فيه كل شي. كمّل من محل ما وقفنا. الأولوية: إصلاح 'no targets above' + Stops + تفعيل CVD حقيقي."

---

## 9. ملاحظات مهمة

- المحرك حاليًا **ليس جاهزًا للتداول الحقيقي**
- هو في مرحلة **Paper Trading / Backtest**
- كل النتائج الحالية هي **backtest فقط** وليست نتائج تداول حقيقي
- التيستات المخصصة (user_ticks) هي بيانات اصطناعية وليست بيانات سوق حقيقية
- النتائج على البيانات الاصطناعية لا تعكس أداء السوق الحقيقي

---

## 10. الملفات في الـ ZIP

```
Engine_V5_Latest.zip
├── engine_v5/
│   ├── __init__.py
│   ├── core.py              ← الميكروستركشر
│   ├── strategy.py          ← الاستراتيجية
│   ├── backtest.py          ← الباكتست
│   ├── run_paper_signals.py ← Paper Trading
│   └── README.md
├── data/
│   ├── BTCUSD_1m_last_month_coinbase.csv
│   ├── user_ticks2.csv
│   └── BTCUSDT_15m_1month_detailed.json
├── reports/
│   ├── Engine_V5_Build_Report_AR.md
│   ├── Engine_V5_Diagnosis_and_Flexible_Plan_AR.md
│   ├── BTC_1Month_V5_Test_Report_AR.md
│   ├── User_Ticks_Test_Report_AR.md
│   └── User_Ticks2_Microscopic_Errors_AR.md
└── HANDOFF.md               ← هذا الملف
```
