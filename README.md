# lyrics_display

Shows the currently-playing line of lyrics from Spotify/apple music on a 16x2 LCD
connected to an Arduino Uno.

## How it works
1. `play_with_lyrics.py` runs on your laptop, polls the Spotify Web API
   for what's playing and the exact playback position.
2. When a new song starts, it fetches synced lyrics (an LRC file, which
   has a timestamp for every line) from **lrclib.net**, a free/open
   lyrics database.
3. Every second it checks which lyric line matches the current playback
   time and sends that line to the Arduino over USB serial.
4. `lyrics_display.ino` on the Arduino just prints whatever text it
   receives onto the LCD.

## 1. Wire the LCD (I2C module — only 4 wires)

| Module Pin | Arduino Pin |
|---|---|
| GND | GND |
| VCC | 5V |
| SDA | A4 |
| SCL | A5 |

Before uploading the sketch, install the **LiquidCrystal_I2C** library:
Arduino IDE > Tools > Manage Libraries > search "LiquidCrystal I2C"
(the version by Frank de Brabander is the common one).

Most of these backpacks default to I2C address `0x27` or `0x3F`. The
sketch is set to `0x27` — if the screen stays blank or shows garbled
blocks, change `LCD_ADDRESS` near the top of `lyrics_display.ino` to
`0x3F`. If neither works, run an "I2C scanner" sketch (search
"Arduino I2C scanner" — it's a short, well-known sketch) to find your
exact address.

## 2. Upload the Arduino sketch
Open `lyrics_display.ino` in the Arduino IDE, select **Arduino Uno** and
the correct port, and upload it. The LCD should show "Waiting for
lyrics..." once it's done.

## 3. Choose your music source

### Option A: Spotify
See "Set up Spotify API access" below, then run `play_with_lyrics.py`.

### Option B: Apple Music (Windows app)
Apple Music on Windows has no public developer API, but Windows apps
that play media report their track info and playback position through
the **System Media Transport Controls** (the same info shown in your
volume flyout / lock screen widget). No API keys or account setup
needed.

Once the LCD is wired up and `lyrics_display.ino` is uploaded to the
Arduino, here's everything to do on the laptop side, step by step:

**Step 1: Check Python is installed**
Open a terminal — click Start, type `PowerShell` (or `cmd`), and press
Enter. Type:
```
python --version
```
If you see a version number (e.g. `Python 3.11.4`), you're good. If
you get an error, install Python from https://python.org/downloads
(check "Add Python to PATH" during install), then reopen the terminal.

**Step 2: Install the required packages**
In that same terminal window, type this and press Enter:
```
pip install winsdk pyserial requests
```
This downloads three small packages the script needs. Wait for it to
finish (you'll see "Successfully installed..." at the end).

**Step 3: Find your Arduino's COM port**
Plug the Arduino into your laptop via USB (if it isn't already).
Open Device Manager (Start > type "Device Manager" > Enter), expand
**Ports (COM & LPT)**, and note the port name, e.g. `COM3`.

**Step 4: Edit the script with your COM port**
Open `apple_music_windows.py` in any text editor (Notepad works —
right-click the file > Open with > Notepad). Near the top, find:
```
SERIAL_PORT = "COM3"
```
Change `"COM3"` to whatever port you found in Step 3, then save the
file (Ctrl+S).

**Step 5: Run the script**
Back in the terminal, navigate to the folder where you saved the
script — for example, if it's in your Downloads folder:
```
cd Downloads\lyrics_display
```
Then run:
```
python apple_music_windows.py
```
You should see `Listening for Apple Music playback...` printed.

**Step 6: Play a song**
Open the Apple Music app and start playing anything. Within a couple
of seconds the terminal should print the song name, and the LCD
should start showing lyrics.

To stop the script later, click back into the terminal window and
press `Ctrl+C`.

**If lyrics stop advancing but the title/artist show correctly:**
some apps don't report live position through SMTC, only title/artist.
If that happens with Apple Music, let me know and I can add a
fallback that estimates position with a local timer instead (started
when the track changes, assuming you don't pause/seek much).

## Set up Spotify API access
1. Go to https://developer.spotify.com/dashboard and create an app.
2. In the app's settings, add this Redirect URI: `http://127.0.0.1:8080/callback`
3. Copy the **Client ID** and **Client Secret** into `play_with_lyrics.py`
   (near the top, under `# ---- CONFIG ----`).

## 4. Install Python dependencies
```
pip install spotipy pyserial requests
```

## 5. Set your serial port
In `play_with_lyrics.py`, set `SERIAL_PORT` to match your Arduino:
- **Windows**: e.g. `COM3` (check Device Manager > Ports)
- **Mac**: e.g. `/dev/cu.usbmodemXXXX` (run `ls /dev/cu.*` in Terminal)
- **Linux**: e.g. `/dev/ttyACM0` (run `ls /dev/tty*` in Terminal)

## 6. Run it
```
python play_with_lyrics.py
```
The first run opens a browser window to log into Spotify and authorize
the app. Start playing a song on any Spotify device (phone, desktop
app, web player) and the LCD should start showing lyrics within a
couple of seconds.

## Notes / troubleshooting
- Not every track has synced lyrics on lrclib.net. If none are found,
  the display just shows the song title and artist instead.
- If nothing shows up, double check `SERIAL_PORT` and that the Arduino
  IDE's Serial Monitor is closed (only one program can use the port
  at a time).
- Reading "currently playing" works on free Spotify accounts too;
  it's controlling playback (skip/pause) that needs Premium — this
  project only reads, so it should work either way.
- Polling interval is 1 second by default (`POLL_SECONDS`) — lower it
  for tighter sync, but avoid going much below 1s to respect Spotify's
  rate limits.
