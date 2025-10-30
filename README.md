#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

// --- OLED ---
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// --- SENSOR MQ-5 ---
const int mq5Pin = A1;

// --- SENSOR ULTRASÓNICO (ENTRADA - 3 PINES) ---
#define ULTRA_PIN_ENTRADA 3  // Trig + Echo combinados

// --- SENSOR ULTRASÓNICO (INTERIOR - 4 PINES) ---
#define TRIG_INTERIOR 4
#define ECHO_INTERIOR 5

// --- DRIVER TB6612FNG (8833) ---
#define AIN1 13
#define AIN2 12
#define BUZZER_PIN 2

// --- VARIABLES ---
float distanciaEntrada = 0;
float distanciaInterior = 0;
float nivelGas = 0;

const float UMBRAL_ENTRADA = 20.0;   // cm
const float UMBRAL_INTERIOR = 15.0;  // cm
const int GAS_UMBRAL = 60;           // % de gas para activar alarma

int pasos = 200;
int delayPaso = 5;
bool puertaAbierta = false;
bool entradaDetectada = false;

void setup() {
  Serial.begin(9600);

  // Inicializar OLED
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("Error al iniciar pantalla OLED");
    while (true);
  }

  // Configurar pines
  pinMode(ULTRA_PIN_ENTRADA, OUTPUT);
  pinMode(TRIG_INTERIOR, OUTPUT);
  pinMode(ECHO_INTERIOR, INPUT);
  pinMode(mq5Pin, INPUT);
  pinMode(AIN1, OUTPUT);
  pinMode(AIN2, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  // Estado inicial
  digitalWrite(TRIG_INTERIOR, LOW);

  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(10, 20);
  display.println("Sistema Iniciando...");
  display.display();
  delay(1500);
}

void loop() {
  // --- MEDICIÓN DE SENSORES ---
  distanciaEntrada = medirDistancia3Pin(ULTRA_PIN_ENTRADA);
  distanciaInterior = medirDistancia4Pin(TRIG_INTERIOR, ECHO_INTERIOR);
  int sensorValue = analogRead(mq5Pin);
  nivelGas = map(sensorValue, 0, 1023, 0, 100);

  bool entradaActiva = (distanciaEntrada > 0 && distanciaEntrada <= UMBRAL_ENTRADA);
  bool interiorActivo = (distanciaInterior > 0 && distanciaInterior <= UMBRAL_INTERIOR);

  // --- DETECCIÓN DE GAS (ALARMA) ---
  if (nivelGas > GAS_UMBRAL) {
    activarAlarmaGas();
  }

  // --- ABRIR PUERTA (solo si no hay detección interior) ---
  if (entradaActiva && !interiorActivo && !puertaAbierta) {
    abrirPuerta();
    puertaAbierta = true;
    entradaDetectada = true;
  }

  // --- CERRAR PUERTA (si el interior detecta algo) ---
  if (interiorActivo && puertaAbierta) {
    cerrarPuerta();
    puertaAbierta = false;
    entradaDetectada = false;
  }

  mostrarOLED(entradaActiva, interiorActivo);

  // --- Depuración Serial ---
  Serial.print("Entrada: ");
  Serial.print(distanciaEntrada);
  Serial.print(" cm | Interior: ");
  Serial.print(distanciaInterior);
  Serial.print(" cm | Gas: ");
  Serial.print(nivelGas);
  Serial.println("%");

  delay(600);
}

// --- MEDIR DISTANCIAS ---
float medirDistancia3Pin(int pin) {
  pinMode(pin, OUTPUT);
  digitalWrite(pin, LOW);
  delayMicroseconds(2);
  digitalWrite(pin, HIGH);
  delayMicroseconds(10);
  digitalWrite(pin, LOW);
  pinMode(pin, INPUT);
  long duracion = pulseIn(pin, HIGH, 40000);
  if (duracion == 0) return 999;
  return duracion * 0.034 / 2;
}

float medirDistancia4Pin(int trigPin, int echoPin) {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);
  long duracion = pulseIn(echoPin, HIGH, 40000);
  if (duracion == 0) return 999;
  return duracion * 0.034 / 2;
}

// --- MOTOR ---
void abrirPuerta() {
  display.clearDisplay();
  display.setCursor(10, 20);
  display.println("Abriendo puerta...");
  display.display();

  for (int i = 0; i < pasos; i++) {
    digitalWrite(AIN1, HIGH);
    digitalWrite(AIN2, LOW);
    delay(delayPaso);
  }
  detenerMotor();
  tone(BUZZER_PIN, 800, 100);
}

void cerrarPuerta() {
  display.clearDisplay();
  display.setCursor(10, 20);
  display.println("Cerrando puerta...");
  display.display();

  for (int i = 0; i < pasos; i++) {
    digitalWrite(AIN1, LOW);
    digitalWrite(AIN2, HIGH);
    delay(delayPaso);
  }
  detenerMotor();

  for (int i = 0; i < 2; i++) {
    tone(BUZZER_PIN, 1000);
    delay(300);
    noTone(BUZZER_PIN);
    delay(200);
  }
}

void detenerMotor() {
  digitalWrite(AIN1, LOW);
  digitalWrite(AIN2, LOW);
}

// --- ALARMA DE GAS ---
void activarAlarmaGas() {
  Serial.println("⚠️ ALERTA: Nivel de gas alto ⚠️");
  for (int i = 0; i < 3; i++) {
    tone(BUZZER_PIN, 1500);
    delay(300);
    noTone(BUZZER_PIN);
    delay(200);
  }
  // Muestra en OLED la alerta
  display.clearDisplay();
  display.setTextSize(1);
  display.setCursor(10, 25);
  display.println("ALERTA: Gas alto!");
  display.setTextSize(2);
  display.setCursor(20, 40);
  display.print(nivelGas, 0);
  display.println("%");
  display.display();
}

// --- OLED ---
void mostrarOLED(bool entradaActiva, bool interiorActivo) {
  display.clearDisplay();
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println("Garaje Inteligente");

  display.setCursor(0, 15);
  if (entradaActiva)
    display.println("Vehiculo ingresando...");
  else if (interiorActivo)
    display.println("Vehiculo estacionado. Cerrando...");
  else if (!puertaAbierta)
    display.println("Puerta cerrada.");
  else
    display.println("Esperando...");

  display.setTextSize(2);
  display.setCursor(0, 35);
  display.print(nivelGas, 0);
  display.println("%");
  display.setTextSize(1);
  display.setCursor(70, 50);
  display.println("Gas");
  display.display();
}
