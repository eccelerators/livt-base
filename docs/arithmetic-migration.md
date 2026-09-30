# Arithmetic migration — 2026-09-30

`Bits` uses shifts/masks for power-of-two arithmetic, including insertion and
rotation assembly. Rotation amount reduction uses 31 scheduled bit steps with a
bounded remainder (width 1..31, amount nonnegative). Base remains independent of
Math. Public calls remain serialized; cycle counts may change.

`Format` shares a decimal workspace and constant place table. It extracts digits
with at most nine scheduled subtractions per place instead of parallel decimal
division expressions. This is a step bound, not a total call-cycle count: function
and loop handshakes add latency. Signed magnitude conversion uses uint, including
INT_MIN. Byte hex/binary formatting uses shifts/masks. Formatting calls must be
serialized; no throughput guarantee is introduced.

Existing conversion helpers use constant power-of-two remainder with deliberate
signed correction. Diagnostics hexadecimal formatting is simulation-only. These
were reviewed and retained. No Base-to-Math dependency was introduced.

## Validation

All 272 Base tests passed with GHDL 4.1.0, including new full-width decimal and
rotation boundary/reuse cases. Run `livt test` to reproduce. See the
[JUnit results](evidence/arithmetic-migration/base.xml) and
[source hashes](evidence/arithmetic-migration/source-sha256.json).

The complete exposed Format interface synthesized on xc7a100tcsg324-1 with
Vivado 2026.1, 20 ns clock, four threads, 20 GiB memory guard, and 600-second
runtime limit (157 seconds actual): **1,137 LUTs, 1,441 FFs, no DSP/BRAM, no
latches; internal pre-route setup WNS +14.879 ns**.

This is component synthesis evidence, not a routed implementation or a measured
before/after area reduction. Input/output delays and a physical clock source
were not supplied; boundary paths remain unconstrained. The generated Format
payload conversion contains no native division/remainder; generated call
arbitration still uses constant `mod 3` on bounded owner indices. The
[retained Tcl](evidence/arithmetic-migration/synth.tcl),
[resource report](evidence/arithmetic-migration/utilization.rpt), and
[timing report](evidence/arithmetic-migration/timing.rpt) record the run. Temporary
paths in the Tcl need relocating for reproduction. The synthesis input snapshot
predates a final Bits-only cleanup; Format was unchanged.
