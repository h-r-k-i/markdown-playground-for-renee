# An Operating System That Definitely Is Not Named Neko Trust Me I Would Never Do That

updated readme updated readme holy shit she's doing it she's gonna rant for 45 pages about bullshit that the
average american does not care about :P but that's why you check my repos isn't it? certainly isn't for
professionally written code

![](https://media1.tenor.com/m/SmipoZPWb1AAAAAd/and-then-he-clicked-post.gif)

Professionally speaking, Neko is a bootloader. I mean, in all actuality, Neko is a stub kernel that prints
something like `test from neko kernel_main(). if this is shown, we are in neko 32-bit kernel space`
and then hangs like how I want to- we're getting ahead of ourselves this is supposed to be professional.
The impressive part is the entire bootloader attached to the repository
that does bootloader stuff like producing a Multiboot-like hardware map and loading a kernel.

Unprofessionally speaking, this directory is SHIT and ASS and should NEVER be considered and yeah I'm not
beating the unemployment allegations but I was already employed by Haruki Media Group where I got my cool
email and I'm never getting a real job am I?

The bootloader is called SBL for Staged Boot Loader or that's the name I give it because I genuinely forgot
the original name. On a good day it's SBL. On a normal one like the one I had when I "finished" adding ELF
support it's Shitty Bitchass Liar.

## Prologue

Have you ever felt like life was going too well and you should instead spend months writing horrible code
to never get a job? I have. This is what brought me to spend a year of my limited lifespan writing a bootloader
instead of having friends.

## So... what does it actually do?

It loads an operating system.

Since some people don't use "trust me bro" as proper documentation and demand proper information on what
the program actually does here's a section-by-section description of what each thing does:

1. Stage 1, in `boot/s1.s`. The most proper one. Loads the second stage and lets itself get torn by
   the wolves. Also enables unreal mode so I can do some hacky pointer bullshit.
2. Stage 2, in `boot/s2/`: Written in a nightmarish conglomerate of assembly and larper-mode 16-bit C,
   this part grabs a memory map of choice (whose choice? the BIOS's), pokes at the BIOS for more information
   because my inferiority complex wasn't big enough, and shoves it all haphazardly into a linked list for
   some poor soul to try to retrieve it and discover my lack of a will to live.
   Some funny bits in there too. Check it out if you survived the nightmare of seeing code that could
   only have been written by a college junior deliriously high off of diphenhydramine and nicotine.
3. Stage 3, in `boot/s3/`: 16-bit C++ now because I am an apostate and didn't wanna rewrite my code.
   It has a disk reader, an ELF loader, console output, and a shutdown function. And once the user
   actually gets the proper kernel to be loaded, it shits out a kernel in upper memory, sets up
   the GDT and IDT, enters protected mode, and jumps to the kernel. Genericized, because any
   Arc-C-compliant kernel works. Documentation on what makes an Arc-C-compliant kernel in approximately
   700 years. I have to cry first.
4. The... "kernel," in `bin/`. It writes a single string to the text-mode framebuffer and dies.
   Peak engineering, I tell you what. Give me my Turing Award and an unregistered firearm **now**.

Anyone who complains about `boot/s3/drivers/` not being drivers is gonna be "found" dead on Peachtree.
Which one? Well, that's where the scare quotes come in.

## Getting Started

It's README dependency all the way down. That never changed. The difference now is some of them are
sometimes written by Claude when I can't give a shit anymore.

### Prerequisites

- Realistically speaking device-wise you only need a computer that can run QEMU but if you want to torment
  yourself like that then you need a 64-bit CPU and a motherboard that supports LBA indexing (most modern ones do).
  From recent tests this OS can run itself off of a USB drive but I will not be testing past the bootloader
  stage because xHCI is... hard. And deeply scares me. I have tested through SATA using bullshit disk interrupts
  though and sometimes it works so everything's all chill and good except for when it isn't but that's
  a concern for future me, not the one writing a README.
  It's recommended to use QEMU, although any emulator that supports 64-bit x86 should work
  (if you don't use QEMU God help you)
  (use the Make on a 4GiB system I fucking ***dare*** you to).  
- Due to the nature of OSes you will need a cross-compiler to build the "OS".
  If you don't know how to build a cross-compiler, search up "osdev xcc" and if that doesn't show anything
  "osdev cross compiler." It is not that hard; it just takes a while. Grab a coffee. And four RedBulls.
  And a pack of cigarettes.
- Speaking of compilers, you specifically need a C compiler that can compile for 32-bit and 64-bit x86. I use GCC here.
  You also need a C++ compiler. I use G++ here.
  You will also need NASM and ld for linking. This isn't a hard requirement; just do `sudo apt install nasm`. NASM doesn't even need to be cross-compiled.
- You will need disk image stuff like `sfdisk` and `mkfs.fat` and `mmd` and `mcopy` because
  I genuinely woke up one day and decided to ruin my life by implementing FAT support.
  This is genuinely how that happened, I'm not even joking.
- You will need Make because God has abandoned us. Otherwise unchanged except now God has abandoned us even more.

> [!WARNING]
> The Makefile in its current state hardcodes locations. Unless you're *also* a Renee who compiled
> her cross-chain compiler in `~/opt/cross/` then you gotta change those. Your problem.

> [!WARNING]
> The Makefile in its current state builds out of `bin32/` which is gitignored because it was
> supposed to be a test directory but now it's just the kernel directory so good luck with that
> lmao.

> [!IMPORTANT]
> This project is incomplete. Search up what that means. Also learn this fun fact: with OS dev,
> there's never a finish line. You're just deliriously searching for one in the dark. Be warned.
> GRUB's behind you. With a ski mask and a Glock 21 Gen4. Don't try to run. It has a switch.

### Building


```bash

# clone repo
git clone https://github.com/h-r-k-i/neko.git
cd neko

# Build the "OS" (aka just the bootloader) (we'll add the kernel later) (and by we, I mean me)
make
```

### Running
```bash

# Run the OS in QEMU
make run

# If it works you'll see the text
# "test from neko kernel_main(). if this is shown, we are in neko 32-bit kernel space"
# on a black screen with more text on it. Ignore what the last README said. One of the
# commit messages was "bootloader done" so we're not above baldfaced lies.
# If it doesn't work, don't worry. There is no God to save you this time. Just qemu.log.
```

### Debugging
```bash
# starts QEMU paused with a GDB stub
make debug

# preferably in a better terminal
gdb -ex "target remote :1234"
```

May God have mercy on your soul. I sold mine already. In my defense, Takis are really fucking good.

### Cleanup

```bash
make clean
```

Thank God this works all the time except for when it doesn't.

### So... what's tools?

Some host-side C programs so I don't have to do it by hand:

- IDTBuilder.c: for when you can't be fucked to write 256 IDT handlers by hand
- MemMapReader.c: deprecated memory map reader when SBL was simpler
- SBLReader.c: the pristine Arc-C hardware map reader which works all the time except when it didn't
  and made me wanna shit myself violently at Five Points.

Compiled with gcc. The normal one. Please. Use the normal one.

### Usage Example

uhhhhhh

lol

check youtube

## Roadmap

- [x] Booting
- [X] Getting to protected mode without eating shit 99% of the time (it's still 98% but that's an improvement!)
      (think of it like this: successes jumped from 1% to 2%. That's 100% improvement)
- [X] FAT32 and ELF reading
- [ ] Writing a kernel
- [ ] long mode
- [ ] working on a webserver (more terrifying than talking to women)
- [ ] testing the most recent version on hardware
- [ ] ATA PIO


what is there to say about it? It's a bad bootloader

## References

- [OSDev Wiki](https://wiki.osdev.org) from which half this code was pilfered from.
- [The IA-32 Intel Architecture Software Developer's Manual](https://www.cs.cmu.edu/~410/doc/intel-sys.pdf)
- [15-410 Project 4](https://www.cs.cmu.edu/~410-s07/p4/p4-boot.pdf) which this has massively diverged from
- Wikipedia
- Ralf Brown's Interrupt List

## Repository layout

Who do you think I am? Claude?

## License

This project is licensed under the [MIT License](LICENSE) but frankly why would you want to use this shit?

## Acknowledgements

I HATE x86 I HATE CR0 I HATE UNREAL MODE I HATE THE 286 I HATE QEMU nah qemu's chill everyone else can fuck themselves.

this is bad and ahh and shid and crap and dookie
