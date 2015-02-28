---
layout: post
title:  "Downmixing overly separated Stereo Files with Sox"
categories: howto music mod sox linux amiga
---
I have a bunch of MP3s here with old Amiga MOD-music in them. If you don't know what that is, you are missing out on a
great chapter of computer music from the early 90s.
[Read more at Wikipedia](https://secure.wikimedia.org/wikipedia/en/wiki/Module_file).

There are many programs for playing the old MOD files, just see the Wiki entry. There is one problem though: If you have
these tracks not as MOD, but alreadys converted to MP3 or something like that, you might find that they sound a bit
weird, especially with headphones. You'll hear some of the instruments only on the left, and some only on the right. The
Amiga had 4 audio channels, 2 left, 2 right, and there was no overlap between them. If you just have speakers standing
somewhere that usually isn't a problem, but nowadays many people use headphones, and in that case it really sounds
weird.

One could simply downmix them to mono now, but that wouldn't sound very nice either. What we really want is to create a
balanced mix, with left and right fully present on the left and right, but a bit of each channels sound mixed into the
other channel. Personally, I found a mix of about 70% toward the "other side" to sound quite nice.

This can more or less quickly be done with any kind of sound editor like Audacity, but if you have more than two or
three such files, it quickly becomes tedious. Luckily, there are command line tools that easily lend themselves for
batching. One of them is Sox, the ["Swiss Army-knife of sound conversion."](http://sox.sourceforge.net/) To quickly
go through several directories worth of tracks, I wrote a small bash script that looks like this:

    #!/bin/bash
    target="adjusted"
    mkdir $target
    for file in "$@"
    do
      tmpfile="$target/$file.tmp.wav"
      oggfile="$target/$file.ogg"
      sox "$file" "adjusted/$file.tmp.wav" mixer 1,0.7,0.7,1
      oggenc -q 5 -o "adjusted/$file.ogg" "adjusted/$file.tmp.wav"
      rm "$tmpfile"
    done

I then simply change into the directory containing the files, say `downmix.sh *.mp3` and a few moments later I have a
subdirectory `adjusted` that contains fresh ogg files with the properly downmixed tracks.

If you just want to use this, go ahead, install the packages sox and libsox-fmt-all on your Ubuntu machine, copy &
paste the above script into a file called `downmix.sh`, `chmod u+x` it, and you're ready to go. If you want to know
what it actually does, read on for a quick overview.

The script iterates over all arguments, so you could call it with `downmix.sh file1 file2 file3` or `downmix.sh *.mp3`,
which is really the same thing, since the latter gets expanded by bash to the former. Sox is called for each file to
decode it into a WAV file, while using the mixer effect to, well, mix the audio channels. The argument for this effect
is a comma-separated list of relative values that tell it how to mix. `0` means "no sound", `1` means "100%". The first
number is left input on left output, the second is left input on right output. #3 is right input on left output and
finally #4 is right input on right output. My choice of `1,0.7,0.7,1` therefore keeps the left channel on 100% and adds
it with 70% volume to the right output channel too, thus moving it perceptually much closer to the center. The latter
two parameters do the same with the right channel - it's sent with 70% of its volume to the left output, and then with
its full strength to the right output.

Now I finally can listen to some of my favorite music properly with headphones.
