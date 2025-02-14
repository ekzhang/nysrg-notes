- Linux executables — [[February 11th, 2024]]
    - What is an executable?
        - A file with code in it
        - Binary 01010101
        - It has a _start() method / entrypoint
        - "You can double-click on it" (execute it)
        - Architecture-specific file that you can execute
            - "The OS knows how to run it"
            - The kernel has code to parse and load various parts of the executable
        - Header, Static data (also arch-specific), machine code (architecture-specific)
        - Is a Python file an executable?
            - "executable permission" — files have this
        - `glibc`
    - > An executable is a file containing binary code with a defined entry point, such as a _main() method, designed for a specific architecture, which an operating system can directly execute, often by double-clicking, and includes components like headers, static data, and machine code; its executability may also depend on permissions, and it can involve libraries like glibc.
    - ELF Header
        - REL = Relocatable file (snippet / "not yet linked" — not quite an executable yet, like a `.o` file)
        - EXEC = Executable
        - DYN = Shared library (or bin `/bin/true`)
        - CORE = Core dump (fake, not an executable)
    - **Q: Why use the ELF file format for core dumps?**
        - "Why not?" Reusability of CLI tools.
    - **Q: Why are shared libraries like .so files still in the ELF format?**
        - They aren't executable, they're just in the executable file format. (Not confusing)
        - It has "executable energy"
        - Also relevant: You can have executables that are actually shared libraries in disguise.
    - Entry point is a 64-bit number
        - To be explained, but these are not actually direct byte offsets? In some cases?
    - Metadata / dynamic sizes for what's to come.
    - Code reading tips
        - Ignore all of the types and generics
        - Nom = parser combinator crate, build a binary structure out of various smaller parts.
        - Parser combinators help you validate data in a structured way, with context
        - The code is just a translation of the ELF header diagram into an executable program. (You could write it in Python, too.)
    - Part 2: Memory mapping and mprotect regions
        - We finally talk about the end of the header.
        - [File header] + [Program header 1] + [Program header 2] + …
        - Program headers define memory mapping regions for code and data.
        - Why do we need to map things into regions? Security: some parts should be readable and writable, you can't just JMP into a random place.
        - The region check is in the OS, but the processor itself also has safety checks (set in rings).
        - Cooperation between OS and hardware. Page table bits.
        - Program headers = memory mapping regions + protection bits on each region.
        - 48-bit addresses
        - Loop through all program headers and find which region corresponds to the `entry_point`.
        - **Q: Why is calling a function from a shared library more expensive? ("DSO how-to")**
            - Follow-up for later
        - **Q: Who determines memory ranges?**
            - Old executables are at a fixed address.
            - New executables look like a shared library, which are "position-independent" — you can offset them +N bytes.
            - `-pie`
            - OS will just add some random number (ld.so interprets the binary!).
        - Paper rec about security: "The geometry of innocent flesh on bone"
    - Part 3: Position-independent code
        - Trivia: "Running executables without exec()"
        - Retracing path:
            - Modified program to accept executables with a single code segment, and jump to them. This works!
            - Try to iterate over all of the memory ranges, call mmap(MAP_FIXED | MAP_ANON) on all of them, and copy the data into those mappings. This doesn't work!
            - Something PIE
            - It turns out you actually need to use /lib/ld.so as an interpreter for PIEs!
            - Link the program with that ld interpreter, and then run it again, and it works.
        - Dynamic linking is literally just mmapping the .so file into memory.
        - The rest of the ELF file is also mmapped with various protection bits.
        - 64-bit address spaces take offsets from %rip (`lea` instruction).
        - This is what happens when you use `LD_PRELOAD` to overwrite some API in a shared library when running a binary. `ld` uses this variable to figure out what to map into virtual memory.
        - **Q: What if libc.so.6 gets bigger?**
            - It's fine because ld is responsible for dynamically determining addresses.
            - Each library is its own ELF object.
        - **Q: What happens if you call a linked function? How does your code determine the correct address to jump to at runtime?**
            - Dynamic linker has a list of all the addresses and rewrites all of them.
            - PLT / GOT — lookup tables for executables for symbols
        - **Q: Why would you want to use LD_PRELOAD?**
            - Plugging in a different version of malloc (tcmalloc).
            - Debugging functions by patching them into an existing binary, print out their arguments.
            - "This is not normal"
            - __"You wouldn't do that in prod, would you?"__
            - Caveat: You need the right ABI.
    - Part 4: ELF relocations
