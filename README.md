#include <WiFi.h>
#include <WiFiClient.h>
#include <WiFiUdp.h>
#include <WebServer.h>
#include <NimBLEDevice.h>

#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

extern "C" {
  #include "lwip/lwip_napt.h"
}

static const char* DEVICE_NAME = "Beacon Navigator";

// ===== AP (captive) =====
static const char* AP_SSID = "Beacon Navigator";
static const char* AP_PASS = nullptr; // rede aberta

// ===== Plataforma =====
static const char* PLATFORM_URL = "https://beaconnavigator.marinhoai.com.br";

// ===== (Opcional) STA/NAT =====
// Se você não quiser tentar STA/NAT agora, deixe STA_ENABLE = false.
// Se quiser tentar depois, coloque SSID/SENHA e mude para true.
static const bool  STA_ENABLE = false;
static const char* STA_SSID   = "SEU_WIFI_DA_INTERNET";
static const char* STA_PASS   = "SUA_SENHA_DO_WIFI";

#define NAPT_AP_NETIF_NO 1

static WebServer g_web(80);
static WiFiUDP   g_dnsUdp;

#define SCREEN_WIDTH  64
#define SCREEN_HEIGHT 48
static Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
static bool g_displayOk = false;

static bool g_natEnabled = false;

// ---------------- OLED ----------------
static bool tryOledOnBus(uint8_t sda, uint8_t scl, uint8_t addr) {
  Wire.end();
  delay(10);
  Wire.begin(sda, scl);
  Wire.setClock(400000);
  delay(20);
  return display.begin(SSD1306_SWITCHCAPVCC, addr);
}

static void oledShowText2(const char* line1, const char* line2) {
  if (!g_displayOk) return;

  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  int16_t x1, y1;
  uint16_t w1, h1, w2, h2;

  display.getTextBounds(line1, 0, 0, &x1, &y1, &w1, &h1);
  display.getTextBounds(line2, 0, 0, &x1, &y1, &w2, &h2);

  int16_t gap = 4;
  int16_t totalH = (int16_t)h1 + gap + (int16_t)h2;
  int16_t startY = (SCREEN_HEIGHT - totalH) / 2;

  int16_t xLine1 = (SCREEN_WIDTH - (int16_t)w1) / 2;
  int16_t xLine2 = (SCREEN_WIDTH - (int16_t)w2) / 2;

  display.setCursor(xLine1, startY);
  display.print(line1);

  display.setCursor(xLine2, startY + h1 + gap);
  display.print(line2);

  display.display();
}

static void startOled() {
  g_displayOk = false;

  const uint8_t pins[][2] = { {21,22},{4,5},{4,15},{16,17} };
  const uint8_t addrs[]   = { 0x3C, 0x3D };

  for (uint8_t i = 0; i < (sizeof(pins) / sizeof(pins[0])); i++) {
    for (uint8_t j = 0; j < (sizeof(addrs) / sizeof(addrs[0])); j++) {
      if (tryOledOnBus(pins[i][0], pins[i][1], addrs[j])) {
        g_displayOk = true;
        oledShowText2("Beacon", "Navigator");
        return;
      }
    }
  }
}

// ---------------- Captive portal page ----------------
static void portalPage() {
  // NAT/STA são opcionais. Mesmo sem NAT, o portal existe.
  bool staOk = (WiFi.status() == WL_CONNECTED);
  bool natOk = g_natEnabled;

  String html;
  html.reserve(2400);

  html += F("<!doctype html><html><head><meta charset='utf-8'>");
  html += F("<meta name='viewport' content='width=device-width,initial-scale=1'>");
  html += F("<title>Beacon Navigator</title>");
  html += F("<style>");
  html += F("body{font-family:Arial;display:flex;min-height:100vh;align-items:center;justify-content:center;background:#0b1220;color:#fff;margin:0}");
  html += F(".card{width:min(560px,92vw);background:#121b2f;border-radius:14px;padding:18px;box-shadow:0 8px 24px rgba(0,0,0,.35)}");
  html += F(".tag{display:inline-block;padding:4px 10px;border-radius:999px;background:#0b2447;margin-right:8px;font-size:12px}");
  html += F(".muted{opacity:.9;line-height:1.4}");
  html += F("a.btn{display:block;text-align:center;background:#3b82f6;color:#fff;text-decoration:none;padding:14px;border-radius:10px;font-weight:700;margin-top:12px}");
  html += F("button.btn2{width:100%;background:#1f2a44;color:#fff;border:0;padding:12px;border-radius:10px;font-weight:700;margin-top:10px}");
  html += F(".warn{margin-top:12px;padding:12px;border-radius:10px;background:#2a1b1b;color:#ffd2d2;font-size:13px}");
  html += F("code{background:#0a1020;padding:2px 6px;border-radius:6px}");
  html += F("</style></head><body><div class='card'>");

  html += F("<h2>Beacon Navigator</h2>");
  html += F("<p class='muted'>Conectado &agrave; rede <b>Beacon Navigator</b>.</p>");

  html += F("<p><span class='tag'>AP</span>OK</p>");
  if (STA_ENABLE) {
    html += F("<p><span class='tag'>STA</span>");
    html += (staOk ? "Conectado" : "Desconectado");
    html += F("</p>");
    html += F("<p><span class='tag'>NAT</span>");
    html += (natOk ? "Ativo" : "Inativo");
    html += F("</p>");
  } else {
    html += F("<p><span class='tag'>Internet</span>Via celular (recomendado)</p>");
  }

  if (!natOk && STA_ENABLE) {
    html += F("<div class='warn'>Sem NAT, seu iPhone pode tentar usar esta Wi-Fi sem internet. Se a plataforma n&atilde;o abrir, desligue o Wi-Fi ou ative <b>Usar Dados M&oacute;veis</b> nesta rede.</div>");
  }

  html += F("<a class='btn' href='");
  html += PLATFORM_URL;
  html += F("'>Abrir plataforma</a>");

  // Botão de copiar link (ajuda no iOS quando o captive browser limita)
  html += F("<button class='btn2' onclick='copyLink()'>Copiar link</button>");
  html += F("<p class='muted' style='margin-top:10px'>Link: <code>");
  html += PLATFORM_URL;
  html += F("</code></p>");

  html += F("<script>");
  html += F("function copyLink(){");
  html += F("const t='");
  html += PLATFORM_URL;
  html += F("';");
  html += F("if(navigator.clipboard&&navigator.clipboard.writeText){navigator.clipboard.writeText(t).then(()=>alert('Link copiado!'));}");
  html += F("else{prompt('Copie o link:',t);}");
  html += F("}");
  html += F("</script>");

  html += F("</div></body></html>");

  g_web.sendHeader("Cache-Control", "no-store");
  g_web.send(200, "text/html", html);
}

static void startPortalHttp() {
  g_web.on("/", HTTP_GET, portalPage);
  g_web.on("/generate_204", HTTP_GET, portalPage);
  g_web.on("/gen_204", HTTP_GET, portalPage);
  g_web.on("/hotspot-detect.html", HTTP_GET, portalPage);
  g_web.on("/connecttest.txt", HTTP_GET, portalPage);
  g_web.on("/ncsi.txt", HTTP_GET, portalPage);
  g_web.onNotFound(portalPage);
  g_web.begin();
}

// ---------------- DNS captive ----------------
static bool isCaptiveDomain(const String& host) {
  String h = host;
  h.toLowerCase();
  if (h == "captive.apple.com") return true;
  if (h.endsWith(".apple.com")) return true;
  if (h == "connectivitycheck.gstatic.com") return true;
  if (h == "clients3.google.com") return true;
  if (h == "www.msftconnecttest.com") return true;
  if (h == "msftconnecttest.com") return true;
  return false;
}

static int readDnsName(const uint8_t* buf, int len, int offset, String& outName) {
  outName = "";
  int i = offset;
  while (i < len) {
    uint8_t l = buf[i++];
    if (l == 0) break;
    if (l & 0xC0) return -1;
    if (i + l > len) return -1;
    if (outName.length()) outName += ".";
    for (uint8_t k = 0; k < l; k++) outName += (char)buf[i++];
  }
  return i;
}

static void dnsSendARecord(const uint8_t* req, int reqLen, const IPAddress& ip) {
  uint8_t resp[512];
  if (reqLen < 12) return;
  if (reqLen > (int)sizeof(resp) - 16) return;

  resp[0] = req[0]; resp[1] = req[1];
  resp[2] = 0x81; resp[3] = 0x80;
  resp[4] = 0x00; resp[5] = 0x01;
  resp[6] = 0x00; resp[7] = 0x01;
  resp[8] = 0x00; resp[9] = 0x00;
  resp[10]= 0x00; resp[11]= 0x00;

  int qEnd = 12;
  while (qEnd < reqLen && req[qEnd] != 0) qEnd++;
  qEnd++;
  if (qEnd + 4 > reqLen) return;

  int qLen = (qEnd + 4) - 12;
  memcpy(resp + 12, req + 12, qLen);
  int pos = 12 + qLen;

  resp[pos++] = 0xC0; resp[pos++] = 0x0C;
  resp[pos++] = 0x00; resp[pos++] = 0x01;
  resp[pos++] = 0x00; resp[pos++] = 0x01;
  resp[pos++] = 0x00; resp[pos++] = 0x00; resp[pos++] = 0x00; resp[pos++] = 0x1E;
  resp[pos++] = 0x00; resp[pos++] = 0x04;
  resp[pos++] = ip[0];
  resp[pos++] = ip[1];
  resp[pos++] = ip[2];
  resp[pos++] = ip[3];

  g_dnsUdp.beginPacket(g_dnsUdp.remoteIP(), g_dnsUdp.remotePort());
  g_dnsUdp.write(resp, pos);
  g_dnsUdp.endPacket();
}

static void dnsSendNxDomain(const uint8_t* req, int reqLen) {
  uint8_t resp[128];
  if (reqLen < 12) return;

  resp[0] = req[0]; resp[1] = req[1];
  resp[2] = 0x81; resp[3] = 0x83;
  resp[4] = 0x00; resp[5] = 0x01;
  resp[6] = 0x00; resp[7] = 0x00;
  resp[8] = 0x00; resp[9] = 0x00;
  resp[10]= 0x00; resp[11]= 0x00;

  int copyLen = min(reqLen, (int)sizeof(resp));
  memcpy(resp + 12, req + 12, copyLen - 12);

  g_dnsUdp.beginPacket(g_dnsUdp.remoteIP(), g_dnsUdp.remotePort());
  g_dnsUdp.write(resp, copyLen);
  g_dnsUdp.endPacket();
}

static void dnsProcess() {
  int packetSize = g_dnsUdp.parsePacket();
  if (packetSize <= 0) return;

  uint8_t buf[512];
  int len = g_dnsUdp.read(buf, sizeof(buf));
  if (len < 12) return;

  String qname;
  int qnameEnd = readDnsName(buf, len, 12, qname);
  if (qnameEnd < 0) return;
  if (qnameEnd + 4 > len) return;

  uint16_t qtype  = (buf[qnameEnd] << 8) | buf[qnameEnd + 1];
  uint16_t qclass = (buf[qnameEnd + 2] << 8) | buf[qnameEnd + 3];

  if (qtype != 1 || qclass != 1) {
    dnsSendNxDomain(buf, len);
    return;
  }

  IPAddress apIP = WiFi.softAPIP();

  // Sempre força captive para o AP (mais estável, mesmo sem NAT)
  if (!g_natEnabled || isCaptiveDomain(qname)) {
    dnsSendARecord(buf, len, apIP);
    return;
  }

  // Se NAT existir, resolve normalmente (opcional)
  IPAddress resolved;
  if (WiFi.hostByName(qname.c_str(), resolved)) dnsSendARecord(buf, len, resolved);
  else dnsSendNxDomain(buf, len);
}

// ---------------- AP + (optional) STA/NAT ----------------
static void startAPNow() {
  WiFi.persistent(false);
  WiFi.disconnect(true, true);
  delay(200);

  WiFi.mode(WIFI_AP_STA);
  WiFi.setSleep(false);

  IPAddress apIP(192, 168, 4, 1);
  IPAddress gw(192, 168, 4, 1);
  IPAddress mask(255, 255, 255, 0);
  WiFi.softAPConfig(apIP, gw, mask);

  bool apOk = WiFi.softAP(AP_SSID, AP_PASS, 1, 0, 4);
  Serial.printf("[AP] start: %s | SSID=%s | IP=%s\n",
                apOk ? "OK" : "FAIL",
                AP_SSID,
                WiFi.softAPIP().toString().c_str());

  g_dnsUdp.begin(53);
  startPortalHttp();
}

static void startSTAIfEnabled() {
  if (!STA_ENABLE) return;
  WiFi.setAutoReconnect(true);
  WiFi.begin(STA_SSID, STA_PASS);
  Serial.printf("[STA] connecting to %s...\n", STA_SSID);
}

static void tryEnableNatIfReady() {
  if (!STA_ENABLE) return;
  if (g_natEnabled) return;
  if (WiFi.status() != WL_CONNECTED) return;

  uint8_t ch = WiFi.channel();

  WiFi.softAPdisconnect(false);
  delay(200);
  WiFi.softAP(AP_SSID, AP_PASS, ch, 0, 4);
  delay(200);

  ip_napt_enable_no(NAPT_AP_NETIF_NO, 1);
  g_natEnabled = true;

  Serial.printf("[NAT] enabled | STA IP=%s | AP IP=%s\n",
                WiFi.localIP().toString().c_str(),
                WiFi.softAPIP().toString().c_str());
}

void setup() {
  Serial.begin(115200);
  delay(200);

  startOled();
  oledShowText2("Beacon", "Navigator");

  startAPNow();
  startSTAIfEnabled();
}

void loop() {
  delay(2);
  dnsProcess();
  g_web.handleClient();
  tryEnableNatIfReady();
}
