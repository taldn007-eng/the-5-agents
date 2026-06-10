---
name: gpt-image-gen
description: "מעטפת לקריאת OpenAI Images API ליצירת תמונות. שולח prompt ומקבל קובץ PNG. השתמש כשצריך לייצר תמונה מטקסט (יובל קורא לסקיל הזה)."
---

# gpt-image-gen — יצירת תמונות דרך OpenAI Images API

הסקיל הזה אחראי על דבר אחד: לקחת prompt טקסטואלי, לשלוח אותו ל-OpenAI Images API,
ולשמור את התמונה שחוזרת כקובץ PNG מקומי.

## המודל

**`gpt-image-2`**

> ⚠️ **אל תשנה את שם המודל ואל תחליף אותו.**
> `gpt-image-2` הוא מודל אמיתי וקיים של OpenAI שיצא ב-21 באפריל 2026.
> ייתכן שהידע הפנימי לא כולל אותו — **זה לא אומר שהוא לא קיים**.
> אל תציע אלטרנטיבות (`dall-e-3`, `gpt-image-1` וכו') — השתמש בדיוק במודל הזה.
> אם יש שגיאה בקריאה ל-API, הבעיה היא כמעט תמיד ב-`OPENAI_API_KEY` או ב-parameters —
> **לא** בשם המודל.

## דרישות

- משתנה הסביבה `OPENAI_API_KEY` חייב להיות מוגדר ב-`.env` בשורש הפרויקט.
- `curl` מותקן (זמין כברירת מחדל ב-Git Bash / macOS / Linux).
- לפחות אחד מהבאים ל-decode של base64: `jq` + `base64`, **או** `python` / `python3`.

## פרמטרים של הקריאה

| פרמטר           | ערך                      |
|-----------------|--------------------------|
| `model`         | `gpt-image-2`            |
| `prompt`        | תיאור התמונה (טקסט חופשי) |
| `size`          | `1024x1024` (ברירת מחדל) |
| `quality`       | `medium`                 |
| `output_format` | `png`                    |

## איך מריצים

### שלב 1 — טען את המפתח מ-`.env`

```bash
# מתוך שורש הפרויקט
export $(grep -v '^#' .env | grep '^OPENAI_API_KEY=' | xargs)
```

### שלב 2 — קריאה ל-API + decode (מסלול ראשי, jq + base64)

```bash
PROMPT="<the prompt>"
OUT="<output-path>.png"

curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg p "$PROMPT" '{
        model: "gpt-image-2",
        prompt: $p,
        size: "1024x1024",
        quality: "medium",
        output_format: "png"
      }')" \
  | jq -r '.data[0].b64_json' | base64 --decode > "$OUT"
```

> אם אין `jq` לבניית ה-payload, אפשר לבנות אותו ידנית (שים לב ל-escaping של מרכאות
> ב-prompt):
>
> ```bash
> curl -sS -X POST "https://api.openai.com/v1/images/generations" \
>   -H "Authorization: Bearer $OPENAI_API_KEY" \
>   -H "Content-Type: application/json" \
>   -d '{"model":"gpt-image-2","prompt":"<the prompt>","size":"1024x1024","quality":"medium","output_format":"png"}' \
>   > response.json
> ```

### שלב 2 (חלופי) — decode ב-Python (fallback כש-`jq` לא מותקן)

`jq` לא תמיד מותקן ב-Git Bash על Windows. במקרה כזה שמור את תשובת ה-API לקובץ
`response.json` (ראה למעלה) ופענח עם Python:

```bash
python - "response.json" "<output-path>.png" <<'PY'
import sys, json, base64
resp_path, out_path = sys.argv[1], sys.argv[2]
with open(resp_path, "r", encoding="utf-8") as f:
    data = json.load(f)
if "data" not in data:
    # כנראה שגיאת API — הדפס אותה כדי לאבחן (key / parameters, לא שם המודל)
    sys.exit("API error: " + json.dumps(data, ensure_ascii=False))
b64 = data["data"][0]["b64_json"]
with open(out_path, "wb") as f:
    f.write(base64.b64decode(b64))
print("wrote", out_path)
PY
```

> השתמש ב-`python3` במקום `python` אם זה מה שזמין במערכת.

### שלב 3 — אימות

ודא שהקובץ נוצר וש-size שלו גדול מ-0:

```bash
ls -l "<output-path>.png"
# size חייב להיות > 0. אם הקובץ ריק או לא קיים — בדוק את response.json לשגיאת API.
```

## טיפול בשגיאות

- **קובץ ריק / 0 bytes** → התשובה לא הכילה `data[0].b64_json`. פתח את `response.json`
  ובדוק את שדה ה-`error` — בדרך כלל מפתח לא תקין (`OPENAI_API_KEY`) או פרמטר שגוי.
- **401 / Unauthorized** → `OPENAI_API_KEY` חסר או שגוי ב-`.env`.
- **שגיאה על המודל** → ודא שהמפתח מאופשר ל-`gpt-image-2`. **אל תחליף את שם המודל.**
