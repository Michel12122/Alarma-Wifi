//# Alarma-Wifi
//Alarma con Esp 8266

//**************************************************************

//  -- Configuracion Hardware
#define PIN_TAMP 13 //D7
#define PIN_LEDPILOTO 14   //D5
#define PIN_SENSOR 5     //D1
#define PIN_SIRENA 12    //D6
//#define PIN_ON_OFF 4 //D2
//-------------------------------------------------------------
#define CANTIDAD_DETECCIONES 5              // Poner la cantidad de detecciones necesarias para considerarse movimiento prolongado
#define TIEMPO_REARMADO 120                  // Es el tiempo en segundos que tiene que pasar SIN detectar movimiento para que se reinicie la alarma.
#define TIEMPO_SONIDO_ALARMA 10 //300            // Es el tiempo que suena la sirena antes de frenar.  Al frenaar si sigue detectando movimiento, va a sonar de nuevo.
#define TIEMPO_LED_STATE 1
//  ------------------------------

#define BLYNK_PRINT Serial //Comente esto para desactivar impresiones y ahorrar espacio
#include <ESP8266WiFi.h>
#include <BlynkSimpleEsp8266.h>

//Su authtoken generado por la aplicacion Blynk
char auth[] = "-------------";
//Datos para la conexion de Red Wifi.
char ssid[] = "-----";     //Nombre de la red WIFI
char pass[] = "----"; //contraseña de la red WIFI
int x = 0;

//int SirenaState = LOW;
volatile bool RifState = true;
volatile bool State_Sirena = true;
const uint8_t PIN_ON_OFF = 4; //D2
bool StateLed = true;
bool Seguro = true;
bool deteccionMovimiento = false;
bool movimientoProlongado = false;
bool intrusion = false;
byte numeroMovimientos = false;
int numeroMovimientosLed = 0;
int numeroMovimientosTampers = 0;
int ct = 20;
unsigned long tiempoUltimaDeteccionMovimiento = 0;
bool correoNormalActivado = true;
unsigned long tiempoInicioSonidoAlarma = 0;
unsigned long tiempoInicioLedState = 0;
unsigned long tiempoStopLedState = 0;
unsigned long tAnterior;

//Funcion para salir de la memoria Cache y ejecutar las inteerupciones sin problema

void ICACHE_RAM_ATTR OnOff();

void setup()
{
  Serial.begin(115200);
  Blynk.begin(auth, ssid, pass);
  attachInterrupt(PIN_ON_OFF, OnOff, RISING);

  pinMode(PIN_ON_OFF, INPUT);
  pinMode(PIN_SENSOR, INPUT);     // pin D1(GPIO5) como entrada
  pinMode(PIN_SIRENA, OUTPUT);    // pin D2(GPIO4) como Salida
  pinMode(PIN_LEDPILOTO, OUTPUT); //pin D0(GPI016)como Salida
  digitalWrite(PIN_SIRENA, LOW);
  digitalWrite(PIN_LEDPILOTO, LOW);


  Serial.println("Espere un momento, activando alarma...");
  for (int i = 0; i <= ct; i++) {
    Serial.println(((i * 100 / ct)));
    Serial.print("% ");
    Serial.println("COMPLETADO.....");
    delay(1000);
  }
  Serial.println("Calibracion completada, Alarma activada");
  delay(50);
}
void OnOff() {

  RifState == false;

  digitalWrite(PIN_SIRENA, LOW);
  digitalWrite(PIN_LEDPILOTO, LOW);


}

void Alarma() {

  deteccionMovimiento = !digitalRead(PIN_SENSOR); // variable para almacenar los estados del PIR
  if (deteccionMovimiento == true && RifState == true)
  {
    delay(1000);
    
    digitalWrite(PIN_SIRENA, HIGH);
    tiempoInicioSonidoAlarma = millis();
    tiempoUltimaDeteccionMovimiento = millis();
    
   
    if (intrusion == false)
    {
      intrusion = true;
      if (correoNormalActivado == true)
      {
        Blynk.email("-------", "Alarma:", "!!Movimiento detectado!!");
        Serial.println("Se ha detectado movimiento y se envio un correo!");
        correoNormalActivado == false;
      }
    }
    else
    {
      numeroMovimientos++;
      numeroMovimientosLed++;

      if (numeroMovimientos == CANTIDAD_DETECCIONES)
      {

        Blynk.email("------", "Alarma:", "!!Se han detectado más de 5 movimientos!!");
        Serial.println("Se ha detectado movimiento constante y se envio un correo!");

      }
    }
  }


  if (millis() - tiempoUltimaDeteccionMovimiento > (TIEMPO_REARMADO * 1000))
  {
    correoNormalActivado = true;
    numeroMovimientos = 0;
    intrusion = false;

  }


  if (millis() - tiempoInicioSonidoAlarma > (TIEMPO_SONIDO_ALARMA * 1000))
  {
    digitalWrite(PIN_SIRENA, LOW);

  }

}
void Led() {

  if ((numeroMovimientosLed == 0) && (StateLed == true)) {
    if (millis() - tiempoInicioLedState > (TIEMPO_LED_STATE * 500)) {

      digitalWrite(PIN_LEDPILOTO, HIGH);
      tiempoInicioLedState = millis();

    } else if (millis() - tiempoStopLedState > (TIEMPO_LED_STATE * 1000) ) {

      digitalWrite(PIN_LEDPILOTO, LOW);
      tiempoStopLedState = millis();

    }
    StateLed == false;
  }

  if ((numeroMovimientosLed >= 1 ) && (StateLed == false)) {

    digitalWrite(PIN_LEDPILOTO, HIGH);

  }

}





void loop()
{
 
  if (WiFi.status() == WL_CONNECTED) {

    Led();
    Alarma();


  } else {

    Led();
    Alarma();
  }

  Blynk.run();

}
