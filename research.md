# UART-BT Konwerter (moduł BT zamiast kabla OTG) - research

Cel: sprzętowy mostek między UART-em kontrolera Bafang (złącze programujące HIGO 5-pin) a
telefonem przez Bluetooth (moduł HM-10) - appka EggSPEED łączy się bez kabla USB OTG.
W README.md (BafSPEED) to punkt "Coming Soon" - **Moduł BT zamiast OTG - 20%**.

Ten plik istnieje, żeby NIE zaczynać tematu od zera przy każdej kolejnej rozmowie - zbiera
wszystko co ustalone, potwierdzone i wciąż otwarte.

## Lista zakupowa (BOM) - kompletna, sprawdzona 09.09.2026

Cały układ, jedna lista, żeby nie dowiadywać się o kolejnych brakujących częściach osobno:

| # | Część | Status |
|---|---|---|
| 1 | HM-10 (moduł BLE) | ✅ posiadany |
| 2 | Przetwornica step-down 5-60V→5V | ✅ posiadana |
| 3 | 4-kanałowy konwerter poziomów logicznych (BSS138) | ✅ posiadany |
| 4 | MOSFET BSS123 (SOT-23) | ✅ posiadany |
| 5 | Stabilizator 3,3V (np. AMS1117-3.3) | ✅ posiadany |
| 6 | Złącze HIGO 5-pin | ✅ posiadane |
| 7 | Rezystor 10kΩ (pull-down Gate→PL) | ⬜ do kupienia - grosze, dowolny sklep. [Pakiet 50szt na Allegro](https://allegro.pl/oferta/10k-1-rezystor-opornik-10k-0-6w-1-10kohm-x50szt-7624943364) |

**To jest cała lista** - żadnych innych komponentów układ nie wymaga. Jeśli w przyszłości pojawi
się coś nowego, trafi tu, w tę samą tabelę, zamiast być wspominane tylko w rozmowie.

## Status (09.09.2026): schemat scalony, wersje 1 i 2 połączone

Wcześniej ten katalog opisywał dwie osobne, niespójne wersje projektu (wersja 1 - stały mostek
P+/PL bez zdalnego sterowania; wersja 2 - dodanie MOSFET-a do zdalnego włącz/wyłącz). **09.09
schemat (`bt-bridge-wiring.html`, ten sam plik co Artifact "BT Bridge Wiring") został scalony w
jedną, spójną wersję** - patrz sekcja "Aktualny schemat" niżej. Artifact zaktualizowany pod tym
samym linkiem: https://claude.ai/code/artifact/c50ae686-184f-428b-8734-26c2a34c23c5

## ⚠️ KONFLIKT: dwa źródła podają różną numerację pinów HIGO (09.09.2026, nierozstrzygnięte)

Mamy dwa źródła numeracji pinów złącza HIGO 5-pin i **się nie zgadzają**:

| Pin | Rozbiórka kabla programującego (`KabelUSB_1/2.jpeg`) | Zdjęcie referencyjne (`Higo5.PNG`) |
|---|---|---|
| 1 | P+ | GND |
| 2 | PL | TxD |
| 3 | RxD | P+ (48V) |
| 4 | GND | RxD |
| 5 | TxD | PL ("Power Lock") |

**Ważne sprostowanie co do rozbiórki kabla** (bo poprzednia wersja tego pliku to zawyżała):
zdjęcia `KabelUSB_1.jpeg`/`KabelUSB_2.jpeg` pokazują stronę USB kabla (płytkę z układem
USB-serial) - widać na nich TYLKO kolory przewodów: 3 przewody (czerwony/zielonkawy/czarny)
idą do padów TXD/RXD/GND na płytce, a 2 przewody (niebieski+żółty) są zlutowane razem, osobno.
To potwierdza SAM MECHANIZM (dwa sygnały zwarte ze sobą = P+/PL), ale **nie pokazuje numerów
pinów na samym złączu HIGO** - te numery (1=P+, 2=PL...) pochodzą tylko z wątku na forum,
pośrednio, nigdy nie były zweryfikowane bezpośrednio na złączu.

`Higo5.PNG` wygląda na oficjalny/sklepowy diagram referencyjny złącza (opisany "Female Bafang
variant") - prawdopodobnie bardziej wiarygodny niż wątek na forum, ale też niepotwierdzony
przez nas fizycznie.

**Przed jakimkolwiek lutowaniem: zmierzyć multimetrem napięcie na każdym pinie fizycznej
wtyczki (przy podłączonej baterii) - pin z ~30-60V to P+, to jedyny pewny sposób.** Cały
obecny schemat (`bt-bridge-wiring.html`) używa WCIĄŻ numeracji z rozbiórki kabla (1=P+, 2=PL,
3=RxD, 4=GND, 5=TxD) - jeśli multimetr potwierdzi zdjęcie referencyjne zamiast tego, schemat
trzeba będzie przenumerować.

- W oryginalnym kablu (fabrycznym) **P+ i PL są zwarte ze sobą kawałkiem drutu** - to jedyny
  sposób na wybudzenie kontrolera bez podłączonego prawdziwego wyświetlacza (brak jakiejkolwiek
  elektroniki/sygnału DTR w tym miejscu). W naszym moście BT ten drut zastępuje teraz MOSFET
  BSS123 (patrz niżej) - sterowalny, nie stały.
- **P+ to realne napięcie baterii (30-60V), NIE 5V** - wcześniejsza, jeszcze starsza wersja
  tego schematu błędnie zakładała że P+ to bezpieczne 5V (pomylone z wyjściem kabla USB) -
  to zostało skorygowane.

## Elementy (zdjęcia w tym katalogu)

- **HM-10** (`HM-10.png`) - moduł BLE, chip CC2541F256. Piny: RXD, TXD, GND, VCC (3,6-6V), PIO
  (nowy - steruje bramką MOSFET-a, patrz niżej). TEN KONKRETNY egzemplarz ma tylko 4 "stałe"
  piny - nie wyprowadza wewnętrznego 3,3V, więc potrzebny jest osobny regulator 3,3V dla LVcc
  konwertera poziomów.
- **Przetwornica step-down P+ → 5V** (`DC-DC_StepDown_DC5-60V_5V.PNG`) - moduł 5-60V → 5V o
  **stałym** wyjściu (4 piny IN+/IN-/OUT+/OUT-, kondensator wejściowy rated 63V, bez
  potencjometru). **Zdecydowana, jedyna przetwornica w projekcie** - wcześniejszy kandydat
  (nastawny XL7015) odrzucony i usunięty z dokumentacji/schematu 09.09.2026.
- **4-kanałowy konwerter poziomów logicznych** (`Konwerter.png`, oparty na tranzystorach
  BSS138) - dwie niezależne szyny: HVcc (5V, z przetwornicy) i LVcc (3,3V, z osobnego
  regulatora np. AMS1117-3.3). Kanały H3↔L3 (TxD kontrolera → RXD HM-10) i H4↔L4 (TXD HM-10 →
  RxD kontrolera) to fizyczne przejścia przez tę samą płytkę.
- **MOSFET BSS123** (`BSS_123.jpeg`) - N-kanałowy, logic-level, SOT-23, **100V/0,17A ciągłe**
  (potwierdzone realnym datasheetem Fairchild). Zastępuje dawny stały mostek drutowy P+/PL -
  Drain→P+ (pin1), Source→PL (pin2), Gate sterowany **bezpośrednio** z pinu PIO modułu HM-10
  (3,3V logiki wystarcza, próg bramki ~1-2,5V, bez dodatkowego drivera/bufora ANI rezystora
  szeregowego - ten ostatni jest tylko opcjonalną dobrą praktyką, nie wymogiem, więc świadomie
  go pomijamy dla prostoty). Jedyny rezystor w tym torze to **rezystor podciągający Gate→PL
  (~10kΩ)** - ten MA realną funkcję: trzyma MOSFET domyślnie WYŁĄCZONY (kontroler wyłączony)
  podczas startu/resetu HM-10, gdy stan PIO jest jeszcze niezdefiniowany/pływający. Bez niego
  kontroler mógłby się przypadkowo włączyć przy starcie modułu BT.
- **Stabilizator 3,3V** (`Stabilizator.PNG`, np. AMS1117-3.3, moduł 12,3×8,6mm) - piny
  VIN/OUT/GND. Zasila LVcc konwertera poziomów, bo ten egzemplarz HM-10 nie wyprowadza
  wewnętrznego 3,3V. VIN z szyny +5V (za przetwornicą), GND wspólna, OUT → LVcc.
- **Złącze HIGO 5-pin** (`Wtyczka_HIGO5.png`).

### Dlaczego BSS123 (100V/0,17A) wystarcza - realne dane prądowe

Specyfikacja wyświetlacza Bafang DPC18 (https://california-ebike.com/products/bafang-color-display-dpc18):
- prąd znamionowy: 10mA
- maks. prąd roboczy: 30mA
- prąd upływu standby: <1µA
- zasilanie do kontrolera: 50mA

Realne prądy na linii P+/PL to pojedyncze dziesiątki mA - ogromny zapas względem 170mA
MOSFET-a.

### Odrzucone opcje przełącznika P+/PL (i dlaczego)

- **NTR4170N** - pomyłka z pamięci, sprawdzony datasheet pokazał **tylko 30V** - stanowczo za
  mało (potrzeba ≥60V z zapasem), NIE UŻYWAĆ.
- **Gotowe moduły przekaźników "10A 250VAC/10A 30VDC"** (popularne niebieskie płytki Arduino) -
  rating **30VDC to za mało** (P+ przy w pełni naładowanym pakiecie może sięgać ~58V) - problem
  z napięciem, nie z prądem (prąd 10A to i tak absurdalny nadmiar na tę aplikację).
- **Przekaźnik kontaktronowy (reed relay, np. Panasonic TQ2-5V)** - rozważony jako opcja z
  izolacją galwaniczną (100-200V DC rating), ale odrzucony na rzecz MOSFET-a - mniejszy,
  prostszy, izolacja niepotrzebna (HM-10 i tak ma wspólną masę z resztą układu).

## Aktualny schemat połączeń (skrót - pełny diagram w bt-bridge-wiring.html)

```
P+ (pin1, 30-60V)  -> przetwornica IN+                          -> BSS123 Drain
przetwornica OUT+ (+5V) -> HVcc konwertera -> VCC modułu HM-10 (bezpośrednio)
                         -> VIN regulatora 3,3V (np. AMS1117-3.3) -> LVcc konwertera
GND (pin4)          -> wspólna masa (obie strony układu, w tym IN-/OUT- przetwornicy)
TxD (pin5)          -> H3 -> L3 -> RXD (HM-10)
RxD (pin3)          <- H4 <- L4 <- TXD (HM-10)
PL (pin2)           <- BSS123 Source
BSS123 Gate         <- PIO (HM-10), bezpośrednio (bez rezystora szeregowego)
BSS123 Gate         -> R ~10kΩ -> PL (pull-down, domyślnie OFF - jedyny rezystor w tym torze)
```

## Do potwierdzenia przed lutowaniem (wciąż otwarte)

- Numeracja/nazwy pinów HIGO potwierdzone tylko pośrednio (wątek na forum), nie oficjalnym
  schematem producenta - **zweryfikować multimetrem przed lutowaniem na stałe**.
- Rzeczywisty poziom logiki UART kontrolera (3,3V czy 5V) - **nieustalony i celowo
  nierozstrzygnięty**: nie da się tego wiarygodnie zmierzyć zwykłym multimetrem (stan wysoki
  trwa milisekundy). HVcc ustawione na 5V jako bezpieczny nadzbiór - BSS138 podciąga
  rezystorem do HVcc, więc jeśli kontroler realnie steruje na 3,3V (aktywny driver), ten
  sygnał "wygrywa" z biernym podciągnięciem - nic się nie uszkadza w żadnym z wariantów.

## Pliki w tym katalogu

| Plik | Co to jest |
|---|---|
| `bt-bridge-wiring.html` | Pełny interaktywny schemat (scalony, 09.09.2026) - ten sam co Artifact "BT Bridge Wiring" |
| `HM-10.png` | Zdjęcie modułu HM-10 (piny: RXD/TXD/GND/VCC) |
| `KabelUSB_1.jpeg`, `KabelUSB_2.jpeg` | Zdjęcia z rozbiórki oryginalnego kabla programującego - potwierdzenie zwarcia P+/PL i numeracji pinów HIGO |
| `Konwerter.png` | Zdjęcie 4-kanałowego konwertera poziomów logicznych (BSS138) |
| `DC-DC_StepDown_DC5-60V_5V.PNG` | Zdjęcie wybranej przetwornicy 5-60V→5V o stałym wyjściu (podpięta w schemacie) |
| `Przetwornica.png` | Zdjęcie XL7015 - **odrzucony kandydat**, zostawione na dysku jako archiwum, usunięte z dokumentacji/schematu |
| `BSS_123.jpeg` | Zdjęcie MOSFET-a BSS123 (SOT-23) - zdalny włącznik P+/PL |
| `Stabilizator.PNG` | Zdjęcie stabilizatora 3,3V (np. AMS1117-3.3) - zasila LVcc konwertera poziomów |
| `Wtyczka_HIGO5.png` | Zdjęcie złącza HIGO 5-pin |
| `Higo5.PNG` | **Zdjęcie referencyjne z numeracją pinów - KONFLIKT z rozbiórką kabla, patrz sekcja na górze pliku** |
| `HM-10_appka_screenshot.jpg` | Ten sam HM-10, użyty jako screenshot w README.md BafSPEED (sekcja Coming Soon) |

## Do uzgodnienia (następnym razem zacznij tutaj)

1. Zweryfikować multimetrem numerację pinów HIGO przed lutowaniem (patrz wyżej).
2. Fizyczne złożenie układu na płytce/prototypie - schemat i dobór elementów są już gotowe
   (przetwornica, MOSFET, konwerter poziomów, HM-10 - wszystko ustalone i spójne), brakuje
   faktycznego montażu i testu na prawdziwym kontrolerze.
