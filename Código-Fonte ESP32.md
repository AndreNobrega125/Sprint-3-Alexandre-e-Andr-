#include <Arduino.h>
#include <WiFi.h>
#include <Firebase_ESP_Client.h>
#include "addons/TokenHelper.h"
#include "addons/RTDBHelper.h"
#include <PZEM004Tv30.h>  // Biblioteca para o PZEM-004T v3.0

// ====== CONFIGURAÇÕES DE REDE ======
#define WIFI_SSID "Skynet"
#define WIFI_PASSWORD "Nobreg@670"

// ====== CONFIGURAÇÕES DO FIREBASE ======
#define API_KEY "AIzaSyDZDDJZ6tRR4K3Pvotv87hUWvkFB-j2-wc"
#define DATABASE_URL "https://ecowatcher-8a5da-default-rtdb.firebaseio.com/"

// Objetos Firebase
FirebaseData fbdo;
FirebaseAuth auth;
FirebaseConfig config;

unsigned long sendDataPrevMillis = 0;
bool signupOK = false;

// ====== CONFIGURAÇÃO DO PZEM ======
// Use os pinos disponíveis no ESP32 (ajuste se precisar)
// RX = 16, TX = 17 neste exemplo
PZEM004Tv30 pzem(Serial2, 16, 17);

void setup() {
  Serial.begin(115200);

  // Conectar no Wi-Fi
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);
  Serial.print("Conectando ao Wi-Fi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(300);
    Serial.print(".");
  }
  Serial.println("\nWi-Fi conectado!");
  Serial.print("IP: ");
  Serial.println(WiFi.localIP());

  // Configuração do Firebase
  config.api_key = API_KEY;
  config.database_url = DATABASE_URL;

  // Login anônimo
  if (Firebase.signUp(&config, &auth, "", "")) {
    Serial.println("Conectado ao Firebase!");
    signupOK = true;
  } else {
    Serial.printf("Erro Firebase: %s\n", config.signer.signupError.message.c_str());
  }

  config.token_status_callback = tokenStatusCallback;
  Firebase.begin(&config, &auth);
  Firebase.reconnectWiFi(true);
}

void loop() {
  // Ler dados do PZEM
  float corrente = pzem.current();
  float energia = pzem.energy();
  float fp = pzem.pf();
  float frequencia = pzem.frequency();
  float potencia = pzem.power();
  float tensao = pzem.voltage();

  // Mostrar no Serial Monitor
  Serial.println("====== Leitura PZEM ======");
  Serial.print("Tensao: "); Serial.println(tensao);
  Serial.print("Corrente: "); Serial.println(corrente);
  Serial.print("Potencia: "); Serial.println(potencia);
  Serial.print("Energia: "); Serial.println(energia);
  Serial.print("Fator Potencia: "); Serial.println(fp);
  Serial.print("Frequencia: "); Serial.println(frequencia);
  Serial.println("==========================");

  // Enviar para o Firebase a cada 2 segundos
  if (Firebase.ready() && signupOK && (millis() - sendDataPrevMillis > 2000 || sendDataPrevMillis == 0)) {
    sendDataPrevMillis = millis();

    Firebase.RTDB.setFloat(&fbdo, "dispositivo1/Corrente", corrente);
    Firebase.RTDB.setFloat(&fbdo, "dispositivo1/Energia", energia);
    Firebase.RTDB.setFloat(&fbdo, "dispositivo1/FP", fp);
    Firebase.RTDB.setFloat(&fbdo, "dispositivo1/Frequencia", frequencia);
    Firebase.RTDB.setFloat(&fbdo, "dispositivo1/Potencia", potencia);
    Firebase.RTDB.setFloat(&fbdo, "dispositivo1/Voltagem", tensao);
  }
}
