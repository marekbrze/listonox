# Listonox – architektura minimalistycznego klienta e-mail

Notatki projektowe: jak podejść do budowy własnego klienta poczty jako aplikacji webowej, czym różni się wariant natywny (macOS/iOS) i jak to wdrożyć w Dockerze.

## Spis treści

1. [Najważniejsze ograniczenie](#1-najważniejsze-ograniczenie)
2. [Wybór sposobu dostępu do poczty](#2-wybór-sposobu-dostępu-do-poczty)
3. [Architektura aplikacji webowej](#3-architektura-aplikacji-webowej)
4. [Plan budowy etapami](#4-plan-budowy-etapami)
5. [Gdzie naprawdę jest trudno](#5-gdzie-naprawdę-jest-trudno)
6. [Web app czy natywna aplikacja na Maca/iOS?](#6-web-app-czy-natywna-aplikacja-na-macaios)
7. [Wdrożenie w Dockerze](#7-wdrożenie-w-dockerze)
8. [Proponowany stos technologiczny](#8-proponowany-stos-technologiczny)

---

## 1. Najważniejsze ograniczenie

**Przeglądarka nie potrafi rozmawiać z serwerem poczty bezpośrednio.**

Protokoły pocztowe:

- **IMAP** (odbieranie, port 993) i **SMTP** (wysyłanie, port 465/587) działają na surowych połączeniach TCP.
- Przeglądarka zna tylko **HTTP** i **WebSocket**.

Dlatego aplikacja webowa prawie zawsze potrzebuje **własnego backendu**, który tłumaczy jedno na drugie. To główna decyzja architektoniczna i reszta z niej wynika.

Klient poczty jest naprawdę trudny dopiero wtedy, gdy chce się zbudować „drugiego Gmaila”. Wersja minimalistyczna to rozsądny projekt.

---

## 2. Wybór sposobu dostępu do poczty

```mermaid
flowchart TD
    A{Do jakiej poczty<br/>się łączę?} --> B[Tylko Gmail / Outlook]
    A --> C[Dowolna skrzynka<br/>IMAP/SMTP]
    A --> D[Serwer wspiera JMAP<br/>Fastmail, Stalwart]

    B --> B1[REST API dostawcy<br/>Gmail API, Microsoft Graph<br/>+ logowanie OAuth]
    B1 --> B2[⚠ Publiczna aplikacja =<br/>płatny audyt bezpieczeństwa Google]

    C --> C1[Backend jako pośrednik<br/>IMAP/SMTP ↔ HTTP/WebSocket]
    C1 --> C2[✅ Najbardziej uniwersalne<br/>⚠ Najwięcej pracy]

    D --> D1[JSON przez HTTP,<br/>prawie bez backendu]
    D1 --> D2[✅ Najprostsze<br/>⚠ Mało dostawców]

    style C1 fill:#d4edda,stroke:#28a745
```

**Rekomendacja dla MVP:** IMAP/SMTP przez własny backend, na początek **jedno konto (moje)**. Działa z każdą skrzynką, a hasło aplikacji (app password) wystarczy zamiast OAuth.

---

## 3. Architektura aplikacji webowej

```mermaid
flowchart TB
    subgraph Browser["🌐 Przeglądarka"]
        FE["Frontend (React / Svelte / ...)<br/>• lista folderów i wiadomości<br/>• podgląd maila (oczyszczony HTML w iframe sandbox)<br/>• edytor nowej wiadomości"]
    end

    subgraph Backend["⚙️ Backend"]
        API[API: logowanie, sesje]
        IMAP["Moduł IMAP<br/>połączenie + IDLE (push)"]
        MIME["Parser MIME<br/>nagłówki, treść, załączniki"]
        SAN["Sanityzacja HTML<br/>usuwa skrypty, blokuje zdalne obrazki"]
        SMTP[Moduł SMTP: wysyłka]
        DB[("Baza (opcjonalnie)<br/>cache, wyszukiwanie,<br/>zaszyfrowane dane logowania")]
    end

    MAIL[("📮 Serwer poczty<br/>Gmail, Fastmail, ...")]

    FE -- "REST / JSON" --> API
    API -- "WebSocket / SSE:<br/>nowy mail" --> FE
    API --> IMAP
    API --> SMTP
    IMAP --> MIME --> SAN
    API -.-> DB
    IMAP -- "IMAP :993" --> MAIL
    SMTP -- "SMTP :465/587" --> MAIL
```

### Za co odpowiada każda warstwa

| Warstwa | Odpowiedzialność |
|---|---|
| **Frontend** | Interfejs: foldery, lista wiadomości, podgląd, edytor. Bez logiki protokołów pocztowych. |
| **API** | Uwierzytelnienie użytkownika aplikacji, endpointy REST, kanał WebSocket/SSE do powiadomień. |
| **Moduł IMAP** | Utrzymuje połączenie z serwerem, pobiera foldery i wiadomości, nasłuchuje przez `IDLE`. |
| **Parser MIME** | Zamienia surowy mail na nagłówki, treść tekstową/HTML i załączniki, obsługuje kodowania. |
| **Sanityzacja** | Mail to treść od obcych osób: usuwa skrypty, blokuje piksele śledzące i zdalne obrazki. |
| **Moduł SMTP** | Wysyła wiadomości. |
| **Baza** | Opcjonalna na start. Później cache nagłówków, wyszukiwanie, przechowywanie kont. |

---

## 4. Plan budowy etapami

Każdy etap daje działającą aplikację. Nie trzeba robić wszystkiego naraz.

```mermaid
flowchart TD
    E1["<b>Etap 1 – Czytanie bez cache</b><br/>backend łączy się z IMAP na każde żądanie<br/>foldery → lista maili → podgląd"]
    E2["<b>Etap 2 – Wysyłanie</b><br/>SMTP + prosty formularz<br/>(odpowiedz / nowa wiadomość)"]
    E3["<b>Etap 3 – Bezpieczny podgląd HTML</b><br/>DOMPurify + iframe sandbox + CSP"]
    E4["<b>Etap 4 – Na żywo</b><br/>IMAP IDLE → WebSocket → odświeżenie listy"]
    E5["<b>Etap 5 – Cache i synchronizacja</b><br/>baza + sync przyrostowy (UID, CONDSTORE)"]
    E6["<b>Etap 6 – Rozbudowa</b><br/>wątki, wyszukiwanie, załączniki,<br/>wiele kont, OAuth"]

    E1 --> E2 --> E3 --> E4 --> E5 --> E6

    style E1 fill:#d4edda,stroke:#28a745
    style E2 fill:#d4edda,stroke:#28a745
    style E3 fill:#d4edda,stroke:#28a745
    style E5 fill:#fff3cd,stroke:#ffc107
    style E6 fill:#fff3cd,stroke:#ffc107
```

- 🟢 **Etapy 1–3**: kilka weekendów pracy, daje używalnego klienta.
- 🟡 **Etapy 5–6**: tu zaczynają się prawdziwe trudności. Do tego momentu będzie już wiadomo, czy są potrzebne.

---

## 5. Gdzie naprawdę jest trudno

| Obszar | Dlaczego boli | Jak to obejść |
|---|---|---|
| **Parsowanie MIME** | Kodowania, zagnieżdżone części, nietypowe maile | Gotowa biblioteka (`postal-mime`, `mailparser`). Nie pisać własnego parsera. |
| **Bezpieczeństwo HTML** | Treść od obcych osób, czyli ryzyko XSS i pikseli śledzących | Sanityzacja, `<iframe sandbox>`, domyślnie blokowane zdalne obrazki, nagłówki CSP |
| **Synchronizacja** | Rozjazd stanu serwera i lokalnego cache, zmiana `UIDVALIDITY` | Na start nie robić cache (odłożone do etapu 5) |
| **Dane logowania** | Hasła do skrzynek przechowywane na serwerze | Wersja dla jednego użytkownika, hasła aplikacji, szyfrowanie, później OAuth |
| **Gmail OAuth (publicznie)** | Dostęp do poczty to „restricted scope” i wymaga płatnego audytu | Do użytku prywatnego wystarczy IMAP + hasło aplikacji |

---

## 6. Web app czy natywna aplikacja na Maca/iOS?

Natywna aplikacja **może otworzyć połączenie TCP**, więc łączy się z IMAP/SMTP bezpośrednio i nie potrzebuje pośrednika.

```mermaid
flowchart LR
    subgraph Web["Aplikacja webowa"]
        W1[Przeglądarka] -- HTTP --> W2[Własny backend] -- IMAP/SMTP --> W3[(Serwer poczty)]
    end
    subgraph Native["Aplikacja natywna"]
        N1[Aplikacja macOS/iOS] -- IMAP/SMTP --> N3[(Serwer poczty)]
    end
```

### Zalety wersji natywnej

- Nie ma serwera do utrzymania.
- Hasła są w **Keychain** na urządzeniu, a nie w bazie na serwerze.
- Na **macOS** aplikacja może trzymać `IDLE` otwarte cały czas, więc nowe maile pojawiają się od razu.

### Wady i ograniczenia

- **iOS w tle:** system usypia aplikację, więc nie utrzyma połączenia `IDLE`. Natychmiastowe powiadomienia wymagają serwera, który pilnuje skrzynki i wysyła push przez **APNs**. Backend wraca tylnymi drzwiami. Bez niego zostaje odświeżanie po otwarciu aplikacji i okazjonalne pobieranie w tle.
- **Słabsze biblioteki:** w Swifcie nie ma odpowiednika `imapflow`.
  - `MailCore2`: dojrzały, ale stary (C++/Obj-C),
  - `swift-nio-imap` od Apple: nowoczesny, ale niskopoziomowy.
  - Parsowanie MIME też jest uboższe.
- **Dystrybucja:** konto deweloperskie Apple (99 USD rocznie), App Store review i dwie platformy UI, nawet jeśli SwiftUI wiele z tego wspólnie obsłuży.
- **HTML maili** i tak trzeba renderować w `WKWebView`, więc problemy bezpieczeństwa są te same.

### Werdykt

| Kryterium | Web app (Docker) | Natywna Mac/iOS |
|---|---|---|
| Połączenie z pocztą | przez backend | bezpośrednio ✅ |
| Dostęp z każdego urządzenia | ✅ | tylko Apple |
| Powiadomienia na iOS | Web Push (PWA) | wymaga serwera + APNs |
| Dojrzałość bibliotek | ✅ (Node.js) | słabsza |
| Utrzymanie | serwer/VPS | App Store, konto deweloperskie |

Natywnie znika problem pośrednika, ale dochodzą ograniczenia platformy i słabsze narzędzia. **Dla jednej osoby, która chce mieć dostęp z każdego urządzenia, web app w Dockerze jest prostszy w całości.**

---

## 7. Wdrożenie w Dockerze

Najlepsza wersja na początek to **self-hosted dla jednego użytkownika**. Wtedy większość problemów z bezpieczeństwem znika, bo aplikacja nie przechowuje cudzych haseł.

```mermaid
flowchart TB
    User((👤 Ja)) -- HTTPS --> Caddy

    subgraph Compose["docker-compose"]
        Caddy["caddy<br/>reverse proxy + automatyczny certyfikat"]
        App["app (Node.js)<br/>• serwuje zbudowany frontend<br/>• API REST + WebSocket<br/>• imapflow / nodemailer"]
        Vol[("SQLite na wolumenie<br/>(cache – później)")]
        Caddy --> App
        App -.-> Vol
    end

    App -- "IMAP :993<br/>SMTP :587" --> Mail[(📮 Serwer poczty)]
```

### Na co uważać

1. **Proces musi działać stale.** `IDLE` to długo otwarte połączenie, więc potrzebny jest zwykły kontener na VPS lub w domu. Platformy serverless (Vercel, AWS Lambda) się do tego nie nadają.
2. **Wystarczy jeden kontener aplikacji.** Backend może serwować zbudowany frontend, więc osobny kontener na frontend to zbędna komplikacja na start.
3. **HTTPS jest obowiązkowy**, bo przez połączenie idą dane logowania i treść maili. Caddy załatwia to jedną linijką konfiguracji.
4. **Dane logowania** przekazywać przez zmienne środowiskowe albo Docker secrets i nie trzymać ich w repozytorium. Do Gmaila użyć hasła aplikacji.
5. **Logowanie do samej aplikacji:** przy wystawieniu do internetu dodać własne hasło albo schować aplikację za **Tailscale** lub **Cloudflare Access**. To najprostsze i bardzo bezpieczne.
6. **Porty wychodzące:** niektórzy dostawcy VPS blokują port 25, ale klient używa 587 lub 465, więc zwykle nie ma problemu.

---

## 8. Proponowany stos technologiczny

| Element | Wybór | Uwagi |
|---|---|---|
| Backend | **Node.js** (TypeScript) | najlepszy ekosystem bibliotek pocztowych |
| IMAP | [`imapflow`](https://github.com/postalsys/imapflow) | nowoczesne API, obsługa `IDLE` i `CONDSTORE` |
| SMTP | [`nodemailer`](https://nodemailer.com/) | standard de facto |
| Parsowanie MIME | [`postal-mime`](https://github.com/postalsys/postal-mime) | lekki, działa też w przeglądarce |
| Sanityzacja HTML | [`DOMPurify`](https://github.com/cure53/DOMPurify) | + `<iframe sandbox>` + CSP |
| Frontend | React lub Svelte | dowolny, byle prosty |
| Baza (później) | SQLite | wystarczy dla jednego użytkownika |
| Reverse proxy | Caddy | automatyczny HTTPS |
| Dostęp zdalny | Tailscale / Cloudflare Access | bez wystawiania aplikacji publicznie |

---

### Następny krok

Szkielet projektu dla **etapu 1**: `docker-compose.yml`, backend z `imapflow` (lista folderów i maili) oraz prosty frontend.
