---
title: Quickstart Google Cloud Platform
menu:
    main:
        parent: guides
        weight: 1
description: Quickstart guide on hosting the LoRa Server components on Google Cloud Platform.
---

# Google Cloud Platform 快速起步

This tutorial describes the steps needed to setup the LoRa Server project
on the [Google Cloud Platform](https://cloud.google.com/). The following
Google Cloud Platform (GCP) services will be used:

* [Cloud IoT Core](https://cloud.google.com/iot-core/) is used to connect
  your LoRa gateways with the Google Cloud Platform.
* [Cloud Pub/Sub](https://cloud.google.com/pubsub/) is used for messaging
  between GCP components and LoRa Server services.
* [Cloud Functions](https://cloud.google.com/functions/) is used to handle
  downlink LoRa gateway communication (calling the Cloud IoT Core API on
  downlink Pub/Sub messages).
* [Cloud SQL](https://cloud.google.com/sql/) is used as hosted PostgreSQL
  database solution.
* [Cloud Memorystore](https://cloud.google.com/memorystore/) is used as
  hosted Redis solution.
* [Compute Engine](https://cloud.google.com/compute/) is used for launcing
  a VM instance.

## 美好的架设

* In this tutorial we will assume that the [LoRa Gateway Bridge](/lora-gateway-bridge/)
  component will be installed on the gateway. We will also assume that
  [LoRa Server](/loraserver/) and [LoRa App Server](/lora-app-server/) will be
  installed on a single Compute Engine VM, to simplify this tutorial.
* The project ID used in this tutorial will be `lora-server-tutorial`, you can
  substitute this with your own project ID in the examples.
* The LoRaWAN region used in this tutorial will be `eu868`, you can substitute
  this with your own region in the examples.

## 安装环境需求

* Google Cloud Platform account. You can create one [here](https://cloud.google.com/).
* LoRa gateway.
* LoRaWAN device.

## 创建 GCP 项目

After logging in into the GCP console, create a new project. For this tutorial
we will name the project `LoRa Server tutorial` with as ID
`lora-server-tutorial`. After creating the project, make sure it is selected
before continuing with the next steps.

## 网关连接

The [LoRa Gateway Bridge](/lora-gateway-bridge/) will use the
[Cloud IoT Core](https://cloud.google.com/iot-core/) MQTT interface to ingest
LoRa gateway events into the Google Cloud Platform. This removes the requirement
to host your own MQTT broker and increases the scalability.

### 新建节点注册

In order to connect your LoRa gateway with the [Cloud IoT Core](https://cloud.google.com/iot-core/)
component (using the MQTT bridge), go to the **IoT Core** service in the GCP
console and **create a new device registry**.

This registry will contain all your gateways for a given region. E.g. when you
are planning to support multiple LoRaWAN regions, it is a good practice to
create separate registries (not covered in this tutorial).

In this tutorial, we are going to create a registry for EU868 gateways, we choose
therefore the Registry ID `eu868-gateways`. Select the region which is closest
to you and select **MQTT** as protocol. The **HTTP** protocol will not be used.

Under **Default telemetry topic** create a new topic. We will call this
`eu868-gateway-events`. Click on **Create**.

### 创建网关认证

In order to authenticate the LoRa gateway with the Cloud IoT Core MQTT bridge,
you need to generate a certificate. You can do this using the following
commands:

{{<highlight bash>}}
ssh-keygen -t rsa -b 4096 -f private-key.pem
openssl rsa -in private-key.pem -pubout -outform PEM -out public-key.pem
{{< /highlight >}}

Do **not** set a passphrase!

### 添加设备 (LoRa gateway)

To add your first LoRa gateway to the just created device registry, click on
the **create device** button.

As **Device ID**, enter your Gateway ID prefixed with `gw-`. So when your
Gateway ID equals to `0102030405060708` you must enter `gw-0102030405060708`.
The `gw-` prefix is needed as the ID must start with a letter, which is not
always the case for a LoRa gateway ID.

Each Cloud IoT Core device (LoRa gateway) will authenticate using its own certificate.
Select **RS256** as **Public key format** and paste the public-key content in the box.
This is the content of `public-key.pem` which was created in the previous step.
Click on **Create**.

### 配置 LoRa Gateway Bridge

As there are different ways to install the [LoRa Gateway Bridge](/lora-gateway-bridge/)
on your gateway, only the configuration is covered here. For installation instructions,
please refer to [LoRa Gateway Bridge gateway installation & configuration](/lora-gateway-bridge/install/gateway/).

As LoRa Gateway Bridge will forwards its data to the Cloud IoT Core MQTT bridge,
you need update the `lora-gateway-bridge.toml` [Configuration file](/lora-gateway-bridge/install/config/).

Minimal configuration example:

{{< highlight toml >}}
[backend.mqtt]
marshaler="protobuf"

  [backend.mqtt.auth]
  type="gcp_cloud_iot_core"

    [backend.mqtt.auth.gcp_cloud_iot_core]
    server="ssl://mqtt.googleapis.com:8883"
    device_id="gw-0102030405060708"
    project_id="lora-server-tutorial"
    cloud_region="europe-west1"
    registry_id="eu868-gateways"
    jwt_key_file="/path/to/private-key.pem"
{{< /highlight >}}

In short:

* This will configure the `protobuf` marshaler (either `protobuf` or `json` must be configured)
* This will configure the Google Cloud IoT Core MQTT authentication
* Configures the GCP project ID, cloud-region and registry ID

Note that `jwt_key_file` must point to the private-key file generated in the
previous step.

After applying the above configuration changes (using your own `device_id`,
`project_id`, `cloud_region` and `jwt_key_file`), validate that LoRa Gateway Bridge
is able to connect with the Cloud IoT Core MQTT bridge. The log output should
look like this when your gateway receives an uplink message from your LoRaWAN device:

{{<highlight text>}}
INFO[0000] starting LoRa Gateway Bridge                  docs="https://www.loraserver.io/lora-gateway-bridge/" version=2.6.0
INFO[0000] gateway: starting gateway udp listener        addr="0.0.0.0:1700"
INFO[0000] mqtt: connected to mqtt broker
INFO[0007] mqtt: subscribing to topic                    qos=0 topic="/devices/gw-0102030405060708/commands/#"
INFO[0045] mqtt: publishing message                      qos=0 topic=/devices/gw-0102030405060708/events/up
{{< /highlight >}}

Your gateway is now communicating succesfully with the Cloud IoT Core MQTT bridge!

### 创建下行 Pub/Sub topic

Instead of MQTT, [LoRa Server](/loraserver/) will use [Cloud Pub/Sub](https://cloud.google.com/pubsub/)
for receiving data from your gateways and sending data back.

In the GCP console, navigate to **Pub/Sub > Topics**. You will see the topic
that was created when you created the device registry. LoRa Server will
subscribe to this topic to receive data (events) from your gateway.

For sending data back to your gateways, we will create a new topic called
`eu868-gateway-commands`. Click on **Create Topic**, enter the name of the
topic and click on **Create**.

### 创建 下行 Cloud Function

In the previous step you have created a topic for sending downlink commands
to your gateways. In order to connect this Pub/Sub topic with your
Cloud IoT Core device-registry, you must create a [Cloud Function](https://cloud.google.com/functions/)
which will subscribe to the downlink Pub/Sub topic and will forward these
commands to your LoRa gateway.

In the GCP console, navigate to **Cloud Functions**. Then click on **Create function**.
As **Name** we will use `eu868-gateway-commands`. As the only thing this function
does is calling an API endpoint, `128 MB` for **Memory allocated** should be fine.

Select **Cloud Pub/Sub** as **trigger** and select `eu868-gateway-commands` as
**topic**.

Select **Inline editor** for entering the source-code and select the **Node.js 8**
runtime. The **Function to execute** is called `sendMessage`. Copy & paste
the scripts below for the `index.js` and `package.json` files. Adjust the
`index.js` configuration to match your `REGION`, `PROJECT_ID` and `REGISTRY_ID`.
**Note:** it is recommended to also click on **More** and select your region
from the dropdown list. Then click on **Create**.

#### `index.js`

{{< highlight js >}}
'use strict';

const {google} = require('googleapis');

// configuration options
const REGION = 'europe-west1';
const PROJECT_ID = 'lora-server-tutorial';
const REGISTRY_ID = 'eu868-gateways';


let client = null;
const API_VERSION = 'v1';
const DISCOVERY_API = 'https://cloudiot.googleapis.com/$discovery/rest';


// getClient returns the GCP API client.
// Note: after the first initialization, the client will be cached.
function getClient (cb) {
  if (client !== null) {
    cb(client);
    return;
  }

  google.auth.getClient({scopes: ['https://www.googleapis.com/auth/cloud-platform']}).then((authClient => {
    google.options({
      auth: authClient
    });

    const discoveryUrl = `${DISCOVERY_API}?version=${API_VERSION}`;
    google.discoverAPI(discoveryUrl).then((c, err) => {
      if (err) {
        console.log('Error during API discovery', err);
        return undefined;
      }
      client = c;
      cb(client);
    });
  }));
}


// sendMessage forwards the Pub/Sub message to the given device.
exports.sendMessage = (event, context, callback) => {
  const deviceId = event.attributes.deviceId;
  const subFolder = event.attributes.subFolder;
  const data = event.data;
  
  getClient((client) => {
    const parentName = `projects/${PROJECT_ID}/locations/${REGION}`;
    const registryName = `${parentName}/registries/${REGISTRY_ID}`;
    const request = {
      name: `${registryName}/devices/${deviceId}`,
      binaryData: data,
      subfolder: subFolder
    };
    
    console.log("start call sendCommandToDevice");
    client.projects.locations.registries.devices.sendCommandToDevice(request, (err, data) => {
      if (err) {
        console.log("Could not send command:", request, "Message:", err);
        callback(new Error(err));
      } else {
        callback();
      }
    });
  });
};
{{< /highlight >}}

#### `package.json`

{{< highlight json >}}
{
  "name": "gateway-commands",
  "version": "2.0.0",
  "dependencies": {
    "@google-cloud/pubsub": "0.20.1",
    "googleapis": "34.0.0"
  }
}
{{< /highlight >}}

## 配置数据库

### 创建 Redis datastore

In the GCP console, navigate to **Memorystore** (which provides a managed
Redis datastore) and click **Create instance**.

You can assign any name to this instance. Make sure that you also select your
**Region**. Click **Create** to create the Redis instance.

### 创建 PostgreSQL 数据库

In the GCP console, navigate to **SQL** (which provides managed PostgreSQL
database instances) and click **Create instance**.

Select **PostgreSQL** and click **Next**. You can assign any name to this
instance. Again, make sure to also select your **Region** from the dropdown.

Configure the **Configuration options** to your needs (the smallest instance
is already sufficient for testing). An important option to configure is
**Authorize networks**. To allow access from *any* IP address, enter
`0.0.0.0/0`. It is recommended to re-configure this afterwards to the IP
address of your server (covered in the next steps). Then click **Create**.

#### 创建用户

Click on the created database instance and click on the **Users** tab.
Create two users:

* `loraserver_ns`
* `loraserver_as`

#### 创建数据库

Click on the **Databases** tab. Create the following databases:

* `loraserver_ns`
* `loraserver_as`

#### 使能 trgm 扩展

In the PostgreSQL instance **Overview** tab, click on **Connect using Cloud Shell**
and when the `gcloud sql connect ...` commands is shown in the console,
hit enter. It will prompt you for the `postgres` user password (which you
configured on creating the PostgreSQL instance).

Then execute the following SQL commands:

{{<highlight sql>}}
-- change to the LoRa App Server database
\c loraserver_as

-- enable the pq_trgm extension
-- (this is needed to facilidate the search feature)
create extension pg_trgm;

-- exit psql
\q
{{< /highlight >}}

You can close the Cloud Shell.

## 安装 LoRa Server

When you have succesfully completed the previous steps, then your gateway is
connected to the Cloud IoT Core MQTT bridge, are all LoRa (App) Server 
requirements setup and is it time to install [LoRa Server](/loraserver/) and
[LoRa App Server](/lora-app-server/).

### 创建虚拟机实例 VM instance

In the GCP console, navigate to **Compute Engine > VM instances** and click
on **Create**.

Again, the name of the instance doesn't matter but make sure you select the
correct **Region**. Depending your needs, the smallest **Machine type** could
already be sufficient to start. For this tutorial we will use the default
**Boot disk** (Debian 9).

Under **Identity and API access**, select **Allow full access to all Cloud APIs**
under the **Access scopes** options.

When you have a SSH public-key, enter this under
**SSH Keys**. You will find this option after clicking on
**Management, security, disks, networking, sole tenancy**. When all is
configured, click **Create**.

### Configure firewall

In order to expose the LoRa App Server web-interface, we need to open port
`8080` (the default LoRa App Server port) to the public.

Click on the created instance to go to the instance details. Under
**Network interfaces** click on **View details**. In the left navigation
menu click **Firewall rules** and then on **Create firewall rule**.
Enter the following details:

* **Name:** can be any name
* **Targets:** `All instances in the network`
* **Source IP ranges:** `0.0.0.0/0`
* **Protocols and ports > TCP:** `8080`

Then click on **Create**.

### Compute Engine 服务账户角色

As the Compute Engine instance (created in the previous step) needs to
be able to subscribe to the Pub/Sub data, we must assign give the
**Compute Engine default service account** the related role.

In the GCP console, navigate to **IAM & admin**. Then edit the **Compute Engine
default service account**. Click **Add another role** and add the following roles:

* `Pub/Sub Publisher`
* `Pub/Sub Subscriber`

### 登入虚拟机实例 VM instance

You will find the public IP address of the created VM instance under
**Compute Engine > VM instances**. When you have configured your public SSH key,
you should be able to login using a SSH client. Alternatively, you could use
SSH web-client provided by the GCP console.

### 配置 LoRa Server 仓库

Execute the following commands to add the LoRa Server repository to your VM
instance:

{{<highlight bash>}}
# add required packages
sudo apt install apt-transport-https dirmngr

# import LoRa Server key
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 1CE2AFD36DBCCA00

# add the repository to apt configuration
sudo echo "deb https://artifacts.loraserver.io/packages/2.x/deb stable main" | sudo tee /etc/apt/sources.list.d/loraserver.list

# update the package cache
sudo apt update
{{< /highlight >}}

### 安装网络服务器 LoRa Server

Execute the following command to install the LoRa Server service:

{{<highlight bash>}}
sudo apt install loraserver
{{< /highlight >}}

#### 配置网络服务器 LoRa Server

The LoRa Server configuration file is located at
`/etc/loraserver/loraserver.toml`. Below you will find two (minimal but working)
configuration examples. Please refer to the LoRa Server
[Configuration](/loraserver/install/config/) documentation for all the
available options.

**Important:** In the examples below the `rx1_delay` value has been set to `3`
as especially on a low message-rate, there might be a higher latency between
the Pub/Sub and Cloud Function components.

You need to replace the following values:

* **[PASSWORD]** with the `loraserver_ns` PostgreSQL user password
* **[POSTGRESQL_IP]** with the **Primary IP address** of the created PostgreSQL instance
* **[REDIS_IP]** with the **IP address** of the created Redis instance

To test if there are no errors, you can execute the following command:

{{<highlight bash>}}
sudo loraserver
{{< /highlight >}}

This should output something like (it is important that there are no errors):

{{<highlight text>}}
INFO[0000] setup redis connection pool                   url="redis://10.0.0.3:6379"
INFO[0000] connecting to postgresql
INFO[0000] gateway/gcp_pub_sub: setting up client
INFO[0000] gateway/gcp_pub_sub: setup downlink topic     topic=eu868-gateway-commands
INFO[0001] gateway/gcp_pub_sub: setup uplink topic       topic=eu868-gateway-events
INFO[0002] gateway/gcp_pub_sub: check if uplink subscription exists  subscription=eu868-gateway-events-loraserver
INFO[0002] gateway/gcp_pub_sub: create uplink subscription  subscription=eu868-gateway-events-loraserver
INFO[0005] applying database migrations
INFO[0006] migrations applied                            count=19
INFO[0006] starting api server                           bind="0.0.0.0:8000" ca-cert= tls-cert= tls-key=
{{< /highlight >}}

If all is well, then you can start the service in the background using:

{{<highlight bash>}}
sudo systemctl start loraserver
sudo systemctl enable loraserver
{{< /highlight >}}

##### EU868 配置示例

{{<highlight toml>}}
[postgresql]
dsn="postgres://loraserver_ns:[PASSWORD]@[POSTGRESQL_IP]/loraserver_ns?sslmode=disable"

[redis]
url="redis://[REDIS_IP]:6379"

[network_server]
net_id="000000"

  [network_server.band]
  name="EU_863_870"

  [network_server.network_settings]
  rx1_delay=3

  [network_server.gateway.stats]
  create_gateway_on_stats=true
  timezone="UTC"

  [network_server.gateway.backend]
  type="gcp_pub_sub"

    [network_server.gateway.backend.gcp_pub_sub]
    project_id="lora-server-tutorial"
    uplink_topic_name="eu868-gateway-events"
    downlink_topic_name="eu868-gateway-commands"
{{< /highlight>}}

##### US915 配置示例

{{<highlight toml>}}
[postgresql]
dsn="postgres://loraserver_ns:[PASSWORD]@[POSTGRESQL_IP]/loraserver_ns?sslmode=disable"

[redis]
url="redis://[REDIS_IP]:6379"

[network_server]
net_id="000000"

  [network_server.band]
  name="US_902_928"

  [network_server.network_settings]
  rx1_delay=3
  enabled_uplink_channels=[0, 1, 2, 3, 4, 5, 6, 7]

  [network_server.gateway.stats]
  create_gateway_on_stats=true
  timezone="UTC"

  [network_server.gateway.backend]
  type="gcp_pub_sub"

    [network_server.gateway.backend.gcp_pub_sub]
    project_id="lora-server-tutorial"
    uplink_topic_name="eu868-gateway-events"
    downlink_topic_name="eu868-gateway-commands"
{{< /highlight>}}

### 安装应用服务器 LoRa App Server

When you have completed all previous steps, then it is time to install the
last component, [LoRa App Server](/lora-app-server/). This is the
application-server that provides a web-interface for device management and
will publish application-data to a Pub/Sub topic.

#### 创建MQTT主题 Pub/Sub topic

In the GCP console, navigate to **Pub/Sub > Topics**. Then click on the
**Create topic** button to create a topic named `lora-app-server`.

#### 安装应用服务器 LoRa App Server

In your SSH client, execute the following command to install LoRa App Server:

{{<highlight bash>}}
sudo apt install lora-app-server
{{< /highlight >}}


#### 配置应用服务器 LoRa App Server

The LoRa App Server configuration file is located at
`/etc/lora-app-server/lora-app-server.toml`. Below you will find a minimal
but working configuration example. Please refer to the LoRa App Server
[Configuration](/lora-app-server/install/config/) documentation for all the
available options.

You need to replace the following values:

* **[PASSWORD]** with the `loraserver_as` PostgreSQL user password
* **[POSTGRESQL_IP]** with the **Primary IP address** of the created PostgreSQL instance
* **[REDIS_IP]** with the **IP address** of the created Redis instance
* **[JWT_SECRET]** with your own random JWT secret (e.g. the output of `openssl rand -base64 32`)

To test if there are no errors, you can execute the following command:

{{<highlight bash>}}
sudo lora-app-server
{{< /highlight >}}

This should output something like (it is important that there are no errors):

{{<highlight text>}}
INFO[0000] setup redis connection pool                   url="redis://10.0.0.3:6379"
INFO[0000] connecting to postgresql
INFO[0000] gateway/gcp_pub_sub: setting up client
INFO[0000] gateway/gcp_pub_sub: setup downlink topic     topic=eu868-gateway-commands
INFO[0001] gateway/gcp_pub_sub: setup uplink topic       topic=eu868-gateway-events
INFO[0002] gateway/gcp_pub_sub: check if uplink subscription exists  subscription=eu868-gateway-events-loraserver
INFO[0002] gateway/gcp_pub_sub: create uplink subscription  subscription=eu868-gateway-events-loraserver
INFO[0005] applying database migrations
INFO[0006] migrations applied                            count=19
INFO[0006] starting api server                           bind="0.0.0.0:8000" ca-cert= tls-cert= tls-key=
{{< /highlight >}}

If all is well, then you can start the service in the background using:

{{<highlight bash>}}
sudo systemctl start lora-app-server
sudo systemctl enable lora-app-server
{{< /highlight >}}

##### 配置示例

{{<highlight toml>}}
[postgresql]
dsn="postgres://loraserver_as:[PASSWORD]@[POSTGRESQL_IP]/loraserver_as?sslmode=disable"

[redis]
url="redis://[REDIS_IP]:6379"

[application_server]

  [application_server.integration]
  backend="gcp_pub_sub"

  [application_server.integration.gcp_pub_sub]
  project_id="lora-server-tutorial"
  topic_name="lora-app-server"

  [application_server.external_api]
  bind="0.0.0.0:8080"
  jwt_secret="[JWT_SECRET]"
{{< /highlight >}}

## 起步

### 配置第一个网关和节点

To get started with LoRa (App) Server, please follow the [First gateway and device]({{<relref "first-gateway-device.md">}})
guide. It will explain how to login into the web-interface and add your first
gateway and device.

### 集成应用

In the LoRa App Server step, you have created a Pub/Sub topic named
`lora-app-server`. This will be the topic used by LoRa Server for publishing
device events and to which your application(s) need to subscribe in order to
receive LoRaWAN device data.

For more information about Cloud Pub/Sub, please refer to the following pages:

* [Cloud Pub/Sub product page](https://cloud.google.com/pubsub/)
* [Cloud Pub/Sub documentation](https://cloud.google.com/pubsub/docs/)
* [Cloud Pub/Sub Quickstarts](https://cloud.google.com/pubsub/docs/quickstarts)
