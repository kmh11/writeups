# Weeb Hunting, 230 pts

This was a heap challenge, and I believe there were multiple ways to solve it. Below I'll describe my solution.

You could get a double free by just using an item twice - the free'd pointer was not cleared, so the check to see if it was an empty slot failed. With this you could create a loop with the fastbins and allocate something that was also on the fastbin list. The `fd` of this fastbin could be modified to point to a fake fastbin in `.bss` and that could then be allocated and modified to overwrite a weapon pointer to an address on the global offset table, leaking a libc address when weapon names were printed.

The same attack with a double free could then be used to overwrite \_\_malloc_hook to the win function and get shell.
