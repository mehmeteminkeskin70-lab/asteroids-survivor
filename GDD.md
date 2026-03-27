# ASTEROIDS: SURVIVOR — Oyun Tasarım Dokümanı (GDD)

**Sürüm:** Kod tabanına göre (tek dosya: `index.html`)  
**Platform:** Web (HTML5 Canvas + JavaScript), yerel `file://` veya statik host  
**Tür:** Roguelite / hayatta kalma / bullet hell — üstten bakışlı arena, sürekli düşman basıncı, meta ilerleme

---

## 1. Özet (Elevator pitch)

Oyuncu geniş bir uzay arenasında (6000×6000 dünya) asteroitleri, düşman gemilerini ve boss’ları yok ederek XP ve coin toplar. Seviye atladıkça **rastgele item kartları**, **Diep.io tarzı stat puanları** ve belirli seviyelerde **kalıcı “gelişim dalı” (Evo)** seçer. Amaç: **30 dakikalık koşuyu tamamlamak (zafer)** veya ölene kadar en yüksek skoru / coin’i toplamak. Kalıcı **mağaza yükseltmeleri** ve **başarımlar** tekrar oynanabilirliği besler.

---

## 2. Hedefler ve son koşullar

| Sonuç | Koşul |
|--------|--------|
| **Zafer** | Oyun süresi `RUN_TARGET_TICKS` = 30×60×60 tick (~30 dk, 60 tick/saniye varsayımı) dolduğunda `victoryRun()` |
| **Yenilgi** | Can segmentleri tükendiğinde `gameOver()` |
| **Rekabetçi mod** | Sabit **UZMAN** zorluk; haftalık skor tablosuna kayıt |

---

## 3. Oyun modları

### 3.1 Macera (Adventure)
- Oyuncu **Kolay / Orta / Zor / Uzman** seçer.
- `diffMult`: easy 0.55, medium 0.8, hard 1, expert 1.4 — spawn, hasar, elite şansı vb. ile ölçeklenir.

### 3.2 Rekabetçi (Competitive)
- Zorluk sabit **UZMAN**.
- Skor haftalık `leaderboard` anahtarına (`getWeekId`) yazılır.

### 3.3 Roguelite
- Alt kurallar:
  - **Hazine Avcısı:** +40% coin, −15% hasar (`coinM` / `dmgM`).
  - **Cam Gövde:** +35% hasar, **max can tavanı 2** (`hpCap`).
  - **Kaos:** İlk **3 seviye** rastgele item yükseltmesi + seviye/Xp eşiği ilerletilir.

---

## 4. Karakterler (run başı seçim)

| ID | İsim | Rol |
|----|------|-----|
| vanguard | Vanguard | Dengeli; her zaman açık |
| berserker | Berserker | +%18 hasar, −1 başlangıç canı — `kill_500` |
| ghost | Hayalet | +%12 hız, max can ~%87 — `surv_300` |
| gambler | Kumarbaz | +%25 sandık şansı hissi, kalkan yenileme cezası, yüksek luck — `coin50` |
| miner | Madenci | +%22 coin, −%8 hasar — `kill_200` |

---

## 5. Kontroller

| Girdi | İşlev |
|--------|--------|
| WASD / ok tuşları | Hareket |
| Fare | Nişan (gemi açısı) |
| Space / sol tık | Ateş |
| Shift | Focus (yavaş hareket, bullet time etkileşimi varsa) |
| V | Dash (cooldown) |
| Q / R | EMP (stamina maliyeti, cooldown) |
| E / F / B | Bomba |
| ESC / P | Duraklat |
| Seviye atlayınca **1–3** | Item kartı seçimi |
| Evo ekranında **1–4** | Dal seçimi |
| Oynarken **1–8** | Diep stat harcaması (veya HUD’daki +) |
| T / Y / U | Ek drone satın alma (collector / raider / repair, coin ile) |

---

## 6. Çekirdek döngü (bir run)

1. Menüden mod → (Macera’da) zorluk → karakter → oyun başlar.
2. Düşman öldürme → XP gem + coin → seviye dolar.
3. Seviye atlayınca **3 karttan 1 item** (aktif veya pasif, max 12 seviye/item).
4. Her seviyede **stat puanı** (`statPts`) ile **8 Diep** statından birine yatırım.
5. **Seviye 6:** birincil Evo; **Seviye 14:** ikincil Evo (koşu boyunca kalıcı).
6. Zaman ilerledikçe **tehlike (`dangerLevel`)**, adaptive zorluk (`adaptDiff`), boss dalgası ve (ara sıra) **Kovan Taşıyıcı** boss fight.
7. Haritada **bölgeler** (iyileşme, hasar, hız, XP bonusu, mayın), **event** (malzeme kutusu, ödül avı), **sandık + slot makinesi**.
8. Run sonu: coin kalıcı havuza, rekorlar güncellenir; başarımlar tetiklenebilir.

---

## 7. Can ve kalkan

- **Çok segmentli can:** `lives` × segment HP (`hpSegMax`); mevcut segment `pHp`.
- **Kalkan:** Tek kullanımlık; kırılınca `shieldRegenRate` ve Diep[0] ile yeniden dolma hızı etkilenir.
- Çarpışmalarda kalkan varsa bazı durumlarda **Diep “gövde”** ek hasarı uygulanır.

---

## 8. Diep — 8 stat (Diep.io tarzı)

Her biri **DIEP_MAX = 12** seviyeye kadar; `statPts` ile artar (tuş 1–8 veya HUD).

| # | Kod etiketi | Oyundaki rol (özet) |
|---|-------------|---------------------|
| 0 | Kalkan hız | Kalkan yenileme süresine çarpan (`diepRegenMul`) |
| 1 | Max can | Her 2 seviyede +1 max can; segment max HP’ye katkı |
| 2 | Gövde | Kalkanlı çarpışmalarda ek gövde hasarı |
| 3 | Mermi hız | Ana mermi hızına ek |
| 4 | Delici | Her 2 seviyede +1 pierce; mermi ömrü bonusu |
| 5 | Hasar | Tüm oyuncu hasarına çarpan |
| 6 | Atış hızı | Ateş aralığına çarpan |
| 7 | Hareket | Max hıza çarpan |

---

## 9. Gelişim dalları (Evo)

**Seviye 6 — Tier 0 (birincil)**

| ID | İsim | Etki (tasarım özeti) |
|----|------|----------------------|
| twin | İkiz Kol | Atışta şansla ek çift zayıf mermi |
| sniper | Nişancı | Mermi hızı çarpanı (`_evoBulletSpd`) |
| flank | Kanat | Şansla arkaya yarım güç mermi |
| spray | Sünger | Daha sık ateş + minimum yayılım (`_evoMinSpread`) |

**Seviye 14 — Tier 1 (ikincil)**

| ID | İsim | Etki (tasarım özeti) |
|----|------|----------------------|
| vamp | Ganimet | Boss dışı öldürmelerde ekstra coin patlaması |
| thick | Zırh | +1 max can, segment senkronu |
| magnet | Çekim | XP/coin çekim gücü (`_evoMagMul` → `pMagnet` / coin `magRange` ölçeği) |
| fury | Öfke | Kombo artış hızı |

---

## 10. Item sistemi

### 10.1 Aktif item’lar (max 12)
scatter, beam, seeker, nova, orbit, rail, mine — ateş, tetik, güdümlü, patlama, yörünge, ray, ölümde mayın.

### 10.2 Pasif item’lar (max 12)
thrust, matrix, barrier, magnet, crit, pen, servo — hız/ivme, hasar, kalkan süresi, çekim, kritik, delici, ateş hızı.

### 10.3 Fusion (Evrilim)
Aynı indeksteki **aktif + pasif ikisi de 12** olduğunda otomatik açılır (7 çift):

1. scatter + thrust → **Kuyruklu Yıldız Serpmesi**  
2. beam + matrix → **Keskin Sürekli Işın**  
3. seeker + magnet → **Ganimet Avcıları**  
4. nova + crit → **Kritik Süpernova**  
5. orbit + barrier → **Kalkan Halkası**  
6. rail + pen → **Sınırsız Hat**  
7. mine + servo → **Seri Mayın**

---

## 11. Dronlar

- Başlangıçta **2 collector** drone.
- **T / Y / U** ile coin karşılığı ek drone (tür başına üst sınır ve fiyat ölçeklemesi).
- Türler: toplayıcı (coin/XP), saldırı (raider), tamir — çizim ve AI `updateDrones` içinde.

---

## 12. Dünya ve etkinlikler

- **Map zone’lar:** İyileşme, hasar, hız, XP bonusu, mayın alanı — süreli, görsel etiketli.
- **Event:** Malzeme kutusu (yaklaşınca ödül), ödül avı (sarı elite hedef).
- **Sandık:** Slot sembolleri (coin, xp, heart, bomb, star, skull); şans ve `pLuck` / `pJackpot` ile etkileşim.
- **Boss:** Periyodik boss dalgası + **Kovan Taşıyıcı** — arena sınırları, özel pattern’ler.

---

## 13. Meta ekonomi ve ilerleme

- **Coin:** Run içi `coinsEarned` + zafer/ölüm bonusları → kalıcı `SAVE.coins`.
- **Mağaza (PERM):** Hasar, hız, can, mıknatıs, kalkan, bomba, XP, coin şansı, luck, sandık frekansı — seviye ve maliyet eğrileri kodda sabit.
- **Kayıt:** `localStorage` anahtarı `ast_surv_v3` (eski `ast_surv_v2` migrasyonu).
- **Günlük seri:** İlk girişte bonus coin.
- **Başarımlar:** Kill, süre, seviye, boss, kombo, coin, sandık, seri, run sayısı, 30 dk zafer vb. — çoğu +3 run coin’i verir.

---

## 14. Ses ve sunum

- **SFX:** Ateş, vuruş, patlama, level up, powerup, bomba, hasar, EMP, vb.
- **Müzik:** ~140 BPM basit loop; sadece `playing` state’te.
- **UI:** Skor, can barı, kombo, coin, süre, 30 dk geri sayım hissi, boss barı, minimap, item ikonları, fusion listesi, Diep şeridi, yüzen metinler (`uiFloats`), tema rengi zamana göre kayar (`updateTheme`).

---

## 15. Teknik notlar (tasarım için)

- Tek HTML; harici asset zorunluluğu yok.
- Performans: object pool’lar (mermi, parçacık, XP, coin, düşman mermisi).
- **Adaptive difficulty:** Ölüm/kill oranına göre `adaptDiff` ile denge.

---

## 16. Bilinen sınırlar / notlar

- README’deki “Canlı GitHub Pages” linki harici bir repoya işaret eder; yerel kopyada yayın yoksa sadece `index.html` açılır.
- Eski `meteor` event’i kodda kaldırılmış (yorum satırı).

---

## 17. Hızlı başvuru — vitrin URL’leri

- **Yerel:** `asteroids-survivor/index.html` dosyasını tarayıcıda aç.
- **Örnek uzak host (README):** `https://mehmeteminkeskin70-lab.github.io/asteroids-survivor/`

---

*Bu doküman oyundaki mevcut davranışı yansıtır; denge veya metin değişiklikleri kod güncellemesiyle birlikte GDD’yi revize etmeyi unutma.*
