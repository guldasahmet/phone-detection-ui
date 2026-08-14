# Phone Detection UI

Raspberry Pi 5, Hailo-8 ve Raspberry Pi Global Shutter Camera için gerçek
zamanlı telefon tespit ve takip arayüzüdür. Resmî COCO YOLOv8s HEF modelinin
yalnız `CELL PHONE` sınıfını işler.

## Özellikler

- `640×480 @ 40 FPS` kamera hattı
- Hailo-8 üzerinde YOLOv8s inference
- Yalnız telefon sınıfına uygulanan ByteTrack
- Doğrulanmış hedeflerde kırmızı kutu
- Aktif hedef için kırmızı merkez çizgisi
- Piksel ve normalize hedef hatası
- Merkezde `±25 px` kilit toleransı
- STM32 için `/dev/ttyACM0 @ 115200` UART bağlantısı
- `<Bhh>` biçiminde 5 baytlık hedef paketi
- Video dosyası ve canlı kamera desteği

## Model

Varsayılan model:

```text
models/yolov8s_h8.hef
```

Model 80 COCO sınıfı üretir; uygulama diğer 79 sınıfı tracking öncesinde
atar ve yalnız `CELL PHONE` hedeflerini gösterir.

## Çalıştırma

```bash
cd /home/raspberrypi/Desktop/hailo-workspace/phone-detection-ui
./run.sh
```

`run.sh`, Hailo Apps ortamını yükler ve native GStreamer overlay eklentisini
gerektiğinde otomatik olarak derler.

## Test

```bash
cd /home/raspberrypi/Desktop/hailo-workspace/hailo-apps
source setup_env.sh
cd ../phone-detection-ui
PYTHONPATH="$PWD/src:$PYTHONPATH" python -m unittest discover -s tests -v
```
