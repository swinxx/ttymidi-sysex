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

**2. Download the fixed source**
```bash
wget https://raw.githubusercontent.com/swinxx/ttymidi-sysex/main/ttymidi-fixed.c
```

**3. Compile**
```bash
gcc ttymidi-fixed.c -o ttymidi-fixed -lasound -pthread
```

**4. Replace the existing binary**
```bash
sudo cp ttymidi-fixed /usr/local/bin/ttymidi
```

**5. Restart ttymidi**
```bash
sudo systemctl restart ttymidi
```

---

## Original

Based on [cchaussat/ttymidi-sysex](https://github.com/cchaussat/ttymidi-sysex), which itself builds on earlier work by johnty, sixeight7, and ElBartoME.
