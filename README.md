# Türkiye Deprem Analizi

Bu proje, USGS Deprem API'sini kullanarak Türkiye'deki depremleri harita üzerinde görselleştiren bir Vue.js uygulamasıdır.

## Özellikler

- Türkiye bölgesindeki depremleri harita üzerinde gösterir
- Tarih aralığına göre filtreleme
- Depremlerin büyüklüğüne göre renkli gösterim
- Her deprem için detaylı bilgi görüntüleme

## Kurulum

1. Projeyi klonlayın:
```bash
git clone [repo-url]
```

2. Bağımlılıkları yükleyin:
```bash
npm install
```

3. Geliştirme sunucusunu başlatın:
```bash
npm run serve
```

4. Tarayıcınızda http://localhost:8080 adresini açın

## Kullanılan Teknolojiler

- Vue.js 3
- Leaflet (Harita görselleştirme)
- Axios (API istekleri)
- USGS Earthquake API 