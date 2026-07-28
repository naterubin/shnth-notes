## Registers

| arm name | shlisp name | details |
|:----|:---:|:---:|
| r0 | lispMEX | Stores the m-expression (operator or nut) of the current s-expression (the mexp is just the `car` of the sexp). |
| r1 | lispACC | Accumulator used when evaluating s-exps. |
| r2 | lispWOR | |
| r3 | lispRET | The result of evaluating an s-exp. |
| r4 | workONE/scanSCAN | Generic “work” register, also has additional function in more complicated opcodes (e.g. `sauce`, `salsa`, `togo`) |
| r5 | workTWO/scanPLAZ | '' |
| r6 | workTRI/scanDEEP | ''  |
| r7 | workFOR/scanCODE | ‘' |
| r8 | workFIV/scanSREF | ‘' |
| r9 | lispSIN | This is "Lisp sign", and indicates whether we are in Arab or Dirac mode. The clue here is line 57 of `<sexpress.s>`, where `lispSIN << 10` is added to the address of the Op to branch to. This seems that it would line up with the layout generated in the `INSTORG` macro, which first generates code for Arab mode and then generates code for Dirac mode. Also jibes with the nut code for `[dirac]` and `[arab]` on line 222 of `<wanillaF0nuts.s>`, which ANDs the MEX code with 1 and stores it in this register. Dirac is opcode 252 and Arab is 253. `252 & 1 = 0, 253 & 1 = 1`. |
| r10 | dacLEFT | The value here accumulates during an iteration of the audio loop. It is transferred to the left 12 bits of DAC register. |
| r11 | dacRITE | Same as dacLEFT but for the right channel. |
| r12 | lispCOD | Bytecode pointer? |

## Memory layout

Reset value for SP: 0x2000C000 (defined as `STACKINIT` in <equates.s>)

_Code_
0: vector table
0x10000: lispVectors

_SRAM_
0x20000: “wanillas”, audio processing routines (also known as “matrix” or “witches vectore”?)

_Peripheral_
0x40000000: timers 1,2 & 3

## Setup

### Sample rate

“srate” is controlled through the Cortex-M’s SysTick timer. This timer counts down from a set value (at register `SYSTICK_RELOAD`/`0xE000E014`), raising an interrupt when it reaches 0, and then reloads the value to begin again. The Shnth code sets the interrupt handler to the contents of <vectorSystick.s>.

The interrupt saves the Link Register, turns SysTick off, updates the left and right DAC channels, does whatever `LODMOTRA SYSTICK_RELOAD, 0x1000` does, turns SysTick back on, and then sets the Program Counter to the saved value of the Link Register.

## Audio loop

The `sysHandle` routine, which is SysTick’s interrupt handler, is where the audio loop happens. At a high level, SysTick is disabled, the **dacLEFT** and **dacRITE** registers are adjusted to be centered in the 12-bit integer space, copied to the DAC, and then zeroed out. The current situation is evaluated (see `<sexpress.s>`), calculating the next sample. Finally, SysTick is reenabled.

### Registers to DAC

### Bytecode evaluation

Evaluation begins in the `<sexpress.s>` file, included in the `<vectorSysTick.s>` file after the DAC has been updated. It’s important to remember that shlisp byte code defines the open parentheses as 255 and the closing parentheses as 0.

## Macro explanations

### Frequently used constants

```
BBAND = 0x22000000
bitband = 0x20000000
```

### <macros.s>

#### SYNTHLOAD(ref, vale)

This looks like a commonly needed pattern when a 32-bit constant needs to be moved to a register. `mov` instructions only have room for 16-bit constants, so a subsequent `movt` instruction is needed to move the most significant halfword into the register as well.

In the rest of this document I will be using `=:=` as a pseudo-code operator to indicate this 32-bit load occurs.

`ref[…]` syntax here refers to the bits being written to or read.

```
if vale < 255:
  ref = vale
else:
  ref[0:15] = vale[0:15]
  ref[16:31] = vale[16:31]
```

#### SYNTHLOADEREQ(ref, vale)

The use of `movweq/movteq` suggest that this to be used in an `it eq` block.

```
synth_ref = vale - bitband) + 0x20000000
if CONDITION_FLAGS->Z:
  ref =:= vale
```

#### SYNTHLOADER(ref, vale)

Same as SYNTHLOADEREQ but without the conditional suffixes.

```
synth_ref = (vale - bitband) + 0x20000000
ref =:= synth_ref
```

#### LODESTRA(reg, val)

Load value into location specified by reg. Both reg and val are 32-bit values. 

```
r1 =:= reg
r0 =:= val
MEMORY[r1] = r0
```

#### LODMOTRA(reg, val)

Same as LODESTRA but val is 16-bit.

```
r1 =:= reg
r0 = val
MEMORY[r1] = r0
```

#### LODESTRAH(reg, val)

Same as LODESTRA but loads only a halfword of val. No idea why we bother SYNTHLOADing the full 32-bits of val into r1 first…?

### <wanillaMACK.s>

#### BITBAND(r, addr)

Grabs the bitband alias for `addr`. Seemingly only writing to bit 0 of any of these bytes?

```
bitref = (addr-bitband) * 32 + BBAND
r =:= bitref
```

#### TRIG_IDEE_TOO(from, too, trigname, ref)

This is used to check triggers for most opcodes (on occasion `TRIG_IDEE_PRIMIF` is used instead). The `from` register is 

```
from =:= 1 if (from >= 1) else 0
# all the …trig variables are one byte, one bit for each instance of the m-exp.
# this fetches the appropriate byte to see if the trigger is active
twoo = BBTOO(twoo, ref, trigname, lispMEX)
# update the trigger bit with the new value in from
(ref + lispMEX * 2) =:= from
twoo = from XOR twoo
twoo = from ANDS twoo
```

### <wanilla10trango.s>

#### RECTA(out, reg)

In Dirac mode:
```
if (reg < 0)
  out = 0 - reg
end
```

In Arab mode:
```
  shifted = reg << 16
  if shifted < 0
    out = 0
  elif shifted > (2**16)
    out = (2**16)
  end
```

#### TRANGO_IDEE_NUME(acc, gwonzname)

```
RECTA(lispRET, lispRET)
bitband(workTWO, gwonzname, lispMEX)
if (workTWO == 1)
  acc = acc + (lispRET >> 4)
else
  acc = acc - (lispRET >> 4)
end
lispWOR[lispMEX << 1] = acc
```

## Opcodes

### wind

### bar

### minor

### major

### horn

### saw

### togo

### toggle

### swoop

### mount

#### Variables

* `lfogwonz`
* `mountVALES`

Pretty much the same idea as `horn`, increment or decrement (depending on if the lfogwonz value is 1 or 0) the mountVALE by the first argument, `nume`, and then check to see if the current value is higher than the second arg, `deno`. If it is, flip the gwonz bit. There is some scaling here that I don’t totally understand yet in the `ALSEROUP` macro that I assume is what differentiates this from `horn`.

Also, 

### smoke

#### Variables

* `noiseVALES`

[Linear congruential generator](https://en.wikipedia.org/wiki/Linear_congruential_generator)

Other reading:
* https://6502.org/source/integers/random/random.html
* https://stackoverflow.com/a/40709661 (this uses the same multiplier and increment)

```
Get current value from noiseVALES
Multiply by 25173
Add 13849
Store output back in noiseVALES
```

### dust

#### Variables

* `dustVALES`
* `dustRAMPS`

Generates noise the same way as `smoke` and also generates a ramp waveform that rises at a rate defined by the first argument to dust. Whenever the ramp rises above the random value, a pulse is output and the ramp is reset to 0.

### fog

### swamp

### haze

### string

### comb

### zither

### wave

### salt

### horse

### slew

### wheel

### gear

### pulse

### sauce

### salsa

### melody

### worm

### scale

### ladder

### rungler

### announce

### voder

### press

### leak

### reflect

### return

### and

### xor

## Nuts

### left/right

### square

### modo

### srate

### mul/add

### tar

### bend

### jump

### pan

### short

### dirac/arab

### lights