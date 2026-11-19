# Otwarte pytania – architektura informacji

Lista tematów do decyzji po etapie makiet lo-fi. Odhaczaj `[x]` po podjęciu decyzji i dopisz ją pod punktem.

Makiety: płótno „Listonox – wireframes” (strony: Wersja 1 – karty, Wersja 2 – bez kart, Style, Archiwum).

## 1. Wybory na poziomie całej aplikacji

- [ ] **Wersja 1 (karty) czy Wersja 2 (bez kart)?** Albo miks, np. lista z V2 i wątek z V1.
  *Rekomendacja:* V2 jako główna – lepiej oddaje minimalizm.
- [ ] **Które układy listy zostają na desktopie?** Karty, wiersze, lista + wątek w panelu. Czy da się przełączać (menu pod awatarem)? Szerokość kolumny 760 px?
- [ ] **Foldery czy etykiety?** Teraz są obie. To decyzja o modelu danych (IMAP = foldery, Gmail = etykiety).
  *Rekomendacja:* MVP na folderach, etykiety później.
- [ ] **Wiele kont w MVP?**
  *Rekomendacja:* jedno konto; przełącznik kont później w menu pod awatarem.

## 2. Lista maili

- [ ] **Gwiazdka na liście w V2** – czy szeroka lista ma mieć klikalną pustą gwiazdkę jak V1?
- [ ] **Zaznaczanie wielu wiadomości** bez checkboxów.
  *Rekomendacja:* klik w awatar (V1) lub przycisk „Zaznacz” włącza tryb zaznaczania.
- [ ] **Gesty na mobile** – przesunięcie w bok = gwiazdka / archiwizacja?
- [ ] **Starsze maile** – grupy dzienne bez końca czy po tygodniu grupy miesięczne? Czy nagłówki dni przyklejają się przy przewijaniu?
  *Rekomendacja:* tak i tak.
- [ ] **Rok w dacie** tylko dla poprzednich lat (np. „12 grudnia 2025”).

## 3. Wątek

- [ ] **Gwiazdka klikana na poziomie wątku** – propozycja: włączenie oznacza najnowszą wiadomość, wyłączenie zdejmuje gwiazdki ze wszystkich.
  Ustalone: gwiazdka dotyczy pojedynczej wiadomości; jeśli któraś wiadomość w wątku ma gwiazdkę, cały wątek jest oznaczony na liście.
- [ ] **Kolejność wiadomości** – najstarsza na górze (obecnie) czy najnowsza?
- [ ] **Długie wątki** – zwijanie środka do „Pokaż 7 wcześniejszych (12–15 listopada)”.
- [ ] **Nieprzeczytane w wątku** – rozwijać wszystkie nieprzeczytane czy tylko ostatnią?
- [ ] **Format daty w wątku** – potwierdzić regułę: „Wczoraj, 11:05” / „wt., 17 lis, 10:42” / z rokiem dla starszych.

## 4. Odbiorcy i kontakty

- [ ] **Jeden odbiorca w pigułce** – sam adres czy imię (adres po najechaniu)?
- [ ] **„Ty” czy Twój adres** na liście odbiorców?
- [ ] **UDW** – trzecia pigułka „udw:” tylko w wiadomościach wysłanych przez Ciebie.
- [ ] **Wejścia do widoku kontaktu** – klik w nadawcę w wątku, w osobę w pigułce odbiorców, sekcja „Osoby” w wyszukiwarce ⌘K.

## 5. Mobile

- [ ] **„Więcej” na telefonie** – menu jak na desktopie czy arkusz od dołu?
  *Rekomendacja:* arkusz.
- [ ] **Podpisy w doku** (10 px) – zostają czy większe?

## 6. Następne kroki

- [ ] Style: kolory, typografia, zaokrąglenia (w toku – moodboardy).
- [ ] Spisanie ustaleń architektury informacji w `docs/`.

## Zamknięte

- Nawigacja: nagłówek z wyszukiwaniem ⌘K + pływający dok; „Więcej” w doku otwiera mini menu (Spam, Kosz, Etykiety).
- Lista grupowana po dniach, nagłówek dnia = dzień tygodnia + data (+ „Dziś”/„Wczoraj”).
- Akcje pod wiadomością: Odpowiedz, Gwiazdka, Archiwizuj; „Odpowiedz wszystkim” i „Przekaż” w menu „⋯”.
- Wątek pokazuje wiadomości z nagłówkami dni; brak podsumowania „N wiadomości, M z gwiazdką”.
- Odbiorcy jako pigułka: jeden adres w całości, więcej = „N odbiorców”.
- Desktop: wątek otwiera się w wysuwanym panelu po prawej; układ „lista na górze, wątek na dole” odrzucony.
- Widok wszystkich wiadomości z wybraną osobą (filtry: Wszystkie / Od / Do / Załączniki).
