# RPi-Twilio-Notification-Enabled-Security-System
This repository contains the code to set up a Raspberry Pi Security System with Twilio text notifications using a Raspberry Pi, RPI Camera Module, OpenCV, and the Twilio Python API

***Note:*** *This project was done with the help of an existing project by Adrian Rosebrock, which can be found* [here](https://www.pyimagesearch.com/2015/06/01/home-surveillance-and-motion-detection-with-the-raspberry-pi-python-and-opencv/)

## Background
This project utilizes the image processing capabilities of the widely used [OpenCV](https://docs.opencv.org/master/d0/de3/tutorial_py_intro.html) Python library. It also takes advantage of the Twilio's SMS Python API, which allows users to create responsive and useful notifications in their applications using SMS messaging and Python. 

## Getting Started
Before we get started, make sure you have all the necessary hardware:
- Raspberry Pi (I am using a [4B with 2GB RAM](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/), but would recommend a model 4 or 3 with more than 2GB RAM just to make sure you dont overload the system, a Zero should work too)
- Raspberry Pi Camera board (I am using a [Camera Module V2](https://www.raspberrypi.org/products/camera-module-v2/) )
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

***Note:*** *At the time of writing, I was using verison 4.4.0. The version can be specified using a command like this:*```pip3 install opencv-contrib-python == 4.4.0```. *To check the version number, run* ```python3``` *in the terminal followed by* ```import cv2``` *and* ```print(cv2.__version__)```

We will also need to set up a Twilio account to receive the following:
- ACCOUNT SID
- Auth Token
- Twilio Phone Number (for sending the messages)

You can set up your account [here](https://www.twilio.com/docs/sms/quickstart/python)

## Project Setup
The project is set up with the following folder architecture
```
|---pi_surveillance.py
|---conf.json
|---pyimageserarch
    |---_init_.py
    |---tempimage.py
```

The main code for the project will be inside of the ```pi_surveillance.py``` file. The ```conf.json``` will be used to define some variables instead of hard-coding them into the ```pi_surveillance.py``` file or using a command line parser. Pyimageserarch is a folder that will house the ```tempimage``` class which will store images before they are send to dropbox (coming soon). 

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

We then set the ```conf.json``` as our configuration file using the --conf file (required), read the ```conf.json``` file, and define the appropriate variables. 
The RPi Camera is then setup and given a second to boot. Additionally, the current time is taken and the motion counter is set to 0.
```
camera = PiCamera()
camera.resolution = tuple(conf["resolution"])
camera.framerate = conf["fps"]
rawCapture = PiRGBArray(camera, size=tuple(conf["resolution"]))

#camera warmup and initialization of the average frame, last uploaded timestamp, and frame motion counter
print("Warming up...")
time.sleep(conf["camera_warmup_time"])
avg = None
lastUploaded = datetime.datetime.now()
motionCounter = 0
```

The last part of the setup is printing the debug message and sending a message via the Twilio API to the Recipient showing the system is operational.
```
print("Up and Running")
message = client.messages \
                .create(
                     body="<INSERT YOUR CUSTOM INITIALIZING MESSAGE HERE>",
                     from_="<INSERT YOUR TWILIO PHONE NUMBER HERE>",
                     to="<INSERT YOUR/THE PERSON TO BE NOTIFIED PHONE NUMBER HERE>"
                 )
print(message.sid)
```

## Image processing
Now, we can begin processing the video feed. 
The main for loop starts by capturing each frame of the camera, converting it into an array, getting the current time, and setting the variable displayed on the screen to its default "No Intruders" value. Then the frame is resized, converted to grayscale, and blurred slightly.
```
for f in camera.capture_continuous(rawCapture,
                                   format="bgr",
                                   use_video_port=True):
    frame = f.array
    timestamp = datetime.datetime.now()
    text = "No Intruders"

    #resizing, converting to grayscale, and blurring
    frame = imutils.resize(frame, width=500)
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    gray = cv2.GaussianBlur(gray, (21, 21), 0)
 ```
 
After this, backgorund depiction of the room is collected to use as a reference frame for what "normal activity" would look like. Then a difference betweeen the current frame and reference frame is calculated, and the image is processed and swept for countours present.
```
if avg is None:
        print("starting background depiction...")
        avg = gray.copy().astype("float")
        rawCapture.truncate(0)
        print("background depiction gathered")
        continue
    
    
    #get weighted avg between current and previous frame and compute difference between current frame and running avg
    cv2.accumulateWeighted(gray, avg, .5)
    frameDelta = cv2.absdiff(gray, cv2.convertScaleAbs(avg))
    
    #threshold delta image, dilate threshold image, and find contours on threshold
    threshold = cv2.threshold(frameDelta,conf["delta_thresh"], 255,cv2.THRESH_BINARY)[1]
    threshold = cv2.dilate(threshold, None, iterations=2)
    cnts = cv2.findContours(threshold.copy(), cv2.RETR_EXTERNAL,
                            cv2.CHAIN_APPROX_SIMPLE)
    cnts = imutils.grab_contours(cnts)
```

Next, the contrours are assesed by their area. If they occupy more space than the minimum area defined in the ```conf.json``` "min_area" argument, than they are considered as a foreign object in the frame and are assigned a bouding box. Additionally, the varaible holding the on-screen text is changed from **"No Intruders"** to **"Occupied"**
```
for c in cnts:
        
        # if the contour is too small, ignore it
        if cv2.contourArea(c) < conf["min_area"]:
            continue        
        (x,y,w,h) = cv2.boundingRect(c)
        cv2.rectangle(frame, (x, y), (x+w, y+h), (255, 255, 255), 2)
        text = "Occupied"
 ```
 
Once the text and timestamp are updated, the program checks to see if the number of frames the object has occupied is greater than the minimum amount of frames the user defines in the ```conf.json``` under "min_motion_frames" using the ```motionCounter``` variable. If so, it executes the Dropbox and Twilio protocols to upload the images to Dropbox (currently not tested) and send a notification to the Recipient via the Twilio API using the account information defined in the ```conf.json```
```if text == "Occupied":
        
        #check to see if enough time has passed btw uploads
        if (timestamp - lastUploaded).seconds >= conf["min_upload_seconds"]:
            motionCounter += 1
            
            #check to see if number of frames with consistent motion is high enough
            if motionCounter >= conf["min_motion_frames"]:
                
                #check to see if dropbox should be used
                #if so do the following
                if conf["use_dropbox"]:
                    t = TempImage()
                    cv2.imwrite(t.path,frame)
                    
                    #upload image to dropbox and clear for new image upload
                    print("[UPLOAD] {}".format(ts))
                    path = "/{base_path}/{timestamp}.jpg".format(base_path=conf["dropbox_base_path"], timestamp=ts)
                    client.files_upload(open(t.path, "rb").read(), path)
                    t.cleanup()
                    
                elif conf["use_twilio"]:
                    message = client.messages \
                .create(
                     body="<INSERT CUSTOM ALERT MESSAGE HERE>",
                     from_="<INSERT TWILIO PHONE NUMBER HERE>",
                     to="<INSERT RECIPIENT'S PHONE NUMBER HERE>"
                 )
                    print(message.sid)
                lastUploaded = timestamp
                motionCounter = 0
    else:
        motionCounter = 0
 ```
 
Finally, if the project is set to display the live feed, the frame is displayed on screen and then cleared in preparation for the next frame.
```
if conf["show_video"]:
        #display feed
        cv2.imshow("Live Feed", frame)
        key = cv2.waitKey(1) & 0xFF
        
        if key == ord("q"):
            break
        
    #clear stream to prep for next frame
    rawCapture.truncate(0)
```

## Current State of the Project
The project currently works to send SMS notifications to a user's mobile device if motion is detected within the camera's field of view. Dropbox functionality can also be enabled with the required account information, however it has not been tested. 

Here is what the end product looks like (excuse the blurryness, I was using the "scrot" command off of a low-resolution TeamViewer session). The Raspberry PI camera is shown in the upper left corner of the picture, and another live feed taken from an XBOX Kinect V1 sensor is seen capturing a different vantage of the room in the horizontal window. 


![demo1](/demo_images/demo1.png)
This image shows a bounding box around my subject (neighbor) intruding in my living room (watering my plants). It also displays the Room Status in the top left corner. 


![demo2](/demo_images/demo4.png)

This image shows the updated room status when the intruder left my PiCamera's Field of View (as they went to the bathroom). The status of the room is updated in the top left corner of the window from "Occupied" to "No Intruders".

**Demo 7** in the *demo_images* folder shows a screenhot of the Twilio Notifications sent to my phone as the status of the room changed from "No Intruders" to "Occupied". Messages are sent as long as motion is detected in the room for the predefined duration.


## TO DO
 * implementing functionality to control dropbox uploading based off user replies from SMS messaging
 * adding streams from XBOX Kinect V1 Depth Sensor for accurate motion detection in low light conditions


### Thank you for having interest in my project. Enjoy and stay tuned for more updates.
