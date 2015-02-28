---
layout: post
title:  "Putting CapsLock to good use"
categories: howto editing windows linux
---
One of the most useless keys on any computer keyboard is `CapsLock`. It might be useful IF YOU ARE PRONE TO YELLING AND
USING LOTS OF !!!!!11!!!!!, but in that case I don't want to read anything you write :-)

One can put `CapsLock` to a better use by remapping it to produce a different signal. What signal that may be is up to
your personal needs. Personally, I have it remapped as `AltGr` (right Alt key). On a German keyboard you have to rather
awkwardly press `AltGr`+`7`, `8`, `9`, `0` to get the braces and brackets `{`, `[`, `]`, `}` we developers need so
often. With the remapping, I can comfortably press the `CapsLock` key with my left hand and the key for the brace I
need with the right.

How is it done? For Linux (or anything with X windows, probably), you just create a file called `.Xmodmap` (note the dot
in front!) with the following contents:

    keycode 66 = ISO_Level3_Shift NoSymbol

After logging out and back in, 'Xmodmap' might ask you to confirm you really want the remapping, and then You'll have
`CapsLock` working as `AltGr`.

For Windows, you need to edit the registry:

    Windows Registry Editor Version 5.00
    [HKEY_CURRENT_USER\Keyboard Layout]
    "Scancode Map"=hex:00,00,00,00,00,00,00,00,02,00,00,00,38,e0,3a,00,00,00,00,00

For convenience, you can copy & paste the above text into a file called `caps2ralt.reg`, double click it, and Windows
will automatically import the setting into its registry. One reboot later you too will have your `CapsLock` uselessness
fixed. (The registry is a fragile and horrible piece of cruft, it may have a nervous breakdown any time you touch it.
You edit it at your own risk!)
