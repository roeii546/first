# Definition of Done for v1 — Sales Call Analyzer

**מקור:** סריקת עומק (Opus, read-only) על קוד+מסמכים, 2026-06-09. נשמר ל-first כדי שלא יאבד.
**הערה:** אין `.git` ב-checkout שנסרק, אז הענפים `master`/`assistant` לא נבדקו ישירות — הממצאים מול קבצי ה-working tree (לפי DEV-STATUS = ענף `assistant`, האחרון, הלא-מודחף).

---

## 0. תקציר מנהלים

המוצר עשיר בפיצ'רים אבל **לא בטוח למכירה כ-self-serve היום**:

1. **נתיב הכסף פרוץ ביסודו.** webhook של Tranzila בלי אימות חתימה, בלי idempotency, סומך על שדה `contact` שניתן לשליטה ע"י תוקף כדי לזהות את המשלם, ה-frontend לא מאשר הצלחה, והמחירים/קרדיטים בדף התמחור לא תואמים ל-backend. בנוסף — **RLS מאפשר לכל משתמש מחובר לתת לעצמו קרדיטים בלי הגבלה** (מדיניות UPDATE על profiles בלי הגבלת עמודה), מה שמרוקן מתוכן את כל כלכלת הקרדיטים עד שזה מתוקן.
2. **ה-RPC `deduct_credit` לא קיים באף migration ברrepo** — רק בהערה. ה-backend קורא לו ו**fails open** על כל שגיאה. כלומר בפרודקשן כנראה אף פעם לא מנוכה קרדיט (או 500 ועוקפים).
3. **Concurrency ~2-4**, by design (job store בזיכרון + קובץ מלא ב-RAM), מוצמד ל-worker יחיד.
4. **פיצ'ר הצוותים מת** (מפנה ל-`role`/`manager_id`/`team_invites` שלא קיימים), וגרוע מכך — ה-lookups המתים רצים על ה-hot path של *כל* ניתוח ו*כל* קריאת קרדיט.
5. **ffmpeg/ffprobe הם תלות ריצה קשה** של ה-analyzer בלי קונפיג התקנה ל-Railway — אם חסר, תמלול/דחיסה נכשלים או מתדרדרים בשקט.
6. **הטסטים תיאטרון**, וה-CI לא מבצע deploy.

מסלול ה"concierge השבוע" **לא** דורש לתקן self-serve — רק שאדמין יוכל לזכות קרדיטים (עובד, `admin.py`) ושניתוח ירוץ אמין למשתמש בודד.

---

## 1. מצאי תקלות

חומרה: **P0** (חוסם/מאבד כסף או שובר ליבה) · **P1** (פיצ'ר שבור) · **P2** (איכות) · **P3** (מינורי).
תגיות: **[SOLO]** self-serve סולו · **[SCALE]** עשרות במקביל · **[TEAMS]** מחוץ ל-v1 · **[POLISH]**.
מאמץ: **S** <2ש' · **M** חצי-יום–יום · **L** רב-יומי.

### A. נתיב תשלום
| # | כותרת | ראיה | חומ' | תג | מאמץ |
|---|---|---|---|---|---|
| A1 | אין אימות חתימת webhook (`import hmac` קיים ולא בשימוש) | payments.py:17,77-103 | P0 | SOLO | M |
| A2 | אין idempotency — Tranzila חוזר, אין dedupe/unique על `tranzila_ref` | payments.py:106-159 | P0 | SOLO | M |
| A3 | זהות המשלם נשלטת ע"י תוקף — `contact=user_id` ב-URL של ה-iframe, נסמך בעיוורון | payments.py:58,86,102 | P0 | SOLO | M |
| A4 | הוספת קרדיט לא אטומית (race/lost-update) | payments.py:117-137 | P0 | SOLO | S |
| A5 | ה-frontend לא מאשר רכישה — iframe בלי callback/polling | Pricing.tsx:220-240 | P0 | SOLO | M |
| A6 | אי-התאמת מחיר/קרדיטים UI מול backend (₪49/90/350·10/25/100 מול ₪99/199/349·10/25/50) | Pricing.tsx:13-76 / payments.py:27-31 | P0 | SOLO | S |
| A7 | מספר free-tier לא עקבי ב-3 מקומות (5 / 1 / 3) | Pricing.tsx:18, Analyze.tsx:33, supabase-setup.sql:11 | P1 | SOLO | S |
| A8 | אין refund בכישלון ניתוח — קרדיט נוכה בתחילת job, אם נכשל אבוד | main.py:198-200,65-68 | P0 | SOLO | M |
| A9 | iframe sandbox עם `allow-top-navigation` (clickjack) | Pricing.tsx:236 | P2 | SOLO | S |
| A10 | אין רשומת pending-order → webhook שאבד = הפסד שקט בלי reconciliation | payments.py:38-74 | P1 | SOLO | M |

### B. קרדיטים / הרשאות / מודל-נתונים
| # | כותרת | ראיה | חומ' | תג | מאמץ |
|---|---|---|---|---|---|
| B1 | `deduct_credit` RPC לא קיים (רק בהערה); backend+frontend קוראים לו | supabase-fixes-2026-04-06.sql:47-66, main.py:159-164 | P0 | SOLO | S(SQL) |
| B2 | בדיקת קרדיט fails OPEN — כל שגיאה/חוסר-קונפיג מחזיר allow | main.py:151,174 | P0 | SOLO | S |
| B3 | **RLS מאפשר self-grant קרדיטים** — UPDATE על profiles בלי הגבלת עמודה/WITH CHECK | supabase-setup.sql:87-88 | P0 | SOLO | M |
| B4 | משתמשים יכולים לזייף `credit_transactions` | supabase-setup.sql:111-112 | P1 | SOLO | S |
| B5 | עמודות `role`/`manager_id` חסרות — נקראות על כל ניתוח+קרדיט | main.py:127-137; חסר מ-setup.sql | P1 | TEAMS (מזהם hot path סולו) | M |
| B6 | `team_invites` + migration multi-rep לא קיימים | supabaseTeam.ts | P1 | TEAMS | M |
| B7 | וידאו "2 קרדיט" ב-UI, backend מנכה 1 | Analyze.tsx:437 / main.py:198 | P2 | SOLO | S |
| B8 | ניתוח אנונימי חסום רק ב-localStorage → compute חינם בעלות אמיתית | Analyze.tsx:33-39,134 | P1 | SOLO/SCALE | M |
| B9 | עמודת `edited_results` בשימוש frontend; אולי ב-migration שלא רץ | supabaseReports.ts:21,133-145 | P2 | SOLO | S |

### C. Concurrency / scale
| # | כותרת | ראיה | חומ' | תג | מאמץ |
|---|---|---|---|---|---|
| C1 | job store בזיכרון, worker יחיד (~2-4) | main.py:37-47, Procfile, railway.toml | P1 | SCALE | L |
| C2 | קובץ שלם ל-RAM (עד 300MB) לכל בקשה | main.py:221-227, analyzer.py:782-791 | P1 | SCALE | M |
| C3 | threads ללא pool/queue — N העלאות = N ניתוחים כבדים | main.py:248-253 | P1 | SCALE | M |
| C4 | cache `.clear()` thrash ב-1000 | auth.py:76-77 | P3 | SCALE | S |

### D. Deploy / CI / env / ops
| # | כותרת | ראיה | חומ' | תג | מאמץ |
|---|---|---|---|---|---|
| D1 | **ffmpeg/ffprobe לא מותקנים** — אין nixpacks/Aptfile; אם חסר ב-Railway, תמלול נכשל | analyzer.py:665-701,1029-1162 | P0 | SOLO | S |
| D2 | אין auto-deploy (deploy ידני) | .github/workflows/ci.yml, DEV-STATUS:81 | P1 | POLISH | M(מייסד) |
| D3 | URL של Supabase מקודד-קשיח | admin.py:12 | P2 | POLISH | S |
| D4 | Telegram bot ב-lifespan — token רע עלול לשבור startup | main.py:77-81 | P2 | SCALE | S |
| D5 | CORS להידוק בפרוד | main.py:87-100 | P3 | POLISH | S |
| D6 | `_decode_unverified` JWT-bypass helper מת/מסוכן | auth.py:103-112 | P3 | POLISH | S |

### E. טסטים / נכונות / איכות
| # | כותרת | ראיה | חומ' | תג | מאמץ |
|---|---|---|---|---|---|
| E1 | אין טסטים לנתיב-כסף | src/test/example.test.ts, e2e/*.spec.ts | P1 | SOLO | M |
| E2 | E2E מיושנים/סותרים קוד (title `/CallSense/`) | e2e/core-flow.spec.ts:5 | P2 | POLISH | S |
| E3 | פער כיסוי זנב-תמליל (~85ש') — איבד התחייבות אמיתית | analyzer.py:1005-1018, BUG-007 | P2 | POLISH | M |
| E4 | אי-התאמת comment/code ב-`_plan_chunks` | analyzer.py:1062-1076 | P3 | POLISH | S |
| E5 | endpoint חושף raw exception ללקוח | main.py:268, payments.py:92 | P3 | POLISH | S |
| E6 | `generate-metric` מוריד אודיו מלא ל-RAM | main.py:477-494 | P2 | SCALE | M |

---

## 2. שדרוג ה-concurrency (≈3 → בנוחות עשרות)

המטרה עשרות, לא אלפים. שינויים מינימליים לפי תלות:

1. **העלאה ישירה ל-storage (מסיר את תקרת ה-RAM). [M]** — frontend מעלה ל-Supabase Storage *קודם*, ואז קורא ל-`/api/analyze` עם ה-path, לא הבייטים. מסיר C2.
2. **מצב-job ל-DB. [M]** — להשתמש בטבלת `analyses` הקיימת (כבר יש status). insert בהגשה, update ע"י worker, poll לפי id. מסיר את בעיית "poll מגיע ל-worker שלא ראה את ה-job", ומאפשר multi-worker. *דורש להעביר את כתיבת רשומת הניתוח מה-frontend ל-backend.*
3. **worker pool/queue חסום. [M-L]** — להחליף daemon-thread-per-request ב-pool קבוע (ThreadPoolExecutor / Redis-RQ). נותן back-pressure: 50 העלאות בתור במקום 50 בו-זמנית.
4. **להסיר את ההצמדה ל-worker יחיד. [S]** — רק אחרי 1-3. עד אז ההצמדה *נכונה*.

נטו ~**L**, בעיקר עיצוב-מחדש של מי-כותב-את-רשומת-הניתוח.

---

## 3. תוכנית ממוינת ל-v1

### מסלול A — מכירת concierge השבוע (ימים)
מטרה: לקוח אחד שמוגש ע"י המייסד יכול לקנות (ידנית) ולממש קרדיטים אמין. **בלי קוד self-serve.**
- **A.0 [מייסד]** לוודא env ב-Railway (GEMINI/ELEVENLABS/GROQ/SUPABASE keys, ADMIN_TOKEN, FRONTEND_URL).
- **A.1 [מייסד, SQL]** להריץ את `deduct_credit` (להוציא מהערה ב-supabase-fixes:53-66) + indexes. חוסם B1/B2.
- **A.2 [DEV]** לתקן B3 (RLS self-grant) + B4 — מדיניות שאוסרת שינוי `credits_remaining`, או לנתב כל שינוי דרך RPC SECURITY DEFINER. **בלי זה כל קרדיט חינם.**
- **A.3 [DEV]** B2: להפוך את בדיקת הקרדיט ל-fail-closed.
- **A.4 [מייסד]** לוודא ffmpeg ב-Railway (D1); אם חסר → [DEV] nixpacks.toml עם ffmpeg.
- **A.5 [מייסד]** `POST /api/admin/add-credits` (עובד) לזכות לקוח; להריץ ניתוח e2e בדפדפן ולאשר.

~2-4 ימים, חסום בעיקר על גישת-env של המייסד + אישור e2e.

### מסלול B — רכישת self-serve (1-2 שבועות) — תלוי ב-A
- **B.1 [מייסד]** Tranzila terminal, **סוד חתימת webhook** + לאמת את שמות השדות האמיתיים שטרנזילה שולחת (השמות בקוד הם ניחוש).
- **B.2 [DEV]** בנייה מחדש של נתיב-הכסף: A1 (חתימה), A3 (זהות מ-token חתום-שרת, לא `contact` מהלקוח), A2+A4 (idempotency + הוספה אטומית דרך RPC), A10 (pending-order).
- **B.3 [DEV]** A5: אישור ב-frontend (polling + UI הצלחה/כישלון); A9: להסיר allow-top-navigation.
- **B.4 [DEV]** A6/A7/B7: ליישר מחירים/קרדיטים/free-tier למקור-אמת אחד (`/api/payments/packs` קיים).
- **B.5 [DEV]** A8: refund בכישלון.
- **B.6 [DEV]** B8: לחסום ניתוח אנונימי בצד-שרת.
- **B.7 [DEV]** E1: טסטים לנתיב-כסף; לתקן E2.
- **B.8 [מייסד]** רכישת-כרטיס אמיתית e2e בדפדפן; אישור.

~1-2 שבועות, נשלט ע"י B.2/B.3 ולולאת-אימות Tranzila עם המייסד.

### מסלול C — scale לעשרות (1-2 שבועות, מקבילי ל-B)
- **C.1 [מייסד]** ליצור bucket `audio-uploads` + RLS (כרגע רק בהערה).
- **C.2 [DEV]** שלבי §2.
- **C.3 [DEV]** D4 (Telegram לא-פטאלי); E6 (stream במקום RAM).
- **C.4 [מייסד/DEV]** להסיר הצמדת worker-יחיד; load-test עשרות.

### מסלול D — היגיינת deploy (ימים, בעיקר מייסד)
- **D.1 [מייסד]** גישה לריפו; לחבר auto-deploy ל-Vercel/Railway.
- **D.2 [DEV]** D3/D5/D6.

**מחוץ ל-v1 (לדחות):** B5/B6 וכל [TEAMS] — אבל [DEV] כן ינטרל את ה-lookups המתים על ה-hot path הסולו (S, ערך גבוה).

---

## 4. מה רק המייסד יכול לעשות
מפתחות/service-role ב-Railway; Tranzila terminal + סוד חתימה + אימות שמות-שדות אמיתיים; להריץ את ה-SQL (deduct_credit RPC, תיקון RLS, indexes); ליצור bucket storage + policies; לוודא ffmpeg; גישה לריפו + auto-deploy; אישור e2e בדפדפן (ניתוח אמיתי + רכישה אמיתית).

## מה ה-DEV (Claude) עושה
כל תיקוני הקוד/SQL: מדיניות RLS, fail-closed, בניית webhook (חתימה/idempotency/זהות), אישור רכישה ב-frontend, יישור מחירים, refund, חסימת אנונימי בצד-שרת, refactor concurrency, טסטים לנתיב-כסף, היגיינת deploy, nixpacks ffmpeg.

---

## 5. לא-ידועים כנים (דורש env חי / הרצה)
1. האם `deduct_credit` קיים ב-DB החי? (אולי הורץ ידנית). בדיקה: `SELECT routine_name FROM information_schema.routines WHERE routine_name='deduct_credit';`
2. האם `role`/`manager_id`/`team_invites`/`edited_results` קיימים חי? בדיקה: `\d profiles`, `\dt`.
3. האם ffmpeg ב-image של Railway? בדיקה: `ffmpeg -version` ב-shell.
4. צורת ה-webhook האמיתית של Tranzila + האם בכלל יש חתימה? בדיקה: עסקת-בדיקה אמיתית.
5. תקרת concurrency אמיתית (~2-4 זה אומדן). בדיקה: load-test.
6. האם CI ירוק (E2E נראים מיושנים). בדיקה: GitHub Actions.
7. מצב RLS החי מול setup.sql (אולי נסחף). בדיקה: Supabase dashboard → policies; לאשר אם B3 באמת ניתן-ניצול חי.
8. מצב הענפים `master`/`assistant` (אין .git ב-checkout). בדיקה: clone + git log/diff.

**שורה תחתונה:** מסלול A (concierge) במרחק ימים, בעיקר חסום-מייסד. self-serve (B) ו-scale (C) ~1-2 שבועות כל אחד, מקבילים — אך שניהם חסומים עד שהמייסד מריץ את ה-SQL (deduct_credit + תיקון RLS) ומאמת את חוזה ה-webhook של Tranzila + ffmpeg.
