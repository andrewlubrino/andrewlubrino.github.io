---
layout: post
title:  "Rendering pixmaps with Xlib on Linux"
date:   2023-01-15
categories: jekyll update
---

# Rendering pixmaps with Xlib on Linux  

I've been doing all of my programming on a Chromebook these days. Chromebook has limited resources and, in general, I've been trying to do more with less. This mostly means cutting out libraries wherever possible. In some ways, my life is harder, but things are also simpler once you learn about how it all fits together in Linux.

I'm on a bit of a graphics programming kick recently. I knew that ray tracing could be simple, and could potentially be done without libraries not already included with a Linux distribution. I followed along with [Peter Shirley's Ray Tracing Tutorial](https://raytracing.github.io/books/RayTracingInOneWeekend.html) to get started. Peter outputs renderings to PPM, probably the simplest image format. This is minimalistic for sure, but not great for quickly debugging images. I wanted a program that would display an image as soon as it was finished with rendering, so I decided to work with Xlib.

### Using the Xlib library  

Xlib is the client library for the X Window System, the standard windowing system for many Linux distributions. There are plenty of large, bloated graphics libraries that build on top of Xlib, but I wanted to keep things simple by working directly with the library. Because Xlib is fairly "low-level" as far as graphics libraries go, searching for documentation is painful. In particular, I had trouble figuring out how to render an array of pixels -- a pixmap in Xlib lingo -- into a window. I want to document the correct way to do this.

### Some computer color basics  

Most modern computers have what is called "True Color". In modern systems, colors are specified by a 24-bit integer split into 3 components -- red, green, and blue. The number of bits used to specify a color is sometimes called the color "depth". Each color channel is an 8-bit unsigned integer that ranges from 0 to 255. This allows for more than 16 million colors. There are some important, non-obvious issues that are easy to overlook when assigning colors to a pixmap inside a client program.

The first major issue is that we cannot really have a 24-bit color. For various reasons that are not clear to me, it's easier for the computer if colors are `int` -- i.e., 32 bits. This means that we need to pad our 24-bit color depth with a byte of 0x00 on the right. The second major issue is that, by convention, colors are specified in the order red, green, blue -- just think of the acronym RGB as proof of this. And the colors *are* actually in this order, but, depending on the endianness of your hardware, you might have to specify the channels backward (so, really BGR). On my Chromebook, which has an Intel processor, I have to specify the color channels backward because Intel hardware is little endian.

### Let's see the code  

Below is part of the final code that I used to render a pixmap to a window:

```
int width = 100;
int height = 100;

unsigned char data[width * height * 4];

for (int i = 0; i < width * height * 4; i+=4){

    data[i] = 255; // blue channel
    data[i + 1] = 0; // green channel
    data[i + 2] = 255; // red channel
    data[i + 3] = 0; // dead byte that is always zero

}
```

The height and width of our image is 100, so we'll have 100 x 100 pixels in this pixmap. We can't just declare an array of `int`s for each pixel. Each of the 3 RGB channels and the dead pixel must be given its own address (in this case achieved by giving the array the type of `unsigned char`). Then we simply loop through the array and assign each channel with the numbers that will achieve the color we intend.

```
XImage *image  = XCreateImage(d, DefaultVisual(d, 0), 24, ZPixmap, 0, (char*) data, width, height, 32, 0);

Pixmap pm = XCreatePixmap(d, w, width, height, 24);

XPutImage(d, pm, gc, image, 0,0,0,0, width, height);
```

This next chunk of code creates the pixmap. Most of this is self-explanatory, but there are a few things to note here. In the call to `XCreateImage()` the third argument is the color depth, which, in this case, is 24 as discussed above. The fourth argument tells the program how to read the array of data specified in the fifth argument. Here we have `ZPixmap` which means that 4 consecutive spaces in memory will be assigned to one pixel. The other option here is `XYPixmap`. The documentation is not exactly clear on what `XYPixmap` is, but one [StackOverflow post](https://stackoverflow.com/questions/28323960/what-is-the-difference-between-xypixmap-zpixmap-ximage-xshmimage-and-bitmap-i) says this:

>\[F\]or an RGB image you have all the red channel data followed by the green and then blue instead of the more traditional RGB triplets

If you are interested in seeing the ideas here fully implemented, you can check out the `DisplayWindow` class in my [simple raytracing project](https://github.com/andrewlubrino/raytracer).