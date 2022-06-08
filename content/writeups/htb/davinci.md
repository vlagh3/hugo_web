---
title: "Davinci Hackthebox"
date: 2019-04-02T12:48:05+03:00
draft: true
---

Once you unzip the file we are given 3 files:
- monalisa.jpg
- Plans.jpg
- Thepassword_is_the_small_name_of_the_actor_named_Hanks.jpg

So let's start with the first one. I usually run strings with tail for a quick overview, which gives the following output:

```
S/-R?
N64dP4
x^[E
+hI$}
"bMe,L
fGS:>
$`"_
#d)$
Mona.jpgUT
famous.zipUT
```

From this output we can easily see that there are 2 files embeded in the jpg. So I ran _binwalk_ in order to extract them. After running _stegsolve_, _fcrackzip_ and other tools on those files, I couldn't find anything. Therefore, I decided to take a look at the other files in hope that I will get some usefull information that will let me unzip the file or provide a password to _steghide_ on the **Mona.jpg** file.

By running strings on Plans.jpg, the following [link](https://www.youtube.com/watch?v=jc1Nfx4c5LQ) was hidden. The video is related to Picasso's Guernica, but nothing interesting, so I just kept it in mind.

The next file is quite interesting since it's hinting us that a password for something is the small name of the actor, and once you open it the name Tom is showed. The first thing that I tried was to provide Tom as a password to the zip file, but unfortunately it didn't work. However, after running this little bash snippet

```bash
for file in .
do
        steghide extract -sf "$file" -p "TOM"
done
```

**S3cr3t_m3ss@g3.txt** was revealed:

```
Hey Filippos,
This is my secret key for our folder.... (key:020e60c6a84db8c5d4c2d56a4e4fe082)
I used an encryption with 32 characters. hehehehehe! No one will find it! ;)
Decrypt it... It's easy for you right?
Don't share it with anyone...plz!


if you are reading that, call me!
I need your advice for my new CTF challenge!

Kisses,
-Luc1f3r
```

After, I just throwed the md5 hash to CrackStation and the password `leonrado` was found. So, upon extracting the famous.zip file we are given an attractive mona lisa jpg image:

IMAGE OF JPG FILE

Going again through the same routine on this file I wasn't able to find anything at all. Something was strange... After all, the video hinted on the second file wasn't of any use until now. So I did what every normal person would do and used the title of that video ("Guernica") as a password for steghide, and the key appeared.
From there was just a matter of decoding the base64 and then submitting the flag


