# Polar H10 Live Recorder

Single-file web app that streams ECG, heart rate, RR intervals and accelerometer data from a Polar H10 over Web Bluetooth, plots them live, and exports a CSV.

## Use
- Needs a Web Bluetooth browser: Chrome/Edge on desktop or Android, or **Bluefy** on iOS (Safari doesn’t support it).
- Must be served over HTTPS (GitHub Pages works).
- Wet the strap electrodes, close Polar Beat/Flow, tap **Connect**, then **Start/Stop recording** and **Share CSV**.

## CSV format
`stream,t_s,v1,v2,v3`
- `ecg`: v1 = µV (130 Hz)
- `acc`: v1,v2,v3 = x,y,z mG (200 Hz)
- `hr`: v1 = bpm, v2 = RR interval ms

## Protocol notes
Raw ECG/ACC come from Polar’s PMD service (`FB005C80-02E7-F387-1CAD-8ACD2D8DF0C8`): write a start command to the control characteristic (`...5C81`), read notifications on the data characteristic (`...5C82`). HR/RR use the standard Heart Rate service.
