# Heart Monitor Recorder

Single-file web app that records live heart rate, RR intervals and HRV from any standard BLE heart-rate strap (Polar H10, WHOOP with Heart Rate Broadcast, etc.), plus raw ECG and accelerometer on the Polar H10. Shows heart-rate zones and exports a CSV.

## Use
- Needs a Web Bluetooth browser: Chrome/Edge on desktop or Android, or **Bluefy** on iOS (Safari doesn’t support it).
- Must be served over HTTPS (GitHub Pages works).
- Wet the strap electrodes, close Polar Beat/Flow, tap **Connect**, then **Start/Stop recording** and **Share CSV**.

## CSV format
`stream,t_s,v1,v2,v3`
- `ecg`: v1 = µV (130 Hz)
- `acc`: v1,v2,v3 = x,y,z mG (200 Hz)
- `hr`: v1 = bpm, v2 = RR interval ms
- `hrv`: summary rows at the end (v1 = value, v2 = metric name)
- `zone`: seconds per heart-rate zone (v1 = seconds, v2 = zone), plus max HR used

## Protocol notes
Raw ECG/ACC come from Polar’s PMD service (`FB005C80-02E7-F387-1CAD-8ACD2D8DF0C8`): write a start command to the control characteristic (`...5C81`), read notifications on the data characteristic (`...5C82`). HR/RR use the standard Heart Rate service.
