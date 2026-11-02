# 🏨 Otel Rezervasyon ve Konaklama Yönetim Sistemi

Bir otelin **oda, oda tipi, misafir, rezervasyon, konaklama, ödeme, ek hizmet ve personel** süreçlerini yöneten bir sistemdir. Proje; ilişkisel veritabanı tasarımı, normalizasyon, çok tablolu SQL, tarih aralığı kontrolü, **TRIGGER, STORED PROCEDURE, VIEW, TRANSACTION** ve **indeksleme** kavramlarını uygulamalı öğrenmek amacıyla geliştirilmektedir. Veritabanının üzerine REST API ve kullanıcı arayüzü eklenecektir.

---

## 👥 Grup Bilgileri

**Grup No:** 21

| Ad Soyad | Öğrenci No |
|---|---|---|
| Sude Nur KARADAĞ | 455126 |
| Sahil Deniz SEZGİN | 455094 |

---

## 🛠 Kullanılan Teknolojiler

| Katman | Teknoloji |
|---|---|
| Veritabanı | **MySQL 8.x** |
| Backend | **Node.js** + Express |
| Veritabanı sürücüsü | `mysql2` (stored procedure / view çağrıları için ham SQL) |
| Kimlik doğrulama | JWT (`jsonwebtoken`) + `bcrypt` |
| API dokümantasyonu | Swagger (`swagger-ui-express`) + Postman koleksiyonu |
| Frontend | HTML / CSS / JavaScript (veya React) |

---

## 📌 En Önemli İş Kuralları

1. **Aynı oda, çakışan tarihlerde iki kez rezerve edilemez.** Bu kural yalnızca uygulama kodunda değil, **veritabanı seviyesinde (TRIGGER + kontrol prosedürü)** uygulanır.
   Çakışma koşulu: `yeniGiris < mevcutCikis AND yeniCikis > mevcutGiris`
2. **Oda fiyatı sezona göre değişir.** Toplam tutar = *her gecenin sezon fiyatının toplamı* + ek hizmetler.
3. **Check-out tek bir transaction içinde** yapılır; herhangi bir adım başarısız olursa `ROLLBACK`.

---

## 🗄 Veritabanı Tasarımı

En az **3NF**'e uygun, **13 tablo**:

| # | Tablo | Açıklama |
|---|---|---|
| 1 | `OdaTipi` | Tek/çift/suit vb., kapasite, açıklama |
| 2 | `Oda` | Oda numarası, kat, oda tipi, durum (müsait / dolu / temizlikte / bakımda) |
| 3 | `SezonFiyati` | Oda tipine göre sezon başlangıç-bitiş ve gecelik fiyat |
| 4 | `Misafir` | Ad, soyad, TC/pasaport, telefon, e-posta |
| 5 | `Kullanici` | Giriş bilgileri, rol (misafir / resepsiyon / yönetici) |
| 6 | `Personel` | Otel çalışanları |
| 7 | `Rezervasyon` | Misafir, oda, giriş-çıkış tarihi, durum |
| 8 | `Konaklama` | Check-in / check-out kayıtları |
| 9 | `EkHizmet` | Kahvaltı, spa, çamaşır vb. ve birim fiyatı |
| 10 | `KonaklamaHizmeti` | Konaklama ↔ ek hizmet (adet, tarih) |
| 11 | `Odeme` | Ödeme tutarı, yöntemi, tarihi |
| 12 | `TemizlikKaydi` | Oda temizlik geçmişi, temizleyen personel |
| 13 | `IslemLog` | Rezervasyon ve ödeme değişiklik günlüğü |

📐 **ER Diyagramı:** [`docs/er-diyagrami.png`](docs/er-diyagrami.png) *(eklenecek)*

---

## ⚙️ Veritabanı Nesneleri

### Trigger'lar
| # | Ad | Görev |
|---|---|---|
| 1 | `trg_rezervasyon_cakisma` | Çakışan tarihli rezervasyonu `SIGNAL SQLSTATE '45000'` ile engeller (INSERT + UPDATE) |
| 2 | `trg_konaklama_oda_durum` | Check-in'de oda **dolu**, check-out'ta **temizlikte** olur |
| 3 | `trg_rezervasyon_log`, `trg_odeme_log` | Rezervasyon ve ödeme değişikliklerini `IslemLog`'a yazar |

### Stored Procedure'ler
| # | Ad | Görev |
|---|---|---|
| 1 | `sp_bos_odalar(giris, cikis, oda_tipi_id)` | Tarih aralığı + oda tipine göre boş odalar |
| 2 | `sp_konaklama_toplam_tutar(konaklama_id)` | Sezon fiyatları + ek hizmetlerin toplamı |
| 3 | `sp_aylik_rapor(yil, ay)` | Aylık doluluk oranı ve gelir |
| 4 | `sp_checkout(konaklama_id, odeme_tutari, yontem)` | Transaction ile check-out (konaklamayı kapat → ödemeyi kaydet → oda durumunu güncelle) |

### View'lar
| # | Ad | Görev |
|---|---|---|
| 1 | `vw_bugun_giris_cikis` | Bugün giriş ve çıkış yapacak misafirler |
| 2 | `vw_oda_durum` | Oda bazında güncel durum + son temizlik tarihi |

### İndeksler
- `Rezervasyon(oda_id, giris_tarihi, cikis_tarihi)`
- `Oda(oda_no)` — UNIQUE
- `Misafir(email)` — UNIQUE

İndeksli / indekssiz sorgu süresi karşılaştırması raporda yer alacaktır (`EXPLAIN` çıktıları dahil).

### Örnek Veri
En az **20 oda, 50 misafir, 200 rezervasyon** → [`database/04_seed.sql`](database/04_seed.sql)

---

## 🔌 Backend (REST API)

**Katmanlı mimari:** `Entity/Model → Repository → Service → Controller`

- JWT tabanlı kimlik doğrulama, **rol bazlı yetkilendirme** (`misafir`, `resepsiyon`, `yonetici`)
- Stored procedure ve view'lar **API üzerinden çağrılır** (`CALL sp_...`, `SELECT * FROM vw_...`)
- GET / POST / PUT / DELETE desteği

### Örnek Endpoint Planı

| Metot | Endpoint | Rol | Açıklama |
|---|---|---|---|
| POST | `/api/auth/register` | herkes | Misafir kaydı |
| POST | `/api/auth/login` | herkes | Giriş, JWT döner |
| GET | `/api/odalar/bos?giris=&cikis=&tip=` | misafir+ | `sp_bos_odalar` |
| POST | `/api/rezervasyonlar` | misafir+ | Rezervasyon oluştur (trigger kontrolü) |
| DELETE | `/api/rezervasyonlar/:id` | misafir+ | Rezervasyon iptali |
| GET | `/api/rezervasyonlar/gecmis` | misafir | Geçmiş konaklamalar |
| GET | `/api/resepsiyon/bugun` | resepsiyon | `vw_bugun_giris_cikis` |
| POST | `/api/konaklamalar/checkin` | resepsiyon | Check-in |
| POST | `/api/konaklamalar/:id/checkout` | resepsiyon | Transaction'lı check-out |
| POST | `/api/konaklamalar/:id/hizmet` | resepsiyon | Ek hizmet ekle |
| POST | `/api/odemeler` | resepsiyon | Ödeme al |
| GET | `/api/odalar/durum` | resepsiyon+ | `vw_oda_durum` |
| CRUD | `/api/yonetici/odalar`, `/oda-tipleri`, `/sezon-fiyatlari`, `/ek-hizmetler`, `/personel` | yonetici | Yönetim paneli |
| GET | `/api/yonetici/rapor?yil=&ay=` | yonetici | `sp_aylik_rapor` |

---

## 🖥 Frontend

- **Misafir:** kayıt/giriş, boş oda arama, rezervasyon yapma/iptal, geçmiş konaklamalar
- **Resepsiyon:** günlük giriş/çıkış listesi, check-in / check-out, ek hizmet, ödeme
- **Yönetici:** oda, oda tipi, sezon fiyatı, ek hizmet, personel CRUD + aylık rapor

---

## 📁 Klasör Yapısı

```
otel-yonetim-sistemi/
├── README.md
├── database/
│   ├── 01_schema.sql          # Tablolar (DDL)
│   ├── 02_triggers.sql
│   ├── 03_procedures_views.sql
│   ├── 04_seed.sql            # Örnek veri
│   └── 05_index_test.sql      # İndeks karşılaştırması
├── backend/
│   ├── src/
│   │   ├── config/
│   │   ├── models/            # Entity
│   │   ├── repositories/
│   │   ├── services/
│   │   ├── controllers/
│   │   ├── middlewares/       # JWT, rol kontrolü
│   │   └── routes/
│   ├── .env.example
│   └── package.json
├── frontend/
├── docs/
│   ├── er-diyagrami.png
│   ├── rapor.pdf
│   └── postman_collection.json
└── .gitignore
```

---

## 🚀 Kurulum

### Gereksinimler
- Node.js 18+
- MySQL 8+

### Adımlar

```bash
# 1. Depoyu klonla
git clone <repo-url>
cd otel-yonetim-sistemi

# 2. Veritabanını oluştur
mysql -u root -p -e "CREATE DATABASE otel_db CHARACTER SET utf8mb4 COLLATE utf8mb4_turkish_ci;"
mysql -u root -p otel_db < database/01_schema.sql
mysql -u root -p otel_db < database/02_triggers.sql
mysql -u root -p otel_db < database/03_procedures_views.sql
mysql -u root -p otel_db < database/04_seed.sql

# 3. Backend
cd backend
cp .env.example .env     # DB bilgilerini ve JWT_SECRET'ı düzenle
npm install
npm run dev
```

**`.env.example`**
```
PORT=3000
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=
DB_NAME=otel_db
JWT_SECRET=degistir_beni
JWT_EXPIRES_IN=1d
```

Swagger arayüzü: `http://localhost:3000/api-docs`

---

## 🧪 Test ve Teslim Edilecekler

- [ ] Postman / Swagger koleksiyonu
- [ ] Tanıtım videosu (YouTube) — şunlar gösterilecek:
  - [ ] Çakışan rezervasyon denemesi (trigger hatası)
  - [ ] Başarısız check-out'ta ROLLBACK
  - [ ] Trigger'ların çalışması (oda durumu, IslemLog)
- [ ] SQL betiği (tablolar + trigger + prosedür + view + örnek veri)
- [ ] Rapor (ER diyagramı, normalizasyon, test senaryoları, indeks karşılaştırması)

**Video bağlantısı:** _eklenecek_

---

## 📋 İlerleme Durumu

- [x] Repo açıldı
- [x] hknaydin collaborator olarak davet edildi
- [ ] ER diyagramı
- [ ] Tablo DDL'leri
- [ ] Trigger'lar
- [ ] Stored procedure'ler
- [ ] View'lar
- [ ] Transaction (check-out)
- [ ] İndeksleme + süre karşılaştırması
- [ ] Örnek veri (20 oda / 50 misafir / 200 rezervasyon)
- [ ] Backend (katmanlı mimari + JWT + roller)
- [ ] Frontend (misafir / resepsiyon / yönetici)
- [ ] Rapor ve video

### ⭐ Ek Puan (+10) — opsiyonel
- [ ] E-posta bildirimi
- [ ] Çoklu otel desteği
- [ ] Raporların grafikle gösterimi

---

## 📝 Lisans

Bu proje ders kapsamında eğitim amaçlı geliştirilmiştir.
