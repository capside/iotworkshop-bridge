# Azure IoT Workshop

This little workshop will show you how to use the [Azure IoT Edge](https://azure.microsoft.com/services/iot-edge/) technology, ensuring a safe connection between your devices and the cloud.

Basically, you will create an [IoT Hub](https://azure.microsoft.com/services/iot-hub) to ingest iot telemetry and then you will deploy the [IoT Edge Agent](https://github.com/Azure/iotedge) on a device to run an [IoT Edge module](https://docs.microsoft.com/azure/marketplace/iot-edge-module). The module provides a simple bridge between http-based devices and the IoTHub. The source code of the module is provided in this repository.

I think the tutorial is a nice and quick way to understand how Azure Iot Edge works. Feel free to write me at `email` at `javier-moreno.com` if you need help to set it up. And, of course, pull request are welcome :)

Enjoy!

## Architecture

![Architecture diagram](architecture.png)

## Environment preparation

* Login into the [Azure portal](https://portal.azure.com)

* Open the cloudshell

* Choose a nice prefix to identify your resources

```bash
PREFIX=workshop && echo $PREFIX
RESOURCE_GROUP_NAME=$PREFIX-rg && echo $RESOURCE_GROUP_NAME
IOT_HUB_NAME=$PREFIX-iothub
```

* Set your very own unique name for your *edge device*

```bash
EDGE_DEVICE_NAME=$USER-edge-$RANDOM

echo Your edge device name will $EDGE_DEVICE_NAME.
```

* Save those variables for easier retrieve

```bash
cat << EOF > variables
export PREFIX=$PREFIX
export RESOURCE_GROUP_NAME=$RESOURCE_GROUP_NAME
export IOT_HUB_NAME=$IOT_HUB_NAME
export EDGE_DEVICE_NAME=$EDGE_DEVICE_NAME
EOF

cat variables
```

* If you need to reload the variables, use

```bash
source variables
```

* Install the *iot* extension to the *az* command so you can manipulate the *IoTHub*

```bash
az extension add --name azure-cli-iot-ext
```

* If you are not using *cloudshell* you need to login in Azure by opening the [device login page](https://aka.ms/devicelogin), typing  your credentials and the code provided by:

```bash
az login
```

## IotHub provisioning

* Create your resource group (if it does not exists) 

```bash
az group create --name $RESOURCE_GROUP_NAME --location westeurope
```

* Create the iot hub

```bash
az iot hub create \
  --resource-group $RESOURCE_GROUP_NAME \
  --name $IOT_HUB_NAME --sku S1

az iot hub list \
  --query "[].name" \
  --output tsv
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

DEVICE_CONN_STRING=$(az iot hub device-identity show-connection-string \
  --device-id $EDGE_DEVICE_NAME \
  --hub-name $IOT_HUB_NAME \
  --query connectionString --output tsv) \
&& echo $DEVICE_CONN_STRING > $EDGE_DEVICE_NAME.txt \
&& cat $EDGE_DEVICE_NAME.txt
```

* We are going to create an Edge gateway using a vm instead of a baremetal server like a Raspberry PI, in order to make this workshop more straightforward. To provision the gateway software, write the init script to a file:

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

* The last step is going to take some time so, why don't you take a look at the [cloud-init script](initvm.sh) in the meanwhile? **You can use the exactly same script to provision the IoT Edge Runtime in your Raspberry PI  :)**

## IotEdge runtime configuration

* Once the command `az vm create` has returned, wait and additional minute or so in order to provide enough time to finish the tool provisioning. Go and get some coffee. Read the news. After that, connect to your new device vm with 

```
IP=$(az vm show \
  --show-details \
  --resource-group $RESOURCE_GROUP_NAME \
  --name vm$EDGE_DEVICE_NAME \
  --query publicIps \
  --output tsv) \
&& echo $IP

scp *.txt iot@$IP:/home/iot/
ssh iot@$IP
```

* You will find a file text with the *Edge Runtime* configuration on it:

```bash
cat *.txt
```

* Check if the tools are ready (if you need `sudo` to run `docker ps`, execute `sudo usermod -aG docker $(whoami)` and then logout and login again)

```
tail /var/log/cloud-init.log
```

* Docker agent is installed, but only root can access it for now:

```bash
docker ps
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
DEVICE_CONN_STRING=$(cat *.txt) && echo $DEVICE_CONN_STRING
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

* Doublecheck you are again using the terminal of your workstation (not the device vm, **you should not see *iot* as user in the prompt**)

* Check it again. Really

* Take a few minutes to check the [workshop/deployment.json](https://github.com/capside/iotworkshop-bridge/blob/master/workshop/deployment.json) as it contains the configuration for the *edge device*

* Now deploy the modules on the device (actually, tell the *iothub* to ask the device to update its status)

```bash
wget https://raw.githubusercontent.com/ciberado/iotworkshop-bridge/master/workshop/deployment.json
az iot edge set-modules \
  --hub-name $IOT_HUB_NAME \
  --device-id $EDGE_DEVICE_NAME \
  --content deployment.json
```

* Open the port 3000 to access the *http-to-iothub* application created by the deployment

```bash
az vm open-port \
  --resource-group $RESOURCE_GROUP_NAME \
  --name vm$EDGE_DEVICE_NAME \
  --port 3000  
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

* Load the env variables

```bash
source variables
```

* Run one or more simulated sensors

```
IP=$(az vm show \
  --show-details \
  --resource-group $RESOURCE_GROUP_NAME \
  --name vm$EDGE_DEVICE_NAME \
  --query publicIps \
  --output tsv) \
&& echo $IP

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
