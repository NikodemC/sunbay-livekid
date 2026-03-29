# Integracja Zoho → Fakturownia

Integracja umożliwia automatyczne przenoszenie faktur z systemu **Zoho Billing** do **Fakturowni** oraz synchronizację statusów **KSeF** (Krajowy System e-Faktur) z powrotem do Zoho. Całość jest obsługiwana przez arkusz **Google Sheets** pełniący rolę kolejki i rejestru operacji, przy użyciu API platformy Sunbay.

---

## Struktura arkusza Google Sheets

| Zakładka | Rola |
|----------|------|
| **Zoho Invoices Queue** | Bieżąca kolejka faktur do przetworzenia |
| **Invoice Action History** | Pełny log wszystkich operacji |
| **Fakturownia KSeF Sync** | Status KSeF każdej faktury |
| **Config** | Parametry konfiguracyjne |

---

## Przepływ danych

```
Zoho Billing
    │
    ▼  (1) Import invoices
Google Sheets – Zoho Invoices Queue
    │
    ▼  (2) Copy to Fakturownia
Fakturownia
    │
    ▼  (3) KSeF Sync
Zoho Billing (pola KSeF)
```

> Operacje można uruchamiać ręcznie z menu **Sunbay** w arkuszu lub skonfigurować jako automatyczne wyzwalacze Google Apps Script.

---

## Operacje

### 1. Import faktur z Zoho do arkusza

Pobiera nowe faktury z Zoho i dodaje je do kolejki w arkuszu.

**Przebieg:**

1. Pobiera faktury z wybranego widoku Zoho (`zohoKsefViewId` z zakładki Config).
2. Sprawdza w bazie Sunbay, które faktury były już wcześniej zaimportowane.
3. Nowe faktury trafiają do zakładki **Zoho Invoices Queue** ze statusem `New`.
4. Operacja jest zapisywana w zakładce **Invoice Action History**.

**Edge cases:**

- Faktury bez identyfikatora Zoho są pomijane – nie blokują importu pozostałych.
- Faktury już zaimportowane nie są duplikowane – operacja jest bezpieczna do wielokrotnego uruchomienia.
- Jeśli nie ma nowych faktur, operacja kończy się normalnie z wynikiem 0 zaimportowanych.

---

### 2. Kopiowanie faktur do Fakturowni

Dla faktur ze statusem `New` tworzy odpowiadające im faktury w Fakturowni.

**Przebieg:**

1. Wczytuje wiersze z **Zoho Invoices Queue** i filtruje te ze statusem `New`.
2. Przetwarza faktury partiami – rozmiar partii konfigurowany przez `batchSizeCopy` w zakładce Config.
3. Dla każdej faktury:
   - Pobiera pełne dane z Zoho Billing.
   - Mapuje pola Zoho na format Fakturowni według poniższych reguł (szczegóły w sekcji [Reguły mapowania](#reguły-mapowania-zoho-→-fakturownia)).
   - Tworzy fakturę w Fakturowni.
   - Zapisuje ID faktury Fakturowni z powrotem do Zoho jako pole `fakturownia_invoice_id`.
   - Aktualizuje wiersz w Zoho Invoices Queue: status `Copied`, ID/numer faktury Fakturowni, link do faktury.

**Edge cases:**

- Faktura już skopiowana (istnieje mapowanie w bazie) → pomijana jako `Skipped`, status aktualizowany na `Copied`.
- Duże liczby zapisane przez Google Sheets w notacji naukowej (np. `2.39E+17`) są automatycznie konwertowane do postaci pełnej przed wywołaniem API Zoho.
- Jeśli zapis ID Fakturowni z powrotem do Zoho nie powiedzie się → faktura i tak jest oznaczana jako `Copied`, a błąd jest logowany w historii. Nie blokuje to dalszego przetwarzania.
- Jeśli pobranie faktury z Zoho lub stworzenie faktury w Fakturowni się nie powiedzie → wiersz otrzymuje status `CopyFailed`.
- Błąd pojedynczej faktury nie zatrzymuje przetwarzania pozostałych w partii.

> **Uwaga dot. rabatów:** Parametr `PRESERVE_DISCOUNTS` (aktualnie `false`) kontroluje, czy rabaty z Zoho są przenoszone do Fakturowni. Przy wartości `false` pozycje są zapisywane z kwotami finalnymi, po uwzględnieniu rabatu.

---

### 3. Retry faktur CopyFailed

Ponowne przetworzenie faktur, które nie zostały skopiowane z powodu błędu.

Identyczna logika jak operacja 2 – jedyna różnica to filtrowanie faktur ze statusem `CopyFailed` zamiast `New`. Rozmiar partii konfigurowany przez `batchSizeRetry` w zakładce Config.

Operacja jest bezpieczna do wielokrotnego uruchomienia – faktury już skopiowane są automatycznie pomijane.

---

### 4. Synchronizacja statusów KSeF

Pobiera status KSeF z Fakturowni i zapisuje go z powrotem do Zoho.

**Przebieg:**

1. Wczytuje zakładki **Zoho Invoices Queue** i **Fakturownia KSeF Sync** z arkusza.
2. Wybiera faktury z ID Fakturowni, których status KSeF nie był jeszcze zapisany do Zoho.
3. Pobiera dane KSeF z Fakturowni hurtowo dla wybranej partii (rozmiar: `batchSizeKsefSync`).
4. Dla każdej faktury:
   - Odczytuje status KSeF (`gov_status`) i kategoryzuje go:
     - `ok` / `demo_ok` → sukces KSeF
     - Inny niepusty status → błąd KSeF
     - Pusty / brak → `Unknown` (KSeF jeszcze nieprzetworzone)
   - Jeśli status jest znany (ok lub błąd) → aktualizuje pola niestandardowe w Zoho:
     - Status, numer KSeF i daty (wysłania, sprzedaży)
     - Link do weryfikacji i link do faktury KSeF
     - Komunikaty błędów (tylko przy błędnym statusie)
   - Aktualizuje wiersz w zakładce **Fakturownia KSeF Sync**.

**Edge cases:**

- Faktury ze statusem `Unknown` → Zoho nie jest aktualizowane; faktura zostanie sprawdzona ponownie przy kolejnym uruchomieniu.
- Faktura z już pomyślnie zapisanym statusem KSeF w Zoho → pomijana w kolejnych uruchomieniach.
- Jeśli aktualizacja Zoho się nie powiedzie → status w zakładce **Fakturownia KSeF Sync** oznaczany jako `Failed`; faktura zostanie podjęta ponownie przy następnym uruchomieniu.
- Daty są przekazywane do Zoho w formacie `yyyy-MM-dd HH:mm:ss`; puste daty są pomijane.

---

## Reguły mapowania Zoho → Fakturownia

### Daty

| Pole Fakturowni | Źródło |
|-----------------|--------|
| Data wystawienia | `InvoiceDate` z Zoho → fallback: pole `Date` → fallback: bieżąca data UTC |
| Data sprzedaży | identyczna z datą wystawienia |
| Termin płatności | `DueDate` z Zoho → jeśli brak: `data wystawienia + PaymentTermsDays` → jeśli brak obu: równa dacie wystawienia |

### Waluta

Przepisywana z Zoho. Jeśli pole jest puste, przyjmowana jest wartość domyślna `PLN`.

### Miejsce wystawienia

Pobierane z adresu organizacji Zoho (miasto). Wymaga podania `zohoOrganizationId` w konfiguracji.

### Pozycje faktury

- Ilość zaokrąglana do liczby całkowitej; wartości ≤ 0 traktowane jako 1.
- Jeśli faktura w Zoho nie ma żadnych pozycji, tworzona jest jedna zastępcza pozycja z numerem faktury jako opisem i wartością netto równą `SubTotal` (lub `Total` gdy brak `SubTotal`).

### Obsługa rabatów

Kontrolowana przez parametr `PRESERVE_DISCOUNTS`:

- **`false` (domyślnie):** rabat jest wkalkulowany w cenę jednostkową netto – Fakturownia nie widzi oddzielnego rabatu. Logika:
  - jeśli `item_total` ≥ `price × qty` (rabat na poziomie całej faktury) → cena netto = `(item_total − discount_amount) / qty`
  - jeśli `item_total` < `price × qty` (rabat na poziomie pozycji) → cena netto = `item_total / qty`
- **`true`:** oryginalna cena z Zoho jest zachowana, a procent rabatu przekazywany jako osobne pole do Fakturowni.

### Pola niestandardowe Zoho (custom fields)

#### Nabywca

| Pole Zoho | Efekt w Fakturowni |
|-----------|-------------------|
| `cf_nip` | NIP nabywcy – typ rozpoznawany automatycznie: zaczyna się od litery → `nip_ue` (NIP UE); zaczyna się od cyfry → polski NIP; brak NIP + kraj ≠ PL → `empty` (dozwolone); brak NIP + kraj PL → **błąd** (NIP wymagany) |
| `cf_nabywca` | Nadpisuje nazwę nabywcy pobraną z Zoho (opcjonalne) |
| `cf_czy_nabywca_to_jst` | Jeśli `true` → faktura oznaczana jako wystawiona dla jednostki samorządu terytorialnego (JST) |

#### Odbiorca

Odbiorca faktury jest wypełniany **tylko wtedy, gdy jednocześnie** pole `Attention` w adresie dostawy Zoho (imię i nazwisko odbiorcy) **oraz** pole `cf_rola_odbiorcy` są niepuste. Brak któregokolwiek z nich powoduje, że faktura jest tworzona bez odbiorcy – nie jest to błąd.

| Pole Zoho | Efekt w Fakturowni |
|-----------|-------------------|
| `ShippingAddress.Attention` | Imię i nazwisko odbiorcy |
| `cf_rola_odbiorcy` | Rola odbiorcy (np. kupujący, płatnik) |
| `cf_nip_odbiorcy` | NIP odbiorcy – ta sama logika rozpoznania typu co dla nabywcy |

### Konwersja nazw krajów

Nazwy krajów z Zoho są automatycznie konwertowane na kody ISO 3166-1 alfa-2. Obsługiwane są zarówno nazwy angielskie, jak i polskie (`Polska` → `PL`, `Niemcy` → `DE`, `Wielka Brytania` → `GB` itp.) oraz niestandardowe warianty (`u.s.a.` → `US`).

---

## Statusy faktur

### Zoho Invoices Queue – statusy wierszy

| Status | Znaczenie |
|--------|-----------|
| `New` | Zaimportowana z Zoho, czeka na skopiowanie |
| `Copied` | Pomyślnie skopiowana do Fakturowni |
| `CopyFailed` | Błąd podczas kopiowania – do ponowienia przez Retry |

### Fakturownia KSeF Sync – status aktualizacji Zoho

| Status | Znaczenie |
|--------|-----------|
| `Pending` | Status KSeF jeszcze nieznany (Fakturownia nie przetworzyła) |
| `Updated` | Status KSeF zapisany w Zoho – operacja zakończona |
| `Failed` | Błąd podczas zapisu do Zoho – zostanie podjęta ponownie |

---

## Konfiguracja (zakładka Config)

| Klucz | Opis |
|-------|------|
| `zohoKsefViewId` | ID widoku w Zoho, z którego importowane są faktury |
| `departmentId` | ID działu w Fakturowni przypisywany do tworzonych faktur |
| `batchSizeCopy` | Liczba faktur przetwarzanych jednorazowo podczas kopiowania |
| `batchSizeRetry` | Liczba faktur przetwarzanych jednorazowo podczas retry |
| `batchSizeKsefSync` | Liczba faktur sprawdzanych jednorazowo podczas synchronizacji KSeF |

---

## Uwagi operacyjne

- **Kolejność operacji:** Import → Copy (lub Retry) → KSeF Sync. Operacje można uruchamiać niezależnie, ale mają sens tylko w tej kolejności.
- **Idempotentność:** Każda operacja jest bezpieczna do wielokrotnego uruchomienia – duplikaty są wykrywane i pomijane.
- **Diagnostyka:** Zakładka **Invoice Action History** przechowuje kompletny log wszystkich operacji z timestampami i komunikatami błędów. To główne narzędzie do debugowania problemów.
- **Błędy częściowe:** Błąd pojedynczej faktury nie zatrzymuje przetwarzania całej partii – pozostałe faktury są dalej przetwarzane.
