# 🎫 System Zgłoszeń (Ticketing System) - Backend

Witajcie w repozytorium backendu naszego systemu zgłoszeń! Architektura opiera się na mikroserwisach, komunikacji asynchronicznej oraz kontenerach. Całość jest zorkiestrowana i zarządzana za pomocą narzędzia Docker Compose.

Poniżej znajdziecie instrukcję, jak krok po kroku uruchomić ten projekt na swoich maszynach oraz dokumentację dostępnych końcówek (endpointów) API.

---

## ⚠️ Krok 1: Konfiguracja sekretów (Plik `.env`)

Zanim wpiszecie jakąkolwiek komendę, **musicie dodać plik ze zmiennymi środowiskowymi**. Ze względów bezpieczeństwa (hasła, klucze szyfrujące) plik ten nie znajduje się i nigdy nie powinien znaleźć się w repozytorium Git.

1. Pobierzcie plik `.env` (wyślę Wam go osobno na komunikatorze).
2. Wrzućcie ten plik **bezpośrednio do głównego folderu projektu** (dokładnie tam, gdzie znajduje się plik `docker-compose.yml`).
3. Plik ten zawiera niezbędne hasła do bazy danych oraz klucz do szyfrowania tokenów JWT.

> **CRITICAL WARNING:** Nigdy nie uruchamiajcie Dockera przed dodaniem tego pliku! Jeśli to zrobicie, baza danych MariaDB zainicjuje się z pustymi hasłami i zablokuje możliwość logowania dla API. Wymusi to na Was ręczne czyszczenie wolumenów poleceniem `docker-compose down -v`.

---

## 🏗️ Krok 2: Architektura (Co znajduje się w `docker-compose.yml`?)

Nasza aplikacja to w pełni skonteneryzowane środowisko składające się z 4 niezależnych serwisów, które współpracują ze sobą w jednej, zamkniętej sieci wewnętrznej:

* **🗄️ `mariadb` (Baza Danych):** Główne źródło prawdy dla naszego systemu, przechowujące użytkowników i tickety (wspierane przez Entity Framework Core). Dla wygody deweloperskiej zmapowaliśmy ją na port lokalny **3307** (dzięki temu nie będziecie mieć konfliktów, jeśli macie już lokalnie zainstalowanego MySQL/MariaDB na domyślnym porcie 3306). Dane nie znikają po wyłączeniu kontenera dzięki dedykowanemu wolumenowi `mariadb_data`.
* **🐇 `rabbitmq` (Broker Wiadomości):** Serce naszej komunikacji asynchronicznej. Zapobiega blokowaniu głównego API podczas ciężkich operacji. API wrzuca tu zdarzenia (np. `TicketCreatedEvent`), a Worker Service je odbiera. Panel zarządzania (Management UI) jest dostępny w przeglądarce pod adresem `http://localhost:15672` (domyślny login i hasło: `guest` / `guest`).
* **🚀 `main-api` (Główne API REST - .NET 8):** Obsługuje całą logikę biznesową HTTP (logowanie JWT, 2FA, CRUD ticketów oraz zarządzanie użytkownikami). Dostępne lokalnie na porcie **5000**.
* **⚙️ `worker-service` (Usługa Tła - .NET 8):** Pracuje cicho w tle (port **5001**). Nasłuchuje kolejki RabbitMQ i przy użyciu WebSockets (SignalR) przesyła powiadomienia na żywo (Real-Time) bezpośrednio do przeglądarek zalogowanych Administratorów, omijając konieczność odświeżania strony.

---

## 🚀 Krok 3: Uruchomienie projektu

Upewnijcie się, że macie włączonego Docker Desktop, plik `.env` leży w głównym folderze i nie macie zajętych portów (5000, 5001, 3307, 5672, 15672). Otwórzcie terminal w głównym katalogu i wpiszcie:

```bash
# Budowanie obrazów na podstawie kodu źródłowego (wykonajcie przy pierwszym uruchomieniu lub pobraniu większych zmian)
docker-compose build

# Uruchomienie wszystkich kontenerów w tle
docker-compose up -d

```

Aby upewnić się, że backend wystartował prawidłowo (i aby odczytać wygenerowane dane testowe), sprawdźcie logi głównego API:

```bash
docker-compose logs -f main-api

```

---

## 🧪 Krok 4: Testowanie (Swagger i Konta Testowe)

Gdy kontenery bezbłędnie wstaną, dokumentacja całego API (Swagger) będzie dostępna pod adresem:
👉 **http://localhost:5000/swagger/index.html** *(Pamiętajcie o wpisaniu pełnej ścieżki z plikiem index.html!)*

### 🔑 Konta Testowe (Data Seeding)

Jeżeli baza danych była pusta podczas uruchamiania (co nastąpi przy Waszym pierwszym odpaleniu Dockera), system wygeneruje automatycznie dwa konta:

* **Admin:** `admin@test.com` | Hasło: `Admin123!`
* **Pracownik:** `user@test.com` | Hasło: `User123!`

**Ważne (2FA dla Admina):** Ponieważ na koncie administratora wymuszone jest logowanie dwuetapowe, podczas pierwszego startu w logach kontenera (`docker compose logs main-api`) znajdziecie unikalny **Admin 2FA Secret**. Użyjcie go, by wygenerować kod w aplikacji Authenticator, lub zasymulujcie ten proces podczas testowania endpointu `/api/Auth/verify-2fa`.

---

## 🔌 Krok 5: Dokumentacja API (Endpointy)

Poniżej znajduje się zestawienie najważniejszych endpointów wystawionych przez `main-api`. Wszystkie (poza autoryzacją) wymagają nagłówka `Authorization: Bearer {token}`.

### 🔐 Autoryzacja i Logowanie (`/api/Auth`)

| Metoda | Endpoint | Wymagane Role | Opis i Logika Biznesowa |
| --- | --- | --- | --- |
| **POST** | `/login` | Brak | Pierwszy etap logowania. Jeśli loguje się Pracownik, API od razu zwraca pełny token JWT. Jeśli loguje się Admin, API zwraca `requires2FA = true` lub `requires2FASetup = true` (z dołączonym kluczem `secretKey` do wygenerowania kodu QR na frontendzie). |
| **POST** | `/verify-2fa` | Brak | Drugi etap logowania dla Admina. Wymaga podania adresu email oraz 6-cyfrowego kodu TOTP. Po pomyślnej weryfikacji API zwraca pełnoprawny token JWT. |

### 👥 Zarządzanie Użytkownikami (`/api/Users`)

**Ważne:** Cały ten kontroler jest zablokowany tylko dla użytkowników z rolą **Admin**.

| Metoda | Endpoint | Opis i Logika Biznesowa |
| --- | --- | --- |
| **GET** | `/` | Pobiera listę wszystkich użytkowników w systemie (bez haseł). |
| **GET** | `/{id}` | Pobiera szczegóły konkretnego użytkownika na podstawie jego UUID. |
| **POST** | `/` | Tworzy nowe konto. Jeśli nowa rola to Admin, system automatycznie wygeneruje dla niego klucz TOTP (`TwoFactorSecret`), aby wymusić setup przy pierwszym logowaniu. |
| **PUT** | `/{id}` | Aktualizuje dane użytkownika (Email, Rola, Informacje Kontaktowe, opcjonalnie Hasło). Jeśli pracownik jest awansowany na Admina, API generuje dla niego klucz 2FA. |
| **DELETE** | `/{id}` | Usuwa użytkownika. Zabezpieczenia: Admin nie może usunąć samego siebie ani użytkownika, do którego są już przypisane jakiekolwiek tickety (błąd bazy danych z powodu klucza obcego). |

### 🎟️ Zarządzanie Zgłoszeniami (`/api/Tickets`)

| Metoda | Endpoint | Wymagane Role | Opis i Logika Biznesowa |
| --- | --- | --- | --- |
| **GET** | `/?includeArchived={bool}` | **Admin** | Pobiera listę zgłoszeń (domyślnie tylko aktywne, bez zarchiwizowanych, chyba że podano parametr `includeArchived=true`). W wynikach dołączone są dane twórcy (Email, ContactInfo). |
| **POST** | `/` | Zalogowany użytkownik | Tworzy nowe zgłoszenie na podstawie tokenu (identyfikuje twórcę). Zgłoszenie natychmiast trafia do bazy, a API publikuje zdarzenie `TicketCreatedEvent` na kolejkę RabbitMQ dla serwisu powiadomień. |
| **PATCH** | `/{id}/status` | **Admin** | Zmienia status ticketa. Przekazywane wartości liczbowe dla statusów to: **0** (`New`), **1** (`InProgress`), **2** (`Resolved`), **3** (`Closed`). **Automatyzacja:** Jeśli nowo ustawiony status to `Closed` (wartość liczbowo równa 3), system automatycznie oznacza zgłoszenie jako zarchiwizowane (`IsArchived = true`). |

### 📡 Real-time Powiadomienia (SignalR)

* **Endpoint:** `/notifications` (Hostowany na porcie **5001** przez kontener `worker-service`).
* **Autoryzacja:** Wymaga przekazania tokenu JWT w QueryStringu jako `?access_token={token}`.
* **Działanie:** Zalogowani Administratorzy są automatycznie dodawani do grupy `AdminsGroup`. WorkerService odbierający zdarzenie z kolejki RabbitMQ wysyła wiadomość typu `ReceiveNewTicket` do przeglądarek za pomocą WebSocketów.
