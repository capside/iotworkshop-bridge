# Azure IoT Workshop

This is the **short** version of the workshop, simplified to be delivered in an hour. You can read the [full version](README.md) in the main `readme` of the repository.

## Architecture

![Architecture diagram](../architecture.png)

## Environment preparation

* Install the *iot* extension to the *az* command so you can manipulate the *IoTHub*

```bash
az extension add --name azure-cli-iot-ext
```

* If you are not using *cloudshell* you need to login in Azure by opening the [device login page](https://aka.ms/devicelogin), typing  your credentials and the code provided by:

```bash
az login
```

* Set a few variables to make the commands more readable

```bash
RESOURCE_GROUP_NAME=workshop-rg
IOT_HUB_NAME=workshop-iothub
EDGE_DEVICE_NAME=$USER-edge 

echo Your edge device name will be $EDGE_DEVICE_NAME
```

## Checking the IoTHub

* Check the existence of the IoTHub

```bash
az iot hub list \
  --query "[?name=='workshop-iothub']" \
  --output table
```

## Device registration and creation

* Register a new shinny device (actually, right now, it's just a registered configuration in the *iot hub*, not a real device)

```bash
az iot hub device-identity create \
  --hub-name $IOT_HUB_NAME \
  --device-id $EDGE_DEVICE_NAME \
  --edge-enabled

az iot hub device-identity list \
  --hub-name $IOT_HUB_NAME \
  --query '[].{id: deviceId, state: connectionState}'
```

* Save the connection string with the identity of the edge device in a file

```bash
DEVICE_CONN_STRING=$(az iot hub device-identity show-connection-string \
  --device-id $EDGE_DEVICE_NAME \
  --hub-name $IOT_HUB_NAME \
  --query connectionString --output tsv) \
&& echo $DEVICE_CONN_STRING > $EDGE_DEVICE_NAME.txt \
&& cat $EDGE_DEVICE_NAME.txt
```

## IotEdge runtime configuration

* You will need a VM (or a raspberry) with `docker`, `python` and [iotedgectl](https://pypi.org/project/azure-iot-edge-runtime-ctl/) to run the *Edge Runtime*. With a bit of luck, we have already created it for you. If this is not the case, look the the [extra section](#how-to-provision-your-edge-device) at the end of this document.


* Get the IP of your Edge Device with

```
IP=$(az vm show \
  --show-details \
  --resource-group $RESOURCE_GROUP_NAME \
  --name vm$EDGE_DEVICE_NAME \
  --query publicIps \
  --output tsv) \
&& echo Your Edge device has the IP $IP
```

* Copy the connection string of the edge device into the vm:

```bash
scp $EDGE_DEVICE_NAME.txt iot@$IP:/home/iot/device.txt
```

* Jump inside your edge device!

```bash
ssh iot@$IP
```

* You will find a file text with the *Edge Runtime* configuration on it:

```bash
cat device.txt
```

* Provide access to docker for non-root user in a very hacky way

```bash
sudo usermod -aG docker $(whoami)
oldg=$(id -g) && newgrp docker && newgrp $oldg
docker ps
```

* Check we have de `iotedgectl` cli

```bash
iotedgectl --version
```

* Configure the *edge runtime* on the device
 
```bash
DEVICE_CONN_STRING=$(cat device.txt) && echo $DEVICE_CONN_STRING
sudo iotedgectl setup --connection-string $DEVICE_CONN_STRING --nopass
```

* Start the device! (and see how the `edgeAgent` is already being deployed)

```bash
sudo iotedgectl start
docker ps
docker logs edgeAgent
exit
```

## Deploying modules to the device

* Doublecheck you are again using the terminal of your workstation (not the device vm, **you should NOT see *iot* as user in the prompt**)

* Check it again. Really

* Take a few seconds to check the [workshop/deployment.json](workshop/deployment.json) as it contains the configuration for the *edge device*

* Now deploy the modules on the device (actually, tell the *iothub* to ask the device to update its status)

```bash
wget https://raw.githubusercontent.com/ciberado/iotworkshop-bridge/master/workshop/deployment.json
az iot edge set-modules \
  --hub-name $IOT_HUB_NAME \
  --device-id $EDGE_DEVICE_NAME \
  --content deployment.json
```

* Check you can access the bridge (it will return a `404`, but that is fine)

```bash
curl -X HEAD -I http://$IP:3000
```

## Watching IoTHub events

* Start monitoring IotHub messages:

```bash
IOTHUB_CONN_STRING=$(az iot hub show-connection-string \
  --name $IOT_HUB_NAME \
  --query connectionString \
  --output tsv) \
&& echo $IOTHUB_CONN_STRING

az iot hub monitor-events --login $IOTHUB_CONN_STRING -y
```

## Generating events

* If you are using [tmux](https://github.com/tmux/tmux/wiki), press `ctrl+b` `"` to create a new panel. Otherway, open a new terminal session (to keep running the event monitoring in the previous one)


* Set the variables if needed

```bash
RESOURCE_GROUP_NAME=workshop-rg
IOT_HUB_NAME=workshop-iothub
EDGE_DEVICE_NAME=$USER-edge 

IP=$(az vm show \
  --show-details \
  --resource-group $RESOURCE_GROUP_NAME \
  --name vm$EDGE_DEVICE_NAME \
  --query publicIps \
  --output tsv) \
&& echo Your Edge device has the IP $IP
```

* Create a program that emulates the sensors that provide information to the edge location device

```
cat << 'EOF' > sensor-simulator
#!/bin/bash

if [ "$#" -ne 2 ]; then
  echo "Please, provide the sensor name as parameter, followed by the gateway endpoint."
  echo "Example: sensor-simulator.sh demo-sensor-1 http://32.12.40.8:3000"
  exit
fi

SENSOR_NAME=$1
ENDPOINT=$2

echo "Simulating sensor $SENSOR_NAME."

while :
do
  SENSOR_LECTURE=$(shuf -i 50-200 -n 1)
  curl -X POST $ENDPOINT --header "Content-type: application/json" \
       --data "{\"sensor\": \"$SENSOR_NAME\", \"value\":$SENSOR_LECTURE}"
  echo " Sensor $SENSOR_NAME, lecture $SENSOR_LECTURE."
  sleep 1
done
EOF

chmod +x sensor-simulator
```

* Run one or more simulated sensors

```

./sensor-simulator $EDGE_DEVICE_NAME-sensor-$RANDOM http://$IP:3000 

```

## Cleanup

* Just delete the resource group

```bash
az group delete --name $RESOURCE_GROUP_NAME
```

## Additional links and resources

* IoTHub events can be easily processed in ETLs by using [Stream Analytics Jobs](https://azure.microsoft.com/services/stream-analytics/)
* This tutorial explains [how to visualize IoTHub events with PowerBI](https://docs.microsoft.com/es-es/azure/iot-hub/iot-hub-live-data-visualization-in-power-bi)

## How to provision your Edge Device

* To provision the gateway software, write the init script to a file:

```bash
cat << EOF > initvm.sh
#!/bin/sh

sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -sSL https://get.docker.com | sh
sudo apt install python-pip -y
sudo pip install azure-iot-edge-runtime-ctl
sudo apt-get remove unscd -y
sudo usermod -aG docker iot
EOF
```

* Create the vm that we are going to use as device

```bash
az vm create \
  --name vm$EDGE_DEVICE_NAME \
  --resource-group $RESOURCE_GROUP_NAME \
  --admin-password Iot123456789 \
  --admin-username iot \
  --custom-data initvm.sh \
  --image Canonical:UbuntuServer:16.04-LTS:latest \
  --location westeurope \
  --nsg-rule ssh \
  --public-ip-address-allocation dynamic \
  --vnet-name vnet$EDGE_DEVICE_NAME
```

* Once the command `az vm create` has returned, wait and additional minute or so in order to provide enough time to finish the tool provisioning. Go and get some coffee. Read the news. Relax.

* Later, you will need access to port 3000 of the VM. So lets open it now

```bash
az vm open-port \
  --resource-group $RESOURCE_GROUP_NAME \
  --name vm$EDGE_DEVICE_NAME \
  --port 3000  
```

