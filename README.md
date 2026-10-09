# راهنمای جامع مسدودسازی ترافیک تورنت (BitTorrent) در پنل پاسارگاد (PasarGuard)
> **هدف:** جلوگیری قطعی از دریافت ابیوزهای کپی‌رایت (Copyright Abuse) در دیتاسنترهای حساس مانند Hetzner، OVH و...  
> **روش پیاده‌سازی:** استراتژی دفاع دولایه (Defense-in-Depth) شامل **هسته پاسارگاد (Xray)** + **فایروال سرور (iptables)**

---

## 🎯 چرا به دو لایه محافظت نیاز داریم؟
* **لایه اول (هسته پاسارگاد):** بیش از ۸۰ تا ۹۰ درصد ترافیک تورنت، سایت‌ها و ترکرهای عمومی را به شکلی تمیز و در سطح لایه اپلیکیشن مسدود می‌کند.
* **لایه دوم (iptables کرنل):** به عنوان کمربند ایمنی سرور عمل می‌کند؛ کلاینت‌های مدرن تورنت قابلیتی به نام Protocol Encryption (MSE/PE) دارند که پکت‌ها را رمزنگاری می‌کند تا پراکسی نتواند پروتکل را تشخیص دهد. iptables در لایه سیستم‌عامل هر پکت رمزنگاری‌نشده یا پورت مشکوک خروجی را پیش از رسیدن به شبکه دیتاسنتر متوقف (Drop) می‌کند.

---

## 🛡️ بخش اول: تنظیمات لایه ۱ (هسته پاسارگاد / Core Config)

وارد پنل پاسارگاد شوید:  
منوی **Core Configs (کانفیگ‌های هسته)** ⬅️ انتخاب کانفیگ مورد نظر (مثلاً `Default Core Config`) ⬅️ ویرایش بخش JSON یا بخش روتینگ.

### ۱. مطمئن شوید Outbound مسدودسازی وجود دارد:
در بخش `"outbounds"` باید تگ `BLOCK` با پروتکل `blackhole` تعریف شده باشد:
```json
{
  "protocol": "blackhole",
  "tag": "BLOCK"
}
```

### ۲. رول‌های مسدودسازی تورنت در بخش Routing:
این رول‌ها را در بالای لیست قوانین `"rules"` داخل `"routing"` قرار دهید:

```json
[
  {
    "type": "field",
    "outboundTag": "BLOCK",
    "ruleTag": "block-torrent-ports",
    "port": "6881-6889,6969,51413,54262"
  },
  {
    "type": "field",
    "outboundTag": "BLOCK",
    "ruleTag": "block-torrent-protocol",
    "protocol": [
      "bittorrent"
    ]
  },
  {
    "type": "field",
    "outboundTag": "BLOCK",
    "ruleTag": "block-torrent-domains",
    "domains": [
      "geosite:torrent",
      "domain:eztvx.to",
      "domain:thepiratebay.org",
      "domain:1337x.to",
      "domain:yts.mx",
      "keyword:torrent",
      "keyword:tracker",
      "keyword:announce"
    ]
  },
  {
    "type": "field",
    "outboundTag": "BLOCK",
    "ip": [
      "geoip:private"
    ]
  }
]
```

> **نکته بسیار مهم:** دقت کنید که قوانین `port`، `protocol` و `domains` باید در آبجکت‌های **جداگانه** تعریف شوند. در Xray اگر چند فیلد را در یک رول بنویسید، رابطه آن‌ها «AND» (شرط همزمان) می‌شود و قانون کار نخواهد کرد.

### ۳. فعال‌سازی Sniffing روی Inboundها:
برای اینکه هسته بتواند پروتکل `bittorrent` را شناسایی کند، اسنیفینگ روی اینباندها باید فعال باشد:
```json
"sniffing": {
  "enabled": true,
  "destOverride": ["http", "tls", "quic"],
  "metadataOnly": false
}
```

پس از اعمال تغییرات، در صورتی که نودها خودکار بروز نشدند، با اجرای دستور `pasarguard restart` یا همگام‌سازی نودها در پنل، تغییرات را اعمال کنید.

---

## 🔒 بخش دوم: تنظیمات لایه ۲ (فایروال سرور با iptables)

این دستورات را باید روی **سرور اصلی و تک‌تک سرورهای نود (Nodes)** در محیط ترمینال SSH اجرا کنید.

### اسکریپت یک‌خطی نصب و اعمال قوانین فایروال:
کافیست دستور زیر را به صورت کامل کپی کرده و در ترمینال سرور اینتر بزنید:

```bash
sudo apt-get update && sudo apt-get install -y iptables-persistent netfilter-persistent && \
sudo iptables -I OUTPUT -p tcp --dport 54262 -j DROP && \
sudo iptables -I OUTPUT -p udp --dport 54262 -j DROP && \
sudo iptables -I FORWARD -p tcp --dport 54262 -j DROP && \
sudo iptables -I FORWARD -p udp --dport 54262 -j DROP && \
sudo iptables -I OUTPUT -p tcp --dport 6881:6889 -j DROP && \
sudo iptables -I OUTPUT -p udp --dport 6881:6889 -j DROP && \
sudo iptables -I FORWARD -p tcp --dport 6881:6889 -j DROP && \
sudo iptables -I FORWARD -p udp --dport 6881:6889 -j DROP && \
sudo iptables -I OUTPUT -p tcp --dport 6969 -j DROP && \
sudo iptables -I FORWARD -p tcp --dport 6969 -j DROP && \
sudo iptables -I OUTPUT -m string --algo bm --string "BitTorrent" -j DROP && \
sudo iptables -I FORWARD -m string --algo bm --string "BitTorrent" -j DROP && \
sudo iptables -I OUTPUT -m string --algo bm --string "peer_id=" -j DROP && \
sudo iptables -I FORWARD -m string --algo bm --string "peer_id=" -j DROP && \
sudo iptables -I OUTPUT -m string --algo bm --string "info_hash" -j DROP && \
sudo iptables -I FORWARD -m string --algo bm --string "info_hash" -j DROP && \
sudo netfilter-persistent save
```

### این دستور چه کارهایی انجام می‌دهد؟
1. پکیج `iptables-persistent` را نصب می‌کند تا رول‌ها با ریبوت شدن سرور از بین نروند.
2. پورت گزارش‌شده هتزنر (`54262`) را در خروجی سرور و ترافیک فوروارد مسدود می‌کند.
3. بازه پورت‌های پیش‌فرض تورنت (`6881` تا `6889` و `6969`) را روی TCP و UDP مسدود می‌کند.
4. با قابلیت **Deep Packet Inspection (DPI)** کرنل لینوکس، تمام بسته‌هایی که حاوی امضاهای متنی پروتکل تورنت (`BitTorrent`, `peer_id=`, `info_hash`) هستند را فیلتر و متوقف (DROP) می‌کند.
5. در نهایت با `netfilter-persistent save` تمامی قوانین را برای همیشه ذخیره می‌کند.

---

## 🔍 بخش سوم: تست و اطمینان از صحت عملکرد

برای اینکه بررسی کنید رول‌ها به درستی روی سرور فعال هستند:
```bash
sudo iptables -L OUTPUT -n -v
```
باید رول‌های `DROP` مربوط به پورت‌های تورنت و رشته‌های BitTorrent را در بالای زنجیره `OUTPUT` مشاهده کنید.

---

## 📌 خلاصه برای همکاران و ادمین‌ها:
1. **کانفیگ هسته پنل:** دامنه‌ها، پورت‌ها و پروتکل تورنت را ببندید (فیلتر نرم‌افزاری تمیز).
2. **فایروال نودها:** اسکریپت یک‌خطی iptables را روی همه نودها بزنید (سپر دفاعی نفوذناپذیر در برابر تورنت رمزگذاری‌شده).
3. با ترکیب این دو، سرور شما در برابر هرگونه ابیوز کپی‌رایت و مسدودسازی توسط هتزنر ایمن خواهد بود.
