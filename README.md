# CoppeliaSim Installation

To install CoppeliaSim, please follow the instructions below:

1.  Download a version of EDUCATIONAL CoppeliaSim from [here](http://www.coppeliarobotics.com/downloads.html), according to your system specification.
2.  Unpack the compressed file to somewhere in your system and rename it to coppeliaSimulator
3.  Go to the coppeliaSimulator folder and run the command:

```BASH
cd ~/coppeliaSimulator
sudo ./coppeliaSim.sh
```
The above command will start the Coppelia simulation software. 
It should look like this.
![enter image description here](https://i.imgur.com/OkiwXXi.png?1)

## Opening a Scene
To open a Scene file, click on the **File->Open Scene** . This will open a file explorer, go to the folder where you unpack the CoppeliaSim.
```
coppeliaSimulator/tutorials/BubbleRob/bubbleRob.ttt
```
- make sure to disable all other scripts other than main script by going to [**tools --> scripts**]

 - now manually start b0-RemoteAPI Server by goint to the menu bar [**Add-ons --> b0RemoteApiServer**]

Now we ready with the server side to use B0 remote API.
## Enabling B0-remoteAPI on Client Side 

### For Python:
To use the B0-based remote API functionality in your Python script, you will need following:

- create a folder named ***python_client***
- copy _programming/remoteApiBindings/b0Based/python/b0RemoteApi.py_ &   _programming/remoteApiBindings/b0Based/python/b0.py_ to the folder python_client
-   copy libb0.so from the coppeliaSim folder to **/usr/lib/** folder
	```BASH
	cd ~
	mkdir python_client
	cd python_client
	cp ~/coppeliaSimulator/programming/b0RemoteApiBindings/python/b0RemoteApi.py .
	cp ~/coppeliaSimulator/programming/b0RemoteApiBindings/python/b0.py .
	sudo cp ~/coppeliaSimulator/libb0.so /usr/lib/
	```
	

-   install [MessagePack](https://msgpack.org/index.html) for Python: 
 `pip3 install msgpack`

- now create a python file ***main.py***
	`touch main.py`
- and copy the code below in main.py
	```python
	import b0RemoteApi
	import math
	import time

	with b0RemoteApi.RemoteApiClient('b0RemoteApi_pythonClient','b0RemoteApiAddOn') as client:
	    client.doNextStep=True

	    def simulationStepStarted(msg):
	        simTime=msg[1][b'simulationTime'];

	    def simulationStepDone(msg):
	        simTime=msg[1][b'simulationTime'];
	        client.doNextStep=True


	    left_motor=client.simxGetObjectHandle('bubbleRob_leftMotor',client.simxServiceCall())

	    client.simxGetSimulationStepStarted(client.simxDefaultSubscriber(simulationStepStarted));
	    client.simxGetSimulationStepDone(client.simxDefaultSubscriber(simulationStepDone));
	    client.simxSynchronous(True)
	    client.simxStartSimulation(client.simxDefaultPublisher())

	    startTime=time.time()
	    while time.time()<startTime+15:
	        if client.doNextStep:
	            client.doNextStep=False
	            client.simxSetJointTargetVelocity(left_motor[1],1.0,client.simxDefaultPublisher())
	            client.simxSynchronousTrigger()
	        client.simxSpinOnce()

	    client.simxStopSimulation(client.simxDefaultPublisher())
	```
-  run the simulator using 
	```BASH
	cd ~
	./coppeliaSimulator/coppeliaSim.sh
	```
- run the python script that we have just created ***python_client.py***
	```BASH
	cd ~/python_client
	python3 main.py
	```
	<br>
 

> if you get a error message as: <br> **AttributeError: 'str' object has no attribute 'decode'**<br>
> then do the following to resolve the error:
> - open b0RemoteApi.py
> - on line number 62 & 73
> - replace  **`msg=msgpack.unpackb(msg)  --> msg=msgpack.unpackb(msg,raw=True)`**
> - replace **`rep = msgpack.unpackb(self._serviceClient.call(packedData)) --> 
rep = msgpack.unpackb(self._serviceClient.call(packedData),raw=True)`**
 


<br/>

### For C/C++

To use the B0-based remote API functionality in your Python script, you will need following:

- create a new console application project in QT Creator in your home directory  **/home/< your username >** with project name "**cpp_client**"

To install **QT Creator** use the following commans

```bash
sudo apt install build-essential
sudo apt install qtcreator
sudo apt install qt5-default
```

![How to Create a new project](https://i.imgur.com/7Ljobvv.gif)
	
- copy following files & directory to **cpp_client** folder using the commands below
	```bash
	cd ~/cpp_client
	cp ~/coppeliaSimulator/programming/b0RemoteApiBindings/cpp/b0RemoteApi.cpp .
	cp ~/coppeliaSimulator/programming/b0RemoteApiBindings/cpp/b0RemoteApi.h .
	cp -r ~/coppeliaSimulator/programming/b0RemoteApiBindings/cpp/msgpack-c/include/ .
	cp -r ~/coppeliaSimulator/programming/bluezero/include/b0/bindings/ .
	sudo cp ~/coppeliaSimulator/libb0.so /usr/lib/
	
	```
- add the following at the last in ***cpp_client.pro***
	```
	INCLUDEPATH +=./include
	INCLUDEPATH +=./bindings/
	HEADERS += b0RemoteApi.h
	SOURCES += b0RemoteApi.cpp
	LIBS += /usr/lib/libb0.so
	```
- copy the following code in ***main.cpp***
	```cpp
	#include "b0RemoteApi.h"
	#ifdef _WIN32
	#include <windows.h>
	#else
	#include <unistd.h>
	#endif

	static float simTime=0.0f;
	static int sensorTrigger=0;
	static long lastTimeReceived=0;
	static b0RemoteApi* cl=nullptr;

	void simulationStepStarted_CB(std::vector<msgpack::object>* msg)
	{
	    std::map<std::string,msgpack::object> data=msg->at(1).as<std::map<std::string,msgpack::object>>();
	    std::map<std::string,msgpack::object>::iterator it=data.find("simulationTime");
	    if (it!=data.end())
	        simTime=it->second.as<float>();
	}

	void proxSensor_CB(std::vector<msgpack::object>* msg)
	{
	    sensorTrigger=b0RemoteApi::readInt(msg,1);
	    lastTimeReceived=cl->simxGetTimeInMs();
	}

	int main()
	{
	    int leftMotorHandle;
	    int rightMotorHandle;
	    int sensorHandle;
	    leftMotorHandle=19;
	    rightMotorHandle=20;
	    sensorHandle=16;
	    b0RemoteApi client("b0RemoteApi_c++Client","b0RemoteApiAddOn");
	    cl=&client;
	    client.simxStartSimulation(client.simxServiceCall());
	    client.simxGetSimulationStepStarted(client.simxDefaultSubscriber(simulationStepStarted_CB));
	    client.simxReadProximitySensor(sensorHandle,client.simxDefaultSubscriber(proxSensor_CB,0));

	    float driveBackStartTime=-99.0f;
	    float motorSpeeds[2];
	    lastTimeReceived=client.simxGetTimeInMs();
	    while (client.simxGetTimeInMs()-lastTimeReceived<2000)
	    {
	        if (simTime-driveBackStartTime<3.0f)
	        { // driving backwards while slightly turning:
	            motorSpeeds[0]=-3.1415f*0.5f;
	            motorSpeeds[1]=-3.1415f*0.25f;
	        }
	        else
	        { // going forward:
	            motorSpeeds[0]=3.1415f;
	            motorSpeeds[1]=3.1415f;
	            if (sensorTrigger>0)
	                driveBackStartTime=simTime; // We detected something, and start the backward mode
	        }
	        client.simxSetJointTargetVelocity(leftMotorHandle,motorSpeeds[0],client.simxDefaultPublisher());
	        client.simxSetJointTargetVelocity(rightMotorHandle,motorSpeeds[1],client.simxDefaultPublisher());
	        client.simxSpinOnce();
	        client.simxSleep(50);

	    }
	    std::cout << "Ended!" << std::endl;
	    client.simxStopSimulation(client.simxServiceCall());
	    return(0);
	}

	```
-  run the simulator using 
	```BASH
	cd ~
	./coppeliaSimulator/coppeliaSim.sh
- and open the [Scene](#Opening-a-Scene)
- And run the ***cpp_client*** project
