---
layout: post
title: IsMyWasherRunning, The washing machine app
tags: [android, apache, washing, machine, january]
---

![castle](https://localwiki.org/chico/the%20castle/_files/Castle.jpg)

In my Junior year of college, from 2015-2016, I lived in a 3 story house with 11 other men.  It was not a fraturnity, but we were a routy group.  Near the door there was a plaque that read 'The Castle, est 1906'.  That is what we called where we lived.  The Castle.

I did not live in the castle, I lived the the basement of the Castle.  I lived in the dungeon.  The dungeon was great, it was a completely seperate unit from the main house.  This was a great setup to avoid the strife and drama of the house, but not so much for the laundry situation.  

The washer and dryer were on the third story of the house.  Which, in my opinon, is insane.  Mostly because it was 2 stories above myself.  With this situation I ran into 2 main problems besides the distance that I needed to travel.  The first of these problems is that I often forgot to set a timer for my laundry, which resulted in my wet close sitting in a pile attop the washer for several hours.  Secondly, if I wanted to check if the washer was avaiable, I would have to do it physically or rely on our group chat.

I soon devised a solution.  Onhand I had my Android Phone, and a RaspberryPi.  I got to work.

What I devised was a very simple setup.  I duct-taped the RPi to the washer.  Attached to a GPIO port of the RPi was a vibration sensor<sup>[[1]](#1)</sup>.  This worked spectacularly.  Every vibration set off the sensor!

On the RPi I installed the appache web server.  With the help of the RPIO Python library I had the power to change the homepage of the server whenever the RPi recieved an impulse.  Set on a 30 second interval, I now had a reliable way to tell if the server was on.

This project took way too little time<sup>[[2]](#2)</sup>.  I was not fond of hitting the static IP of the RPi through my browser, at all.  The solution?  A simple android app.  It ended up being basically one view that connected to the static IP of the RPi, and displayed the homepage.

This worked.  It worked very well.  Though, there was a problem.  It was the enviornment, and not the development enviornment.  I soon burned through 4 vibration sensors, each getting knocked off by a stray sock or shirt fresh out of the washer.  The fruits of my labor were wasted.  I'm surpised the exposed RPi was not fried.

For a short 3 weeks I had time to develop the setup, not have to check upstairs to do my laundry, and I didn't have to remember to set a timer.  It was blissful.

The source is up on my github.

Keep it real,

Matthew

#### SuperScripts
<sup id="1">[1]</sup> Heres a [link](https://www.adafruit.com/products/1766?gclid=CPTf_oWh49ECFYKJfgodqswE9Q)<br />
<sup id="2">[2]</sup> It took around an hour of writing the python, and 3 days to get the damn sensor working.
