# ttymidi-sysex (norns ShieldXL)

Patched fork of [cchaussat/ttymidi-sysex](https://github.com/cchaussat/ttymidi-sysex) with critical bug fixes for use on **norns ShieldXL**.

Fixes SysEx/realtime interleave corruption, running-status buffer bugs, signal handler safety, and several stability issues.

**Current version: v1.03**

---

## Install / Update on norns

```bash
ssh we@norns.local
# password: sleep

# Stop service (use kill if stop hangs)
echo sleep | sudo -S systemctl kill -s SIGKILL ttymidi0.service
echo sleep | sudo -S systemctl stop ttymidi0.service

# Download, compile, install
cd ~/shieldXL/ttymidi
wget -O ttymidi.c https://raw.githubusercontent.com/swinxx/ttymidi-sysex/main/ttymidi-fixed.c
gcc ttymidi.c -o ttymidi -lasound -pthread
echo sleep | sudo -S rm -f /usr/bin/ttymidi
echo sleep | sudo -S cp ttymidi /usr/bin/ttymidi

# Start service
echo sleep | sudo -S systemctl start ttymidi0.service
```

**Important:** After installing, reboot norns (SYSTEM > SLEEP > RESTART) so that MIDI mods (e.g. midirouter) rescan all ports correctly.

---

## Check installed version

```bash
ttymidi --version
```

---

## Fixes by version

### v1.03
- **CRITICAL:** printf format crash (4 specifiers, 3 args in SPP handler)
- **CRITICAL:** Signal handler: printf() replaced with async-safe write()
- Partial write() error checks on all serial writes
- Error checks: tcgetattr, tcsetattr, pthread_create, read() in printonly mode
- SysEx max-length guard (8192 bytes in midirouter)
- sleep(100s) replaced with usleep(100ms) in main loop
- snd_seq_close() on shutdown, break after baudrate switch, bzero replaced with memset

### v1.02
- Signal handler closes serial fd for immediate read() unblock on shutdown
- Double-close guard for serial fd in main()
- write() error checks in ALSA-to-serial path

### v1.01
- volatile sig_atomic_t for run flag (signal/thread visibility)
- strncpy null-termination for device name and serial device
- read() return value checks (prevents garbage MIDI on serial disconnect)
- System Common 0xF1 (MTC) and 0xF3 (Song Select) data byte handling
- pthread_join for midi_in_thread (prevents undefined behaviour on shutdown)

### v1.0
- Real-time messages (0xF8-0xFF) no longer interrupt SysEx accumulation
- SysEx buffer reset after forwarding (prevents running-status corruption)
- SysEx buffer overflow protection (cap at 256 bytes)
- Verbose/silent guard corrected for unknown events
- bytes[2] sentinel for 2-byte messages (Program Change, Channel Pressure)

See [CHANGES.md](CHANGES.md) for detailed descriptions of each fix.

---

## Original

Based on [cchaussat/ttymidi-sysex](https://github.com/cchaussat/ttymidi-sysex), which itself builds on earlier work by johnty, sixeight7, and ElBartoME.
