<div align="center">

<img src="web/public/logos/shrewdness-icon.svg" width="104" alt="">

# Shrewdness

</div>

Two languages and an IDE for them.

![The Shrewdness IDE: Savvy on the left, the assembly it compiles to and a live
console on the right](docs/ide.png)

**Shrewd** is a Turing-complete language whose programs are flat lists of
integers. Any list of integers is a valid program. There's no parse step, no
invalid opcode, nothing that can fault, so editing a program at random gives you
different behaviour instead of a syntax error.
→ [`src/shrewd/README.md`](src/shrewd/README.md)

**Savvy** is the language you actually write: small, C-like, one type, and it
compiles to Shrewd exactly. It also goes the other way. Hand `shrewdc savvy` any
list of integers at all and you get Savvy back that compiles and runs.
→ [`src/savvy/README.md`](src/savvy/README.md)

**Shrewdness** is the IDE. A browser front end over a small C++ backend, where
you edit Savvy, watch it turn into assembly and genes as you type, run it
against a live terminal, and step through it one instruction at a time.
→ [`web/README.md`](web/README.md)

## The `shrewdc` toolchain

```
shrewdc build  <file.savvy> [-o out.shrewd]   compile Savvy to a genome file
shrewdc run    <file.savvy|file.shrewd>       compile if needed, then run
shrewdc asm    <file.savvy|file.shrewd>       show Shrewd assembly
shrewdc savvy  <file.savvy|file.shrewd>       decompile a genome to Savvy
shrewdc genes  <file.savvy|file.shrewd>       show the raw gene list

run options:
  --seed N          seed for rand()  (default 0; a run is reproducible per seed)
  --steps N         stop after N steps (default: no limit -- loops run forever)
  --trace           report steps, halt reason and offspring on stderr
  --offspring DIR   write each committed child to DIR/childN.shrewd
```

`run` wires `input()` and `putchar()` to the real terminal, so interactive
programs like `examples/repl.savvy` prompt and wait the way anything else does.
A program that loops forever will loop forever; that's the point. Kill it like
any other process, or hand it a `--steps` budget.

Programs can also write genomes. `EMIT` and `SPAWN` build a child gene by gene,
and `--offspring` writes each one out. Children are genomes, so they run like
anything else:

```sh
shrewdc run examples/mutator.savvy --seed 1 --offspring gen1
shrewdc run gen1/child0.shrewd     --seed 2 --offspring gen2
diff gen1/child0.shrewd gen2/child0.shrewd   # what one generation changed
```

Programs can span files: `include "lib";` splices `lib.savvy` (resolved relative
to the including file) into the build, once. See the
[Savvy README](src/savvy/README.md#programs-can-span-files).

## The Shrewdness IDE

```sh
./build/shrewdness                       # 127.0.0.1:7070
./build/shrewdness --net --port 8080     # listen on all interfaces
```

An editor with the toolchain wired into the panes beside it. Type Savvy on the
left; the assembly, the gene list and the decompiled form on the right keep up
as you go.

![Typing a Savvy program while the assembly pane recompiles alongside
it](docs/live.gif)

You also get multi-file projects, split panes, a command palette, vim and emacs
keymaps, a terminal that really talks to `input()`, and a step debugger that
replays a run with the stack, registers and memory writes at every step. It
rearranges itself down to a phone rather than dropping features. Full tour in
[`web/README.md`](web/README.md).

The backend holds no database and no accounts. Projects live in your browser's
local storage, and the server just compiles, runs and traces what it's sent. To
put it on a public domain, see [Hosting it](web/README.md#hosting-it).

## Performance

Measured by `shrewd_bench` on a Ryzen 7 5800X, GCC 16.1.1 `-O3` (July 2026):

| workload | per step | throughput |
|---|---|---|
| tight counting loop | 2.4 ns | ~415 Mstep/s |
| recursive fib (call-heavy) | 2.7 ns | ~365 Mstep/s |
| self-replication | 2.6 ns | 1.45 µs per replication |
| Brainfuck interpreter in Savvy | 2.5 ns | ~400 Mstep/s |
| 20k random genomes, 10k-step budget | 3.6 ns | 2.0 µs per genome |

A whole short run (control-map build, execution, result) costs about 0.14 µs, so
one core gets through **roughly half a million budgeted genomes per second**.
Through the buffer-recycling `run_into` API a steady-state evaluation loop
performs zero heap allocations. Runs are independent pure functions, so N cores
give you N× by giving each thread its own `VM`. How it gets there is documented
in the [Shrewd README](src/shrewd/README.md#speed).
