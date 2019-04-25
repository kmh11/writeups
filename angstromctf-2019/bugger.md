# Bugger, 200 pts

The binary was packed with UPX (findable with `strings`), but the packer said it could not unpack it. This was because the `UPX!` header was replaced with null bytes, so it had to be added back in. The binary then performed some weird calculations (modified SHA512 with some random stuff) to get the flag. The values could be pulled from within GDB with a breakpoint set in the proper place.
