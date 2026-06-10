# VSD Internship — VLSI System Design

> A hands-on internship journey through VLSI design, RISC-V toolchains, and digital systems.

---

## Table of Contents

| # | Task | Description |
|---|------|-------------|
| 1 | [Task 1](#task-1-risc-v-toolchain-setup--c-compilation) | Compile C code using GCC and the RISC-V GNU Toolchain |
| 2 | [Task 2](#task-2-spike-simulation-of-the-compiled-c-code) | Spike simulation of RISC-V assembly code and observation |

---

## Task 1: RISC-V Toolchain Setup & C Compilation

**Objective:** Install the RISC-V GNU cross-compiler toolchain and compile a C program for both the native host (x86-64) and the RISC-V target architecture. Observe the differences in assembly output between optimization levels.

---

### Step 1 — Install Dependencies

Install the required packages for building the cross-compiler on a Fedora/RHEL-based system:

```bash
sudo dnf install autoconf automake python3 libmpc-devel mpfr-devel gmp-devel \
  gawk bison flex texinfo patchutils gcc gcc-c++ zlib-devel expat-devel \
  libslirp-devel ncurses-devel
```

---

### Step 2 — Build the Linux Cross-Compiler

Choose an install prefix (a writable directory). Here, `~/riscv` is used:

```bash
./configure --prefix=~/riscv
make linux
echo 'export PATH=$PATH:~/riscv/bin' >> ~/.bashrc
source ~/.bashrc
```

> Note: This step takes a while — grab a coffee while it compiles!

---

### Step 3 — Verify the Installation

List the installed binaries:

```bash
cd ~/riscv/bin
ls
```

You should see a set of `riscv64-unknown-linux-gnu-*` tools:

![Installed RISC-V toolchain binaries](images/task1/a.png)

Confirm the cross-compiler version:

```bash
riscv64-unknown-linux-gnu-gcc --version
```

**Expected output:**
```
riscv64-unknown-linux-gnu-gcc (GCC) 14.1.0
Copyright (C) 2024 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

Also confirm the native GCC is available:

```bash
gcc -v
```

---

### Step 4 — Write the C Program

Create a file `sum1ton.c` — a simple program that computes the sum from 1 to *n*:

```c
#include <stdio.h>

int main()
{
    int i, sum = 0, n = 100;

    for (i = 1; i <= n; ++i)
    {
        sum += i;
    }

    printf("Sum of numbers from 1 to %d is %d\n", n, sum);
    return 0;
}
```

---

### Step 5 — Compile and Run on the Host (GCC)

Compile for the native host architecture and run:

```bash
gcc sum1ton.c -o sum1ton
./sum1ton
```

**Output:**
```
Sum of numbers from 1 to 100 is 5050
```

![GCC compilation and output](images/task1/b.png)

---

### Step 6 — Cross-Compile for RISC-V & Inspect Assembly

#### With `-O1` optimization:

```bash
riscv64-unknown-linux-gnu-gcc -O1 -mabi=lp64d -march=rv64g -o sum1ton.o sum1ton.c
riscv64-unknown-linux-gnu-objdump -d sum1ton.o | less
```

![Disassembly with -O1](images/task1/c.png)

#### With `-Ofast` optimization:

```bash
riscv64-unknown-linux-gnu-gcc -Ofast -mabi=lp64d -march=rv64g -o sum1ton.o sum1ton.c
riscv64-unknown-linux-gnu-objdump -d sum1ton.o | less
```

![Disassembly with -Ofast (1)](images/task1/d.png)
![Disassembly with -Ofast (2)](images/task1/e.png)

> **Key Observation:** With `-Ofast`, the compiler aggressively optimizes the loop — notice the significantly fewer instructions compared to `-O1`. The compiler may even evaluate the sum at compile time or use vectorization.

---

## Task 2: Spike Simulation of the compiled C code

**Objective:** Simulate the C code compiled using RISC-V GNU Toolchain using Spike, a RISC-V simulator, and observe the output. Along with that, also use debug tools in spike. 

---

### Step 1 — Install dependencies and Spike

First, install all dependencies : 
```bash
sudo dnf install -y gcc-c++ make dtc libmpc-devel mpfr-devel gmp-devel
```
Next, clone the github repository of Spike: 
```bash
git clone https://github.com/riscv-software-src/riscv-isa-sim.git
cd riscv-isa-sim
```
Now, build and install Spike: 
```bash
mkdir build
cd build
../configure --host=riscv64-unknown-linux-gnu --prefix=/usr/local
make -j$(nproc)
sudo make install
```
Verify installation of Spike:

```bash
spike --help
```

A help menu should open if you have installed it correctly.

---

### Step 2 — Install RISC-V proxy kernel

```bash
cd ~
git clone https://github.com/riscv-software-src/riscv-pk.git
cd riscv-pk
mkdir build
cd build
../configure --host=riscv64-unknown-linux-gnu --prefix=/usr/local
make -j$(nproc)
sudo make install
```
Set a symlink to pk so we do not have to type an absolute path every single time.
```bash
sudo ln -s /usr/local/riscv64-unknown-linux-gnu/bin/pk /usr/local/bin/pk
```

---

### Step 3 — Compiling the program using gcc and RISC-V toolchain

**Using GCC:**

```bash
gcc sum1ton.c
./a.out
```

**Expected output:**
```
Sum of numbers from 1 to 100 is 5050
```
**Using RISC-V GNU Toolchain:**

```bash
riscv64-unknown-linux-gnu-gcc -march=rv64g -mabi=lp64d -static -o sum1ton.o sum1ton.c
spike $(which pk) sum1ton.o
```
Observe that the outputs will be same for both GCC and RISC-V toolchain.
![Compilation using GCC and RISC-V toolchain](images/task2/a.png)

---

### Step 4 — Using the debugger in Spike

To debug a program using spike, first we need to look at the object dump of the program.
```bash
riscv64-unknown-linux-gnu-objdump -d sum1ton.o | less
```
![Object dump](images/task2/b.png)
Now, use spike debugger to debug the program.
```bash
spike -d $(which pk) sum1ton.o
```
If we look at the object dump of the program, main function starts at address 10340. So when we start debug, we run the program counter until that address and then we look at individual registers to find out a bug if there is any.
```
0000000000010340 <main>:
   10340:       0004f537                lui     a0,0x4f
   10344:       00001637                lui     a2,0x1
   10348:       ff010113                addi    sp,sp,-16
   1034c:       3ba60613                addi    a2,a2,954 # 13ba <__libc_dlerror_result+0x1372>
   10350:       68850513                addi    a0,a0,1672 # 4f688 <__rseq_flags+0x4>
   10354:       06400593                li      a1,100
   10358:       00113423                sd      ra,8(sp)
   1035c:       181000ef                jal     10cdc <_IO_printf>
   10360:       00813083                ld      ra,8(sp)
   10364:       00000513                li      a0,0
   10368:       01010113                addi    sp,sp,16
   1036c:       00008067                ret
```
In the image, we can see that I ran the Program counter from 0 to 10340. Then checked a0. The reg value has not been updated yet. To move forward you press enter and on each step after the PC ticks, we can check and see how the register values are changing. 
![Spike debugger](images/task2/c.png)


---

## Repository Structure

```
VSD-internship/
├── images/
│   ├── task1/          # Screenshots for Task 1
│   │   ├── a.png       # Toolchain binaries
│   │   ├── b.png       # GCC host compilation output
│   │   ├── c.png       # RISC-V -O1 disassembly
│   │   ├── d.png       # RISC-V -Ofast disassembly (part 1)
│   │   └── e.png       # RISC-V -Ofast disassembly (part 2)
│   └── task2/          # Screenshots for Task 2
│       ├── a.png       # GCC and RISC-V toolchain compilation output
│       ├── b.png       # Object dump of sum1ton.o
│       └── c.png       # Spike debugger session
└── README.md
```

---

## Acknowledgements

- [VLSI System Design (VSD)](https://www.vlsisystemdesign.com/) — for the internship program and curriculum
- [RISC-V GNU Toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain) — the open-source cross-compilation toolchain used throughout
