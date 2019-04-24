# Server, 180 pts

In this challenge you were given a web server written in assembly. After disassembling the binary, you could see there were several syscalls which allowed the program to listen on port 19303 and fork a new process to serve each connection. You could also see there was a buffer overflow when reading in the path, since it just just kept reading until a space.

With this buffer overflow you could modify a syscall and ultimately get RCE.
