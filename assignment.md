# Assignment: a divider by repeated subtraction

*Verilog HDL: from modules to a working UART · PGCP (VLSI) · case study 2*

In class you designed the multiplier: exercise E9 took you through registers, transfers,
the status signal and the state diagram on paper, and then you watched the code, the
testbench, the per-cycle table and the flop count. This assignment is E9 again, on a
different operation, and this time the parts you only watched are done by your own hands:
the four blocks, the coding, simulation, synthesis, and the flip-flop count. Everything you need as a
model is in the course folder: `course/d8_mult_add/mult_add.v` and its testbench, and slides 65 to 76.

Files in this folder:

| file | what it is |
|---|---|
| `div_sub.v` | the module header and its ports; the body is yours |
| `tb_div_sub.v` | the self-checking testbench; do not edit it, the instructor runs the same one |
| `report.docx` | the report form: one section per question, with the tables to fill in (opens in Word or LibreOffice Writer) |
| `assignment.md` | this sheet |
| `.github/workflows/test.yml` | the automatic check that runs the testbench when you push |

Read the whole sheet once before you start, including section 9, so that you know what
is assessed and in which order. Section 10 is a hint ladder; it is there for after an
honest hour of being stuck, not before.

## 1. The task

Build a circuit that computes the quotient and the remainder of two 8-bit unsigned
numbers: `q = a / b`, `r = a % b`. The method is mandated, because the assignment is about
the machine around the arithmetic, not the arithmetic: **repeated subtraction**. Subtract
the divisor from the dividend again and again until what is left is smaller than the
divisor; the number of subtractions is the quotient and what is left is the remainder.

By hand, 7 ÷ 2: 7 − 2 = 5, 5 − 2 = 3, 3 − 2 = 1, and 1 < 2, stop. Three subtractions:
q = 3, r = 1. And 5 ÷ 9: 5 < 9 before any subtraction, so q = 0, r = 5.

Arithmetic allowed, as in E9: one 8-bit subtractor, one 8-bit comparator, one 8-bit
incrementer. No `/`, no `%`, no multiplier. `b ≥ 1` is guaranteed by the testbench, so
division by zero never happens, and `q` always fits in 8 bits.

It is slow: 255 ÷ 1 takes 255 subtractions. That is fine, and it is one of the things the
testbench checks.

## 2. The interface and the handshake

The testbench connects to these names. Do not rename, add or remove ports.

| port | dir | width | meaning |
|---|---|---|---|
| `clk` | in | 1 | the clock; 10 ns period in the testbench |
| `rst` | in | 1 | synchronous reset, active high: a rising edge with `rst = 1` puts the machine in its idle state |
| `start` | in | 1 | "begin a division": looked at only while the machine is idle |
| `a` | in | 8 | dividend, read at the start edge only |
| `b` | in | 8 | divisor, read at the start edge only, never 0 |
| `done` | out | 1 | 1 for exactly one clock cycle, when `q` and `r` are valid |
| `q` | out | 8 | quotient |
| `r` | out | 8 | remainder |

**The handshake.** The user puts `a` and `b` on the inputs and raises `start` for one cycle.
The first rising edge at which the machine is idle and sees `start = 1` is **edge 0**. Some
edges later the machine raises `done` for one cycle; during that cycle `q` and `r` are
correct. After it the machine is idle again. `a` and `b` are read at edge 0 and may change
afterwards without affecting the division in progress; the multiplier had the same
requirement, and you know what it did about it.

**The latency is part of the specification.** `done` must be 1 exactly **q + 1** rising
edges after edge 0: for 7 ÷ 2, four edges; for 5 ÷ 9, one edge; for 255 ÷ 1, 256 edges.
A circuit is judged by its results and by its cycle count, and a divider that takes twice
as long as it needs is a worse divider. The testbench counts the edges on every division
and reports how many were on time; the assignment requires all eight.

## 3. What you are given

- `div_sub.v`: the header comment, the module line with the ports above, `endmodule`.
  Nothing else: no registers, no states, no signal names. Those are yours to choose, and
  choosing them is the assignment.
- `tb_div_sub.v`: eight divisions with a model that shares nothing with your design (the
  simulator's own `/` and `%`); section 5 shows its output.
- `course/d8_mult_add/mult_add.v`: the multiplier from class, complete and commented. It is the model
  for the **shape** of your code, the four blocks in the same order. Its registers, states
  and signals are the multiplier's, not the divider's; what carries over and what does not
  is for you to decide.
- Slides 65 to 76 in `course/slides/Verilog_HDL.pdf`: the two boxes, the recipe, E9 and its
  answer, the timing rule with the multiplier's per-cycle table, the code, the testbench,
  the flop count.

## 4. The nine questions

Answer them in `report.docx`, in this order. Questions 1 to 6 are done on paper before any
Verilog is written, and the report must show that: the register list in Q1 must be the one
your code uses, and a register that appears in the code but not in Q1 costs more than a
wrong width would.

**Q1, registers before states.** What must the circuit remember between clock edges?
For each register: its name, its width, what it holds, and why it must exist. The question
E9 asked still applies: the inputs may change after the start edge.

**Q2, the transfers.** For each register, what is written into it and when, in the form
`register ← expression`, and in which state. Each distinct transfer is one control signal;
name them.

**Q3, the status signal.** What is the one thing the controller must know to decide? Give
the signal as an expression of registers, and say which registers. The controller reads
this bit and nothing else from the datapath.

**Q4, the state diagram.** Draw it with the transfers written on the arcs, and put it in
`report.docx` (a photo of paper, inserted with Insert → Pictures, is fine).
Say for each output whether it is Moore or Mealy.

**Q5, the per-cycle table, before any code.** For 7 ÷ 2 and for 5 ÷ 9, in the format of
the multiplier's table from class, which is reproduced here for a = 3, b = 2:

| edge | state after the edge | `cnt` | `acc` | `done` | what this edge did |
|---|---|---|---|---|---|
| 0 | RUN | 2 | 0 | 0 | IDLE saw `start`: load |
| 1 | RUN | 1 | 3 | 0 | RUN saw `cnt = 2`, not zero: add |
| 2 | RUN | 0 | 6 | 0 | RUN saw `cnt = 1`, not zero: add |
| 3 | DONE | 0 | 6 | 1 | RUN saw `cnt = 0`: go to DONE, write nothing |
| 4 | IDLE | 0 | 6 | 0 | DONE: back to IDLE |

Edge 0 is the edge that samples `start`; "after the edge" is what the registers hold once
that edge has happened. Your table will have your registers as columns.

**Q6, the longest case.** How many edges does 255 ÷ 1 take from edge 0 to `done`, and what
does that number mean for a testbench that gives up after a fixed number of edges? (The
testbench you were given gives up after 600 edges.)

**Q7, the code and the run.** Write the four blocks in `div_sub.v`, run `tb_div_sub.v`
(section 5), and paste the verdict line. It must read `0 mismatches -> PASS; done after
q+1 edges on 8 of 8`.

**Q8, the flop count.** Before synthesis, the number of flip-flops you expect from Q1,
state register included, and the encoding you assume for it. After synthesis (section 6),
the `Register as Flip Flop` line, and one sentence reconciling the two.

**Q9, two pairs.** The testbench includes a = b (200 ÷ 200) and b = 1 (255 ÷ 1). Say what
each of those two pairs tests in *your* design: which decision in your code it exercises,
and what the output would have been with the other choice.

## 5. Running the testbench

Run it as the getting-started document, section 3.2, shows (route A or B), inside your
assignment folder.

**With `div_sub.v` as given**, nothing drives the outputs, and you get:

```
a=  7 b=  2: done never came (600 edges) FAIL
a=100 b=  7: done never came (600 edges) FAIL
... eight such lines
tb_div_sub: 8 divisions, 8 mismatches -> FAIL; done after q+1 edges on 0 of 8
```

That is what "not working yet" looks like.

**With a correct design:**

```
a=  7 b=  2 q=  3 r=  1 (model q=  3 r=  1) done after   4 edges ok
a=100 b=  7 q= 14 r=  2 (model q= 14 r=  2) done after  15 edges ok
a=255 b=  1 q=255 r=  0 (model q=255 r=  0) done after 256 edges ok
a=  0 b=  5 q=  0 r=  0 (model q=  0 r=  0) done after   1 edges ok
a=  5 b=  9 q=  0 r=  5 (model q=  0 r=  5) done after   1 edges ok
a=200 b=200 q=  1 r=  0 (model q=  1 r=  0) done after   2 edges ok
a=255 b= 16 q= 15 r= 15 (model q= 15 r= 15) done after  16 edges ok
a= 37 b=  3 q= 12 r=  1 (model q= 12 r=  1) done after  13 edges ok
tb_div_sub: 8 divisions, 0 mismatches -> PASS; done after q+1 edges on 8 of 8
$finish called at time : 3260 ns
```

**Reading a line.** `q=` and `r=` are your results; `model q=` and `model r=` are the right
answers; `ok` or `WRONG` is the comparison; `done after N edges` is your latency. The
verdict line counts mismatches (`PASS` needs zero) and, separately, how many of the eight
divisions finished in exactly q + 1 edges; section 2 requires 8 of 8.

## 6. Synthesis and the flop count

Run synthesis as the getting-started document, section 3.2, shows (route A or B), inside
your assignment folder.

Write down your prediction (Q8) *before* you run this. Then read one row of the report,
`Register as Flip Flop`, and the `Synth 8-` lines a little above it; one of them says what
the state machine was encoded as, which is usually not what you wrote. With the free
tools, `yosys -p "read_verilog div_sub.v; synth -top div_sub; stat"` and read the
`$_DFF` and `$_SDFFE` rows.

A count that is not what Q1 predicts, after the encoding is accounted for, means a
register exists that you did not name. Find it before anything else; it is a step-1
mistake, not a tool mistake.

## 7. When it does not work

**How to debug an FSMD.** Take the first failing pair from the output. Write its per-cycle
table from your state diagram, the way you did in Q5. Then add to `div_sub.v`, inside the
state-register block, a line such as

```verilog
if (!rst) $display("edge: state=%0d <your registers here>", state, ...);
```

run again, and compare the printed rows with your table. The first row where they differ
is where the diagram and the code disagree; the row before it tells you which. Remove the
`$display` before you submit. (A `$display` inside a module is allowed in simulation and
ignored by synthesis, but it is not part of the design.)

Three things the tools say that are worth knowing:

- `done never came (600 edges)` on every line: `done` is never raised, or never for the
  edge the testbench waits on. Look at your diagram's path to the state that raises it.
- The Vivado log says `inferring latch` for one of your control signals: a path through
  the `case` in block 2 leaves it unassigned. Defaults first, before the `case`.
- The automatic check on GitHub is red while your xsim run passes: it runs Icarus Verilog,
  which is stricter about a few things (an undeclared signal, a missing width). Open the
  `simulate` step in the job log and read the first message.

## 8. The structural checklist

Before you submit, read your `div_sub.v` once against this list. The instructor reads the
code, not only the verdict line.

- [ ] Block 1 is the state register; block 2 sets `next_state` and every control signal,
      with a default for each of them *before* the `case`; block 3 is the only place a
      datapath register is written; block 4 is `assign` lines for the status signal and the
      outputs.
- [ ] No register is written inside the `case` of block 2.
- [ ] Every clocked block uses `<=`; block 2 uses `=`.
- [ ] The status signal is an `assign` from registers, never from a "next" value.
- [ ] `a` and `b` are read in exactly one place.
- [ ] The names in Q1 to Q3 are the names in the code.

## 9. What to submit, and how it is assessed

Through VS Code and Git, as the getting-started document from the class group describes: `div_sub.v` with your name in its header, and
`report.docx` with Q1 to Q9, the diagram inside it. The
deadline is announced in class.

Assessment, in the order it is applied:

1. The verdict line: `PASS` with 0 mismatches.
2. The latency: 8 of 8 in q + 1 edges, as section 2 requires.
3. Q1 to Q6 on paper, and consistent with the code.
4. Q8 reconciled: predicted, reported, explained.
5. The code against the checklist of section 8.
6. Q9, which is where the marks for understanding sit: the two pairs were put in the
   testbench for a reason, and the report should say what it is.

## 10. The hint ladder

Each hint costs something: if you used it, say so in `report.docx` under the question it
belongs to. Nobody loses marks for an honest hint; the report is read differently, that
is all. Do not read a hint before an hour of your own effort on that question; the hour is
where the learning is.

**Hint 1, for Q1 and Q2.** The divider has the same shape as the multiplier with the sign
of one operation flipped. Take the multiplier's register list from the E9 answer and ask of
each register: what does this become when adding b times turns into subtracting until
nothing is left? One of them counts up instead of down. One of them holds something for
the same reason `areg` did.

**Hint 2, for Q3.** The controller needs exactly one bit: "is there anything left to
subtract?" It is a comparison of two registers, and it must be computed from the registers
as they are, not from what they are about to become.

**Hint 3, for Q5 and the latency.** In the multiplier, RUN tests `cnt_zero` on the values
it arrived with; if the test fails it adds, and the addition lands at the edge that leaves
RUN; if the test passes it goes to DONE and writes nothing at that edge. Your RUN does
the same with your test. Write the 7 ÷ 2 table with that rule and count the edges to
`done`; if you get five instead of four, one subtraction is happening at an edge where only
a decision should.

**Hint 4, for Q9.** For a = b: make it fail on purpose by changing one character in your
status signal, run, and watch which lines change (there are two). For b = 1: ask what is
largest in that run, the count and the number of edges, and what would break if your
quotient register were 7 bits wide, or if the testbench gave up after 200 edges instead of
600. What you see is the answer.
