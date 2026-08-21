# h0 binary analysis report

## What I did
After teacher permission I followed the steps that I presume most of the class did; google "Hello World" C++, pasted that to VSCODE and compiled it on Mac

## Compilation Command
g++ /Users/repe/Desktop/Häkki2/hello.cpp -o /Users/repe/Desktop/Häkki2/hello

## Results

### File type 
hello: Mach-O 64-bit executable arm64
Program was compiled natively to system currently in use

### File Size
The binary is 37KB -rwxr-xr-x  1 repe  staff    37K Aug 20 15:56 hello

### Libraries
Program is depending on libc++ and libSystem.B

/usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 2100.43.0)
/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1356.0.0)

### Strings
Only visible text is "Hello World" 

### Symbols
"nm hello" gave quite the mouthful, considering it was quite simple program. Apparently C++ encodes additional information in the machine code files, process being "name mangling"

## Ovservations
It's a nice start for the semester, having to face stuff you have only a vague understanding of. Feels like learning!

## AI disclaimer
Claude did help me with the compiling errors on VSCODE and framework for the report, all text is my own. 
