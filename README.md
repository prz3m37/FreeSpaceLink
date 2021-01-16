# FREE SPACE LINK CHANNEL CHARACTERIZATION SOFTWARE

**NAME:**     Free Space Link Channel Characterization

**AUTHOR:**   Przemyslaw Zielinski

**DATE:**     December 2020

**FUNCTION:** Synchronization, tty port communication, Arduino sensors libraries, uEye camera driver, RedPitaya communication.

## **Requirements**

</br>

* C++ compiler 
* Python 3.6 +
* Linux OS 
* ClusterSSH
* Arduino IDE
* Fast internet connection :D 


## **Overview**
</br>
The following document describes the **Free Space link Channel Characterization project software**.
User will find here full description of the software packages posted on [GITHUB PAGE](https://github.com/IQOQI-Ursin-group/FreespaceChannelcharacterization). Software is split onto port communication / synchronization, camera driver and Arduino measurement part.


## **Synchronization / port communication software**
</br> 

Purpose of the synchronization software is to run Arduino devices and measure weather ("lab") conditions in parallel in two separated places - in ou case Bisamberg and IQOQI observatory. Further described setup can be replicated N times due to user needs.

In brief we can imagine our setup as N-node system where central part is PC responsible for processing (sending and recieving) signals comming from external nodes - Arduino devices and one reference node - Clock Arduino. In practice everything is based on port communication which in Linux OS case is reduced to reading from and writing to tty files. 

> **NOTE**: We have to remember that any change of tty file in Linux OS requires root permissions. To grant them open Terminal and type :

    sudo chmod a+rw /dev/tty[PORT NAME] 


Make sure that you opened correct ports. This can be easily checked in "Tools" tab in the Arduino IDE. 

Now we can take a closer look to the whole process of communication:

1. GPS (or other) clock sends PPS signal to the Clock Arduino via cable.
2. Clock Arduino saves timestamp (each second) to the tty file which is immediately readed by PC. 
3. Timestamp is sent to the Arduino's with weather sensors triggering the measurement.
4. Measurement results and timestamps are readed from allArduino's tty ports / files and saved in txt file. 


> **NOTE:** Clock Arduino software is placed in MasterSlave directory; procedure the same as in *Arduino sensors libraries / software* section.

Responsible for port communication software is placed in ReadArduino directory. We can distinguish here two main parts: configuration and reading ports placed respectively in *PortCommunication.cpp* and *ReadArduino.cpp* files. While the destiny / role of the first one is rather obvious the second one require short description. 

```saveResultsToFile()``` function can be considered as a central element. In this module PC reads / writes to tty ports. Eveything takes place in infinite while loop where ports are constantly checked if some information are available. 
> **NOTE**: Speed of loop depends on CPU version !
```cpp
void ReadArduino::saveResultsToFile()
{
    signal(SIGINT, staticSignalCallbackHandler);
    std::string timestamp;
    tempErrorNBytes = 0;
    sendStartTrigger();

    int linesReaded = 0;
    while (true) 
    {    
        nBytes = read(*(this->fileUSBs), &clockBuf , sizeof(clockBuf));
        timestamp = std::string(clockBuf);
		std::cout << timestamp << " [PC time]: "
		 << utils.getTime() << " [nBytes]: " << nBytes;
  		if(nBytes >= messageBytes)
			handleArduinoMessage(timestamp);
		if(nBytes < messageBytes && tempErrorNBytes + nBytes < messageBytes)
		{
			timestamp = "\n" + std::to_string(std::time(0));
			handleArduinoMessage(timestamp);
			tempErrorNBytes = nBytes;
		}
		else
			tempErrorNBytes = 0;
		memset(clockBuf, 0, 22);
    }
}
```

If timestamp from Clock Arduino is saved then all measuring Arduino's are triggered one by one in for loop (```handleArduinoMessage()``` function). To assure correct operations exceptions were added. They are handling cases of splitting messages into two parts comming from Clock Arduino. That's why user has to keep length of message (Clock Arduino) or change ```messageBytes``` filed in the header file. 



### **ReadArduino usage instruction**
</br>

1. Compile program (use MakeFile)

       make 

2. Configure params.txt file providing following informations:
      * number of ports 
      * path to results 
      * paths of Clock and measuring Arduino's 

3. Type in terminal

       /path/to/file/read_arduino

4. To stop program push ```Crtl + C```. Program will save latest results. All actions are logged in logFile_Arduino.txt file. 


> **NOTE**: To erase compiled progrma type: make clean.

As mentioned before we can copy such setup as many times as we want and we can easily synchronize them. First step is to provide GPS clocks which are ticking in the same momnent in every setup. For second step which is simultaneous running C++ code we will use ```ClusterSSH``` tool. This software is designed for managing many separate terminals at once. After installation simply run application by:

    cssh [USER1]@[IP ADDR] [USER2]@[IP ADDR] ...

This will open required number of terminals by which we can comtrol all PC at once. To communicate only one PC just click on chosen terminal. 

> **NOTE**: CluserSSH tool will work without additional permissions only when all PC are connected via the same network. In other case you need to grant acces to connect via ssh. 




## **Arduino sensors libraries / software**
</br>

In our case we are using three types of weather sensors:
* [Pressure - temperature sensor](https://www.mikroe.com/pressure-click)
* [Humidity - temperature sensor](https://www.mikroe.com/htu21d-click)
* [Light sensor](https://www.mikroe.com/light-click)

which are connected with [Arduino UNO](https://store.arduino.cc/arduino-uno-rev3) /  [Arduino MEGA](https://store.arduino.cc/arduino-mega-2560-rev3) devices via proper [shields](https://www.mikroe.com/arduino-uno-click-shield). To be able to read from each device we need two things: sensor driver and library which is responisble for communication between Arduino and sensor. First one we can easily find (and change due to your needs) in the following directories: 
* lightHumidityDriver
* pressureDriver

Required libraries included in Arduino code must be added to the Arduino IDE. The easiest way is to add them in ZIP format (all folders with libraries are in git repostory or you can find them in the web).



 > **NOTE:**  If you want to change Arduino code ([FILE NAME].ino file) and you want to keep triggered measurement you have to apply following scheme: 

```cpp
  ...
  if (Serial.available() > 0)
  {
    """ Your code here """

    processSyncMessage();
  }
  else
    ticks = 1;
  }
  ...
```
## **Camera software**
</br>

Software which saves image from uEye camera in npz matrices format. Python progrma can be found in camera_driver directory. 

1. Download and Install uEye [software](https://de.ids-imaging.com/download-details/AB02536.html), check [readme](https://de.ids-imaging.com/files/downloads/ids-software-suite/readme/readme-ids-software-suite-linux-49400_EN.html) before 

2. Install required libraries by running (check camera_driver dir)
     
       pip3 install -r /path/to/file/requirements.txt

2. Open USB port for camera (also in [readme](https://de.ids-imaging.com/files/downloads/ids-software-suite/readme/readme-ids-software-suite-linux-49400_EN.html)):

       sudo systemctl start ueyeusbdrc

4. To run camera and savebit maps:  
     
       python3 ueye_cam.py

> **NOTE**: Before runnig python script close all uEye application. Accessing the same port by two independent programmes may cause problems.


## **RedPitaya communication software**
</br>

This software is provided by software developers. To get more details please refer to [RedPitaya Streaming App](https://redpitaya.readthedocs.io/en/latest/appsFeatures/apps-featured/streaming/appStreaming.html). 

> **NOTE**: Wav file ca be easily opened by python library

```python
from scipy.io import wavfile
path = "path_to_file"
samplerate, data = wavfile.read(path)
```
