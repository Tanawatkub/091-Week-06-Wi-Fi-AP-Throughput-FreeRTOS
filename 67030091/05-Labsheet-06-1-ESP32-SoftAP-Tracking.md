# ใบงานที่ 6.1: การคอนฟิก ESP32 SoftAP และการสกัด Forensic Log ข้อมูล Client (Wi-Fi Access Point Mode)

## 0. กล่าวนำ (Introduction)
ในใบงานนี้ นักศึกษาจะได้สลับบทบาทของ ESP32 จากการเป็นลูกข่าย (Station) มาเป็นผู้ให้บริการจุดเชื่อมต่อไร้สาย **SoftAP (Software Access Point Mode)** ด้วยสถาปัตยกรรม ESP-IDF 

นักศึกษาจะได้เรียนรู้การเปิดบริการ DHCP Server การตั้งค่าโหมดความปลอดภัย WPA2-PSK และดักจับ Forensic Log เมื่อมีอุปกรณ์ลูกข่าย (เช่น สมาร์ตโฟน หรือ ESP32 Station ของเพื่อน) เข้ามาเชื่อมต่อ โดยสกัดข้อมูลในระดับ Link Layer เช่น **MAC Address** และ **Association ID (AID)** จาก Event `WIFI_EVENT_AP_STACONNECTED`

---

## 1. วัตถุประสงค์ (Objectives)
1. สามารถคอนฟิก ESP32 ให้ทำงานในโหมด SoftAP (`WIFI_MODE_AP`) และเปิดบริการ DHCP Server ได้สำเร็จ
2. สามารถใช้ Event Loop ในการดักจับ Event `WIFI_EVENT_AP_STACONNECTED` และ `WIFI_EVENT_AP_STADISCONNECTED`
3. สกัดและวิเคราะห์ข้อมูล MAC Address (BSSID) และ Association ID (AID) ของอุปกรณ์ลูกข่ายที่เข้ามาเชื่อมต่อ
4. เข้าใจกลไกการจำกัดจำนวนการเชื่อมต่อสูงสุด (`max_connection`) บน ESP32

---

## 2. อุปกรณ์และซอฟต์แวร์ที่ใช้ในการทดลอง (Equipment & Tools)
1. บอร์ดไมโครคอนโทรลเลอร์ ESP32 จำนวน 1 บอร์ด
2. สายเชื่อมต่อ Micro-USB หรือ USB-C จำนวน 1 เส้น
3. สมาร์ตโฟน หรือ คอมพิวเตอร์ สำหรับทดสอบเชื่อมต่อ Wi-Fi ที่ ESP32 สร้างขึ้น
4. โปรแกรม IDE เช่น VS Code พร้อม ESP-IDF Toolchain

---

## 3. ความรู้พื้นฐานที่เกี่ยวข้อง (Theoretical Background)

### 3.1 สถาปัตยกรรม Event และ DHCP Server ในโหมด SoftAP

```mermaid
sequenceDiagram
    autonumber
    participant App as Application Code
    participant Evt as ESP Event Loop
    participant AP as ESP32 SoftAP Driver
    participant STA as Mobile Client

    App->>AP: esp_netif_create_default_wifi_ap()
    App->>AP: esp_wifi_set_config(WIFI_IF_AP, &ap_config)
    App->>AP: esp_wifi_start()
    note over AP: กระจาย Beacon Frame (SSID)<br/>เปิดบริการ DHCP Server (192.168.4.1)

    STA->>AP: Connect Wi-Fi
    AP->>Evt: Post WIFI_EVENT_AP_STACONNECTED
    Evt->>App: Callback: wifi_event_handler()
    note over App: อ่าน MAC Address และ AID ของ Client
```

### 3.2 โครงสร้างข้อมูล `wifi_event_ap_staconnected_t` (Class Diagram)

```mermaid
classDiagram
    class wifi_event_ap_staconnected_t {
        +uint8_t[6] mac
        +uint8_t aid
        +bool is_mesh_child
    }
```

---

## 4. ขั้นตอนการทดลอง (Experimental Procedures)

1. สแกนและตั้งชื่อ SSID ของ ESP32 AP เป็นชื่อเฉพาะของตนเอง (เช่น `"ESP32_AP_XXXX"` โดยระบุรหัสนักศึกษา 4 ตัวท้าย)
2. กำหนดรหัสผ่าน Wi-Fi เป็น `"12345678"` (WPA2-PSK) และจำกัดจำนวนการเชื่อมต่อไว้ที่ 4 เครื่อง (`.max_connection = 4`)
3. ทำการ Build และ Flash ซอร์สโค้ดลงบอร์ด ESP32
4. นำสมาร์ตโฟนกดค้นหา Wi-Fi และป้อนรหัสผ่านเพื่อเชื่อมต่อเข้ากับ ESP32 AP
5. สังเกต Forensic Log ใน Serial Monitor และบันทึกค่า MAC Address, AID และ IP Address ที่ ESP32 แจกจ่ายให้

---

## 5. ซอร์สโค้ดการทดลอง (Complete ESP-IDF Source Code - `main.c`)

ดูใน Lab6-1-Wi-Fi-SoftAP\main\main.c

---

## 6. ตารางบันทึกผลการทดลอง (Experiment Results)

### 6.1 บันทึกข้อมูล Client ที่เชื่อมต่อเข้ากับ ESP32 SoftAP

| อุปกรณ์ที่ใช้ทดสอบ (เช่น iPhone/Android) | MAC Address ที่ดักจับได้ | Association ID (AID) | หมายเลข IP Address ที่ได้ (ถ้าทราบ) |
| :--- | :--- | :---: | :---: |
| iPhone | F6:CA:5E:0D:75:32 | 1 | 192.168.4.2 |
| ipad | E6:90:19:8C:83:F1 | 2 | 192.168.4.4 |

---

## 7. คำถามท้ายการทดลอง (Post-Lab Questions)

1. เหตุใด IP Address เริ่มต้นของ ESP32 SoftAP จึงเป็น `192.168.4.1` และ DHCP Server บน ESP32 เริ่มแจกจ่าย IP ที่หมายเลขใด?

   `192.168.4.1` เป็นค่า default ที่ ESP-IDF ตั้งไว้ให้กับ Interface `esp_netif_create_default_wifi_ap()` โดยอัตโนมัติ ซึ่งอยู่ในช่วง Private IP (RFC1918) เหมือน Router ทั่วไปที่มักใช้ `192.168.x.1` เป็น Gateway ของวงเครือข่าย
   ส่วน DHCP Server จะเริ่มแจก IP ให้ Client เครื่องแรกที่ `192.168.4.2` เป็นต้นไป (เพราะ `.1` ถูกจองไว้ให้ตัว ESP32 เองแล้ว) ดังที่เห็นใน log ว่าอุปกรณ์ที่ 1 ได้ `192.168.4.2` และอุปกรณ์ที่ 2 ได้ `192.168.4.3`

2. สมาชิกตัวแปร `mac` ในโครงสร้าง `wifi_event_ap_staconnected_t` สามารถนำไปประยุกต์ใช้ทำระบบความปลอดภัยขั้นสูง (เช่น MAC Filtering) ได้อย่างไร?

   สามารถนำค่า `mac` ที่ได้จาก Event ไปเทียบกับ "รายชื่อ MAC ที่อนุญาต" (Whitelist) ที่เก็บไว้ล่วงหน้า เช่นเก็บใน Array หรือ NVS ถ้า MAC ที่เชื่อมต่อเข้ามาไม่ตรงกับรายการ ก็สั่ง `esp_wifi_deauth_sta(aid)` เพื่อตัดการเชื่อมต่อทันที เป็นการทำ Access Control ระดับ Layer 2 เพิ่มเติมจากรหัสผ่าน Wi-Fi ปกติ
   ข้อควรระวัง: มือถือรุ่นใหม่มักใช้ MAC แบบสุ่ม (Randomized MAC) เพื่อความเป็นส่วนตัว ทำให้ MAC เปลี่ยนทุกครั้งที่เชื่อมต่อใหม่ วิธีนี้จึงอาจใช้ไม่ได้ผลกับอุปกรณ์กลุ่มนี้

3. หากมี Client พยายามเชื่อมต่อเป็นเครื่องที่ 5 (เกินค่า `max_connection = 4`) จะเกิดเหตุการณ์ใดขึ้นในระดับสัญญาณวิทยุ?

   ESP32 จะปฏิเสธการเชื่อมต่อในขั้นตอน Association โดยตอบกลับด้วย **Association Response Frame** ที่มี Status Code เป็น "Reject" (เช่น AP is full / unable to handle new associations) กลับไปยัง Client เครื่องที่ 5 ทำให้ Client เครื่องนั้นเชื่อมต่อไม่สำเร็จ แม้จะใส่รหัสผ่านถูกต้องก็ตาม เพราะ AP ปฏิเสธตั้งแต่ระดับ Management Frame ก่อนจะไปถึงขั้นตอนแจก IP จาก DHCP

---
output log 0091 ที่ได้
```
entry 0x40080644
--- 0x40080644: call_start_cpu0 at /Users/tanwat/.espressif/v6.0.2/esp-idf/components/bootloader/subproject/main/bootloader_start.c:27
I (27) boot: ESP-IDF v6.0.2 2nd stage bootloader
I (27) boot: compile time Aug 11 2026 08:49:07
I (27) boot: Multicore bootloader
I (29) boot: chip revision: v3.1
I (32) boot.esp32: SPI Speed      : 40MHz
I (35) boot.esp32: SPI Mode       : DIO
I (39) boot.esp32: SPI Flash Size : 2MB
I (42) boot: Enabling RNG early entropy source...
I (47) boot: Partition Table:
I (49) boot: ## Label            Usage          Type ST Offset   Length
I (56) boot:  0 nvs              WiFi data        01 02 00009000 00006000
I (62) boot:  1 phy_init         RF data          01 01 0000f000 00001000
I (69) boot:  2 factory          factory app      00 00 00010000 00100000
I (75) boot: End of partition table
I (79) esp_image: segment 0: paddr=00010020 vaddr=3f400020 size=1a240h (107072) map
I (124) esp_image: segment 1: paddr=0002a268 vaddr=3ffb0000 size=04528h ( 17704) load
I (131) esp_image: segment 2: paddr=0002e798 vaddr=40080000 size=01880h (  6272) load
I (134) esp_image: segment 3: paddr=00030020 vaddr=400d0020 size=87e00h (556544) map
I (334) esp_image: segment 4: paddr=000b7e28 vaddr=40081880 size=13d88h ( 81288) load
I (368) esp_image: segment 5: paddr=000cbbb8 vaddr=50000000 size=00028h (    40) load
I (379) boot: Loaded app from partition at offset 0x10000
I (379) boot: Disabling RNG early entropy source...
I (389) cpu_start: Multicore app
I (397) cpu_start: GPIO 3 and 1 are used as console UART I/O pins
I (398) cpu_start: Pro cpu start user code
I (398) cpu_start: cpu freq: 160000000 Hz
I (400) app_init: Application information:
I (403) app_init: Project name:     wifi_softap_tracking
I (408) app_init: App version:      029da6d
I (412) app_init: Compile time:     Aug 11 2026 08:49:02
I (417) app_init: ELF file SHA256:  151e25a82...
I (422) app_init: ESP-IDF:          v6.0.2
I (426) efuse_init: Min chip rev:     v0.0
I (429) efuse_init: Max chip rev:     v3.99 
I (433) efuse_init: Chip rev:         v3.1
I (437) heap_init: Initializing. RAM available for dynamic allocation:
I (444) heap_init: At 3FFAE6E0 len 00001920 (6 KiB): DRAM
I (449) heap_init: At 3FFB8A20 len 000275E0 (157 KiB): DRAM
I (454) heap_init: At 3FFE0440 len 00003AE0 (14 KiB): D/IRAM
I (459) heap_init: At 3FFE4350 len 0001BCB0 (111 KiB): D/IRAM
I (465) heap_init: At 40095608 len 0000A9F8 (42 KiB): IRAM
I (472) spi_flash: detected chip: generic
I (474) spi_flash: flash io: dio
W (477) spi_flash: Detected size(4096k) larger than the size in the binary image header(2048k). Using the size in the binary image header.
I (491) main_task: Started on CPU0
I (491) main_task: Calling app_main()
I (491) LAB_SOFTAP: [FORENSIC]: Call nvs_flash_init()
I (531) LAB_SOFTAP: [FORENSIC]: Call esp_netif_init()
I (531) LAB_SOFTAP: [FORENSIC]: Call esp_event_loop_create_default()
I (531) LAB_SOFTAP: [FORENSIC]: Call esp_netif_create_default_wifi_ap()
I (541) LAB_SOFTAP: [FORENSIC]: SoftAP Interface created at 0x3ffbddfc (Default IP: 192.168.4.1)
I (551) LAB_SOFTAP: [FORENSIC]: Call esp_wifi_init(&cfg)
I (561) wifi:wifi driver task: 3ffc0580, prio:23, stack:6656, core=0
I (581) wifi:wifi firmware version: 00ad238
I (581) wifi:wifi certification version: v7.0
I (581) wifi:config NVS flash: enabled
I (581) wifi:config nano formatting: disabled
I (581) wifi:Init data frame dynamic rx buffer num: 32
I (591) wifi:Init static rx mgmt buffer num: 5
I (591) wifi:Init management short buffer num: 32
I (601) wifi:Init dynamic tx buffer num: 32
I (601) wifi:Init static rx buffer size: 1600
I (601) wifi:Init static rx buffer num: 10
I (611) wifi:Init dynamic rx buffer num: 32
I (611) wifi_init: rx ba win: 6
I (611) wifi_init: accept mbox: 6
I (621) wifi_init: tcpip mbox: 32
I (621) wifi_init: udp mbox: 6
I (621) wifi_init: tcp mbox: 6
I (631) wifi_init: tcp tx win: 5760
I (631) wifi_init: tcp rx win: 5760
I (631) wifi_init: tcp mss: 1440
I (641) wifi_init: WiFi IRAM OP enabled
I (641) wifi_init: WiFi RX IRAM OP enabled
I (641) LAB_SOFTAP: [FORENSIC]: Call esp_event_handler_instance_register(WIFI_EVENT)
I (651) LAB_SOFTAP: [FORENSIC]: Call esp_wifi_set_mode(WIFI_MODE_AP)
I (661) LAB_SOFTAP: [FORENSIC]: Call esp_wifi_set_config(WIFI_IF_AP, &wifi_config)
I (941) LAB_SOFTAP: [FORENSIC]: Call esp_wifi_start()
I (941) phy_init: phy_version 4863,a3a4459,Oct 28 2025,14:30:06
I (1011) wifi:mode : softAP (84:1f:e8:20:54:c1)
I (1021) wifi:Total power save buffer number: 16
I (1021) wifi:Init max length of beacon: 752/752
I (1021) wifi:Init max length of beacon: 752/752
I (1021) LAB_SOFTAP: ==================================================================
I (1031) esp_netif_lwip: DHCP server started on interface WIFI_AP_DEF with IP: 192.168.4.1
I (1041) LAB_SOFTAP:   ESP32 SoftAP Running! SSID: "MY_ESP32_AP_0091", Channel: 1
I (1051) LAB_SOFTAP: ==================================================================
I (1051) LAB_SOFTAP: [TCP SERVER]: Listening on 192.168.4.1:8080
I (1061) main_task: Returned from app_main()
I (18361) wifi:new:<1,0>, old:<1,1>, ap:<1,0>, sta:<255,255>, prof:1, snd_ch_cfg:0x0
I (18361) wifi:station: f6:ca:5e:0d:75:32 join, AID=1, bgn, 20
I (18391) LAB_SOFTAP: =======================================================
I (18391) LAB_SOFTAP: [FORENSIC EVENT]: Client Connected to ESP32 SoftAP!
I (18391) LAB_SOFTAP:   -> Client MAC Address : F6:CA:5E:0D:75:32
I (18401) LAB_SOFTAP:   -> Assigned AID       : 1
I (18401) LAB_SOFTAP: =======================================================
I (19671) esp_netif_lwip: DHCP server assigned IP to a client, IP is: 192.168.4.2
I (21771) wifi:<ba-add>idx:2 (ifx:1, f6:ca:5e:0d:75:32), tid:0, ssn:2, winSize:64
I (21921) wifi:<ba-clr>idx:2, tid:0, ssn:2, winSize:64
I (22981) wifi:station: e2:1b:69:2b:3a:eb join, AID=2, bgn, 20
I (23001) LAB_SOFTAP: =======================================================
I (23001) LAB_SOFTAP: [FORENSIC EVENT]: Client Connected to ESP32 SoftAP!
I (23011) LAB_SOFTAP:   -> Client MAC Address : E2:1B:69:2B:3A:EB
I (23011) LAB_SOFTAP:   -> Assigned AID       : 2
I (23021) LAB_SOFTAP: =======================================================
I (24111) esp_netif_lwip: DHCP server assigned IP to a client, IP is: 192.168.4.3
I (43931) wifi:<ba-add>idx:3 (ifx:1, f6:ca:5e:0d:75:32), tid:1, ssn:0, winSize:64
I (43991) wifi:<ba-add>idx:4 (ifx:1, f6:ca:5e:0d:75:32), tid:5, ssn:0, winSize:64
I (177261) wifi:[ADDBA]RX DELBA, reason:37, delete tid:0, initiator:0(recipient)
I (186691) wifi:[ADDBA]RX DELBA, reason:1, delete tid:0, initiator:0(recipient)
W (206041) wifi:[ADDBA]RX addba response, tid:0, txa_attempts:1, state wrong txa_flags:0x9
W (294201) wifi:[ADDBA]RX addba response, tid:0, txa_attempts:2, state wrong txa_flags:0x9
W (382271) wifi:[ADDBA]RX addba response, tid:0, txa_attempts:3, state wrong txa_flags:0x9
I (645431) wifi:[ADDBA]RX DELBA, reason:37, delete tid:0, initiator:0(recipient)
I (938121) wifi:station: e6:90:19:8c:83:f1 join, AID=3, bgn, 20
I (938161) LAB_SOFTAP: =======================================================
I (938161) LAB_SOFTAP: [FORENSIC EVENT]: Client Connected to ESP32 SoftAP!
I (938161) LAB_SOFTAP:   -> Client MAC Address : E6:90:19:8C:83:F1
I (938171) LAB_SOFTAP:   -> Assigned AID       : 3
I (938171) LAB_SOFTAP: =======================================================
I (939431) esp_netif_lwip: DHCP server assigned IP to a client, IP is: 192.168.4.4

```

ตัวอย่าง output log

```
entry 0x40080644
--- 0x40080644: call_start_cpu0 at C:/Users/koson/esp/v5.5.1/esp-idf/components/bootloader/subproject/main/bootloader_start.c:28
I (27) boot: ESP-IDF v6.1-beta1-685-g6a9c44fe7e7 2nd stage bootloader
I (27) boot: compile time Aug  9 2026 16:16:02
I (28) boot: Multicore bootloader
I (31) boot: chip revision: v3.0
I (33) boot.esp32: SPI Speed      : 40MHz
I (37) boot.esp32: SPI Mode       : DIO
I (41) boot.esp32: SPI Flash Size : 2MB
I (44) boot: Enabling RNG early entropy source...
I (49) boot: Partition Table:
I (51) boot: ## Label            Usage          Type ST Offset   Length
I (58) boot:  0 nvs              WiFi data        01 02 00009000 00006000
I (64) boot:  1 phy_init         RF data          01 01 0000f000 00001000
I (71) boot:  2 factory          factory app      00 00 00010000 00100000
I (77) boot: End of partition table
I (81) esp_image: segment 0: paddr=00010020 vaddr=3f400020 size=1b154h (110932) map
I (128) esp_image: segment 1: paddr=0002b17c vaddr=3ffb0000 size=04a0ch ( 18956) load
I (135) esp_image: segment 2: paddr=0002fb90 vaddr=40080000 size=00488h (  1160) load
I (136) esp_image: segment 3: paddr=00030020 vaddr=400d0020 size=8fa68h (588392) map
I (351) esp_image: segment 4: paddr=000bfa90 vaddr=40080488 size=17b5ch ( 97116) load
I (391) esp_image: segment 5: paddr=000d75f4 vaddr=50000000 size=00028h (    40) load
I (403) boot: Loaded app from partition at offset 0x10000
I (403) boot: Disabling RNG early entropy source...
I (414) cpu_start: Multicore app
I (422) cpu_start: GPIO 3 and 1 are used as console UART I/O pins
I (422) cpu_start: Pro cpu start user code
I (422) cpu_start: cpu freq: 160000000 Hz
I (424) app_init: Application information:
I (428) app_init: Project name:     wifi_softap_tracking
I (433) app_init: App version:      1
I (436) app_init: Compile time:     Aug  9 2026 16:16:41
I (441) app_init: ELF file SHA256:  c617ced60...
--- Warning: Checksum mismatch between flashed and built applications. Checksum of built application is 968d1fc34c0226e0c39f04a5ed8b482138f54cdf061866029782361dfd7bb4db
I (446) app_init: ESP-IDF:          v6.1-beta1-685-g6a9c44fe7e7
I (451) efuse_init: Min chip rev:     v0.0
I (455) efuse_init: Max chip rev:     v3.99
I (459) efuse_init: Chip rev:         v3.0
I (463) heap_init: Initializing. RAM available for dynamic allocation:
I (469) heap_init: At 3FFAE6E0 len 00001920 (6 KiB): DRAM
I (474) heap_init: At 3FFB9528 len 00026AD8 (154 KiB): DRAM
I (480) heap_init: At 3FFE0440 len 00003AE0 (14 KiB): D/IRAM
I (485) heap_init: At 3FFE4350 len 0001BCB0 (111 KiB): D/IRAM
I (490) heap_init: At 40097FE4 len 0000801C (32 KiB): IRAM
I (497) spi_flash: detected chip: generic
I (499) spi_flash: flash io: dio
W (502) spi_flash: Detected size(4096k) larger than the size in the binary image header(2048k). Using the size in the binary image header.
I (516) main_task: Started on CPU0
I (516) main_task: Calling app_main()
I (516) LAB_SOFTAP: [FORENSIC]: Call nvs_flash_init()
I (556) LAB_SOFTAP: [FORENSIC]: Call esp_netif_init()
I (556) LAB_SOFTAP: [FORENSIC]: Call esp_event_loop_create_default()
I (556) LAB_SOFTAP: [FORENSIC]: Call esp_netif_create_default_wifi_ap()
I (566) LAB_SOFTAP: [FORENSIC]: SoftAP Interface created at 0x3ffbf360 (Default IP: 192.168.4.1)
I (576) LAB_SOFTAP: [FORENSIC]: Call esp_wifi_init(&cfg)
I (586) wifi:wifi driver task: 3ffc1ae4, prio:23, stack:6656, core=0
I (606) wifi:wifi firmware version: e12a754
I (606) wifi:wifi certification version: v7.0
I (606) wifi:config NVS flash: enabled
I (606) wifi:config nano formatting: disabled
I (606) wifi:Init data frame dynamic rx buffer num: 32
I (616) wifi:Init static rx mgmt buffer num: 5
I (616) wifi:Init management short buffer num: 32
I (616) wifi:Init dynamic tx buffer num: 32
I (626) wifi:Init static rx buffer size: 1600
I (626) wifi:Init static rx buffer num: 10
I (636) wifi:Init dynamic rx buffer num: 32
I (636) wifi_init: rx ba win: 6
I (636) wifi_init: accept mbox: 6
I (646) wifi_init: tcpip mbox: 32
I (646) wifi_init: udp mbox: 6
I (646) wifi_init: tcp mbox: 6
I (646) wifi_init: tcp tx win: 5760
I (656) wifi_init: tcp rx win: 5760
I (656) wifi_init: tcp mss: 1440
I (656) wifi_init: WiFi IRAM OP enabled
I (666) wifi_init: WiFi RX IRAM OP enabled
I (666) LAB_SOFTAP: [FORENSIC]: Call esp_event_handler_instance_register(WIFI_EVENT)
I (676) LAB_SOFTAP: [FORENSIC]: Call esp_wifi_set_mode(WIFI_MODE_AP)
I (686) LAB_SOFTAP: [FORENSIC]: Call esp_wifi_set_config(WIFI_IF_AP, &wifi_config)
I (696) LAB_SOFTAP: [FORENSIC]: Call esp_wifi_start()
I (696) phy_init: phy_version 4863,a3a4459,Oct 28 2025,14:30:06
I (766) wifi:mode : softAP (94:b5:55:f2:60:0d)
I (776) wifi:Total power save buffer number: 16
I (776) wifi:Init max length of beacon: 752/752
I (776) wifi:Init max length of beacon: 752/752
I (776) LAB_SOFTAP: ==================================================================
I (786) esp_netif_lwip: DHCP server started on interface WIFI_AP_DEF with IP: 192.168.4.1
I (796) LAB_SOFTAP:   ESP32 SoftAP Running! SSID: "MY_ESP32_AP", Channel: 1
I (806) LAB_SOFTAP: ==================================================================
I (806) LAB_SOFTAP: [TCP SERVER]: Listening on 192.168.4.1:8080
I (816) main_task: Returned from app_main()
```