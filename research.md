# UART-BT Konwerter (moduł BT zamiast kabla OTG) - research

Cel: sprzętowy mostek między UART-em kontrolera Bafang (złącze programujące HIGO 5-pin) a
telefonem przez Bluetooth - appka EggSPEED łączy się bez kabla USB OTG.

Rozważane są dwa, niekompatybilne ze sobą moduły - **HC-06** i **HM-10** - patrz sekcja
"HC-06 kontra HM-10" niżej. Celem numer 1 jest obsługa **OEM Bafang** (zdecydowana większość
użytkowników EggSPEED), bbs-fw dopiero po nim.

Ten plik zbiera wszystko, co dotąd ustalone, potwierdzone i wciąż otwarte w tym projekcie.

## Poziom logiki UART kontrolera: 3,3V (ustalone)

Rozbiórka oryginalnego kabla programującego (`KabelUSB_1.jpeg`, `KabelUSB_2.jpeg`) pokazała, że
w środku jest zwykła płytka USB-UART z opisanymi pinami 5V/VCC/3V3/TXD/RXD/GND, a żółta zworka
łączy VCC z 3V3 - czyli logika, na której ten kabel rozmawia z kontrolerem, to 3,3V, nie 5V. Do
złącza HIGO idą tylko TXD, RXD i GND (P+/PL są zwarte osobno, patrz niżej) - DTR/RTS nie są
podłączone.

Takich kabli sprzedaje się tysiące i działają na kontrolerach BBS (OEM i bbs-fw) - to
przyjmujemy jako pewnik, bez dalszych zastrzeżeń. Konsekwencja: **konwerter poziomów logicznych
i osobny regulator 3,3V są zbędne** - oba moduły (HC-06, HM-10) mają logikę 3,3V i podłącza się
je bezpośrednio.

## Status: uproszczony mostek, bez MOSFET-a na razie

Wcześniejsze wersje tego projektu (do 09.09.2026) zakładały konwerter poziomów logicznych
(BSS138), osobny stabilizator 3,3V dla HM-10 i MOSFET BSS123 do zdalnego włączania/wyłączania
kontrolera przez PIO. Po ustaleniu poziomu 3,3V (wyżej) konwerter i stabilizator odpadają.
**MOSFET na razie też odpada** - P+ i PL są po prostu zwarte przewodem, dokładnie jak w
oryginalnym kablu OTG (patrz "Aktualny schemat połączeń" niżej). Skutek: kontroler jest włączony
przez cały czas, gdy mostek jest wpięty, a przetwornica pobiera niewielki prąd spoczynkowy z P+
nawet gdy rower stoi (P+ ma napięcie baterii niezależnie od tego, czy kontroler jest włączony).

Stary, bardziej rozbudowany schemat z MOSFET-em i konwerterem poziomów zostaje w repo
(`diagram.svg`, `bt-bridge-wiring.html`, `Schemat_hybrydowy.png`) jako poprzednia wersja, ale nie
jest już domyślnym planem budowy - domyślny jest uproszczony schemat V2 (`diagram_v2.svg`).

## Numeracja pinów HIGO - wyjaśnione 09.09.2026

Mieliśmy chwilowo zapisane jako "konflikt" dwa źródła numeracji pinów, które pozornie się nie
zgadzały:

| Pin | Rozbiórka kabla programującego / strona KONTROLERA (`KabelUSB_1/2.jpeg`, `Wtyczka_HIGO5.png`) | `Higo5.PNG` - strona WYŚWIETLACZA |
|---|---|---|
| 1 | P+ | GND |
| 2 | PL | TxD |
| 3 | RxD | P+ (36V, 48V, 52V) |
| 4 | GND | RxD |
| 5 | TxD | PL ("Power Lock") |

**To nie jest sprzeczność.** `Wtyczka_HIGO5.png` i `Higo5.PNG` to zdjęcia DWÓCH RÓŻNYCH,
parujących się połówek tego samego złącza - jedna od strony kontrolera, druga od strony
wyświetlacza. Kontroler i wyświetlacz to dwie płcie tej samej wtyczki, patrzące na siebie
"twarzą w twarz" przy łączeniu - stąd naturalnie odwrócona numeracja pozycji między nimi,
dokładnie tak jak przy patrzeniu na dowolne złącze od strony styków vs od strony przeciwnej.

**Do naszego mostka BT liczy się WYŁĄCZNIE strona kontrolera** (bo to w to złącze się
podłączamy, zastępując wyświetlacz) - `Higo5.PNG` pokazuje inne, niepowiązane bezpośrednio
złącze i zostaje w tym katalogu tylko jako materiał poglądowy, nie jako konkurencyjne źródło
numeracji.

**Wciąż otwarte, mimo wyjaśnienia:** numeracja strony kontrolera (1=P+, 2=PL, 3=RxD, 4=GND,
5=TxD) pochodzi tylko pośrednio z wątku na forum, nigdy nie zweryfikowana bezpośrednio na
złączu ani oficjalnym schematem producenta. **Nadal warto zmierzyć multimetrem napięcie na
każdym pinie fizycznej wtyczki (przy podłączonej baterii, pin z ~30-60V to P+) przed
lutowaniem na stałe** - dla pewności, że to pośrednie źródło się nie myli, niezależnie od
sprawy z `Higo5.PNG`.

- W oryginalnym kablu (fabrycznym) **P+ i PL są zwarte ze sobą kawałkiem drutu** - to jedyny
  sposób na wybudzenie kontrolera bez podłączonego prawdziwego wyświetlacza (brak jakiejkolwiek
  elektroniki/sygnału DTR w tym miejscu). W naszym moście BT ten sam sposób zostaje: zwarcie
  przewodem, bez MOSFET-a na razie (patrz wyżej).
- **P+ to realne napięcie baterii (30-60V), NIE 5V** - wcześniejsza, jeszcze starsza wersja
  tego schematu błędnie zakładała że P+ to bezpieczne 5V (pomylone z wyjściem kabla USB) -
  to zostało skorygowane.

## HC-06 kontra HM-10

Oba moduły mają logikę 3,3V i tę samą prędkość docelową (1200 baud), ale **to dwa różne,
niekompatybilne ze sobą protokoły Bluetooth** - nie da się napisać jednej obsługi "Bluetooth" w
EggSPEED, tylko dwie osobne integracje za wspólnym interfejsem transportu.

- **HC-06** - Bluetooth Classic (profil SPP). Paruje się jak bezprzewodowy kabel szeregowy:
  Windows i Android widzą go jako wirtualny port szeregowy (RFCOMM). Po sparowaniu zachowuje się
  dokładnie jak oryginalny kabel OTG - wymusza stałe załączenie kontrolera, dopóki moduł jest
  zasilany. **Nie ma programowalnego wyjścia** (żadnej komendy AT do sterowania pinem) - jedyny
  pin ze stanem to LED/STATE, który przed sparowaniem miga (fala prostokątna ok. 102 ms), a
  dopiero po sparowaniu jest stale wysoki, więc nie nadaje się wprost do sterowania bramką
  MOSFET-a bez filtrowania (RC).
- **HM-10** - Bluetooth Low Energy (BLE). Aplikacja rozmawia z nim przez usługę/charakterystykę
  GATT, w małych paczkach (ok. 20 bajtów) - inny sposób wymiany danych po stronie Androida niż
  RFCOMM. W odróżnieniu od HC-06 ma programowalny pin PIO (komenda AT), co pozwala w przyszłości
  zbudować zdalny, sterowany programowo wyłącznik kontrolera (MOSFET na P+/PL), zamiast stałego
  załączenia.

**Kolejność testów:** najpierw test na OEM (cel numer 1), dopiero potem na bbs-fw. Wybór, który
moduł dostanie pierwszą integrację w EggSPEED, zapada po testach na OEM.

## Elementy (zdjęcia w tym katalogu)

- **HM-10** (`HM-10.png`) - moduł BLE, chip CC2541F256. Piny: RXD, TXD, GND, VCC (3,6-6V), PIO
  (niewykorzystane na razie - patrz "Status" wyżej, potencjalnie do zdalnego wyłącznika w
  przyszłości). TEN KONKRETNY egzemplarz ma tylko 4 "stałe" piny - nie wyprowadza wewnętrznego
  3,3V, ale to teraz bez znaczenia, bo zasilany jest wprost 5V z przetwornicy (VCC 3,6-6V).
- **HC-06 (typ ZS-040)** - moduł Bluetooth Classic, oferta np. Botland (49,90 PLN, wysyłka 24h,
  piny niezlutowane). Zasilanie 3,6-6V (własny stabilizator na płytce), logika komunikacyjna
  3,3V (producent deklaruje tolerancję 5V, ale sam zaleca ostrożność). Domyślnie 9600 baud, PIN
  1234. Komendy AT (z dołączonej dokumentacji sklepu): `AT` (test), `AT+BAUDx` (1=1200, 2=2400,
  3=4800, 4=9600, 5=19200, 6=38400, 7=57600, 8=115200, zapisywane trwale), `AT+NAMEx`,
  `AT+PINxxxx`, parzystość (`AT+PN/PO/PE`). Brak komend GPIO/PIO. Wejście RXD modułu nie ma
  rezystora podciągającego - jeśli źródło sygnału (TxD kontrolera) jest typu open-drain, trzeba
  dodać pull-up.
- **BT-06 (DSD TECH)** - odpowiednik HC-06 od konkretnego producenta (układ BC417), tylko 4 piny
  (VCC, GND, TXD, RXD, bez LED/KEY), 3,6-6V, logika "TTL level 3,3V", domyślnie 9600 baud, PIN
  1234, tylko slave. Ta sama rodzina komend AT co HC-06/HC-05 tego producenta. Dostępność:
  gorsza niż generyczny HC-06/ZS-040 - **wybór padł na zwykły HC-06** (Allegro, od ok. 20 zł,
  dostępny krajowo bez opłat celnych, w przeciwieństwie do zamówień z AliExpress).
- **Przetwornica step-down P+ → 5V** (`DC-DC_StepDown_DC5-60V_5V.PNG`) - moduł 5-60V → 5V o
  **stałym** wyjściu (4 piny IN+/IN-/OUT+/OUT-, kondensator wejściowy rated 63V, bez
  potencjometru). **Zdecydowana, jedyna przetwornica w projekcie.** IN- i OUT- są połączone
  wewnątrz modułu (typ nieizolowany) - do zweryfikowania miernikiem ciągłości przed budową.
- **Złącze HIGO 5-pin** (`Wtyczka_HIGO5.png`).

### Komendy AT HC-06 (referencja)

Ustawienia fabryczne: slave, 9600 N81, nazwa "linvor", PIN 1234.

```
AT                  test komunikacji, odpowiedź OK

AT+BAUD1            1200
AT+BAUD2            2400
AT+BAUD3            4800
AT+BAUD4            9600
AT+BAUD5            19200
AT+BAUD6            38400
AT+BAUD7            57600
AT+BAUD8            115200

AT+NAMEname1        zmiana nazwy urządzenia
AT+PIN1234          zmiana kodu parowania
AT+VERSION          wersja firmware

AT+PN               brak parzystości (tylko wersje firmware >1.5)
AT+PO               parzystość nieparzysta
AT+PE               parzystość parzysta
```

Dla naszego mostka istotna jest `AT+BAUD1` (1200 baud, docelowa prędkość kontrolera Bafang).

## Aktualny schemat połączeń (uproszczony, bez konwertera i bez MOSFET-a)

Pełny diagram wektorowy: [`diagram_v2.svg`](diagram_v2.svg). Poprzedni, bardziej rozbudowany
wariant (z konwerterem poziomów i MOSFET-em) nadal w [`diagram.svg`](diagram.svg) i
[`bt-bridge-wiring.html`](bt-bridge-wiring.html).

```
P+ (pin1, 30-60V)  -> zwarte z PL (pin2), jak w oryginalnym kablu OTG
P+ (pin1)          -> przetwornica IN+
GND (pin4)          -> przetwornica IN-
przetwornica OUT-   -> GND modułu BT (IN- i OUT- połączone wewnątrz przetwornicy - nie ma
                        osobnego przewodu GND-GND)
przetwornica OUT+ (+5V) -> VCC modułu BT (HM-10 lub HC-06/BT-06 - ten sam schemat dla obu)
TxD (pin5)          -> RXD modułu BT (bezpośrednio, 3,3V)
RxD (pin3)          <- TXD modułu BT (bezpośrednio, 3,3V)
```

## Do potwierdzenia / do zrobienia przed lutowaniem

- Numeracja/nazwy pinów HIGO potwierdzone tylko pośrednio (wątek na forum), nie oficjalnym
  schematem producenta - **zweryfikować multimetrem przed lutowaniem na stałe**.
- **Ustawić moduł BT na 1200 baud** komendą AT (`AT+BAUD1` dla HC-06/BT-06; dla HM-10 numer
  komendy zależy od wersji firmware - sprawdzić `AT+VERS?`). Wymaga adaptera USB-UART z logiką
  3,3V - może nim być sama płytka z oryginalnego kabla programującego (ryzyko: trzeba ją
  rozpiąć, a jest potrzebna do programowania roweru) albo osobny adapter (CP2102/CH340/FTDI).
  Komendy bez znaku końca linii, moduł musi być niesparowany. HC-06 nie przyjmuje komend AT
  przez radio (w odróżnieniu od HM-10) - tylko przez UART.
- **Zmienić domyślny PIN modułu** (`AT+PIN`) - domyślne 1234/000000 pozwalają każdemu w zasięgu
  się sparować i zapisać konfigurację do kontrolera.
- Przy zakupie HC-06 (klony różnej jakości) **przetestować każdy egzemplarz** przed lutowaniem:
  `AT` -> `OK`, `AT+VERSION`, `AT+BAUD1` -> `OK1200`.
- Fizyczne złożenie układu na płytce/prototypie i test na prawdziwym kontrolerze - **najpierw
  OEM, potem bbs-fw** (patrz "HC-06 kontra HM-10" wyżej).

## Pliki w tym katalogu

| Plik | Co to jest |
|---|---|
| `diagram_v2.svg` | Aktualny, uproszczony schemat (bez konwertera poziomów, bez MOSFET-a) |
| `bt-bridge-wiring.html` | Poprzedni, pełny interaktywny schemat (z konwerterem i MOSFET-em, scalony 09.09.2026) |
| `HM-10.png` | Zdjęcie modułu HM-10 (piny: RXD/TXD/GND/VCC) |
| `KabelUSB_1.jpeg`, `KabelUSB_2.jpeg` | Zdjęcia z rozbiórki oryginalnego kabla programującego - potwierdzenie zwarcia P+/PL, numeracji pinów HIGO i poziomu logiki 3,3V |
| `Konwerter.png` | Zdjęcie 4-kanałowego konwertera poziomów logicznych (BSS138) - element poprzedniej wersji, już niepotrzebny |
| `DC-DC_StepDown_DC5-60V_5V.PNG` | Zdjęcie wybranej przetwornicy 5-60V→5V o stałym wyjściu (nadal używana) |
| `Przetwornica.png` | Zdjęcie XL7015 - **odrzucony kandydat**, zostawione na dysku jako archiwum, usunięte z dokumentacji/schematu |
| `BSS_123.jpeg` | Zdjęcie MOSFET-a BSS123 (SOT-23) - element poprzedniej wersji, na razie nieużywany |
| `Wtyczka_HIGO5.png` | Zdjęcie złącza HIGO 5-pin |
| `Higo5.PNG` | Zdjęcie referencyjne złącza HIGO **od strony wyświetlacza** (nie kontrolera) - materiał poglądowy, patrz wyjaśnienie wyżej |
| `Stabilizator.PNG` | Zdjęcie stabilizatora 3,3V - element poprzedniej wersji, już niepotrzebny |
| `Schemat_hybrydowy.png` | Poprzedni schemat hybrydowy (na zdjęciach realnych płytek, z konwerterem i MOSFET-em) |
| `HM-10_appka_screenshot.jpg` | Ten sam HM-10, użyty jako screenshot w README.md BafSPEED |

## Do uzgodnienia (następnym razem zacznij tutaj)

1. Kupić moduł HC-06 (Allegro, krajowo, od ok. 20 zł - nie AliExpress, opłaty celne zjadają
   oszczędność) i przetestować go komendami AT przed lutowaniem.
2. Ustawić moduł (HC-06 lub HM-10, ten który testujemy jako pierwszy) na 1200 baud.
3. Złożyć uproszczony układ (`diagram_v2.svg`) na płytce/prototypie.
4. Przetestować na prawdziwym kontrolerze - **najpierw OEM Bafang, potem bbs-fw**: najpierw
   odczyt/telemetria, dopiero potem zapis.
