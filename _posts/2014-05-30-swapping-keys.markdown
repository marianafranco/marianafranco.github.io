---
layout: post
title:  "Swapping Keyboard Keys"
date:   2014-05-30 22:45:00
tags: xmodmap linux
---

Some weeks ago, my laptop `k` key just stopped to work. By chance, the `k` letter is not used to much in Portuguese and I could wait some days to see whether it returns to live or not. With its death declared, I started looking for some ways of mapping another key to the `k` key in Ubuntu, when I found about the `xmodmap` utility.

<!--more-->

In Ubuntu, and all Linux systems using the X11 window system, the keyboard configuration is completely accessible in the form a customizable keymap that can be modified through `xmodmap`. So I decided to use the `xmodmap` utility to map the `AltGr` + `j` to the `k` key in my laptop. These are the instructions that I followed:

### 1. Keymap

Find the current keymap table by typing the following command:

``` bash
$ xmodmap -pke

keycode  42 = g G g G
keycode  43 = h H h H
keycode  44 = j J j J
keycode  45 = k K k K
keycode  46 = l L l L

```
Each keycode is followed by the keysym it is mapped to. Each keysym column in the table corresponds to a particular combination of modifier keys:

```
1. Key
2. Shift + Key
3. Mode_switch + Key
4. Mode_switch + Shift + Key
5. AltGr + Key
6. AltGr + Shift + Key
```

Therefore, the above example indicates that the keycode 44 is mapped to the lowercase `j`, while the uppercase `J` is mapped to keycode 44 plus `Shift`, and so on.


### 2. AltGr keycode

Looking to the keymap table I could not identity any keycode mapped to the `AltGr` key, so I used the `xev` command to find the keycode that correspond to the `AltGr` key in my laptop.

The `xev` is a program that displays X events. For any key that you hit, the console where you started `xev` will display an event that includes the keycode for that key.

This was what I got when pressed the `AltGr` key in my laptop:

``` bash
$ xev

KeyRelease event, serial 40, synthetic NO, window 0x4200001,
    root 0x90, subw 0x0, time 41801531, (829,465), root:(879,517),
    state 0x80, keycode 108 (keysym 0xfe03, ISO_Level3_Shift), same_screen YES,
    XLookupString gives 0 bytes: 
    XFilterEvent returns: False
```
So the `AltGr` key on my keyboard has keycode 108 and it's mapped to the `ISO_Level3_Shift`:

``` bash
$ xmodmap -pke | grep 108

keycode 108 = ISO_Level3_Shift NoSymbol ISO_Level3_Shift NoSymbol
```

### 3. ISO\_Level3\_Shift vs Mode_switch

I tried to change the keycode 44 to map the `AltGr` + `j` to the letter `k` trough the following command:

``` bash
$ xmodmap -e "keycode 44 = j J j J k k"
```

But this did not work. In fact, I noticed that there is no keycode in my keymap where the 5th and 6th column of the keysym are used. That is, there is no value mapped to combinations that use the `AltGr` modifier.

I also did not find any keycode mapped to the `Mode_switch` modifier. Then I tried to change the `AltGr` key to be mapped to the `Mode_switch` instead of the `ISO_Level3_Shift`, and to change the keycode 44 to map the `Mode_switch` + `j` to `k`:

``` bash
$ xmodmap -e "keycode 108 = Mode_switch NoSymbol Mode_switch NoSymbol Mode_switch"
$ xmodmap -e "keycode 44 = j J k K"
```

This worked, and now in my laptop the `AltGr` + `j` outputs `k`, while `AltGr` + `Shift` + `j` generates the uppercase `K`.

However, what exactly is this `Mode_switch` key? And what is its difference to the `ISO_Level3_Shift`?

After google a lot, the best explanation that I found was [this one](http://unix.stackexchange.com/questions/55076/what-is-the-mode-switch-modifier-for) that I pasted here:

> `Mode_switch` is the old-style (pre-XKB) name of the key that is called `AltGr` on many keyboard layouts. It is similar to `Shift`, in that when you press a key that corresponds to a character, you get a different character if `Shift` or `AltGr` is also pressed. Unlike `Shift`, `Mod_switch` is not a modifier in the X11 sense because it normally applies to characters, not to function keys, so applications only need to perform a character lookup to obtain the desired effect.

> `ISO_Level3_Shift` is the XKB version of this key. Generally speaking, XKB is a lot more complicated and can do some extra fancy stuff. XKB's mechanism is more general as it allows keyboard layouts to vary in which keys are influenced by which modifiers, it generalizes sticky (`CapsLock`-style) and simultaneous-press (`Shift`-style) modifiers and so on.

So take care before change your `AltGr` keymap. In my case, I did not notice any change in the keyboard operation and all keys are working correctly.


### 4. Activating the .Xmodmap at startup

After restart the laptop, I noticed that my changes have not take effect. If you want your xmodmap changes to run each time you log in, you need to create a file called ~/.Xmodmap with your modifications to the default keymap. For example, in my case the ~/.Xmodmap file looks like this:

``` bash
keycode 108 = Mode_switch NoSymbol Mode_switch NoSymbol Mode_switch
keycode 44 = j J k K
```

You can check if you ~/.Xmodmap file is correct by calling:

``` bash
$ xmodmap ~/.Xmodmap
```

And that’s it! :)

