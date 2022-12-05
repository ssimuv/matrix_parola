#include <ESP8266WiFi.h>
#include <WiFiUdp.h>
#include <MD_Parola.h>
#include <MD_MAX72xx.h>
#include <SPI.h>
#include <NTPClient.h>

WiFiUDP u;
NTPClient n(u, "1.vn.pool.ntp.org", 7 * 3600);

#include "DHT.h"
#define DHTPIN 4

#define DHTTYPE DHT11   // DHT 11

DHT dht(DHTPIN, DHTTYPE);

float h;
float t;
char text;
String formatTime;

// Turn on debug statements to the serial output
#define  DEBUG  0

#if  DEBUG
#define PRINT(s, x) { Serial.print(F(s)); Serial.print(x); }
#define PRINTS(x) Serial.print(F(x))
#define PRINTX(x) Serial.println(x, HEX)
#else
#define PRINT(s, x)
#define PRINTS(x)
#define PRINTX(x)
#endif

// Define the number of devices we have in the chain and the hardware interface
// NOTE: These pin numbers are for ESO8266 hardware SPI and will probably not
// work with your hardware and may need to be adapted
#define HARDWARE_TYPE MD_MAX72XX::FC16_HW
#define MAX_DEVICES 4

#define CLK_PIN   14 // or SCK
#define DATA_PIN  13 // or MOSI
#define CS_PIN    12 // or SS

// HARDWARE SPI
MD_Parola P = MD_Parola(HARDWARE_TYPE, CS_PIN, MAX_DEVICES);
// SOFTWARE SPI
//MD_Parola P = MD_Parola(HARDWARE_TYPE, DATA_PIN, CLK_PIN, CS_PIN, MAX_DEVICES);

// WiFi login parameters - network name and password
const char* ssid = "LIEU";
const char* password = "lieu03091971";

// WiFi Server object and parameters
WiFiServer server(80);

// Scrolling parameters
uint8_t frameDelay = 25;  // default frame delay value
textEffect_t  scrollEffect = PA_SCROLL_LEFT;

// Global message buffers shared by Wifi and Scrolling functions
#define BUF_SIZE  512
char IPconnected[BUF_SIZE];
char curMessage[BUF_SIZE];
char newMessage[BUF_SIZE];
bool newMessageAvailable = false;
bool flag = false;

const char WebResponse[] = "HTTP/1.1 200 OK\nContent-Type: text/html\n\n";

const char WebPage[] =
  "<!DOCTYPE html>" \
  "<html>" \
  "<head>" \
  "<title>MajicDesigns Test Page</title>" \

  "<script>" \
  "strLine = \"\";" \

  "function SendData()" \
  "{" \
  "  nocache = \"/&nocache=\" + Math.random() * 1000000;" \
  "  var request = new XMLHttpRequest();" \
  "  strLine = \"&MSG=\" + document.getElementById(\"data_form\").Message.value;" \
  "  strLine = strLine + \"/&SD=\" + document.getElementById(\"data_form\").ScrollType.value;" \
  "  strLine = strLine + \"/&I=\" + document.getElementById(\"data_form\").Invert.value;" \
  "  strLine = strLine + \"/&SP=\" + document.getElementById(\"data_form\").Speed.value;" \
  "  request.open(\"GET\", strLine + nocache, false);" \
  "  request.send(null);" \
  "}" \
  "</script>" \
  "</head>" \

  "<body>" \
  "<p><b>MD_Parola set message</b></p>" \

  "<form id=\"data_form\" name=\"frmText\">" \
  "<label>Message:<br><input type=\"text\" name=\"Message\" maxlength=\"255\"></label>" \
  "<br><br>" \
  "<input type = \"radio\" name = \"Invert\" value = \"0\" checked> Normal" \
  "<input type = \"radio\" name = \"Invert\" value = \"1\"> Inverse" \
  "<br>" \
  "<input type = \"radio\" name = \"ScrollType\" value = \"L\" checked> Left Scroll" \
  "<input type = \"radio\" name = \"ScrollType\" value = \"R\"> Right Scroll" \
  "<br><br>" \
  "<label>Speed:<br>Fast<input type=\"range\" name=\"Speed\"min=\"10\" max=\"200\">Slow"\
  "<br>" \
  "</form>" \
  "<br>" \
  "<input type=\"submit\" value=\"Send Data\" onclick=\"SendData()\">" \
  "</body>" \
  "</html>";

const char *err2Str(wl_status_t code)
{
  switch (code)
  {
    case WL_IDLE_STATUS:    return ("IDLE");           break; // WiFi is in process of changing between statuses
    case WL_NO_SSID_AVAIL:  return ("NO_SSID_AVAIL");  break; // case configured SSID cannot be reached
    case WL_CONNECTED:      return ("CONNECTED");      break; // successful connection is established
    case WL_CONNECT_FAILED: return ("CONNECT_FAILED"); break; // password is incorrect
    case WL_DISCONNECTED:   return ("CONNECT_FAILED"); break; // module is not configured in station mode
    default: return ("??");
  }
}

uint8_t htoi(char c)
{
  c = toupper(c);
  if ((c >= '0') && (c <= '9')) return (c - '0');
  if ((c >= 'A') && (c <= 'F')) return (c - 'A' + 0xa);
  return (0);
}

void getData(char *szMesg, uint16_t len)
// Message may contain data for:
// New text (/&MSG=)
// Scroll direction (/&SD=)
// Invert (/&I=)
// Speed (/&SP=)
{
  char *pStart, *pEnd;      // pointer to start and end of text

  // check text message
  pStart = strstr(szMesg, "/&MSG=");
  if (pStart != NULL)
  {
    flag = true;
    char *psz = newMessage;

    pStart += 6;  // skip to start of data
    pEnd = strstr(pStart, "/&");

    if (pEnd != NULL)
    {
      while (pStart != pEnd)
      {
        if ((*pStart == '%') && isdigit(*(pStart + 1)))
        {
          // replace %xx hex code with the ASCII character
          char c = 0;
          pStart++;
          c += (htoi(*pStart++) << 4);
          c += htoi(*pStart++);
          *psz++ = c;
        }
        else
          *psz++ = *pStart++;
      }

      *psz = '\0'; // terminate the string
      newMessageAvailable = (strlen(newMessage) != 0);
      PRINT("\nNew Msg: ", newMessage);
    }
  }

  // check scroll direction
  pStart = strstr(szMesg, "/&SD=");
  if (pStart != NULL)
  {
    flag = true;
    pStart += 10;  // skip to start of data

    PRINT("\nScroll direction: ", *pStart);
    scrollEffect = (*pStart == 'R' ? PA_SCROLL_RIGHT : PA_SCROLL_LEFT);
    P.setTextEffect(scrollEffect, scrollEffect);
    P.displayReset();
  }

  // check invert
  pStart = strstr(szMesg, "/&I=");
  if (pStart != NULL)
  {
    flag = true;
    pStart += 4;  // skip to start of data

    PRINT("\nInvert mode: ", *pStart);
    P.setInvert(*pStart == '1');
  }

  // check speed
  pStart = strstr(szMesg, "/&SP=");
  if (pStart != NULL)
  {
    flag = true;
    pStart += 5;  // skip to start of data

    int16_t speed = atoi(pStart);
    PRINT("\nSpeed: ", P.getSpeed());
    P.setSpeed(speed);
    frameDelay = speed;
  }
}

void handleWiFi(void)
{
  static enum { S_IDLE, S_WAIT_CONN, S_READ, S_EXTRACT, S_RESPONSE, S_DISCONN } state = S_IDLE;
  static char szBuf[1024];
  static uint16_t idxBuf = 0;
  static WiFiClient client;
  static uint32_t timeStart;

  switch (state)
  {
    case S_IDLE:   // initialise
      PRINTS("\nS_IDLE");
      idxBuf = 0;
      state = S_WAIT_CONN;
      break;

    case S_WAIT_CONN:   // waiting for connection
      {
        client = server.available();
        if (!client) break;
        if (!client.connected()) break;

#if DEBUG
        char szTxt[20];
        sprintf(szTxt, "%03d:%03d:%03d:%03d", client.remoteIP()[0], client.remoteIP()[1], client.remoteIP()[2], client.remoteIP()[3]);
        PRINT("\nNew client @ ", szTxt);
#endif

        timeStart = millis();
        state = S_READ;
      }
      break;

    case S_READ: // get the first line of data
      PRINTS("\nS_READ ");

      while (client.available())
      {
        char c = client.read();

        if ((c == '\r') || (c == '\n'))
        {
          szBuf[idxBuf] = '\0';
          client.flush();
          PRINT("\nRecv: ", szBuf);
          state = S_EXTRACT;
        }
        else
          szBuf[idxBuf++] = (char)c;
      }
      if (millis() - timeStart > 1000)
      {
        PRINTS("\nWait timeout");
        state = S_DISCONN;
      }
      break;

    case S_EXTRACT: // extract data
      PRINTS("\nS_EXTRACT");
      // Extract the string from the message if there is one
      getData(szBuf, BUF_SIZE);
      state = S_RESPONSE;
      break;

    case S_RESPONSE: // send the response to the client
      PRINTS("\nS_RESPONSE");
      // Return the response to the client (web page)
      client.print(WebResponse);
      client.print(WebPage);
      state = S_DISCONN;
      break;

    case S_DISCONN: // disconnect client
      PRINTS("\nS_DISCONN");
      client.flush();
      client.stop();
      state = S_IDLE;
      break;

    default:  state = S_IDLE;
  }
}

float sinVal;
int toneVal;
unsigned long tepTimer;

void setup()
{
  pinMode(5, OUTPUT);
  Serial.begin(57600);
  dht.begin();
  PRINTS("\n[MD_Parola WiFi Message Display]\nType a message for the scrolling display from your internet browser");

  P.begin();
  P.displayClear();
  P.displaySuspend(false);

  P.displayScroll(curMessage, PA_LEFT, scrollEffect, frameDelay);

  IPconnected[0] = curMessage[0] = newMessage[0] = '\0';

  // Connect to and initialise WiFi network
  PRINT("\nConnecting to ", ssid);

  WiFi.begin(ssid, password);

  while (WiFi.status() != WL_CONNECTED)
  {
    PRINT("\n", err2Str(WiFi.status()));
    delay(500);
  }

  // Start the server
  server.begin();
  PRINTS("\nServer started");
  //printsensor_time();
}

void loop()
{
  Serial.print("\nWiFi connected: ");
  sprintf(IPconnected, "%03d:%03d:%03d:%03d", WiFi.localIP()[0], WiFi.localIP()[1], WiFi.localIP()[2], WiFi.localIP()[3]);
  Serial.println(IPconnected);
  if (flag == false) {
    printsensor_time();
    animate();
    delay(110);
  }
  handleWiFi();
  if (flag == true) {
    animate();
    sensorThings();
  }
}
void sensorThings() {
  h = dht.readHumidity();
  t = dht.readTemperature();

  // Check if any reads failed and exit early (to try again).
  if (isnan(h) || isnan(t)) {
    Serial.println("Failed to read from DHT sensor!");
    return;
  }
  if (t > 29 ) {
    for (int x = 0; x < 180; x++) {
      sinVal = (sin(x * (3.1412 / 180)));
      toneVal = 2000 + (int(sinVal * 1000));
      tone(5, toneVal);
      delay(2);
    }
  }
  else
    noTone(5);
  Serial.print("Humidity: ");
  Serial.print(h);
  Serial.print(" %\t");
  Serial.print("Temperature: ");
  Serial.print(t);
  Serial.print(" *C ");
}
void Time() {
  n.update();
  formatTime = n.getFormattedTime();
  Serial.println(formatTime);
}
void printsensor_time() {
  sensorThings();
  Time();
  char data[120];
  String bufferr;
  bufferr = "T:" + (String)t + "oC" + " H:" + (String)h + "%RH " + "Time:" + formatTime + "              ";
  bufferr.toCharArray(data, 120);
  strcpy(curMessage, data);
}
void animate() {
  if (P.displayAnimate())
  {
    if (newMessageAvailable)
    {
      strcpy(curMessage, newMessage);
      newMessageAvailable = false;
    }
    P.displayReset();
  }
}
