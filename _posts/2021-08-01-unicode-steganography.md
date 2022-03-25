---
layout: post
author: Graham Thompson
---


# Adventures with Text Steganography

So recently I revisited some old code I had messed with at a job for embedding
information into text. The goal at the time was to find a way to track users 
who were copy/pasting text from the site into other sites. For tracking file types 
like word documents, images, pdfs and etc. there is a wealth of metadata associated 
with the file. It's very easy to tag files you are generating or to get yourself 
in trouble with files that have more [information](https://blog.erratasec.com/2017/06/how-intercept-outed-reality-winner.html) associated with them than is 
visible. Text data is trickier. There isn't really any 'metadata' in utf-8 or ascii 
that can be used as a watermark. But the unicode standard is vast and infinite and 
there are a number of ways you can subtly alter the content of a text to encode 
information. 

A couple of good review papers on the subject:

[[Ahvanooey, et. al, 2017]](https://www.hindawi.com/journals/scn/2018/5325040/)

[[Ahvanooey, et. al, 2019]](https://pubmed.ncbi.nlm.nih.gov/33267069/)


## Simple Python Package

While playing around with some techniques I decided to put together a small python 
package to show off some of the simpler methods. You can try it with pip (python3 only)

```bash
pip install pyUnicodeSteganography
```

Or you can grab the package from my github.

[github](https://github.com/grahamwthompson/pyUnicodeSteganography)


## Watermarking and Message Hiding 

Two main uses of text steganography are watermarking or message hiding. The method
is more or less the same but the requirements can be different. Watermarking can 
require the watermarked text to not change any words or overall meaning which 
precludes some methods.

### Watermarking

If you need to watermark some plain text, such as for a website, text steganography is pretty much the only way. Can also be used as an additional watermarking method for file formats that include text such as word documents or source code. So if you are planning to copy and release some sensitive document don't think you are out of the woods just by scrubbing file metadata. There are a number of fairly subtle ways text can be altered to include a watermark. 

Here's a fairly simple example of how you can watermark some text using the hash 
of some identifying information (user id) with the default zero width character 
method from my package. It's a good idea to use the hash rather than the id. The 
encoding method is not terribly complicated so a clever actor could decode an id 
and modify it to frame someone else. 

```python
import hashlib
import pyUnicodeSteganography as usteg

def watermark_user(user_id, my_string):
    '''
    Simple watermarking scheme
    embed the hash of some user id into a text string with zwc steganography
    '''
    byte_user_id = user_id.to_bytes(2, byteorder='big')
    user_id_hash = hashlib.sha1(byte_user_id).hexdigest()

    return usteg.encode(user_id_hash, my_string)

important_ip = "my secret to making great omlettes is a 1 to 1 ratio of eggs to sticks of butter"
user_id = 123

watermarked_ip = watermark_user(user_id, important_ip)
```

### Message Hiding

Message hiding is the other interesting application for this technique. Passing a 
concealed message between two parties without revealing to any observing parties 
that the message was present. Text steganography is pretty useful for this in the 
case where both parties are passing messages over a platform that allows mostly 
text but may be observed by third parties such as a social media platform. This is 
not the same as encryption, an encrypted message is intended to keep your message 
contents safe but won't by itself stop an observer from knowing a secret message is 
being sent.

```python
import pyUnicodeSteganography as usteg 

secret = "a did it"
cover = "for this example imagine we are sending a secret message over a fairly restrictive messaging platform that removes a lot of zero width characters. We can use the lookalike method instead which substitutes visually similar printable characters instead."

output = usteg.encode(secret, cover, method="lookalike")
print(output)

# other side...
decoded = usteg.decode(output, method="lookalike")
```

## Types of Text Steganography 

I was interested to learn while working on the package that there are quite a few 
methods of text steganography. They can be very generally categorized as methods 
which conceal information by inserting or changing unicode characters without 
altering the visual appearance of the text.

Examples of weird character methods:

- zero width character steganography methods, invisible characters are inserted into the text encoding the secret message but are not generally visually rendered in browsers or editors 

- replacing space characters with alternate unicode space characters which are not 
visually distinct 

Examples of text altering methods:

- changing words in a text to synonyms to encode information

- using common mispellings to encode information 

There are also some methods which are not so easily categorized but might be 
useful in particular circumstances. For example you use the vast space of 
unicode emojis to encode secret messages. 

## Detecting Text Steganography

For watermarking, methods that use weird unicode characters without altering the 
text are the most common. Many of these methods are not that difficult to detect. In english text there's no valid reason for most of the weird unicode used to be present. A simple check like the following can pick up most methods.

```python
blacklist = ['\u200c', '\u202c', '\u200e'] # and etc...

def detect_trickery(str):
    for b in blacklist:
        if b in str:
            return True
    return False
```

Methods that modify the text itself without using special characters aren't as trivial to detect and are better methods for message hiding if the text is examined
in any detail. At the very least some sort of frequency analysis would be required.

## Countermeasures for Text Steganography

1) Remove everything other than printable ascii characters 

```python
import string

fishy_string = "h‌‍‏‍e‌‎‎‍l‍‍‎‍l‍‎‏‍o‌‌‎‌ ‏‍‎‍m‏‏‎‍y‌‍‏‍ ‌‌‎‌f‍‏‎‍r‍‍‎‍iend, how is the weather today?"
out = ''.join([c for c in fishy_string if c in string.printable])
```

2) optical character recognition 

- convert text to pdf or other image format 
- ocr to recover text (converts lookalike chars and removes non-printing chars)

3) retype message 

- labor intensive
- guarantees no special characters/formatting are retained
- still won't disrupt methods that use language structure unless you alter word choice and etc. during transcription

## Wrapping Up

Writing the package was a fun little project and I got a little more familiar with 
pip and the python packaging tools while working on it. I mostly implemented a 
couple simple steganography methods with special characters. If I continue 
playing with the subject in the future I think I'd like to attempt some of the 
language based methods which I think are more relevant for message hiding. 

![](https://bunnylab.net/img/blog-entry-8-21.png)