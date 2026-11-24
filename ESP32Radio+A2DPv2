/* -------------------------------------------------
   ESP32 Internet Radio + Bluetooth A2DP + Web UI
   Proyek asal: Tech Talkies YouTube Channel
   https://www.youtube.com/@techtalkies1
   -------------------------------------------------
   PERBAIKAN A2DP (baru di versi ini):
   - Tidak ada double init I2S (toggle BT_MANUAL_I2S_BEGIN)
   - Sample rate tidak dipaksa 44.1k -> ikut negosiasi HP (44.1/48k)
   - AVRCP event mask eksplisit -> status PLAY/PAUSE update
   - set_auto_reconnect(true)
   - Flag "bt" persisten + pengaman reset PANIC
   - Mute hardware XSMT (GPIO 14) -> tidak ada "pop" saat boot/switch
   PERBAIKAN RADIO (baru di versi ini):
   - Deteksi stream stall (diam tapi TCP masih tersambung)
   - Retry tak terbatas, backoff plafon 60 detik
   - Reset hitungan retry setelah 60 detik sehat
   Sudah ada sebelumnya:
   - Filter ArduinoJson root ARRAY (filter[0])
   - Wire 400kHz + throttle OLED, pumpAudio(), buffer PSRAM
   - Login Web UI (cookie HttpOnly, SHA-256 bersalt, lockout)
   - Mutex metadata BT, debounce rotary, mDNS, cek ruang LittleFS
   ------------------------------------------------- */

#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <Audio.h>
#include <Adafruit_GFX.h>
#include <Wire.h>
#include <Adafruit_SSD1306.h>
#include <ESPmDNS.h>
#include <freertos/FreeRTOS.h>
#include <freertos/task.h>
#include <freertos/queue.h>
#include <freertos/semphr.h>
#include <Preferences.h>
#include <LittleFS.h>
#include <BluetoothA2DPSink.h>
#include <esp_bt.h>
#include <esp_system.h>
#include <mbedtls/md.h>
#include <math.h>
#include <string.h>
#include <esp_heap_caps.h>

#if __has_include(<esp_idf_version.h>)
#include <esp_idf_version.h>
#endif
#if defined(ESP_IDF_VERSION) && (ESP_IDF_VERSION >= ESP_IDF_VERSION_VAL(5, 1, 0))
#include <ESP_I2S.h>
#define RADIO_USE_NEW_I2S_API 1
#else
#define RADIO_USE_NEW_I2S_API 0
#endif

/* ---------------- PIN ---------------- */
#define I2S_DOUT 32
#define I2S_BCLK 25
#define I2S_LRCK 33
#define OLED_SDA 21
#define OLED_SCL 22
#define OLED_WIDTH 128
#define OLED_HEIGHT 64
#define OLED_ADDR 0x3C
#define ENC_CLK 26
#define ENC_DT 27
#define ENC_SW 13

/* XSMT PCM5102A -> GPIO 14 (pull-down 10k). LOW = mute, HIGH = aktif.
   Kalau tidak dipasang, set DAC_MUTE_ENABLED 0. */
#define DAC_MUTE_ENABLED 1
#define DAC_MUTE_PIN 14

#define UI_WHITE 1
#define UI_BLACK 0

/* ---------------- KONFIG ---------------- */
#define RB_HOST "http://all.api.radio-browser.info"
#define RB_RESULT_LIMIT 150
#define WEB_PAGE_SIZE 150
#define M3U_STATION_LIMIT 1500
#define STATION_STORAGE_LIMIT M3U_STATION_LIMIT
#define M3U_MAX_FILE_SIZE 180000
#define M3U_FILE_PATH "/radio.m3u"
#define M3U_TEMP_PATH "/radio_upload.tmp"
#define M3U_MERGE_PATH "/radio_merge.tmp"
#define M3U_BACKUP_PATH "/radio_backup.m3u"
#define CUSTOM_RADIO_MAX 30
#define VOL_MAX 21
#define INITIAL_VOLUME 18
#define BT_NAME "Radio ESP32"
#define VLONG_MS 3000
#define OLED_MIN_FRAME_MS 60
#define STREAM_TIMEOUT_MS 20000
#define STREAM_STALL_MS 15000

/* ---- TUNING A2DP: ubah di sini kalau perlu ---- */
#define BT_MANUAL_I2S_BEGIN 0   // 0 = library yang init I2S (disarankan)
                                // 1 = kita init sendiri (kalau suara hilang total)
#define BT_MANUAL_RATE 48000    // dipakai hanya jika BT_MANUAL_I2S_BEGIN = 1
#define BT_EXTRA_API 1          // 0 = matikan set_avrc_rn_event_mask & auto_reconnect
                                //     (kalau versi library tidak punya fungsi itu)
#define AUDIO_HAS_ISRUNNING 1   // 0 = kalau audio.isRunning() tidak ada

/* ---------------- KEAMANAN WEB ---------------- */
#define WEB_AUTH_ENABLED 1
#define WEB_USER_DEFAULT "admin"
#define WEB_SESSION_MAX 4
#define WEB_SESSION_TTL_MS (30UL * 60UL * 1000UL)
#define WEB_LOGIN_MAX_FAIL 5
#define WEB_LOGIN_LOCK_MS (60UL * 1000UL)
#define WEB_PASS_MIN 8

/* ---------------- JSON di PSRAM ---------------- */
#if defined(ARDUINOJSON_VERSION_MAJOR) && (ARDUINOJSON_VERSION_MAJOR >= 7)
#define BIG_JSON(name, cap) JsonDocument name
#else
struct PsramAlloc {
  void* allocate(size_t n) {
    void* p = heap_caps_malloc(n, MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT);
    return p ? p : malloc(n);
  }
  void deallocate(void* p) { free(p); }
  void* reallocate(void* p, size_t n) { return realloc(p, n); }
};
typedef BasicJsonDocument<PsramAlloc> BigJson;
#define BIG_JSON(name, cap) BigJson name(cap)
#endif

/* ---------------- TABEL ---------------- */
struct CountryEntry { const char* code; const char* name; };
static const CountryEntry COUNTRIES[] = {
  {"all", "All"}, {"US", "USA"}, {"IN", "India"}, {"GB", "UK"}, {"DE", "Germany"},
  {"FR", "France"}, {"JP", "Japan"}, {"CA", "Canada"}, {"AU", "Australia"},
  {"IT", "Italy"}, {"ES", "Spain"}, {"BR", "Brazil"}, {"MX", "Mexico"},
  {"NL", "Netherlands"}, {"SE", "Sweden"}, {"NO", "Norway"}, {"ZA", "South Africa"},
  {"SG", "Singapore"}, {"ID", "Indonesia"}, {"AE", "UAE"}, {"ZZ", "Belum diketahui"},
};
static const int COUNTRY_COUNT = sizeof(COUNTRIES) / sizeof(COUNTRIES[0]);

struct GenreEntry { const char* tag; const char* label; };
static const GenreEntry GENRES[] = {
  {"all", "All"}, {"music", "Music"}, {"news", "News"},
  {"jazz", "Jazz"}, {"classical", "Classical"}, {"rock", "Rock"},
};
static const int GENRE_COUNT = sizeof(GENRES) / sizeof(GENRES[0]);

/* ---------------- OBJEK ---------------- */
Audio audio;
Adafruit_SSD1306 display(OLED_WIDTH, OLED_HEIGHT, &Wire, -1);
GFXcanvas1 canvas(OLED_WIDTH, OLED_HEIGHT);
QueueHandle_t encQueue;
SemaphoreHandle_t btMetaMutex;
Preferences prefs;
WebServer server(80);

#if RADIO_USE_NEW_I2S_API
I2SClass btI2S;
BluetoothA2DPSink a2dp_sink(btI2S);
#else
BluetoothA2DPSink a2dp_sink;
#endif

/* ---------------- STATE ---------------- */
bool oledOK = false;
bool apMode = false;
bool webStarted = false;
bool restartRequested = false;
String wifiIP = "No WiFi";
String wifiSsid = "";
static const char* CONFIG_AP_SSID = "Radio-ESP32-Setup";
static const char* CONFIG_AP_PASS = "12345678";

bool btMode = false;
bool btConnected = false;
bool btPlaying = false;
int btVolume = 64;
char btTitle[48] = "";
char btArtist[40] = "";
char btAlbum[40] = "";
bool btVolumeDirty = false;
uint32_t btVolumeChangedAt = 0;

enum EncEvent { EV_CW, EV_CCW, EV_PRESS, EV_LONG, EV_VLONG };
enum UiMode { MODE_NORMAL, MODE_BROWSE, MODE_EDIT };
enum FocusItem { F_NOWPLAYING, F_COUNTRY, F_TYPE };

struct Station { char name[65]; char url[513]; uint16_t bitrate; char country[4]; };
Station* stations = nullptr;
int stationCount = 0;
String stationSource = "radiobrowser";

File m3uUploadFile;
bool filesystemReady = false;
size_t m3uUploadSize = 0;
bool m3uUploadOK = false;

int currentStation = -1;
int stationPage = 0;
int focusIndex = 0;
UiMode uiMode = MODE_NORMAL;
bool uiDirty = true;
uint32_t lastOledMs = 0;
String streamTitle = "Loading...";
int previewStation = 0;
String previewTag = "all";
String previewCountry = "ID";
String selectedGenre = "all";
String selectedCountry = "ID";
String searchName = "";
bool muted = false;
int lastVol = INITIAL_VOLUME;
bool volDirty = false;
uint32_t volChangedAt = 0;

volatile bool radioStreamActive = false;
String radioStreamState = "stopped";
String lastError = "";
bool userPaused = false;
String activeStreamUrl = "";
uint32_t streamStartedAt = 0;
uint32_t streamRetryAt = 0;
uint8_t streamRetryCount = 0;
uint32_t lastAudioAliveMs = 0;
uint32_t wifiRetryAt = 0;
uint32_t ipPollAt = 0;
uint32_t browseLastAction = 0;
volatile uint32_t holdStartMs = 0;
volatile bool buttonHolding = false;

/* ---------------- STATE AUTH ---------------- */
struct WebSession { char token[33]; uint32_t lastSeen; bool used; };
WebSession webSessions[WEB_SESSION_MAX];
String webUser = WEB_USER_DEFAULT;
String webSalt = "";
String webHash = "";
String webNewPass = "";
bool webCredJustCreated = false;
uint8_t webLoginFail = 0;
uint32_t webLockUntil = 0;

/* ---------------- DEKLARASI AWAL ---------------- */
bool fetchStations(String tag, int offset = 0, String name = "");
void playStation(int i);
void startStream(const String& url);
void renderOLED();
bool sessionValid();

static inline void pumpAudio() { if (!btMode) audio.loop(); }

static inline void dacMute(bool m) {
#if DAC_MUTE_ENABLED
  digitalWrite(DAC_MUTE_PIN, m ? LOW : HIGH);
#else
  (void)m;
#endif
}

static void sendJsonOk(bool ok, const char* msg, int code = 200) {
  DynamicJsonDocument d(384);
  d["ok"] = ok;
  d["message"] = msg;
  String s; serializeJson(d, s);
  server.send(code, "application/json", s);
}

/* ================= AUTENTIKASI ================= */
static String randomHex(uint8_t bytes) {
  static const char* hx = "0123456789abcdef";
  String s; s.reserve(bytes * 2);
  for (uint8_t i = 0; i < bytes; i++) {
    uint8_t b = (uint8_t)(esp_random() & 0xFF);
    s += hx[b >> 4]; s += hx[b & 0x0F];
  }
  return s;
}

static String randomPassword(uint8_t len) {
  static const char* al = "abcdefghijkmnpqrstuvwxyz23456789ABCDEFGHJKLMNPQRSTUVWXYZ";
  size_t n = strlen(al);
  String s; s.reserve(len);
  for (uint8_t i = 0; i < len; i++) s += al[esp_random() % n];
  return s;
}

static String sha256Hex(const String& input) {
  uint8_t out[32];
  const mbedtls_md_info_t* info = mbedtls_md_info_from_type(MBEDTLS_MD_SHA256);
  if (!info) return "";
  mbedtls_md_context_t ctx;
  mbedtls_md_init(&ctx);
  if (mbedtls_md_setup(&ctx, info, 0) != 0) { mbedtls_md_free(&ctx); return ""; }
  mbedtls_md_starts(&ctx);
  mbedtls_md_update(&ctx, (const unsigned char*)input.c_str(), input.length());
  mbedtls_md_finish(&ctx, out);
  mbedtls_md_free(&ctx);
  static const char* hx = "0123456789abcdef";
  String s; s.reserve(64);
  for (int i = 0; i < 32; i++) { s += hx[out[i] >> 4]; s += hx[out[i] & 0x0F]; }
  return s;
}

static String hashWebPass(const String& pass, const String& salt) {
  String h = sha256Hex(salt + ":" + pass);
  if (h.length() != 64) return "";
  for (int i = 0; i < 200; i++) h = sha256Hex(h + salt);
  return h;
}

static bool constTimeEquals(const String& a, const String& b) {
  if (a.length() != b.length() || a.length() == 0) return false;
  uint8_t diff = 0;
  for (size_t i = 0; i < a.length(); i++) diff |= (uint8_t)(a[i] ^ b[i]);
  return diff == 0;
}

void loadOrCreateWebCredentials() {
  prefs.begin("radio", false);
  webUser = prefs.getString("webUser", WEB_USER_DEFAULT);
  webSalt = prefs.getString("webSalt", "");
  webHash = prefs.getString("webHash", "");
  if (webSalt.length() < 16 || webHash.length() != 64) {
    webNewPass = randomPassword(10);
    webSalt = randomHex(8);
    webHash = hashWebPass(webNewPass, webSalt);
    webUser = WEB_USER_DEFAULT;
    prefs.putString("webUser", webUser);
    prefs.putString("webSalt", webSalt);
    prefs.putString("webHash", webHash);
    webCredJustCreated = true;
  }
  prefs.end();
}

bool saveWebCredentials(const String& user, const String& pass) {
  if (user.length() < 3 || user.length() > 24) return false;
  if (pass.length() < WEB_PASS_MIN || pass.length() > 48) return false;
  String salt = randomHex(8);
  String hash = hashWebPass(pass, salt);
  if (hash.length() != 64) return false;
  prefs.begin("radio", false);
  prefs.putString("webUser", user);
  prefs.putString("webSalt", salt);
  prefs.putString("webHash", hash);
  prefs.end();
  webUser = user; webSalt = salt; webHash = hash;
  for (int i = 0; i < WEB_SESSION_MAX; i++) webSessions[i].used = false;
  return true;
}

static String cookieToken() {
  if (!server.hasHeader("Cookie")) return "";
  String c = server.header("Cookie");
  int at = c.indexOf("RSID=");
  if (at < 0) return "";
  at += 5;
  int e = c.indexOf(';', at);
  String t = (e < 0) ? c.substring(at) : c.substring(at, e);
  t.trim();
  if (t.length() != 32) return "";
  return t;
}

bool sessionValid() {
#if !WEB_AUTH_ENABLED
  return true;
#else
  String t = cookieToken();
  if (!t.length()) return false;
  uint32_t now = millis();
  for (int i = 0; i < WEB_SESSION_MAX; i++) {
    if (!webSessions[i].used) continue;
    if ((uint32_t)(now - webSessions[i].lastSeen) > WEB_SESSION_TTL_MS) {
      webSessions[i].used = false; continue;
    }
    if (constTimeEquals(t, String(webSessions[i].token))) {
      webSessions[i].lastSeen = now;
      return true;
    }
  }
  return false;
#endif
}

String createWebSession() {
  uint32_t now = millis();
  int slot = -1;
  uint32_t oldest = 0xFFFFFFFFUL;
  for (int i = 0; i < WEB_SESSION_MAX; i++) {
    if (!webSessions[i].used ||
        (uint32_t)(now - webSessions[i].lastSeen) > WEB_SESSION_TTL_MS) { slot = i; break; }
    if (webSessions[i].lastSeen < oldest) { oldest = webSessions[i].lastSeen; slot = i; }
  }
  if (slot < 0) slot = 0;
  String t = randomHex(16);
  strlcpy(webSessions[slot].token, t.c_str(), sizeof(webSessions[slot].token));
  webSessions[slot].lastSeen = now;
  webSessions[slot].used = true;
  return t;
}

void denyAuth() {
  server.sendHeader("Cache-Control", "no-store");
  if (server.uri() == "/") {
    server.sendHeader("Location", "/login", true);
    server.send(302, "text/plain", "");
  } else {
    server.send(401, "application/json",
                "{\"ok\":false,\"auth\":false,\"message\":\"Sesi berakhir. Silakan login ulang.\"}");
  }
}

#define GUARD() do { if (!sessionValid()) { denyAuth(); return; } } while (0)

static bool csrfOK() {
  if (!sessionValid()) return false;
  return server.hasHeader("X-Requested-With") && server.header("X-Requested-With") == "fetch";
}

static void denyCSRF() {
  if (!sessionValid()) { denyAuth(); return; }
  server.send(403, "text/plain", "Forbidden");
}

void showWebCredentials() {
  Serial.printf("[AUTH] Web UI -> user: %s | pass: %s\n", webUser.c_str(), webNewPass.c_str());
  if (!oledOK) { delay(500); return; }
  pinMode(ENC_SW, INPUT_PULLUP);
  while (digitalRead(ENC_SW) == LOW) delay(20);
  delay(100);
  uint32_t t0 = millis();
  while ((uint32_t)(millis() - t0) < 60000) {
    int sisa = 60 - (int)((millis() - t0) / 1000);
    canvas.fillScreen(UI_BLACK);
    canvas.setTextSize(1); canvas.setTextColor(UI_WHITE);
    canvas.setCursor(0, 0);   canvas.print("WEB UI LOGIN");
    canvas.drawLine(0, 10, 127, 10, UI_WHITE);
    canvas.setCursor(0, 14);  canvas.print("User:");
    canvas.setCursor(36, 14); canvas.print(webUser);
    canvas.setCursor(0, 26);  canvas.print("Pass:");
    canvas.setCursor(0, 37);  canvas.print(webNewPass);
    canvas.setCursor(0, 54);  canvas.print("Klik=OK ("); canvas.print(sisa); canvas.print(")");
    renderOLED();
    if (digitalRead(ENC_SW) == LOW) {
      delay(50);
      while (digitalRead(ENC_SW) == LOW) delay(20);
      break;
    }
    delay(250);
  }
  webNewPass = "";
}

/* ================= RADIO MANUAL (NVS) ================= */
int getCustomRadioCount() {
  prefs.begin("radio", true);
  int count = prefs.getUChar("customCount", 0);
  prefs.end();
  return constrain(count, 0, CUSTOM_RADIO_MAX);
}

String customKey(char prefix, int index) { return String(prefix) + String(index); }

bool readCustomRadio(int index, String& name, String& url, String& country, int count = -1) {
  if (count < 0) count = getCustomRadioCount();
  if (index < 0 || index >= count) return false;
  prefs.begin("radio", true);
  name = prefs.getString(customKey('n', index).c_str(), "");
  url = prefs.getString(customKey('u', index).c_str(), "");
  country = prefs.getString(customKey('c', index).c_str(), "ZZ");
  prefs.end();
  return name.length() && (url.startsWith("http://") || url.startsWith("https://"));
}

bool addCustomRadio(String name, String url, String country) {
  name.trim(); url.trim(); country.trim(); country.toUpperCase();
  if (!name.length() || name.length() > 64 || url.length() < 8 || url.length() > 240) return false;
  if (!url.startsWith("http://") && !url.startsWith("https://")) return false;
  if (!(country.length() == 2 && country[0] >= 'A' && country[0] <= 'Z' &&
        country[1] >= 'A' && country[1] <= 'Z')) country = "ZZ";
  int count = getCustomRadioCount();
  if (count >= CUSTOM_RADIO_MAX) return false;
  prefs.begin("radio", false);
  prefs.putString(customKey('n', count).c_str(), name);
  prefs.putString(customKey('u', count).c_str(), url);
  prefs.putString(customKey('c', count).c_str(), country);
  prefs.putUChar("customCount", count + 1);
  prefs.end();
  return true;
}

bool deleteCustomRadio(int index) {
  int count = getCustomRadioCount();
  if (index < 0 || index >= count) return false;
  prefs.begin("radio", false);
  for (int i = index; i < count - 1; i++) {
    prefs.putString(customKey('n', i).c_str(), prefs.getString(customKey('n', i + 1).c_str(), ""));
    prefs.putString(customKey('u', i).c_str(), prefs.getString(customKey('u', i + 1).c_str(), ""));
    prefs.putString(customKey('c', i).c_str(), prefs.getString(customKey('c', i + 1).c_str(), "ZZ"));
  }
  prefs.remove(customKey('n', count - 1).c_str());
  prefs.remove(customKey('u', count - 1).c_str());
  prefs.remove(customKey('c', count - 1).c_str());
  prefs.putUChar("customCount", count - 1);
  prefs.end();
  return true;
}

/* ================= DEDUP INDEX ================= */
class M3UDedupIndex {
 public:
  static constexpr uint16_t MAX_URLS = M3U_STATION_LIMIT;
  static constexpr uint16_t HASH_SLOTS = 4096;
  static constexpr size_t URL_BYTES = 513;

  bool begin() {
    if (urls && slots) return true;
    urls = static_cast<char*>(heap_caps_malloc((size_t)MAX_URLS * URL_BYTES,
                                               MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT));
    slots = static_cast<uint16_t*>(heap_caps_calloc(HASH_SLOTS, sizeof(uint16_t),
                                                    MALLOC_CAP_SPIRAM | MALLOC_CAP_8BIT));
    if (!urls || !slots) { end(); return false; }
    count = 0;
    return true;
  }
  void end() {
    if (urls) heap_caps_free(urls);
    if (slots) heap_caps_free(slots);
    urls = nullptr; slots = nullptr; count = 0;
  }
  ~M3UDedupIndex() { end(); }

  int add(const char* url) {
    if (!urls || !slots || !url) return -1;
    size_t len = strnlen(url, URL_BYTES);
    if (len == 0 || len >= URL_BYTES) return -1;
    uint32_t h = hashUrl(url);
    uint16_t pos = (uint16_t)(h & (HASH_SLOTS - 1));
    for (uint16_t probe = 0; probe < HASH_SLOTS; ++probe) {
      uint16_t slot = slots[pos];
      if (slot == 0) {
        if (count >= MAX_URLS) return -2;
        char* dst = urls + ((size_t)count * URL_BYTES);
        memcpy(dst, url, len);
        dst[len] = '\0';
        slots[pos] = count + 1;
        ++count;
        return 1;
      }
      if (strcmp(urls + ((size_t)(slot - 1) * URL_BYTES), url) == 0) return 0;
      pos = (uint16_t)((pos + 1) & (HASH_SLOTS - 1));
    }
    return -2;
  }

 private:
  char* urls = nullptr;
  uint16_t* slots = nullptr;
  uint16_t count = 0;
  static uint32_t hashUrl(const char* v) {
    uint32_t h = 2166136261UL;
    while (*v) { h ^= (uint8_t)*v++; h *= 16777619UL; }
    return h;
  }
};

/* ================= HELPER M3U ================= */
String normalizeCountryCode(String code) {
  code.trim(); code.toUpperCase();
  if (code.length() == 2 && code[0] >= 'A' && code[0] <= 'Z' &&
      code[1] >= 'A' && code[1] <= 'Z') return code;
  return "ZZ";
}

static int extinfComma(const String& s) {
  bool q = false;
  for (size_t i = 0; i < s.length(); i++) {
    char c = s[i];
    if (c == '"') q = !q;
    else if (c == ',' && !q) return (int)i;
  }
  return -1;
}

String extractM3UCountry(const String& extinf) {
  const char* keys[] = { "tvg-country=\"", "country=\"" };
  for (uint8_t k = 0; k < 2; ++k) {
    int at = extinf.indexOf(keys[k]);
    if (at < 0) continue;
    at += strlen(keys[k]);
    int e = extinf.indexOf('"', at);
    if (e > at) return normalizeCountryCode(extinf.substring(at, e));
  }
  return "ZZ";
}

static String setExtinfCountry(String e, const String& code) {
  if (!e.startsWith("#EXTINF:")) e = "#EXTINF:-1," + e;
  const char* key = "tvg-country=\"";
  int k = e.indexOf(key);
  if (k >= 0) {
    int s = k + strlen(key);
    int q = e.indexOf('"', s);
    if (q > 0) {
      int end = q + 1;
      if (end < (int)e.length() && e[end] == ' ') end++;
      e.remove(k, end - k);
    }
  }
  int c = extinfComma(e);
  if (c < 0) { e += ","; c = e.length() - 1; }
  if (code != "ZZ") e = e.substring(0, c) + " tvg-country=\"" + code + "\"" + e.substring(c);
  return e;
}

bool validStreamUrl(const String& url) {
  return url.startsWith("http://") || url.startsWith("https://");
}

static bool readM3ULine(File& file, String& line) {
  line = "";
  bool consumed = false;
  while (file.available()) {
    int v = file.read();
    if (v < 0) break;
    consumed = true;
    char c = (char)v;
    if (c == '\n') break;
    if (line.length() < 600) line += c;
  }
  line.trim();
  return consumed;
}

static size_t fsFree() {
  if (!filesystemReady) return 0;
  size_t t = LittleFS.totalBytes(), u = LittleFS.usedBytes();
  return (t > u) ? (t - u) : 0;
}

int countValidM3UEntries(const char* path) {
  if (!filesystemReady) return 0;
  File f = LittleFS.open(path, "r");
  if (!f) return 0;
  M3UDedupIndex seen;
  if (!seen.begin()) { f.close(); return 0; }
  int count = 0;
  String line; line.reserve(600);
  while (f.available() && count < M3U_STATION_LIMIT) {
    if (!readM3ULine(f, line)) break;
    if (!line.length() || line.startsWith("#")) continue;
    if (!validStreamUrl(line) || line.length() > 512) continue;
    if (seen.add(line.c_str()) == 1) ++count;
    if ((count & 15) == 0) pumpAudio();
  }
  f.close();
  return count;
}

bool mergeM3UFiles(const char* oldPath, const char* newPath, const char* outPath,
                   int& totalCount, int& addedCount, bool& capped) {
  capped = false;
  if (!filesystemReady) return false;
  LittleFS.remove(outPath);
  File out = LittleFS.open(outPath, "w");
  if (!out) return false;
  M3UDedupIndex seen;
  if (!seen.begin()) { out.close(); LittleFS.remove(outPath); return false; }
  totalCount = 0; addedCount = 0;
  const char* paths[2] = { oldPath, newPath };

  for (int source = 0; source < 2; source++) {
    File in = LittleFS.open(paths[source], "r");
    if (!in) {
      if (source == 0 && !LittleFS.exists(oldPath)) continue;
      out.close(); LittleFS.remove(outPath); return false;
    }
    String pendingName; pendingName.reserve(72);
    String pendingCountry = "ZZ";
    String line; line.reserve(600);
    while (in.available()) {
      if (!readM3ULine(in, line)) break;
      if (!line.length()) continue;
      if (line.startsWith("#EXTINF:")) {
        int comma = extinfComma(line);
        pendingName = (comma >= 0) ? line.substring(comma + 1) : "";
        pendingName.trim();
        if (pendingName.length() > 64) pendingName = pendingName.substring(0, 64);
        pendingCountry = extractM3UCountry(line);
        continue;
      }
      if (line.startsWith("#")) continue;
      if (!validStreamUrl(line) || line.length() > 512) {
        pendingName = ""; pendingCountry = "ZZ"; continue;
      }
      int dr = seen.add(line.c_str());
      if (dr == -1) { pendingName = ""; pendingCountry = "ZZ"; continue; }
      if (dr == -2) { capped = true; break; }
      if (dr == 1) {
        String nm = pendingName.length() ? pendingName : ("Radio M3U " + String(totalCount + 1));
        String extinf = "#EXTINF:-1";
        if (pendingCountry != "ZZ") extinf += " tvg-country=\"" + pendingCountry + "\"";
        extinf += "," + nm;
        if (out.println(extinf) == 0 || out.println(line) == 0) {
          in.close(); out.close(); LittleFS.remove(outPath); return false;
        }
        totalCount++;
        if (source == 1) addedCount++;
      }
      pendingName = ""; pendingCountry = "ZZ";
      if ((totalCount & 15) == 0) pumpAudio();
    }
    in.close();
    if (capped) break;
  }
  out.flush();
  bool writeOK = (bool)out;
  out.close();
  if (!writeOK || totalCount <= 0) { LittleFS.remove(outPath); return false; }
  return true;
}

bool loadM3UStations() {
  if (!filesystemReady || !stations) return false;
  File f = LittleFS.open(M3U_FILE_PATH, "r");
  if (!f) return false;
  M3UDedupIndex seen;
  if (!seen.begin()) { f.close(); return false; }
  int newCount = 0;
  String pendingName; pendingName.reserve(72);
  String pendingCountry = "ZZ";
  String line; line.reserve(600);

  while (f.available() && newCount < M3U_STATION_LIMIT) {
    if (!readM3ULine(f, line)) break;
    if (!line.length()) continue;
    if (line.startsWith("#EXTINF:")) {
      int comma = extinfComma(line);
      pendingName = (comma >= 0) ? line.substring(comma + 1) : "";
      pendingName.trim();
      if (pendingName.length() > 64) pendingName = pendingName.substring(0, 64);
      pendingCountry = extractM3UCountry(line);
      continue;
    }
    if (line.startsWith("#")) continue;
    if (!validStreamUrl(line) || line.length() > 512) {
      pendingName = ""; pendingCountry = "ZZ"; continue;
    }
    if (seen.add(line.c_str()) == 1) {
      String nm = pendingName.length() ? pendingName : ("Radio M3U " + String(newCount + 1));
      strlcpy(stations[newCount].name, nm.c_str(), sizeof(stations[newCount].name));
      strlcpy(stations[newCount].url, line.c_str(), sizeof(stations[newCount].url));
      stations[newCount].bitrate = 0;
      strlcpy(stations[newCount].country, pendingCountry.c_str(), sizeof(stations[newCount].country));
      newCount++;
    }
    pendingName = ""; pendingCountry = "ZZ";
    if ((newCount & 15) == 0) pumpAudio();
  }
  f.close();
  if (newCount <= 0) return false;

  String playing = activeStreamUrl;
  stationCount = newCount;
  stationSource = "m3u";
  stationPage = 0;
  currentStation = -1;
  if (playing.length()) {
    for (int i = 0; i < stationCount; i++)
      if (playing == stations[i].url) { currentStation = i; break; }
  }
  previewStation = (currentStation >= 0) ? currentStation : 0;
  uiDirty = true;
  return true;
}

bool saveM3UStationCountry(int index, const String& country) {
  if (!filesystemReady || stationSource != "m3u" || index < 0 || index >= stationCount) return false;
  String code = normalizeCountryCode(country);
  File in = LittleFS.open(M3U_FILE_PATH, "r");
  if (!in) return false;
  LittleFS.remove(M3U_TEMP_PATH);
  File out = LittleFS.open(M3U_TEMP_PATH, "w");
  if (!out) { in.close(); return false; }

  String pending, line;
  line.reserve(600); pending.reserve(200);
  int idx = -1;
  bool writeOK = true;
  while (in.available()) {
    if (!readM3ULine(in, line)) break;
    if (!line.length()) continue;
    if (line.startsWith("#EXTINF:")) { pending = line; continue; }
    if (line.startsWith("#")) { if (out.println(line) == 0) writeOK = false; continue; }
    if (!validStreamUrl(line) || line.length() > 512) { pending = ""; continue; }
    idx++;
    String ext;
    if (idx == index) {
      String base = pending.length() ? pending : (String("#EXTINF:-1,") + stations[idx].name);
      ext = setExtinfCountry(base, code);
    } else if (pending.length()) ext = pending;
    else ext = String("#EXTINF:-1,") + stations[idx].name;
    if (out.println(ext) == 0 || out.println(line) == 0) writeOK = false;
    pending = "";
    if ((idx & 15) == 0) pumpAudio();
  }
  in.close(); out.flush(); out.close();
  if (!writeOK || idx + 1 != stationCount) { LittleFS.remove(M3U_TEMP_PATH); return false; }

  LittleFS.remove(M3U_BACKUP_PATH);
  bool hadOld = LittleFS.exists(M3U_FILE_PATH);
  if (hadOld && !LittleFS.rename(M3U_FILE_PATH, M3U_BACKUP_PATH)) {
    LittleFS.remove(M3U_TEMP_PATH); return false;
  }
  if (!LittleFS.rename(M3U_TEMP_PATH, M3U_FILE_PATH)) {
    if (hadOld) LittleFS.rename(M3U_BACKUP_PATH, M3U_FILE_PATH);
    return false;
  }
  LittleFS.remove(M3U_BACKUP_PATH);
  strlcpy(stations[index].country, code.c_str(), sizeof(stations[index].country));
  return true;
}

void handleM3UUpload() {
  if (!filesystemReady || !csrfOK()) { m3uUploadOK = false; return; }
  HTTPUpload& upload = server.upload();
  if (upload.status == UPLOAD_FILE_START) {
    if (m3uUploadFile) m3uUploadFile.close();
    LittleFS.remove(M3U_TEMP_PATH);
    m3uUploadSize = 0;
    String fn = upload.filename; fn.toLowerCase();
    m3uUploadOK = (fn.endsWith(".m3u") || fn.endsWith(".m3u8")) &&
                  fsFree() > ((size_t)M3U_MAX_FILE_SIZE * 3);
    if (m3uUploadOK) m3uUploadFile = LittleFS.open(M3U_TEMP_PATH, "w");
    if (!m3uUploadFile) m3uUploadOK = false;
  } else if (upload.status == UPLOAD_FILE_WRITE) {
    m3uUploadSize += upload.currentSize;
    if (m3uUploadSize > M3U_MAX_FILE_SIZE) m3uUploadOK = false;
    if (m3uUploadOK && m3uUploadFile) {
      if (m3uUploadFile.write(upload.buf, upload.currentSize) != upload.currentSize)
        m3uUploadOK = false;
    }
    pumpAudio();
  } else if (upload.status == UPLOAD_FILE_END) {
    if (m3uUploadFile) m3uUploadFile.close();
  } else if (upload.status == UPLOAD_FILE_ABORTED) {
    if (m3uUploadFile) m3uUploadFile.close();
    LittleFS.remove(M3U_TEMP_PATH);
    m3uUploadOK = false;
  }
}

/* ================= HALAMAN LOGIN ================= */
static const char LOGIN_PAGE[] PROGMEM = R"HTML(<!doctype html><html lang="id"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1"><title>Masuk - ESP32 Radio</title><style>
*{box-sizing:border-box}body{margin:0;min-height:100vh;display:flex;align-items:center;justify-content:center;padding:16px;font-family:system-ui,-apple-system,Roboto,Arial,sans-serif;background:radial-gradient(900px 420px at 20% -10%,#0e2d63 0,#050d1c 62%) fixed,#050d1c;color:#eaf2ff}
.cd{width:100%;max-width:380px;background:linear-gradient(#0c1c38,#08142a);border:1px solid #1a3a6e;border-radius:16px;padding:22px}
h1{margin:0 0 4px;font-size:21px}.sb{color:#8ea6c8;font-size:13px;margin-bottom:16px}
label{display:block;margin:12px 0 5px;font-size:14px;color:#cfe0ff}
input{width:100%;padding:12px;border-radius:10px;border:1px solid #23477f;background:#07122a;color:#fff;font-size:16px}
button{width:100%;margin-top:18px;padding:13px;border:0;border-radius:10px;background:linear-gradient(#2a8bff,#0a63e6);color:#fff;font-weight:700;font-size:16px;cursor:pointer}
button:disabled{opacity:.5}.m{margin-top:14px;font-size:14px;min-height:20px;color:#ff8080}
.h{margin-top:16px;font-size:12px;color:#7f97b8;line-height:1.55}
</style></head><body><form class="cd" id="f">
<h1>ESP32 Internet Radio</h1><div class="sb">Masuk untuk mengendalikan perangkat</div>
<label>Username</label><input id="u" autocomplete="username" autocapitalize="none" spellcheck="false" required>
<label>Password</label><input id="p" type="password" autocomplete="current-password" required>
<button id="b" type="submit">Masuk</button><div class="m" id="m"></div>
<div class="h">Lupa password? Matikan perangkat, <b>tahan tombol encoder</b>, lalu nyalakan sambil tetap menahan &plusmn;5 detik. Password baru akan tampil di layar OLED.</div>
</form><script>
const $=i=>document.getElementById(i);
$('f').addEventListener('submit',async e=>{e.preventDefault();$('b').disabled=true;$('m').textContent='';
let fd=new URLSearchParams();fd.set('user',$('u').value);fd.set('pass',$('p').value);
try{let r=await fetch('/login',{method:'POST',headers:{'Content-Type':'application/x-www-form-urlencoded','X-Requested-With':'fetch'},body:fd});
let j=await r.json();if(!r.ok||!j.ok)throw Error(j.message||'Login gagal');location.replace('/');}
catch(err){$('m').textContent=err.message;$('b').disabled=false;$('p').value='';$('p').focus();}});
$('u').focus();
</script></body></html>)HTML";

/* ================= HALAMAN UTAMA ================= */
static const char PAGE_A[] PROGMEM = R"HTML(<!doctype html><html lang="id"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1"><title>ESP32 Internet Radio</title><style>
*{box-sizing:border-box;-webkit-tap-highlight-color:transparent}
body{margin:0;font-family:system-ui,-apple-system,Roboto,Arial,sans-serif;background:radial-gradient(1100px 520px at 15% -10%,#0e2d63 0,#050d1c 62%) fixed,#050d1c;color:#eaf2ff;min-height:100vh}
.w{max-width:1180px;margin:auto;padding:12px}
svg.i{width:1.25em;height:1.25em;fill:none;stroke:currentColor;stroke-width:2;stroke-linecap:round;stroke-linejoin:round;flex:none;vertical-align:-.22em}
svg.i .f{fill:currentColor;stroke:none}
header{display:flex;align-items:center;gap:12px;padding:4px 2px 12px;border-bottom:1px solid #12305c}
header .logo{width:46px;height:46px;color:#2a8bff}
h1{font-size:21px;margin:0;line-height:1.15}.sub{color:#cfe0ff;font-size:13px}
.hr{margin-left:auto;display:flex;align-items:center;gap:10px;text-align:right;font-size:13px}
.hr .sep{width:1px;align-self:stretch;background:#16335f}
#wst{font-weight:700;font-size:15px}#wst.ok{color:#22d35e}#wst.bad{color:#ff6b6b}
.gear{background:none;border:0;color:#2a8bff;padding:2px;font-size:26px;cursor:pointer}
nav{display:flex;gap:8px;overflow-x:auto;padding:12px 0;scrollbar-width:none}nav::-webkit-scrollbar{display:none}
nav button{display:flex;align-items:center;gap:8px;flex:none;padding:11px 18px;border-radius:10px;border:1px solid #1d3d70;background:#0a1830;color:#eaf2ff;font-size:15px;font-weight:600;cursor:pointer}
nav button.on{background:linear-gradient(#2a8bff,#0a63e6);border-color:#4aa0ff}
.g{display:grid;grid-template-columns:1.05fr 1fr;gap:14px;align-items:start}
.pn{background:linear-gradient(#0c1c38,#08142a);border:1px solid #1a3a6e;border-radius:14px;padding:12px;margin-bottom:14px}
.pn h3{margin:0 0 10px;font-size:17px;display:flex;align-items:center;gap:8px}.pn h3 svg{color:#2a8bff}
.row{display:flex;gap:8px;align-items:center}
input,select{width:100%;padding:11px 12px;border-radius:10px;border:1px solid #23477f;background:#07122a;color:#fff;font-size:15px}
input[type=range]{padding:0;height:26px;background:none;border:0;accent-color:#2a8bff}
input[type=checkbox]{width:auto}
.btn{border:0;border-radius:10px;padding:11px 16px;background:linear-gradient(#2a8bff,#0a63e6);color:#fff;font-weight:700;font-size:15px;cursor:pointer;display:inline-flex;align-items:center;justify-content:center;gap:7px}
.btn.s{background:#14305a}.btn:disabled{opacity:.4}.btn.gn{background:linear-gradient(#22d35e,#0fa845)}
.chips{display:flex;gap:7px;overflow-x:auto;padding:10px 0;scrollbar-width:none}.chips::-webkit-scrollbar{display:none}
.chip{flex:none;display:flex;align-items:center;gap:6px;padding:8px 13px;border-radius:9px;border:1px solid #1d3d70;background:#0a1830;color:#eaf2ff;font-size:14px;cursor:pointer}
.chip.on{background:#1e7bff;border-color:#4aa0ff}
.chips select{width:auto;padding:8px 10px;font-size:14px;flex:none}
#lst{max-height:min(60vh,500px);overflow-y:auto;border-radius:10px}
.gh{margin:12px 4px 4px;color:#6fb0ff;font-weight:700;font-size:14px}
.st{display:flex;align-items:center;gap:11px;padding:10px;border-bottom:1px solid #12274a;cursor:pointer}
.st.act{background:linear-gradient(90deg,#1e7bff,#1470f0);border-radius:10px;border-color:transparent}
.st .ic{width:36px;height:36px;color:#cfe0ff}
.st .in{flex:1;min-width:0}.st b{display:block;font-size:16px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.st small{color:#9fb6da;font-size:13px;overflow-wrap:anywhere}.st.act small{color:#e3efff}
.br{font-size:12px;padding:5px 9px;border-radius:8px;background:#102a52;color:#dbe8ff;flex:none}
.ib{background:#0f2447;border:0;color:#e8f1ff;width:36px;height:36px;border-radius:50%;display:flex;align-items:center;justify-content:center;cursor:pointer;flex:none;font-size:16px}
.ib.fv.on,.btn.fv.on{color:#ffc93c}.st .sel{margin-top:6px;padding:6px;font-size:13px}
#pg{display:flex;justify-content:center;gap:7px;padding:12px 0 2px;flex-wrap:wrap}
.pb{min-width:40px;height:40px;border-radius:9px;border:1px solid #1d3d70;background:#0a1830;color:#fff;font-size:15px;cursor:pointer}
.pb.on{background:#1e7bff;border-color:#4aa0ff}.pb:disabled{opacity:.35}
.dots{align-self:center;color:#8ea6c8}.mu{color:#8ea6c8;font-size:13px}
.np{display:flex;gap:14px}
.art{width:112px;height:112px;border-radius:12px;background:linear-gradient(135deg,#eaf4ff,#9cc8ff);color:#0a4fc4;display:flex;flex-direction:column;align-items:center;justify-content:center;flex:none;font-weight:900;font-size:30px}
.art svg{width:36px;height:36px}
.npt{min-width:0;flex:1}.npt h2{margin:0 0 4px;font-size:24px;line-height:1.15;overflow-wrap:anywhere}
.npt div{color:#cfe0ff;font-size:15px;margin-top:3px}
#viz{display:flex;align-items:flex-end;gap:3px;height:38px;margin-top:10px}
#viz i{flex:1;background:#1e7bff;border-radius:2px;height:4px;transition:height .15s}
.vol{display:flex;align-items:center;gap:12px;margin-top:12px}.vol svg{color:#cfe0ff;font-size:26px}
.tr{display:grid;grid-template-columns:repeat(4,1fr);gap:8px;margin-top:2px}.tr .btn{padding:12px 6px}
.ms{display:grid;grid-template-columns:1fr 1fr;gap:6px 14px;font-size:15px}
.ms div{display:flex;gap:8px;align-items:center}.ms .ok{color:#22d35e}.ms svg{color:#2a8bff;font-size:20px}
#toast{position:fixed;left:50%;bottom:22px;transform:translateX(-50%);background:#0f2447;border:1px solid #2a8bff;padding:11px 18px;border-radius:12px;opacity:0;pointer-events:none;transition:.25s;max-width:90vw;z-index:9}
#toast.on{opacity:1}
label{display:block;margin:10px 0 5px;color:#cfe0ff;font-size:14px}
.sec{display:none}.sec.on{display:block}
@media(max-width:860px){.g{grid-template-columns:1fr}.rt{order:-1}.art{width:92px;height:92px;font-size:24px}.npt h2{font-size:21px}.hr .sep{display:none}}
@media(max-width:480px){h1{font-size:18px}.ms{grid-template-columns:1fr}.tr .btn span{display:none}}
</style></head><body>
<svg width="0" height="0" style="position:absolute"><defs>
<symbol id="i-play" viewBox="0 0 24 24"><path class="f" d="M8 5v14l11-7z"/></symbol>
<symbol id="i-pause" viewBox="0 0 24 24"><path class="f" d="M6 5h4v14H6zM14 5h4v14h-4z"/></symbol>
<symbol id="i-star" viewBox="0 0 24 24"><path d="M12 3l2.9 5.9 6.5.9-4.7 4.6 1.1 6.5L12 17.8 6.2 20.9l1.1-6.5L2.6 9.8l6.5-.9z"/></symbol>
<symbol id="i-gear" viewBox="0 0 24 24"><circle cx="12" cy="12" r="3"/><path d="M12 2v3M12 19v3M2 12h3M19 12h3M4.9 4.9l2.1 2.1M17 17l2.1 2.1M19.1 4.9L17 7M7 17l-2.1 2.1"/></symbol>
<symbol id="i-wifi" viewBox="0 0 24 24"><path d="M2 9a15 15 0 0120 0M5.5 12.5a10 10 0 0113 0M9 16a5 5 0 016 0"/><circle class="f" cx="12" cy="19.5" r="1.3"/></symbol>
<symbol id="i-radio" viewBox="0 0 24 24"><rect x="3" y="8" width="18" height="12" rx="2.5"/><path d="M7 8l10-5"/><circle cx="8.5" cy="14" r="2.5"/><path d="M14 12.5h4M14 16h4"/></symbol>
<symbol id="i-vol" viewBox="0 0 24 24"><path d="M3 10v4h4l5 4V6l-5 4z"/><path d="M16 9a4 4 0 010 6M18.5 6.5a8 8 0 010 11"/></symbol>
<symbol id="i-mute" viewBox="0 0 24 24"><path d="M3 10v4h4l5 4V6l-5 4z"/><path d="M16 9l5 6M21 9l-5 6"/></symbol>
<symbol id="i-bt" viewBox="0 0 24 24"><path d="M7 7l10 10-5 4V3l5 4L7 17"/></symbol>
<symbol id="i-globe" viewBox="0 0 24 24"><circle cx="12" cy="12" r="9"/><path d="M3 12h18M12 3c3 3 3 15 0 18M12 3c-3 3-3 15 0 18"/></symbol>
<symbol id="i-music" viewBox="0 0 24 24"><path d="M9 18V5l11-2v13"/><circle cx="6.5" cy="18" r="2.5"/><circle cx="17.5" cy="16" r="2.5"/></symbol>
<symbol id="i-info" viewBox="0 0 24 24"><circle cx="12" cy="12" r="9"/><path d="M12 11v5"/><circle class="f" cx="12" cy="7.8" r="1.1"/></symbol>
<symbol id="i-reload" viewBox="0 0 24 24"><path d="M21 12a9 9 0 11-3-6.7M21 4v5h-5"/></symbol>
<symbol id="i-clock" viewBox="0 0 24 24"><circle cx="12" cy="12" r="9"/><path d="M12 7v5l3 2"/></symbol>
<symbol id="i-cal" viewBox="0 0 24 24"><rect x="3" y="5" width="18" height="16" rx="2"/><path d="M3 10h18M8 3v4M16 3v4"/></symbol>
<symbol id="i-ant" viewBox="0 0 24 24"><circle cx="12" cy="9" r="2"/><path d="M12 11v10M7.8 4.8a6 6 0 000 8.4M16.2 4.8a6 6 0 010 8.4M4.9 2a10 10 0 000 14M19.1 2a10 10 0 010 14"/></symbol>
<symbol id="i-out" viewBox="0 0 24 24"><path d="M10 4H6a2 2 0 00-2 2v12a2 2 0 002 2h4M16 8l4 4-4 4M20 12H10"/></symbol>
</defs></svg>
<div class="w"><header><svg class="i logo"><use href="#i-ant"/></svg>
<div><h1>ESP32 Internet Radio</h1><div class="sub">WROVER &bull; A2DP &bull; Web UI</div></div>
<div class="hr"><div><div id="wst" class="bad"><svg class="i"><use href="#i-wifi"/></svg> ...</div><div id="ip">IP: -</div></div>
<div class="sep"></div>
<button class="gear" onclick="showTab('set')" aria-label="Pengaturan"><svg class="i"><use href="#i-gear"/></svg></button>
<button class="gear" onclick="doLogout()" aria-label="Keluar" title="Keluar" style="color:#ff7b7b"><svg class="i"><use href="#i-out"/></svg></button>
</div></header>
<nav id="tabs">
<button data-t="browser" class="on" onclick="showTab('browser')"><svg class="i"><use href="#i-globe"/></svg>Radio Browser</button>
<button data-t="m3u" onclick="showTab('m3u')"><svg class="i"><use href="#i-music"/></svg>M3U</button>
<button data-t="man" onclick="showTab('man')"><svg class="i"><use href="#i-star"/></svg>Manual</button>
<button data-t="set" onclick="showTab('set')"><svg class="i"><use href="#i-gear"/></svg>Settings</button>
<button data-t="about" onclick="showTab('about')"><svg class="i"><use href="#i-info"/></svg>About</button></nav>
<div class="g"><div class="lf">
<div class="pn sec on" id="s-list">
<div id="tb-browser"><div class="row"><input id="q" placeholder="Cari stasiun..." oninput="renderStations()" onkeydown="if(event.key==='Enter')loadPage(0)"><button class="btn" onclick="loadPage(0)">Search</button></div>
<div class="chips" id="chips"></div>
<div class="row" style="margin-bottom:8px"><select id="genre" onchange="loadPage(0)"><option value="all">Semua genre</option><option value="music">Music</option><option value="news">News</option><option value="jazz">Jazz</option><option value="classical">Classical</option><option value="rock">Rock</option></select></div></div>
<div id="tb-m3u" style="display:none"><form id="m3uForm" enctype="multipart/form-data"><input id="m3uFile" type="file" accept=".m3u,.m3u8,audio/x-mpegurl" required><button class="btn" type="submit" style="width:100%;margin-top:10px">Unggah &amp; Muat M3U</button></form>
<button class="btn s" id="fmtBtn" type="button" style="display:none;width:100%;margin-top:8px" onclick="fmtFS()">Format LittleFS (ESP32 Baru)</button>
<p class="mu">Maks. 180 KB per unggahan, koleksi maks. 1500 stasiun, URL duplikat dilewati, tersimpan setelah restart.</p>
<div class="row"><select id="m3uCountryFilter" onchange="renderStations()"><option value="all">Semua negara</option></select></div>
<div class="mu" id="m3uInfo" style="margin:8px 0">Memuat playlist M3U...</div></div>
<div class="row" style="justify-content:space-between;margin:2px 2px 8px"><span class="mu" id="count">Memuat stasiun...</span><select id="country" style="width:auto;display:none"></select></div>
<div id="lst"><div class="mu" id="stations">Memuat...</div></div><div id="pg"></div></div>
<div class="pn sec" id="s-man"><h3><svg class="i"><use href="#i-star"/></svg>Tambah alamat radio sendiri</h3>
<form id="customForm"><div style="display:grid;gap:8px"><input id="customName" maxlength="64" placeholder="Nama radio" required><input id="customUrl" maxlength="240" type="url" placeholder="https://alamat-stream-radio" required><select id="customCountry"></select></div>
<button class="btn" type="submit" style="width:100%;margin-top:10px">+ Simpan alamat radio</button></form>
<p class="mu">URL streaming HTTP/HTTPS. Maksimal 30 alamat, disimpan permanen di ESP32.</p>
<h3 style="margin-top:14px">Radio tambahan saya</h3><div id="customStations" class="mu">Memuat radio tersimpan...</div></div>
<div class="pn sec" id="s-set"><h3><svg class="i"><use href="#i-wifi"/></svg>Pengaturan Wi-Fi</h3>
<form id="wifiForm"><label>Nama Wi-Fi (SSID)</label><input id="ssid" maxlength="32" required>
<label>Password Wi-Fi</label><input id="pass" type="password" maxlength="64" placeholder="Kosongkan jika tidak ingin mengubah password">
<label style="display:flex;gap:8px;align-items:center"><input type="checkbox" id="clr"> Hapus password (jaringan terbuka)</label>
<button class="btn" type="submit" style="width:100%;margin-top:12px">Simpan Wi-Fi &amp; Restart</button></form>
<p class="mu">Jika Wi-Fi gagal tersambung, hubungkan HP ke hotspot <b>Radio-ESP32-Setup</b> (password <b>12345678</b>) lalu buka <b>http://192.168.4.1</b>. Setelah tersambung ke Wi-Fi rumah, bisa juga lewat <b>http://radio.local</b>.</p>
<h3 style="margin-top:18px"><svg class="i"><use href="#i-gear"/></svg>Keamanan Web UI</h3>
<form id="authForm"><label>Username</label><input id="au" maxlength="24" autocomplete="username" required>
<label>Password lama</label><input id="ao" type="password" autocomplete="current-password" required>
<label>Password baru (min. 8 karakter)</label><input id="an" type="password" minlength="8" maxlength="48" autocomplete="new-password" required>
<label>Ulangi password baru</label><input id="an2" type="password" minlength="8" maxlength="48" autocomplete="new-password" required>
<button class="btn" type="submit" style="width:100%;margin-top:12px">Simpan &amp; Login Ulang</button></form>
<p class="mu">Password disimpan sebagai hash SHA-256 bersalt di NVS, bukan teks asli. Sesi berlaku 30 menit tanpa aktivitas, maksimal 4 perangkat.</p></div>
<div class="pn sec" id="s-about"><h3><svg class="i"><use href="#i-info"/></svg>About</h3>
<p class="mu" style="font-size:14px;line-height:1.6">Internet Radio ESP32 WROVER dengan Radio Browser, playlist M3U, radio manual, dan mode Bluetooth A2DP (ganti mode: tahan tombol encoder 3 detik). Proyek asal: Tech Talkies. Jam/tanggal diambil dari HP Anda. Favorit disimpan di browser ini.</p></div>
</div><div class="rt">
<div class="pn"><h3><svg class="i"><use href="#i-music"/></svg>Now Playing</h3>
<div class="np"><div class="art" id="art"><svg class="i"><use href="#i-radio"/></svg></div>
<div class="npt"><h2 id="now">Belum ada stasiun</h2><div id="ncty"></div><div id="detail" class="mu" style="font-size:14px">Menunggu status perangkat...</div><div id="viz"></div></div></div>
<div class="vol"><svg class="i"><use href="#i-vol"/></svg><input type="range" id="vol" min="0" max="21" value="0" oninput="volInput(this.value)"><b id="vp" style="min-width:74px;text-align:right">Vol: -</b><button class="btn s" id="mt" onclick="api('/mute')" aria-label="Mute"><svg class="i"><use href="#i-mute"/></svg></button></div></div>
<div class="pn"><h3><svg class="i"><use href="#i-info"/></svg>Mode &amp; Status</h3><div class="ms">
<div><svg class="i"><use href="#i-wifi"/></svg>Wi-Fi: <b id="s1">-</b></div>
<div><svg class="i"><use href="#i-clock"/></svg>Time: <b id="tm">-</b></div>
<div><svg class="i"><use href="#i-bt"/></svg>Bluetooth: <b id="s3">Off</b></div>
<div><svg class="i"><use href="#i-cal"/></svg>Date: <b id="dt">-</b></div>
<div><svg class="i"><use href="#i-music"/></svg>Mode: <b id="s2">-</b></div></div></div>
<div class="tr"><button class="btn gn" onclick="tr('play')"><svg class="i"><use href="#i-play"/></svg><span>Play</span></button>
<button class="btn" onclick="tr('pause')"><svg class="i"><use href="#i-pause"/></svg><span>Pause</span></button>
<button class="btn s fv" id="fvb" onclick="favNow()"><svg class="i"><use href="#i-star"/></svg><span>Favorite</span></button>
<button class="btn" onclick="tr('reload')"><svg class="i"><use href="#i-reload"/></svg><span>Reload</span></button></div>
</div></div></div><div id="toast"></div>
)HTML";

static const char PAGE_B[] PROGMEM = R"HTML(<script>
let stations=[],current=-1,page=0,loading=false,m3uMode=false,m3uTotal=0,st={},favOnly=false,fav=new Set(),volDrag=false,volT=0;
const $=id=>document.getElementById(id),PER=150,H={'X-Requested-With':'fetch'};
const PH=Object.assign({'Content-Type':'application/x-www-form-urlencoded'},H);
try{fav=new Set(JSON.parse(localStorage.getItem('fav')||'[]'))}catch(e){}
function saveFav(){try{localStorage.setItem('fav',JSON.stringify([...fav]))}catch(e){}}
function chk(r){if(r.status===401){location.replace('/login');throw Error('Sesi berakhir');}return r;}
async function doLogout(){try{await fetch('/logout',{method:'POST',headers:H});}catch(e){}location.replace('/login');}
function toast(m){let t=$('toast');t.textContent=m;t.classList.add('on');clearTimeout(t._h);t._h=setTimeout(()=>t.classList.remove('on'),2800);}
function ic(n){return '<svg class="i"><use href="#i-'+n+'"/></svg>';}
function flag(c){c=(c||'').toUpperCase();if(!/^[A-Z]{2}$/.test(c)||c==='ZZ')return '\u{1F310}';return String.fromCodePoint(...[...c].map(x=>127397+x.charCodeAt(0)));}
function cname(c){let o=CO.find(x=>x[0]===c);return o?o[1]:((!c||c==='ZZ')?'Belum diketahui':c);}
function mk(t,c,x){let e=document.createElement(t);if(c)e.className=c;if(x!==undefined)e.textContent=x;return e;}
const api=async u=>{try{let r=chk(await fetch(u,{headers:H,cache:'no-store'}));if(!r.ok)throw Error(await r.text());loadStatus();return true;}catch(e){toast('Gagal: '+e.message);return false;}};
function initControls(){
let c=$('country'),cu=$('customCountry'),mf=$('m3uCountryFilter');
CO.forEach(o=>{c.append(new Option(o[1],o[0]));if(o[0]!=='all'){cu.append(new Option(o[1],o[0]));mf.append(new Option(o[1],o[0]));}});
c.value='ID';cu.value='ZZ';$('ssid').value=SSID;
let ch=$('chips');
ch.innerHTML='<button class="chip" data-c="__fav">'+ic('star')+'Favorit</button>';
ch.firstChild.onclick=()=>{favOnly=!favOnly;renderChips();renderStations();};
['all','ID','US','JP','GB'].forEach(code=>{let b=mk('button','chip');b.dataset.c=code;b.innerHTML=(code==='all'?'':flag(code)+' ')+cname(code);b.onclick=()=>setCountry(code);ch.append(b);});
let more=document.createElement('select');more.id='more';more.append(new Option('More \u25BE',''));
CO.forEach(o=>{if(!['all','ID','US','JP','GB'].includes(o[0]))more.append(new Option(flag(o[0])+' '+o[1],o[0]));});
more.onchange=()=>{if(more.value)setCountry(more.value);};ch.append(more);
let v=$('viz');for(let i=0;i<32;i++)v.append(document.createElement('i'));
}
function renderChips(){let cv=$('country').value;[...$('chips').children].forEach(b=>{if(b.tagName!=='BUTTON')return;let c=b.dataset.c;b.classList.toggle('on',c==='__fav'?favOnly:c===cv);});let m=$('more');if(m)m.value=['all','ID','US','JP','GB'].includes(cv)?'':cv;}
function setCountry(c){$('country').value=c;m3uMode=false;renderChips();loadPage(0);}
function showTab(t){
document.querySelectorAll('#tabs button').forEach(b=>b.classList.toggle('on',b.dataset.t===t));
let list=(t==='browser'||t==='m3u');
['s-list','s-man','s-set','s-about'].forEach(id=>$(id).classList.remove('on'));
$(list?'s-list':'s-'+t).classList.add('on');
$('tb-browser').style.display=t==='browser'?'':'none';$('tb-m3u').style.display=t==='m3u'?'':'none';
if(t==='browser'){$('q').value='';favOnly=false;renderChips();if(m3uMode||!stations.length){m3uMode=false;loadPage(0);}else renderStations();}
else if(t==='m3u'){$('q').value='';favOnly=false;$('m3uCountryFilter').value='all';openM3U();}
else if(t==='man')loadCustom();
}
const openM3U=async()=>{
if(loading)return;$('stations').textContent='';
try{let r=chk(await fetch('/m3u/load',{headers:H,cache:'no-store'}));let j=await r.json();if(!r.ok||!j.ok)throw Error(j.message||'Belum ada playlist M3U');m3uMode=true;m3uTotal=j.count||0;await loadM3UPage(0);}
catch(e){m3uMode=false;stations=[];renderStations();$('count').textContent='';$('m3uInfo').textContent=e.message+'. Unggah file .m3u di atas.';$('pg').innerHTML='';}
};
const fmtFS=async()=>{
if(!confirm('Format LittleFS? Hanya untuk ESP32 baru atau partisi belum aktif.'))return;
try{let r=chk(await fetch('/m3u/format',{method:'POST',headers:H}));let j=await r.json();toast(j.message||'Selesai');loadStatus();if(j.ok)openM3U();}
catch(e){toast('Gagal format: '+e.message);}
};
const loadM3UPage=async p=>{if(loading||p<0)return;loading=true;page=p;$('stations').textContent='Memuat playlist...';
try{let off=page*PER;let r=chk(await fetch('/stations?offset='+off+'&limit='+PER,{headers:H,cache:'no-store'}));if(!r.ok)throw Error('Gagal membaca playlist M3U');stations=await r.json();renderStations();renderPager();$('count').textContent=stations.length+' dari '+m3uTotal+' stasiun pada halaman ini';$('m3uInfo').textContent='Playlist tersimpan: '+m3uTotal+' stasiun';}
catch(e){$('stations').textContent=e.message;}finally{loading=false;loadStatus();}
};
const loadPage=async p=>{if(m3uMode)return loadM3UPage(p);if(loading||p<0)return;loading=true;page=p;$('stations').textContent='Memuat daftar radio...';$('count').textContent='Menghubungi Radio Browser...';
try{let u='/refresh?page='+page+'&country='+encodeURIComponent($('country').value)+'&genre='+encodeURIComponent($('genre').value)+'&q='+encodeURIComponent($('q').value.trim());
let r=chk(await fetch(u,{headers:H,cache:'no-store'}));let j=await r.json();if(!r.ok||!j.ok)throw Error(j.message||'Gagal mengambil daftar');
let sr=chk(await fetch('/stations?offset=0&limit='+PER,{headers:H,cache:'no-store'}));stations=await sr.json();m3uMode=false;renderChips();renderStations();renderPager();$('count').textContent=stations.length+' stasiun pada halaman ini';}
catch(e){stations=[];$('stations').textContent=e.message;$('count').textContent='Periksa koneksi internet perangkat';$('pg').innerHTML='';}
finally{loading=false;loadStatus();}
};
function goPage(p){m3uMode?loadM3UPage(p):loadPage(p);}
function renderPager(){let n=m3uMode?Math.max(1,Math.ceil(m3uTotal/PER)):page+1+(stations.length>=PER?1:0);
let h='<button class="pb" '+(page<1?'disabled':'')+' onclick="goPage('+(page-1)+')">\u2039</button>';
let a=[...new Set([0,page-1,page,page+1,n-1])].filter(x=>x>=0&&x<n).sort((x,y)=>x-y),l=-1;
a.forEach(x=>{if(l>=0&&x-l>1)h+='<span class="dots">\u2026</span>';h+='<button class="pb'+(x===page?' on':'')+'" onclick="goPage('+x+')">'+(x+1)+'</button>';l=x;});
h+='<button class="pb" '+(page>=n-1?'disabled':'')+' onclick="goPage('+(page+1)+')">\u203A</button>';$('pg').innerHTML=h;}
function renderStations(){
let q=$('q').value.trim().toLocaleLowerCase(),cf=m3uMode?$('m3uCountryFilter').value:'all',el=$('stations');
let a=stations.filter(s=>(s.name||'').toLocaleLowerCase().includes(q)&&(cf==='all'||(s.country||'ZZ')===cf)&&(!favOnly||fav.has(s.name)));
el.innerHTML='';if(!a.length){el.textContent=favOnly?'Belum ada favorit pada daftar ini.':'Tidak ada stasiun yang cocok pada filter ini.';return;}
let groups={};a.forEach(s=>{let c=s.country||'ZZ';(groups[c]=groups[c]||[]).push(s);});
let keys=Object.keys(groups).sort((x,y)=>x==='ZZ'?1:y==='ZZ'?-1:x.localeCompare(y));
keys.forEach(code=>{
if(m3uMode&&keys.length>1)el.append(mk('div','gh',flag(code)+' '+cname(code)+' ('+groups[code].length+')'));
groups[code].forEach(s=>{
let d=mk('div','st'+(s.index===current?' act':''));d.dataset.stationIndex=s.index;d.onclick=()=>play(s.index);
let i=mk('div','ic');i.innerHTML=ic('radio');i.firstChild.style.cssText='width:36px;height:36px';
let inf=mk('div','in'),b=mk('b','',s.name),sm=mk('small','',flag(s.country)+' '+cname(s.country||'ZZ'));inf.append(b,sm);
if(m3uMode){let sel=mk('select','sel');sel.onclick=e=>e.stopPropagation();[...$('customCountry').options].forEach(o=>sel.append(new Option(o.textContent,o.value)));sel.value=s.country||'ZZ';
sel.onchange=async()=>{let fd=new URLSearchParams();fd.set('i',s.index);fd.set('country',sel.value);
try{let r=chk(await fetch('/m3u/country',{method:'POST',headers:PH,body:fd}));if(!r.ok)throw Error(await r.text());s.country=sel.value;renderStations();toast('Negara tersimpan');}catch(e){toast('Gagal menyimpan negara');}};
inf.append(sel);}
d.append(i,inf);
if(s.bitrate)d.append(mk('span','br',s.bitrate+' kbps'));
let f=mk('button','ib fv'+(fav.has(s.name)?' on':''));f.innerHTML=ic('star');f.onclick=e=>{e.stopPropagation();togFav(s.name);};
let p=mk('button','ib');p.innerHTML=ic('play');p.onclick=e=>{e.stopPropagation();play(s.index);};
d.append(f,p);el.append(d);});});
}
function togFav(n){if(!n)return;fav.has(n)?fav.delete(n):fav.add(n);saveFav();renderStations();updFavBtn();}
function favNow(){togFav(st.nowPlaying);}
function updFavBtn(){$('fvb').classList.toggle('on',!!st.nowPlaying&&fav.has(st.nowPlaying));}
const play=async i=>{if(await api('/play?i='+i)){current=i;updateActiveStationUI();}};
function tr(a){api('/transport?a='+a);}
function updateActiveStationUI(){document.querySelectorAll('[data-station-index]').forEach(el=>el.classList.toggle('act',Number(el.dataset.stationIndex)===current));}
function volInput(v){volDrag=true;let mx=Number($('vol').max)||21;$('vp').textContent='Vol: '+Math.round(v/mx*100)+'%';clearTimeout(volT);
volT=setTimeout(async()=>{await fetch('/vol?v='+v,{headers:H,cache:'no-store'}).catch(()=>{});volDrag=false;loadStatus();},180);}
const STL={streaming:'Streaming aktif',connecting:'Menghubungkan stream...',retry:'Mencoba menyambung ulang...',ended:'Stream berakhir',error:'Stream gagal',paused:'Dijeda','wifi-disconnected':'Wi-Fi terputus',stopped:'Belum memutar'};
const loadStatus=async()=>{try{let r=chk(await fetch('/status',{headers:H,cache:'no-store'}));let s=await r.json();st=s;current=s.station;
let sta=s.mode==='STA',w=$('wst');w.innerHTML=ic('wifi')+' '+(sta?'Connected':(s.mode==='AP'?'Hotspot':'Offline'));w.className=sta?'ok':'bad';
$('ip').textContent='IP: '+(s.ip||'-');
$('s1').textContent=sta?'Connected':(s.mode==='AP'?'Hotspot':'Offline');$('s1').className=sta?'ok':'';
$('s2').textContent=s.source==='m3u'?'M3U Playlist':'Internet Radio';
let nm=s.nowPlaying||'Belum ada stasiun';$('now').textContent=nm;
let it=stations.find(x=>x.index===s.station);let cc=it?it.country:s.country;
$('ncty').textContent=it?(flag(cc)+' '+cname(cc||'ZZ')):(s.station>=0?flag(s.country)+' '+cname(s.country):'');
let state=STL[s.streamState]||'Status tidak diketahui';
if(s.streamState==='error'&&s.lastError)state+=' \u00B7 '+s.lastError;
$('detail').textContent=state+(s.genre&&s.genre!=='all'?' \u00B7 '+s.genre:'');
let ini=nm.replace(/[^A-Za-z0-9 ]/g,'').trim().split(/\s+/).slice(0,2).map(x=>x[0]).join('').toUpperCase();
$('art').innerHTML=(ini&&s.nowPlaying)?'<div>'+ini+'</div>'+ic('radio'):ic('radio');
if(!volDrag){let mx=s.volMax||21;$('vol').max=mx;$('vol').value=s.volume;$('vp').textContent='Vol: '+Math.round(s.volume/mx*100)+'%';}
$('mt').innerHTML=s.muted?ic('mute'):ic('vol');$('mt').style.color=s.muted?'#ff7b7b':'';
$('fmtBtn').style.display=s.filesystemReady?'none':'inline-flex';
if(!s.filesystemReady)$('m3uInfo').textContent='LittleFS belum siap; klik Format LittleFS di atas jika ini ESP32 baru.';
if(s.user&&!$('au').value)$('au').value=s.user;
updFavBtn();updateActiveStationUI();
}catch(e){let w=$('wst');w.textContent='Perangkat tidak terjangkau';w.className='bad';}};
function tick(){let d=new Date();$('tm').textContent=d.toLocaleTimeString('id-ID',{hour12:false}).replace(/\./g,':');$('dt').textContent=d.toLocaleDateString('en-GB');}
function viz(){let on=st.streamState==='streaming'&&!st.paused&&!st.muted&&!document.hidden;
[...$('viz').children].forEach((b,i)=>{b.style.height=on?(4+Math.random()*34*(1-Math.abs(i-8)/40))+'px':'4px';});}
const loadCustom=async()=>{try{let r=chk(await fetch('/custom',{headers:H,cache:'no-store'}));let list=await r.json();let el=$('customStations');el.innerHTML='';
if(!list.length){el.textContent='Belum ada alamat radio tambahan.';return;}
list.forEach((s,i)=>{let d=mk('div','st');let inf=mk('div','in');inf.append(mk('b','',s.name),mk('small','',s.url+' \u00B7 '+cname(s.country)));
let p=mk('button','ib');p.innerHTML=ic('play');p.onclick=()=>api('/playcustom?i='+i);
let x=mk('button','ib','\u2715');x.onclick=async()=>{if(!confirm('Hapus '+s.name+'?'))return;let fd=new URLSearchParams();fd.set('i',i);
let rr=await fetch('/custom/delete',{method:'POST',headers:PH,body:fd});if(!rr.ok)toast(await rr.text());loadCustom();};
d.append(inf,p,x);el.append(d);});}catch(e){$('customStations').textContent='Gagal membaca radio tersimpan.';}};
$('m3uForm').addEventListener('submit',async e=>{e.preventDefault();let f=$('m3uFile').files[0];if(!f)return;
let fd=new FormData();fd.append('m3u',f);$('m3uInfo').textContent='Mengunggah '+f.name+'...';
try{let r=chk(await fetch('/m3u/upload',{method:'POST',headers:H,body:fd}));let j=await r.json();if(!r.ok||!j.ok)throw Error(j.message||'Gagal memuat M3U');
m3uMode=true;m3uTotal=j.count||0;current=-1;page=0;$('q').value='';await loadM3UPage(0);
$('m3uInfo').textContent='Koleksi: '+m3uTotal+' stasiun \u00B7 baru: '+(j.added||0);
toast('Impor selesai: '+(j.added||0)+' stasiun baru. Total '+m3uTotal+'.');}
catch(err){$('m3uInfo').textContent='Gagal: '+err.message;toast('Impor M3U gagal: '+err.message);}});
$('customForm').addEventListener('submit',async e=>{e.preventDefault();let fd=new URLSearchParams();
fd.set('name',$('customName').value);fd.set('url',$('customUrl').value);fd.set('country',$('customCountry').value);
try{let r=chk(await fetch('/custom/add',{method:'POST',headers:PH,body:fd}));if(!r.ok)throw Error(await r.text());
$('customName').value='';$('customUrl').value='';await loadCustom();toast('Alamat radio disimpan di ESP32.');}
catch(err){toast('Gagal menyimpan: '+err.message);}});
$('wifiForm').addEventListener('submit',async e=>{e.preventDefault();let fd=new URLSearchParams();
fd.set('ssid',$('ssid').value);fd.set('pass',$('pass').value);if($('clr').checked)fd.set('clear','1');
try{let r=chk(await fetch('/save',{method:'POST',headers:PH,body:fd}));if(!r.ok)throw Error(await r.text());
toast('Wi-Fi tersimpan. Perangkat restart...');}catch(err){toast('Gagal: '+err.message);}});
$('authForm').addEventListener('submit',async e=>{e.preventDefault();
if($('an').value!==$('an2').value){toast('Konfirmasi password tidak sama');return;}
let fd=new URLSearchParams();fd.set('old',$('ao').value);fd.set('user',$('au').value);fd.set('new',$('an').value);fd.set('new2',$('an2').value);
try{let r=chk(await fetch('/webauth',{method:'POST',headers:PH,body:fd}));if(!r.ok)throw Error(await r.text());
toast('Kredensial tersimpan. Silakan login ulang.');setTimeout(()=>location.replace('/login'),1200);}
catch(err){toast('Gagal: '+err.message);}});
const initStations=async()=>{try{let r=chk(await fetch('/status',{headers:H,cache:'no-store'}));let s=await r.json();
if(s.source==='m3u'){m3uTotal=s.stationCount||0;showTab('m3u');}else{m3uMode=false;await loadPage(0);}}
catch(e){$('count').textContent='Gagal memuat daftar radio';}};
initControls();renderChips();tick();setInterval(tick,1000);setInterval(viz,170);initStations();loadStatus();setInterval(loadStatus,4000);
</script></body></html>)HTML";

static String jsStr(const String& in) {
  String o = "\"";
  for (size_t i = 0; i < in.length(); i++) {
    char c = in[i];
    if (c == '\\' || c == '"') { o += '\\'; o += c; }
    else if (c == '<') o += "\\u003c";
    else if ((uint8_t)c < 32) o += ' ';
    else o += c;
  }
  o += '"';
  return o;
}

void sendConfigPage() {
  server.sendHeader("Cache-Control", "no-store");
  server.setContentLength(CONTENT_LENGTH_UNKNOWN);
  server.send(200, "text/html; charset=utf-8", "");
  server.sendContent_P(PAGE_A);
  String d = "<script>const CO=[";
  for (int i = 0; i < COUNTRY_COUNT; i++) {
    if (i) d += ',';
    d += "[" + jsStr(String(COUNTRIES[i].code)) + "," + jsStr(String(COUNTRIES[i].name)) + "]";
  }
  d += "];const SSID=" + jsStr(wifiSsid) + ";</script>";
  server.sendContent(d);
  server.sendContent_P(PAGE_B);
  server.sendContent("");
}

/* ================= WEB SERVER ================= */
void startWebUI() {
  if (webStarted) return;
  const char* hdrs[] = { "X-Requested-With", "Cookie" };
  server.collectHeaders(hdrs, 2);

  server.on("/", HTTP_GET, []() { GUARD(); sendConfigPage(); });

  server.on("/login", HTTP_GET, []() {
    if (sessionValid()) {
      server.sendHeader("Location", "/", true);
      server.send(302, "text/plain", "");
      return;
    }
    server.sendHeader("Cache-Control", "no-store");
    server.send_P(200, "text/html; charset=utf-8", LOGIN_PAGE);
  });

  server.on("/login", HTTP_POST, []() {
    uint32_t now = millis();
    if (webLockUntil && (int32_t)(now - webLockUntil) < 0) {
      uint32_t s = (webLockUntil - now) / 1000 + 1;
      DynamicJsonDocument d(256);
      d["ok"] = false;
      d["message"] = "Terlalu banyak percobaan. Coba lagi dalam " + String(s) + " detik.";
      String j; serializeJson(d, j);
      server.send(429, "application/json", j);
      return;
    }
    String u = server.arg("user"), p = server.arg("pass");
    u.trim();
    String h = hashWebPass(p, webSalt);
    bool ok = (u == webUser) && h.length() == 64 && constTimeEquals(h, webHash);
    if (!ok) {
      webLoginFail++;
      if (webLoginFail >= WEB_LOGIN_MAX_FAIL) {
        webLockUntil = millis() + WEB_LOGIN_LOCK_MS;
        webLoginFail = 0;
      }
      delay(250);
      sendJsonOk(false, "Username atau password salah.", 401);
      return;
    }
    webLoginFail = 0; webLockUntil = 0;
    String t = createWebSession();
    server.sendHeader("Set-Cookie", "RSID=" + t + "; Path=/; HttpOnly; SameSite=Strict; Max-Age=86400");
    server.sendHeader("Cache-Control", "no-store");
    sendJsonOk(true, "Login berhasil");
  });

  server.on("/logout", HTTP_POST, []() {
    String t = cookieToken();
    for (int i = 0; i < WEB_SESSION_MAX; i++)
      if (webSessions[i].used && t == String(webSessions[i].token)) webSessions[i].used = false;
    server.sendHeader("Set-Cookie", "RSID=; Path=/; HttpOnly; SameSite=Strict; Max-Age=0");
    sendJsonOk(true, "Anda sudah keluar");
  });

  server.on("/webauth", HTTP_POST, []() {
    if (!csrfOK()) { denyCSRF(); return; }
    String oldp = server.arg("old"), u = server.arg("user");
    String np = server.arg("new"), np2 = server.arg("new2");
    u.trim();
    String h = hashWebPass(oldp, webSalt);
    if (!constTimeEquals(h, webHash)) {
      delay(250);
      server.send(403, "text/plain", "Password lama salah");
      return;
    }
    if (np != np2) { server.send(400, "text/plain", "Konfirmasi password baru tidak sama"); return; }
    if (!saveWebCredentials(u, np)) {
      server.send(400, "text/plain", "Username 3-24 karakter, password 8-48 karakter");
      return;
    }
    server.sendHeader("Set-Cookie", "RSID=; Path=/; HttpOnly; SameSite=Strict; Max-Age=0");
    server.send(200, "text/plain", "Kredensial diperbarui. Silakan login ulang.");
  });

  server.on("/status", HTTP_GET, []() {
    GUARD();
    DynamicJsonDocument doc(2048);
    doc["mode"] = apMode ? "AP" : (WiFi.status() == WL_CONNECTED ? "STA" : "OFFLINE");
    doc["ip"] = wifiIP;
    doc["ssid"] = wifiSsid;
    doc["station"] = currentStation;
    doc["stationCount"] = stationCount;
    doc["page"] = stationPage;
    doc["country"] = selectedCountry;
    doc["genre"] = selectedGenre;
    doc["source"] = stationSource;
    String title = streamTitle;
    if (title.length() > 120) title = title.substring(0, 120);
    doc["nowPlaying"] = title;
    doc["playing"] = radioStreamActive;
    doc["streamState"] = radioStreamState;
    doc["lastError"] = lastError;
    doc["filesystemReady"] = filesystemReady;
    doc["volume"] = muted ? lastVol : audio.getVolume();
    doc["volMax"] = VOL_MAX;
    doc["muted"] = muted;
    doc["paused"] = userPaused;
    doc["user"] = webUser;
    String json; serializeJson(doc, json);
    server.sendHeader("Cache-Control", "no-store");
    server.send(200, "application/json", json);
  });

  server.on("/refresh", HTTP_GET, []() {
    GUARD();
    int requestedPage = server.hasArg("page") ? server.arg("page").toInt() : 0;
    if (requestedPage < 0) requestedPage = 0;
    if (server.hasArg("country")) {
      String c = server.arg("country");
      for (int k = 0; k < COUNTRY_COUNT; k++) if (c == COUNTRIES[k].code) { selectedCountry = c; break; }
    }
    if (server.hasArg("genre")) {
      String g = server.arg("genre");
      for (int k = 0; k < GENRE_COUNT; k++) if (g == GENRES[k].tag) { selectedGenre = g; break; }
    }
    searchName = server.hasArg("q") ? server.arg("q") : "";
    searchName.trim();
    if (searchName.length() > 64) searchName = searchName.substring(0, 64);
    stationPage = requestedPage;
    bool ok = fetchStations(selectedGenre, stationPage * RB_RESULT_LIMIT, searchName);
    if (ok) { stationSource = "radiobrowser"; uiDirty = true; }
    DynamicJsonDocument doc(384);
    doc["ok"] = ok;
    doc["page"] = stationPage;
    doc["count"] = stationCount;
    doc["message"] = ok ? "OK" : "Tidak ada hasil atau koneksi gagal";
    String json; serializeJson(doc, json);
    server.send(ok ? 200 : 503, "application/json", json);
  });

  server.on("/stations", HTTP_GET, []() {
    GUARD();
    if (!stations) { server.send(503, "application/json", "[]"); return; }
    int offset = server.hasArg("offset") ? server.arg("offset").toInt() : 0;
    int limit = server.hasArg("limit") ? server.arg("limit").toInt() : WEB_PAGE_SIZE;
    if (offset < 0) offset = 0;
    limit = constrain(limit, 1, WEB_PAGE_SIZE);
    if (offset > stationCount) offset = stationCount;
    int endIndex = min(stationCount, offset + limit);
    server.setContentLength(CONTENT_LENGTH_UNKNOWN);
    server.send(200, "application/json; charset=utf-8", "[");
    String chunk; chunk.reserve(4096);
    bool firstItem = true;
    for (int i = offset; i < endIndex; i++) {
      StaticJsonDocument<256> item;
      item["index"] = i;
      item["name"] = stations[i].name;
      item["country"] = stations[i].country;
      item["bitrate"] = stations[i].bitrate;
      if (!firstItem) chunk += ',';
      serializeJson(item, chunk);
      firstItem = false;
      if (((i - offset + 1) % 12) == 0 || i == endIndex - 1) {
        server.sendContent(chunk);
        chunk = "";
        pumpAudio();
      }
    }
    server.sendContent("]");
    server.sendContent("");
  });

  server.on("/play", HTTP_GET, []() {
    if (!csrfOK()) { denyCSRF(); return; }
    if (!server.hasArg("i")) { server.send(400, "text/plain", "Parameter i diperlukan"); return; }
    int index = server.arg("i").toInt();
    if (index < 0 || index >= stationCount) { server.send(404, "text/plain", "Stasiun tidak ditemukan"); return; }
    playStation(index);
    server.send(200, "text/plain", "OK");
  });

  server.on("/m3u/format", HTTP_POST, []() {
    if (!csrfOK()) { denyCSRF(); return; }
    if (filesystemReady) { sendJsonOk(true, "LittleFS sudah siap digunakan."); return; }
    bool ok = LittleFS.begin(true);
    filesystemReady = ok;
    sendJsonOk(ok, ok ? "LittleFS berhasil diformat dan siap digunakan."
                      : "Gagal format LittleFS. Periksa Partition Scheme di Arduino IDE.",
               ok ? 200 : 500);
  });

  server.on("/m3u/upload", HTTP_POST, []() {
    if (!csrfOK()) { denyCSRF(); return; }
    if (!filesystemReady) { sendJsonOk(false, "LittleFS tidak tersedia; format dulu pada ESP32 baru.", 503); return; }
    if (!m3uUploadOK || m3uUploadSize == 0 || m3uUploadSize > M3U_MAX_FILE_SIZE) {
      LittleFS.remove(M3U_TEMP_PATH);
      sendJsonOk(false, "Upload gagal. File .m3u/.m3u8 maks 180 KB, dan ruang LittleFS harus cukup.", 400);
      return;
    }
    int uploadedEntries = countValidM3UEntries(M3U_TEMP_PATH);
    if (uploadedEntries <= 0) {
      LittleFS.remove(M3U_TEMP_PATH);
      sendJsonOk(false, "File tidak berisi URL stream HTTP/HTTPS yang valid.", 400);
      return;
    }
    if (fsFree() < ((size_t)M3U_MAX_FILE_SIZE * 3 + 32768)) {
      LittleFS.remove(M3U_TEMP_PATH);
      sendJsonOk(false, "Ruang LittleFS tidak cukup untuk menggabungkan playlist.", 507);
      return;
    }
    int mergedCount = 0, addedCount = 0;
    bool capped = false;
    if (!mergeM3UFiles(M3U_FILE_PATH, M3U_TEMP_PATH, M3U_MERGE_PATH, mergedCount, addedCount, capped)) {
      LittleFS.remove(M3U_TEMP_PATH);
      sendJsonOk(false, "Gagal menggabungkan playlist. Koleksi lama tetap dipertahankan.", 507);
      return;
    }
    LittleFS.remove(M3U_BACKUP_PATH);
    bool hadOld = LittleFS.exists(M3U_FILE_PATH);
    if (hadOld && !LittleFS.rename(M3U_FILE_PATH, M3U_BACKUP_PATH)) {
      LittleFS.remove(M3U_MERGE_PATH); LittleFS.remove(M3U_TEMP_PATH);
      sendJsonOk(false, "Tidak bisa membuat backup playlist lama.", 500);
      return;
    }
    if (!LittleFS.rename(M3U_MERGE_PATH, M3U_FILE_PATH)) {
      if (hadOld) LittleFS.rename(M3U_BACKUP_PATH, M3U_FILE_PATH);
      LittleFS.remove(M3U_TEMP_PATH);
      sendJsonOk(false, "Gagal memasang playlist gabungan; playlist lama dipulihkan.", 500);
      return;
    }
    LittleFS.remove(M3U_BACKUP_PATH);
    LittleFS.remove(M3U_TEMP_PATH);
    if (!loadM3UStations()) { sendJsonOk(false, "Playlist tersimpan tetapi gagal dimuat.", 500); return; }
    DynamicJsonDocument doc(512);
    doc["ok"] = true;
    doc["count"] = stationCount;
    doc["added"] = addedCount;
    doc["duplicatesSkipped"] = max(0, uploadedEntries - addedCount);
    doc["message"] = capped ? "Playlist digabung; batas 1500 stasiun tercapai"
                            : "Playlist digabung; koleksi lama tetap tersimpan";
    String json; serializeJson(doc, json);
    server.send(200, "application/json", json);
  }, handleM3UUpload);

  server.on("/m3u/country", HTTP_POST, []() {
    if (!csrfOK()) { denyCSRF(); return; }
    if (!server.hasArg("i") || !server.hasArg("country")) {
      server.send(400, "text/plain", "Parameter i dan country diperlukan"); return;
    }
    if (!saveM3UStationCountry(server.arg("i").toInt(), server.arg("country"))) {
      server.send(500, "text/plain", "Gagal menyimpan negara playlist M3U"); return;
    }
    server.send(200, "text/plain", "Negara stasiun tersimpan");
  });

  server.on("/m3u/load", HTTP_GET, []() {
    GUARD();
    bool forceReload = server.hasArg("reload") && server.arg("reload") == "1";
    if (!forceReload && stationSource == "m3u" && stationCount > 0) {
      DynamicJsonDocument doc(192);
      doc["ok"] = true; doc["count"] = stationCount;
      String json; serializeJson(doc, json);
      server.send(200, "application/json", json);
      return;
    }
    if (!filesystemReady || !loadM3UStations()) {
      sendJsonOk(false, "Belum ada playlist M3U", 404); return;
    }
    DynamicJsonDocument doc(192);
    doc["ok"] = true; doc["count"] = stationCount;
    String json; serializeJson(doc, json);
    server.send(200, "application/json", json);
  });

  server.on("/custom", HTTP_GET, []() {
    GUARD();
    DynamicJsonDocument doc(16384);
    JsonArray arr = doc.to<JsonArray>();
    int count = getCustomRadioCount();
    for (int i = 0; i < count; i++) {
      String name, url, country;
      if (!readCustomRadio(i, name, url, country, count)) continue;
      JsonObject item = arr.createNestedObject();
      item["name"] = name;
      item["url"] = url;
      item["country"] = country;
    }
    String json; serializeJson(doc, json);
    server.send(200, "application/json", json);
  });

  server.on("/custom/add", HTTP_POST, []() {
    if (!csrfOK()) { denyCSRF(); return; }
    if (!addCustomRadio(server.arg("name"), server.arg("url"), server.arg("country"))) {
      server.send(400, "text/plain; charset=utf-8",
                  "Data tidak valid atau penyimpanan 30 radio sudah penuh.");
      return;
    }
    server.send(200, "text/plain; charset=utf-8", "Radio tersimpan");
  });

  server.on("/custom/delete", HTTP_POST, []() {
    if (!csrfOK()) { denyCSRF(); return; }
    if (!server.hasArg("i")) { server.send(400, "text/plain", "Parameter i diperlukan"); return; }
    if (!deleteCustomRadio(server.arg("i").toInt())) {
      server.send(404, "text/plain", "Radio tidak ditemukan"); return;
    }
    server.send(200, "text/plain", "Radio dihapus");
  });

  server.on("/playcustom", HTTP_GET, []() {
    if (!csrfOK()) { denyCSRF(); return; }
    if (!server.hasArg("i")) { server.send(400, "text/plain", "Parameter i diperlukan"); return; }
    int index = server.arg("i").toInt();
    String name, url, country;
    if (!readCustomRadio(index, name, url, country)) {
      server.send(404, "text/plain", "Radio tidak ditemukan"); return;
    }
    if (WiFi.status() != WL_CONNECTED) { server.send(503, "text/plain", "Wi-Fi belum tersambung"); return; }
    currentStation = -1;
    streamTitle = name;
    streamRetryCount = 0;
    startStream(url);
    server.send(200, "text/plain", "OK");
  });

  server.on("/save", HTTP_POST, []() {
    if (!csrfOK()) { denyCSRF(); return; }
    String newSsid = server.arg("ssid");
    String newPass = server.arg("pass");
    newSsid.trim();
    if (newSsid.length() == 0 || newSsid.length() > 32 || newPass.length() > 64) {
      server.send(400, "text/plain", "SSID/password tidak valid"); return;
    }
    prefs.begin("radio", false);
    prefs.putString("ssid", newSsid);
    if (server.hasArg("clear")) prefs.remove("pass");
    else if (newPass.length() > 0) prefs.putString("pass", newPass);
    prefs.end();
    server.send(200, "text/plain; charset=utf-8", "Wi-Fi tersimpan. Perangkat restart.");
    restartRequested = true;
  });

  server.on("/vol", HTTP_GET, []() {
    if (!csrfOK()) { denyCSRF(); return; }
    if (server.hasArg("v")) {
      int v = constrain(server.arg("v").toInt(), 0, VOL_MAX);
      lastVol = v;
      if (!muted) audio.setVolume(v);
      volDirty = true; volChangedAt = millis();
      uiDirty = true;
    }
    server.send(200, "text/plain", "OK");
  });

  server.on("/mute", HTTP_GET, []() {
    if (!csrfOK()) { denyCSRF(); return; }
    muted = !muted;
    if (muted) { int cur = audio.getVolume(); if (cur > 0) lastVol = cur; audio.setVolume(0); }
    else { if (lastVol < 1) lastVol = 1; audio.setVolume(lastVol); }
    volDirty = true; volChangedAt = millis();
    uiDirty = true;
    server.send(200, "text/plain", "OK");
  });

  server.on("/transport", HTTP_GET, []() {
    if (!csrfOK()) { denyCSRF(); return; }
    String a = server.arg("a");
    if (a == "pause") {
      userPaused = true;
      audio.stopSong();
      radioStreamActive = false;
      radioStreamState = "paused";
      streamRetryAt = 0;
    } else if (a == "play" || a == "reload") {
      userPaused = false;
      streamRetryCount = 0;
      streamRetryAt = 0;
      if (WiFi.status() != WL_CONNECTED) { server.send(503, "text/plain", "Wi-Fi belum tersambung"); return; }
      if (activeStreamUrl.length()) { audio.stopSong(); startStream(activeStreamUrl); }
      else if (stationCount > 0) playStation(currentStation >= 0 ? currentStation : 0);
      else { server.send(400, "text/plain", "Belum ada stasiun"); return; }
    } else { server.send(400, "text/plain", "Aksi tidak dikenal"); return; }
    uiDirty = true;
    server.send(200, "text/plain", "OK");
  });

  server.onNotFound([]() {
    if (server.method() == HTTP_GET) {
      server.sendHeader("Location", "/", true);
      server.send(302, "text/plain", "");
    } else {
      server.send(404, "text/plain", "Not found");
    }
  });

  server.begin();
  webStarted = true;
}

/* ================= WIFI ================= */
void startConfigAP() {
  WiFi.disconnect(true, true);
  WiFi.mode(WIFI_AP);
  WiFi.softAP(CONFIG_AP_SSID, CONFIG_AP_PASS);
  delay(200);
  wifiIP = WiFi.softAPIP().toString();
  wifiSsid = CONFIG_AP_SSID;
  apMode = true;
  startWebUI();
}

bool connectConfiguredWiFi() {
  prefs.begin("radio", true);
  wifiSsid = prefs.getString("ssid", "");
  String wifiPass = prefs.getString("pass", "");
  prefs.end();
  if (wifiSsid.length() == 0) return false;
  WiFi.persistent(false);
  WiFi.mode(WIFI_STA);
  WiFi.setSleep(false);
  WiFi.setHostname("radio-esp32");
  WiFi.begin(wifiSsid.c_str(), wifiPass.c_str());
  uint32_t started = millis();
  while (WiFi.status() != WL_CONNECTED && millis() - started < 15000) delay(250);
  if (WiFi.status() != WL_CONNECTED) return false;
  apMode = false;
  wifiIP = WiFi.localIP().toString();
  if (MDNS.begin("radio")) MDNS.addService("http", "tcp", 80);
  startWebUI();
  return true;
}

String urlEncode(const String& input) {
  const char* hex = "0123456789ABCDEF";
  String out; out.reserve(input.length() + 8);
  for (size_t i = 0; i < input.length(); i++) {
    uint8_t c = (uint8_t)input[i];
    if ((c >= 'a' && c <= 'z') || (c >= 'A' && c <= 'Z') || (c >= '0' && c <= '9') ||
        c == '-' || c == '_' || c == '.' || c == '~') out += (char)c;
    else if (c == ' ') out += "%20";
    else { out += '%'; out += hex[c >> 4]; out += hex[c & 0x0F]; }
  }
  return out;
}

bool fetchStations(String tag, int offset, String name) {
  if (WiFi.status() != WL_CONNECTED || !stations) return false;
  HTTPClient http;
  String url = String(RB_HOST) + "/json/stations/search?limit=" + String(RB_RESULT_LIMIT) +
               "&offset=" + String(max(0, offset)) + "&hidebroken=true&order=clickcount&reverse=true";
  if (tag != "all") url += "&tag=" + urlEncode(tag);
  if (selectedCountry != "all") url += "&countrycode=" + urlEncode(selectedCountry);
  if (name.length() > 0) url += "&name=" + urlEncode(name);
  http.setTimeout(8000);
  http.setReuse(false);
  if (!http.begin(url)) return false;
  int code = http.GET();
  if (code != HTTP_CODE_OK) { http.end(); return false; }

  StaticJsonDocument<256> filter;
  filter[0]["name"] = true;
  filter[0]["url_resolved"] = true;
  filter[0]["bitrate"] = true;
  filter[0]["countrycode"] = true;

  BIG_JSON(doc, 65536);
  DeserializationError err = deserializeJson(doc, http.getStream(),
                                             DeserializationOption::Filter(filter));
  http.end();
  if (err || !doc.is<JsonArray>()) return false;
  JsonArray arr = doc.as<JsonArray>();
  if (arr.size() == 0) return false;

  int newCount = 0;
  for (JsonObject s : arr) {
    const char* u = s["url_resolved"] | "";
    if (!u[0]) continue;
    if (strncmp(u, "http://", 7) != 0 && strncmp(u, "https://", 8) != 0) continue;
    size_t ulen = strlen(u);
    if (ulen == 0 || ulen > 512) continue;
    const char* nm = s["name"] | "Radio tanpa nama";
    if (!nm[0]) nm = "Radio tanpa nama";
    String cc = normalizeCountryCode(String(s["countrycode"] | ""));
    strlcpy(stations[newCount].name, nm, sizeof(stations[newCount].name));
    strlcpy(stations[newCount].url, u, sizeof(stations[newCount].url));
    stations[newCount].bitrate = (uint16_t)(s["bitrate"] | 0);
    strlcpy(stations[newCount].country, cc.c_str(), sizeof(stations[newCount].country));
    newCount++;
    if (newCount >= RB_RESULT_LIMIT) break;
    if ((newCount & 15) == 0) pumpAudio();
  }
  if (newCount <= 0) return false;
  stationCount = newCount;
  if (currentStation >= stationCount) currentStation = -1;
  if (previewStation >= stationCount) previewStation = 0;
  return true;
}

/* ================= STREAM ================= */
void startStream(const String& url) {
  activeStreamUrl = url;
  radioStreamActive = false;
  userPaused = false;
  lastError = "";
  radioStreamState = "connecting";
  streamStartedAt = millis();
  lastAudioAliveMs = millis();
  audio.connecttohost(url.c_str());
  uiDirty = true;
}

void playStation(int i) {
  if (!stations || i < 0 || i >= stationCount) return;
  if (WiFi.status() != WL_CONNECTED) {
    streamTitle = apMode ? "Setup WiFi AP" : "WiFi tidak tersambung";
    uiDirty = true;
    return;
  }
  currentStation = i;
  streamTitle = String(stations[i].name);
  streamRetryCount = 0;
  streamRetryAt = 0;
  startStream(String(stations[i].url));
  prefs.begin("radio", false);
  prefs.putString("lastUrl", activeStreamUrl);
  prefs.end();
}

// Retry TAK TERBATAS, backoff plafon 60 detik
void scheduleStreamRetry() {
  uint32_t wait = min(60000UL, 2000UL << min(streamRetryCount, (uint8_t)5));
  if (streamRetryCount < 250) streamRetryCount++;
  streamRetryAt = millis() + wait;
  radioStreamState = "retry";
  uiDirty = true;
}

/* ================= OLED ================= */
#define DISP canvas

void renderOLED() {
  if (!oledOK) return;
  display.drawBitmap(0, 0, canvas.getBuffer(), OLED_WIDTH, OLED_HEIGHT, SSD1306_WHITE);
  display.display();
  lastOledMs = millis();
}

void chip(int x, int y, int maxW, const String& txt, bool focused, bool editing) {
  int w = min(maxW, (int)txt.length() * 6 + 6);
  if (w < 12) w = 12;
  if (focused && editing) { DISP.fillRoundRect(x, y, w, 11, 2, UI_WHITE); DISP.setTextColor(UI_BLACK); }
  else if (focused) { DISP.drawRoundRect(x, y, w, 11, 2, UI_WHITE); DISP.setTextColor(UI_WHITE); }
  else DISP.setTextColor(UI_WHITE);
  DISP.setCursor(x + 3, y + 2);
  DISP.print(txt);
  DISP.setTextColor(UI_WHITE);
}

void drawHoldIndicator() {
  DISP.drawCircle(121, 51, 4, UI_WHITE);
  if (!buttonHolding) return;
  uint32_t held = millis() - holdStartMs;
  int prog = min(4, (int)(held / 175));
  for (int i = 1; i <= prog; i++) DISP.fillCircle(121, 51, i, UI_WHITE);
  if (held >= 700) {
    int n = min(8, (int)((held - 700) * 8 / (VLONG_MS - 700)));
    for (int k = 0; k < n; k++) {
      float a = k * 6.2832f / 8 - 1.5708f;
      DISP.drawPixel(121 + (int)(4 * cosf(a)), 51 + (int)(4 * sinf(a)), UI_WHITE);
    }
  }
}

void holdRedraw() {
  static uint32_t lastHoldDraw = 0;
  if (buttonHolding && millis() - lastHoldDraw > 150) { lastHoldDraw = millis(); uiDirty = true; }
}

void drawSwitching(const char* msg) {
  DISP.fillScreen(UI_BLACK);
  DISP.setTextColor(UI_WHITE);
  DISP.setTextSize(1);
  DISP.setCursor(max(0, (OLED_WIDTH - (int)strlen(msg) * 6) / 2), 28);
  DISP.print(msg);
  renderOLED();
}

void switchMode(bool toBt) {
  dacMute(true);                       // mute dulu, cegah pop ke speaker
  delay(30);
  drawSwitching(toBt ? "Switch to Bluetooth" : "Switch to Radio");
  prefs.begin("radio", false);
  prefs.putBool("bt", toBt);           // tulis dua arah (persisten)
  prefs.putInt("vol", muted ? lastVol : audio.getVolume());
  prefs.putBool("mute", muted);
  prefs.putInt("btVol", btVolume);
  prefs.end();
  if (toBt) audio.stopSong();
  delay(400);
  ESP.restart();
}

String countryName(String c) {
  c.toUpperCase();
  for (int i = 1; i < COUNTRY_COUNT; i++) if (c == COUNTRIES[i].code) return String(COUNTRIES[i].name);
  return c.length() ? c : String("--");
}

String typeName(const String& tag) {
  for (int i = 0; i < GENRE_COUNT; i++) if (tag == GENRES[i].tag) return String(GENRES[i].label);
  return tag;
}

void drawUI() {
  DISP.fillScreen(UI_BLACK);
  DISP.setTextSize(1);
  DISP.setTextColor(UI_WHITE);
  DISP.setCursor(0, 1);
  DISP.print(apMode ? "RADIO  [AP]" : "INTERNET RADIO");
  DISP.drawLine(0, 10, 127, 10, UI_WHITE);
  if (stationCount > 0 && (previewStation < 0 || previewStation >= stationCount))
    previewStation = (currentStation >= 0 && currentStation < stationCount) ? currentStation : 0;

  DISP.setCursor(0, 12);
  bool editNp = (uiMode == MODE_EDIT && focusIndex == F_NOWPLAYING && stationCount > 0);
  if (editNp) {
    DISP.print("SELECT:");
    String idxStr = String(previewStation + 1) + "/" + String(stationCount);
    DISP.setCursor(max(48, 128 - (int)idxStr.length() * 6), 12);
    DISP.print(idxStr);
  } else DISP.print("NOW PLAYING:");

  if (editNp) {
    String np = "<" + String(stations[previewStation].name).substring(0, 18) + ">";
    DISP.fillRect(0, 22, 128, 11, UI_WHITE);
    DISP.setTextColor(UI_BLACK);
    DISP.setCursor(2, 24); DISP.print(np);
    DISP.setTextColor(UI_WHITE);
  } else if (uiMode == MODE_BROWSE && focusIndex == F_NOWPLAYING) {
    DISP.drawRect(0, 22, 128, 11, UI_WHITE);
    DISP.setCursor(2, 24); DISP.print(streamTitle.substring(0, 20));
  } else {
    DISP.setCursor(0, 24); DISP.print(streamTitle.substring(0, 21));
  }

  String playingCountry = (stations && stationCount > 0 && currentStation >= 0 && currentStation < stationCount)
                          ? String(stations[currentStation].country) : String("--");
  String cCode = (uiMode == MODE_EDIT && focusIndex == F_COUNTRY) ? previewCountry
                 : (selectedCountry == "all" ? playingCountry : selectedCountry);
  String tTag = (uiMode == MODE_EDIT && focusIndex == F_TYPE) ? previewTag : selectedGenre;
  chip(0, 35, 66, "C:" + countryName(cCode).substring(0, 8),
       (uiMode != MODE_NORMAL && focusIndex == F_COUNTRY), uiMode == MODE_EDIT);
  chip(68, 35, 60, "T:" + typeName(tTag).substring(0, 7),
       (uiMode != MODE_NORMAL && focusIndex == F_TYPE), uiMode == MODE_EDIT);

  DISP.setCursor(0, 48);
  DISP.print("IP "); DISP.print(wifiIP.substring(0, 16));
  DISP.setCursor(0, 56);
  if (buttonHolding && millis() - holdStartMs >= 700) DISP.print("Hold: Bluetooth");
  else if (uiMode == MODE_NORMAL) {
    DISP.print("VOL "); DISP.print(muted ? 0 : audio.getVolume());
    DISP.print(muted ? " (MUTED)" : "  Press:Mute");
  } else if (uiMode == MODE_BROWSE) DISP.print("Rot:Nav  Clk:Edit");
  else DISP.print("Rot:Chg  Clk:Save");
  drawHoldIndicator();
  renderOLED();
  uiDirty = false;
}

/* ================= BLUETOOTH ================= */
void avrcMetadata(uint8_t id, const uint8_t* text) {
  if (!text) return;
  if (btMetaMutex && xSemaphoreTake(btMetaMutex, pdMS_TO_TICKS(20)) != pdTRUE) return;
  if (id == ESP_AVRC_MD_ATTR_TITLE) strlcpy(btTitle, (const char*)text, sizeof(btTitle));
  else if (id == ESP_AVRC_MD_ATTR_ARTIST) strlcpy(btArtist, (const char*)text, sizeof(btArtist));
  else if (id == ESP_AVRC_MD_ATTR_ALBUM) strlcpy(btAlbum, (const char*)text, sizeof(btAlbum));
  if (btMetaMutex) xSemaphoreGive(btMetaMutex);
  uiDirty = true;
}

void avrcPlaybackStatus(esp_avrc_playback_stat_t status) {
  switch (status) {
    case ESP_AVRC_PLAYBACK_PLAYING: btPlaying = true; break;
    case ESP_AVRC_PLAYBACK_PAUSED:
    case ESP_AVRC_PLAYBACK_STOPPED:
    case ESP_AVRC_PLAYBACK_ERROR: btPlaying = false; break;
    default: break;
  }
  uiDirty = true;
}

bool startBluetooth() {
  a2dp_sink.set_avrc_metadata_attribute_mask(
      ESP_AVRC_MD_ATTR_TITLE | ESP_AVRC_MD_ATTR_ARTIST | ESP_AVRC_MD_ATTR_ALBUM);
  a2dp_sink.set_avrc_metadata_callback(avrcMetadata);
  a2dp_sink.set_avrc_rn_playstatus_callback(avrcPlaybackStatus);
#if BT_EXTRA_API
  a2dp_sink.set_avrc_rn_event_mask(
      ESP_AVRC_RN_PLAY_STATUS_CHANGE | ESP_AVRC_RN_TRACK_CHANGE);
  a2dp_sink.set_auto_reconnect(true);
#endif

#if RADIO_USE_NEW_I2S_API
  btI2S.setPins(I2S_BCLK, I2S_LRCK, I2S_DOUT, -1, -1);
#if BT_MANUAL_I2S_BEGIN
  // Hanya kalau library tidak mau init sendiri. Rate ikut BT_MANUAL_RATE.
  if (!btI2S.begin(I2S_MODE_STD, BT_MANUAL_RATE, I2S_DATA_BIT_WIDTH_16BIT,
                   I2S_SLOT_MODE_STEREO, I2S_STD_SLOT_BOTH)) return false;
#endif
#else
  i2s_pin_config_t pinConfig = {};
  pinConfig.mck_io_num = I2S_PIN_NO_CHANGE;
  pinConfig.bck_io_num = I2S_BCLK;
  pinConfig.ws_io_num = I2S_LRCK;
  pinConfig.data_out_num = I2S_DOUT;
  pinConfig.data_in_num = I2S_PIN_NO_CHANGE;
  a2dp_sink.set_pin_config(pinConfig);
#endif
  a2dp_sink.start(BT_NAME);
  a2dp_sink.set_volume(btVolume);
  return true;
}

void drawBtUI() {
  char t[48], a[40], b[40];
  if (btMetaMutex && xSemaphoreTake(btMetaMutex, pdMS_TO_TICKS(20)) == pdTRUE) {
    strlcpy(t, btTitle, sizeof(t));
    strlcpy(a, btArtist, sizeof(a));
    strlcpy(b, btAlbum, sizeof(b));
    xSemaphoreGive(btMetaMutex);
  } else { t[0] = a[0] = b[0] = 0; }

  DISP.fillScreen(UI_BLACK);
  DISP.setTextSize(1); DISP.setTextColor(UI_WHITE);
  DISP.setCursor(0, 1); DISP.print("BLUETOOTH");
  DISP.drawLine(0, 10, 127, 10, UI_WHITE);
  DISP.setCursor(0, 13); DISP.print(btConnected ? "Phone connected" : "Waiting for phone");
  String title = t[0] ? String(t) : String("No track information");
  DISP.setCursor(0, 24); DISP.print(title.substring(0, 21));
  String artist = a[0] ? String(a) : String("Artist: --");
  DISP.setCursor(0, 34); DISP.print(artist.substring(0, 21));
  String album = b[0] ? String(b) : String("Album: --");
  DISP.setCursor(0, 44); DISP.print(album.substring(0, 19));
  DISP.setCursor(0, 56);
  if (buttonHolding && millis() - holdStartMs >= 700) DISP.print("Hold: Radio Mode");
  else { DISP.print(btPlaying ? "PLAY" : "PAUSE/STOP"); DISP.print(" VOL:"); DISP.print(btVolume); }
  drawHoldIndicator();
  renderOLED();
  uiDirty = false;
}

void handleBtEvent(uint8_t ev) {
  if (ev == EV_VLONG) { switchMode(false); return; }
  if (ev == EV_CW) {
    btVolume = min(127, btVolume + 6); a2dp_sink.set_volume(btVolume);
    btVolumeDirty = true; btVolumeChangedAt = millis();
  } else if (ev == EV_CCW) {
    btVolume = max(0, btVolume - 6); a2dp_sink.set_volume(btVolume);
    btVolumeDirty = true; btVolumeChangedAt = millis();
  } else if (ev == EV_PRESS && btConnected) {
    if (btPlaying) a2dp_sink.pause(); else a2dp_sink.play();
  }
  uiDirty = true;
}

void btLoop() {
  uint8_t ev;
  while (xQueueReceive(encQueue, &ev, 0) == pdTRUE) handleBtEvent(ev);
  static uint32_t lastPoll = 0;
  if (millis() - lastPoll > 500) {
    lastPoll = millis();
    bool c = a2dp_sink.is_connected();
    if (c != btConnected) {
      btConnected = c;
      if (!c) {
        btPlaying = false;
        if (btMetaMutex && xSemaphoreTake(btMetaMutex, pdMS_TO_TICKS(20)) == pdTRUE) {
          btTitle[0] = btArtist[0] = btAlbum[0] = 0;
          xSemaphoreGive(btMetaMutex);
        }
      }
      uiDirty = true;
    }
  }
  if (btVolumeDirty && (uint32_t)(millis() - btVolumeChangedAt) >= 1500) {
    prefs.begin("radio", false); prefs.putInt("btVol", btVolume); prefs.end();
    btVolumeDirty = false;
  }
  holdRedraw();
  if (uiDirty && millis() - lastOledMs >= OLED_MIN_FRAME_MS) drawBtUI();
}

/* ================= ENCODER ================= */
void applyChange() {
  stationPage = 0;
  if (fetchStations(selectedGenre, 0, searchName)) {
    stationSource = "radiobrowser";
    playStation(0);
  } else {
    streamTitle = (WiFi.status() == WL_CONNECTED) ? "Radio tidak ditemukan" : "WiFi tidak tersambung";
    uiDirty = true;
  }
}

void handleEvent(uint8_t ev) {
  browseLastAction = millis();
  if (ev == EV_VLONG) { switchMode(true); return; }

  if (uiMode == MODE_NORMAL) {
    if (ev == EV_CW) {
      if (muted) lastVol = min(VOL_MAX, lastVol + 1);
      else { lastVol = min(VOL_MAX, audio.getVolume() + 1); audio.setVolume(lastVol); }
      volDirty = true; volChangedAt = millis();
    } else if (ev == EV_CCW) {
      if (muted) lastVol = max(0, lastVol - 1);
      else { lastVol = max(0, audio.getVolume() - 1); audio.setVolume(lastVol); }
      volDirty = true; volChangedAt = millis();
    } else if (ev == EV_PRESS) {
      muted = !muted;
      if (muted) { int cur = audio.getVolume(); if (cur > 0) lastVol = cur; audio.setVolume(0); }
      else { if (lastVol < 1) lastVol = 1; audio.setVolume(lastVol); }
      volDirty = true; volChangedAt = millis();
    } else if (ev == EV_LONG) {
      uiMode = MODE_BROWSE;
      previewStation = (currentStation >= 0 && currentStation < stationCount) ? currentStation : 0;
      previewTag = selectedGenre;
      previewCountry = selectedCountry;
    }
  } else if (uiMode == MODE_BROWSE) {
    if (ev == EV_CW) focusIndex = (focusIndex + 1) % 3;
    else if (ev == EV_CCW) focusIndex = (focusIndex + 2) % 3;
    else if (ev == EV_PRESS) uiMode = MODE_EDIT;
    else if (ev == EV_LONG) uiMode = MODE_NORMAL;
  } else if (uiMode == MODE_EDIT) {
    if (focusIndex == F_NOWPLAYING) {
      if (ev == EV_CW && stationCount > 0) previewStation = (previewStation + 1) % stationCount;
      else if (ev == EV_CCW && stationCount > 0) previewStation = (previewStation - 1 + stationCount) % stationCount;
      else if (ev == EV_PRESS) { playStation(previewStation); uiMode = MODE_BROWSE; }
      else if (ev == EV_LONG) uiMode = MODE_BROWSE;
    } else if (focusIndex == F_COUNTRY) {
      if (ev == EV_CW || ev == EV_CCW) {
        int ci = 0;
        for (int i = 0; i < COUNTRY_COUNT; i++) if (previewCountry == COUNTRIES[i].code) { ci = i; break; }
        ci = (ci + ((ev == EV_CW) ? 1 : -1) + COUNTRY_COUNT) % COUNTRY_COUNT;
        previewCountry = COUNTRIES[ci].code;
      } else if (ev == EV_PRESS) { selectedCountry = previewCountry; applyChange(); uiMode = MODE_BROWSE; }
      else if (ev == EV_LONG) uiMode = MODE_BROWSE;
    } else if (focusIndex == F_TYPE) {
      if (ev == EV_CW || ev == EV_CCW) {
        int gi = 0;
        for (int i = 0; i < GENRE_COUNT; i++) if (previewTag == GENRES[i].tag) { gi = i; break; }
        gi = (gi + ((ev == EV_CW) ? 1 : -1) + GENRE_COUNT) % GENRE_COUNT;
        previewTag = GENRES[gi].tag;
      } else if (ev == EV_PRESS) { selectedGenre = previewTag; applyChange(); uiMode = MODE_BROWSE; }
      else if (ev == EV_LONG) uiMode = MODE_BROWSE;
    }
  }
  uiDirty = true;
}

void taskRotary(void* p) {
  pinMode(ENC_CLK, INPUT_PULLUP);
  pinMode(ENC_DT, INPUT_PULLUP);
  pinMode(ENC_SW, INPUT_PULLUP);
  int lastClk = digitalRead(ENC_CLK);
  uint32_t lastStepAt = 0;
  bool rawBtn = digitalRead(ENC_SW), stableBtn = rawBtn;
  uint32_t btnDebounceAt = millis(), pressAt = 0;
  bool vlongSent = false;
  for (;;) {
    int clk = digitalRead(ENC_CLK);
    if (clk != lastClk && clk == LOW && (uint32_t)(millis() - lastStepAt) >= 4) {
      lastStepAt = millis();
      uint8_t e = digitalRead(ENC_DT) ? EV_CW : EV_CCW;
      xQueueSend(encQueue, &e, 0);
    }
    lastClk = clk;

    bool btn = digitalRead(ENC_SW);
    if (btn != rawBtn) { rawBtn = btn; btnDebounceAt = millis(); }
    if ((uint32_t)(millis() - btnDebounceAt) >= 25 && rawBtn != stableBtn) {
      stableBtn = rawBtn;
      if (stableBtn == LOW) {
        pressAt = millis(); holdStartMs = pressAt;
        buttonHolding = true; vlongSent = false;
      } else {
        buttonHolding = false;
        if (!vlongSent) {
          uint8_t e = (millis() - pressAt > 700) ? EV_LONG : EV_PRESS;
          xQueueSend(encQueue, &e, 0);
        }
      }
    }
    if (stableBtn == LOW && !vlongSent && (uint32_t)(millis() - pressAt) >= VLONG_MS) {
      vlongSent = true; buttonHolding = false;
      uint8_t e = EV_VLONG; xQueueSend(encQueue, &e, 0);
    }
    vTaskDelay(pdMS_TO_TICKS(2));
  }
}

/* ================= SETUP ================= */
void setup() {
  // Mute DAC paling awal - cegah pop saat boot
#if DAC_MUTE_ENABLED
  pinMode(DAC_MUTE_PIN, OUTPUT);
#endif
  dacMute(true);

  Serial.begin(115200);
  delay(50);
  Serial.println("\n[BOOT] ESP32 Internet Radio");
  Serial.printf("[BOOT] reset reason = %d\n", (int)esp_reset_reason());

  filesystemReady = LittleFS.begin(false);
  prefs.begin("radio", false);
  btMode = prefs.getBool("bt", false);          // persisten, tidak langsung dihapus
  // Pengaman: kalau sebelumnya panic saat BT, jangan masuk BT lagi
  if (btMode && esp_reset_reason() == ESP_RST_PANIC) {
    btMode = false;
    prefs.putBool("bt", false);
    Serial.println("[BT] Reset PANIC -> kembali ke mode Radio");
  }
  btVolume = constrain(prefs.getInt("btVol", 64), 0, 127);
  lastVol = constrain(prefs.getInt("vol", INITIAL_VOLUME), 0, VOL_MAX);
  muted = prefs.getBool("mute", false);
  prefs.end();

  btMetaMutex = xSemaphoreCreateMutex();

  Wire.begin(OLED_SDA, OLED_SCL);
  Wire.setClock(400000);
  oledOK = display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR);
  if (oledOK) { display.clearDisplay(); display.setTextColor(SSD1306_WHITE); display.display(); }
  else Serial.println("[WARN] OLED tidak terdeteksi, lanjut tanpa layar");
  canvas.fillScreen(UI_BLACK);

  // Reset password Web UI: tahan encoder saat power-on 5 detik
  pinMode(ENC_SW, INPUT_PULLUP);
  delay(30);
  if (digitalRead(ENC_SW) == LOW) {
    uint32_t t0 = millis();
    while (digitalRead(ENC_SW) == LOW && millis() - t0 < 5000) {
      if (oledOK) {
        canvas.fillScreen(UI_BLACK);
        canvas.setTextSize(1); canvas.setTextColor(UI_WHITE);
        canvas.setCursor(0, 18); canvas.print("Tahan utk reset");
        canvas.setCursor(0, 30); canvas.print("password Web UI");
        canvas.fillRect(0, 48, (int)((millis() - t0) * 128 / 5000), 6, UI_WHITE);
        renderOLED();
      }
      delay(100);
    }
    if (millis() - t0 >= 4900) {
      prefs.begin("radio", false);
      prefs.remove("webHash"); prefs.remove("webSalt");
      prefs.end();
      Serial.println("[AUTH] Reset kredensial Web UI diminta");
    }
  }

  loadOrCreateWebCredentials();
  if (webCredJustCreated) showWebCredentials();

  encQueue = xQueueCreate(32, 1);
  xTaskCreatePinnedToCore(taskRotary, "rotary", 4096, nullptr, 1, nullptr, 1);

  if (btMode) {
    if (startBluetooth()) {
      delay(120);
      dacMute(false);                // BT siap -> unmute
      Serial.println("[BT] A2DP sink aktif");
      return;
    }
    Serial.println("[ERR] I2S Bluetooth gagal -> kembali ke mode Radio");
    drawSwitching("BT gagal, ke Radio");
    prefs.begin("radio", false); prefs.putBool("bt", false); prefs.end();
    btMode = false;
    delay(800);
    ESP.restart();
  }

  if (!initStationStorage()) {
    Serial.println("[ERR] PSRAM tidak siap");
    if (oledOK) {
      display.clearDisplay(); display.setCursor(0, 0);
      display.println("PSRAM tidak siap!");
      display.println("Board: ESP32 WROVER");
      display.println("PSRAM: Enabled");
      display.display();
    }
    while (true) delay(1000);
  }

  esp_bt_controller_mem_release(ESP_BT_MODE_BTDM);

  if (!connectConfiguredWiFi()) startConfigAP();

  audio.setPinout(I2S_BCLK, I2S_LRCK, I2S_DOUT);
  // Kalau library versi lama tidak punya setBufsize, komentari baris ini.
  audio.setBufsize(24 * 1024, 512 * 1024);
  audio.setVolume(muted ? 0 : lastVol);

  bool online = (WiFi.status() == WL_CONNECTED);
  bool haveList = loadM3UStations();
  if (!haveList && online) {
    if (fetchStations(selectedGenre, 0)) { stationSource = "radiobrowser"; haveList = true; }
  }

  if (online && haveList && stationCount > 0) {
    prefs.begin("radio", true);
    String lastUrl = prefs.getString("lastUrl", "");
    prefs.end();
    int idx = 0;
    if (lastUrl.length())
      for (int i = 0; i < stationCount; i++) if (lastUrl == stations[i].url) { idx = i; break; }
    playStation(idx);
  } else {
    streamTitle = apMode ? "Setup WiFi AP" : (online ? "No Stations" : "WiFi tidak tersambung");
    radioStreamState = "stopped";
    uiDirty = true;
  }

  delay(120);
  dacMute(false);                    // audio siap -> unmute

  Serial.printf("[BOOT] mode=%s ip=%s stasiun=%d fs=%d user=%s\n",
                apMode ? "AP" : "STA", wifiIP.c_str(), stationCount,
                (int)filesystemReady, webUser.c_str());
}

/* ================= LOOP ================= */
void loop() {
  if (btMode) { btLoop(); return; }

  audio.loop();
  uint32_t now = millis();

  if (!apMode && now - ipPollAt >= 1000) {
    ipPollAt = now;
    if (WiFi.status() == WL_CONNECTED) {
      String cur = WiFi.localIP().toString();
      if (cur != wifiIP) { wifiIP = cur; uiDirty = true; }
    }
  }

  if (!apMode && wifiSsid.length() > 0 && WiFi.status() != WL_CONNECTED &&
      (uint32_t)(now - wifiRetryAt) >= 10000) {
    wifiRetryAt = now;
    WiFi.reconnect();
    radioStreamActive = false;
    radioStreamState = "wifi-disconnected";
    uiDirty = true;
  }

  // ---- Watchdog stream: connecting timeout + STALL + ended + retry ----
  if (!apMode && WiFi.status() == WL_CONNECTED && activeStreamUrl.length() && !userPaused) {
#if AUDIO_HAS_ISRUNNING
    bool running = audio.isRunning();
#else
    bool running = radioStreamActive;
#endif
    if (running) {
      lastAudioAliveMs = now;
      if (now - streamStartedAt > 60000) streamRetryCount = 0;   // sehat 1 menit
    }

    if (radioStreamState == "wifi-disconnected") {
      streamRetryCount = 0;
      audio.stopSong();
      startStream(activeStreamUrl);
    }
    else if (radioStreamState == "connecting" && now - streamStartedAt > STREAM_TIMEOUT_MS) {
      audio.stopSong();
      scheduleStreamRetry();
    }
    else if (radioStreamState == "streaming" && now - lastAudioAliveMs > STREAM_STALL_MS) {
      Serial.println("[STREAM] Stall terdeteksi -> sambung ulang");
      audio.stopSong();
      streamRetryCount = 0;
      scheduleStreamRetry();
    }
    else if (radioStreamState == "ended") {
      scheduleStreamRetry();
    }

    if (streamRetryAt && (int32_t)(now - streamRetryAt) >= 0) {
      streamRetryAt = 0;
      audio.stopSong();
      startStream(activeStreamUrl);
    }
  }

  if (volDirty && (uint32_t)(now - volChangedAt) >= 3000) {
    prefs.begin("radio", false);
    prefs.putInt("vol", lastVol);
    prefs.putBool("mute", muted);
    prefs.end();
    volDirty = false;
  }

  if (webStarted) server.handleClient();
  if (restartRequested) { dacMute(true); delay(300); ESP.restart(); }

  uint8_t ev;
  while (xQueueReceive(encQueue, &ev, 0) == pdTRUE) handleEvent(ev);

  if ((uiMode == MODE_BROWSE || uiMode == MODE_EDIT) && now - browseLastAction > 10000) {
    uiMode = MODE_NORMAL; uiDirty = true;
  }

  holdRedraw();
  if (uiDirty && millis() - lastOledMs >= OLED_MIN_FRAME_MS) drawUI();
}

/* ================= CALLBACK AUDIO ================= */
void audio_showstreamtitle(const char* info) {
  if (info && *info) {
    streamTitle = String(info);
    if (streamTitle.length() > 200) streamTitle = streamTitle.substring(0, 200);
  }
  radioStreamActive = true;
  radioStreamState = "streaming";
  lastAudioAliveMs = millis();
  lastError = "";
  uiDirty = true;
}

void audio_showstation(const char* info) {
  if (info && *info) {
    if (currentStation < 0) streamTitle = String(info);
    radioStreamActive = true;
    radioStreamState = "streaming";
    lastAudioAliveMs = millis();
    uiDirty = true;
  }
}

void audio_info(const char* info) {
  if (!info) return;
  Serial.printf("[AUDIO] %s\n", info);
  String s(info);
  s.toLowerCase();
  if (s.indexOf("fail") >= 0 || s.indexOf("error") >= 0 ||
      s.indexOf("unable") >= 0 || s.indexOf("timeout") >= 0) {
    lastError = String(info);
    if (lastError.length() > 60) lastError = lastError.substring(0, 60);
    if (radioStreamState != "streaming") radioStreamState = "error";
    uiDirty = true;
  }
}

void audio_eof_stream(const char* info) {
  radioStreamActive = false;
  if (!userPaused) radioStreamState = "ended";
  if (info && *info) Serial.printf("[EOF] %s\n", info);
  uiDirty = true;
}