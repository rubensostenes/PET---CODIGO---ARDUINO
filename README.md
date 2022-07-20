//Código da esp8866

#include <ESP8266HTTPClient.h>
#include <ESP8266WiFiMulti.h>
#include <SPI.h>
#include <MFRC522.h>
#include <ArduinoJson.h>

#include <Adafruit_NeoPixel.h>
#ifdef __AVR__
 #include <avr/power.h>
#endif

#define ETAPA "Montagem"
#define PIN_LED D8
#define QTD_LEDS 8

#define STAND_BY 0
#define ERRO_WIFI 1
#define ERRO_LEITURA 2
#define ERRO_ENVIO 3

WiFiClient client;
HTTPClient http;

//const char* ssid_1 = "SENAC-Mesh";
//const char* senha_1 =  "09080706";

const char* ssid = "Vivo-Internet-E532";
const char* senha = "6EFC366C";

constexpr uint8_t RST_PIN = D3;
constexpr uint8_t SS_PIN = D4;

MFRC522 rfid(SS_PIN, RST_PIN);
MFRC522::MIFARE_Key key;
Adafruit_NeoPixel pixels(QTD_LEDS, PIN_LED, NEO_GRB + NEO_KHZ800);

String cartao = "";
//byte codigoErro = 0;
byte estado = 0;
String url = "http://joseromildo.pythonanywhere.com/registros";
String tipoDado = "application/json";
int respostaHttp = 0;
bool conectou = false;

void setup()
{
  delay(10);
  #if defined(__AVR_ATtiny85__) && (F_CPU == 16000000)
    clock_prescale_set(clock_div_1);
  #endif
  pixels.begin();
  Serial.begin(115200);
  SPI.begin();
  rfid.PCD_Init();
  do{
    
    conectou = conectaWifi(ssid, senha);
    //conectou = conectaWifi (ssid_2, senha_2);
    
    if(!conectou)
      verificaStatus(QTD_LEDS, ERRO_WIFI);
  }while(!conectou);
 
}
void loop()
{
  if(WiFi.status()== WL_CONNECTED)
  {
    verificaStatus(QTD_LEDS, STAND_BY);
    do{
        cartao = lerCartao();
        if(cartao == "x")
          verificaStatus(QTD_LEDS, ERRO_LEITURA);  
    }while(cartao == "x");
    verificaStatus(QTD_LEDS, STAND_BY);
    do{
      estado = enviaDados(ETAPA, cartao, url, tipoDado);
      verificaStatus(QTD_LEDS, ERRO_ENVIO);
    }while(estado == ERRO_ENVIO);
  }else
    verificaStatus(QTD_LEDS, ERRO_WIFI);
}
//------------------------------------------------------------------------------------
bool conectaWifi(const char* ssid, const char* senha)
{
  bool conectou = false;
  WiFi.begin(ssid, senha);
  Serial.print("\nConectando...");
  byte qtd = 0;
 
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
    qtd++;
    if(qtd > 30){
      break;
    }
  }
  if(WiFi.status() == WL_CONNECTED){
    Serial.println("Conectado com sucesso, meu IP é: ");
    Serial.println(WiFi.localIP());
    conectou = true;
  }
  return conectou;
}
//--------------------------------------------------------------------------------------
byte enviaDados(String etapa, String card, String url, String tipoDado)
{
  String dados = "";
  int retorno = 0, cont = 0;
  DynamicJsonDocument doc(150);
  http.begin(client, url);
  http.addHeader("Content-Type", tipoDado);
  // acrescentar chave de autenciacao no header
  // attp.addHeader("Authentication-key", keyheader );
  doc["etapa"] = etapa;
  doc["cartao"] = card;
  serializeJson(doc, dados);
  do{
    retorno = http.POST(dados);
    delay(500);
    cont++;
  }while(retorno < 0 || cont < 3);
 
  if(retorno > 0){
    String resposta = http.getString();
    Serial.println(retorno);
    Serial.println(resposta);
  }else{
    retorno = 3;
  }
  http.end();
  return retorno;
}
//--------------------------------------------------------------------------------------
String lerCartao()
{
  String card = "x";
  if (rfid.PICC_IsNewCardPresent())
  {
   
    if (rfid.PICC_ReadCardSerial())
    {
      card = "";
      for (byte i = 0; i < rfid.uid.size; i++)
      {
         card.concat(String(rfid.uid.uidByte[i] < 0x10 ? " 0" : " "));
         card.concat(String(rfid.uid.uidByte[i], HEX));
      }
      card.toUpperCase();
      rfid.PICC_HaltA();
      rfid.PCD_StopCrypto1();
    }
    return card;
  }    
 
}
//--------------------------------------------------------------------------------------
void verificaStatus(int qtdLed, byte estado)
{
  byte R, G, B;
  //Serial.println(estado);
     
  switch(estado)
  {
   
    case 1: //Erro de wifi - led azul
      R = 0; G = 0; B = 255;
      acionaLed(qtdLed, R, G, B);
      break;
    case 2: //Erro de leitura - led amarelo
       R = 0; G = 0; B = 255;
       acionaLed(qtdLed, R, G, B);
       break;
    case 3: //Erro de envio - led vermelho
       R = 255; G = 0; B = 0;
       acionaLed(qtdLed, R, G, B);
       break;
   case 4: //Stand by
       R = 255; G = 0; B = 0;
       acionaLed(qtdLed, R, G, B);
       break;
    default:
       R = 0; G = 255; B = 0;
       acionaLed(qtdLed, R, G, B);
       break;
    }
}
//--------------------------------------------------------------------------------------
void acionaLed(byte qtdLed, byte R, byte G, byte B){
  for(int i=0; i < qtdLed; i++)
    pixels.setPixelColor(i, pixels.Color(R,G,B));
  pixels.show();
}
