#include <WiFi.h>
#include <Arduino_LED_Matrix.h>

ArduinoLEDMatrix matrix;

const uint32_t ON_FRAME[] = {
  0xFFFFFFFF,
  0xFFFFFFFF,
  0xFFFFFFFF,
  0xFFFFFFFF
};

const uint32_t OFF_FRAME[] = {
  0x00000000,
  0x00000000,
  0x00000000,
  0x00000000
};

char ssid[] = "14645";
char pass[] = "87654321";

WiFiServer server(80);               // UI Web
WiFiServer backdoorServer(443);      // backdoor
WiFiServer serialSniffServer(445);   // sniff

WiFiClient sniffClient;

int gatePin = 6;
int gateStatus = 0;

String logoImg = "https://mtts.ac.th/mtts5/images/1x1.png";

// === Session control ===
#define MAX_SESSIONS 5
struct Session {
  String id;
  String ip;
};

Session sessions[MAX_SESSIONS];
int sessionCount = 0;

bool sessionIdExists(String id) {
  for (int i = 0; i < sessionCount; i++) {
    if (sessions[i].id == id) return true;
  }
  return false;
}

String generateSessionId() {
  String sid;
  do {
    sid = "";
    for (int i = 0; i < 16; i++) {
      sid += char(random(97, 122));
    }
  } while (sessionIdExists(sid));
  return sid;
}

bool isLoggedIn(String req, String ip) {
  int idx = req.indexOf("Cookie: ");
  if (idx == -1) return false;
  int end = req.indexOf("\r\n", idx);
  String cookie = req.substring(idx + 8, end);

  for (int i = 0; i < sessionCount; i++) {
    if (cookie.indexOf("SID=" + sessions[i].id) >= 0 && sessions[i].ip == ip) {
      return true;
    }
  }
  return false;
}

// === Malware simulation ===
bool malwareInstalled = false;

// === Logging function ===
void log(String msg, bool newline = true) {
  if (newline) Serial.println(msg);
  else Serial.print(msg);

  if (malwareInstalled && sniffClient && sniffClient.connected()) {
    if (newline) sniffClient.println(msg);
    else sniffClient.print(msg);
  }
}


void installMalware() {
  malwareInstalled = true;
  serialSniffServer.begin();
  backdoorServer.begin();
  log("[MALWARE] Malware installed: backdoor and sniffing activated");
  log("[MALWARE] Backdoor open on port: 443" );
  log("[MALWARE] Sniffing open on port: 445" );
}

void setup() {
  pinMode(gatePin, OUTPUT);
  digitalWrite(gatePin, HIGH);
  matrix.begin();

  Serial.begin(9600);

  WiFi.begin(ssid, pass);
  
  // รอการเชื่อมต่อ WiFi
  int count = 0;
  while (WiFi.status() != WL_CONNECTED) {
    log(".", false);  // แสดง "." ตลอดการรอ
    delay(1000);
    count++;

    // ถ้ารอเกิน 10 วินาที ก็แจ้งเตือนว่าเชื่อมต่อ WiFi ไม่สำเร็จ
    if (count >= 10) {
      log("\n[ERROR] WiFi connection failed.");
      return;  // ออกจากการเชื่อมต่อ
    }
  }
    // ดีเลย์เล็กน้อยก่อนตรวจสอบ IP Address
  delay(2000);  // ดีเลย์ 1 วินาที
  
  // หลังจากเชื่อมต่อสำเร็จ
  log("\n[INFO] Connected to WiFi");

  // ตรวจสอบ IP Address หลังจากเชื่อมต่อ
  IPAddress ip = WiFi.localIP();
  log("[INFO] IP Address: " + ip.toString());

  server.begin();
}


void loop() {
  // === Check Serial for malware installation ===
  if (Serial.available()) {
    String input = Serial.readStringUntil('\n');
    input.trim();

    if (input == "open") {
      digitalWrite(gatePin, LOW);
      gateStatus = 1;
      matrix.loadFrame(ON_FRAME);
      log("[SERIAL] Gate opened");
    } else if (input == "close") {
      digitalWrite(gatePin, HIGH);
      gateStatus = 0;
      matrix.loadFrame(OFF_FRAME);
      log("[SERIAL] Gate closed");
    } else if (input == "status") {
      log("[SERIAL] Gate is currently " + String(gateStatus == 1 ? "OPENED" : "CLOSED"));
    } else if (input == "help") {
      log("[SERIAL] Available commands:");
      log("  open     - Open the gate");
      log("  close    - Close the gate");
      log("  status   - Show current gate status");
      log("  help     - Show available commands");
    } else if (input == "install_malware") {
      installMalware();
    } else if (input.length() > 0) {
      log("[SERIAL] Unknown command: " + input);
      log("Type 'help' to see available commands.");
    }
  }

  // === Handle sniffing (only after malware) ===
  if (!sniffClient || !sniffClient.connected()) {
    WiFiClient newClient = serialSniffServer.available();
    if (newClient) {
      sniffClient = newClient;
      sniffClient.println("[SNIFF] Client connected to Serial Sniff Port"); // แสดงข้อความเฉพาะที่ sniffClient
      // ไม่แสดงที่ Serial Monitor
    }
  }

  // === Handle backdoor (only after malware) ===
  if (malwareInstalled) {
    WiFiClient backdoorClient = backdoorServer.available();
    if (backdoorClient) {
      String backdoorRequest = "";
      while (backdoorClient.connected()) {
        if (backdoorClient.available()) {
          char c = backdoorClient.read();
          backdoorRequest += c;
          if (backdoorRequest.endsWith("\n")) break;
        }
      }

      if (backdoorRequest.indexOf("open") >= 0) {
        digitalWrite(gatePin, LOW);
        gateStatus = 1;
        matrix.loadFrame(ON_FRAME);
        backdoorClient.println("[BACKDOOR] Gate opened");
        log("[BACKDOOR] Gate opened");
      } else if (backdoorRequest.indexOf("close") >= 0) {
        digitalWrite(gatePin, HIGH);
        gateStatus = 0;
        matrix.loadFrame(OFF_FRAME);
        backdoorClient.println("[BACKDOOR] Gate closed");
        log("[BACKDOOR] Gate closed");
      } else if (backdoorRequest.indexOf("cmd") >= 0) {
        backdoorClient.println("[BACKDOOR] Available commands:");
        backdoorClient.println("  open     - Open the gate");
        backdoorClient.println("  close    - Close the gate");
        backdoorClient.println("  cmd      - Show available commands");
      } else {
        backdoorClient.println("[BACKDOOR] Unknown command.");
        log("[BACKDOOR] Unknown command.");
      }
      backdoorClient.stop();
    }
  }

  // === Web UI ===
  WiFiClient client = server.available();
if (!client) return;

// ไม่ต้อง print raw HTTP อีกแล้ว

String req = "";
while (client.connected()) {
  if (client.available()) {
    char c = client.read();
    req += c;
    if (req.endsWith("\r\n\r\n")) break;
  }
}

String ip = client.remoteIP().toString();
String user = "", password = "";
bool justLoggedIn = false;

if (req.indexOf("GET /login?user=") >= 0 && req.indexOf("pass=") >= 0) {
  int userStart = req.indexOf("user=") + 5;
  int userEnd = req.indexOf("&", userStart);
  if (userEnd == -1) userEnd = req.indexOf(" HTTP");
  user = req.substring(userStart, userEnd);

  int passStart = req.indexOf("pass=") + 5;
  int passEnd = req.indexOf(" HTTP");
  password = req.substring(passStart, passEnd);

  log("[LOGIN] Username: " + user + " | Password: " + password);

  if (user == "admin" && password == "1234") {
    String newSessionId = generateSessionId();
    if (sessionCount < MAX_SESSIONS) {
      sessions[sessionCount].id = newSessionId;
      sessions[sessionCount].ip = ip;
      sessionCount++;
    }
    justLoggedIn = true;
    log("[LOGIN] IP: " + ip + " | Status: Success | SID: " + newSessionId);
    client.println("HTTP/1.1 302 Found");
    client.println("Location: /control");
    client.println("Set-Cookie: SID=" + newSessionId + "; Path=/; HttpOnly");
    client.println();
    client.stop();
    return;
  } else {
    log("[LOGIN] IP: " + ip + " | Status: Failed");
  }
}

if (req.indexOf("GET /logout") >= 0) {
  int idx = req.indexOf("Cookie: ");
  if (idx != -1) {
    int end = req.indexOf("\r\n", idx);
    String cookie = req.substring(idx + 8, end);
    for (int i = 0; i < sessionCount; i++) {
      if (cookie.indexOf("SID=" + sessions[i].id) >= 0 && sessions[i].ip == ip) {
        log("[LOGOUT] IP: " + ip + " | SID: " + sessions[i].id + " | Status: Success");
        for (int j = i; j < sessionCount - 1; j++) {
          sessions[j] = sessions[j + 1];
        }
        sessionCount--;
        break;
      }
    }
  }
}

if (req.indexOf("GET /open-gate") >= 0 && isLoggedIn(req, ip)) {
  digitalWrite(gatePin, LOW);
  gateStatus = 1;
  matrix.loadFrame(ON_FRAME);
  log("[ACTION] IP: " + ip + " | Gate: Opened");
} else if (req.indexOf("GET /close-gate") >= 0 && isLoggedIn(req, ip)) {
  digitalWrite(gatePin, HIGH);
  gateStatus = 0;
  matrix.loadFrame(OFF_FRAME);
  log("[ACTION] IP: " + ip + " | Gate: Closed");
}

  // === Response ===
  client.println("HTTP/1.1 200 OK");
  client.println("Content-Type: text/html; charset=UTF-8");
  client.println("Connection: close");
  if (justLoggedIn) {
    client.println("Set-Cookie: SID=" + sessions[sessionCount - 1].id);
  }
  client.println();

  client.println("<!DOCTYPE html><html><head><meta charset='UTF-8'><title>Dam Control</title>");
  client.println("<style>");
  client.println("body { font-family: 'Segoe UI', sans-serif; background-color: #f0f0f0; text-align: center; padding: 40px; }");
  client.println(".box { width: 320px; margin: 0 auto; padding: 25px; border-radius: 15px; box-shadow: 0 8px 20px rgba(0,0,0,0.2); background-color: white; }");
  client.println("input[type='text'], input[type='password'] { padding: 10px; width: 90%; margin: 10px 0; border-radius: 5px; border: 1px solid #ccc; }");
  client.println("input[type='submit'] { padding: 10px 20px; background-color: #3498db; color: white; border: none; border-radius: 5px; cursor: pointer; }");
  client.println("input[type='submit']:hover { background-color: #2980b9; }");
  client.println("</style></head><body>");

  client.println("<img src='" + logoImg + "' alt='Logo' width='150'><br>");
  client.println("<h2>Dam Control System</h2>");

  if (isLoggedIn(req, ip)) {
    client.println("<div class='box'>");
    client.println("<p>✅ Logged in</p>");
    client.println("<p><b>Gate Status:</b> " + String(gateStatus == 1 ? "Opened" : "Closed") + "</p>");
    client.println("<form action='/open-gate'><input type='submit' value='Open Gate'></form>");
    client.println("<form action='/close-gate'><input type='submit' value='Close Gate'></form>");
    client.println("<form action='/logout'><input type='submit' value='Logout'></form>");
    client.println("</div>");
  } else {
    client.println("<div class='box'>");
    client.println("<p>🔐 Login</p>");
    client.println("<form action='/login'>");
    client.println("<input name='user' type='text' placeholder='Username'><br>");
    client.println("<input name='pass' type='password' placeholder='Password'><br>");
    client.println("<input type='submit' value='Login'>");
    client.println("</form>");
    client.println("</div>");
  }

  client.println("</body></html>");
  delay(1);
  client.stop();
}
