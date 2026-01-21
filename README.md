# DATA ANALİZİ : Freqtrade vs Backtrader

**Hazırlayan:** Ege Şahin  
**Tarih:** Ocak 2026  
**Konu:** Algoritma temelli trading ekosistemindeki en dominant iki kütüphanenin data çekme  , data akışı ve data operasyonlarının karşılaştırmalı analizi.

## Yönetici Özeti(Executive Sum.)
Quantative-Trading konusunu daha iyi anlayabilmek için  algoritmik ticaret dünyasındaki  iki dominant kütüphane olan Freqtrade ve Backtrader'ın veri işleme, saklama ve görselleştirme yeteneklerini karşılaştırarak bu raporu hazırladım.
**Freqtrade:**  Sadece kripto para odaklıdır. Yüksek otomasyonlu ve standartlaştırılmış veri yapısına sahiptir. API tabanlı otomatik veri çeker, kurulumu hızlıdır ve disk tabanlıdır  ancak verileri katıdır değiştirilemez. Özelleştirilmiş data eklemesi oldukça zordur. 
**BackTrader:**  Varlık sınıfına takılmadan yüksek esneklilte işlem yapabilir. Farklı tip dosyalardan veri entegre edilebilir, ram tabanlıdır. object-oriented programming sayesinde basit bir şekilde yeni veri sütunu eklenebilir.

## Veri Çekme(Data Ingestion)
### FreqTrade Yaklaşımı: 
Freqtrade, veri çekmek için kendi içinde bir Exchange sınıfı barındırır. Bu sınıf, ünlü CCXT(CryptoCurrency eXchange Trading Library) kütüphanesini sarar(wrapper). Bu nedenle sadece kripto varlıklara odaklıdır.

Asenkron Yapı: Python'un asyncio kütüphanesini kullandığı için binlerce mum verisini çekerken program donmaz, ağ isteklerini paralel yönetir.

Rate Limit Yönetimi: Borsaların API limitlerine takılmamak için araya otomatik bekleme süreleri koyar. 

Veri indirme : Genellikle terminal komutu ile veri indirilir:
 ```bash 
freqtrade download-data --exchange binance --pairs ETH/USDT --days 30 
```

### Arka Planda çalışan  basitleştirilmiş python kodu : 
```python
import ccxt.async_support as ccxt
import asyncio

async def download_candles(pair, timeframe, since):
    # 1. Borsa nesnesini oluştur (Binance örneği)
    exchange = ccxt.binance()
    
    # 2. Veriyi asenkron olarak çek (Network I/O)
    # OHLCV: Open, High, Low, Close, Volume
    ohlcv = await exchange.fetch_ohlcv(pair, timeframe, since)
    
    # 3. Veriyi işle ve listeye ekle
    # Freqtrade burada ham veriyi alır, doğrular ve boşlukları (missing data) kontrol eder.
    return ohlcv 
```

### Backtrader Yaklaşımı

Backtrader, verinin nereden geldiğiyle ilgilenmez. Ona sadece "tarih, açılış, kapanış" sütunlarını vermemiz yeterlidir. Bu, onu kripto dışı varlıklar (Hisse, Tahvil, Opsiyon) için de ideal yapar.

Ancak Backtrader'da hazır bir "indirici" yoktur. Veriyi **Cerebro** (bir nevi beyin) adı verilen ana motora "beslememiz" gerekir.

#### Kod Bloğu İncelemesi:

```python
import backtrader as bt
import pandas as pd
import yfinance as yf  # Yahoo Finance'den veri çekmek için harici kütüphane

# Adım 1: Veriyi Dışarıdan Çek (Backtrader yapmaz, sen yaparsın)
# Apple hissesinin günlük verilerini indiriyoruz
raw_data = yf.download('AAPL', start='2024-01-01', end='2025-01-01')

# Adım 2: Veriyi Backtrader Formatına Dönüştür
# Backtrader, Pandas DataFrame'i okuyabilir ama ona hangi sütunun ne olduğunu söylemelisin.
data_feed = bt.feeds.PandasData(
    dataname=raw_data,
    # Sütun eşleştirmeleri (Opsiyonel, eğer standart isimlerse otomatik tanır)
    open=0,    # İlk sütun Open olsun
    high=1,    # İkinci sütun High olsun
    close=3,
    volume=5
)

# Adım 3: Beyne (Cerebro) Veriyi Yükle
cerebro = bt.Cerebro()
cerebro.adddata(data_feed)  # Motor artık bu veriyi tanıyor

print("Veri yüklendi, strateji çalışmaya hazır.")
```
## Veri Yapısı ve Depolama (Data Structure & Persistence)
### FreqTrade:
Disk tabanlı (JSON/HDF5). Disk tabanlı olması büyük veri setlerinde BackTrader a göre daha iyi çalışmaını sağlar 

Standart OHLCV (Open, High, Low, Close, Volume) formatı.

Veri yapısı katıdır (Rigid), değiştirilemez.
### BackTrader:
RAM tabanlı (In-Memory).

Esnek "Lines" (Çizgiler) yapısı. Zaman serisi mantığı (close[0], close[-1]). Esnekliği karmaşık Veri modellemesinde avantaj sağlar.

## Alternatif Veri ve Özelleştirme (Custom Data)
### FreqTrade: 
Stratejimizde kullanmak amacıyla yeni bir data seti eklemek  için çekirdek kodda değişiklik veya karmaşık dataframe manipülasyonu yapmamız gerekir. Oldukça zordur
### BackTrader:
Object-Oriented-Programming(oop) kullanılarak kolayca yeni veri sütunu eklenebilir. 
#### Örnek Uygulama: BackTrader' a (P/E) verisi ekleyelim
Adım 1: Yeni Veri Yapısını tanımlayalım (Class Inheritence)
 Backtrader'ın standart CSV okuyucusunu "miras alıp" (subclassing) ona yeni bir damar (line) ekliyoruz.
```python
import backtrader as bt

# Standart veri setini genişletiyoruz
class FundamentalData(bt.feeds.GenericCSVData):
    # Standart OHLCV hatlarına ek olarak 'pe_ratio' adında yeni bir hat ekliyoruz.
    lines = ('pe_ratio',)

    # CSV dosyasındaki hangi sütunun pe_ratio olduğunu varsayılan olarak belirtiyoruz.
    # (Örneğin CSV'de 7. sütun olsun, indeks 6 olur)
    params = (
        ('pe_ratio', 6),
    )
```
Adım 2: Veriyi Yükleme (Data Loading)
CSV dosyamız (hisse_verisi.csv) şöyle görünüyor olsun: Date,Open,High,Low,Close,Volume,PE_Ratio 2024-01-01,100,105,99,102,1000,12.5
```python
# Veriyi Cerebro'ya (Beyne) yüklerken kendi yazdığımız sınıfı kullanıyoruz
data = FundamentalData(
    dataname='hisse_verisi.csv',
    dtformat='%Y-%m-%d',
    # Standartlar
    datetime=0, open=1, high=2, low=3, close=4, volume=5,
    # Bizim özel verimiz
    pe_ratio=6 
)

cerebro.adddata(data)
```
Adım 3: Strateji İçinde Kullanma (Trading Logic)
İşte büyünün gerçekleştiği yer. Strateji dosyasında (next metodunda) artık bu veriye erişebilirsin.
```python
class SmartStrategy(bt.Strategy):
    def __init__(self):
        # Basit Hareketli Ortalama (SMA)
        self.sma = bt.indicators.SimpleMovingAverage(self.data.close, period=20)

    def next(self):
        # Erişim: self.data.pe_ratio[0] -> Şu anki PE oranı
        
        # LOGIC: Fiyat ortalamanın üstünde VE şirket ucuz (PE < 15)
        if self.data.close[0] > self.sma[0] and self.data.pe_ratio[0] < 15:
            
            if not self.position:
                self.buy() # ALIM YAP
                print(f"ALINDI: Fiyat: {self.data.close[0]}, PE Oranı: {self.data.pe_ratio[0]}")
        
        # Satış mantığı...
        elif self.data.pe_ratio[0] > 25: # Eğer hisse çok pahalanırsa sat
            self.close()
```
## Görselleştirme ve Analiz (Plotting)
**FreqUI (Freqtrade):** Web tabanlı, interaktif, operasyonel izleme için ideal. Plotly kullanır.

**Matplotlib (Backtrader):** Statik, akademik ve yayın kalitesinde çıktılar için ideal. Python tabanlı.


## Sonuç ve Kişisel Değerlendirme
Sonuç olarak iki kütüphane arasında "data" açısından çok net ve ayırt edici farklar var. 
FreqTrade kütüphanesi disk tabanlı olması sebebiyle büyük verilerle uğraşırken daha rahat olmamızı sağlayacaktır ancak sadece kripto varlıklarını hedef alması ve  katı data yapısı sebebiyle özelleştirilmiş ya da alternatif datalar eklenememesi işlevselliğini oldukça kısıtlamaktadır.
BackTrader ise Ram Tabanlı olması sebebiyle büyük verilerde zorluk yaşamamıza sebep olacaktır ancak bir çok farklı data türünü entegre edebilmemiz, birden çok varlığı hedef alan trading yapabilmesi  ve benim açımdan en önemli avantajı olan  özelleştirilmiş data sütunlarını kolayca ekleyebilmemiz sayesinde teknik analizin ötesinde duygusal ve toplumsal parametreler ekleyip daha tutarlı  trading yapmamıza olanak sağlayacaktır. 

