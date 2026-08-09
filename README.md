# Nexus

A multi-protocol anti-censorship client for **Android** and **Windows**, built with Flutter on top of
[sing-box](https://github.com/SagerNet/sing-box).

Nexus is a *client only*. It connects to servers you supply — through a subscription link, a pasted
config, a QR code or a file. It does not run, sell or bundle any proxy infrastructure.

فارسی: [راهنمای فارسی](#فارسی)

---

## What it does

**Protocols** — VMess, VLESS (with REALITY + uTLS), Shadowsocks, Trojan, WireGuard, Hysteria2, TUIC.

**Connecting**
- One big Connect button; sing-box's URLTest group picks the fastest server and fails over
  automatically when one stops answering.
- Live upload/download speed, live latency, connection uptime.
- Switching servers on a live tunnel happens through the Clash API — no reconnect, no dropped
  sockets.

**Servers**
- One unified list, whatever the source, each row tagged with where it came from.
- Search, filter by protocol or country, pin favourites, hide dead entries.
- Per-row latency with colour coding, measured concurrently across the whole list.
- Servers that fail three health checks in a row are hidden automatically (and restorable).

**Subscriptions**
- Multiple subscriptions at once, each with its own refresh interval.
- Base64, plain-text and JSON (including sing-box `outbounds`) bodies are all understood.
- Refresh failures are shown per subscription in plain language, not as a stack trace.
- Deleting a subscription removes exactly its servers, keeping your manual ones.

**Routing and privacy**
- **Smart mode**: Iranian and private destinations go direct, everything else through the tunnel —
  so local banking and Iranian sites stay fast and reachable.
- **Global** and **rules-only** modes for the other two cases.
- **Kill switch** (`strict_route`): if the tunnel drops, traffic is blocked rather than leaking.
- **Split tunneling** on Android: include or exclude specific apps.
- **IP / DNS leak check**: compares your address before and after connecting and inspects the
  resolvers answering while the tunnel is up.
- No traffic logs. Nothing about your browsing leaves the device.
- Server credentials are stored in an AES-encrypted Hive box whose key lives in the Android Keystore.

**Everything else**
- Persian and English, RTL-aware, with a light/dark/system theme.
- Daily and monthly traffic statistics with a per-server breakdown.
- Backup and restore of all settings, servers and subscriptions to a single file.
- Share any server as a link or QR code.
- Quick Settings tile and a notification with a one-tap disconnect.
- In-app update check against this repository's GitHub releases.

---

## Getting the app

### The easy way — download a build

1. Open the [**Releases**](../../releases) page of this repository.
2. Download the APK that matches your phone:
   | File | For |
   |---|---|
   | `nexus-*-arm64-v8a.apk` | almost every phone from 2018 onward — **start here** |
   | `nexus-*-armeabi-v7a.apk` | older or low-end phones |
   | `nexus-*-x86_64.apk` | emulators, x86 tablets |
   | `nexus-*-universal.apk` | works everywhere, roughly three times larger |
   | `nexus-*-windows-x64.zip` | Windows — unzip and run `nexus.exe` **as administrator** |
3. Open the file on your phone and allow installation from unknown sources when Android asks.

If there is no release yet, produce one from the **Actions** tab:

1. Go to **Actions → Build Android → Run workflow** (or **Build Windows** for the desktop app).
2. When the run finishes, the APKs are attached to it under **Artifacts** (`nexus-apk`), and the
   Windows ZIP under `nexus-windows`.
3. To publish a proper release instead, push a version tag:
   ```bash
   git tag v1.0.0 && git push origin v1.0.0
   ```
   The same workflow then creates a GitHub Release with the APKs attached.

### Signing

Without a keystore the APKs are signed with Android's debug key. They install and run fine, but
Android will not upgrade one signature to another — so set up a real key before your first public
release, then keep using it.

Create a keystore and add these repository secrets:

```bash
keytool -genkey -v -keystore nexus-release.jks -keyalg RSA \
        -keysize 2048 -validity 10000 -alias nexus
base64 -w0 nexus-release.jks   # paste into ANDROID_KEYSTORE_BASE64
```

| Secret | Value |
|---|---|
| `ANDROID_KEYSTORE_BASE64` | base64 of the `.jks` file |
| `ANDROID_KEYSTORE_PASSWORD` | keystore password |
| `ANDROID_KEY_ALIAS` | `nexus` |
| `ANDROID_KEY_PASSWORD` | key password |

The workflow picks them up automatically on the next run.

---

## Building from source

### Requirements

- Flutter **3.44.9** or newer (Dart 3.12+)
- JDK **17**
- Go **1.23+** and the Android **NDK** — only needed to build the sing-box library
- Android SDK with platform 35+

### Android

The app links against `libbox.aar`, the sing-box Android library. It is ~40 MB of compiled native
code, so it is not committed; build it once:

```bash
export ANDROID_HOME=$HOME/Android/Sdk        # wherever your SDK lives
tool/build_libbox.sh                          # writes android/app/libs/libbox.aar
```

Then build the app as usual:

```bash
flutter pub get
flutter build apk --release --split-per-abi   # per-architecture APKs
flutter build apk --release                   # one universal APK
```

The APKs land in `build/app/outputs/flutter-apk/`.

To run on a connected device: `flutter run --release`.

### Windows

The desktop build runs sing-box as a child process (it creates the TUN device itself through
wintun), plus a system tray icon with a quick connect/disconnect menu. Closing the window hides the
app to the tray so the tunnel keeps running.

```bash
flutter build windows --release
```

Then put the sing-box binary where the app expects it:

```
build/windows/x64/runner/Release/
  nexus.exe
  sing-box/
    sing-box.exe        <- from https://github.com/SagerNet/sing-box/releases
```

The **Build Windows** workflow does exactly this and uploads a ready-to-run ZIP.

**Run it as administrator.** Creating a TUN interface on Windows requires elevation; without it the
app shows a clear message instead of failing silently.

### Tests

```bash
flutter test        # 73 tests: parsers, subscriptions, config generation, widgets
flutter analyze
```

### Screenshots

`test/screenshots.dart` renders the real screens — production stores, repositories and controllers,
with a seeded server list — and writes PNGs to `build/screenshots/`:

```bash
flutter test test/screenshots.dart
```

It is not part of `flutter test` (the filename does not end in `_test.dart`), so CI stays fast.

---

## How it is put together

```
lib/
  core/
    singbox/      config generation: outbounds, routing rules, DNS, TUN inbound
    vpn/          tunnel contract, Android method channel, desktop sidecar
    clash_api_client.dart   live traffic, delay tests, live server switching
    latency_tester.dart     concurrent TCP latency probing
  data/
    models/       ServerConfig, Subscription, AppSettings, traffic, IP info
    parsers/      one parser per protocol + subscription/JSON decoding
    local/        encrypted Hive stores
    remote/       subscription fetch, IP lookup, update check
    repositories/ server list, subscriptions, settings + backup
  features/
    connection/   home screen, tunnel orchestration, IP check
    servers/      list, import (paste/QR/file), share sheet
    subscription/ subscription manager
    settings/     settings, split tunneling
    stats/        usage charts
  shared/         theme, fa/en strings, reusable widgets
android/          VpnService + libbox integration, QS tile, boot receiver
windows/          Flutter desktop runner (tray via tray_manager)
tool/             build_libbox.sh
```

**How a connection is made.** The Dart layer turns your server list plus your settings into a
complete sing-box JSON document — a `selector` outbound holding up to 60 servers, a `urltest` group
for automatic selection and failover, split DNS, routing rules, and a TUN inbound carrying the
split-tunnel package lists. That document is handed to the native `VpnService`, which calls
`libbox.newService()` and hands sing-box a TUN file descriptor built from the options sing-box asks
for. Live stats and instant server switching then run over sing-box's Clash API on a random
loopback port protected by a per-session secret.

**Why routing rules are built in rather than downloaded.** The usual home for `geoip-ir.srs` and
`geosite-ir.srs` is `raw.githubusercontent.com`, which is not reachable from the networks this app
targets. Nexus ships the Iranian bypass list as static rules instead (see
`lib/core/singbox/iran_rules.dart`), so Smart mode works on first launch with no downloads.

---

## Known gaps

- The Android and Windows tunnels have been built and tested through CI, but the native layers have
  not been run on real hardware by the author — treat the first install as a smoke test.
- TUIC servers can be imported and connected to, but have no share-link export.
- The "notify me about faster servers" setting is stored but not yet acted on.
- Split tunneling is Android-only; Windows routes everything the rules allow.

---

<div dir="rtl">

## فارسی

**Nexus** یک کلاینت ضدفیلترینگ چندپروتکلی برای **اندروید** و **ویندوز** است که با Flutter و بر پایهٔ
[sing-box](https://github.com/SagerNet/sing-box) ساخته شده.

این برنامه فقط «کلاینت» است: به سرورهایی وصل می‌شود که شما وارد می‌کنید — از طریق لینک سابسکریپشن،
کانفیگ، کد QR یا فایل. خودش هیچ سرور یا زیرساختی ندارد و نمی‌فروشد.

### امکانات

- پشتیبانی از VMess، VLESS (به‌همراه REALITY و uTLS)، Shadowsocks، Trojan، WireGuard، Hysteria2 و TUIC
- دکمهٔ یک‌کلیکهٔ اتصال، انتخاب خودکار سریع‌ترین سرور و جابه‌جایی خودکار وقتی سروری از کار می‌افتد
- نمایش زندهٔ سرعت آپلود/دانلود، پینگ و مدت اتصال
- تعویض سرور بدون قطع تانل
- لیست یکپارچهٔ سرورها با جستجو، فیلتر پروتکل/کشور، پین کردن و نمایش پینگ رنگی
- چند سابسکریپشن هم‌زمان با به‌روزرسانی خودکار و نمایش خطا به زبان ساده
- **حالت هوشمند**: سایت‌های ایرانی و شبکهٔ محلی مستقیم می‌روند و فقط بقیه از تانل عبور می‌کنند
- **Kill Switch**: اگر تانل قطع شود، ترافیک مسدود می‌شود تا IP واقعی لو نرود
- **تانل تفکیکی** در اندروید: انتخاب اینکه کدام اپ‌ها از VPN رد شوند
- **بررسی IP و نشت DNS**: مقایسهٔ آدرس قبل و بعد از اتصال
- بدون هیچ لاگ ترافیکی؛ اطلاعات ورود سرورها با AES رمزنگاری و کلیدش در Keystore اندروید ذخیره می‌شود
- فارسی و انگلیسی، تم روشن/تاریک/خودکار
- آمار مصرف روزانه و ماهانه، پشتیبان‌گیری و بازیابی، اشتراک‌گذاری سرور با QR
- تایل تنظیمات سریع و اعلان با دکمهٔ قطع فوری

### چطور برنامه را بگیرم؟

۱. به بخش [**Releases**](../../releases) همین مخزن بروید.
۲. فایل مناسب گوشی‌تان را دانلود کنید:

| فایل | مناسب برای |
|---|---|
| `nexus-*-arm64-v8a.apk` | تقریباً همهٔ گوشی‌های ۱۳۹۷ به بعد — **همین را بگیرید** |
| `nexus-*-armeabi-v7a.apk` | گوشی‌های قدیمی‌تر |
| `nexus-*-x86_64.apk` | شبیه‌ساز و تبلت‌های x86 |
| `nexus-*-universal.apk` | همه‌جا کار می‌کند، حجمش بیشتر است |

۳. فایل را روی گوشی باز کنید و اجازهٔ «نصب از منابع ناشناس» را بدهید.

اگر هنوز ریلیزی ساخته نشده، از تب **Actions** گزینهٔ **Build Android → Run workflow** را بزنید؛
بعد از پایان اجرا، فایل‌های APK در بخش **Artifacts** همان اجرا قرار می‌گیرند. برای ساخت یک ریلیز
رسمی هم کافی است یک تگ نسخه پوش کنید:

```bash
git tag v1.0.0 && git push origin v1.0.0
```

### نسخهٔ ویندوز

فایل `nexus-*-windows-x64.zip` را باز کنید و `nexus.exe` را **با دسترسی Administrator** اجرا کنید
(ساخت اینترفیس شبکه در ویندوز به این دسترسی نیاز دارد). آیکون برنامه در System Tray می‌نشیند و
بستن پنجره برنامه را نمی‌بندد، فقط به Tray می‌فرستد تا اتصال قطع نشود.

### بیلد از روی سورس

```bash
export ANDROID_HOME=$HOME/Android/Sdk
tool/build_libbox.sh          # ساخت کتابخانهٔ sing-box
flutter pub get
flutter build apk --release --split-per-abi

flutter build windows --release   # نسخهٔ ویندوز
```

### نکتهٔ امضای برنامه

بدون کلید امضا، فایل‌ها با کلید دیباگ امضا می‌شوند؛ نصب می‌شوند ولی اندروید اجازهٔ به‌روزرسانی از
یک امضا به امضای دیگر را نمی‌دهد. قبل از اولین انتشار عمومی یک keystore بسازید و مقادیر آن را در
Secrets مخزن ثبت کنید (جدول انگلیسی بالا).

</div>

---

## License and credits

Nexus is a client built on [sing-box](https://github.com/SagerNet/sing-box) by SagerNet, which does
the actual protocol work.
