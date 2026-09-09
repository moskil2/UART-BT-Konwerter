# UART-BT Konwerter

Sprzętowy mostek między UART-em kontrolera Bafang (złącze programujące HIGO 5-pin) a telefonem
przez Bluetooth (moduł HM-10) - appka [EggSPEED](https://github.com/moskil2/EggSPEED) łączy się
bez kabla USB OTG. W README EggSPEED to punkt "Coming Soon" - **Moduł BT zamiast OTG - 20%**.

To repo istnieje, żeby nie zaczynać tematu od zera przy każdej kolejnej sesji projektowej -
zbiera wszystko co ustalone, potwierdzone i wciąż otwarte. Pełna, szczegółowa wersja tej
dokumentacji: [`research.md`](research.md). Pełny interaktywny schemat (z podpiętymi zdjęciami,
klikalny): [`bt-bridge-wiring.html`](bt-bridge-wiring.html).

## Lista zakupowa (BOM) - kompletna

| # | Część | Status |
|---|---|---|
| 1 | HM-10 (moduł BLE) | ✅ posiadany |
| 2 | Przetwornica step-down 5-60V→5V | ✅ posiadana |
| 3 | 4-kanałowy konwerter poziomów logicznych (BSS138) | ✅ posiadany |
| 4 | MOSFET BSS123 (SOT-23) | ✅ posiadany |
| 5 | Stabilizator 3,3V (np. AMS1117-3.3) | ✅ posiadany |
| 6 | Złącze HIGO 5-pin | ✅ posiadane |
| 7 | Rezystor 10kΩ (pull-down Gate→PL) | ⬜ do kupienia - grosze, dowolny sklep. [Pakiet 50szt na Allegro](https://allegro.pl/oferta/10k-1-rezystor-opornik-10k-0-6w-1-10kohm-x50szt-7624943364) |

To jest cała lista - żadnych innych komponentów układ nie wymaga.

## ⚠️ Konflikt numeracji pinów HIGO - nierozstrzygnięty

Mamy dwa źródła numeracji pinów złącza HIGO 5-pin i **się nie zgadzają**:

| Pin | Rozbiórka kabla programującego | Zdjęcie referencyjne (`Higo5.PNG`) |
|---|---|---|
| 1 | P+ | GND |
| 2 | PL | TxD |
| 3 | RxD | P+ (48V) |
| 4 | GND | RxD |
| 5 | TxD | PL ("Power Lock") |

Rozbiórka kabla (`KabelUSB_1.jpeg`, `KabelUSB_2.jpeg`) potwierdza tylko **mechanizm** (dwa
sygnały - P+/PL - zwarte ze sobą kawałkiem drutu w oryginalnym kablu), nie same numery pinów -
te pochodzą pośrednio z wątku na forum. `Higo5.PNG` wygląda na oficjalny diagram referencyjny
("Female Bafang variant"), ale też niepotwierdzony przez nas fizycznie.

**Przed jakimkolwiek lutowaniem: zmierzyć multimetrem napięcie na każdym pinie fizycznej
wtyczki (przy podłączonej baterii) - pin z ~30-60V to P+.** Schemat poniżej używa wciąż
numeracji z rozbiórki kabla (1=P+, 2=PL, 3=RxD, 4=GND, 5=TxD).

## Schemat połączeń

<img src="diagram.svg" alt="Schemat: HIGO 5-pin -> MOSFET BSS123 -> przetwornica step-down -> konwerter poziomów BSS138 -> HM-10" width="100%">

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

## Elementy

| Element | Zdjęcie | Opis |
|---|---|---|
| **HM-10** | ![HM-10](HM-10.png) | Moduł BLE, chip CC2541F256. Piny: RXD, TXD, GND, VCC (3,6-6V), PIO (steruje bramką MOSFET-a). Ten egzemplarz nie wyprowadza wewnętrznego 3,3V - potrzebny osobny regulator dla LVcc konwertera. |
| **Przetwornica step-down P+ → 5V** | ![Przetwornica](DC-DC_StepDown_DC5-60V_5V.PNG) | Moduł 5-60V → 5V, stałe wyjście, 4 piny IN+/IN-/OUT+/OUT-, kondensator wejściowy rated 63V. Jedyna, zdecydowana przetwornica w projekcie (wcześniejszy kandydat XL7015 odrzucony). |
| **Konwerter poziomów logicznych** | ![Konwerter](Konwerter.png) | 4-kanałowy, oparty na BSS138. Dwie niezależne szyny: HVcc (5V) i LVcc (3,3V, z osobnego regulatora np. AMS1117-3.3). |
| **MOSFET BSS123** | ![BSS123](BSS_123.jpeg) | N-kanałowy, logic-level, SOT-23, 100V/0,17A ciągłe. Zastępuje dawny stały mostek drutowy P+/PL - Drain→P+, Source→PL, Gate←PIO (HM-10), **bezpośrednio, bez rezystora szeregowego** (to tylko opcjonalna dobra praktyka, nie wymóg - świadomie pominięty dla prostoty). Jedyny rezystor w tym torze to rezystor podciągający Gate→PL (~10kΩ), który trzyma MOSFET domyślnie WYŁĄCZONY przy starcie/resecie HM-10. |
| **Stabilizator 3,3V** | ![Stabilizator](Stabilizator.PNG) | Np. AMS1117-3.3, moduł 12,3×8,6mm, piny VIN/OUT/GND. Zasila LVcc konwertera poziomów (ten egzemplarz HM-10 nie wyprowadza wewnętrznego 3,3V). |
| **Złącze HIGO 5-pin** | ![HIGO5](Wtyczka_HIGO5.png) | Złącze programujące kontrolera. |
| **Zdjęcie referencyjne HIGO5** | ![Higo5 ref](Higo5.PNG) | Podaje INNĄ numerację pinów niż rozbiórka kabła - patrz sekcja konfliktu wyżej. |

### Dlaczego BSS123 (100V/0,17A) wystarcza

Specyfikacja wyświetlacza Bafang DPC18
([california-ebike.com](https://california-ebike.com/products/bafang-color-display-dpc18)):
prąd znamionowy 10mA, maks. roboczy 30mA, upływ w standby <1µA, zasilanie do kontrolera 50mA.
Realne prądy na linii P+/PL to pojedyncze dziesiątki mA - ogromny zapas względem 170mA MOSFET-a.

### Odrzucone opcje przełącznika P+/PL

- **NTR4170N** - pomyłka z pamięci, datasheet pokazał tylko 30V - za mało, NIE UŻYWAĆ.
- **Gotowe moduły przekaźników "10A 250VAC/10A 30VDC"** - rating 30VDC za mało (P+ może sięgać ~58V).
- **Przekaźnik kontaktronowy (reed relay)** - rozważony, odrzucony na rzecz MOSFET-a (mniejszy, izolacja niepotrzebna).

## Zdjęcia z rozbiórki oryginalnego kabla programującego

| | |
|---|---|
| ![Kabel 1](KabelUSB_1.jpeg) | ![Kabel 2](KabelUSB_2.jpeg) |

Potwierdzają: 3 przewody (TXD/RXD/GND) idą do płytki USB-serial, 2 przewody (P+/PL) są
zlutowane razem, osobno - to mechanizm wybudzenia kontrolera bez prawdziwego wyświetlacza.

## Do potwierdzenia przed lutowaniem

- **Numeracja pinów HIGO - konflikt dwóch źródeł, patrz sekcja na górze.** Zweryfikować multimetrem.
- Rzeczywisty poziom logiki UART kontrolera (3,3V czy 5V) - nieustalony i celowo
  nierozstrzygnięty: HVcc konwertera ustawione na 5V jako bezpieczny nadzbiór, BSS138 podciąga
  rezystorem do HVcc, więc aktywny driver 3,3V ze strony kontrolera i tak "wygrywa" - nic się
  nie uszkadza w żadnym wariancie.

## Do zrobienia

1. Zweryfikować multimetrem numerację pinów HIGO przed lutowaniem.
2. Fizyczne złożenie układu na płytce/prototypie - schemat i dobór elementów są już gotowe,
   brakuje faktycznego montażu i testu na prawdziwym kontrolerze.

## Pliki w repo

| Plik | Co to jest |
|---|---|
| `research.md` | Pełna, szczegółowa wersja tej dokumentacji |
| `bt-bridge-wiring.html` | Pełny interaktywny schemat (ten sam co Artifact "BT Bridge Wiring") |
| `diagram.svg` | Sam wektorowy schemat (bez zdjęć), użyty w tym README |
| `HM-10.png`, `HM-10_appka_screenshot.jpg` | Zdjęcia modułu HM-10 |
| `KabelUSB_1.jpeg`, `KabelUSB_2.jpeg` | Rozbiórka oryginalnego kabla programującego |
| `Konwerter.png` | Konwerter poziomów logicznych (BSS138) |
| `DC-DC_StepDown_DC5-60V_5V.PNG` | Wybrana przetwornica step-down |
| `Przetwornica.png` | XL7015 - odrzucony kandydat, archiwum |
| `BSS_123.jpeg` | MOSFET BSS123 |
| `Wtyczka_HIGO5.png`, `Higo5.PNG` | Złącze HIGO 5-pin (drugie - zdjęcie referencyjne, konflikt numeracji) |
| `Stabilizator.PNG` | Stabilizator 3,3V (np. AMS1117-3.3) |
