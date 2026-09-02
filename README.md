# FaultPass Windows — نقشه راه ساخت

اپ اندروید شما (`FaultPass v10`) یک کلاینت **Xray-core** است: لینک‌های
`vless / vmess / trojan / ss` را پارس می‌کند، از روی آن‌ها یک کانفیگ JSON برای
Xray می‌سازد، و از `VpnService` اندروید برای مسیردهی ترافیک کل سیستم استفاده
می‌کند. نسخه ویندوز باید همین منطق را با ابزارهای ویندوزی بازسازی کند —
همان‌طور که ابزارهای شناخته‌شده‌ای مثل v2rayN یا Hiddify این کار را می‌کنند.

## ۴ مرحله ساخت

**مرحله ۱ — اسکلت پروژه + پارسر کانفیگ (انجام شد)**
پروژه WPF/.NET 8 راه‌اندازی شد و `ConfigParser.kt` عیناً به C# پورت شد
(همان regex، همان فرمت‌های لینک، همان خروجی `ProxyConfig`).

**مرحله ۲ — موتور اتصال (Xray-core + TUN) — انجام شد، همین تحویل**
باینری رسمی **Xray-core v26.3.27** برای ویندوز (مستقیم از ریلیز رسمی
GitHub، نه یک بازسازی) داخل پروژه باندل شده. خبر خوب: نسخه‌های جدید
Xray-core یک ماژول **tun بومی برای ویندوز** دارند (`proxy/tun`، با
درایور **wintun.dll** که همان WireGuard هم استفاده می‌کند) — یعنی دقیقاً
همان چیزی که اپ اندروید شما با `"protocol": "tun"` در کانفیگش می‌خواست،
واقعاً روی ویندوز هم پشتیبانی می‌شود. `CoreConfigBuilder.kt` عیناً به
C# پورت شد و برای حالت TUN تنظیمات آدرس/MTU لازم را هم اضافه کردیم.

دو تا حالت اتصال پیاده شده (دقیقاً همون انتخابی که ابزارهایی مثل v2rayN
هم می‌دن):
- **System Proxy** (پیش‌فرض، بدون نیاز به دسترسی ادمین) — مرورگرها و
  بیشتر اپ‌ها را فوراً از پراکسی SOCKS محلی Xray عبور می‌دهد.
- **Full TUN** (نیاز به ادمین) — معادل واقعی `VpnService.Builder` اندروید:
  کل ترافیک سیستم از طریق آداپتور wintun رد می‌شود، با یک مسیر جداگانه
  برای خود سرور پراکسی تا حلقهٔ مسیریابی (routing loop) پیش نیاد.

⚠️ صادقانه بگم: بخش مسیریابی TUN (`TunRouteManager.cs`) رو با دقت نوشتم
ولی چون سندباکس من لینوکسیه نمی‌تونم `netsh`/`route.exe` واقعی رو تست
کنم — این طبیعی‌ترین جایی‌ست که موقع تست روی ویندوز واقعی ممکنه به تنظیم
metric یا زمان‌بندی نیاز داشته باشه. حالت System Proxy باید همون اول کار
کنه.

**مرحله ۳ — رابط کاربری کامل**
معادل `MainActivity` / `ConnectScreen` / `MainViewModel`: لیست سرورها،
افزودن از اشتراک (Subscription URL) یا پیست دستی، دکمهٔ Connect/Disconnect،
تست پینگ هر سرور، نمایش ترافیک آپلود/دانلود زنده.

**مرحله ۴ — بسته‌بندی و نصب‌کننده**
آیکون برنامه، اجرای خودکار در استارتاپ ویندوز (اختیاری)، ساخت نصب‌کننده
(Inno Setup یا MSIX) و امضای فایل اجرایی برای عبور راحت‌تر از هشدار
SmartScreen ویندوز، و تست نهایی end-to-end.

## چیزی که الان داخل زیپ هست

```
FaultPass.sln
FaultPass/
  FaultPass.csproj             ← پروژه WPF, .NET 8, x64
  app.manifest                 ← درخواست دسترسی ادمین (لازم برای حالت TUN)
  App.xaml / App.xaml.cs
  MainWindow.xaml(.cs)         ← پنجرهٔ تستی: Parse + Connect/Disconnect با انتخاب حالت
  Models/ProxyConfig.cs        ← پورت مستقیم ProxyConfig.kt
  Core/ConfigParser.cs         ← پورت مستقیم ConfigParser.kt
  Core/CoreConfigBuilder.cs    ← پورت مستقیم CoreConfigBuilder.kt (+ تنظیمات tun ویندوز)
  Core/XrayEngine.cs           ← اجرا/مدیریت پروسهٔ xray.exe (معادل CoreController اندروید)
  Core/ConnectionManager.cs    ← ماشین حالت اتصال (معادل FaultPassVpnService.kt)
  Net/ProxyModeManager.cs      ← تنظیم پراکسی سیستم ویندوز (حالت بدون ادمین)
  Net/TunRouteManager.cs       ← مسیردهی کامل سیستم از طریق آداپتور wintun (حالت ادمین)
  Engine/xray.exe              ← باینری رسمی Xray-core v26.3.27 (ریلیز GitHub, XTLS/Xray-core)
  Engine/wintun.dll            ← درایور رسمی Wintun (همراه ریلیز Xray-core)
  Engine/LICENSE-xray.txt, LICENSE-wintun.txt
```

### چطور اجرا کنم؟
روی یک ویندوز با **.NET 8 SDK** نصب‌شده (این پروژه روی سندباکس لینوکسی من
قابل کامپایل نیست چون WPF فقط روی ویندوز build می‌شود):

```
dotnet build
dotnet run --project FaultPass
```

۱. یک لینک `vless://...` / `vmess://...` / `trojan://...` / `ss://...` پیست کنید و **Parse** بزنید.
۲. حالت **System Proxy** را انتخاب کنید (نیاز به ادمین ندارد) و **Connect** بزنید.
۳. برای تست مرورگر: به یک سایت مسدود بروید و ببینید از پراکسی رد می‌شود.
۴. برای تست حالت **Full TUN**: برنامه را با **Run as Administrator** اجرا کنید،
   حالت TUN را انتخاب کنید و Connect بزنید؛ این حالت باید کل ترافیک سیستم
   (نه فقط مرورگر) را از تونل رد کند.

اگر حالت TUN بالا نیامد یا مرورگر متوجه اتصال نشد، لاگ کامل `xray.exe`
همان لحظه در پایین پنجره چاپ می‌شود — همان چیزی که برای دیباگ لازم دارید.









این اپ اندروید vpn من هست میخوام یک نسخه ویندوزی هم بسازم که مثل نسخه اندروید باشه همه چیزش.
۴ مرحله ساختی که تعیین کردی و تا مرحله 2 رو ساختیم حالا مرحله 3 رو شروع کن
۱. اسکلت پروژه + پارسر کانفیگ ← همین الان انجام شد
۲. موتور اتصال — باندل کردن Xray-core برای ویندوز + درایور Wintun برای مسیردهی کل سیستم (معادل VpnService)
۳. رابط کاربری کامل — اتصال/قطع، تست پینگ، نمایش ترافیک (معادل MainActivity/ConnectScreen)
۴. بسته‌بندی و نصب‌کننده — آیکون، اجرای خودکار، installer، امضای فایل اجرایی




<p align="center">
  <img src="assets/faultpass-logo.png" width="150" alt="FaultPass">
</p>

<h1 align="center">FaultPass</h1>

<p align="center">
  <strong>Secure. Fast. Global.</strong>
</p>

<p align="center">
  International VPN service built for a secure, fast and reliable internet experience.
</p>

<p align="center">
  <a href="https://github.com/FaultPass/FaultPass/releases/tag/v4.2.1">
    <img src="https://img.shields.io/github/v/release/FaultPass/FaultPass?style=flat-square&label=Version" alt="Version">
  </a>
  <a href="https://github.com/FaultPass/FaultPass/releases/download/v4.2.1/FaultPass-v4.2.1.apk">
    <img src="https://img.shields.io/badge/Android-Download-3DDC84?style=flat-square&logo=android&logoColor=white" alt="Android">
  </a>
  <a href="https://github.com/USERNAME/REPOSITORY/releases/latest/download/FaultPass-Windows.exe">
    <img src="https://img.shields.io/badge/Windows-Download-0078D6?style=flat-square&logo=windows&logoColor=white" alt="Windows">
  </a>
</p>

<p align="center">
  <a href="#english">English</a> •
  <a href="#فارسی">فارسی</a> •
  <a href="#русский">Русский</a> •
  <a href="#中文">中文</a>
</p>

---

<h2 align="center">⬇️ Download FaultPass</h2>

<p align="center">
  <a href="https://github.com/FaultPass/FaultPass/releases/download/v4.2.1/FaultPass-v4.2.1.apk">
    📱 <strong>Android</strong>
  </a>
  &nbsp;&nbsp;&nbsp;&nbsp;
  <a href="https://github.com/USERNAME/REPOSITORY/releases/latest/download/FaultPass-Windows.exe">
    💻 <strong>Windows</strong>
  </a>
</p>

---

<a id="english"></a>

## 🇬🇧 English

### About

**FaultPass** is an international VPN service designed to provide a secure, fast and reliable connection to the internet.

Built with simplicity and performance in mind, FaultPass delivers a modern connection experience without unnecessary complexity.

### Key Features

- 🌍 International connectivity
- ⚡ Fast and reliable connections
- 🔒 Secure and private communication
- 🎯 Clean and intuitive user experience
- 🚀 Continuous performance improvements

### Downloads

**Android**  
[Download the latest version →](https://github.com/FaultPass/FaultPass/releases/download/v4.2.1/FaultPass-v4.2.1.apk)

**Windows**  
[Download the latest version →](https://github.com/USERNAME/REPOSITORY/releases/latest/download/FaultPass-Windows.exe)

### Software License

FaultPass is **proprietary software** and is **not open source**.

This repository is the official GitHub repository of FaultPass and is used exclusively for official releases and project information.

---

<a id="فارسی"></a>

<div dir="rtl" align="right">

## 🇮🇷 فارسی

### درباره FaultPass

**FaultPass** یک سرویس VPN بین‌المللی است که با هدف ارائه تجربه‌ای **امن، سریع و پایدار** از اتصال به اینترنت طراحی شده است.

FaultPass با تمرکز بر **سادگی، عملکرد و تجربه کاربری مدرن**، تلاش می‌کند اتصال قابل‌اعتماد به اینترنت را بدون پیچیدگی‌های غیرضروری در اختیار کاربران قرار دهد.

### ویژگی‌ها

- 🌍 اتصال بین‌المللی
- ⚡ اتصال سریع و پایدار
- 🔒 ارتباط امن و خصوصی
- 🎯 رابط کاربری ساده و مدرن
- 🚀 بهبود مستمر عملکرد و کیفیت سرویس

### دانلود

**اندروید**  
[دانلود آخرین نسخه ←](https://github.com/FaultPass/FaultPass/releases/download/v4.2.1/FaultPass-v4.2.1.apk)

**ویندوز**  
[دانلود آخرین نسخه ←](https://github.com/USERNAME/REPOSITORY/releases/latest/download/FaultPass-Windows.exe)

### وضعیت نرم‌افزار

FaultPass یک نرم‌افزار **اختصاصی (Proprietary)** و **غیرمتن‌باز** است.

این مخزن، صفحه رسمی FaultPass در GitHub است و صرفاً برای انتشار نسخه‌های رسمی و ارائه اطلاعات مربوط به پروژه استفاده می‌شود.

</div>

---

<a id="русский"></a>

## 🇷🇺 Русский

### О FaultPass

**FaultPass** — международный VPN-сервис, разработанный для обеспечения **безопасного, быстрого и стабильного** подключения к интернету.

FaultPass сочетает **простоту, производительность и современный пользовательский интерфейс**, обеспечивая удобный и надежный опыт подключения.

### Возможности

- 🌍 Международное подключение
- ⚡ Быстрое и стабильное соединение
- 🔒 Безопасность и конфиденциальность
- 🎯 Простой и современный интерфейс
- 🚀 Постоянное улучшение производительности

### Скачать

**Android**  
[Скачать последнюю версию →](https://github.com/FaultPass/FaultPass/releases/download/v4.2.1/FaultPass-v4.2.1.apk)

**Windows**  
[Скачать последнюю версию →](https://github.com/USERNAME/REPOSITORY/releases/latest/download/FaultPass-Windows.exe)

### О программном обеспечении

FaultPass является **проприетарным программным обеспечением** и **не является проектом с открытым исходным кодом**.

Этот репозиторий является официальной страницей FaultPass на GitHub и предназначен исключительно для публикации официальных версий и информации о проекте.

---

<a id="中文"></a>

## 🇨🇳 中文

### 关于 FaultPass

**FaultPass** 是一款国际 VPN 服务，旨在提供**安全、快速且稳定**的互联网连接体验。

FaultPass 专注于**简洁性、性能和现代化用户体验**，让用户能够更加轻松地享受稳定可靠的网络连接。

### 主要功能

- 🌍 国际网络连接
- ⚡ 快速稳定的连接
- 🔒 安全与隐私保护
- 🎯 简洁现代的用户界面
- 🚀 持续优化服务性能

### 下载

**Android**  
[下载最新版本 →](https://github.com/FaultPass/FaultPass/releases/download/v4.2.1/FaultPass-v4.2.1.apk)

**Windows**  
[下载最新版本 →](https://github.com/USERNAME/REPOSITORY/releases/latest/download/FaultPass-Windows.exe)

### 软件说明

FaultPass 是**专有软件**，**不是开源项目**。

此仓库是 FaultPass 的官方 GitHub 页面，仅用于发布官方版本及提供项目相关信息。

---

<p align="center">
  <strong>FaultPass</strong><br>
  Secure. Fast. Global.
</p>








