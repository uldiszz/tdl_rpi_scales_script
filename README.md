# SIA TestDevLab coffee drawer scales

scale.py
```
#/home/pi/.local/lib/python3.5/site-packages
from hx711 import HX711
import sys
import RPi.GPIO as GPIO
import math
import statistics
import os
import datetime
import http.client
import array
from time import sleep
import logging
 
repeatMeasurements = True
lowMea = []
goodMea = []
dateSaveFile = "/home/pi/Desktop/scalescript_save.txt"
 
def saveLastMessageSentTimeToFile():
    with open(dateSaveFile, mode='w') as file:
        file.write("%s" % (datetime.datetime.now()))
       
def lastMessageSentDateFromFile():
    dateString = ""
    with open(dateSaveFile, "r") as file:
        dateString = file.read()
    if dateString == "":
        logging.warning("Could not read last send date from file: " + dateSaveFile)
    else:
        lastSentDatetime = datetime.datetime.strptime(dateString, '%Y-%m-%d %H:%M:%S.%f')
        dateString = lastSentDatetime.date()
    return dateString
   
 
def sendToDiscord(message):
    if lastMessageSentDateFromFile() == datetime.datetime.now().date():
        logging.info("Will not send message to discord because last message was sent today")
        return ""
    url = <your-discord-webhook-url>
    formdata = "------:::BOUNDARY:::\r\nContent-Disposition: form-data; name=\"content\"\r\n\r\n" + message + "\r\n------:::BOUNDARY:::--"
    connection = http.client.HTTPSConnection("discordapp.com")
    connection.request(
        "POST", url, formdata, {
            'Content-Type': "multipart/form-data; boundary=----:::BOUNDARY:::",
            'cache-control': "no-cache"
        }
    )
    response = connection.getresponse().read()
    saveLastMessageSentTimeToFile()
    return response.decode("utf-8")
 
 
def processMeasurement(mea):
    emptyWeight = 63
    onePackWeight = 10
    avg = mea
    if len(lowMea) > 0:
        avg = sum(lowMea) / len(lowMea)
    packsLeft = round(((avg - emptyWeight) / onePackWeight), 2)
    weight = round(((avg - emptyWeight) * 100), 2)
    repeatMeasurements = False
    if packsLeft <= 2.1:
        lowMea.append(mea)
        repeatMeasurements = True
    else:
        goodMea.append(mea)
 
    sleep(10)
   
    if len(lowMea) >= 20:
        repeatMeasurements = False
        logging.info(
            "Sending to Discord. Coffee in drawer: " + str(packsLeft) + " packs (" + str(weight) + "g). " + "Response: \n" +
            sendToDiscord("Coffee drawer has " + str(packsLeft) + " packs (" + str(weight) + "g)!")
        )
        raise SystemExit(0)
    elif len(goodMea) >= 20:
        logging.info("We have enough coffee in the drawer, exiting")
        raise SystemExit(0)
 
 
try:
    logging.basicConfig(
        filename="/home/pi/Desktop/scale_logger.log",
        filemode="w",
        format="%(name)s - %(levelname)s - %(message)s",
        level=logging.INFO
        )
    logging.info(datetime.datetime.now())
    logging.info("Connecting to HX711...")
    hx711 = HX711(
            dout_pin=23,
            pd_sck_pin=24,
            channel='A',
            gain=64
        )
    # Reset before start
    if hx711.reset():
        logging.info("Successful HX711 reset")
    else:
        logging.info("HX711 reset failed")
        raise ValueError()
    logging.info("Connected to HX711")
 
    while(repeatMeasurements):
        measures = hx711.get_raw_data()
        mea = int(sorted(measures)[1]/1000)
        logging.info("\n" + str(mea) + "\t\t" + str(measures) + "\t\t\t" + str(datetime.datetime.now()))
        processMeasurement(mea)
 
except Exception as e:
    print(e)
    logging.error("Exception occured", exc_info=True)
 
finally:
    logging.info("Cleaning up")
    GPIO.cleanup()
```

### CRON
Install gnome-schedule for easier crontab management
</br>
```sudo apt-get install gnome-schedule```
</br>
Now to edit `crontab -e`
</br>
To view `crontab -l`
</br>
</br>

If you want to run script after each reboot then add to your crontab
</br>
```@reboot python /usr/bin/python3 /home/pi/Desktop/scale.py```
</br>
</br>

To run At every 10th minute from 9:00 through 17:00 (Mon-Fri):
```
0 9-17 * * 1-5 /usr/bin/python3 /home/pi/Desktop/scale.py
10 9-16 * * 1-5 /usr/bin/python3 /home/pi/Desktop/scale.py
20 9-16 * * 1-5 /usr/bin/python3 /home/pi/Desktop/scale.py
30 9-16 * * 1-5 /usr/bin/python3 /home/pi/Desktop/scale.py
40 9-16 * * 1-5 /usr/bin/python3 /home/pi/Desktop/scale.py
50 9-16 * * 1-5 /usr/bin/python3 /home/pi/Desktop/scale.py
```
