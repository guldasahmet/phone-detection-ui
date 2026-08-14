# Phone Detection UI

Raspberry Pi 5, Hailo-8 ve Raspberry Pi Global Shutter Camera üzerinde gerçek
zamanlı telefon tespiti, ByteTrack takibi ve pan/tilt hedef telemetrisi üreten
GTK tabanlı kontrol arayüzüdür.

Uygulama, COCO YOLOv8s modelinin 80 sınıflı çıktısından yalnız `CELL PHONE`
sınıfını kabul eder. Diğer sınıflar tracking ve overlay aşamasından önce
filtrelenir.

## Donanım

- Raspberry Pi 5 8 GB
- Hailo-8 26 TOPS
- Raspberry Pi Global Shutter Camera (IMX296)
- 16 mm lens
- İsteğe bağlı STM32 pan/tilt denetleyicisi

## Temel özellikler

- `640×480 @ 40 FPS` canlı kamera hattı
- Hailo-8 üzerinde YOLOv8s inference
- Yalnız telefon sınıfı için ByteTrack
- Tüm doğrulanmış telefonlarda kırmızı kutu
- Yalnız aktif telefonda kırmızı merkez çizgisi
- Piksel ve normalize X/Y hedef hatası
- Merkezde sarı `±25 px` kilit toleransı
- Confidence değerini arayüzden değiştirme
- Canlı kamera ve video dosyası desteği
- `/dev/ttyACM0 @ 115200` STM32 bağlantısı
- Düşük gecikmeli GStreamer kuyrukları ve native Wayland görüntüleme

## Sistem akışı

```text
IMX296 / video dosyası
        │
        ▼
GStreamer RGB pipeline
        │
        ▼
Hailo-8 YOLOv8s inference
        │
        ▼
Yalnız CELL PHONE filtresi
        │
        ▼
ByteTrack → aktif hedef → overlay / arayüz / UART
```

Arayüz thread'i video karesi taşımaz. Kamera, inference, tracking metadata,
native overlay ve Wayland sunumu ayrı GStreamer aşamalarında çalışır. Kuyruklar
eski kare biriktirmek yerine güncel kareyi koruyacak şekilde sınırlandırılmıştır.

## Model

Model proje içinde bulunur:

```text
models/yolov8s_h8.hef
```

Beklenen model özellikleri:

- Hailo-8 mimarisi
- `640×640×3` giriş
- 80 sınıflı Hailo NMS çıkışı
- COCO sınıf eşlemesi

Uygulama başlangıçta HEF biçimini ve sınıf sayısını doğrular. Arayüzden farklı
bir model seçilecekse modelin aynı çıkış yapısına sahip olması gerekir.

## Hedef ve hata hesabı

Aktif hedef görünür olduğu sürece aynı ByteTrack ID'si korunur. Aktif hedef
kaybolursa merkeze en yakın doğrulanmış telefon seçilir.

`640×480` görüntünün merkezi `(320, 240)` pikseldir:

```text
hata_x = hedef_merkez_x - 320
hata_y = hedef_merkez_y - 240

hata_x_norm = hata_x / 320
hata_y_norm = hata_y / 240
```

Görüntü koordinatında sağ ve aşağı pozitiftir. Pan mekanizmasının fiziksel
yönü nedeniyle UART X değeri config yerine kod içindeki donanım yön ayarıyla
ters çevrilir.

## UART protokolü

Paket little-endian 5 bayttır:

```text
<Bhh
```

| Alan | Tür | Açıklama |
|---|---|---|
| Header | `uint8` | `0xFF`: takip, `0xFE`: kilit |
| hata_x | `int16` | Pan ekseni hatası |
| hata_y | `int16` | Tilt ekseni hatası |

Hedef bulunmadığında `0xFF, 0, 0` gönderilir. Kilit için her iki eksenin de
`±25 px` toleransında olması gerekir.

## Proje yapısı

```text
phone-detection-ui/
├── models/       # YOLOv8s HEF modeli
├── config/       # COCO labels eşlemesi
├── src/          # GTK, runtime ve tracking kodu
├── native/       # Native GStreamer overlay
├── tests/        # Tracking ve donanım smoke testleri
├── run.sh
└── README.md
```

## Çalıştırma

Hailo Apps çalışma alanı projenin yanında bulunmalıdır:

```text
hailo-workspace/
├── hailo-apps/
└── phone-detection-ui/
```

Uygulamayı başlatmak için:

```bash
cd /home/raspberrypi/Desktop/hailo-workspace/phone-detection-ui
./run.sh
```

`run.sh`, Hailo ortamını yükler, Python yolunu ayarlar ve native overlay'i
gerektiğinde otomatik olarak derler.

## Test

```bash
cd /home/raspberrypi/Desktop/hailo-workspace/hailo-apps
source setup_env.sh
cd ../phone-detection-ui
PYTHONPATH="$PWD/src:$PYTHONPATH" python -m unittest discover -s tests -v
```

Native overlay'i elle derlemek için:

```bash
./native/build.sh
```
