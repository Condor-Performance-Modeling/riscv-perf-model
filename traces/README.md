# Generating Input for Olympia

Olympia can take input in two formats: JSON and
[STF](https://github.com/sparcians/stf_spec).

JSON files are usually reserved for doing *very* small and quick
"what-if" analysis or general testing.  JSON format should not be used
to represent an entire program.

STF format is the preferred format to represent a workload to run on
Olympia.  STFs are typically created by a RISC-V functional model that
has been instrumented to generate one.

## JSON Inputs

JSON inputs are typically generated by hand.  There are no automated
tools in this repository (or in
[Sparcians](https://github.com/sparcians)) that will create these
files.

The grammar for the JSON file is simple.  Keep in mind that JSON does
not support comments.

Top level:

```
[
   {
      ... instruction record 1
   },
   ....
   {
      ... instruction record N
   }
]
```
Instruction record can have:
```
    {
        "mnemonic" : "<risc-v mnemonic>",

        ... for integer
        "rs1"      :  <numeric value, decimal>,
        "rs2"      :  <numeric value, decimal>,
        "rd"       :  <numeric value, decimal>,

        ... for float
        "fs1"      :  <numeric value, decimal>,
        "fs2"      :  <numeric value, decimal>,
        "fd"       :  <numeric value, decimal>,

        ... for immediates
        "imm"      :  <numeric value, decimal>,

        ... for branches, loads, stores
        "vaddr"    :  "<string value>",
    }
```

Example:
```
[
    {
        "mnemonic": "add",
        "rs1": 1,
        "rs2": 2,
        "rd": 3
    },
    {
        "mnemonic": "mul",
        "rs1": 3,
        "rs2": 1,
        "rd": 4
    },
    {
        "mnemonic": "lw",
        "rs1": 4,
        "rd": 3,
        "imm":  128,
        "vaddr" : "0xdeadbeef"
    }

]
```

To run the JSON file, just provide it to olympia:
```
cd build
./olympia ../traces/example_json.json`
```

## STF Inputs

Using the [stf_lib]() in the [Sparcians](https://github.com/sparcians)
repo, Olympia is capable of reading a RISC-V STF trace generated from
a functional simulator.  Included in Olympia (in this directory) is a
trace of the "hot spot" of Dhrystone generated from the Dromajo
simulator. This directory also contains a trace of Coremark generated
similarly.

To run the STF file, just provide it to olympia:
```
cd build
./olympia ../traces/dhrystone.zstf
./olympia ../traces/core_riscv.zstf
```

### Generating an STF Trace with Dromajo

#### Build an STF-Capable Dromajo

Included with olympia is a patch to Dromajo to enable it to generate
an STF trace.  This patch must be applied to Dromajo before a trace
can be generated.

```
# Enter into the STF trace generation directory (not required)
cd olympia/traces/stf_trace_gen

# Clone Dromajo from Chips Alliance and cd into it
git clone https://github.com/chipsalliance/dromajo

# Checkout a Known-to-work SHA
cd dromajo
git checkout 6f68f74

# Apply the patch
git apply ../dromajo_stf_lib.patch

# Create a sym link to the stf_lib used by Olympia
ln -s ../../../stf_lib

# Build Dromajo
mkdir build; cd build
cmake ..
make
```
If compilation was successful, dromajo will have support for generating an STF trace:
```
./dromajo
usage: ./dromajo {options} [config|elf-file]
       .... other stuff
       --stf_trace <filename>  Dump an STF trace to the given file
       .... more stuff
```

#### Instrument a Workload for Tracing

Dromajo can run either a [baremetal application](https://github.com/chipsalliance/dromajo/blob/master/doc/setup.md#small-baremetal-program) or
a full application using Linux.

For this example, since Dromajo does not support system call
emulation, the workload dhrystone is instrumented and run inside
Linux.

The Dhrystone included with Olympia (`dhry_1.c`) is instrumented with
two extra lines of code to start/stop tracing.

``` main() {
    // ... init code

    START_TRACE;
    // Dhrystone benchmark code
    STOP_TRACE;

    // ... teardown
   }

```
The `START_TRACE` and `STOP_TRACE` macros are defined in `traces/stf_trace_gen/trace_macros.h`.

[Follow the directions](https://github.com/chipsalliance/dromajo/blob/master/doc/setup.md#linux-with-buildroot)
on booting Linux with a buildroot on Dromajo's page.

Build `dhrystone` that's included with olympia.  Set `RISCV_TOOLSUITE`
to a local copy of a RISC-V gcc location.

```
% cd dromajo/run
% $RISCV_TOOLSUITE/bin/riscv64-unknown-elf-gcc -O3 -DTIME ../../dhrystone/*.c -o dhry_riscv.elf
```
Copy `dhry_riscv.elf` into the buildroot and rebuild the root file system:
```
% cp dhry_riscv.elf ./buildroot-2020.05.1/output/target/sbin/
% make -C buildroot-2020.05.1
% cp buildroot-2020.05.1/output/images/rootfs.cpio .
```

To build `coremark` (not included with olympia), clone RISC-V ported `coremark` from
the below url. Set `RISCV_TOOLSUITE` to a local copy of a RISC-V toolchain location.
```
# Clone Coremark from EEMBC and cd into it
% git clone --recurse-submodules https://github.com/riscv-boom/riscv-coremark.git
% cd riscv-coremark/coremark

# Build coremark
% make RISCVTOOLS=$RISCV_TOOLSUITE PORT_DIR=../riscv64/ compile
```

Copy the generated `coremark.riscv` into the buildroot and rebuild the root file system:
```
% cp coremark.riscv ./buildroot-2020.05.1/output/target/sbin/
% make -C buildroot-2020.05.1
% cp buildroot-2020.05.1/output/images/rootfs.cpio .
```

Run Dromajo with the flag `--stf_trace`, log in, and run the benchmark for
1000 iterations (as an example):

```
% ../build/dromajo --stf_trace dhry_riscv.zstf boot.cfg

OpenSBI v0.8
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

... more bootup ...

Welcome to Dromajo Buildroot
buildroot login: root
Password: root
# dhry_riscv.elf
Dhrystone Benchmark, Version 2.1 (Language: C)

Program compiled without 'register' attribute

Please give the number of runs through the benchmark: 10000<return>

Execution starts, 10000 runs through Dhrystone
>>> DROMAJO: Tracing Staring at 101ba
>>> DROMAJO: Traced 2390026 insts

... rest of Dhrystone output


```
After Dhrystone has finished, `Ctrl-C` out of Dromajo and observe the generated trace:
```
% ls dhry_riscv.zstf
dhry_riscv.zstf
```
Now, run that trace on olympia:
```
% cd olympia/build
% ./olympia ../traces/stf_trace_gen/dromajo/run/dhry_riscv.zstf
Running...
olympia: STF file input detected

...
```
