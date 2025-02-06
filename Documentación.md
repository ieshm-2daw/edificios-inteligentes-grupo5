<h1>COnectar 2 ESP32 con los sensores PIR </h1>
<h2>Paso 1:</h2>
<p>Configuramos el código a través de la aplicación Arduino con lo 
  que introduciremos este primer código para configurar 
  el primer relé.</p>

  
# Conectar 2 ESP32 con los sensores PIR

## Paso 1:
Configuramos el código a través de la aplicación Arduino, introduciendo este primer código para configurar el primer relé.

## Configuración de los ESP32 con MQTT y sensores PIR
Este archivo muestra cómo configurar dos ESP32 para controlar un relé y monitorear sensores PIR a través de MQTT. A continuación, se detalla el código para ambos ESP32.

### Código para el primer ESP32 (Control de relé y monitoreo de PIR remoto)

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

// Configuración Wi-Fi
const char* ssid = "2DAW_IoT";
const char* password = "Somos2DAW";

// Configuración MQTT
const char* mqtt_server = "ha.ieshm.org"; 
const int mqtt_port = 1883;
const char* mqtt_user = "mqtt"; 
const char* mqtt_password = "mqtt"; 
const char* mqtt_topic = "g5/rele";  // Tópico para controlar el relé
const char* mqtt_log_topic = "g5/logs"; // Tópico para enviar logs
const char* mqtt_topic_pir2 = "g5/pir2"; // Tópico del PIR de la segunda ESP32


// Pines
#define LED_BUILTIN 2      // LED integrado en la placa
#define RELAY_PIN 23       // Pin donde está conectado el relé (GPIO 23)
#define PIR_PIN 27         // Pin donde está conectado el sensor PIR (GPIO 27)

// Variables globales
WiFiClient espClient;
PubSubClient client(espClient);
unsigned long relayEndTime = 0; // Tiempo en el que se debe apagar el relé
bool relayActive = false;       // Estado del relé
bool mqttControl = false;       // Prioridad de control desde MQTT

// Función para conectarse a la red Wi-Fi
void setup_wifi() {
  delay(10);
  Serial.println("Conectando a Wi-Fi...");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  Serial.println("\nWi-Fi conectado.");
  Serial.print("Dirección IP: ");
  Serial.println(WiFi.localIP());
}

// Función para enviar logs al broker MQTT
void sendLog(String logMessage) {
  if (client.connected()) {
    client.publish(mqtt_log_topic, logMessage.c_str());
  }
}

// Función que se ejecuta cuando se recibe un mensaje en el broker
void callback(char* topic, byte* payload, unsigned int length) {
  String message = "";
  for (unsigned int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  Serial.print("Mensaje recibido en el tópico ");
  Serial.print(topic);
  Serial.print(": ");
  Serial.println(message);

  // Manejo de mensajes MQTT
  if (String(topic) == mqtt_topic) { // Mensajes para el control del relé
    if (message == "ON") {
      mqttControl = true; // Habilita el control por MQTT
      sendLog("Relé habilitado por MQTT.");
    } else if (message == "OFF") {
      mqttControl = false; // Deshabilita el control por MQTT
      sendLog("Relé deshabilitado por MQTT.");
    }
  } else if (String(topic) == mqtt_topic_pir2) { // Mensajes del PIR de la segunda ESP32
    if (mqttControl && message == "DETECTED") {
      // Activa el relé si MQTT está en ON y hay movimiento en el PIR remoto
      Serial.println("Movimiento detectado en la segunda ESP32: Activando relé.");
      digitalWrite(RELAY_PIN, HIGH);
      digitalWrite(LED_BUILTIN, HIGH);
      relayActive = true;
      relayEndTime = millis() + 5000; // Temporizador de 5 segundos
      sendLog("Relé activado por segundo PIR.");
    }
  }
}


// Función para reconectar al broker MQTT si la conexión se pierde
void reconnect() {
  while (!client.connected()) {
    Serial.print("Intentando conectar al broker MQTT...");
    if (client.connect("ESP32", mqtt_user, mqtt_password)) {
      Serial.println("Conectado.");
      client.subscribe(mqtt_topic);      // Tópico para el control del relé
      client.subscribe(mqtt_topic_pir2); // Tópico del segundo PIR
      sendLog("Conectado al broker MQTT y suscrito a los tópicos.");
    } else {
      Serial.print("Fallo en la conexión, código de error: ");
      Serial.println(client.state());
      Serial.println("Reintentando en 5 segundos...");
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);

  pinMode(LED_BUILTIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(PIR_PIN, INPUT);

  setup_wifi();
  client.setServer(mqtt_server, mqtt_port);
  client.setCallback(callback);

  digitalWrite(RELAY_PIN, LOW);  // Apaga el relé al inicio
  digitalWrite(LED_BUILTIN, LOW); // Apaga el LED al inicio

  sendLog("ESP32 i...");
}



# Configuración ESP32 con Sensor PIR y MQTT

Este archivo contiene el código para configurar un ESP32 que se conecta a una red Wi-Fi y se comunica con un broker MQTT para enviar el estado de un sensor PIR.

## Código para el ESP32 (Sensor PIR y envío de estado vía MQTT)
#include <WiFi.h>
#include <PubSubClient.h>

// Configuración Wi-Fi
const char* ssid = "2DAW_IoT";
const char* password = "Somos2DAW";

// Configuración MQTT
const char* mqtt_server = "ha.ieshm.org";
const int mqtt_port = 1883;
const char* mqtt_user = "mqtt";
const char* mqtt_password = "mqtt";
const char* mqtt_topic_pir = "g5/pir2"; // Tópico para enviar el estado del PIR

// Pines
#define PIR_PIN 26 // Pin donde está conectado el sensor PIR (GPIO 27)

// Variables globales
WiFiClient espClient;
PubSubClient client(espClient);
bool lastPirState = LOW; // Estado anterior del PIR

// Función para conectarse a la red Wi-Fi
void setup_wifi() {
  delay(10);
  Serial.println("Conectando a Wi-Fi...");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  Serial.println("\nWi-Fi conectado.");
  Serial.print("Dirección IP: ");
  Serial.println(WiFi.localIP());
}

// Función para reconectar al broker MQTT
void reconnect() {
  while (!client.connected()) {
    Serial.print("Intentando conectar al broker MQTT...");
    if (client.connect("ESP32_PIR2", mqtt_user, mqtt_password)) {
      Serial.println("Conectado.");
    } else {
      Serial.print("Fallo en la conexión, código de error: ");
      Serial.println(client.state());
      Serial.println("Reintentando en 5 segundos...");
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);

  pinMode(PIR_PIN, INPUT);

  setup_wifi();
  client.setServer(mqtt_server, mqtt_port);

  Serial.println("ESP32 con PIR lista.");
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // Leer el estado del sensor PIR
  bool pirState = digitalRead(PIR_PIN);

  // Enviar estado si hay un cambio
  if (pirState != lastPirState) {
    lastPirState = pirState;
    if (pirState == HIGH) {
      client.publish(mqtt_topic_pir, "DETECTED");
      Serial.println("Movimiento detectado, enviado mensaje a MQTT.");
    }
  }
  delay(100); // Reducir la frecuencia de lectura
}

<h2>Segundo codigo </h2>

Al segundo código se le mete un segundo sensor PIR a lo igual que al  primero y su funcionamiento es igual pero actúa como una especie de repetidor si uno se enciende se le manda información al otro, lo detecta y enciende el relé.

#include <WiFi.h>
#include <PubSubClient.h>

// Configuración Wi-Fi
const char* ssid = "2DAW_IoT";
const char* password = "Somos2DAW";

// Configuración MQTT
const char* mqtt_server = "ha.ieshm.org";
const int mqtt_port = 1883;
const char* mqtt_user = "mqtt";
const char* mqtt_password = "mqtt";
const char* mqtt_topic_pir = "g5/pir2"; // Tópico para enviar el estado del PIR

// Pines
#define PIR_PIN 26 // Pin donde está conectado el sensor PIR (GPIO 27)

// Variables globales
WiFiClient espClient;
PubSubClient client(espClient);
bool lastPirState = LOW; // Estado anterior del PIR

// Función para conectarse a la red Wi-Fi
void setup_wifi() {
  delay(10);
  Serial.println("Conectando a Wi-Fi...");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  Serial.println("\nWi-Fi conectado.");
  Serial.print("Dirección IP: ");
  Serial.println(WiFi.localIP());
}

// Función para reconectar al broker MQTT
void reconnect() {
  while (!client.connected()) {
    Serial.print("Intentando conectar al broker MQTT...");
    if (client.connect("ESP32_PIR2", mqtt_user, mqtt_password)) {
      Serial.println("Conectado.");
    } else {
      Serial.print("Fallo en la conexión, código de error: ");
      Serial.println(client.state());
      Serial.println("Reintentando en 5 segundos...");
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);

  pinMode(PIR_PIN, INPUT);

  setup_wifi();
  client.setServer(mqtt_server, mqtt_port);

  Serial.println("ESP32 con PIR lista.");
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  client.loop();

  // Leer el estado del sensor PIR
  bool pirState = digitalRead(PIR_PIN);

  // Enviar estado si hay un cambio
  if (pirState != lastPirState) {
    lastPirState = pirState;
    if (pirState == HIGH) {
      client.publish(mqtt_topic_pir, "DETECTED");
      Serial.println("Movimiento detectado, enviado mensaje a MQTT.");
    }
  }
  delay(100); // Reducir la frecuencia de lectura
}

  <h1>Shelly EM </h1>

<p>La Shelly EM es una placa de medición de voltaje e intensidad. 
  Pero lo que más necesitamos para nuestro proyecto es la potencia.</p>
<p>Vamos a utilizar la placa para medir el consumo de energía que va a utilizar el aula,
  este proyecto puede ser escalable para otros usos como programar un relé wifi.O simplemente medir y tratar de ahorrar energía 
  que es el proyecto principal que tenemos en mente ahora mismo</p>

  <h2>Requerimiento</h2>
  <ul>
        <li>Shelly EM</li>
        <li>Shelly Transformador De Corriente 50A</li>
        <li>Conexión WiFi</li>
        <li>Cables</li>
        <li>Dispositivo Móvil</li>
    </ul>
    <h1>Conectar pinza </h1>
    <p>Conectar la pinza a la Shelly a través de uno de los puertos, ya sea P1 o P2. 
</p>
<h1>Paso 2: Alimentar el Shelly EM </h1>
<p>Conecta el Shelly EM a la red eléctrica (230V AC) siguiendo el esquema del fabricante.</p>
<p>Asegúrate de que la fase y el neutro estén bien conectados </p>



<h1>Paso 3: Configurar la conexión WiFi </h1>
<p>Enciende el Shelly EM y conéctate a su red WiFi (Shelly-XXXXXX) </p>
<p>Accede a la IP 192.168.33.1 desde un navegador o desde la aplicación móvil (recomendable)</p>
<p>Configura la conexión WiFi para que se una a tu red doméstica.</p>


<h1>Paso 4: Configurar en la App </h1>
<p>Si usas la app de Shelly, agrégalo a tu cuenta y personaliza las mediciones.</p>
<p>También puedes añadir la sala en la que quieras configurar la shelly</p>


<h1>Paso 5: Comprobar las mediciones</h1>
<p>Accede a la interfaz web o app para ver los valores en tiempo real.</p>
<p>Asegúrate de que la pinza está correctamente colocada en el cable de fase.</p>


<h1>Paso 6:  Configurar en Home Assistant</h1>
<p>Entra en tu Home Assistant y entra en dispositivos y servicios descarga su complemento</p>
<p>Configura la dirección IP</p>

<p></p>

