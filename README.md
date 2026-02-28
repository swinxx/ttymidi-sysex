# ttymidi-sysex (norns ShieldXL)

Patched fork of [cchaussat/ttymidi-sysex](https://github.com/cchaussat/ttymidi-sysex) with critical bug fixes for use on **norns ShieldXL**.

Fixes SysEx/realtime interleave corruption, running-status buffer bugs, and several stability issues. See [CHANGES.md](CHANGES.md) for details.

---

## Update on norns via SSH

**1. Connect to norns**
```bash
ssh we@norns.local
# password: sleep
```

**2. Run the following commands on norns**
```bash
sudo systemctl stop ttymidi0.service
cd ~/shieldXL/ttymidi
wget -O ttymidi.c https://raw.githubusercontent.com/swinxx/ttymidi-sysex/main/ttymidi-fixed.c
gcc ttymidi.c -o ttymidi -lasound -pthread
sudo systemctl start ttymidi0.service
```

---

## Original

Based on [cchaussat/ttymidi-sysex](https://github.com/cchaussat/ttymidi-sysex), which itself builds on earlier work by johnty, sixeight7, and ElBartoME.
