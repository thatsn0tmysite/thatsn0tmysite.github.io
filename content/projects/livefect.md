---
title: "Livefect"
date: 2024-03-14T02:19:30+02:00
draft: false

---
# Livefect

Livefect is a PoC tool to play around with `/proc/pid/mem`, `process_vm_readv` and `process_vm_write` as alternatives ways of injecting into processes without using `ptrace` directly.

Technically we still need the same privileges to use this as we need for `ptrace` but I mean, why not.
It was fun to play around with this as I had to learn about ELF, shellcodes and Linux in general.

Excuse the terrible C code, it has been so long since I actually wrote anything in it.

I literally have no idea why I did this or if this is useful in any way or form. Enjoy :).

Some scenarios where this might be helpful:
   - Overwrite of a function's code with our payload is necessary (see options -e, -S, --symbol, --library)
   - Overwrite of executable data is necessary (i.e. modify a variable's dlsym value to an arbitrary address)
   - Overwrite of data (i.e. values which control process flow)

> If you end up using this software any feedback or contribution is appreciated. If you go through the code... I'm really, really sorry for the mess (lol?).

## TODOs and possible new features
   - [x] Rewrite code to be less messy
   - [ ] Rewrite ELF parsing in a more structured way
   - [x] When segment to rwx mem is found, write payload at given address (-a), then pad the rest of the mapped area with `jmp address`, to maximize a chanche of execution
   - [ ] Regex filters for function and export names
   - [ ] iovecs splitting if payload is > page_size (jump added at pagesize_bytes-jmpsize to jmp to next available area)
   - [ ] Find PLT/GOT entries of in-memory/loaded modules, overwrite address (this should be possible on RELRO binaries if the library is loaded at runtime) 
   - [ ] See if feasable/useful allow reading ROP gadget addresses from file, file format could be something like that:
   ```
   0x123456 // write first gadget here
   0xff1122 // this is the first gadget
   0x88eeaa // this is the second gadget
   0x1122da // this is the third gadget
   ...
   ...
   ```
   - [ ] It'd be very cool if it could also look for XREFs in the binary to identify possible injection points
   ```
   Get ELF
   Disassemble and find XREFs
   Fix offsets
   Find all segments with specified permissions (i.e. rwx)
   Return output of 
   ADDR     XREF_COUNT
   0x123    1
   0x456    12
   0x789    234         //then one can further inspect this addr, see if it's a function (--disams) and write here (-a 0x789)
   ```


## Compile
To compile `livefect`, [zydis](https://github.com/zyantific/zydis) is needed. Just run:

```bash
make # make livefect with all features and victim program

make victim # make the victim program to play around
```

## Usage
You will likely need to be `root` to run this.
```text
Livefect = Infect live, running processes
Usage: ./livefect [opts]
        -h --help:              show this help message
        -V --process-vm:        use process_vm_writev instead of /proc/[pid]/mem
        -p --pid PID:           run against specified pid (default: NULL)
        -f --file FILE:         file containing data to write/payload (default: NULL)
        -a --addr ADDR:         (raw mode) address to write data/payload to
        -v --verbosity LEVEL:   set verbosity level (default: 0)
        -P --perms PERMS:       which permissions to look for (default: rwxp)
        -S --func-only:         only display function symbols
           --symbol SYMBOL:     write FILE to all symbols with name SYMBOL
           --library LIBRARY:   to use with --symbol allow write to specific SYMBOL from specific LIBRARY
        -D --data-only:         only display data symbols
        -I --inject-all:        (potentially unsafe) force payload injection in any matching memory area
        -m --maps:              only list memory maps
           --disasm N:          (implies -m) try disassemble the first N instructions and show output (default: 0)
           --disasm-bytes N:    (implies -m) bytes to pass to disassembler (default: 64)
        -e --exports:           only list exported functions
        -F --force-disk:        force ELF exports lookup from disk
        -s --skip-root:         do not enforce root checks

Made while screaming in *PANIC* by @thatsn0mysite (https://thatsn0tmy.site)
```

### Recon
To simply display interesting segments we can use the `-m` or `-e`, to show memory segments and exports and quit. No writing is done (unless **raw mode** is on). 
```bash
./livefect -me --pid 123
```

### Auto-mayhem
The simplest usage is to just run `livefect` with a payload as such:
```bash
./livefect -f shellcodes/reverse_shell 

```

this will attempt to find all writable segments containing exports and write our payload there, in ALL of em.

### Controlled-mayhem
Another basic usage is using `livefect`'s **raw mode**:
```bash
./livefect -f shellcodes/reverse_shell -a 0x7fb75ed44109
``` 

this will attempt to write our payload at the specified address.

### Filters
Some basic filtering has been implemented to semplify injection. For example to write payload at the address of symbol `test_func` we can use the `--symbol` and `--library` flags along with `-P`:
```bash
./livefect -f shellcode/reverse_shell --symbol test_func --library /usr/lib/test.so -P "r*x*" --pid 123
```

if we are lucky, this will write our payload into the `test_func` of library `/usr/lib/test.so` from pid `123`, `-P` specifies the permissions we are looking for, using `proc maps` format.  
