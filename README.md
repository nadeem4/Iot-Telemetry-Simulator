---
page_type: sample
languages:
- csharp
- powershell
- dockerfile
products:
- azure-iot-hub
- azure-event-hubs
- azure-container-instances
- azure-container-registry
name: "Azure IoT Device Telemetry Simulator"
urlFragment: azure-iot-device-telemetry-simulator
description: "The IoT Telemetry Simulator allows you to test Azure IoT Hub, Event Hub or Kafka ingestion at scale."
---

![Master test and push](https://github.com/Azure-Samples/Iot-Telemetry-Simulator/workflows/Master%20test%20and%20push/badge.svg)

# Azure IoT Device Telemetry Simulator

The IoT Telemetry Simulator allows you to test Azure IoT Hub, Event Hub or Kafka ingestion at scale. The implementation is communicating with Azure IoT Hub using multiplexed AMQP connections. An automation library allows you to run it as load test as part of a CI/CD pipeline.

A single AMQP connection can handle approximately 995 devices.

## Quick Start

The quickest way to generate telemetry is using Docker with the following command:

```bash
docker run -it -e "IotHubConnectionString=HostName=your-iothub-name.azure-devices.net;SharedAccessKeyName=device;SharedAccessKey=your-iothub-key" mcr.microsoft.com/oss/azure-samples/azureiot-telemetrysimulator
```

**The simulator expects the devices to already exist in Azure IoT Hub**. If you need help creating simulation devices in an Azure IoT Hub use the included project IotSimulatorDeviceProvisioning or the Docker image:

```bash
docker run -it -e "IotHubConnectionString=HostName=your-iothub-name.azure-devices.net;SharedAccessKeyName=registryReadWrite;SharedAccessKey=your-iothub-key" -e DeviceCount=1000 mcr.microsoft.com/oss/azure-samples/azureiot-simulatordeviceprovisioning
```

## Simulator input parameters

The amount of devices, their names and telemetry generated can be customized using parameter. The list below contains the supported configuration parameters:

|Name|Description|
|-|-|
|IotHubConnectionString|Iot Hub connection string. "Device" our "Iot Hub owner" scopes are good. Example: HostName=your-iothub-name.azure-devices.net;SharedAccessKeyName=device;SharedAccessKey=your-iothub-key|
|EventHubConnectionString|Event Hub connection string. SAS Policy "Send" is required. For EventHub no device registration is required. Example: Endpoint=sb://your-eventhub-namespace.servicebus.windows.net/;SharedAccessKeyName=send;SharedAccessKey=your-send-sas-primary-key;EntityPath=your-eventhub-name.|
|KafkaConnectionProperties|Kafka connection properties as a JSON string. Example: `{"bootstrap.servers=kafka"}`.|
|KafkaTopic|Kafka topic name.|
|DeviceList|comma separated list of device identifiers (default = ""). Use it to generate telemetry for specific devices instead of numeric generated identifiers. If the parameter has a value the following parameters are ignored: DevicePrefix, DeviceIndex and DeviceCount are ignored|
|DevicePrefix|device identifier prefix (default = "sim")|
|DeviceIndex|starting device number (default = 1)|
|DeviceCount|amount of simulated devices (default = 1)|
|MessageCount|amount of messages to send by device (default = 10). Set to zero if you wish to send messages until cancelled|
|Interval|interval between each message in milliseconds (default = 1000)|
|Template|telemetry payload template (see telemetry template)|
|FixPayload|fix telemetry payload in base64 format. Use this setting if the content of the message does not need to change|
|FixPayloadSize|fix telemetry payload size (in bytes). Use this setting if the content of the message does not need to change (will be an array filled with zeros)|
|PayloadDistribution|Allows the generation of payloads based on a distribution<br />Example: "fixSize(10, 12) template(25, default) fix(65, aaaaBBBBBCCC)" generates 10% a fix payload of 10 bytes, 25% a template generated payload and 65% of the time a fix payload from values aaaaBBBBBCCC|
|Header|telemetry header template (see telemetry template)|
|PartitionKey|optional [partition key](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-features#partitions) template for Event Hubs (see telemetry template and [Advanced options](#advanced-options))|
|Variables|telemetry variables (see telemetry template)|
|DuplicateEveryNEvents|if > 0, send duplicates of the given fraction of messages. See [Advanced options](#advanced-options) (default = 0)|
|File|Defines a json file where templates, variables and device based intervals can be defined. File and environment variable configuration can be used in conjunction.|
|Intervals|Allows customizing the intervals between messages per device. If an array is given, its elements are iterated one after another, so the interval can vary for the same device. The array is cycled when there's more values to send than elements in it.|


## Telemetry template

The simulator is able to create user customizable telemetry with dynamic variables (random, counter, time, unique identifier, value range).

To generate a custom telemetry it is required to set the template and, optionally, variables.

The **template** defines how the telemetry looks like, having placeholders for variables.
Variables are declared in the telemetry as `$.VariableName`. The optional **header** defines properties that will be transmitted as message properties.

**Variables** are declared defining how values in the template will be resolved.

### Built-in variables

The following variables are provided out of the box:

|Name|Description|
|-|-|
|DeviceId|Outputs the device identifier|
|Guid|A unique identifier value|
|Time|Outputs the utc time in which the telemetry was generated in ISO 8601 format|
|LocalTime|Outputs the local time in which the telemetry was generated in ISO 8601 format|
|Ticks|Outputs the ticks in which the telemetry was generated|
|Epoch|Outputs the time in which the telemetry was generated in epoch format (seconds)|
|MachineName|Outputs the machine name where the generator is running (pod name if running in Kubernetes)|

### Customizable variables

Customizable variables can be created with the following properties:

|Name|Description|
|-|-|
|name|Name of the property. Defines what will be replaced in the template telemetry $.Name|
|random|Make the value random, limited by min and max|
|step|If the value is not random, will be incremented each time by the value of step|
|randomDouble|Make the value random and double, limited by min and max|
|min|For random (integer or double) values defines it's minimum. Otherwise, will be the starting value|
|max|The maximum value generated|
|values|Defines an array of possible values. Example ["on", "off"]|
|customlengthstring|Creates a random string of n bytes. Provide n as parameter|
|sequence|Create a sequence of values as defined in `values` property, producing one after the other. Values can reference other non-sequence variables|


#### Example 1: Telemetry with temperature between 23 and 25 and a counter starting from 100

Template:

```json
{ "deviceId": "$.DeviceId", "rand_int": $.Temp, "rand_double": $.DoubleValue, "Ticks": $.Ticks, "Counter": $.Counter, "time": "$.Time" }
```

Variables:

```json
[{"name": "Temp", "random": true, "max": 25, "min": 23}, {"name":"Counter", "min":100, "max":102}, {"name": "DoubleValue", "randomDouble":true, "min":0.22, "max":1.25}]
```

Output:

```json
{ "deviceId": "sim000001", "rand_int": 23, "rand_double": 0.207759137669466, "Ticks": 637097550115091350, "Counter": 100, "time": "2019-11-19T10:10:11.5091350Z" }
{ "deviceId": "sim000001", "rand_int": 23, "rand_double": 1.207232427664231, "Ticks": 637097550115952079, "Counter": 101, "time": "2019-11-19T10:10:11.5952079Z" }
{ "deviceId": "sim000001", "rand_int": 24, "rand_double": 0.992871827638167, "Ticks": 637097550116627320, "Counter": 102, "time": "2019-11-19T10:10:11.6627320Z" }
{ "deviceId": "sim000001", "rand_int": 24, "rand_double": 0.779288272162327, "Ticks": 637097550117027320, "Counter": 100, "time": "2019-11-19T10:10:11.7027320Z" }
```

Running with Docker:

```powershell
docker run -it -e "IotHubConnectionString=HostName=your-iothub-name.azure-devices.net;SharedAccessKeyName=device;SharedAccessKey=your-iothub-key" -e Template="{ \"deviceId\": \"$.DeviceId\", \"rand_int\": $.Temp, \"rand_double\": $.DoubleValue, \"Ticks\": $.Ticks, \"Counter\": $.Counter, \"time\": \"$.Time\" }" -e Variables="[{name: \"Temp\", \"random\": true, \"max\": 25, \"min\": 23}, {\"name\":\"Counter\", \"min\":100, \"max\":102} ]" mcr.microsoft.com/oss/azure-samples/azureiot-telemetrysimulator
```

calling from PowerShell:

```powershell
docker run -it -e "IotHubConnectionString=HostName=your-iothub-name.azure-devices.net;SharedAccessKeyName=iothubowner;SharedAccessKey=your-iothub-key" -e Template="{ \"""deviceId\""": \"""$.DeviceId\""", \"""rand_int\""": $.Temp, \"""rand_double\""": $.RandomDouble , \"""Ticks\""": $.Ticks, \"""Counter\""": $.Counter, \"""time\""": \"""$.Time\""", \"""engine\""": \"""$.Engine\""" }" -e Variables="[{name: \"""Temp\""", \"""random\""": true, \"""max\""": 25, \"""min\""": 23}, {\"""name\""":\"""Counter\""", \"""min\""":100, \"""max\""":102}, {name:\"""Engine\""", values: [\"""on\""", \"""off\"""]}]" -e DeviceCount=1 -e MessageCount=3 mcr.microsoft.com/oss/azure-samples/azureiot-telemetrysimulator
```

#### Example 2: Adding the engine status ("on" or "off") to the telemetry

Template:

```json
{ "deviceId": "$.DeviceId", "rand_int": $.Temp, "Ticks": $.Ticks, "Counter": $.Counter, "time": "$.Time", "engine": "$.Engine" }
```

Variables:

```json
[{"name": "Temp", "random": true, "max": 25, "min": 23}, {"name":"Counter", "min":100}, {"name": "Engine", "values": ["on", "off"]}]
```

Output:

```json
{ "deviceId": "sim000001", "rand_int": 23, "Ticks": 637097644549666920, "Counter": 100, "time": "2019-11-19T12:47:34.9666920Z", "engine": "off" }
{ "deviceId": "sim000001", "rand_int": 24, "Ticks": 637097644550326096, "Counter": 101, "time": "2019-11-19T12:47:35.0326096Z", "engine": "on" }
```

Running with Docker:

```bash
docker run -it -e "IotHubConnectionString=HostName=your-iothub-name.azure-devices.net;SharedAccessKeyName=device;SharedAccessKey=your-iothub-key" -e Template="{ \"deviceId\": \"$.DeviceId\", \"rand_int\": $.Temp, \"Ticks\": $.Ticks, \"Counter\": $.Counter, \"time\": \"$.Time\", \"engine\": \"$.Engine\" }" -e Variables="[{name: \"Temp\", \"random\": true, \"max\": 25, \"min\": 23}, {\"name\":\"Counter\", \"min\":100}, {name:\"Engine\", values: [\"on\", \"off\"]}]" mcr.microsoft.com/oss/azure-samples/azureiot-telemetrysimulator
```

#### Example 3: Using a configuration file to customize simulation

```bash
docker run -it -e "IotHubConnectionString=HostName=your-iothub-name.azure-devices.net;SharedAccessKeyName=device;SharedAccessKey=your-iothub-key" -e "File=/config_files/test4-config-multiple-internals-per-device.json" -e DeviceCount=3 --mount type=bind,source=$pwd\test\IotTelemetrySimulator.Test\test_files,target=/config_files,readonly mcr.microsoft.com/oss/azure-samples/azureiot-telemetrysimulator
```

Where the file content is:

```plain
{
  "Variables": [
    {
      "name": "DeviceSequenceValue1",
      "sequence": true,
      "values": [ "$.Counter", "$.Counter", "$.Counter", "$.Counter", "$.Counter", "true", "false", "$.Counter" ]
    },
    {
      "name": "Device1Tags",
      "sequence": true,
      "values": [ "['ProducedPartCount']", "['ProducedPartCount']", "['ProducedPartCount']", "['ProducedPartCount']", "['ProducedPartCount']", "['Downtime']", "['Downtime']", "['ProducedPartCount']" ]
    },
    {
      "name": "Device1Downtime",
      "values": [ "true", "true", "true", "true", "false" ]
    },
    {
      "name": "Counter"
    }
  ],
  "Intervals": {
    "sim000001": 10000,
    "sim000002": [ 100 , 200 ]
  },
  "Payloads": [
    {
      "type": "template",
      "deviceId": "sim000001",
      "template": "{\"device\":\"$.DeviceId\",\"value\":\"$.DeviceSequenceValue1\",\"tags\": $.Device1Tags}"
    },
    {
      "type": "fix",
      "deviceId": "sim000002",
      "value": "{\"value\":\"myfixvalue\"}"
    },
    {
      "type": "template",
      "deviceId": "sim000003",
      "template": "{\"device\":\"$.DeviceId\",\"a\":\"b\",\"value\":\"$.DeviceSequenceValue1\"}"
    }
  ]
}
```

## Generating high volume of telemetry

In order to generate a constant high volume of messages a single computer might not be enough. This section describes two way to run the simulator to ingest a high volume of messages

### Azure Container Instance

Azure has container instances which allow the execution of containers with micro billing. This repository has a PowerShell script that creates azure container instances in your subscription. Requirements are having [az cli installed](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest).

To start the simulator in a single container instance:

```powershell
.\SimulatorCloudRunner.ps1
```

You will be asked to enter the Azure IoT Hub Connection string. After that, a resource group and one or more container instances will be created.

The cloud runner can be customized with the following parameters (as `-ParameterName ParameterValue`):

|Name|Description|
|-|-|
|Location|Location of the resource group being created. (Default = westeurope). For a list of locations try `az account list-locations -o table`|
|ResourceGroup|Resource group (will be created if it does not exist) where container instances will be created. (Default = iothubsimulator)|
|DeviceCount|Total amout of devices (Default = 100)|
|ContainerCount|Total amount of container instances to create. The total DeviceCount will be divided among all instances (Default = 1)|
|MessageCount|Total amount of messages to send per device. 0 means no limit, **causing the container to never end. It is your job to stop and delete it!** (Default = 100)|
|Interval|Interval in which each device will send messages in milliseconds (Default = 1000)|
|Template|Telemetry payload template to be used<br />(Default = '{ \"deviceId\": \"$.DeviceId\", \"temp\": $.Temp, \"Ticks\": $.Ticks, \"Counter\": $.Counter, \"time\": \"$.Time\", \"engine\": \"$.Engine\", \"source\": \"$.MachineName\" }')|
|PayloadDistribution|Allows the generation of payloads based on a distribution<br />Example: "fixSize(10, 12) template(25, default) fix(65, aaaaBBBBBCCC)" generates 10% a fix payload of 10 bytes, 25% a template generated payload and 65% of the time a fix payload from values aaaaBBBBBCCC|
|Header|Header properties template to be used<br />(Default = '')|
|Variables|Variables used to create the telemetry<br />(Default = '[{name: \"Temp\", random: true, max: 25, min: 23}, {name:\"Counter\", min:100}, {name:\"Engine\", values: [\"on\", \"off\"]}]')|
|Cpu|Amount of cpus allocated to each container instance (Default = 1.0)|
|IotHubConnectionString|Azure Iot Hub connection string|

### Kubernetes

This repository also contains a helm chart to deploy the simulator to a Kubernetes cluster. An example release with helm for 5000 devices in 5 pods:

```bash
helm install sims iot-telemetry-simulator\. --namespace iotsimulator --set iotHubConnectionString="HostName=xxxx.azure-devices.net;SharedAccessKeyName=iothubowner;SharedAccessKey=xxxxxxx" --set replicaCount=5 --set deviceCount=5000
```

## Automation

The `IotTelemtrySimulator.Automation` .NET Core 2.1 library allows you to run the IoT Telemetry Simulator as part of a pipeline or any other automation framework. See the [automation guide](AUTOMATION.md).

## Advanced options

* *PartitionKey option for Event Hubs*: When processing events downstream with a distributed engine, it is often more efficient to have related messages in the same partition. For example, if computing values over windows of events per device in Spark, having all the events for a given device in the same partition ensures they are received by a single compute node and ordered. On the other hand, the default round-robin behavior has advantages too, such as ensuring all partitions have equal load and that the system remains available even if a partition fails. Therefore, it is good to give the user control over the partition key.
* *DuplicateEvery option to duplicate events*: Upstream systems generating events usually offer at-least-once delivery guarantees. Given that there is no mechanism in IoT Hub / Event Hubs for distributed transactions, it is always possible for the client not to receive the response from the server after sending a message, or crashing before persisting the response, and therefore to retry sending an event. This results in message duplicates. When testing event processing solution, it's important to inject such behavior so as to test if/how the system reacts to it, e.g. by deduplication. The option `DuplicateEvery` allows randomly duplicating a fraction of messages, e.g. setting `DuplicateEvery` to 1000 will duplicate messages with a probability of 1/1000 for each, i.e. will duplicate on average every 1000th message. Note that messages may be duplicated more than once: a message will be duplicated N times with a probability of `1/(DuplicateEvery)^N`.
