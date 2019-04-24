# GiantURL, 190 pts

This challenge gave a "URL lengthener" that also had a report link, where the admin would visit the lengthened URL and click on the link to follow the redirect. 

You needed to change an admin's password through a POST request to `/admin/changepass`. At first it looked like this could be done just with CSRF, but that wouldn't work because server set cookies to be `SameSite: Lax` and the cookie was not sent with cross origin POST requests.

Instead, you had to use the [ping](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/a) attribute on the link you sent (since the href attribute wasn't quoted you could break out of it with a space) and set it to `/admin/changepass?password=<some valid password>`. Since the PHP used `$_REQUEST` both GET and POST parameters were used to get the sent password.

After the admin clicked on the link the admin password would be changed and you could log in and get the flag.
