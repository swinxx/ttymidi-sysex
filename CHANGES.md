# ttymidi Bug Fixes — Change Log

## File: `ttymidi-fixed.c`

---

### WP1 — `read_midi_from_serial_port()` — Serial→ALSA direction (CRITICAL)

#### Fix A — Real-time messages no longer interrupt SysEx or channel message accumulation

**Location:** Inner `while (bytesleft > 0)` loop, top of loop body.

**Problem:** Real-time messages (0xF8 CLOCK, 0xFA START, 0xFB CONTINUE, 0xFC STOP,
0xFF RESET) all have the high bit set and are not 0xF7, so the original condition
`(readbyte & 0x80) && (readbyte != 0xF7)` treated them as new message starts.
This reset `buflen = 0` and interrupted SysEx accumulation mid-stream.

**Fix:** Added a check `if (readbyte >= 0xF8)` BEFORE the `bytesleft--` decrement.
Real-time bytes are forwarded to ALSA immediately via `write_midi_to_alsa` and the
loop `continue`s — leaving `bytesleft`, `buflen`, and `buf` untouched. The real-
time byte also does not count against `bytesleft` because we skip the decrement.

```c
// ADDED in read_midi_from_serial_port(), inner loop:
if (readbyte >= 0xF8) {
    char rt_buf[1] = {(char)readbyte};
    write_midi_to_alsa(seq, port_out_id, rt_buf, 1);
    continue; // bytesleft NOT decremented; accumulation state preserved
}
bytesleft--;  // moved: was before the real-time check
```

#### Fix B — buf[0] and buflen reset after SysEx ends

**Location:** Outer `while (run)` loop, after `write_midi_to_alsa` call.

**Problem:** After a SysEx message (`F0 ... F7`) is processed and forwarded to
ALSA, `buf[0]` retains `0xF0` and `buflen` retains the SysEx byte count. If the
device then sends running-status Note On data bytes (no repeated `0x90` status),
those bytes accumulate starting at `buf[buflen]` with `buf[0]` still `0xF0`.
When `bytesleft` eventually reaches zero, a garbled SysEx blob is forwarded to
ALSA — producing the observed corruption `F0 63 00` instead of `90 00 63`.

**Fix:** After writing a SysEx message to ALSA, clear `buf[0]` to `0x00` and reset
`buflen` to `0`. This means:

- If the next byte is a proper status byte (`0x90` etc.), the existing new-status
  detection branch fires correctly as before.
- If the next byte is a running-status data byte (high bit clear), it accumulates
  but `buf[0] == 0x00` so the ALSA-write guard `buf[0] & 0x80` is false — the
  stale data is silently discarded. This is correct per the MIDI spec: SysEx
  cancels running status.

```c
// ADDED after write_midi_to_alsa() call in outer loop:
if (buf[0] == 0xF0) {
    buf[0] = 0x00; // clear stale SysEx status
    buflen = 0;    // start fresh for next message
}
```

---

### WP2 — `write_midi_action_to_serial_port()` — ALSA→Serial direction

#### Fix C — SysEx buffer overflow protection

**Location:** `case SND_SEQ_EVENT_SYSEX:` in `write_midi_action_to_serial_port`.

**Problem:** `sysex_data` is declared as `unsigned char sysex_data[256]`. ALSA
SysEx events can be larger than 256 bytes. The copy loop had no bounds check,
causing a stack buffer overflow for long SysEx messages.

**Fix:** Cap `sysex_len` at `sizeof(sysex_data)` before the copy loop.

```c
// CHANGED in write_midi_action_to_serial_port(), SND_SEQ_EVENT_SYSEX case:
sysex_len = ev->data.ext.len;
// FIX: bounds check — sysex_data is 256 bytes; truncate if needed
if (sysex_len > (int)sizeof(sysex_data))
    sysex_len = (int)sizeof(sysex_data);
```

#### Fix D — Default case verbose/silent guard corrected

**Location:** `default:` case in first `switch (ev->type)` inside
`write_midi_action_to_serial_port`.

**Problem:** Original code had `if (!arguments.verbose)` — this printed the
"Unknown/Unsupported command!" warning when NOT in verbose mode, which is
backwards. Unknown events are warnings that should always appear unless silenced
(i.e., `-q` flag), not only when verbose mode is off.

**Fix:** Changed to `if (!arguments.silent)` to match the convention used for
all other warning/error paths in the file.

```c
// CHANGED:
// was: if (!arguments.verbose)
if (!arguments.silent)
    printf("Unknown/Unsupported command!\n");
```

#### Fix E — bytes[2] sentinel for 2-byte messages

**Location:** `case SND_SEQ_EVENT_PGMCHANGE:` and `case SND_SEQ_EVENT_CHANPRESS:`
in `write_midi_action_to_serial_port`.

**Problem:** Program Change and Channel Pressure are 2-byte messages; `bytes[2]`
is never set for them but retains stale values from the previous iteration. While
the write call correctly uses only 2 bytes (`write(serial, bytes, 2)`), the stale
`bytes[2]` could confuse future debugging or if write lengths change.

**Fix:** Explicitly set `bytes[2] = 0xFF` (the same initial sentinel value as the
array declaration) for both 2-byte message types.

```c
// ADDED for PGMCHANGE and CHANPRESS:
bytes[2] = 0xFF; // mark as unused for 2-byte message
```

---

## Summary table

| ID | Function | Bug | Fix |
|---|---|---|---|
| A | `read_midi_from_serial_port` | Real-time bytes interrupt SysEx | Check `>= 0xF8` before decrement; pass through immediately |
| B | `read_midi_from_serial_port` | `buf[0]` stays `0xF0` after SysEx; running-status data corrupts next message | Reset `buf[0]=0x00`, `buflen=0` after SysEx forwarded to ALSA |
| C | `write_midi_action_to_serial_port` | Stack overflow for SysEx > 256 bytes | Cap `sysex_len` at `sizeof(sysex_data)` |
| D | `write_midi_action_to_serial_port` | Unknown-event warning only printed in non-verbose mode (inverted) | Change guard to `!arguments.silent` |
| E | `write_midi_action_to_serial_port` | `bytes[2]` stale for 2-byte messages | Set `bytes[2] = 0xFF` for PGMCHANGE, CHANPRESS |

---

## Build & deploy

```bash
# Build locally (cross-compile) or on norns:
gcc ttymidi-fixed.c -o ttymidi -lasound -lpthread

# Deploy to norns:
scp ttymidi-fixed.c we@norns.local:~/shieldXL/ttymidi/ttymidi.c
ssh we@norns.local "cd ~/shieldXL/ttymidi && \
  gcc ttymidi.c -o ttymidi -lasound -lpthread && \
  sudo cp ttymidi /usr/bin/ttymidi && \
  sudo systemctl restart ttymidi0.service"
```

## Test

In maiden REPL (http://norns.local/maiden):
```lua
midi.connect(4).event = function(data)
  local s = ""
  for i,b in ipairs(data) do s = s .. string.format("%02X ", b) end
  print("PORT4: " .. s)
end
```

Activate M8 Launchpad Mode.

- **Before fix:** `PORT4: F0 63 00` — corrupted, LEDs stay dark
- **After fix:** `PORT4: 90 00 63` — correct Note On, LEDs light up
