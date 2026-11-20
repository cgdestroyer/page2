# Email Signature Builder — specyfikacja odtworzeniowa

Dokument opisuje projekt tak, aby inny agent AI (lub developer) mógł zbudować
funkcjonalnie równoważny edytor od zera, bez dostępu do oryginalnego kodu.
Oryginał: pojedynczy, samodzielny plik `email-signature-editor/index.html`
(~2700 linii, HTML + CSS + JS w jednym `<script>`), bez bundlera i bez
zależności npm.

---

## 1. Cel produktu

Wizualny builder **sygnatur e-mail typu "email-safe"**: użytkownik składa
sygnaturę z klocków, a wynik to czysty HTML oparty na **tabelach i inline
style** (bez `<style>`, bez flexboxa, bez klas), który wkleja się do Gmaila,
Outlooka, Apple Mail itp. i wygląda identycznie.

Builder musi:
- pozwolić edytować wizualnie (drag & drop, klik, edycja inline),
- utrzymać "profil" (imię, zdjęcie, stanowisko, kontakty, social) rozdzielony
  od layoutu, tak by zmiana szablonu nie kasowała danych,
- **importować istniejącą sygnaturę** (wklejony HTML / plik) i automatycznie
  rozpoznać jej elementy,
- eksportować czysty HTML gotowy do wklejenia,
- działać w 100% w przeglądarce, offline, z autosave w `localStorage`.

Grupa docelowa: osoby nietechniczne + marketing/IT ujednolicający sygnatury.

---

## 2. Ograniczenia techniczne (twarde wymagania)

1. **Jeden plik HTML**, otwierany `file://`, bez kroku budowania.
2. **Vanilla JS** w jednym IIFE (`(function(){ ... })()`), `"use strict"`.
   Brak frameworków (React/Vue), brak jQuery.
3. **Wyjściowy HTML musi być email-safe:**
   - layout wyłącznie przez `<table>`/`<tr>`/`<td>` z atrybutami
     `cellpadding="0" cellspacing="0" border="0" role="presentation"` i
     `style="border-collapse:collapse"`,
   - **wszystkie style inline** (atrybut `style="..."` na elementach),
   - obrazy: `<img>` z atrybutem `width`, `style="display:block;border:0"`,
     `alt`; ostrzegać przed obrazami `data:` (base64),
   - linki: `mailto:`, `tel:`, `https://`,
   - żadnych znaczników skryptów/stylów w eksporcie.
4. Dane trzymane w `localStorage` (autosave z debounce ~300 ms).
5. Działa w nowoczesnych przeglądarkach; funkcje opcjonalne (EyeDropper API,
   `navigator.clipboard.read`, `ClipboardItem`) wykrywać warunkowo i mieć
   fallback.

---

## 3. Model danych (źródło prawdy)

Jeden obiekt `state`:

```js
state = {
  profile,       // wartości pól: { name, title, company, tagline, phone,
                 //   mobile, email, website, address, photo, logo, signature,
                 //   banner, disclaimer }  (image-role'e trzymają URL)
  socials,       // [{ id, type, url, icon?, label? }]  type: linkedin|twitter|
                 //   github|instagram|facebook|youtube|custom
  style,         // tokeny wyglądu (patrz niżej)
  html,          // STRING: żywy markup ciała sygnatury = ŹRÓDŁO PRAWDY layoutu
  customFields,  // [{ key, label, type:'text'|'image', value }]
  lineIcons,     // { roleKey: iconUrl } — opcjonalna ikonka wiodąca linii
  device,        // 'desktop' | 'laptop' | 'mobile'
  bg,            // 'light' | 'dark' (podgląd tła)
  editMode,      // bool — chrome edycyjne wł/wył
  showLabels, showBoxes  // overlaye na canvasie
}
```

`style` (wartości domyślne przykładowe):
```js
{
  fontFamily:"Arial, Helvetica, sans-serif",
  fontSize:13, nameSize:18,
  textColor:"#333", nameColor:"#1a1a2e", accentColor:"#6c8cff",
  mutedColor:"#777", linkColor:"#6c8cff",
  buttonRadius:6, buttonBg:"#6c8cff", buttonText:"#fff",
  imageRadius:0, rowSpacing:3,
  customTokens:[]   // {id,name,kind:'color'|'font',value}
}
```

**Kluczowa zasada:** `state.html` jest źródłem prawdy dla STRUKTURY. `profile`
to osobne wartości, które są "wstrzykiwane" do markupu po rolach. Dzięki temu
zmiana szablonu/layoutu zachowuje dane, a zmiana danych nie przebudowuje layoutu.

---

## 4. Mechanizm ról (serce aplikacji)

Każdy edytowalny klocek w markupie oznaczony jest dwoma atrybutami:
- `data-sig-block` — "to jest atomowy klocek" (zaznaczalny, przeciągalny),
- `data-sig-role="<klucz>"` — semantyczna rola (np. `name`, `email`, `photo`).

Słownik ról (ROLES) — stała lista z `{key,label,type}`:
`photo, logo, signature, name, title, company, tagline, phone, mobile, email,
website, address, banner, disclaimer`. Typy: `image | text | email | url`.

Dodatkowo:
- role ikon kontaktowych: `"<klucz>__icon"` (np. `email__icon`) — wiążą osobny
  `<img>` ikony przy linii,
- `customFields` dokładają własne role dynamicznie.

**Dwukierunkowe wiązanie profil ⇄ DOM:**
- **model → view** (`applyProfileToDom`): dla każdej roli wpisz wartość z
  `profile` do węzła z tą rolą. MUSI być **niedestrukcyjne** — pusta wartość
  w profilu NIE kasuje treści z importu. Przy wpisywaniu tekstu zachować
  pojedynczy wewnętrzny `<a>` (żeby nie zgubić href linku).
- **view → model** (`reflectDomToProfile` / `harvestProfileFromDom`): edycja
  tekstu/obrazu na canvasie aktualizuje `profile`.

---

## 5. Generatory layoutów (startery)

Funkcje czystego HTML zwracające tabele z poprawnymi `data-sig-*`. Minimum
cztery startery:
- **classic** — zdjęcie po lewej, kolumna z danymi po prawej,
- **stacked** — wszystko w pionie,
- **divider** — zdjęcie | pionowa linia akcentu | dane,
- **logo bar** — pasek z logo u góry, dane poniżej.

Pomocnicze generatory (wszystkie dokładają `data-sig-block`/`data-sig-role`):
`imgTag(role,src,w,alt)`, `textBlock(role,html,styleOverride)`,
`socialRowHtml()` (tabela z komórkami ikon/linków), `buttonHtml(role,label,href)`
(tabela z `data-sig-button`), `dividerHtml()`, `spacerHtml(h)`,
`infoStack()` (imię + "tytuł · firma" + tagline + linie kontaktowe + social).

Styl bazowy tekstu składany ze `state.style`:
`font-family:…;font-size:…px;color:…;line-height:1.45;`.

---

## 6. UI / layout aplikacji

Ciemny motyw (zmienne CSS w `:root`). Trzy kolumny + nagłówek:

**Header:** tytuł, Undo/Redo, Import, Save/Export, Reset, toggle panelu HTML,
"Copy signature".

**Sidebar (lewy, resizowalny)** — 4 zakładki:
- **Content** — pola profilu (connectable), licznik połączeń `n/n`, filtr
  (all/linked/unlinked), sekcja social links, "+ Custom field".
- **Add** — inspektor zaznaczonego elementu + rozwijana paleta klocków do
  przeciągnięcia (Text, Heading, Image/GIF, Button, Social row, Divider,
  Spacer, Table, 2 Columns, Raw HTML).
- **Style** — typografia (font, rozmiary), kolory (z próbnikiem + **EyeDropper**),
  przyciski, obrazy/odstępy, "custom tokens" (dodatkowe kolory/fonty).
- **Templates** — startery + własne szablony zapisane w `localStorage`
  (przełączenie szablonu zachowuje profil).

**Center (canvas/stage):**
- toolbar: wybór urządzenia (🖥/💻/📱), toggle tła light/dark, toggle edit
  chrome, overlaye "Labels"/"Boxes", Fit, zoom −/%/+, "Normalize", wskaźnik
  zdrowia ("✓ email-safe" / "⚠ …").
- scena z siatką, pan (środkowy przycisk myszy), zoom (Ctrl/Cmd + scroll),
  "fit to view".
- "paper" (biała kartka) renderuje `state.html`; w trybie mobile kartka o
  naturalnej szerokości jest **skalowana transformem** do ekranu telefonu.
- chrome edycyjne per-klocek: etykieta roli, mini-toolbar (↑ ↓ duplikuj
  usuń), uchwyty resize dla img/button/table, outline przy hover/zaznaczeniu.
  **Chrome oznaczać `data-chrome="1"`** i usuwać przy każdej synchronizacji/
  eksporcie.

**Right (panel Raw HTML, zwijany):** textarea z **dwukierunkową synchronizacją**
na żywo + podświetlanie składni (nakładka `<pre>` pod `<textarea>` o
przezroczystym tekście) + przycisk "Format" (lekki pretty-print).

**Modale:** Import, Connection Review, Normalize (wybór trybu), Reset,
Save/Export.

---

## 7. Kluczowe przepływy (zachowania do odtworzenia)

### Render
`renderCanvas()`: wstaw `state.html` do kartki → `applyProfileToDom` →
`applyStyleToDom` → `applyLineIcons` → `decorateBlocks` (dodaj chrome) →
zsynchronizuj panel Raw → health → licznik → mobile scaling. Po renderze
chować stare overlaye hover.

### Synchronizacja view→model
`syncHtmlFromPaper()`: sklonuj kartkę, zdejmij chrome (`stripEditorChrome`),
zapisz `innerHTML` do `state.html`.

### Edycja inline
Dblclick na tekstowym klocku → `contenteditable=true`; na blur → zdejmij,
zcommituj (snapshot + sync + odbij do profilu).

### Operacje na klockach
Move up/down (zamiana z sąsiadem), duplicate (klon bez chrome), delete. Po
każdej zmianie struktury: `snapshot()` → `syncHtmlFromPaper()` → `renderCanvas()`
→ `persist()`.

### Drag & drop
- paleta → canvas: oblicz najbliższy punkt wstawienia (dystans do środków
  klocków), pokaż "drop-line", wstaw nowy klocek z `newBlockHtml(kind)`.
- reorder klocków wewnątrz canvasu (również w kolumnach tabel).
- przeciągnięcie elementu z canvasu na pole profilu = połączenie roli.

### Łączenie ról (3 drogi)
1. "connect mode": klik "link" przy polu → banner → klik elementu na canvasie.
2. drag elementu na pole.
3. dropdown roli w inspektorze.
`connectRoleToBlock(key, block)`: wybierz najciaśniejszy klocek
(`tightestBlock`), zdejmij rolę z poprzedniego posiadacza, ustaw na nowym,
zassaj istniejącą wartość do profilu jeśli pusta.

### Import (najtrudniejsza część)
1. `extractBody` + `sanitizeImported` (usuń `script/style/meta/head`, odwiń
   `body`), bez zmiany layoutu.
2. `markBlocks(root)` — oznacz **atomowe liście tekstu**: węzeł jest liściem
   tylko gdy trzyma tekst bezpośrednio i NIE zawiera innego elementu z własnym
   widocznym tekstem (duże kontenery jak `<td>` z wieloma `<div>` NIE są
   liśćmi). `img` i `a` zawsze atomowe. Inline (`span/b/strong/em`) nie są
   osobnym liściem, jeśli siedzą w blokowym liściu.
3. `recognizeRoles(root)` — heurystyki:
   - obrazy: social (po href/alt), "signature" (src/alt zawiera "sign"),
     pierwszy pozostały = `photo`, ostatni szeroki = `banner`/`logo`.
   - linki: `mailto:`→email, `tel:`→phone, inne http→website; social po
     domenie (`SOCIAL_HOSTS` regex).
   - tekst: `name` (bold/heading z pełnym imieniem — obsłuż imię rozbite na
     dwa `<span>`), potem do 2 linii opisowych → `title`, `company`; `phone`
     (regex cyfr), `email` (regex), `website` (www/http), `address` (przecinki
     / słowa jak "street","ave","poland").
4. `blankProfile()` — **wyzeruj profil** przed harvestem, by defaulty nie
   wyciekły do importu.
5. opcjonalnie `extractStyleFromDom` — wyłów dominujący kolor/font do Style.
6. `harvestProfileFromDom` — wyciągnij wartości z otagowanego DOM do profilu.
7. opcjonalnie otwórz Connection Review.

### Connection Review (modal)
Dwie kolumny: lista pól (connect/re-connect/disconnect, hover podświetla
element) i podgląd designu (hover elementu podświetla pole + pokazuje etykietę;
drag elementu na pole = połączenie; "Show all labels" rysuje etykiety na
wszystkich rozpoznanych elementach). Odświeżanie review **bez utraty pozycji
scrolla**.

### Normalize (dwa tryby)
- **tag-only** (zalecany): zachowaj dokładny design, tylko wyczyść i uzupełnij
  `data-sig-*` (re-mark + re-recognize, braki → `role="block"`).
- **full rebuild**: zbuduj czysty starter z rozpoznanych wartości (zmienia
  wygląd — za potwierdzeniem).

### Eksport
`cleanExportHtml()`: sklonuj kartkę, zdejmij chrome, **usuń wszystkie atrybuty
`data-sig*`, `draggable`, `contenteditable`** → `innerHTML`. Copy do schowka
jako `text/html` + `text/plain` (fallback: zaznaczenie + `execCommand('copy')`).
Eksport JSON granularny: full / style / profile / template. Eksport `.html`
z opakowaniem `<!DOCTYPE html>…`.

### Reset
Miękki (czyść autosave, zachowaj szablony) albo twardy (wyczyść wszystko z
`localStorage`), z potwierdzeniem.

### Undo/Redo
Stos snapshotów (JSON `profile/socials/style/html/customFields/lineIcons`),
limit ~60, `history.lock` by nie nagrywać podczas restore. Ctrl+Z / Ctrl+Y
(oraz Ctrl+Shift+Z). Delete/Backspace usuwa zaznaczony klocek (poza polami
input).

### Health check
Licz obrazy `data:` i puste `href` → pokaż ostrzeżenie; inaczej "✓ email-safe".

---

## 8. Health-of-architecture / zasady jakości

- **Idempotencja:** `applyLineIcons`, dekoracja i strip chrome muszą dać się
  wołać wielokrotnie bez narastania śmieci (najpierw usuń stare, potem dodaj).
- **Nigdy nie zostawiać chrome w `state.html`** ani w eksporcie.
- **Nie kasować treści importu** przez puste wartości profilu.
- `tightestBlock(node)` — zawsze celować w najmniejszy sensowny klocek przy
  klikaniu/łączeniu.
- `esc()` na każdym wstawianiu wartości użytkownika do HTML.
- `normUrl()` / `smartHref()` — automatyczne `https://`, `mailto:`, `tel:`.

---

## 9. Sugerowana kolejność budowy (dla agenta)

1. Szkielet HTML + CSS (header, sidebar, canvas, raw panel, modal, toast).
2. `state`, `deepClone`, persistencja + autosave, toast, helpery
   (`$`, `$$`, `el`, `esc`, `uid`, `normUrl`, `debounce`).
3. Generatory (`imgTag`, `textBlock`, `socialRowHtml`, `buttonHtml`,
   `dividerHtml`, `spacerHtml`, `infoStack`, `STARTERS`).
4. Render + dekoracja klocków + zaznaczanie + inline edit.
5. Sidebar: pola profilu, social, Style menu, startery, szablony.
6. Inspektor zaznaczonego elementu (image/button/spacer/text + rich-text).
7. Drag & drop (paleta + reorder) i uchwyty resize.
8. Panel Raw HTML (dwukierunkowy + highlight + format).
9. Łączenie ról (3 drogi) + overlay hover.
10. Import + recognizer + `blankProfile` + harvest.
11. Connection Review modal.
12. Normalize (oba tryby).
13. Export (HTML + JSON) + copy + Reset.
14. Pan/zoom/fit, resizery paneli, skróty klawiszowe, health check.

---

## 10. Pułapki (lekcje z iteracji projektu)

- Import "wielkich kontenerów" jako jednego klocka — rozwiązane przez
  wykrywanie **tight leaves**.
- Wyciek domyślnego profilu do importu — rozwiązane przez `blankProfile`
  przed harvestem.
- Imię rozbite na dwa `<span>` (np. "Dariusz" + "Łacheta") — recognizer musi
  wspiąć się do bloku z pełnym tekstem imienia i nie oznaczyć nazwiska jako
  tytułu.
- Mobile overflow — kartka renderowana w naturalnej szerokości i **skalowana
  transformem** do ekranu telefonu, wysokość ramki liczona po `requestAnimationFrame`.
- Dark-bg podgląd — neutralizować białe tła tabel/td `!important`, by nic nie
  zostało białe.
- Stałe overlaye hover po re-renderze / po pan/zoom — zawsze chować.
- Normalize tag-only vs full — nie rebuildować layoutu bez wyraźnej zgody.
