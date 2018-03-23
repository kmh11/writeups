# File Storer, 160 pts

You are given a link to a (incredibly ugly) website where you can create accounts and upload files. However, if you try a common name like `test` for a file, it says the file already exists! This means all files are stored in the same place. Going a step further, the files may be stored in the same place as the rest of the website. Trying to access `files/index.py` (it can be determined it is Flask from the 404 page) gives a special message, so there are protections against reading it, but it is confirmed it is reading from the root directory of the website. Through the hint or just knowledge of common web vulnerabilities, one decides to try to access `files/.git`, and, luckily enough, it says the directory exists.

However, git can not be downloaded the normal way since there is no directory listing. For this, you can either manually reconstruct git from known files or use a [pre-made script](https://github.com/kost/dvcs-ripper) to do that. Once you have .git, you can checkout the files and see the source of the website.

Looking at index.py, you see a "beta feature" that uses getattr to get information about a user. The `user` class has two attributes: `username` and `__password`. Accessing username works just fine, but the password does not! Why could this be? This is the fault of [name-mangling](https://stackoverflow.com/a/1301369). If you instead access `_user__password` for admin, you get the flag.

There was also an unintended solution using URL-encoded slashes for relative paths to access /proc/self/environ.
```http://web2.angstromctf.com:8899/files/..%2f..%2f..%2f..%2f..%2f..%2f..%2f..%2fproc%2fself%2fenviron```
