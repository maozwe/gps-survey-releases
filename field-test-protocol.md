# Field test protocol

The purpose of this document is narrow: turn "the tests pass" into "this works." Nothing in
GPS Survey has been run against a real receiver, a real RTK subscription, or a real orthophoto —
see the root `README.md`'s Limitations section for the full list of what is verified only in
software. This protocol is the checklist for the first time a real person takes this app into a
real field with a real receiver in hand.

Follow it in order — later steps assume earlier ones passed. Every step has an acceptance
criterion stated as a number, not an impression; where the app itself has no built-in check
(there usually is one), the number here is what to compare against by hand.

A Hebrew version of this checklist is at the end of this document.

## What to bring

- [ ] Android phone/tablet with GPS Survey installed (`minSdk` 26 / Android 8.0 or newer),
      battery charged, and enough free storage for logs.
- [ ] A GNSS/RTK receiver, its pole, and its own battery charged. If it is a u-blox unit you
      intend to configure yourself, know its serial port wiring (USB/UART) in advance.
- [ ] A **known control point** with published coordinates in your project's CRS (ITM by
      default) — ideally one you can independently check with a total station or a published
      geodetic control sheet. Without this, Part 4 below cannot run.
- [ ] A **tape measure** (for Part 3 — verifying the antenna-height arithmetic against reality,
      not against the app's own number).
- [ ] Whatever your correction source needs:
  - An active NTRIP subscription (host/port/mountpoint/username/password from your provider —
    see `docs/receiver-setup.md`), **or**
  - A second receiver, its own pole/tripod, and either a radio link or a mobile hotspot, if you
    are testing the local base/rover path instead.
- [ ] A small design/known-points file (CSV or XLSX, a handful of points) if you want to exercise
      the site-calibration known-point import.
- [ ] A USB cable and a computer with `adb` if you want to exercise the DXF or orthophoto path —
      see Part 9's note on why this needs developer assistance today.
- [ ] Something to record findings with (this document has a "what to record" section — use it
      verbatim, it is designed to be filled in as you go).

## Before you go

- [ ] Build and install a debug or release build (`docs/dev-setup.md`).
- [ ] Launch the app once with a network connection and grant every permission it asks for
      (Bluetooth, Location, Notifications, Camera) — the app can start most transports without
      Location, but BLE scanning below Android 12 silently returns nothing without it (a known
      gap — see the root README's Limitations).
- [ ] Create a test project: pick your project's coordinate system (ITM unless you have a
      specific reason otherwise) and geoid mode. If you have a real ILUM file from מפ״י, load it
      now (project editor → Geoid → File) — this is the one file-import path that works fully
      end to end today.
- [ ] Note the app version (`README.md`'s header) and your device's Android version — both go in
      every finding you record.

---

## Part 1 — the receiver connects

Follow `docs/receiver-setup.md` for your specific transport (Bluetooth SPP / USB-OTG / BLE / TCP)
and connect.

**Acceptance criteria:**

- [ ] The Receiver screen's status reads **מחובר / Connected** within a reasonable time
      (a few seconds for Bluetooth/TCP; USB/BLE should be near-instant once permission is
      granted).
- [ ] The satellites tab shows a non-zero, growing satellite count within about 30 seconds under
      open sky.
- [ ] The raw console tab shows live traffic (NMEA sentences, or `UBX-NAV-...` frames for a
      u-blox receiver) — confirms the app is actually parsing what the receiver sends, not just
      holding an idle socket open.
- [ ] If the connection drops, it visibly reconnects on its own (Bluetooth SPP and TCP only — USB
      and BLE do not auto-reconnect by design; you reconnect manually).

**If this fails**, see `docs/receiver-setup.md`'s troubleshooting section before continuing —
nothing past this point can be tested without a live connection.

## Part 2 — reaching RTK Fixed

With a correction source configured (`docs/receiver-setup.md`), watch the status bar under open
sky.

**Acceptance criteria — all four must hold before you trust a single measured point:**

| Field | Required value |
|---|---|
| Fix quality badge | **קבוע / Fixed** (RTK Fixed) |
| Satellites used | **≥ 6** |
| PDOP | **< 3** |
| Correction age | **< 10 s** (the app's own default gate; below this, Fixed is achievable — above it the app itself will demote or block a store) |

If the fix sits at **Float** and never converges to Fixed: move to more open sky, check that
corrections are actually arriving (bytes/second > 0 on the Corrections screen, RTCM message list
non-empty), and check the baseline to your reference station is not excessive (see
`docs/receiver-setup.md`'s troubleshooting section — this is the single most common field
problem and it has a short, specific list of causes).

**Record**, once Fixed is reached and held for at least 30 seconds: time-to-first-fix from
connection, and time-to-Fixed from first autonomous/DGPS fix. Neither has an app-enforced
acceptance number — record them as a baseline for this receiver/network combination.

## Part 3 — antenna height arithmetic vs. a tape measure

The app computes `groundHeight = receivedHeight − poleHeight − arpOffset` (the default
"phase centre" reference — see the session-start dialog's own worked-example text). This is the
single most safety-critical piece of arithmetic in the app: a wrong pole height does not fail
loudly, it silently biases every height in the whole job by exactly that amount.

- [ ] Extend the pole to a specific, round height (e.g. exactly 2.000 m) and **measure it with
      the tape measure independently of the pole's own markings**, if you have any doubt about
      them.
- [ ] Enter that pole height, and your receiver's documented ARP→APC offset (from its own data
      sheet — the app cannot know this for you), into the session-start dialog.
- [ ] Confirm the dialog's own worked-example line shows the arithmetic you expect:
      `received − pole − arp = ground`, with your actual numbers substituted (not the sample
      42.179/2.000/0.062 numbers from the app's own documentation).
- [ ] Measure a point, then check `groundH` against what a tape measure from a nearby fixed
      reference (a curb, a manhole rim — anything you can also independently level to) would
      suggest. This is a sanity check, not a substitute for Part 4.
- [ ] Change the pole height mid-session (Measure screen → "Change antenna height") to a
      **different** round number, confirm the app opens a **new session** rather than silently
      reinterpreting already-stored points, and confirm the point you stored before the change
      still shows the old antenna height in the points table.

**Acceptance criterion:** the reduction the dialog displays (pole + ARP offset, or pole alone,
depending on your receiver's `HeightReference`) matches your tape-measured pole height to within
your tape's own precision (a few millimetres) — not the app's number in isolation, the **tape's**
number. A discrepancy here is either a units mistake (cm entered as m, or vice versa — the app
rejects a pole height over 10 m or negative, but a plausible-looking wrong number like 1.5 m
typed as 15 would not be caught) or a wrong `HeightReference` choice for your receiver.

## Part 4 — coordinates against a known control point (site calibration)

This is where you find out whether "the tests pass" actually means "this measures correctly
outdoors, on this network, today." Read `docs/calibration.md` before this step if you have not —
it is the authoritative reference and has its own Hebrew section for exactly this moment.

- [ ] Occupy your known control point. Measure it the normal way (same quality gate as any other
      point — averaged, at least 30 epochs recommended for a control observation).
- [ ] Open the Calibration screen and enter (or import) the point's **published** coordinates.
      The screen shows the residual — published minus measured — **before anything is applied**.
- [ ] **Do not accept the calibration yet if you only have one control point.** Record the
      residual (ΔE, ΔN, ΔH, horizontal). A residual under a few centimetres, consistent with a
      known-epoch/monument-movement difference, is expected and normal — see the worked example
      in `docs/calibration.md`. A residual **over about 1 metre almost always means a mistake**
      (wrong point, wrong CRS, transposed digits), not a real site difference — stop and check
      before doing anything else.
- [ ] If you have (or can occupy) a **second** control point, do so, and repeat. With two points
      the model is a plan rotation+scale+translation and every residual becomes zero again by
      construction — this proves nothing about accuracy either. Turn on **"Hold scale at 1"** and
      re-check the residuals; this frees up one redundant observation that now says something
      real about the fit.
- [ ] If you have a **third** control point, this is where the numbers start to mean something:
      the fit becomes least-squares, with real degrees of freedom.

**Acceptance criteria for a 3+-point calibration:**

| Metric | Target | Interpretation if exceeded |
|---|---|---|
| RMS horizontal | **< 2 cm** for a small site | A single point 3× worse than the others is almost always *that point*, not the site — try disabling it and re-checking its residual against the surviving model |
| RMS vertical | Judge against your own control's known height quality | If your published heights come from a single benchmark, or the points lie roughly on a line, turn on **"Heights as one shift"** rather than trusting a fitted tilted plane |
| Degrees of freedom | **> 0** before trusting RMS as a real accuracy figure | Zero means the model reproduces the points exactly by arithmetic, which the app itself states outright on screen |

- [ ] Whether or not you activate the calibration, confirm the **raw** coordinate is still
      visible/exportable somewhere (points table, or the raw fields if you inspect an export) —
      this is the safety net the whole feature depends on.

## Part 5 — measuring

- [ ] Store a single-epoch point at RTK Fixed. Confirm every metadata field is populated: fix
      quality, σE/σN/σU, satellites used, PDOP/HDOP, correction age, antenna height, timestamp.
- [ ] Switch to averaged mode (default 5 epochs) and store a point on the **same physical spot**
      you just measured single-epoch. Compare the two — they should agree to within the
      receiver's own quoted RTK-Fixed accuracy (roughly 1–3 cm horizontal, per
      `docs/receiver-setup.md`'s accuracy table), not necessarily to the millimetre.
- [ ] Deliberately trigger each gate and confirm the app's behaviour matches:

| Trigger | Expected behaviour |
|---|---|
| Move to where the fix drops to Float/DGPS | An acknowledgement dialog appears every time you try to store; storing is still possible after confirming |
| Move to where the fix drops to Autonomous | Store is **blocked** unless you turn on the override switch, which shows a persistent red banner while active |
| Widen σ past 0.05 m horizontal / 0.08 m vertical (e.g. by degrading your antenna's sky view) | Store is blocked, always, with no override — the message states the actual σ and the limit |
| Let corrections age past 10 s (disconnect your NTRIP data briefly) | Store is blocked, always, with no override |

- [ ] Undo the last stored point (Measure screen) and confirm it disappears from the points
      table, and — if it was extending a line — that the line's vertex list is correctly rolled
      back too.
- [ ] Turn on line mode, measure three points, and confirm a line is created starting from the
      **second** point (the first point alone cannot start a line — this is intentional, not a
      bug) and grows with each subsequent point.

## Part 6 — stake-out

- [ ] Pick a target: a project point you already measured is the simplest first test. Confirm
      the navigation view's distance/azimuth/forward-right numbers make sense as you walk (walk
      a known direction and confirm "forward" and "right" track your actual movement, not a
      frozen or inverted value).
- [ ] Watch the direction arrow: confirm it visibly grows as you approach, and that its rotation
      **stops looking jittery** and instead freezes/hands over to the bullseye once you are
      inside the tightest tolerance ring (2 cm by default) — this is the behaviour
      `feature/stakeout/README.md` describes as unverified against a real receiver's actual noise
      characteristics; this is the test that verifies it.
- [ ] Confirm **"within tolerance"** (the haptic/audio cue, if enabled, and the bullseye colour
      change) fires only once you are inside the **smallest configured ring** (2 cm default), not
      the outer ones — the outer rings are a walk-in aid only.
- [ ] Store a staked point and confirm the resulting stake record shows the correct design vs.
      measured deviation (ΔE/ΔN/ΔH, horizontal distance).
- [ ] Try each of the other four target kinds if your project data supports it: typed
      coordinates, and chainage/offset along a project line, are testable with no external file.
      DXF-vertex and DXF-line targets need a `.dxf` file already present in the project's
      storage — see Part 9's note on why this needs developer assistance today, since there is no
      in-app way to place one there.

## Part 7 — peg mode

- [ ] Turn on peg mode ("יתד"). Stake to a target — confirm the app now asks for **Step 2:
      measure the ground point** and will not let you finish without it.
- [ ] Measure the ground point within the configured maximum distance (**0.5 m by default**) of
      the peg top. Confirm the finished record shows both points, the correct
      `pegAboveGround` (peg top height minus ground height), and a peg badge in the records list.
- [ ] Deliberately measure a ground point **further than 0.5 m** from the peg top. Confirm the
      app rejects it, names the actual measured distance and the limit, and lets you try again
      without losing the peg top.
- [ ] Start a peg, reach step 2, and press **Abandon** (or use the system back gesture, which
      should trigger the same confirmation rather than silently discarding the peg top). Confirm
      no stake record is created and the peg-top point is removed from the points table — a
      half-finished peg should leave **no trace**, by design.
- [ ] Export the as-built report (CSV, XLSX, and PDF — all three) and confirm the peg's design/
      peg/ground rows and pass/fail column match what the screen showed.

## Part 8 — import/export: what is actually testable today

The general import wizard and a generic point/line export screen do not exist in the shipped
app yet (`feature/io` is a navigation placeholder — see the root README's Limitations section).
Test what is actually reachable:

- [ ] **Geoid file**: pick a real GTX/ISG file (or a small hand-made one) via the project editor
      and confirm heights change appropriately once it is active.
- [ ] **Known-point import** (Calibration screen → "Import from a file"): import a small CSV or
      XLSX of control points with Hebrew names/descriptions and confirm they appear correctly in
      the known-points list, with Hebrew text intact.
- [ ] **As-built report export**: covered in Part 7.
- [ ] **Map-image export**: from the map screen's overflow menu, export a PNG and a PDF across at
      least two different extents (selected points, current view) and two DPI settings, with
      labels and legend on. Confirm both open correctly and — for the PDF — confirm any Hebrew
      point names/codes in the legend read right-to-left (see Part 9's PDF note; this is one of
      the two places that needs it).
- [ ] Confirm, and record, that there is **no way** in the running app to import a DXF file, a
      SHP file, a LandXML file, or a GeoTIFF orthophoto, or to export the point/line table to any
      format — this is expected, not a bug you found; it is already documented. Do not spend
      field time hunting for a button that is not there.

## Part 9 — the orthophoto and DXF paths (developer-assisted)

Because there is no in-app file picker for either path today, exercising them needs someone with
`adb` access to the device (a developer, or a field tester temporarily borrowing one), not just a
surveyor with the app installed. This is worth doing at least once before the app is put in
front of a real customer, since both paths are otherwise completely unverified on a device.

**DXF:**

- [ ] Convert or obtain a real Israeli DXF drawing (Hebrew layer names/text is the more
      interesting case, since that is what `core/dxf`'s own tests are built around).
- [ ] `adb push` it into the project's private storage under
      `files/projects/<project-id>/dxf/` (the project id is visible in the project's own file
      paths; a developer can find it via `adb shell run-as com.gpssurvey.app.debug` or by reading
      the app's database).
- [ ] Reopen the map screen for that project and confirm the DXF's layers render, with correct
      colours and Hebrew text, at the correct position.
- [ ] In Stake-out, pick the "DXF vertex" or "nearest point on a DXF line" target kind and
      confirm real vertices/points from the drawing are offered and stake correctly.

**Orthophoto:**

- [ ] Build an `.mbtiles` file with `tools/ortho-converter` (either from a real Israeli
      orthophoto, or with `--self-test`'s synthetic red/green/blue/yellow test image — the
      easiest way to visually catch an upside-down or mirrored result).
- [ ] Run `convert.ps1 --verify` on the output and confirm it reports the tile rows as correctly
      ordered **before** copying it to the device — this is the check that would have caught a
      known historical bug class (an upside-down basemap) before it ever reached a screen.
- [ ] `adb push` it into `files/projects/<project-id>/raster/`.
- [ ] Since there is no "add raster layer" button in today's build either, this currently needs a
      developer to insert the matching `layers` database row directly (`kind = RASTER`,
      `sourcePath` pointing at the pushed file) for the map to pick it up. Confirm, once that row
      exists, that the imagery renders **right-side-up** — this is the single most important
      visual check in this whole document, since an inverted orthophoto looks like a map and is
      the one failure mode a surveyor cannot catch by eye alone without a known landmark in the
      frame.

## Part 10 — Hebrew UI walkthrough

- [ ] Switch the app language to Hebrew (Settings → Language) and confirm the whole UI mirrors:
      icons, navigation, button order.
- [ ] Confirm the map itself, the compass, the scale bar, and every coordinate/number on screen
      **do not mirror** — these must stay geographically/numerically correct regardless of UI
      language (east is always right, "12.345" is never shown right-to-left).
- [ ] Read through a few screens specifically looking for a Hebrew sentence whose **word order**
      looks reversed — this project has a documented history of exactly this bug class (a
      translated sentence accidentally wrapped in the same LTR-forcing wrapper used for bare
      numbers) and it was fixed each time it was found by a human reading the actual screen, not
      by an automated check.
- [ ] Switch back to English and spot-check the same screens for a similarly garbled or
      untranslated string.

## What to record when something is wrong

For every finding, note:

- **What you were doing** (which part/step of this document).
- **What you expected** vs. **what happened**.
- **Device**: model, Android version.
- **Receiver**: model, firmware (Receiver screen shows this once connected via `MON-VER` for a
  u-blox unit), transport used.
- **Correction source**: which kind, and — for NTRIP — the provider and mountpoint (**never**
  record the password).
- **The exact numbers on screen** at the time: fix quality, σH/σV, PDOP, satellites used,
  correction age.
- **A screenshot**, if the issue is visual.
- **The raw console log**, if the issue might be a parsing/protocol problem: turn on
  "רישום לקובץ / Log to file" on the Receiver screen before reproducing, and attach the resulting
  file — the recorded stream is the single most useful artifact for diagnosing a protocol-level
  problem after the fact.

## Acceptance criteria — summary

| Check | Requirement |
|---|---|
| Fix quality before storing a point | RTK Fixed (Float/DGPS/PPS need explicit confirmation; worse is blocked) |
| Max horizontal σ | 0.05 m |
| Max vertical σ | 0.08 m |
| Max correction age | 10 s |
| Satellites used | ≥ 6 |
| PDOP | < 3 |
| Antenna height arithmetic | Matches independent tape measurement to a few mm |
| Single control point residual | Recorded, not acted on; > ~1 m means a mistake, not a real difference |
| 3+ point calibration RMS horizontal | < 2 cm for a small site |
| Peg ground-point distance | ≤ 0.5 m from the peg top |
| Stake-out "within tolerance" | Inside the 2 cm inner ring (defaults) |
| Orthophoto orientation | Right-side-up, checked visually with a known landmark or the converter's own asymmetric self-test image |
| PDF Hebrew text | Reads right-to-left, checked in a real PDF viewer |

---

## עברית — פרוטוקול בדיקת שטח

מסמך זה הוא הצ'קליסט להפיכת "הבדיקות עוברות" ל"זה עובד בשטח". שום דבר באפליקציה לא נבדק מול
מקלט אמיתי, מנוי RTK אמיתי או תצלום אוויר אמיתי — ראו את סעיף המגבלות ב-`README.md`. יש לעקוב
אחר הסדר — שלבים מאוחרים מניחים ששלבים מוקדמים הצליחו.

### מה להביא

- [ ] מכשיר אנדרואיד עם האפליקציה מותקנת, סוללה טעונה, ומקום פנוי ללוגים.
- [ ] מקלט GNSS/RTK, מוט, וסוללת המקלט טעונה.
- [ ] **נקודת ביקורת ידועה** עם קואורדינטות מפורסמות במערכת הקואורדינטות של הפרויקט (ITM
      כברירת מחדל) — רצוי כזו שניתן לאמת באופן עצמאי. בלי זה, שלב 4 למטה לא ניתן להרצה.
- [ ] **סרט מדידה** (לשלב 3 — בדיקת חשבון גובה האנטנה מול המציאות, לא מול המספר של האפליקציה).
- [ ] מה שמקור התיקונים שלכם דורש: מנוי NTRIP פעיל (מארח/יציאה/mountpoint/משתמש/סיסמה מהספק —
      ראו `docs/receiver-setup.md`), **או** מקלט שני + חצובה + חיבור רדיו/hotspot אם בודקים את
      מסלול בסיס-רובר מקומי.
- [ ] קובץ נקודות ידועות קטן (CSV או XLSX) אם רוצים לבדוק את ייבוא נקודות הכיול.
- [ ] כבל USB ומחשב עם `adb` אם רוצים לבדוק את מסלולי ה-DXF או האורתופוטו (ראו שלב 9 — דורש
      סיוע של מפתח כיום).

### לפני היציאה לשטח

- [ ] מתקינים גרסת debug או release (`docs/dev-setup.md`).
- [ ] מפעילים פעם אחת עם חיבור רשת ומעניקים את כל ההרשאות המבוקשות.
- [ ] יוצרים פרויקט בדיקה: בוחרים מערכת קואורדינטות (ITM) ומצב גאואיד. אם יש קובץ ILUM אמיתי
      ממפ״י — טוענים אותו כעת.

### שלב 1 — המקלט מתחבר

עוקבים אחר `docs/receiver-setup.md` לפי אמצעי התקשורת שלכם. **קריטריון קבלה**: הסטטוס מציג
**מחובר** תוך זמן סביר, מספר הלוויינים עולה תוך כ-30 שניות תחת שמיים פתוחים, וקונסול הגלם מציג
תעבורה חיה.

### שלב 2 — הגעה לפתרון קבוע (RTK Fixed)

**קריטריוני קבלה, כולם יחד**: תגית פתרון **קבוע**; **6 לוויינים** לפחות בשימוש; **PDOP מתחת ל-3**;
**גיל תיקונים מתחת ל-10 שניות**. אם הפתרון נשאר צף ולא הופך לקבוע — ראו את פתרון הבעיות ב-
`docs/receiver-setup.md`.

### שלב 3 — חשבון גובה האנטנה מול סרט מדידה

מודדים בעצמכם את גובה המוט עם סרט מדידה (לא סומכים רק על הסימונים על המוט), מזינים אותו יחד עם
היסט ה-ARP של המקלט (מדף הנתונים שלו) בתיבת הדו-שיח של תחילת הסשן, ובודקים שהנוסחה המוצגת
תואמת את מה שמדדתם. **קריטריון קבלה**: ההפרש הנוצג תואם את המדידה העצמאית שלכם עד כמה מילימטרים.
משנים את גובה האנטנה באמצע סשן ומוודאים שנפתח סשן חדש, ושנקודות שכבר נשמרו שומרות על הגובה הישן.

### שלב 4 — קואורדינטות מול נקודת ביקורת ידועה (כיול אתר)

זהו השלב שבו מגלים אם "הבדיקות עוברות" באמת אומר "זה מודד נכון בשטח". קוראים את `docs/calibration.md`
(כולל הסעיף העברי בסופו) לפני השלב הזה.

- תופסים את נקודת הביקורת, מודדים (ממוצע, לפחות 30 אפוקים מומלץ).
- מזינים את הקואורדינטות המפורסמות במסך הכיול ובודקים את השארית **לפני** אישור.
- **עם נקודה אחת בלבד**: לא מאשרים עדיין. שארית של כמה סנטימטרים סבירה; שארית של **מעל כמטר**
  כמעט תמיד אומרת טעות (נקודה לא נכונה, מערכת קואורדינטות שגויה) — לא הפרש אתר אמיתי.
- **עם שתי נקודות**: מדליקים "קיבוע קנה המידה על 1" כדי שתישאר תצפית עודפת שאומרת משהו.
- **עם שלוש נקודות ומעלה**: **RMS אופקי מתחת ל-2 ס״מ** סביר לאתר קטן; נקודה שגדולה פי 3 מהשאר
  היא כמעט תמיד הנקודה השגויה.

### שלב 5 — מדידה

מודדים נקודה בודדת ובממוצע (5 אפוקים) על אותה נקודה פיזית ומשווים (צריכות להתאים לרמת הדיוק
המוצהרת של המקלט ב-RTK Fixed, כ-1–3 ס״מ). מפעילים בכוונה כל אחד מהשערים (צף/DGPS דורש אישור;
עצמאי חסום; σ מעל 0.05/0.08 מ׳ חסום תמיד; גיל תיקונים מעל 10 שנ׳ חסום תמיד) ומוודאים שההתנהגות
תואמת.

### שלב 6 — סימון (Stake-out)

מסמנים ליעד, מוודאים שהמרחק/אזימוט/קדימה-ימינה נכונים תוך כדי הליכה, שהחץ גדל ככל שמתקרבים
ועובר לטבעות בתוך הטבעת הפנימית (2 ס״מ), ושה"בתוך סבילות" מופעל רק בטבעת הפנימית ולא בחיצוניות.

### שלב 7 — מצב יתד

מסמנים יתד, מוודאים ששלב 2 (מדידת קרקע) נדרש ולא ניתן לדלג עליו, בודקים דחייה של נקודת קרקע
במרחק **מעל 0.5 מ׳**, ומוודאים שביטול יתד באמצע (Abandon) לא משאיר שום עקבות בטבלת הנקודות.

### שלב 8 — ייבוא/ייצוא: מה שבאמת ניתן לבדוק היום

אשף ייבוא כללי ומסך ייצוא כללי **אינם קיימים עדיין** באפליקציה (`feature/io` הוא placeholder
בלבד). בודקים את מה שכן נגיש: קובץ גאואיד, ייבוא נקודות ידועות לכיול, ייצוא דוח as-built
(CSV/XLSX/PDF), וייצוא תמונת מפה (PNG/PDF). מוודאים ורושמים שאין אפשרות לייבא DXF/SHP/LandXML/
אורתופוטו או לייצא את טבלת הנקודות בשום פורמט — זו עובדה ידועה, לא באג שגיליתם.

### שלב 9 — מסלולי DXF ואורתופוטו (בסיוע מפתח)

מכיוון שאין בורר קבצים באפליקציה לאף אחד מהמסלולים האלה, בדיקתם דורשת גישת `adb` (מפתח, לא רק
מודד). מעתיקים קובץ DXF ל-`files/projects/<id>/dxf/` ובודקים תצוגה נכונה במפה ובסימון. בונים
`.mbtiles` עם `tools/ortho-converter`, מריצים `--verify` **לפני** ההעתקה למכשיר, מעתיקים ל-
`files/projects/<id>/raster/`, ומוודאים (לאחר הוספת שורת שכבה במסד הנתונים, גם היא בסיוע מפתח
כרגע) שהתצלום מוצג **נכון ולא הפוך** — זו הבדיקה החזותית החשובה ביותר במסמך הזה.

### שלב 10 — סיור בממשק העברי

מחליפים שפה לעברית ובודקים שהממשק כולו מתהפך, אך שהמפה, המצפן, סרגל קנה המידה וכל מספר על המסך
**לא** מתהפכים. קוראים כמה מסכים בחיפוש אחרי משפט עברי שסדר המילים בו הפוך — לפרויקט הזה יש היסטוריה
מתועדת בדיוק של הבאג הזה.

### מה לרשום כשמשהו לא בסדר

מה עשיתם, מה ציפיתם מול מה קרה, דגם המכשיר וגרסת אנדרואיד, דגם וקושחת המקלט, מקור התיקונים
(ספק ו-mountpoint — **לעולם לא** הסיסמה), המספרים המדויקים על המסך באותו רגע, צילום מסך אם רלוונטי,
וקובץ הלוג הגולמי (מפעילים "רישום לקובץ" במסך המקלט **לפני** השחזור).

### קריטריוני קבלה — סיכום

פתרון קבוע לפני שמירה; σ אופקי מקסימלי 0.05 מ׳; σ אנכי מקסימלי 0.08 מ׳; גיל תיקונים מקסימלי
10 שנ׳; לפחות 6 לוויינים; PDOP מתחת ל-3; חשבון גובה אנטנה תואם סרט מדידה עד כמה מ״מ; RMS אופקי
מתחת ל-2 ס״מ בכיול עם 3+ נקודות; נקודת קרקע של יתד עד 0.5 מ׳ מהיתד; "בתוך סבילות" רק בטבעת
הפנימית (2 ס״מ); אורתופוטו מוצג נכון ולא הפוך; טקסט עברי ב-PDF נקרא מימין לשמאל.
