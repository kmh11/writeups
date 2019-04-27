# Returns, 160 pts

This is a format string challenge.

The first step is getting main to loop. The last printf has been changed to puts due to compiler optimizations or something, so the GOT of puts can be overwritten with the address of main and the function will loop. This can be done in a way similar to how [this article](http://codearcana.com/posts/2013/05/02/introduction-to-format-string-exploits.html) describes it, although note that since this is 64-bit and the addresses have null bytes, the addresses must go after your format string.

Next you have to leak a libc addresses - this can be done by popping addresses off the stack (with `%x` or `%p`) until you get to `__libc_start_main_ret`. From this and the libc provided, you can calculate the base address and thus the address of any function in libc. 

After this, one last write is required to change strcmp to system, and then `/bin/sh` can be entered as the item and you have shell.
