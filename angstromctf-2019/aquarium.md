# Aquarium, 50 pts

This challenge is a relatively basic buffer overflow. You have a win function, so you use the unbounded input in `gets` to overflow the buffer until the return address is overwritten with the address of the win function.

Plenty of tutorials online (and hopefully community created writeups for this challenge) will go into more detail about how to do this.
