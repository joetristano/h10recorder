# Heart Monitor Recorder

Single-file web app that records live heart rate, RR intervals and HRV from any standard BLE heart-rate strap (Polar H10, WHOOP with Heart Rate Broadcast, etc.), plus raw ECG and accelerometer on the Polar H10. Shows heart-rate zones and exports a CSV.

## Use
- Needs a Web Bluetooth browser: Chrome/Edge on desktop or Android, or **Bluefy** on iOS (Safari doesn’t support it).
- Must be served over HTTPS (GitHub Pages works).
- Wet the strap electrodes, close Polar Beat/Flow, tap **Connect**, then **Start/Stop recording** and **Share CSV**.

## v4.2 — Whoop link diagnostics (2026-11-25, local only so far)
- **The app now says which path it took:** the first live test of the 4.0 fell back to the standard heart-rate broadcast with no visible reason — the fallback was silent. The connect flow now records the failure stage (find service / get characteristics / subscribe / hello / start streaming) plus the error message. On a WHOOP-named device the status line reports it, e.g. "Whoop 4.0 direct link failed (hello: Whoop command 0x5 timed out)", and the Whoop 4.0 card appears with the failure in its header instead of staying hidden. A 3.0 (no custom service) still falls back to broadcast, as before — the diagnosis only shows when a WHOOP device was connected.

## v4 — Whoop 4.0 direct BLE (2026-11-21, pushed to muse branch commit fe2bdff2, verified live)
- **Direct link, no broadcast needed:** when the connected device exposes Whoop's custom BLE service (`61080000-8d6d-82b8-614a-1c8cb0f8dcc6`), the app talks to it directly using the community reverse-engineered protocol from [christianmeurer/whoop-reader](https://github.com/christianmeurer/whoop-reader) (MIT) — HELLO handshake, START_REALTIME, then 96-byte real-time packets on characteristic `...8004`. HR (bytes 1–2, ×100) and RR (bytes 3–4, ms) feed the same pipeline as the standard HR service: live HR, zones, HRV, strain, recording.
- **SpO2 and skin temp:** byte 5 (SpO2 %) and byte 6 (skin temp = raw − 25 °C) show in a new Whoop 4.0 card and are recorded to CSV as `spo2` and `temp` rows. Battery is queried with the Whoop command (0x01) instead of the standard battery service.
- **Whoop 3.0 stays the same:** devices without the custom service use the standard heart-rate broadcast path exactly as before — the 4.0 attempt is tried first and silently falls back. The standard HR service is skipped once the 4.0 stream is running, so beats are never double-counted.
- **Unconfirmed accel:** packet bytes 7–12 are candidate accelerometer axes in whoop-reader. They are plotted live in the Accelerometer card (header reads "Whoop 4.0 · raw, unconfirmed" while connected) and recorded raw in the CSV as `wacc` rows — labelled unconfirmed in the app, since the shape is indicative but the scale is raw, not mG.
- **Testing:** the Whoop CRC-32 variant (poly 0x04C11DB7, init FFFFFFFF, xor-out F43F44AC) and all four command frames were cross-checked byte-for-byte against the Python whoop-reader code; a synthetic 96-byte packet (HR 72.34, RR 831, SpO2 97, 36 °C) parses and feeds the live pipeline; corrupt-CRC packets are counted and ignored. 42/42 whoop checks pass; strain (17/17) and info-button (12/12) suites still pass. **Not yet tested against a real Whoop 4.0** — that needs the band on a wrist.

## v2 upgrades (2026-11-14)
- **Exercise types:** General, Running, Cycling, Strength / bench press, HIIT, Rowing, Walking. Each sets its own low-pass (30–40 Hz) and refractory (250–300 ms) preset; changing it re-runs analysis on the loaded CSV.
- **Bench-press DSP:** `analyze()` now subtracts an 80 ms moving-window local mean (so a compressed ~303 µV R-peak in a −616 µV baseline cave is judged against its local floor), uses an adaptive sliding-window threshold (median energy of the preceding 1.5 s, no global max), and enforces a 250–300 ms blanking/refractory window to drop trailing pectoral EMG spikes. Filters stay zero-phase: 0.5 Hz high-pass, 30–35 Hz low-pass, 60 Hz notch. Heavy work runs in an inline Blob Web Worker. Tested on synthetic bench data (HR 97→108 bpm, −616 µV cave, post-peak EMG): 19/19 peaks at LP 30 and LP 35, no false peaks.
- **VO2max estimate:** HRmax/resting-HR ratio (Uth et al., 15.3 × HRmax/HRrest), plus Cooper 12-min and 1.5-mile run options. Enter resting HR in the Workout card. Estimate only, ±10–15% at best — not a medical measurement.
- **Moving HRV:** trailing 60 s and 5 min RMSSD keep updating on every beat and every 5 s, even when not recording, with an RMSSD trend chart. The main HRV panel still freezes on the full recording after Stop.
- **ECG analysis:** clinical grid is 0.04 s × 100 µV per small square (0.20 s × 0.5 mV major). Optional simulated-lead view (I, II, III, aVR, aVL, aVF) is derived from the single recorded lead and labelled simulated — not a real 12-lead, not diagnostic.
- **Sessions:** Sessions drawer lists saved runs (date, exercise, avg HR, HR slope, RMSSD, VO2max est.) with Load-as-ghost and Delete. Ghost overlay now has its own checkbox. Export JSON downloads a full session JSON (metrics, VO2max, peaks, RR, filtered + raw ECG, time array); Share JSON keeps the share-sheet route.

## v3.1 — info buttons (2026-11-21, pushed to muse branch commit 111805da, verified live)
- **Decluttered cards:** the explanatory/documentation text under each card (Workout note, zone definitions, strain formulas, HRV notes, VO₂max methods, R-peak legend, CSV format) now lives in a hidden panel behind a small ⓘ button in the card header (CSV card: in the button row). Tap ⓘ to show it, tap again to hide. Data, values, and controls stay front and center; nothing was deleted, only tucked away.

## v3 upgrades — strain (2026-11-21)
- **Live Strain card** (new card after HR zones): Edwards load, Banister TRIMP, and a 0–21 strain score, updating live on every beat while connected and freezing on the recording after Stop.
- **Edwards load** = Σ minutes in zone × weight (Z1=1 … Z5=5, below Z1=0), from the time-in-zone the app already tracks.
- **0–21 score** = 21×(1−e^(−Edwards/150)), a saturating map onto a familiar 0–21 scale. This is our mapping, not WHOOP's (their formula is proprietary) — labelled as such in the app.
- **Banister TRIMP** = duration(min) × ΔHR × 0.64×e^(1.92×ΔHR), ΔHR=(avg HR−resting HR)/(max HR−resting HR). Resting HR is the value entered in the Workout card, else the lowest HR seen; shown as "–" until a resting HR exists.
- **Analysis, sessions, exports:** the ECG analysis summary shows Strain / 21, Edwards load, and Banister TRIMP (zone seconds rebuilt from the HR samples in the CSV, so uploaded files get the same strain); saved sessions list the strain in the drawer; CSV gains `strain` rows (edwards_load, strain_0_21, banister_trimp) and JSON export carries `strain_0_21`, `edwards_load`, `banister_trimp` in metrics.
- **Caveat:** HR-based strain under-reads strength work — a bench session at 97–108 bpm scores low even though the muscular effort is high. Tested: 30 min Z2 + 20 min Z3 + 10 min Z4 → Edwards 160, strain 13.8/21, TRIMP 59.7 (avg 130, rest 58, max 190, 60 min); all 17 strain checks pass, including a units check that caught TRIMP being fed seconds instead of minutes in analysis.

## CSV format
`stream,t_s,v1,v2,v3`
- `ecg`: v1 = µV (130 Hz)
- `acc`: v1,v2,v3 = x,y,z mG (200 Hz)
- `hr`: v1 = bpm, v2 = RR interval ms
- `spo2` (Whoop 4.0 only): v1 = SpO2 %
- `temp` (Whoop 4.0 only): v1 = skin temp °C
- `wacc` (Whoop 4.0 only): v1,v2,v3 = raw accel x,y,z — unconfirmed decoding, recorded only
- `hrv`: summary rows at the end (v1 = value, v2 = metric name)
- `zone`: seconds per heart-rate zone (v1 = seconds, v2 = zone), plus max HR used
- `strain`: Edwards load, 0–21 score, Banister TRIMP (v1 = value, v2 = metric name)

## Protocol notes
Raw ECG/ACC come from Polar’s PMD service (`FB005C80-02E7-F387-1CAD-8ACD2D8DF0C8`): write a start command to the control characteristic (`...5C81`), read notifications on the data characteristic (`...5C82`). HR/RR use the standard Heart Rate service. Whoop 4.0 uses its own custom service (`61080000-8d6d-82b8-614a-1c8cb0f8dcc6`): framed commands `[0xAA, CMD, LEN_LO, LEN_HI, payload, CRC32_LE]` on `...8001`, responses on `...8002`, 96-byte real-time packets on `...8004` (HR bytes 1–2 ×100, RR bytes 3–4 ms, SpO2 byte 5, skin temp byte 6 − 25 °C, CRC-32 with xor-out 0xF43F44AC over bytes 0–91 in bytes 92–95 LE).
