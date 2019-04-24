# DOM Validator, 130 pts

This challenge had tons of unintended solutions - I'm sure people will make writeups for those. My intended solution was much simpler. Just change the URL from `https://dom.2019.chall.actf.co/posts/asdfasdfsadf.html` to `https://dom.2019.chall.actf.co/posts//asdfasdfsadf.html` and the relative source for DOMValidator.js no longer loads (404). This behavior is due to how express's static file serving works (double slashes are collapsed).

This XSS is then used to steal the admin's cookie, which has the flag.
