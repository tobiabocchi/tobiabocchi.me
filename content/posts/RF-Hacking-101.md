---
date: "2022-11-30T15:08:23+02:00"
title: "RF Hacking 101"
cover:
  image: "/img/posts/rf_hacking_101/rf.jpg"
---

This is my first blog post! With it, I would like to tell you a little bit about
how I got into RF hacking and what tools I use to analyze RF signals.

## My experience

Radio Frequency (**RF**) communications are used in a plethora of devices, the
first time I got in touch with RF communication was when in middle school I bought
a radio-controlled (RC) car.

![crystal](/img/posts/rf_hacking_101/rc_crystal.png)

The communication between the remote and the car used RF to RX/TX throttling and
steering, and the remote had a tiny crystal that you could swap with other remotes
to control different cars, the same crystal was on the car's receiver, you can see
what it looked like in the picture.

Growing up, I later discovered that switching crystals changed the operating frequency
of the remote, or the car, hence another remote could communicate with another car!

If I had to point out a specific moment that made me want to deepen my knowledge
about RF communication is when I stumbled upon Samy Kamkar's [OpenSesame](https://samy.pl/opensesame/).
I was fascinated by how well a repurposed kid's toy could perform as a penetration
testing tool, so I decided to start exploring the subject.

![opensesamine](/img/posts/rf_hacking_101/opensesamine.png)

Since then I have been trying to make my version of Samy's OpenSesame, my first
attempt was quite successful but only able to transmit/receive on a single frequency.
It is when I first started working on an updated version, using a TI CC1101 chip
(a transceiver capable of operating at multiple frequencies) that I saw Flipperzero's
Kickstarter and decided to back their project and stop working on mine. This is
what my project looked like, I built it using the transceiver, an OLED display and
a rotary encoder, all powered by an ESP32.

## Hardware Tools

My penetration testing arsenal has gotten bigger over time, but to get started with RF hacking you can do **very much** with **very little** hardware/software and money. Here I will describe a few tools that I use daily when doing RF-related tasks.

### SDR Dongle

![SDR dongle](/img/posts/rf_hacking_101/dongle.png)

This one is a **must-have** to get started! This affordable (~15$) USB SDR dongle
allows you to "listen" (**receive**) on frequencies ranging **from 500kHz up to
1.7GHz**. It can be used to determine on which frequency a communication takes place,
as well as **recording signals** for further analysis!

### Arduino modules

![Arduino Nano](/img/posts/rf_hacking_101/nano.png)

Arduino is an amazing tool, and it's great to get started with hardware hacking.
The one I am more familiar with is the Arduino Nano (the one in the picture above).
It's compact power efficient and cheap (~20\$ for the original, 5\$ for clones).

#### Single Frequency

![Arduino RX/TX modules](/img/posts/rf_hacking_101/arduino_tx_rx.png)

Using an SDR dongle for receiving signals can be fun, but it's when you become
able to **transmit** that the real fun begins. These extremely cheap (~1\$)
Arduino modules allow you to start hacking with a board as simple as the Arduino
Nano, they can only communicate over a single frequency but depending on where
you live you can decide whether to buy a 433MHz one (Europe) or a 315MHz one (US).

#### Multi-Frequency

![CC1101](/img/posts/rf_hacking_101/cc1101.png)

The evolution of these simple RX/TX modules is a **transceiver** a single module
capable of both receiving and transmitting and we can take it even a step further
with a multi-frequency transceiver: a very popular one is the **TI CC1101** and
as you might have guessed it can RX/TX on multiple frequencies. The CC1101 can be
found for quite cheap online (~4\$), and is the same chip used by Flipperzero!

### EvilCrow-RF

![EvilCrow-RF](/img/posts/rf_hacking_101/ecrf.png)

This open-source device developed by [Joel Serna Moreno](https://github.com/joelsernamoreno)
includes not one, but **two** CC1101 transceiver modules! Having two transceivers
opens the possibility to perform some specific rolling code attacks, as well as
scanning/listening to multiple frequencies at once when trying to figure out where
a signal is transmitted. I have not had much time to try out this device but I am
sure it will come in handy at some point.

### Flipperzero

![Flipperzero](/img/posts/rf_hacking_101/flipperzero.png)

When I saw the Kickstarter campaign for this device I knew I had to back it. I
supported the project by buying not one, but two Flipperzeros :)

Defining this as an RF hacking tool **only** is very diminishing, it has many
other capabilities: it implements a lot of other communication protocols such as
Bluetooth, NFC, infrared, and more! If you want to learn more about it head over
to their [website](https://flipperzero.one) where you can try to buy one, unfortunately,
they are often sold out.

## Software Tools

> Over time I have come to the conclusion that powerful hardware is useless
> without good software supporting it.

Now that we have seen the main **hardware** tools to transmit and receive RF let's
go over what **software** can be used to further analyze and reverse engineer the
signals we capture.

### gqrx

![gqrx](/img/posts/rf_hacking_101/gqrx.png)

This program, together with the SDR dongle, lets you record signals on different
frequencies in the form of audio files, it can also be used as a spectrum analyzer
to see what frequencies are used around you and sniff radio communications. It can
be configured to use different modulations and channel widths and supports a lot
of different hardware.

### Audacity

![audacity](/img/posts/rf_hacking_101/audacity.png)

Audacity, as the name suggests, is a program that handles audio files. I mainly
use it to analyze the signals recorded using gqrx. Zooming in with audacity, we
can see in the picture shows the recorded signal as a square wave, from the timeline
we can determine the duration of each pulse and figure out how ones and zeros are
encoded.

### URH

![urh](/img/posts/rf_hacking_101/urh.png)

Universal Radio Hacker (**URH**) is the most complex and complete software on the
list. It can read a multitude of different file formats and display them as square
waves. I am still learning how to take advantage of its full capabilities but it
looks very promising so far.
