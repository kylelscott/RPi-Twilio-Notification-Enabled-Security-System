# RPi-Twilio-Notification-Enabled-Security-System
This repository contains the code to set up a Raspberry Pi Security System with Twilio text notifications using a Raspberry Pi, RPI Camera Module, OpenCV, and the Twilio Python API

***Note:*** *This project was done with the help of an existing project by Adrian Rosebrock, which can be found* [here](https://www.pyimagesearch.com/2015/06/01/home-surveillance-and-motion-detection-with-the-raspberry-pi-python-and-opencv/)

## Background
This project utilizes the image processing capabilities of the widely used [OpenCV](https://docs.opencv.org/master/d0/de3/tutorial_py_intro.html) Python library. It also takes advantage of the Twilio's SMS Python API, which allows users to create responsive and useful notifications in their applications using SMS messaging and Python. 

## Getting Started
Before we get started, make sure you have all the necessary hardware:
- Raspberry Pi (I am using a [4B with 2GB RAM](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/), but would recommend a model 4 or 3 with more than 2GB RAM just to make sure you dont overload the system, a Zero should work too)
- Raspberry Pi Camera board (I am using a [Camera Module V2](https://www.raspberrypi.org/products/camera-module-v2/)
- 5V 3A Power Suppy [(this one works)](https://www.raspberrypi.org/products/type-c-power-supply/)

Now, we can install all the necessary dependancies. First, set up your Raspberry Pi using an external monitor or headless mode (if pursuing headless mode I would recommend reading/watching a tutorial for how to set it up). 

***Note:*** *I actually programmed this project, which is running at my house, from my parent's house using TeamViewer Host for the Raspberry Pi. Highly recommend installing this and setting it up if you want to get this set up before a trip but don't have the time to set up at home. Here are some [instructions](https://www.teamviewer.com/en-us/download/raspberry-pi/) on how to install TeamViewer for the Raspberry Pi.*
First, run the following in a new terminal:
```
sudo apt-get update
sudo apt-get upgrade
```
It's always good practice to update the OS before any development. 
Next, we'll pip install some packages using pip/pip3
*if you dont have pip/pip3 installed, use *```sudo apt install python3-pip```*for the Python3 and*```sudo apt install python-pip```*for Python2*

1. Install ```imutils``` using the following command:
```pip3 install --imutils```
2. Install OpenCV using the following command:
```pip3 install opencv-contrib-python```
***Note:*** *At the time of writing, I was using verison 4.4.0. The version can be specified using a command like this:*```pip3 install opencv-contrib-python == 4.4.0```. *To check the version number, run ```python3``` in the terminal followed by ```import cv2``` and ```print(cv2.__version__)```

## Project Setup
The project is set up with the following folder architecture
```
|---pi_surveillance.py
|---conf.json
|---pyimageserarch
    |---_init_.py
    |---tempimage.py
```
The main code for the project will be inside of the pi_surveillance.py file. The conf.json will be used to define some variables instead of hard-coding them into the pi_surveillance.py file or using a command line parser. Pyimageserarch is a folder that will house the ```tempimage``` class which will store images before they are send to dropbox (coming soon). 

To start off we import our dependencies. 
```
from pyimagesearch.tempimage import TempImage
from picamera.array import PiRGBArray
from picamera import PiCamera
from twilio.rest import Client
import os
import argparse
import warnings
import datetime
import imutils
import json
import time
import cv2
```
We then set the conf.json as our configuration file using the --conf file (required), read the conf.json file, and define the appropriate variables. 
The RPi Camera is then setup and given a second to boot. Additionally, the current time is taken and the motion counter is set to 0
