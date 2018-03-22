# Weird Message, 100 pts

This challenge gave a single text file, containing a message. From the hint `xn--`, it could be determined that this was punycode. Punycode can be decoded in Python with:
```python
"<string>".decode("punycode")
```

When you do this, you find that the part of the message after the last dash has been removed, and the one before it has changed. Since punycode appends a dash each time a string is encoded, you know that this was probably encoded many times. Trying to decode again, however, gives an error because of unicode characters. Upon further inspection, the end of the string now has [homoglyphs](https://www.irongeek.com/homoglyph-attack-generator.php). Replacing these with the similar ASCII characters, the string can be decoded again. However, since there are about 200 dashes, the string was probably encoded 200 times. Decoding by hand would take a very long time. Luckily, this is not too hard to automate. You can either build up a mapping of homoglyphs to regular characters manually, or use a [prebuilt list](https://github.com/codebox/homoglyph/blob/master/raw_data/chars.txt) like I did.

After decoding fully, you get the flag.
