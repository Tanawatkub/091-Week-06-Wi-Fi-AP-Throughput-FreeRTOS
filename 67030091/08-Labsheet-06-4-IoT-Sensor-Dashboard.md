# ใบงานที่ 6.4: IoT Sensor Dashboard — แสดงผลค่าเซนเซอร์แบบ Real-Time ผ่าน Web Browser บนมือถือ

## 0. กล่าวนำ (Introduction)

ในใบงาน 6.3 นักศึกษาได้สร้างระบบ FreeRTOS Multi-Tasking ที่ `vSensorTask` อ่านค่าเซนเซอร์ → ส่งผ่าน Queue → `vNetworkTask` รับและ "เตรียม JSON" แต่ข้อมูลยังไม่ได้ออกไปสู่โลกภายนอก

ในใบงานนี้ นักศึกษาจะ**ต่อยอดโค้ด Lab 6-3 โดยตรง** โดยเพิ่ม
1. **ESP32 SoftAP** — ให้มือถือเชื่อมต่อ Wi-Fi ตรงโดยไม่ต้อง Router
2. **HTTP Web Server (`esp_http_server`)** — เปิด Endpoint 2 ตัว
   - `GET /` → หน้า Dashboard HTML Auto-refresh ทุก 2 วินาที
   - `GET /api/data` → ส่งค่า JSON ล่าสุดให้ Browser

---

## 1. วัตถุประสงค์ (Objectives)

1. เชื่อมโยง FreeRTOS Queue Pipeline กับ HTTP Web Server เพื่อส่งข้อมูลออกสู่ Browser จริง
2. ใช้งาน `esp_http_server` component ของ ESP-IDF ในการสร้าง REST API Endpoint บน ESP32
3. ออกแบบ ESP32 ให้ทำงานเป็น **SoftAP + HTTP Server** พร้อมกัน
4. เข้าใจการใช้ `SemaphoreHandle_t` (Mutex) เพื่อป้องกัน Race Condition เมื่อ HTTP Handler และ FreeRTOS Task แชร์ข้อมูลร่วมกัน

---

## 2. อุปกรณ์และซอฟต์แวร์ที่ใช้ในการทดลอง (Equipment & Tools)

1. บอร์ดไมโครคอนโทรลเลอร์ ESP32 จำนวน 1 บอร์ด
2. สายเชื่อมต่อ USB จำนวน 1 เส้น
3. สมาร์ตโฟนหรือ PC (สำหรับเปิด Browser ดู Dashboard)

---

## 3. สถาปัตยกรรมระบบ (System Architecture)

```mermaid
sequenceDiagram
    autonumber
    participant Sens as vSensorTask
    participant Queue as FreeRTOS Queue
    participant Net as vNetworkTask
    participant HTTP as HTTP Server (esp_http_server)
    participant Phone as Browser (มือถือ)

    loop ทุกๆ 1.5 วินาที
        Sens->>Queue: xQueueSend(&sensor_data)
        Queue->>Net: xQueueReceive(&rx_data)
        Net->>Net: Mutex Lock → อัพเดต g_latest_data → Mutex Unlock
    end

    Phone->>HTTP: GET /api/data
    HTTP->>Net: Mutex Lock → อ่าน g_latest_data → Mutex Unlock
    HTTP-->>Phone: JSON: {temp, hum, lux, ts}

    Phone->>HTTP: GET /
    HTTP-->>Phone: HTML Dashboard (Auto-refresh ทุก 2s)
```

---

## 4. แนวคิดสำคัญ: Mutex ป้องกัน Race Condition

```
vNetworkTask                    HTTP GET Handler
─────────────────────────       ─────────────────────────
xSemaphoreTake(mutex)           xSemaphoreTake(mutex)
  g_latest_data = rx_data;         read g_latest_data
xSemaphoreGive(mutex)           xSemaphoreGive(mutex)
```

> [!WARNING]
> หากไม่ใช้ Mutex: HTTP Handler อาจอ่านข้อมูลขณะที่ `vNetworkTask` กำลังเขียนอยู่ ทำให้ได้ค่าที่ไม่สมบูรณ์ (Torn Read)

---

## 5. ซอร์สโค้ดการทดลอง (`main/main.c`)

ดูใน `ESP32_Project/Lab6-4-IoT-Sensor-Dashboard/main/main.c`

---

## 6. ขั้นตอนการทดลอง (Experimental Procedures)

1. Build และ Flash โค้ดลงบอร์ด ESP32
2. เปิด Serial Monitor ดู SSID และยืนยัน `[HTTP SERVER]: Started`
3. ใช้มือถือ **เชื่อมต่อ Wi-Fi ชื่อ `MY_ESP32_SENSOR_AP`** (Password: `12345678`)
4. เปิด Browser บนมือถือ แล้วไปที่ `http://192.168.4.1`
5. ควรเห็นหน้า Dashboard แสดง Temperature / Humidity / Light Lux และ Auto-refresh ทุก 2 วินาที
6. ทดสอบ JSON API โดยเปิด `http://192.168.4.1/api/data` ดู Raw JSON

ตัวอย่างหน้า browser
![](images/253776.jpg)
![alt text](IMG_4683.PNG) ![alt text](IMG_4685.PNG) ![alt text](IMG_4686.PNG)

---

## 7. ตารางบันทึกผลการทดลอง (Experiment Results)

### 7.1 บันทึกข้อมูลจาก Dashboard

| ครั้งที่ | Temperature (°C) | Humidity (%) | Light Lux | Timestamp (ms) |
| :------: | :--------------: | :----------: | :-------: | :------------: |
|  **1**   |        34.5          |       65.0       |    686       |        91280        |
|  **2**   |          29.7        |      53.1        |     489      |       168300         |
|  **3**   |        25.5          |       66.5       |     685      |       172830         |


### 7.2 ทดสอบ JSON API (`/api/data`)

บันทึก Raw JSON Response จาก Browser:
![alt text](image-1.png)

```json
{"temperature":33.30,"humidity":66.20,"light_lux":245,"timestamp_ms":360070}

```

---

## 8. คำถามท้ายการทดลอง (Post-Lab Questions)


# 1. เหตุใดจึงต้องใช้ **Mutex** ในการป้องกันการเข้าถึงตัวแปร `g_latest_data` ร่วมกันระหว่าง `vNetworkTask` และ HTTP Handler? ถ้าไม่ใช้จะเกิดอะไรขึ้น?

**ตอบ:**
จำเป็นต้องใช้ Mutex เพราะระบบทำงานแบบ Multi-Tasking (Asynchronous) 
* **ถ้าไม่ใช้ Mutex จะเกิดปัญหาที่เรียกว่า "Torn Read" (อ่านข้อมูลได้ไม่สมบูรณ์):** สมมติว่า `vNetworkTask` กำลังอัปเดตข้อมูลเซนเซอร์ชุดใหม่ลงใน `g_latest_data` (เช่น เขียนอุณหภูมิเสร็จแล้ว แต่ยังไม่ทันเขียนความชื้น) แล้วจังหวะนั้นมีผู้ใช้กด Refresh หน้าเว็บ ทำให้ HTTP Handler เข้ามาดึงข้อมูลไปแสดงผลพอดี ผลลัพธ์คือผู้ใช้จะได้ข้อมูลที่ผสมกันระหว่างชุดเก่าและชุดใหม่ ซึ่งไม่ถูกต้อง
* **เมื่อใช้ Mutex:** จะเป็นการรับประกันว่าการเขียนหรืออ่านโครงสร้างข้อมูล `g_latest_data` จะต้องทำจนเสร็จครบถ้วนก่อน Task อื่นถึงจะเข้ามาใช้งานต่อได้ ป้องกันปัญหาข้อมูลบิดเบือน

---

# 2. `esp_http_server` รัน Handler บน Thread ใด — เป็น Thread เดียวกับ FreeRTOS Task ของเราหรือไม่?

**ตอบ:**
`esp_http_server` **ไม่ได้รันอยู่บน Thread (Task) เดียวกับ `vSensorTask` หรือ `vNetworkTask` ของเรา** 
เมื่อเราเรียกใช้คำสั่ง `httpd_start()` ระบบของ ESP-IDF จะทำการสร้าง FreeRTOS Task เบื้องหลังขึ้นมาใหม่โดยอัตโนมัติ (มักจะมีชื่อ Task เช่น `httpd`) เพื่อทำหน้าที่เป็น Web Server คอยดักฟัง (Listen) และจัดการ HTTP Request จาก Client โดยเฉพาะ ด้วยเหตุนี้ Handler ที่เราเขียนขึ้นจึงถูกเรียกใช้งานจาก Task ของ Web Server ซึ่งแยกขาดจาก Task หลักที่เราสร้างเอง (นี่จึงเป็นเหตุผลตอกย้ำว่าทำไมข้อ 1 จึงต้องใช้ Mutex ในการแชร์ข้อมูลข้าม Task)

---

# 3. การที่ Dashboard ใช้ `<meta http-equiv="refresh" content="2">` แทนที่จะใช้ JavaScript `fetch()` มีข้อดีและข้อเสียอย่างไร?

**ตอบ:**
* **ข้อดี (Pros):**
  1. **ใช้งานง่ายมาก:** เขียนเพียงบรรทัดเดียวใน HTML ก็ทำงานได้ทันที โดยไม่ต้องมีความรู้หรือเขียนโค้ด JavaScript เลย
  2. **กินทรัพยากรฝั่ง Client น้อย:** เบราว์เซอร์รุ่นเก่าหรืออุปกรณ์ที่มีข้อจำกัดก็สามารถรันได้โดยไม่มีปัญหาความเข้ากันได้ (Compatibility)

* **ข้อเสีย (Cons):**
  1. **หน้าเว็บกระตุก (Flickering):** การใช้ Meta Refresh คือการโหลดหน้าเว็บใหม่ทั้งหมด (Full Page Reload) ทำให้หน้าจอกระพริบทุกๆ 2 วินาที ประสบการณ์ใช้งาน (UX) ไม่ลื่นไหล
  2. **สิ้นเปลือง Bandwidth และทรัพยากร ESP32:** ทุกๆ 2 วินาที ESP32 จะต้องส่งโค้ด HTML ทั้งหน้ากลับไปให้ Browser ใหม่ทั้งหมด แทนที่จะส่งไปแค่เฉพาะตัวเลขข้อมูล (JSON) ทำให้ ESP32 ทำงานหนักขึ้นและตอบสนองได้ช้าลงหากมีผู้ใช้งานหลายคนพร้อมกัน ต่างจาก `fetch()` ที่ส่งเฉพาะข้อมูล (Data Payload) ทำให้ประหยัด Bandwidth กว่ามาก


---

## 9. ความรู้เพิ่มเติม: ESP-IDF `esp_http_server` API

| ฟังก์ชัน                                   | ความหมาย                                     |
| :----------------------------------------- | :------------------------------------------- |
| `httpd_start(&server, &config)`            | เริ่มต้น HTTP Server (เปิด Port 80)          |
| `httpd_register_uri_handler(server, &uri)` | ลงทะเบียน Handler สำหรับ URL path            |
| `httpd_resp_send(req, buf, len)`           | ส่ง Response กลับไปยัง Browser               |
| `httpd_resp_set_type(req, type)`           | กำหนด Content-Type (เช่น `application/json`) |
| `xSemaphoreCreateMutex()`                  | สร้าง Mutex สำหรับป้องกัน Race Condition     |
| `xSemaphoreTake(mutex, ticks)`             | Lock Mutex ก่อนอ่าน/เขียน Shared Data        |
| `xSemaphoreGive(mutex)`                    | Unlock Mutex หลังเสร็จ                       |
