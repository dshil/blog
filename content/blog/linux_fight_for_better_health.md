---
title: "Linux: fight for better health"
date: 2019-03-16T09:53:26+03:00
draft: false
---

## Preface

For a long time, I was working in the office in front of my lovely
Dell 24 Monitor but recently I have been starting working more in front of
my laptop. It is MacBook Pro 17 Late 2011, very old guy with Fedora on a board.
Suddenly I started having more and more headaches. It was sad and I
decided to visit a doctor. Well, we made tons of analysis, checked eyes,
checked vessels of the brain, checked the neck and a lot more and surprisingly
we did not find anything. Doctors said that I just tired and I need to make
a break and take some medications. I did it and but it did not help. I was not
able to work in front of my laptop for longer than 25 minutes. It was a time
when the real fight has begun.

## Pulse Wave Modulation

One day I found a [post](https://www.reddit.com/r/linux4noobs/comments/2ygdpc/eyestrain_problems_while_using_linux/) on Reddit where one guy had the similar
problem that I had. From this post, I got the idea that some people have
headaches because of PWM (pulse wave modulation). PWM is a mechanism that
controls the backlight of monitors by turning on and off led flashlights very
quickly. Happy owners of the Intel GPU card [can control](https://wiki.archlinux.org/index.php/backlight#Troubleshooting) this annoying PWM.

For the first time, I checked the current PWM frequency and it was 0x0:

```sh
sudo intel_reg_read 0xC8254
0x0
```

From this result, you can get the idea. When you decrease the brightness of the
screen to the minimum level you will be under higher PWM influence. For some
people it is OK but for a little group of people, including me, it will cause
eye stress and headaches. As a result, we need to [eliminate LED screen flicker](http://devbraindom.blogspot.com/2013/03/eliminate-led-screen-flicker-with-intel.html)
when we work in front of a display with a brightness which differs from the
maximum (PWM will be eliminated automatically for the maximum brightness).
You just need to choose the desired PWM frequency:

```sh
sudo intel_reg_write 0xC8254 0x3d103d1 # 1000Hz in my case
```

Of course, not all of us have the Intel GPU on board. PWM usually appears on
middle or low brightness level. As a result, you can set the brightness level to
the maximum and use [redshift](http://jonls.dk/redshift/) or [iris](https://iristech.co/how-iris-reduces-pwm-flicker-medium/) to adjust the brightness. In reality, these
tools do not change the hardware brightness, they adjust the gamma settings
instead. For some people, it might work. Personally, I do not like this approach
because I have even more headaches because of mixing maximum brightness and
gamma change.

## Screen

So far, so good. We fixed the PWM problem and as a result, I felt much better
but some level of headaches was still there. As you remember I have a Fedora
Linux on the board with I3 tiling window manager and I did not perform any
screen and fonts adjusting after replacing the original OS X operation system.
As you might understand my fonts were no more than terrible. For development
I use [Terminus](http://terminus-font.sourceforge.net/) and I am pretty happy with it but the remaining system looked
very awful. So, take a hammer in a hand and make own life more pleasant.

### DPI

XServer usually sets some default value, e.g., 96 is frequently used. All of us
have various displays that have different resolution, hight, width parameters.
As a result, we can't simply use the same DPI value just because it is somewhere
near our true DPI. There is a good [instruction](https://askubuntu.com/questions/197828/how-to-find-and-change-the-screen-dpi) for DPI calculation. After doing short math I got the value 110 for my display parameters.
After all, you need to ensure that the following commands produce the same DPI:

```sh
xdpyinfo | grep dots
# resolution:    110x110 dots per inch

xrdb -query | grep dpi
# Xft.dpi:        110

grep DPI /var/log/Xorg.0.log
# [    26.888] (++) RADEON(0): DPI set to (110, 110)
# [    27.114] (++) modeset(G0): DPI set to (110, 110)
```

### Font Rendering

We adjusted the DPI. From the personal view, I did not see a huge difference
but probably my sensitive eyes recognized some changes. The fonts rendering was
still terrible. After a little bit of googling, I found an [instruction](https://wiki.manjaro.org/index.php?title=Improve_Font_Rendering)
for font rendering. After applying all suggested settings
the fonts rendering became very neat and my eyes started to get less stress.

## Conclusion

You must understand your system to survive. After playing a little bit with
settings I have found some patterns that allow me to work longer in
front of my laptop without any injuries:

* Zero glare on the screen
* Use low contrast themes everywhere (vim, urxvt, mutt and friends)
* Take a break at least after 1 hour of working
* Use 50% brightness
* Use 100% gamma
* Use 1000Hz PWM frequency

I still have a little bit of eye stress and headaches but I think that it is
a recovery period.
