# 🎫 System Zgłoszeń (Ticketing System) - Backend

Witajcie w repozytorium backendu naszego systemu zgłoszeń! Architektura opiera się na mikroserwisach, komunikacji asynchronicznej oraz kontenerach. Całość jest zorkiestrowana i zarządzana za pomocą narzędzia Docker Compose.

Poniżej znajdziecie instrukcję, jak krok po kroku uruchomić ten projekt na swoich maszynach.

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

* 🗄️ **`mariadb` (Baza Danych)**
    * Główne źródło prawdy dla naszego systemu, przechowujące użytkowników i tickety (wspierane przez Entity Framework Core).
    * Dla wygody deweloperskiej zmapowaliśmy ją na port lokalny **3307** (dzięki temu nie będziecie mieć konfliktów, jeśli macie już lokalnie zainstalowanego MySQL/MariaDB na domyślnym porcie 3306).
    * Dane nie znikają po wyłączeniu kontenera dzięki dedykowanemu wolumenowi `mariadb_data`.
* 🐇 **`rabbitmq` (Broker Wiadomości)**
    * Serce naszej komunikacji asynchronicznej. Zapobiega blokowaniu głównego API podczas ciężkich operacji. API wrzuca tu zdarzenia (np. "TicketCreatedEvent"), a Worker Service je odbiera.
    * Panel zarządzania (Management UI) jest dostępny w przeglądarce pod adresem `http://localhost:15672` (domyślny login i hasło: `guest` / `guest`).
* 🚀 **`main-api` (Główne API REST - .NET 8)**
    * Obsługuje całą logikę biznesową HTTP (logowanie JWT, 2FA, CRUD ticketów oraz zarządzanie użytkownikami).
    * Dostępne lokalnie na porcie **5000**.
* ⚙️ **`worker-service` (Usługa Tła - .NET 8)**
    * Pracuje cicho w tle (port **5001**).
    * Nasłuchuje kolejki RabbitMQ i przy użyciu WebSockets (SignalR) przesyła powiadomienia na żywo (Real-Time) bezpośrednio do przeglądarek zalogowanych Administratorów, omijając konieczność odświeżania strony.

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

Jeżeli baza danych była pusta podczas uruchamiania (co nastąpi przy Waszym pierwszym odpaleniu Dockera), system wygeneruje automatycznie dwa konta.

* **Admin:** `admin@test.com` | Hasło: `Admin123!`
* **Pracownik:** `user@test.com` | Hasło: `User123!`

**Ważne (2FA dla Admina):** Ponieważ na koncie administratora wymuszone jest logowanie dwuetapowe, podczas pierwszego startu w logach kontenera (`docker compose logs main-api`) znajdziecie unikalny **Admin 2FA Secret**. Użyjcie go, by wygenerować kod w aplikacji Authenticator, lub zasymulujcie ten proces podczas testowania endpointu `/api/auth/verify-2fa`.
