# HD430S — ölçmə necə aparılır və sərf necə hesablanır

Bu sənəd sadə dillə izah edir: radar nəyi ölçür, rəqəmlər hansı yoxlamalardan və düzəlişlərdən
keçir, V1, V2, V3, V4 sürətləri necə alınır, anlıq sərf və toplam həcm necə hesablanır.

Hər addımda üç şey var: **nə edirik**, **düstur**, **nümunə**. Nümunə Ceyranbatan kanalının
real oxunuşudur və sonda cihazın bazaya yazdığı sərflə eyni nəticəni verir (12.94 m³/s).

Mənbə: repodakı `docs/CALCULATION_FORMULAS.md` (v1.1) və `FlowCalculator.cpp`.

---

## Qısa sxem

```
1. Radar ölçür: suya qədər məsafə (d) və səthin sürəti (V1)
2. Səhv kodlu oxunuşlar atılır
3. Kalman filtri rəqəmləri hamarlayır
4. Suyun dərinliyi tapılır: h
5. Kanalın formasına görə su axan sahə tapılır: A
6. Sürət yoxlanır (əks axın, çox kiçik sürət)
7. Küləyin təsiri çıxılır            → V2
8. Səth sürəti orta sürətə çevrilir   → V3
9. Anlıq sərf: Q = A × orta sürət
10. Toplam həcm: hər oxunuşda Q × vaxt əlavə olunur
```

---

## 1. Radar nəyi ölçür

Radar kanalın üstündə asılıb və aşağıya baxır. Cihaz onu RS-485 kabeli ilə (Modbus) sorğulayır.
Normalda bu hər 5 saniyədə bir olur, Ceyranbatanda enerjiyə qənaət üçün hər 30 saniyədə bir.

Hər sorğuda radar iki rəqəm verir:

| Rəqəm | Mənası | Nümunə |
|---|---|---|
| d (airGap) | Radardan su səthinə qədər boş məsafə | 3.047 m |
| V1 (v_raw) | Suyun **səthinin** sürəti | 1.666 m/s |

Radar yalnız səthi görür. Suyun dibə yaxın hissəsi daha yavaş axır, ona görə sonradan
səth sürətini orta sürətə çevirmək lazımdır (addım 8).

---

## 2. Səhv kodlu oxunuşların atılması

Radar gündə bir neçə dəfə ölçmə əvəzinə öz səhv kodunu göndərir. Bu kodlar adi rəqəm kimi
görünür, məsələn −32.767 m/s. Belə rəqəm hesaba düşsəydi, sərf bir neçə dəqiqəlik ya 0 olardı,
ya da həddən artıq böyük.

Ona görə bu oxunuşlar **filtrdən əvvəl** atılır:

| Nə | Atılan dəyərlər |
|---|---|
| Sürət | ±32.767, −32.768 m/s (radarın səhv kodları), rəqəm deyilsə, 20 m/s-dən böyükdürsə |
| Məsafə | 32.767, 32.768, 65.534 m (radarın səhv kodları), rəqəm deyilsə |

Atılan oxunuş heç yerə yazılmır: nə bazaya, nə web-ə, nə totalayzerə. Ekranda son düzgün
rəqəm qalır. Növbəti düzgün oxunuş gələndə aradakı vaxt normal hesablanır, həcm itmir.

Səhv kodlar **60 saniyədən çox** dalbadal gəlirsə, radar "etibarsız" sayılır və sərf
hesablanmır. Məsafə üçün 65.535 m kodu ("ölçmə yoxdur") həmişə ölçməni etibarsız edir.

Diaqnostika → Modbus ekranında neçə oxunuşun atıldığı görünür.

---

## 3. Kalman filtri

Radarın rəqəmləri bir oxunuşdan digərinə bir az "titrəyir". Kalman filtri bu titrəməni
hamarlayır, amma suyun real dəyişməsini izləyir. Məsafə və sürət ayrı-ayrı filtrdən keçir.

- Filtrdən çıxan sürət: **v_f**. Sərf bununla hesablanır.
- Radarın filtrsiz sürəti V1 isə hesabat üçün ayrıca saxlanır (V1–V4 zənciri, addım 9).
- Rabitə 60 saniyədən çox kəsilibsə, filtr sıfırdan başlayır.

**Nümunə:** V1 = 1.666 m/s → v_f = **1.640 m/s**

---

## 4. Suyun dərinliyi

Radardan kanalın dibinə qədər məsafə qurulanda bir dəfə ölçülüb yazılır: **H0** (zero_level).
Radar suya qədər məsafəni ölçür. Fərq suyun dərinliyidir.

```
h = H0 − d          (mənfi çıxsa 0, yəni kanal qurudur)
```

- h 2 sm-dən azdırsa, sərf hesablanmır (bu, radarın öz xətası qədərdir).
- Dərinlik kanalın hündürlüyündən (Ht) böyük götürülmür.

**Nümunə:** H0 = 520 sm = 5.200 m → h = 5.200 − 3.047 = **2.153 m**

---

## 5. Su axan sahə (kanalın formasına görə)

Kanalın en kəsiyində suyun tutduğu sahə **A**-dır (m²). Hansı düstur işlənir, kanalın formasından
asılıdır. Kanal ölçüləri cihazda santimetrlə yazılır, hesabda metrə çevrilir.

Bundan əlavə hər forma üçün **B** — su səthinin eni hesablanır. O, külək düzəlişində lazımdır.

| Forma | Ölçülər | Sahə A | Səthin eni B |
|---|---|---|---|
| Düzbucaqlı | en B, hündürlük Ht | A = B × h | B |
| Trapesiya | dib Bb, üst Bt, hündürlük Ht | A = h × (Bb + m × h) | Bb + 2 × m × h |
| Üçbucaq (V) | üst Bt, hündürlük Ht | trapesiya kimi, Bb = 0 | 2 × m × h |
| Dairə (boru) | diametr D | dairə seqmenti (aşağıda) | 2 × √(h × (D − h)) |
| U-kanal (düzbucaqlı + dairə) | D, Ht | dib yarımdairə + üstü düz divar | aşağıda |
| U-kanal (trapesiya + dairə) | D, Bt, Ht | dib yarımdairə + üstü açılan divar | aşağıda |
| Xüsusi | A, B və ya R | sabit A; yoxdursa B × h; yoxdursa R × h | yalnız B verilibsə |

**Trapesiyada m** — yamacın açılması: hündürlük 1 m artanda kanal bir tərəfdə nə qədər genişlənir.

```
m = (Bt − Bb) / (2 × Ht)
```

**Dairə (boru), su qismən doludursa:**

```
r  = D / 2
dc = r − h
A  = r² × arccos(dc / r) − dc × √(2 × r × h − h²)
(boru tam doludursa: A = π × r²)
```

**U-kanallar.** Dib yarımdairədir (r = D/2). Su dibin içindədirsə (h ≤ r), sahə dairə ilə eynidir.
Su yarımdairədən yuxarı qalxanda üstünə divar hissəsi əlavə olunur:

```
Düzbucaqlı + dairə:   A = π × r² / 2 + D × (h − r)

Trapesiya + dairə:    w = D + (Bt − D) × (h − r) / (Ht − r)
                      A = π × r² / 2 + (D + w) / 2 × (h − r)
```

Lazımi ölçü yazılmayıbsa və ya müsbət deyilsə, sərf hesablanmır və ekranda "--" görünür.

**Nümunə (Ceyranbatan, trapesiya: Bb = 210, Bt = 1420, Ht = 400 sm):**

```
m = (14.20 − 2.10) / (2 × 4.00)          = 1.5125
A = 2.153 × (2.10 + 1.5125 × 2.153)       = 11.530 m²
B = 2.10 + 2 × 1.5125 × 2.153             = 8.612 m
```

---

## 6. Sürət yoxlaması

| Vəziyyət | Nə olur | Niyə |
|---|---|---|
| Sürət mənfidir | Sərf = 0, "əks axın" kimi göstərilir | Geriyə axın sərf sayılmır, amma gizlədilmir |
| Sürət 0.05 m/s-dən azdır | Sərf = 0 | Bu qədər kiçik rəqəm küydür |
| Dərinlik 2 sm-dən azdır | Sərf = 0 | Radarın xətası qədərdir |

**Nümunə:** v_f = 1.640 m/s — yoxlamadan keçir.

---

## 7. Külək düzəlişi → V2

Radar suyun yalnız səthini görür, külək isə məhz səthi itələyir:

- külək axın istiqamətində əsəndə səth sürətlənir, radar real sürətdən **çox** göstərir;
- külək axına qarşı əsəndə səth ləngiyir, radar **az** göstərir;
- külək yandan əsəndə təsir demək olar ki, yoxdur.

Küləyin sürəti (W) və istiqaməti (φ) internetdən, MET Norway hava proqnozundan alınır
(10 m hündürlükdə, saatlıq). Saatlar arasındakı anlar üçün iki saatın arası götürülür.

```
Küləyin əsdiyi tərəf    = φ + 180°
θ (bucaq)               = küləyin əsdiyi tərəf − kanalın axın istiqaməti
k_w (əmsal)             = 0.015 + 0.001 × B        (B — su səthinin eni, m)
Δ (düzəliş)             = k_w × W × cos θ × s
V2 (küləksiz sürət)     = sürət − Δ               (mənfi çıxsa 0)
```

- **θ = 0°** — külək axınla eyni istiqamətdə: Δ müsbətdir, sürət azaldılır.
- **θ = 180°** — külək axına qarşı: Δ mənfidir, sürət artırılır.
- **s** — istiqamətə görə əlavə əmsal: külək axın boyunca əsəndə `tail_scale`, axına qarşı
  əsəndə `head_scale`. İkisi də standart olaraq 1.00-dır (yəni təsir etmir), kalibrləmə üçündür.

Düzəliş yalnız bu şərtlərdə işləyir: radar üçün açılıb, kanalın axın istiqaməti yazılıb, cihazın
koordinatı var, hava məlumatı 3 saatdan köhnə deyil. Şərt ödənməsə, sürət dəyişmir və səbəb
`wind_status`-a yazılır.

Külək çox güclüdürsə (düzəliş sürətin 30%-dən çoxudur), nəticə "yüksək külək təsiri" kimi
işarələnir. Hesab dəyişmir, bu, sadəcə xəbərdarlıqdır.

**Nümunə (külək 16.2 m/s, gəldiyi tərəf 342.9°, kanal 150° istiqamətində axır):**

```
Küləyin əsdiyi tərəf = 342.9 + 180      = 162.9°
θ   = 162.9 − 150                        = 12.9°   → cos θ = 0.9748
k_w = 0.015 + 0.001 × 8.612              = 0.02361
Δ   = 0.02361 × 16.2 × 0.9748 × 1.00     = 0.373 m/s
Sürət küləkdən sonra = 1.640 − 0.373     = 1.267 m/s
```

Yəni bu anda külək səthi 0.373 m/s sürətləndirirdi, bu da səth sürətinin ~23%-i qədərdir.

---

## 8. Səth sürətindən orta sürətə: α əmsalı → V3

Kanalda su hər yerdə eyni sürətlə axmır: səthdə ən sürətli, dibdə ən yavaşdır. Sərf üçün
**bütün kəsiyin orta sürəti** lazımdır. Bunun üçün səth sürəti α əmsalına vurulur. α 1-dən kiçikdir.

α üç cür götürülə bilər:

| Rejim | α necə tapılır |
|---|---|
| Sabit | Əl ilə yazılan rəqəm, standart **0.85** (açıq kanallar üçün qəbul olunmuş dəyər) |
| Avtomatik (log-qanun) | Dərinliyə və dibin kələ-kötürlüyünə görə hesablanır (aşağıda) |
| Kalibrləmə cədvəli | "Dərinlik → α" cədvəli doldurulubsa, ondan götürülür; avtomatikdən üstündür |

**Avtomatik rejim (Prandtl–von Kármán düsturu).** Dib nə qədər kələ-kötürdürsə, dibdə su o qədər
çox ləngiyir və α kiçik olur. Su nə qədər dərindirsə, dibin təsiri o qədər az olur və α böyüyür.

```
ks  — dibin kələ-kötürlüyü:  yeni beton 2 mm, köhnə beton 10 mm, torpaq 50 mm,
                              daşlı və ya bitkili 150 mm
z0  = ks / 30
L   = ln(h / z0)
α   = (L − 1 + z0 / h) / L
α   = α × düzəliş əmsalı      (standart 1.000; 0.80–1.25 arası əl ilə dəyişilə bilər)
```

- α həmişə 0.5 ilə 1.0 arasında saxlanılır.
- Su çox dayazdırsa və ya dibin tipi seçilməyibsə, α = 0.85 götürülür.

**Nümunə (köhnə beton, ks = 10 mm, h = 2.153 m):**

```
z0 = 0.010 / 30                       = 0.000333 m
L  = ln(2.153 / 0.000333)             = 8.773
α  = (8.773 − 1 + 0.0002) / 8.773     = 0.886
```

Yəni bu dərinlikdə orta sürət səth sürətinin 88.6%-idir.

---

## 9. Sürət zənciri V1 → V2 → V3 → V4

Bu dörd sürət **hesabat üçündür**: ekranda, bazada və web-də göstərilir ki, hər addımın təsiri
görünsün. Zəncir filtrsiz V1-dən qurulur.

| Sürət | Necə alınır | Mənası | Nümunə |
|---|---|---|---|
| **V1** | radardan | Səthin xam sürəti | 1.666 m/s |
| **V2** | V1 − Δ | Küləksiz səth sürəti | 1.666 − 0.373 = **1.293** |
| **V3** | V2 × α | Kəsiyin orta sürəti | 1.293 × 0.886 = **1.146** |
| **V4** | V3 × β | Radarın kanaldakı yerinə görə düzəlişli orta sürət | β açılmayıbsa V4 = V3 = **1.146** |

**β nədir.** Kanalın ortasında su sahillərdəkindən sürətli axır. Radar ortada deyilsə, gördüyü
sürət kanalın orta sürətindən fərqlənir. β bunu düzəldir:

```
x  — radarın kanalın orta xəttindən məsafəsi
y  = B / 2 − x                          (radardan yaxın sahilə qədər)
β  = 0.875 / (2 × y / B)^(1/7)          (0.7 ilə 1.3 arasında saxlanılır)
```

Ceyranbatanda β söndürülüb (β = 1).

**Diqqət:** sərfin özü V1-dən yox, filtrdən keçmiş v_f-dən hesablanır (addım 10). Ona görə V3 (1.146)
ilə sərfin orta sürəti (1.122) bir az fərqlənir.

---

## 10. Anlıq sərf

```
Orta sürət = (v_f − Δ) × α × (1 − e / 100)
Q          = A × orta sürət                      (m³/s)
```

**e** — operatorun xəta payı (%), standart 0. +1% nəticəni 1% azaldır, −1% 1% artırır.
±50%-dən çox yazılsa, nəzərə alınmır.

Sərf mənfi ola bilməz. Addım 6-dakı hallarda Q = 0.

Bundan sonra **sərfin orta sürəti** də hesablanır. Ekranda "Orta sürət (sərfdən)" və web-də
`water_speed_ms` budur:

```
v_flow = Q / A
```

Bu sayədə həmişə **A × v_flow = Q** olur: ekrandakı sürət, sahə və sərf bir-biri ilə uyğundur.

**Nümunə:**

```
Orta sürət = (1.640 − 0.373) × 0.886 × 1   = 1.122 m/s
Q          = 11.530 × 1.122                 = 12.94 m³/s     (cihazın bazasında: 12.9414)
v_flow     = 12.94 / 11.530                 = 1.122 m/s
```

---

## 11. Toplam həcm (totalayzer)

Hər yeni oxunuşda əvvəlki və indiki sərfin ortası götürülür, aradakı saniyələrə vurulur və
toplama əlavə olunur:

```
Əlavə həcm = (Q əvvəlki + Q indiki) / 2 × Δt      (Δt — iki oxunuş arası, saniyə)
Toplam     = Toplam + Əlavə həcm                  (m³)
```

**Nümunə:** iki oxunuş arası 30 s, sərf 12.94 və 12.98 m³/s:
(12.94 + 12.98) / 2 × 30 = **388.8 m³** əlavə olunur.

Qaydalar:

- iki oxunuş arası 15 dəqiqədən çoxdursa (məsələn, cihaz uzun müddət söndürülüb), bu boşluq
  toplanmır;
- etibarsız oxunuş, əks axın və atılan (səhv kodlu) oxunuş toplanmır;
- cihaz yenidən başlayanda toplam bazadan oxunur, sıfırdan başlamır;
- toplam yalnız operator "sıfırla" düyməsini basanda sıfırlanır.

---

## 12. Nümunənin xülasəsi

| Addım | Rəqəm |
|---|---|
| Radar: məsafə d / səth sürəti V1 | 3.047 m / 1.666 m/s |
| Filtrdən sonra v_f | 1.640 m/s |
| Dərinlik h | 2.153 m |
| Sahə A / səth eni B | 11.530 m² / 8.612 m |
| Külək düzəlişi Δ | 0.373 m/s |
| α (köhnə beton) | 0.886 |
| Orta sürət | 1.122 m/s |
| **Anlıq sərf Q** | **12.94 m³/s** |
| V1 / V2 / V3 / V4 | 1.666 / 1.293 / 1.146 / 1.146 m/s |

---

## 13. Nəticələr harada saxlanır və hara gedir

| Harada | Nə |
|---|---|
| Ekran | Anlıq sərf, toplam həcm, suyun hündürlüyü, orta sürət |
| Baza (`radar_readings`) | Hər oxunuşun bütün addımları: h, A, v_f, V1–V4, Δ, külək, α, Q, v_flow, toplam |
| Web, əsas rəqəmlər | `water_speed_calc` = Q, `water_speed_ms` = v_flow, `water_speed_total_calc` = toplam |
| Web, `velocity_breakdown` | V1–V4, v_f, külək, α, β və statuslar — hər paketdə |
| Web, `weather` | Hava proqnozu: temperatur, külək, rütubət, təzyiq, buludluluq, yağıntı |
