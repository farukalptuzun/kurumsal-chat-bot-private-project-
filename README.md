
# Chatbotv2 — Yapısal Özeti

Bu doküman, projede yer alan katmanları, bileşenleri ve canlı çalışmayı sağlayan işlemleri koddan bağımsız olarak detaylandırır. Her katman, sistemin doğal dil anlayışı, bilgi tabanı araması ve yanıt üretimi için birlikte çalışır.

## Genel Bakış

- Kullanıcı mesajları önce doğal dil anlayışı katmanına girer, intent ve entity çıkarımı yapılır. Bu katman, kural tabanlı hızlı sınıflayıcılarıyla bazı istekleri doğrudan yönlendirir, diğerlerini LLM çağrısı için hazırlar.
- Daha sonra diyalog yöneticisi, hafıza durumu, önceki slotlar ve policy kurallarına göre bir plan seçer; örneğin bilgi talebi mi, sipariş mi, slot doldurma mı yoksa küçük konuşma mı olduğu belirlenir.
- Planın gerektirdiği ek bilgiyi elde etmek için önce bilgi bankası (knowledge base) sorgulanır. Metin tabanlı aramalar, embedding sıralaması ve gerektiğinde multi-query + fusion yoluyla en alakalı parçalar seçilir.
- Sonuçlar response generator katmanında toplanır: deterministik kurallar (fiyat, adres gibi sabit yanıtlar) uygulanır, eksik slot varsa kullanıcıdan bilgi istenir, ardından LLM sistem/user prompt taslağı oluşturularak model çağrısı yapılır.
- Oluşan yanıt çıktı olarak gönderilmeden önce fiyat kilidi, stok gizleme, URL temizleme gibi son işlemlerden geçer; ardından hafıza güncellenir ve log kaydı sağlanır.

## Çalışma Katmanları

### 1. NLU (Natural Language Understanding)

- Metin normalizasyonu, regex yardımcıları ve çok dilli intent çıkarımı sayesinde mesajın niyeti saptanır.
- Slot extraction, entity detection ve intent promotion mekanizmaları ile belirsiz ifadeler daha net kategorilere taşınır (örneğin “almak istiyorum” otomatik olarak sipariş niyetine çevrilir).
- Hafıza’dan gelen rolling summary, recent entities ve aktif order/service frameleriyle güncel bilgi birleştirilir.
- Audience detection sayesinde yanıt tonu (influencer, teknik, müşteri) belirlenebilir.

### 2. Dialog Manager

- Güncel context, önceki mesajlar, hafıza snapshotları ve policy outcome’ları değerlendirilir.
- Kayıtlı “skill” önerileri çalıştırılır; talebin bir sipariş, stok sorgusu, bilgi isteği veya hizmet süreci olup olmadığı kontrol edilir.
- Uygun plan seçilir (bilgi / slot doldurma / sipariş oluşturma / küçük konuşma). Eksik slotlar tespit edilirse slotful mod başlar.
- Follow-up mesajlarda odak (ürün/tenant) eklenerek sorgular yeniden yazılır ve aramalarda ilgili odak korunur.

### 3. RAG (Retrieval Augmented Generation)

- MongoDB üzerinde metin indeksi önü ve embedding re-rank zinciri kullanılır. İlk text-search sonuçları embedding benzerlikleriyle sıralanır; yoksa embedding fallback devreye girer.
- Numeric ve kod bazlı bonuslar (örneğin 25CrMo4 gibi kodları tanıma) pre-processing sırasında uygulanır.
- Filtreler (sektor, modality, user) ortam değişkenleri veya hafızadan alınır; multi-query expansion + RRF ile birden fazla alt sorgu birleştirilebilir.
- Embedding modeli gibi parametreler environment üzerinden yönetilir, çok formatlı belge içeriği GUI/API ile vektör tabanına eklenir.

### 4. Response Generation

- Slot ve memory verileri birleştirilerek eksik bilgiler belirlenir. Deterministik senaryolarda (fiyat sorusu, genel merkez isteği) doğrudan sabit yanıt üretilir.
- LLM çağrısı öncesi system prompt’a:
  - temel konuşma talimatları,
  - policy outcome’ları,
  - davranış talimatları (örneğin sistem davranışı),
  - tone (friendly/formal/concise/empathetic/enthusiastic),
  - strict RAG modu gibi ek bilgiler bir araya gelir.
- User prompt’ta intent, bilinen slotlar, eksikler ve KB sonuçları yer alır. Model olarak Anthropic, OpenAI veya Gemini tercih edilebilir.
- Çıktıdan sonra fiyat kilidi (sadece belirlenen fiyat çıkmalı), stok gizleme ve URL temizliği gibi süreçler çalışır; policy şablonları gerekiyorsa cevap formatlanır.

### 5. Memory ve Logging

- Rolling summary, recent entities ve aktif sipariş/hizmet frameleri ile context tutulur. Slot dolumu sırasında bu bilgiler tekrar gözden geçirilir.
- Memory güncellemesi hem hafıza koleksiyonunda hem de log dosyalarında tutulur. Her tur, intent, plan, KB sonuçları, response meta gibi alanlarla MongoDB’ye yazılır.
- Order süreçleri log dosyalarına (JSONL) kaydedilir; her order’a deterministik ID üretilerek yazılır.

## Girdi Kaynakları

- CLI: Proje kökünden başlatılan ana sohbet döngüsü; kullanıcı mesajları alınır, komutlar (`:behavior`, `:refine`, `:strict`) desteklenir.
- Document GUI: PDF/DOCX/Excel/CSV ve web URL’lerini indeksleyen grafik araç. Behaviour alanı ile sistem talimatları güncellenebilir; yüklenen belgeler otomatik vektörlenir.
- API: FastAPI tabanlı REST uç noktaları dosya/web yükleme, sorgu, davranış ve refiners işlemlerini sunar. Her istek en az `user_id` içerir. Health check ve CORS yapılandırması vardır.
- Web soket ve web chat araçları da canlı ortam entegrasyonları için hazırdır.

## Ortam Değişkenleri (Kullanım ve Kategori)

- **LLM & Davranış**: sağlayıcı (Anthropic/OpenAI/Gemini), model adı, API anahtarları, temperature, sistem davranış metinleri, strict RAG modu, tone.
- **RAG**: retriever backend, embedding modeli, top-k ve multi-query, strict mod, eklenecek filtreler.
- **Features**: deterministic ürün yanıtı, stok adeti gösterimi, linklere izin, behaviour refinements, selling mode (product/service/auto).
- **Veritabanı/Logging**: MongoDB URI, database adı, collection’lar (knowledge, products, chat logs, orders, behaviour, files).

## Operasyonel Adımlar

1. İlgili Python ortamı (üçüncü parti bağımlılıklar) kurulur; ardından `.env` oluşturulup gerekli anahtarlar girilir.
2. MongoDB bağlantısı sağlanır; başta knowledge ve behaviour koleksiyonları indexlenir.
3. Belge yükleme aracıyla metin formatları veya URL’ler indekslenir; embedding modeli bunları vektöre dönüştürür.
4. CLI veya API üzerinden sohbet başlatılır; ilk turda memory snapshot’ı çekilerek planlama başlar. her firma özelinde user_id ile firmaya özel mesajlaşma
5. Gelişmiş ihtiyaçlarda behaviour refiners ile sistem talimatı OpenAI destekli olarak keskinleştirilebilir.

## Sonuç

Bu yapı, bağlamsal hafıza, çok formatlı bilgi tabanı araması ve policy tabanlı yanıt üretimi sayesinde hem hızlı cevaplar hem de kompleks slot/detaylı plan gerektiren senaryolar için uygundur. Davranış talimatları, strict RAG modu ve deterministic kontroller opsiyonel ama güçlü konfigürasyon noktalarıdır.

