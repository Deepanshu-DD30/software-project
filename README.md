# software-project
"""therac25_demo.py - simplified model of two Therac-25 software faults.
Illustrative only: this is NOT the original PDP-11 assembly code."""
import threading
import time

# ---------- Fault 1: 1-byte counter overflow (Yakima, 1987) ----------
class Therac25:
    def __init__(self):
        self.class3 = 0              # 1-byte flag, nonzero => "check turntable"

    def setup_pass(self, turntable_ok):
        # BUG: the flag is incremented, not set to a fixed value like 1
        self.class3 = (self.class3 + 1) & 0xFF      # wraps 255 -> 0
        if self.class3 != 0:
            if not turntable_ok:
                return "BLOCKED: turntable position error"
        return "check skipped" if not turntable_ok else "OK"

m = Therac25()
print("--- Fault 1: counter overflow ---")
for n in range(1, 257):
    result = m.setup_pass(turntable_ok=False)    # turntable is WRONG every time
    if n in (1, 255, 256):
        print(f"pass {n:3d}: class3={m.class3:3d} -> {result}")

# ---------- Fault 2: race condition on operator edit (Tyler, 1986) ----------
entry = {"mode": "X-ray"}                         # shared data, no locking
applied = {}

def setup_thread():
    snapshot = entry["mode"]                      # reads the entry ONCE
    time.sleep(0.5)                               # slow hardware set-up
    applied["mode"] = snapshot                    # never re-reads the edit

print("\n--- Fault 2: race condition ---")
t = threading.Thread(target=setup_thread)
t.start()
time.sleep(0.1)
entry["mode"] = "electron"                        # fast operator edit
t.join()
print("Screen shows :", entry["mode"])
print("Machine uses :", applied["mode"])
if entry["mode"] != applied["mode"]:
    print("MISMATCH -> beam set-up does not match the screen")
