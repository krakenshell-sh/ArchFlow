# ArchFlow — Desktop Productivity Tracker

A minimalist productivity tracker for **Arch Linux** — written in C++17 with a
Qt6/QML frontend, SQLite3 storage, and an X11 background daemon.

---

## Features

| Feature               | Detail                                                             |
|-----------------------|--------------------------------------------------------------------|
| **Window tracking**   | X11 `XGetInputFocus` + `_NET_WM_NAME` + `/proc/<pid>/comm`         |
| **Session recording** | SQLite3 WAL — every 5 s poll, 5-min periodic flush                  |
| **Discipline streak** | 56-day heatmap (14 × 4 grid), productivity %-coloured              |
| **Focus trend**       | Smooth Bézier area chart — last 7 days                             |
| **Pie chart**         | Productive vs unproductive time distribution                       |
| **Daily routine**     | Checkbox tasks with animated progress bar                          |
| **Specific tasks**     | CRUD with deadline, green/red status                               |
| **Dual theme**        | Dark + Light — toggle with **one button**, instant animated switch |
| **Daemon service**    | systemd user service, auto-restart on crash                        |

---

## Dependencies

```bash
sudo pacman -S \
    cmake \
    qt6-base \
    qt6-declarative \
    libx11 \
    sqlite

# Recommended (monospace font used by the UI)
sudo pacman -S ttf-liberation
```

---

## Quick Install

```bash
git clone https://github.com/you/archflow.git
cd archflow
chmod +x install.sh
bash install.sh
```

The script will:
1. Check and offer to install missing packages
2. Build with CMake (Release, all CPU cores)
3. Install `archflow` + `archflow-daemon` to `/usr/local/bin/`
4. Install and start the systemd user service

---

## Manual Build

```bash
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel $(nproc)

# Run the GUI directly (daemon optional)
./build/archflow

# Run the daemon manually (requires X11)
DISPLAY=:0 ./build/archflow-daemon
```

---

## Project Structure

```
archflow/
├── CMakeLists.txt
├── archflow-daemon.service
├── install.sh
├── README.md
└── src/
    ├── core/
    │   ├── window_tracker.hpp / .cpp   # X11 focus detection + classification
    │   └── database.hpp / .cpp         # SQLite3 interface (WAL, RAII stmts)
    ├── daemon/
    │   └── archflow_daemon.cpp         # Background session recorder
    └── gui/
        ├── backend_model.hpp / .cpp    # Q_PROPERTY bridge: DB → QML
        ├── main.cpp                    # Qt entry point
        └── qml/
            └── Main.qml               # Full UI — dual theme, all charts
```

---

## Architecture

```
┌─────────────────┐   SIGTERM    ┌────────────────────────────┐
│  archflow-daemon │ ─────────>   │  systemd user service      │
│                 │              │  archflow-daemon.service    │
│  WindowTracker  │              └────────────────────────────┘
│  (X11 poll/5s)  │
│       │         │  WAL write   ┌─────────────────────────┐
│  Database       │ ──────────>  │  archflow.db (SQLite3)   │
└─────────────────┘              │  sessions               │
                                 │  daily_tasks            │
┌─────────────────┐              │  specific_tasks          │
│  archflow (GUI)  │  WAL read    └─────────────────────────┘
│                 │ <──────────
│  BackendModel   │   every 30s
│  (Q_PROPERTY)   │
│       │         │
│  QML Engine     │
│  Main.qml       │
│  ┌────────────┐ │
│  │ Ring/Pie/  │ │
│  │ Trend/Heat │ │  ← all drawn with QML Canvas (no QtCharts)
│  │ (Canvas)   │ │
│  └────────────┘ │
└─────────────────┘
```

---

## Productivity Classification

The daemon classifies every focused window into one of three categories:

| Category         | Detected by                                |
|------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Productive**   | Process: `code`, `cursor`, `nvim`, `vim`, `emacs`, `kitty`, `alacritty`, `konsole`, `clion` … Title: `github.com`, `docs.`, `stackoverflow`, `archlinux`, `cppreference` … |
| **Unproductive** | Title: `youtube`, `reddit`, `twitter`, `x.com`, `instagram`, `facebook`, `tiktok`, `twitch`, `netflix` …                                                       |
| **Neutral**      | Everything else                                                                                              |

To add custom rules, edit `src/core/window_tracker.cpp` → `WindowTracker::classify()`.

---

## Theme Toggle

Press the **☀ Light Mode** / **🌙 Dark Mode** button in the top-right of the
dashboard. All colours, charts (Canvas repaint), and borders animate to the new
theme in **250 ms**.

|              | Dark      | Light     |
|--------------|-----------|-----------|
| Background   | `#0F1117` | `#EDE8D8` |
| Card         | `#191C24` | `#F5F1E4` |
| Productive   | `#52B788` | `#5A9E70` |
| Unproductive | `#E76F51` | `#B85A50` |
| Heatmap high | `#52B788` | `#1B5E3C` |

---

## Daemon Control

```bash
# Status
systemctl --user status archflow-daemon

# Live logs
journalctl --user -u archflow-daemon -f

# Restart
systemctl --user restart archflow-daemon

# Disable on login
systemctl --user disable archflow-daemon
```

If the DISPLAY is not forwarded automatically (e.g., after a manual login):

```bash
systemctl --user import-environment DISPLAY XAUTHORITY
systemctl --user restart archflow-daemon
```

---

## Database Schema

```sql
-- Recorded window sessions
CREATE TABLE sessions (
    id           INTEGER PRIMARY KEY AUTOINCREMENT,
    window_title TEXT    NOT NULL,
    category     TEXT    NOT NULL,   -- 'Productive' | 'Unproductive' | 'Neutral'
    start_time   INTEGER NOT NULL,   -- Unix timestamp
    end_time     INTEGER NOT NULL,   -- Unix timestamp
    date_ymd     TEXT    NOT NULL    -- 'YYYY-MM-DD'
);

-- Per-day routine tasks (re-seeded each day)
CREATE TABLE daily_tasks (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    name           TEXT    NOT NULL,
    completed_bool INTEGER DEFAULT 0,
    date_ymd       TEXT    NOT NULL
);

-- Persistent project tasks with optional deadline
CREATE TABLE specific_tasks (
    id                 INTEGER PRIMARY KEY AUTOINCREMENT,
    name               TEXT    NOT NULL,
    deadline_timestamp INTEGER DEFAULT 0,  -- 0 = no deadline
    completed_bool     INTEGER DEFAULT 0
);
```

Inspect at any time:

```bash
sqlite3 ~/.local/share/archflow/archflow.db \
    "SELECT date_ymd, category, SUM(end_time-start_time)/60 AS mins
     FROM sessions GROUP BY date_ymd, category ORDER BY date_ymd DESC LIMIT 20;"
```

---

## Uninstall

```bash
bash install.sh --remove
# Database is preserved at ~/.local/share/archflow/
rm -rf ~/.local/share/archflow   # to also delete data
```

---

## License

MIT — do whatever you like with it.
