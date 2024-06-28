
## ESP & AWS IoT Core, MQTT-Based Temperature and Humidity Monitoring with S3 Data Storage

In this how you can connect the ESP with AWS IoT Core & publish sensor reading to AWS MQTT

## Hardware Setup::

![alt text](image/image.png)


Connect the DHT11 Sensor to ESP Board as per circuit diagram here.

## Amazon AWS IoT Core?
![alt text](image/image1.png)

AWS offers Internet of Things (IoT) services and solutions to connect and manage of devices. These cloud services connect your IoT devices to other devices and AWS cloud services. AWS IoT provides device software that can help you integrate your IoT devices into AWS IoT-based solutions. To manage and support your IoT devices in the field, AWS IoT Core supports these protocols:

- MQTT (Message Queuing and Telemetry Transport)
- MQTT over WSS (Websockets Secure)
- HTTPS (Hypertext Transfer Protocol – Secure)

## Started with Amazon AWS IoT Core with ESP
started with AWS IoT core, you just need to follow the steps given below to successfully set up the ESP board and start with your project.

 - **Signing in** :: [aws.amazon.com/iot-core/](https://aws.amazon.com/iot-core/)
 ## AWS IoT Core Dashboard 
 After signing in, the AWS Management Console window will open, search tab at the top write ‘IoT core’ & hit enter. On the left side of the dashboard, we need to work with two options, One is **manage option** and other is **secure option**.

 **Manage option** ::  
 create a thing associated with our project. For this, follow the steps:

- select create a single thing
- Specify Thing Properties, give a thing name. For example, I will name it *ESP32_DHT11*.
- Under additional configurations, there is no need any changes.
- Under device shadow option, select as No shadow.

### **Generate Device Certificate**
-  Need to configure device certificate. So here you can auto-generate a new certificate.
- we need to attach a policy to the Things, need to create a policy first.
- create policy, give any name to the policy. For example, name **ESP32_Policy**.
- In add statement part, Under the action, type IoT:Connect, and resource ARN :client/ESP32_DH11, and effect allow.
- Another add statement part, Under the action, type IoT:Receive, and resource ARN :topic/esp32/sub, and effect allow.
- create the policy, so policy has been created successfully.

Now go back to Create Thing, need to attach the policies to the certificate. So select the appeared policy and click on create a thing.

### Downloading Certificates and Keys
we need to download the required certificates from this list.
- First, download the **device certificate** and rename as a device certificate.
- Download the **public key** and rename as public key. Then download the **private key** and rename as private key.

- In the Root CA Certificates, there are two certificates here. But we just need a **Root CA1 certificate**, so download it.


## Code 
<details>
  <summary>Click to expand!</summary>
  
```
#include "secrets.h"
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>
#include "WiFi.h"
 
#include "DHT.h"
#define DHTPIN 14     // Digital pin connected to the DHT sensor
#define DHTTYPE DHT11   // DHT 11
 
#define AWS_IOT_PUBLISH_TOPIC   "esp32/pub"
#define AWS_IOT_SUBSCRIBE_TOPIC "esp32/sub"
 
float h ;
float t;
 
DHT dht(DHTPIN, DHTTYPE);
 
WiFiClientSecure net = WiFiClientSecure();
PubSubClient client(net);
 
void connectAWS()
{
  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
 
  Serial.println("Connecting to Wi-Fi");
 
  while (WiFi.status() != WL_CONNECTED)
  {
    delay(500);
    Serial.print(".");
  }
 
  // Configure WiFiClientSecure to use the AWS IoT device credentials
  net.setCACert(AWS_CERT_CA);
  net.setCertificate(AWS_CERT_CRT);
  net.setPrivateKey(AWS_CERT_PRIVATE);
 
  // Connect to the MQTT broker on the AWS endpoint we defined earlier
  client.setServer(AWS_IOT_ENDPOINT, 8883);
 
  // Create a message handler
  client.setCallback(messageHandler);
 
  Serial.println("Connecting to AWS IOT");
 
  while (!client.connect(THINGNAME))
  {
    Serial.print(".");
    delay(100);
  }
 
  if (!client.connected())
  {
    Serial.println("AWS IoT Timeout!");
    return;
  }
 
  // Subscribe to a topic
  client.subscribe(AWS_IOT_SUBSCRIBE_TOPIC);
 
  Serial.println("AWS IoT Connected!");
}
 
void publishMessage()
{
  StaticJsonDocument<200> doc;
  doc["humidity"] = h;
  doc["temperature"] = t;
  char jsonBuffer[512];
  serializeJson(doc, jsonBuffer); // print to client
 
  client.publish(AWS_IOT_PUBLISH_TOPIC, jsonBuffer);
}
 
void messageHandler(char* topic, byte* payload, unsigned int length)
{
  Serial.print("incoming: ");
  Serial.println(topic);
 
  StaticJsonDocument<200> doc;
  deserializeJson(doc, payload);
  const char* message = doc["message"];
  Serial.println(message);
}
 
void setup()
{
  Serial.begin(115200);
  connectAWS();
  dht.begin();
}
 
void loop()
{
  h = dht.readHumidity();
  t = dht.readTemperature();
 
 
  if (isnan(h) || isnan(t) )  // Check if any reads failed and exit early (to try again).
  {
    Serial.println(F("Failed to read from DHT sensor!"));
    return;
  }
 
  Serial.print(F("Humidity: "));
  Serial.print(h);
  Serial.print(F("%  Temperature: "));
  Serial.print(t);
  Serial.println(F("°C "));
 
  publishMessage();
  client.loop();
  delay(1000);
}

```
</details>         


**Modifying Arduino Sketch according to the Thing**     
Now modify the Arduino Sketch File. Go to **secrets.h** and begin the modification.

- we need to include a thing name, go to the things section of AWS Console and copy the thing name. Paste the thing name to the following line of code.
  > #define THINGNAME "***************"   
- Under the WiFi SSID and password, enter the WiFi SSID and Password of your local network.
  ```
  const char WIFI_SSID[] = "***************";               
  const char WIFI_PASSWORD[] = "***************"; 
  ```
- we need to insert the AWS IoT Endpoint. To get the endpoint, go to the settings part of AWS Dashboard. Yow will get the endpoint.  
Copy the endpoint, back to Arduino IDE and paste it on the following line.
  > const char AWS_IOT_ENDPOINT[] = "***************";  
-  Need to insert the Amazon Root CA1 certificate text in between the following line.
   ```
   // Amazon Root CA 1
   static const char AWS_CERT_CA[] PROGMEM = R"EOF(
   -----BEGIN CERTIFICATE-----
 
   -----END CERTIFICATE-----
   )EOF";
   ```

 - Under the **Device Certificate** lines, we need to paste the device certificate text.

   ```
   // Device Certificate                                              
   static const char AWS_CERT_CRT[] PROGMEM = R"KEY(
   -----BEGIN CERTIFICATE-----
 
   -----END CERTIFICATE-----
   ```
  
  - Under the **Device Private Key**, we need to insert the device’s private key text.
    
    ```
    // Device Private Key                                              
    static const char AWS_CERT_PRIVATE[] PROGMEM = R"KEY(
    -----BEGIN RSA PRIVATE KEY-----
 
    -----END RSA PRIVATE KEY-----
    ```

## Secrets.h
<details>
  <summary>Click to expand!</summary>
  
```
#include <pgmspace.h>
 
#define SECRET
#define THINGNAME "***************"                         //change this
 
const char WIFI_SSID[] = "***************";               //change this
const char WIFI_PASSWORD[] = "***************";           //change this
const char AWS_IOT_ENDPOINT[] = "***************";       //change this
 
// Amazon Root CA 1
static const char AWS_CERT_CA[] PROGMEM = R"EOF(
-----BEGIN CERTIFICATE-----
 
-----END CERTIFICATE-----
)EOF";
 
// Device Certificate                                               //change this
static const char AWS_CERT_CRT[] PROGMEM = R"KEY(
-----BEGIN CERTIFICATE-----
 
-----END CERTIFICATE-----
 
 
)KEY";
 
// Device Private Key                                               //change this
static const char AWS_CERT_PRIVATE[] PROGMEM = R"KEY(
-----BEGIN RSA PRIVATE KEY-----
 
-----END RSA PRIVATE KEY-----
 
 
)KEY";

```
</details>

## Testing of Data
Once all the modification is done, connect the ESP to your computer to upload the code to the ESP board.

- Open the Serial Monitor. The ESP will try connecting to the WiFi Network. Once it gets connected to the WiFi Network, it will try connecting to the AWS IoT Server.

  ![alt text](image\image2.png)


## Sensor Data to AWS Dashboard
|![alt text](image\image3.png) | ![alt text](image\image4.png)|
| - | - |

same thing should also be posted to the AWS Server. To check, go to **AWS IOT => test section**. Under the test section, option for subscribe and publish.   
- When you subscribe button, immediately the data from ESP32 will be uploaded to AWS Dashboard. Thus, you have successfully sent the DHT11 Sensor data to Amazon AWS IoT Core using ESP32.

- The data is updated after an interval of every one second, to receive the data to AWS IoT Core Dashboard sent from ESP32 via MQTT protocol.



## **Store AWS IoT data into S3 Bucket**
### Create S3 Bucket:
![alt text](image\image5.png)

AWS account, type S3 into the search box and select S3 from the services menu.
- Create **bucket**
- Enter the Bucket name as **s3-bucket-for-iot-data** and select an AWS region.

### Create IoT rule to send data to the S3:
![alt text](image\image6.png)

AWS account, type **IoT Core** and pick IoT Core from the services.
- In IoT Core, click rule and create the rule to transfer IoT data to the S3 bucket.
- Enter **IoT_data_rule_for_S3** as the **name** and define the rule in the description section.
- In Rule Query statement as **SELECT * FROM ‘iotdevice/+/datas3**, where * denotes reading the whole data sequence from the IoT topic iotdevice/+/datas3 and + denotes a wildcard character.
- Add an action to a rule, select **Add action**
- click **Store a message in an Amazon S3 bucket**, and then click **Configure action**.
![alt text](image\image7.png)

- section **Configure action** Choose name is **s3-bucket-for-iot-data**, for S3 bucket name. In the key, type **${cast(topic(2) AS DECIMAL)}**
- Need to provide role for AWS IoT rule to send data to S3 to **create Role**, enter **Iotdata_S3_role** as the role name.
- click **Add action** to add the role, Select Create rule. 

### 3. Test IoT rule and S3:
To test an IoT rule, go **AWS IoT Core**, select **Test**, and then **MQTT test client**. Click the **publish to a topic** and enter **iotdevice/55/datas3** as the Topic name and **Message payload {“temperature”: 22, “humidity”: 88}**. To publish the data, click the **Publish** option. Now the data is published.

![alt text](image\image8.png)

- To verify the published data, open the **S3** console and choose the bucket, creating the first step, **s3-bucket-for-iot-data** and find the created folder **55** and open that folder to see the published data.
- Published data found in objects console; click the time streamed data and Download to see the data.
![alt text](image\image9.png)

## Conclusion:
As a result, AWS IoT Core with the IoT Rule engine will assist in filtering IoT topics and the storage of data in AWS S3. AWS IoT core can receive and send of IoT data at a time, and the AWS IoT Rule engine can filter MQTT topics from IoT devices and send them to other AWS Services. For data backup and archive, AWS S3 will be used.
