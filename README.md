# Heart Monitor Recorder

Single-file web app that records live heart rate, RR intervals and HRV from any standard BLE heart-rate strap (Polar H10, WHOOP with Heart Rate Broadcast, etc.), plus raw ECG and accelerometer on the Polar H10. Shows heart-rate zones and exports a CSV.

## Use
- Needs a Web Bluetooth browser: Chrome/Edge on desktop or Android, or **Bluefy** on iOS (Safari doesn’t support it).
- Must be served over HTTPS (GitHub Pages works).
- Wet the strap electrodes, close Polar Beat/Flow, tap **Connect**, then **Start/Stop recording** and **Share CSV**.

## v2 upgrades (2026-11-14)
- **Exercise types:** General, Running, Cycling, Strength / bench press, HIIT, Rowing, Walking. Each sets its own low-pass (30–40 Hz) and refractory (250–300 ms) preset; changing it re-runs analysis on the loaded CSV.
- **Bench-press DSP:** `analyze()` now subtracts an 80 ms moving-window local mean (so a compressed ~303 µV R-peak in a −616 µV baseline cave is judged against its local floor), uses an adaptive sliding-window threshold (median energy of the preceding 1.5 s, no global max), and enforces a 250–300 ms blanking/refractory window to drop trailing pectoral EMG spikes. Filters stay zero-phase: 0.5 Hz high-pass, 30–35 Hz low-pass, 60 Hz notch. Heavy work runs in an inline Blob Web Worker. Tested on synthetic bench data (HR 97→108 bpm, −616 µV cave, post-peak EMG): 19/19 peaks at LP 30 and LP 35, no false peaks.
- **VO2max estimate:** HRmax/resting-HR ratio (Uth et al., 15.3 × HRmax/HRrest), plus Cooper 12-min and 1.5-mile run options. Enter resting HR in the Workout card. Estimate only, ±10–15% at best — not a medical measurement.
- **Moving HRV:** trailing 60 s and 5 min RMSSD keep updating on every beat and every 5 s, even when not recording, with an RMSSD trend chart. The main HRV panel still freezes on the full recording after Stop.
- **ECG analysis:** clinical grid is 0.04 s × 100 µV per small square (0.20 s × 0.5 mV major). Optional simulated-lead view (I, II, III, aVR, aVL, aVF) is derived from the single recorded lead and labelled simulated — not a real 12-lead, not diagnostic.
- **Sessions:** Sessions drawer lists saved runs (date, exercise, avg HR, HR slope, RMSSD, VO2max est.) with Load-as-ghost and Delete. Ghost overlay now has its own checkbox. Export JSON downloads a full session JSON (metrics, VO2max, peaks, RR, filtered + raw ECG, time array); Share JSON keeps the share-sheet route.

## CSV format
`stream,t_s,v1,v2,v3`
- `ecg`: v1 = µV (130 Hz)
- `acc`: v1,v2,v3 = x,y,z mG (200 Hz)
- `hr`: v1 = bpm, v2 = RR interval ms
- `hrv`: summary rows at the end (v1 = value, v2 = metric name)
- `zone`: seconds per heart-rate zone (v1 = seconds, v2 = zone), plus max HR used

## Protocol notes
Raw ECG/ACC come from Polar’s PMD service (`FB005C80-02E7-F387-1CAD-8ACD2D8DF0C8`): write a start command to the control characteristic (`...5C81`), read notifications on the data characteristic (`...5C82`). HR/RR use the standard Heart Rate service.
