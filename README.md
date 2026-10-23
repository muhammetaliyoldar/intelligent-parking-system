# Akıllı Otopark Sistemi (Intelligent Parking System)

Plaka tanıma ile park yeri doluluk sensörünü birleştiren bir otopark prototipi. Kamera görüntüsündeki plakayı YOLO11n tabanlı bir modelle bulur, Tesseract OCR ile okuyup Türk plaka formatına göre doğrular ve aracın giriş/çıkışını SQLite'a kaydeder. Çıkışta, kalış süresine göre saatlik 10 TL üzerinden ücret hesaplar. Park yerinin dolu ya da boş olduğunu ESP32 üzerindeki PIR ve ultrasonik sensör algılar ve TCP üzerinden Python sunucusuna gönderir. Sonuç bir Pygame ekranında görselleştirilir.

<table>
<tr>
  <td><img src="görseller/giriş.png" width="100%" alt="Giriş"></td>
  <td><img src="görseller/çıkış.png" width="100%" alt="Çıkış"></td>
</tr>
<tr>
  <td><img src="görseller/dolu.png" width="100%" alt="Dolu park yeri"></td>
  <td><img src="görseller/boş.png" width="100%" alt="Boş park yeri"></td>
</tr>
</table>

## Bileşenler ve Akış

```
ESP32-CAM ──MJPEG /stream──► main-live-cam.py ─┐
USB kamera ────────────────► main-live.py ─────┼─► ocr.py: YOLO plaka tespiti → Tesseract → TR plaka doğrulama
Klasördeki görseller ──────► main.py ──────────┘          └─► giriş / çıkış + ücret → parking_log (SQLite)

ESP32 + PIR + HC-SR04 + DS3231 ──TCP:54321──► tcp_server.py ──► parking_status (SQLite)
                                                                     └─► Parking_Lot_Visualization.py (Pygame)
```

- **Plaka tespiti** (`ocr.py`): `plate-detector.pt` dosyası YOLO11n tabanlıdır (Ultralytics 8.3.39) ve tek sınıf (`License_Plate`) tanır. Bulunan kutu kırpılır ve `pytesseract` ile `--psm 7` modunda okunur. Okunan metin `^[0-9]{2}[A-Z]{1,3}[0-9]{1,4}$` kalıbıyla doğrulanır.
- **Giriş/çıkış ve ücret**: plakanın çıkışı yapılmamış bir kaydı varsa işlem çıkış sayılır ve ücret `süre (saat) × 10 TL` olarak hesaplanır. Böyle bir kayıt yoksa yeni giriş kaydı açılır.
- **Doluluk sensörü** (`Otopark_Sistemi_ardunio/`): bir ESP32 kartı, PIR sensörü hareket algılayınca HC‑SR04 ile 5 ölçüm yapıp ortalamasını alır. Ortalama 160 cm eşiğinin altındaysa yer "Dolu" kabul edilir. Durum kırmızı/yeşil LED ile gösterilir ve DS3231 RTC zaman damgasıyla birlikte `Park Yeri: Dolu, Tarih: ..., Saat: ...` biçiminde TCP sunucusuna gönderilir. Durum değiştiğinde ya da en az 30 saniye geçtiğinde yeni mesaj yollanır.
- **Kamera** (`cam_ardunio/`): ESP32‑CAM, `/stream` adresinden MJPEG yayını yapar (SVGA, JPEG).
- **Görselleştirme** (`Parking_Lot_Visualization.py`): Pygame ile 10 park yeri çizer. Yalnızca **1 numaralı yer** sensöre bağlıdır ve durumunu veritabanından her saniye okur. 2–10 numaralı yerlerin durumu zamana bağlı sahte (pseudo‑random) değerlerle belirlenir.
- **Veritabanı** (`database.py`): `parking_system.db` dosyasında `parking_log(id, plate, entry_time, exit_time, fee)` ve `parking_status(id, status, timestamp)` tabloları bulunur. `database-menagement.py`, kayıtları elle eklemek, silmek ve listelemek için Türkçe bir konsol menüsü sunar.

## Veri Seti ve Model

- Plaka modeli Roboflow verisiyle Google Colab'da eğitilmiştir. Repoda yalnızca bu sürecin ekran görüntüleri var: `görseller/roboflow.png` ve `roboflow-2.png` Roboflow Universe veri setini, `plaka-tanıma-model-eğitim.png` ise eğitimin başlangıcını gösteriyor. Görüntüye göre eğitim YOLO11n ile 40 epoch olarak başlatılmış ve 7.057 eğitim, 2.048 doğrulama görüntüsü kullanılmış.
- Eğitim kodu, veri seti ve nihai metrikler repoda **yok**. Ekran görüntüsünde yalnızca ilk epoch'lar görünüyor, bu nedenle burada bir doğruluk değeri verilmiyor.
- Eğitilmiş ağırlık: `plate-detector.pt` (~5,4 MB, repoda mevcut).

## Donanım

- ESP32 geliştirme kartı, HC‑SR04 ultrasonik sensör (TRIG 16, ECHO 4), PIR sensör (14), DS3231 RTC (I²C), kırmızı LED (26) ve yeşil LED (27).
- ESP32‑CAM (AI‑Thinker pin düzeni).
- PCB tasarımı `pcb/` klasöründe (`pcb.pdf`, `pcb.jpg`, `pcb.rar`). Fotoğraflar: `görseller/pcb-ön.jpg`, `görseller/pcb-arka.jpg`, `görseller/esp32-cam.jpg`.

## Teknoloji Yığını

Python · Ultralytics YOLO11 (`ultralytics==8.3.44`) · PyTorch 2.5.1 · OpenCV · pytesseract / Tesseract OCR 5.5 · SQLite · Pygame 2.6 · socket (TCP) · Arduino (ESP32, ESP32‑CAM, RTClib)

## Klasör Yapısı

```
intelligent-parking-system/
├── main.py                    # konsol menüsü: klasördeki görsellerle giriş/çıkış
├── main-live.py               # USB kamera ile canlı plaka okuma
├── main-live-cam.py           # ESP32-CAM akışı ile canlı plaka okuma
├── ocr.py                     # YOLO + Tesseract, TR plaka doğrulama, giriş/çıkış, ücret
├── database.py                # SQLite tabloları ve CRUD
├── database-menagement.py     # veritabanı yönetim menüsü
├── tcp_server.py              # ESP32 doluluk mesajlarını alan TCP sunucusu
├── Parking_Lot_Visualization.py  # Pygame otopark ekranı
├── plate-detector.pt          # plaka tespit modeli
├── parking_system.db          # boş SQLite veritabanı
├── Otopark_Sistemi_ardunio/   # ESP32 sensör + TCP istemcisi (.ino)
├── cam_ardunio/               # ESP32-CAM MJPEG sunucusu (.ino)
├── pcb/                       # PCB tasarım dosyaları
├── ocr_kurulum/               # Tesseract 5.5 Windows kurulum dosyası + kurulum notu
├── görseller/                 # ekran görüntüleri ve fotoğraflar
├── arac1.jpg, arac2.jpg       # örnek araç görselleri
├── BENİ_OKU!!!!!.txt          # kısa kurulum notu
└── requirements.txt
```

## Kurulum

```bash
pip install -r requirements.txt
```

1. **Tesseract**: `ocr_kurulum/tesseract-ocr-w64-setup-5.5.0.20241111.exe` dosyasını `C:\Program Files\Tesseract-OCR` konumuna kurun ve bu klasörü PATH'e ekleyin (adımlar `ocr_kurulum/ocr_kurulum.txt` dosyasında). Linux veya macOS'ta paket yöneticisiyle kurup `ocr.py` içindeki `tesseract_cmd` yolunu değiştirin.
2. **Yollar**: `ocr.py` içindeki `MODEL_PATH` ve `main.py` içindeki `folder_path` mutlak Windows yollarıdır (`C:\Users\muham\PycharmProjects\Otopark_Sistemi\...`). Bunları kendi yollarınızla değiştirin.
3. **Ağ**: `tcp_server.py` sunucusu `192.168.123.1:54321` adresini dinler. ESP32 sketch'i `serverIP` olarak `192.168.123.57` adresine bağlanır, `main-live-cam.py` ise `http://192.168.123.200/stream` adresini okur. Bu adresleri kendi ağınıza göre eşleştirin. WiFi bilgilerini Arduino IDE'den yüklemeden önce `.ino` dosyalarına kendiniz girin.
4. **Arduino**: ESP32 kartı için `RTClib` ve ESP32 board paketini kurun. ESP32‑CAM için AI‑Thinker kart tanımını seçin.

## Kullanım

```bash
python main.py                         # 1: giriş, 2: çıkış, 3: kayıtları listele, 4: sıfırla
python main-live.py                    # USB kamera; q ile çıkış, okumalar arası 30 sn bekleme
python main-live-cam.py                # ESP32-CAM akışı
python tcp_server.py                   # sensör mesajlarını parking_status tablosuna yazar
python Parking_Lot_Visualization.py    # otopark görselleştirmesi
python database-menagement.py          # veritabanı yönetimi
```

## Proje Durumu

- Kod iki commit ile **22 Ocak 2025** tarihinde yüklendi ve sonrasında geliştirilmedi.
- `main-live.py` ve `main-live-cam.py`, `handle_vehicle_entry_exit` fonksiyonunu import etmeden çağırıyor (`NameError`). Ayrıca `ocr.process_plate` bu fonksiyonu zaten kendi içinde çağırıyor, bu yüzden düzeltme yapılırken aynı plakanın çift işlenmemesine dikkat edilmeli.
- Görselleştirmede yalnızca 1 numaralı park yeri gerçek sensör verisi kullanıyor.
- `requirements.txt` bir `pip freeze` çıktısıdır (135 satır) ve birden fazla OpenCV paketi içeriyor.
- Lisans dosyası yok.
