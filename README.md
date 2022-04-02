# Incorporate-MQTT-In-NVIDIADeepStreamSDK  
As part of my C200 special project module, I was tasked to find a way to send data out of NVIDIA Deepstream SDK to a database. And thus, I wanted to incorporate MQTT in deepstream-test5-app.c to publish MQTT messages out to Node-Red that will then update the database accordingly.

Our project was to build an object detection system that is able to detect shapes. With the modified test app, it will count the number of a specific shape and send the data to Node-Red using MQTT.

## Prerequisite:
The hardware used was the Jetson Nano Developer Kit. Setup for the jetson nano was done with the [Getting Started with Jetson Nano Developer Kit](https://developer.nvidia.com/embedded/learn/get-started-jetson-nano-devkit#intro) guide.

Next, We incorporated IoT Edge in the jetson nano. IoT Edge modules are containers that run Azure services. The setup was done with the following guide: [Intelligent-Video-Analytics-with-NVIDIA-Jetson-and-Microsoft-Azure](https://github.com/toolboc/Intelligent-Video-Analytics-with-NVIDIA-Jetson-and-Microsoft-Azure). We followed the guide till the end of Module 3. You can click [HERE](https://github.com/Jim-Chiew/C200) to view the file dumps git repository to view our specific changes and modification.

## Installing MQTT Broker and Clients:
At this point, you should have set up iotedge that is running  NVIDIADeepStreamSDK docker container on the jetson. You should have also setup any RTSP/camera stream and any custom models in the deepstream.

### Installing Mosquitto MQTT Broker:
The MQTT protocol provides a lightweight method of carrying out messaging using a publish/subscribe model. Eclipse Mosquitto is an open source (EPL/EDL licensed) message broker that implements the MQTT protocol versions 5.0, 3.1.1 and 3.1.

Installing MQTT Broker with the following command:

Resynchronize the package index fileswith:
```
sudo apt-get update
```

Install mosquitto:
```
sudo apt install mosquitto
```

Install mosquitto-clients:
```
sudo apt install mosquitto-clients
```

Test mosquitto by first subscribing to start listen with the following command:
```
mosquitto_sub -t "test"
```

Publishing a massage with the following command:
```
mosquitto_pub -m "message from mosquitto_pub client" -t "test"
```


### Installing Paho MQTT C Client on the Jetson:
The Paho MQTT C Client is a fully featured MQTT client written in ANSI standard C. 

Installation:
clone git repository:
```
git clone https://github.com/eclipse/paho.mqtt.c.git
```

Change directory:
```
cd paho.mqtt.c/
```

Install supporting libraries:
```
sudo apt-get install libssl-dev
```

Build:
```
make
```

Install:
```
sudo make install
```

You are able to test your paho installation with paho command line utilities for subscribing and publishing mqtt messages.
Assuming mosquitto as been installed, start publishing a massage with the following command:
```
paho_c_pub -t my_topic --connection tcp://localhost:1883
```

In a new terminal, use the following command to subscribe/listen to the publishing:
```
paho_c_sub -t my_topic --connection tcp://localhost:1883
```

From here, start typing and entering the terminal with the paho_c_pub. You should see the massages in the terminal with paho_c_sub. To exit, use CTRL+C.  
Note: C clients has to connect to a broker over a TCP/IP connection.  

At this point, if you are running Deepstream without using iotedge, you can proceed straight to modifying the deepstream-test5-app file [here](https://github.com/Jim-Chiew/Incorporate-MQTT-In-NVIDIADeepStreamSDK#modifying-deepstream-test5-appc-to-publish-mqtt).  
  
  
### Installing Paho MQTT C Client on the NVIDIADeepStreamSDK Docker container:
If you had incorporated Microsoft-Azure following the above Prerequisite, you will need to reinstall paho on the docker container as well as modifying the manifest files to open port 1883 of the docker container to the host machine(Jetson nano).
  
  
#### Modifying deployment file to open port 1883 of NVIDIADeepStreamSDK docker container to the host machine:
In the deployment.template.json, you will need to add the following line of code right after `"IpcMode":"host"` in the json NVIDIADeepStreamSDK -> settings -> HostConfig:
```
,
                   "PortBindings": {
                    "1883/tcp": [
                      {
                        "HostPort": "1883"
                      }
                    ]
                  }
```

Your deployment template should look something like:
```json
{
   "NVIDIADeepStreamSDK":{
      "version":"1.0",
      "type":"docker",
      "status":"running",
      "restartPolicy":"always",
      "settings":{
         "image":"nvcr.io/nvidia/deepstream-l4t:6.0-iot",
         "createOptions":{
            "Entrypoint":[
               "/usr/bin/deepstream-test5-app",
               "-c",
               "DSConfig-CustomVisionAI.txt"
            ],
            "HostConfig":{
               "runtime":"nvidia",
               "NetworkMode":"host",
               "Binds":[
                  "/data/misc/storage:/data/misc/storage",
                  "/tmp/argus_socket:/tmp/argus_socket",
                  "/tmp/.X11-unix/:/tmp/.X11-unix/"
               ],
               "IpcMode":"host",
               "PortBindings": {
                    "8883/tcp": [
                      {
                        "HostPort": "8883"
                      }]
            },
            "NetworkingConfig":{
               "EndpointsConfig":{
                  "host":{
                     
                  }
               }
            },
            "WorkingDir":"/data/misc/storage/Intelligent-Video-Analytics-with-NVIDIA-Jetson-and-Microsoft-Azure/services/DEEPSTREAM/configs"
         }
      },
      "env":{
         "DISPLAY":{
            "value":":0"
         }
      }
   }
}
}
```  

Next, redeploy NVIDIADeepStreamSDK and wait for it to run.

To enter the docker container:
```
docker exec -it NVIDIADeepStreamSDK /bin/bash
```  

Next change to the home directory and repeat the installation of paho:
Note that if the container were to be repulled or remove, you will need to repeat this step again.   

You also should be root, thus sudo is not required.
Cd to home:
```
cd ~
```

Clone git repository:
```
git clone https://github.com/eclipse/paho.mqtt.c.git
```

Change directory:
```
cd paho.mqtt.c/
```

Install supporting libraries:
```
apt-get install libssl-dev
```

Build:
```
make
```

Install:
```
make install
```

You could use the same paho [command utilities](https://github.com/Jim-Chiew/Incorporate-MQTT-In-NVIDIADeepStreamSDK#:~:text=You%20are%20able%20to%20test%20your%20paho%20installation%20with%20paho%20command%20line%20utilities%20for%20subscribing%20and%20publishing%20mqtt%20messages.%20Assuming%20mosquitto%20as%20been%20installed%2C%20start%20publishing%20a%20massage%20with%20the%20following%20command%3A) within the container to test your paho.

## Modifying deepstream-test5-app.c to publish MQTT:
The mqtt publishing code was based on the resource [Paho MQTT C Client Library](https://www.eclipse.org/paho/files/mqttdoc/MQTTClient/html/pubsync.html). I recommend modify and changing the provided [pahoTest.c](https://github.com/Jim-Chiew/Incorporate-MQTT-In-NVIDIADeepStreamSDK/blob/main/pahotest.c) file first before incorporating it in the deepstream SDK.

To compile pahoTest.C file. You need to link the supporting libraries with the `-L<Path>` followed by the option `-lpaho-mqtt3c`:
```
gcc -L/home/c200/paho.mqtt.c/build/output <path to pahoTest.c File> -lpaho-mqtt3c
```

To Run
```
./pahoTest
```

To access the deepstream-test5-app.c go to the following path:

```
cd /opt/nvidia/deepstream/deepstream-6.0/sources/apps/sample_apps/deepstream-test5
```
Extract the file or modify it with your preferred method.

### Adding per object counter in deepstream-test5-app.c
you would first need to define some variables first. In the provided [deepstream-test5-app.c](https://github.com/Jim-Chiew/Incorporate-MQTT-In-NVIDIADeepStreamSDK/blob/main/deepstream_test5_app_main.c), it is done in `line 185`. I have 4 objects that I want to detect and count with a char variable to hold the output message of the mqtt payload:
```c
/* Jim Counter Initialize*/
int Circle;
int Heart;
int Square;
int Triangle;
char mqttOutput[200];
```

Next is to set a reset to reset the counter on a new frame. This is done in `line 615`:
```c
  Circle = 0; //By JIM reset counter
  Heart = 0;
  Square = 0;
  Triangle = 0; 
```

For each object, identify the object and increment the corresponding counter. Done in `line 647`:
```c
      /* Count Object Jim MQTT */
      if (obj_meta->class_id == 0) {
          Circle ++;
      } else if (obj_meta->class_id == 1) {
          Heart ++;
      } else if (obj_meta->class_id == 2) { 
          Square ++;
      } else if (obj_meta->class_id == 3){
          Triangle ++;
      }
```  

The object ID is based on the `labels.txt` file in `/data/misc/storage/Intelligent-Video-Analytics-with-NVIDIA-Jetson-and-Microsoft-Azure/services/CUSTOM_VISION_AI/`

Finally, format the string output and publish the MQTT massage. Done in `line 730`:
```c
  /*Publish to MQTT By JIM*/
    pubmsg.payload;
    pubmsg.qos = QOS;
    pubmsg.retained = 0;

    snprintf(mqttOutput, 200,"Circle %d Heart %d Square %d Triangle %d", Circle, Heart, Square, Triangle);
    pubmsg.payload = mqttOutput;
    pubmsg.payloadlen = strlen(pubmsg.payload);
    MQTTClient_publishMessage(client, TOPIC, &pubmsg, &token);
```

### Incorporating MQTT in deepstream-test5-app.c:
You would first need to include and define some variables first. In the provided [deepstream-test5-app.c](https://github.com/Jim-Chiew/Incorporate-MQTT-In-NVIDIADeepStreamSDK/blob/main/deepstream_test5_app_main.c), it is done in `line 54`:
```c
/* Paho */
#include "MQTTClient.h"

#define ADDRESS     "tcp://localhost:1883"
#define CLIENTID    "deepstreamapp"
#define TOPIC       "deepstream"
#define QOS         0
#define TIMEOUT     10000L
```

Initialize was added in `line 178`:
```c
/* Paho MQTT Initialize*/
MQTTClient client;
MQTTClient_connectOptions conn_opts = MQTTClient_connectOptions_initializer;
MQTTClient_message pubmsg = MQTTClient_message_initializer;
MQTTClient_deliveryToken token;
int rc;
```

Publishing massage is in `line 735`:
```c
  /*Publish to MQTT By JIM*/
    pubmsg.payload;
    pubmsg.qos = QOS;
    pubmsg.retained = 0;

    snprintf(mqttOutput, 200,"Circle %d Heart %d Square %d Triangle %d", Circle, Heart, Square, Triangle);
    pubmsg.payload = mqttOutput;
    pubmsg.payloadlen = strlen(pubmsg.payload);
    MQTTClient_publishMessage(client, TOPIC, &pubmsg, &token);
```

Disconnect and Destroy Client in `line 1680`:
```c
  /** Paho MQTT*/
  g_print("Disconnecting MQTT Client\n");
  if ((rc = MQTTClient_disconnect(client, 10000)) != MQTTCLIENT_SUCCESS)
      printf("Failed to disconnect, return code %d\n", rc);
  MQTTClient_destroy(&client);
  g_print("Destroying MQTT Client\n");
```

### Compiling deepstream-test5-app.c:
To compile deepstream-test5-app.c, you first need to modify the `Makefile` to add the supporting library. You can refer to the [Makefile](https://github.com/Jim-Chiew/Incorporate-MQTT-In-NVIDIADeepStreamSDK/blob/main/Makefile) provided.

Add the following code in a new line after `LIBS:= -L/usr/local/cuda-$(CUDA_VER)/lib64/ -lcudart` or at `line 57`:
```c
LIBS+= -L/home/c200/paho.mqtt.c/build/output -lpaho-mqtt3c #By JIm
```

You would need to Set CUDA_VER in the Makefile as per platform as mention in the `README` file if you have not done so.
```
  $ Set CUDA_VER in the MakeFile as per platform.
      For Jetson, CUDA_VER=10.2
      For x86, CUDA_VER=11.4
```

To Compile:
```
sudo make
```

If you did not use Microsoft Azure, you should be able to run the app and have the app publish MQTT massages.

## Moving modified deepstream-test5-app file to NVIDIADeepStreamSDK docker container:
while `NVIDIADeepStreamSDK` is running, enter the container using:
```
docker exec -it NVIDIADeepStreamSDK /bin/bash
```

Go to the path of the deepstream-test5-app
```
cd /opt/nvidia/deepstream/deepstream-6.0/bin/
```

Remove old deepstream-test5-app
```
rm deepstream-test5-app
```

**In your host(Jetson) terminal**, navigate back to `/opt/nvidia/deepstream/deepstream-6.0/sources/apps/sample_apps/deepstream-test5` and copy the file over to the container with the following command:
```
sudo docker cp deepstream-test5-app NVIDIADeepStreamSDK:/opt/nvidia/deepstream/deepstream-6.0/bin/
```

Finally, restart the container with the following command:
```
sudo docker restart NVIDIADeepStreamSDK 
```

After the restart, your NVIDIADeepStreamSDK should be publishing the MQTT massages.

I have created a simple script file that automate creating and moving of deepstream-test5-app. You would still need to remove the deepstream-test5-app file in the container first. Script: [MakeAndMove.sh](https://github.com/Jim-Chiew/Incorporate-MQTT-In-NVIDIADeepStreamSDK/blob/main/MakeAndMove.sh)

## Installing Node-Red with Docker:
Node-RED is a flow-based development tool for visual programming developed originally by IBM for wiring together hardware devices, APIs and online services as part of the Internet of Things.

Pull the Node-Red docker container with the following command:
```
sudo docker run -it -p 1880:1880 -v node_red_data:/data --name mynodered nodered/node-red
```

Simple commands to note:
Start Node-Red in docker:
```
docker start mynodered
```

Stop Node-Red in docker:
```
docker stop mynodered
```

Access Note-Red in web browser:
```
http://<IP Address off Jetson>:1880
```

From here, you are able to set up a simple listener and function nodes to publish to a database using APIs.

## Reference:  
https://www.vultr.com/docs/how-to-install-mosquitto-mqtt-broker-server-on-ubuntu-16-04/  
https://mosquitto.org/  
https://www.arubacloud.com/tutorial/how-to-install-and-secure-mosquitto-on-ubuntu-20-04.aspx  
http://www.steves-internet-guide.com/install-mosquitto-linux/  
https://www.eclipse.org/paho/index.php?page=clients/c/index.php  
https://nodered.org/docs/getting-started/docker  
https://docs.nvidia.com/metropolis/deepstream/sdk-api/struct__NvDsObjectMeta.html  
