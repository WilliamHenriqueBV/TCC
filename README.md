#include <Wire.h>
#include "MAX30100.h"
#include <MAX30100_PulseOximeter.h>
#include <Blynk.h>
#include <BlynkSimpleEsp8266.h>
#include <ESP8266WiFi.h>
#include <Adafruit_SSD1306.h> 


#define BLYNK_PRINT Serial
#define OLED_Address 0x3C 

Adafruit_SSD1306 oled(128, 64); // cria resolução de configuração do objeto de tela para 128x64
MAX30100 sensor;


char auth[] = "XEXO6hia3ih2ePfk155UBZRgO0TF9HoI";              // Token de autenticação enviado por Blynk
char ssid[] = "Virus";                                        // WiFi nome da rede
char pass[] = "filhosteamo";                                 // WiFi senha

// Conexões: SCL PIN - D1, SDA PIN - D2, INT PIN - D0
PulseOximeter pox;

uint32_t tsLastReport = 0;

// # Plot
int a=0;
int lasta=0;
int lastb=0;
int LastTime=0;
int ThisTime;
bool BPMTiming=false;
bool BeatComplete=false;
int BPM = 0;

// # Limitadores
#define UpperThreshold 520  
#define LowerThreshold 450

// # Bias
int LevelSea = 0;

// Variaveis
const byte RATE_SIZE = 36; // Aumentar para obter mais média. 18 é bom.
byte rates[RATE_SIZE]; //Matriz de freqüência cardíaca
byte rateSpot = 0;
long lastBeat = 0; //Hora em que ocorreu a última batida

int beatsPerMinute; // estava em float, alterado para verificação na mudança da variavel 
int beatAvg;

int CHECK = 0;
byte TestledBrightness = 50; // 0=Off to 255=50mA //0x1F

 
void setup()
{
  Serial.begin(115200); // Configura a taxa de transferência em bits por segundo (baud rate) para transmissão serial
  // Serial.print("Initializing...Display");
  if(!oled.begin(SSD1306_SWITCHCAPVCC, 0x3C)) { // Verificação para endereço do OLED, 0x3C para 128x64
  Serial.println(F("SSD1306 allocation failed"));
  for(;;);
  }
    
  // Limpar buffer..
  oled.clearDisplay();
  oled.setTextSize(3);

    // imprimir o HeartRate e v.TCC determinando o local no oled
    oled.setTextSize(2); oled.setTextColor(WHITE); oled.setCursor(5,0); oled.println("HeartRate");
    oled.setTextSize(1); oled.setCursor(60,20); oled.println("v. TCC");
    oled.display();
    delay(5000);

    // Imprimir no display Aluno "By William H"
    //display.clearDisplay();
    oled.setTextSize(1); oled.setTextColor(WHITE); 
    oled.setCursor(60,40); oled.println("Aluno");
    oled.setCursor(73,50); oled.println("William H");
    oled.display();
    delay(8000);
    oled.clearDisplay(); 
    oled.display();

  Serial.println("OK!");

  // Inicializa sensor
  // Serial.println("Initializing...MAX30100"); retirada no momento
  pinMode(19, OUTPUT); // pin 19 = d0 do nodemcu


  Blynk.begin(auth, ssid, pass); // autenticação com 
  pox.begin();
  
        pox.setOnBeatDetectedCallback(onBeatDetected);
        pox.setIRLedCurrent(MAX30100_LED_CURR_24MA); // Alterar junto com os default no MAX.h para evitar variação.
}

void loop()

{
    pox.update();
    Blynk.run();
    
    int value;
    int valueOrignal= pox.getHeartRate();   
    int valueSPO2 = pox.getSpO2();          

        
    LevelSea = (LevelSea + valueOrignal) / (2) ;
    value = valueOrignal-LevelSea + 500 ;
    Serial.println(valueOrignal); //Envia dados brutos para o plotter

    if(a>127)
    {
      oled.clearDisplay();
      a=0;
      lasta=a;
    }
    
    ThisTime=millis();
    oled.setTextColor(WHITE);
    int b=80-(value/8);
    oled.writeLine(lasta,lastb,a,b,WHITE);
    lastb=b;
    lasta=a;
    
    if(value<LowerThreshold)
    {
      if(BeatComplete)
      {
      BPM=ThisTime-LastTime;
      BPM=int(60/(float(BPM)/1000));
      BPMTiming=false;
      BeatComplete=false;
      }
      if(BPMTiming==false)
      {
        LastTime=millis();
        BPMTiming=true;
      }
    }
    
    if((value>UpperThreshold)&(BPMTiming))
      BeatComplete=true;

    beatsPerMinute = BPM;

    if (beatsPerMinute < 130 && beatsPerMinute >= 40)
    {
      rates[rateSpot++] = (byte)beatsPerMinute; //Armazene esta leitura na matriz
      rateSpot %= RATE_SIZE; 

      //Determina a média das leituras
      beatAvg = 0;
      for (byte x = 0 ; x < RATE_SIZE ; x++)
        beatAvg += rates[x];
        beatAvg /= RATE_SIZE;     
    }
    
    oled.writeFillRect(0,40,128,26,BLACK);
    oled.setTextSize(1.2); 
    oled.setTextSize(1.2); oled.setCursor(30,40);
    oled.print(" BPM:"); oled.print(pox.getHeartRate()); // oled.print("Avg");
    oled.setTextSize(1.2); oled.setCursor(32,50);
    a++;
    oled.setTextSize(1.2);oled.setCursor(30,55);
    oled.print(" SpO2:");oled.print(valueSPO2,1);
    oled.print("%"); 
    oled.display();

    Blynk.virtualWrite(V1, valueOrignal);
    Blynk.virtualWrite(V2, valueSPO2);
}
