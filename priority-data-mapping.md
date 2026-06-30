# מיפוי נתונים מ-Priority עבור מערכת MBI (Read-Only)

> מסמך אפיון לצוות Priority — אילו טבלאות ושדות מערכת MBI צריכה **לקרוא** מ-Priority דרך ה-API.
> עודכן: 2026-06-30 · גרסה 1.0

---

## 1. מבוא ומטרה

מערכת MBI (ניהול בקשות, מכרזים, ספקים, תמחור ומלאי) צריכה למשוך נתונים מ-Priority ERP.
מסמך זה מפרט בדיוק אילו ישויות (טבלאות) ושדות נדרשים, כדי שצוות Priority יוכל לבנות את ה-API (OData) בהתאם.

**עקרונות מנחים:**

- 🔒 **קריאה בלבד (read-only).** למערכת אין ולא תהיה שום פעולת כתיבה/עדכון/מחיקה ל-Priority. בארכיטקטורת הקוד אין אף מתודת write — רק `GET`.
- 📋 שמות הטבלאות והשדות שלהלן הם **ההנחות הנוכחיות של MBI** (מבוססות על שמות סטנדרטיים ב-Priority). כל שם מסומן כ**"לאישור/תיקון"** — נא לאשר שהם נכונים ל-tenant שלכם או לתת את השמות המדויקים.
- 🔁 המערכת מושכת לפי דרישה (on-demand) וגם בריענון מתוזמן (ראו §7).

---

## 2. עקרונות חיבור (לאישור צוות Priority)

| נושא | מה שמומש/מונח כיום ב-MBI | לאישור/השלמה |
|------|---------------------------|---------------|
| פרוטוקול | OData v4 (REST) | ✔ נתמך? |
| Base path | `{base_url}/odata/Priority/{company}/{Entity}` | ✔ תקין? |
| שם ה-company | `mbi` (קשיח בקוד כרגע) | ❓ מהו שם ה-company המדויק? |
| אימות | HTTP Basic Auth (user + password) | ❓ Basic / API-Key / OAuth? |
| פרמטרים נתמכים | `$top`, `$orderby`, `$filter`, `$expand` | ✔ נתמכים? |
| פורמט תאריך | ISO-8601 (`YYYY-MM-DD`), השוואות ללא גרשיים (`CURDATE ge 2026-01-01`) | ✔ תקין? |
| Pagination | כיום `$top` בלבד (עד 1000) | ❓ יש `$skip` / server-side paging? |
| גישת רשת | מהשרת הפנימי בלבד (VPC/VPN) | ❓ נדרש allow-list של IP? |
| קידוד | UTF-8 (כולל עברית בשדות תיאור) | ✔ |

---

## 3. הטבלאות הנדרשות — ליבה (כבר ממומש בקוד)

חמש ישויות נקראות כיום דרך `/api/priority/*`. לכל אחת — העמודות הנדרשות והשדה שאליו הן ממופות אצלנו.

### 3.1 לקוחות — `CUSTOMERS`
מטרה: רשימת לקוחות (קופות חולים, בתי חולים, מוסדות), תנאי תשלום, אשראי.
`$orderby=CUSTNAME` · חיפוש: `contains(CUSTNAME,'…') or contains(CUSTDES,'…')`

| עמודה ב-Priority | שדה ב-MBI | משמעות |
|------------------|-----------|---------|
| `CUSTNAME` | priority_id | מזהה לקוח |
| `CUSTDES` | name | שם הלקוח |
| `CONTACTPERSON` | contact_name | איש קשר |
| `EMAIL` | email | דוא"ל |
| `PHONE` | phone | טלפון |
| `STATEA` | city | עיר |
| `COUNTRYNAME` | country | מדינה |
| `PAYDES` | payment_terms | תנאי תשלום |
| `CREDITLIMIT` | credit_limit | מסגרת אשראי |
| `CODE` | currency | מטבע |
| `INACTIVE` | status | פעיל/לא פעיל (Y=לא פעיל) |

### 3.2 ספקים — `SUPPLIERS`
מטרה: רשימת ספקים (יצרני API/אקסיפיינטים/אריזה), תנאי תשלום, מטבע.
`$orderby=SUPNAME` · חיפוש: `contains(SUPNAME,'…') or contains(SUPDES,'…')`

| עמודה ב-Priority | שדה ב-MBI | משמעות |
|------------------|-----------|---------|
| `SUPNAME` | priority_id | מזהה ספק |
| `SUPDES` | name | שם הספק |
| `CONTACTPERSON` | contact_name | איש קשר |
| `EMAIL` | email | דוא"ל |
| `PHONE` | phone | טלפון |
| `STATEA` | city | עיר |
| `COUNTRYNAME` | country | מדינה |
| `PAYDES` | payment_terms | תנאי תשלום |
| `CODE` | currency | מטבע |
| `INACTIVE` | status | פעיל/לא פעיל |

### 3.3 מלאי — `LOGPART`
מטרה: יתרות מלאי לפי מחסן, כמות שמורה, נקודת הזמנה — בסיס לתחזיות חוסר/חידוש.
חיפוש: `LOCNAME eq 'WH-MAIN'` ו/או `contains(PARTNAME,'…') or contains(PARTDES,'…')`

| עמודה ב-Priority | שדה ב-MBI | משמעות |
|------------------|-----------|---------|
| `PARTNAME` | part_number | מק"ט |
| `PARTDES` | name | תיאור הפריט |
| `LOCNAME` | warehouse | קוד מחסן |
| `TBALANCE` | on_hand | יתרה כוללת במלאי |
| `ORDERS` | reserved | שמור להזמנות |
| *(מחושב)* | available | `TBALANCE − ORDERS` |
| `UNITNAME` | unit | יחידת מידה |
| `MINBAL` | reorder_point | מינימום/נקודת הזמנה |
| `LASTRECDATE` | last_receipt_date | תאריך קליטה אחרון |

### 3.4 קטלוג פריטים — `PART`
מטרה: כרטיס פריט בסיסי (מק"ט, CAS, קטגוריה, זמן אספקה).
`$orderby=PARTNAME` · חיפוש: `contains(PARTNAME,'…') or contains(PARTDES,'…')`

| עמודה ב-Priority | שדה ב-MBI | משמעות |
|------------------|-----------|---------|
| `PARTNAME` | part_number | מק"ט |
| `PARTDES` | name | תיאור |
| `SPEC2` | cas_number | מספר CAS |
| `FAMILYDES` | category | משפחת מוצר |
| `UNITNAME` | unit | יחידת מידה |
| `MINQUANT` | min_order_qty | כמות מינימום להזמנה |
| `LEADTIME` | lead_time_days | זמן אספקה (ימים) |
| `INACTIVE` | status | פעיל/לא פעיל |

> ⚠️ זהו כרטיס פריט **בסיסי** (8 שדות). מודל הפריט המלא של MBI דורש הרבה יותר — ראו §4.

### 3.5 הזמנות — `ORDERS` + `ORDERITEMS_SUBFORM`
מטרה: היסטוריית הזמנות + שורות, לניתוח מכירות, ביצועי ספקים, וגזירת היסטוריית מחירים (§5).
`$expand=ORDERITEMS_SUBFORM` · `$orderby=CURDATE desc` · סינון: `CUSTNAME eq '…'`, `CURDATE ge 2026-01-01`

| עמודה ב-Priority (כותרת) | שדה ב-MBI | משמעות |
|--------------------------|-----------|---------|
| `ORDNAME` | order_number | מספר הזמנה |
| `CUSTNAME` | customer_id | מזהה לקוח |
| `CUSTDES` | customer_name | שם לקוח |
| `CURDATE` | order_date | תאריך הזמנה |
| `STATDES` | status | סטטוס |
| `CODE` | currency | מטבע |

| עמודה ב-Priority (שורה) | שדה ב-MBI | משמעות |
|-------------------------|-----------|---------|
| `PARTNAME` | part_number | מק"ט |
| `PARTDES` | name | תיאור |
| `TQUANT` | qty | כמות |
| `UNITNAME` | unit | יחידה |
| `PRICE` | unit_price | מחיר ליחידה |
| `QPRICE` | total | סך השורה |

---

## 4. הרחבה נדרשת: כרטיס פריט מלא (Item Master) — **הפער הגדול ביותר**

מודל הפריט של MBI (`Item`, ~1,800 SKU) דורש **הרבה מעבר** ל-8 השדות של `PART` (§3.4). כיום הנתונים האלה נטענים מ-Excel (`כרטיס פריט-20.04.26.xlsx`, ~120 עמודות). **בייצור הם צריכים להגיע מ-Priority.**

**הבקשה לצוות Priority:** לחשוף את כל שדות טופס כרטיס הפריט (או תת-טופס ייעודי). להלן 26 שדות הליבה שהמערכת כבר משתמשת בהם; נא לספק גם את **רשימת העמודות המלאה** של כרטיס הפריט (~95 עמודות נוספות יישמרו אצלנו כ-metadata).

| # | שדה (כותרת בכרטיס הפריט) | שדה ב-MBI | סוג |
|---|---------------------------|-----------|-----|
| 0 | מק"ט | sku | טקסט (מזהה) |
| 1 | תאור | description_he | טקסט |
| 2 | תאור לועזי | description | טקסט |
| 3 | פעיל/לא פעיל | status | טקסט |
| 4 | שם ספק | supplier_name | טקסט |
| 5 | משפחת מוצר | product_family | טקסט |
| 6 | צורת מינון (dosage form) | dosage_form | טקסט |
| 7 | אופן מתן (administration) | form_of_administration | טקסט |
| 8 | חוזק (strength) | strength | טקסט |
| 9 | סטטוס רישום (29ג/רשום) | registration_status | טקסט |
| 10 | ריכוז | concentration | טקסט |
| 11 | נפח | volume | טקסט |
| 12 | מחיר קניה אחרון | last_purchase_price | מספר |
| 13 | חומר פעיל | active_ingredient | טקסט |
| 14 | יצרן | manufacturer | טקסט |
| 15 | מספר רישום | registration_number | טקסט |
| 16 | מוסדי/פרטני | mosdi_partani | טקסט |
| 17 | תאריך כניסה אחרון | last_entry_date | תאריך |
| 18 | גודל אריזה | pack_size | טקסט |
| 19 | קוד PIP | pip_code | טקסט |
| 20 | טמפרטורת אחסון | storage_temperature | טקסט |
| 21 | מצריך banding | requires_banding | בוליאני |
| 22 | מצריך תווית (label) | requires_label | בוליאני |
| 23 | ציטוטוקסי (cyto) | is_cyto | בוליאני |
| 24 | סם מבוקר | is_controlled_drug | בוליאני |
| 25 | הערות | notes | טקסט |

> שדה נגזר: `is_29c` = true אם `registration_status == "29c"`. מטבע מחיר הקנייה: `last_purchase_currency`.

---

## 5. היסטוריית מחירי ספק (SupplierPrice)

המערכת מנהלת היסטוריית מחירים לכל פריט/ספק (להשוואת מחירים, הצעת מחיר, חישוב מרווח). אפשר **לגזור אותה מהזמנות הרכש** ב-Priority — אין צורך בטבלה ייעודית.

**בקשה:** לוודא ש-`ORDERS`/`ORDERITEMS_SUBFORM` (או טבלת רכש מקבילה) חושפים, להזמנות **רכש מספקים**: מק"ט (`PARTNAME`), מחיר יחידה (`PRICE`), מטבע (`CODE`), תאריך (`CURDATE`), ומזהה הספק (`SUPNAME`). מתוך אלה תיבנה היסטוריית `SupplierPrice` (item, price, currency, valid_from, supplier).

❓ לאישור: האם היסטוריית רכש מספקים חשופה ב-OData עם מחירי יחידה? באיזו ישות?

---

## 6. נתונים שאינם מגיעים מ-Priority (לתיאום ציפיות)

כדי שלא תכינו אותם לחינם — הדומיינים הבאים מגיעים ממקורות אחרים, **לא** מ-Priority:

| דומיין | מקור אמיתי |
|--------|-------------|
| קטלוג תרופות משב"ר (מחירוני מקסימום, רשומות + 29ג) | קבצי משרד הבריאות (Excel שנתי) |
| מודיעין שוק / מתחרים (Market Intelligence) | מקור BI חיצוני (Generics big data) |
| משלוחים נכנסים (Friday shipment, batch/expiry) | קובץ Excel / EDI ממחסן ה-UK |
| אישורי יבוא 29ג (טפסי MoH) | טפסי PDF סרוקים שמוזנים ידנית |

> אם בעתיד יוחלט שחלקם יזרמו דרך Priority (למשל משלוחים מתנועות מלאי) — זה תיאום נפרד.

---

## 7. תדירות ריענון מוצעת

| ישות | תדירות מוצעת | נימוק |
|------|---------------|-------|
| `CUSTOMERS` | יומי | משתנה לאט |
| `SUPPLIERS` | יומי | משתנה לאט |
| `PART` / כרטיס פריט | שבועי (+ on-demand) | קטלוג יציב |
| `LOGPART` (מלאי) | כל 2–4 שעות | רגיש לזמן (תחזיות חוסר) |
| `ORDERS` | יומי | ארכיון/אנליטיקה |

---

## 8. שאלות פתוחות לצוות Priority

1. **שם ה-company** ב-path — האם `mbi` נכון, או אחר?
2. **שיטת אימות** — Basic Auth / API-Key / OAuth? ומהם הקרדנציאלס לסביבת הקריאה?
3. **שמות שדות מותאמים** — לאשר/לתקן: CAS ב-`SPEC2`? קטגוריה ב-`FAMILYDES`? נקודת הזמנה ב-`MINBAL`? יתרה ב-`TBALANCE`, שמור ב-`ORDERS`?
4. **כרטיס פריט מלא** — לספק את רשימת **כל** העמודות של טופס כרטיס הפריט (השמות המדויקים), כדי שנמפה את 95 השדות הנוספים (§4).
5. **היסטוריית רכש/מחירי ספק** — באיזו ישות, והאם מחירי יחידה חשופים (§5)?
6. **Pagination ומגבלות** — `$skip` נתמך? מהי מגבלת `$top` המקסימלית? rate-limit?
7. **גישת רשת** — נדרש allow-list של כתובת ה-IP של שרת MBI (פנימי, מאחורי VPN)?
8. **שדות נוספים אפשריים** — האם יש ב-`CUSTOMERS`/`SUPPLIERS` שדות שימושיים נוספים (מס' ח.פ/ע.מ, כתובת מלאה, סיווג לקוח) שכדאי לחשוף?

---

## 9. טבלת-על

| דומיין | ישות Priority | תדירות | סטטוס |
|--------|---------------|---------|--------|
| לקוחות | `CUSTOMERS` | יומי | ✅ ממומש |
| ספקים | `SUPPLIERS` | יומי | ✅ ממומש |
| מלאי | `LOGPART` | 2–4 שעות | ✅ ממומש |
| הזמנות + שורות | `ORDERS` + `ORDERITEMS_SUBFORM` | יומי | ✅ ממומש |
| קטלוג בסיסי | `PART` | שבועי | ✅ ממומש |
| כרטיס פריט מלא (1,800 SKU) | `PART` / טופס כרטיס פריט מורחב | שבועי | 🟡 הרחבה נדרשת (§4) |
| היסטוריית מחירי ספק | `ORDERS`/רכש | יומי | 🟡 הרחבה נדרשת (§5) |
| קטלוג תרופות משב"ר | — | — | ⬜ לא מ-Priority |
| מודיעין שוק | — | — | ⬜ לא מ-Priority |
| משלוחים נכנסים | — | — | ⬜ לא מ-Priority |
| אישורי 29ג | — | — | ⬜ לא מ-Priority |

---

*מסמך זה נגזר ממצב הקוד בפועל (`backend/app/services/priority_service.py`, `backend/app/models/models.py`, `backend/scripts/etl_*.py`). כל שמות הטבלאות/השדות ב-Priority טעונים אישור צוות Priority.*
