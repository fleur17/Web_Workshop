# Contrôle de LED ESP32 et Portfolio de Fleuriane

Ce dépôt contient deux projets web : **Contrôle de LED ESP32** et **Portfolio de Fleuriane**. Ces deux projets utilisent des technologies simples comme HTML, CSS et JavaScript pour des utilisations éducatives et personnelles.

---

## 1. Contrôle de LED ESP32

Cette page web vous permet de contrôler la luminosité d'une LED connectée à un ESP32 via une communication WebSocket. Elle inclut un curseur interactif qui envoie les valeurs de luminosité à l'ESP32 en temps réel.

### Fonctionnalités
- **Communication WebSocket** : Envoie les valeurs de luminosité à l'ESP32 lorsque l'utilisateur ajuste le curseur.
- **Interface dynamique** : Le curseur reflète les changements en temps réel de la luminosité de la LED.
- **Design responsive** : Utilise du CSS pour un affichage optimal sur différents appareils et tailles d'écran.

### Technologies utilisées
- HTML
- CSS
- JavaScript (WebSocket)
- ESP32

### Instructions d'utilisation
1. Remplacez l'adresse IP dans l'URL de connexion WebSocket dans la balise `<script>` par l'adresse IP locale de votre ESP32.
   ```js
   const ws = new WebSocket('ws://<ADRESSE_IP_ESP32>:81');


![FIGMA](Images/Untitled.png)
### Vidéo 1 : ESP32 LED Control

[![ESP32 LED Control](https://img.youtube.com/vi/_ns90kN3uhI/0.jpg)](https://youtube.com/shorts/_ns90kN3uhI?si=kKHP-_lT6LdO7jMJ)

### Vidéo 2 : ESP32 Potentiometer

[![Potentiometer](https://img.youtube.com/vi/aDV1lDQf9Tw/0.jpg)](https://youtube.com/shorts/aDV1lDQf9Tw?si=-hLq9VQSi8pdX7zh)


### Code arduino

#include <Arduino.h>
#include <WiFi.h>
#include <WebSocketsServer.h>

// Informations de connexion Wi-Fi
const char* ssid = "devolo-711";     // Remplacez par le nom de votre réseau WiFi
const char* password = "UFXPYLKMMYJFYVFW";  // Remplacez par votre mot de passe WiFi

// Pins pour la LED, le potentiomètre, et le bouton
const int ledPin = 5;   // Pin pour la LED contrôlée par le potentiomètre
const int potPin = 0;   // Pin analogique pour le potentiomètre
const int ledPin1 = 6;  // Pin pour la LED contrôlée par WebSocket
const int buttonPin = 8;  // Pin pour le bouton physique

// Variables globales pour le bouton et le minuteur
unsigned long startTime = 0;
bool timerActive = false;
unsigned long elapsedTime = 0;
bool ledState = false; // État de la LED
int potValue = 0; // Valeur du potentiomètre

// Serveur WebSocket
WebSocketsServer webSocket = WebSocketsServer(81);

// Fonction pour gérer les événements WebSocket
void webSocketEvent(uint8_t num, WStype_t type, uint8_t * payload, size_t length) {
  switch(type) {
    case WStype_DISCONNECTED:
      Serial.printf("[%u] Disconnected!\n", num);
      break;
    case WStype_CONNECTED: {
      IPAddress ip = webSocket.remoteIP(num);
      Serial.printf("[%u] Connected from %s\n", num, ip.toString().c_str());
      break;
    }
    case WStype_TEXT: {
      Serial.printf("[%u] Received text: %s\n", num, payload);

      // Si "LED_ON" est reçu, allumer la LED
      if (strcmp((char*)payload, "LED_ON") == 0) {
        digitalWrite(ledPin1, HIGH);  // Allumer la LED
        ledState = true;
        Serial.println("LED is ON");
        webSocket.sendTXT(num, "LED is ON");
      }
      // Si "LED_OFF" est reçu, éteindre la LED
      else if (strcmp((char*)payload, "LED_OFF") == 0) {
        digitalWrite(ledPin1, LOW);  // Éteindre la LED
        ledState = false;
        Serial.println("LED is OFF");
        webSocket.sendTXT(num, "LED is OFF");
      }
      // Si un nombre (pourcentage de luminosité) est reçu
      else {
        int brightness = atoi((char*)payload);
        Serial.printf("Setting brightness to: %d\n", brightness);
        analogWrite(ledPin1, map(brightness, 0, 100, 0, 255)); // Ajuster la luminosité de la LED
      }
      break;
    }
  }
}

void setup() {
  // Initialisation des pins
  pinMode(ledPin, OUTPUT);   // LED contrôlée par potentiomètre
  pinMode(ledPin1, OUTPUT);  // LED contrôlée par WebSocket
  pinMode(buttonPin, INPUT_PULLUP); // Bouton pour le minuteur
  digitalWrite(ledPin1, LOW); // Éteindre la LED contrôlée par WebSocket

  // Démarrage du moniteur série
  Serial.begin(115200);

  // Connexion au Wi-Fi
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(1000);
    Serial.println("Connecting to WiFi...");
  }
  Serial.println("Connected to WiFi");
  Serial.print("ESP32 IP address: ");
  Serial.println(WiFi.localIP());

  // Démarrer le serveur WebSocket
  webSocket.begin();
  webSocket.onEvent(webSocketEvent); // Attacher la fonction de gestion des événements
  Serial.println("WebSocket server started");
}

void loop() {
  // Gestion des connexions WebSocket
  webSocket.loop();

  // Lire la valeur du potentiomètre
  potValue = analogRead(potPin);

  // Mapper la valeur du potentiomètre (0-4095) à la luminosité de la LED (0-255)
  int ledBrightness = map(potValue, 0, 4095, 0, 255);
  
  // Ajuster la luminosité de la LED contrôlée par le potentiomètre
  analogWrite(ledPin, ledBrightness);

  // Envoyer les données du potentiomètre sur WebSocket
  String potData = "POT_VALUE:" + String(potValue);
  webSocket.broadcastTXT(potData);

  // Affichage de débogage dans le moniteur série
  Serial.print("Potentiometer Value: ");
  Serial.print(potValue);
  Serial.print(" -> LED Brightness: ");
  Serial.println(ledBrightness);

  // Lecture de l'état du bouton
  int buttonState = digitalRead(buttonPin);

  if (buttonState == LOW && !timerActive) {
    // Si le bouton est pressé et que le minuteur n'est pas actif, on le démarre
    startTime = millis();  // Enregistrer l'heure de départ
    timerActive = true;
    Serial.println("Timer started!");
  }

  if (timerActive) {
    // Calculer le temps écoulé
    elapsedTime = millis() - startTime;

    // Envoyer le temps écoulé au client WebSocket toutes les secondes
    if (elapsedTime % 1000 == 0) {
      String timerMessage = "TIME_ELAPSED:" + String(elapsedTime / 1000);  // Temps en secondes
      webSocket.broadcastTXT(timerMessage);  // Envoyer le temps à tous les clients connectés
      Serial.println(timerMessage);          // Affichage dans le moniteur série
    }
  }

  delay(100); // Ajuster ce délai pour des mises à jour plus fluides
}




