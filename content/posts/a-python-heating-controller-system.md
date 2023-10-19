+++
title = 'A Python Heating Controller System and How I Plan to Update it.'
date = 2023-09-17T21:28:39+01:00
draft = true
+++

## Controlling an Oil Based Heating System With Python? A story of how I got into Programming.

If you've ever wondered how you might control your central heating with Python, then this article is for you!. Or, as is infinately more likely, you've not wondered this but your interest is now peaked, then it is also for you. This article is the first in a series I have planned around the theme of this longterm heating system project  and more generally my journey with coding. Within the series there will be several different types of articles, from how-to's to more discussion based and conversational ones. This article, however, is one of the longer ones and serves to provide the initial background and context to the project and the series as a whole. As such, it is written in a more narrative style than some of the others will be, and if that doesn't appeal to you, please feel free to skip ahead to the next in the series. (Link to be added).

If you're still here though, welcome to a slightly more open and honest 'stream of conciousness' type post than I intend to mostly write on this site going forward; I hope you enjoy the ride. Please note, the code in this article is provided **as is** from a time when I was **not very good at programming**. The intention is transparency, honesty and just generally for the narative of the journey from then until now, but it should not be used by any beginner programmers as an example of *good code*. Future articles, I hope, will provide more informational and educational content for those getting started in programming, but *please, please,* do not learn any of my bad habits from the code shown in this article!


> **Important**: I do not recommend attemmpting anything similar to this project on a gas based system. Oil fired systems are quite a bit simpler and generally lower risk than gas ones and only qualified gas engineers should *ever* perform any work on a gas boiler.  

  
    

## But... why?

First of all, I would like to to address the *why?*. Long story short, I like the cold, and shortly after I moved into my house *as a single man*, the boiler broke, and I never bothered to get it fixed. A second important detail, as you'll likely have already guessed, is that I am a classic, unashamed, nerd.

Fast forward a few years from then, life had changed somewhat, as it tends to do. My (then) girlfriend was planning to move in with me and she is not quite as fond of the cold as I am. I begrudgingly had to get the boiler fixed. However, in the intervening years (yes years), I had done some DIY renovations which included the removal of part of a wall, crucially where the **heating controls** had been situated. 

Clumsily, I had misplaced the circuitry from the control unit well before I ever considered I would one day, umm... have a wife. With the need to keep things on a tight budget (I fixed the actual boiler itself on my own, with a second hand burner, some literal digging [it's a long story] and youtube videos), I wasn't keen on buying a new heating controller after seeing the price of even the most basic models. My house is not huge, and has a single loop in it's heating system, no separate loop for hot water or anything. So my boiler simply needs to be told to turn on, or off... nothing more. The cheapest controller I could find was about £50, and from memory only performed timer based controls. That just didn't make sense to me, I could do that with a £9 raspberry pi (zero) I thought. Then I thought some more... 

Yes, I actually could make my own cotroller, and likely a more feature-rich one for a fraction of the cost.

This all excited the nerd in me too, which helped with motivation, as I was already dreading the impending localised climate change about to occur in my own home and was really putting off getting the heating working to the last possible minute (when my now wife moved in).

So rather than buy a new heating controller, I found a suitably rated (30A) relay board from my stock of miscelaneous electronics components and ordered a digital, I2C thermometer. With that I could provide some basic thermostatic control as well as time based controls of the heating system's burner. All the wiring was already in the wall, I just needed to switch it on or off.

### The Original Version

My initial version of the 'Heating Controller' was definitely not a masterstroke of genious, or indeed really code pleasant to look at or even be aware exists. Back then, electronics was more of my hobby, (along with synthesisers) and coding was very much a nerdy extension of that, but not what I would call, a *skill*. And like many hobbies of mine in the past, it often took several attempts to get into a flow and actually produce anything of any value. Also, the eagle eyed of you may notice something a little bit lacking in the below code...


```
#!/usr/bin/python3
#v1.0
from time import sleep
from datetime import datetime
import board
from adafruit_bme280 import basic as ADA_bme280
import digitalio
import sqlite3



relay = digitalio.DigitalInOut(board.D14)
relay.direction = digitalio.Direction.OUTPUT
i2c = board.I2C()
bme280 = ADA_bme280.Adafruit_BME280_I2C(i2c)


#also, log temperature and store in postgresql db


temp = 20

#loop ---------------
while True:
    temp = bme280.temperature
    con = sqlite3.connect('temp_record.db')
    cur = con.cursor()
    #timestamp = datetime.now()
    print(datetime.now(), " - ", temp)
    if temp < 19:
        relay.value = True
        command = "INSERT INTO temp_history VALUES (\'" + str(datetime.now()) + "\',\'" + str(temp) + " C\', 1)"
        cur.execute(command)
    elif temp > 20.5:
        relay.value = False
        command = "INSERT INTO temp_history VALUES (\'" + str(datetime.now()) + "\',\'" + str(temp) + " C\', 0)"
        cur.execute(command)
    con.commit()
    con.close()
    sleep(30)

    #put temp in db
#loop end -----------
```

Yes, there is no implimentation to actually turn the burner on or off here. This is nothing more than a temperature logger. From the file metadata, I can see this file came a full month before the next iteration... You can see how much of a priority heating was for me back then!

The above code is the earliest version I can find on the original raspberry pi (I was not familiar with git at that point in time so we are limited to file metadata archealogy).
I recall at the time, I was just learning about classes and OOP and the like, so I refactored it into what you see below in the next version, and crucially added the very much needed functionality to actually turn the boiler on or off... 


```
#!/usr/bin/env python
from time import sleep
from datetime import datetime
from datetime import time as T
import board
from adafruit_bme280 import basic as ADA_bme280
import digitalio
import sqlite3

#functions
#set_temp()
#set_night(temp)
#boost()
#burner_state()

class heating_control:
    """Central Heating Controller Class"""
    #to add connection to postgresql db for logging amd loading configs
    def __init__(self):
        self.relay = digitalio.DigitalInOut(board.D14)
        self.relay.direction = digitalio.Direction.OUTPUT
        self.i2c = board.I2C()
        self.bme280 = ADA_bme280.Adafruit_BME280_I2C(self.i2c)
        self.day_temp = 21.0
        self.boost = True
        self.night_temp = 17.0
        t_start = T(hour=6)
        t_end = T(hour=22)
        self.day_time = { "daystart" : t_start , "dayend": t_end}
        #self.burner =

    def burn(self,bool): #when true relay is on
        self.relay.value = bool

    def check_state(self,last_state):
        #temp = self.bme280.temperature
        #time = datetime.now()
        self.update_state()
        #need to get datetime.datetime.now() and convert to datetime.time
        if self.boost:
            return True
        elif self.time.time() > self.day_time['daystart'] and self.time.time() < self.day_time['dayend']:
            #is currently day_time
            if self.temp < self.day_temp:
                return True
            elif self.temp > self.day_temp + 1.0:
                return False
            else:
                return last_state
        else:
            #must be night then
            if self.temp < self.night_temp:
                return True
            elif self.temp > self.night_temp + 1.0:
                return Falses
            else:
                return last_state

    def update_state(self):
        self.temp = self.bme280.temperature
        self.time = datetime.now()

    def log(self):
        con = sqlite3.connect('/home/pi/heating-system/temp_record.db')
        cur = con.cursor()
        command = "INSERT INTO temp_history VALUES (\'" + str(self.time) + "\',\'" + str(self.temp) + " C\', " + str(self.relay.value) + ")"
        cur.execute(command)
        con.commit()
        con.close()

#mainloop
#check state (temp for time period, override, )
#action (turn burner on/off)
#log (output timestamp, temp, control and burner states to db)

#instantiate controller
if __name__ == "__main__":

    heater = heating_control()
    heat = False
    while True:
        heat = heater.check_state(heat) #if cold returns true, if hot returns false, if between temps returns last state
        heater.burn(heat) #check_state returns bool for burn - on/off

        heater.log()

        sleep(60) #process happens roughly every minute + time taken to process python code)
```


At this point in time, I had just started a new role as an 'IT admin' at a family run SME in the distribution space. I was making a move away from a larger company (major television network in the UK) where I had found myself under-challenged and under-utilised. I was at the time looking to really broaded my skills professionally speaking and had started to look at Python in a professional context in order to automate certain tasks. As such, I was learning certain things at a fairly rapid pace, but as you can see, the code above leaves a lot to be desired. Looking back at it now, it's clear that the temperature and time settings had to be manually set in the code itself, but more than that, whether boostr