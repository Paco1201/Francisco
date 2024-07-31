#include <WiFi.h>
#include "DHTesp.h"
#include <ESP32Servo.h>

// Definición del pin y tipo de sensor DHT
int pinDHT = 18;
DHTesp dht;

// Definición del pin del servo
int servoPin = 19;
Servo myservo;

// Configuración de la red Wi-Fi
const char* ssid = "11T Pro";       // Reemplaza con tu SSID de Wi-Fi
const char* password = "Lema2023";   // Reemplaza con tu contraseña de Wi-Fi

WiFiServer server(80);

void setup() {
  Serial.begin(115200);
  dht.setup(pinDHT, DHTesp::DHT11);  // Inicializa el sensor DHT
  myservo.attach(servoPin);          // Inicializa el servo

  // Conexión a la red Wi-Fi
  WiFi.begin(ssid, password);
  Serial.println("Conectando a WiFi...");
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.print(".");
  }
  Serial.println("");
  Serial.println("Conectado a WiFi");
  Serial.println(WiFi.localIP());

  server.begin();
}

void loop() {
  WiFiClient client = server.available();
  if (client) {
    Serial.println("Cliente conectado");
    String currentLine = "";
    while (client.connected()) {
      if (client.available()) {
        char c = client.read();
        Serial.write(c);
        if (c == '\n') {
          if (currentLine.length() == 0) {
            TempAndHumidity data = dht.getTempAndHumidity();
            String temperature = String(data.temperature, 2);
            String humidity = String(data.humidity, 1);

            // Mueve el servo basado en la temperatura
            int angle = map(data.temperature, 0, 40, 0, 180); // Mapea la temperatura (0-40°C) al ángulo del servo (0-180°)
            myservo.write(angle);

            client.println("HTTP/1.1 200 OK");
            client.println("Content-type:text/html");
            client.println();

            client.println("<!DOCTYPE html><html>");
            client.println("<head><meta http-equiv='refresh' content='5'/>");
            client.println("<title>ESP32 DHT11</title></head>");
            client.println("<body><h1>Temperatura y Humedad</h1>");
            client.println("<p>Temperatura: " + temperature + " °C</p>");
            client.println("<p>Humedad: " + humidity + " %</p>");
            client.println("<p>Servo Angle: " + String(angle) + "°</p>");
            client.println("</body></html>");

            break;
          } else {
            currentLine = "";
          }
        } else if (c != '\r') {
          currentLine += c;
        }
      }
    }
    client.stop();
    Serial.println("Cliente desconectado");
  }
}
