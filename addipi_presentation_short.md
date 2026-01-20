---
marp: true
---

# System zarządzania pracą drukarki 3D w laboratorium studenckim
## AddiPi - Prezentacja projektu inżynierskiego

---

# 📋 Agenda

1. Wprowadzenie i motywacja
2. Architektura systemu
3. Komponenty systemu
4. Przepływ danych
5. Bezpieczeństwo
6. Demonstracja
7. Rozwój i wnioski

---

# 🎯 Problem

**Wyzwania w laboratorium:**

- ❌ Brak systemu kolejkowania
- ❌ Ręczne przesyłanie plików
- ❌ Brak zdalnego monitoringu
- ❌ Trudności w zarządzaniu użytkownikami
- ❌ Konflikty rezerwacji

**Rozwiązanie:**  
> Zintegrowany system do zdalnego zarządzania drukarką 3D

---

# 💡 AddiPi - Rozwiązanie

**Komponenty:**
- 🌐 Aplikacja webowa (React + TypeScript)
- ⚙️ Mikrousługi backendowe (Node.js, Python)
- 🤖 Agent na Raspberry Pi
- ☁️ Azure Cloud (IoT Hub, Cosmos DB, Blob Storage)

**Funkcjonalności:**
✅ Zdalne przesyłanie G-code  
✅ Harmonogramowanie druków  
✅ Monitoring real-time  
✅ System kolejkowania  
✅ Zarządzanie użytkownikami (RBAC)  
✅ Podgląd z kamery

---

# 🏗️ Architektura systemu

```
Frontend (React/Vercel)
         ↓
   Mikrousługi:
   ├─ Auth Service (JWT)
   ├─ User Service
   ├─ Files Service (Python)
   ├─ Queue Service
   ├─ Printer Service
   └─ Video Service
         ↓
   Azure Cloud:
   ├─ Cosmos DB
   ├─ Blob Storage
   ├─ Service Bus
   ├─ IoT Hub
   └─ Event Hubs
         ↓
   Raspberry Pi Agent
         ↓
   OctoPrint → Prusa i3 MK3
```

---

# 🔐 Auth Service

**Technologie:** Node.js, TypeScript, JWT, bcrypt

**Funkcjonalności:**
- Rejestracja z weryfikacją email (token 24h)
- Login z tokenami:
  - Access Token (15 min)
  - Refresh Token (7 dni)
- Automatyczne odświeżanie sesji

**Bezpieczeństwo:**
- Hashowanie haseł (bcrypt, salt=10)
- Weryfikacja emaila (nodemailer)
- Token rotation

---

# 👥 User Service

**Endpointy użytkownika:**
- `GET /users/me` - profil
- `PATCH /users/me` - edycja
- `GET /users/me/jobs` - lista zadań
- `GET /users/me/stats` - statystyki

**Panel administratora:**
- `GET /users` - wszyscy użytkownicy
- `PATCH /users/:id/role` - zmiana roli
- `DELETE /users/:id` - usunięcie

**RBAC:** Role-Based Access Control (user/admin)

---

# 📁 Files Service

**Stack:** Python + Flask + Azure Blob Storage

**Proces uploadu:**
1. Walidacja:
   - Rozszerzenie: `.gcode`
   - Rozmiar: max 50MB
   - Zawartość (opcjonalnie)

2. Upload do Blob Storage
   - Nazwa: `timestamp_original.gcode`

3. Wiadomość do Service Bus
   - Event: `file_uploaded`
   - Metadane: userId, timestamp, scheduledAt

---

# 📋 Queue Service

**Architektura:**
```
Service Bus Queue
      ↓
  Listener
      ↓
 Cosmos DB (jobs)
      ↓
  REST API
```

**Statusy zadań:**
`scheduled` → `pending` → `printing` → `completed`

**Alternatywnie:** `failed`, `cancelled`

---

# 🖨️ Printer Service

**Harmonogramowanie (cron):**
- Co minutę sprawdza zaplanowane zadania
- Jeśli drukarka wolna → pobiera następne zadanie
- Wysyła komendę do Raspberry Pi (IoT Hub Direct Method)

**Direct Methods:**
- `startPrint` - rozpocznij druk
- `cancelPrint` - anuluj druk
- `getStatus` - pobierz status

**Telemetria:** Event Hubs listener aktualizuje statusy w czasie rzeczywistym

---

# 🤖 Raspberry Pi Agent

**Technologie:** Python + Azure IoT Device SDK + OctoPrint API

**Główne zadania:**
1. Obsługa Direct Methods z chmury
2. Pobieranie plików z Blob Storage
3. Upload do OctoPrint i start druku
4. Monitoring co 10s (progress, temperatura)
5. Wysyłanie telemetrii do Event Hubs

**Zdarzenia telemetryczne:**
- `print_started`
- `print_progress` (co 10s)
- `print_completed`
- `print_failed`

---

# 💻 Frontend

**Stack:**
- React 18.2 + TypeScript 5.2
- Vite 5.1 (bundler)
- Tailwind CSS 3.4
- Zustand (state management)
- Axios (HTTP client)

**Główne widoki:**
- Dashboard (overview + statystyki)
- Upload & Schedule
- Print Control (monitoring + kamera)
- Admin Panel (użytkownicy + zadania)
- Profil użytkownika

**Auto-refresh:** Polling co 5s

---

# 📊 Przepływ danych - Kompletny scenariusz

**1. Upload:**
User → Frontend → Files Service → Blob Storage → Service Bus

**2. Kolejkowanie:**
Service Bus → Queue Service → Cosmos DB (status: scheduled/pending)

**3. Harmonogramowanie:**
Cron → Printer Service → IoT Hub → Raspberry Pi

**4. Wykonanie:**
Pi → Blob download → OctoPrint → Event Hubs (telemetria)

**5. Monitoring:**
Event Hubs → Printer Service → Cosmos DB → Frontend (polling)

**Opóźnienie:** ~15s maksymalnie

---

# 🔒 Bezpieczeństwo

**Uwierzytelnianie:**
- Weryfikacja email (24h)
- JWT Access Token (15 min)
- JWT Refresh Token (7 dni)
- Hashowanie haseł (bcrypt)

**Autoryzacja (RBAC):**
- Middleware: `requireAuth` + `requireAdmin`
- Kontrola dostępu per endpoint
- Walidacja tokenów

**Walidacja:**
- Typ pliku, rozmiar, zawartość
- Input validation
- Sanityzacja danych

---

# 🐳 Deployment

**Backend:**
- Docker + Docker Compose
- Multi-stage builds (mniejsze obrazy)
- Health checks
- Environment variables

**Frontend:**
- Vercel (auto-deploy z GitHub)
- Edge CDN
- Automatyczne HTTPS

**Opcje produkcyjne:**
- Azure Container Instances
- Azure App Service
- Docker na VPS

---

# 📈 Metryki i monitoring

**Dashboard pokazuje:**
- Aktualnie drukuje: 1
- W kolejce: 3
- Ukończone (dziś): 12
- Ukończone (ogółem): 127
- Nieudane (24h): 2
- Użytkownicy: 45 (23 aktywnych)

**Częstotliwość:**
- Agent → Cloud: co 10s
- Frontend polling: co 5s
- Maksymalne opóźnienie: ~15s

---

# ✅ Zalety systemu

**Modularność:**
- Niezależne mikrousługi
- Łatwa rozbudowa
- Możliwość skalowania

**Niezawodność:**
- Kolejkowanie zapobiega konfliktom
- Retry mechanism
- Health checks

**Skalowalność:**
- Cosmos DB (auto-scaling)
- Możliwość wielu drukarek
- Cloud-native

**UX:**
- Responsywny interfejs
- Real-time updates
- Intuicyjna obsługa

---

# 🚀 Możliwości rozwoju

**Krótkoterminowe:**
- Notyfikacje email/push
- Historia i ulubione pliki
- Estymacja czasu druku
- Wykresy temperatury

**Średnioterminowe:**
- Multi-printer support
- Advanced scheduling (priorytety)
- WebSockets (zamiast polling)
- Zarządzanie materiałami

**Długoterminowe:**
- Machine Learning (predykcja błędów)
- Integracja z systemami uczelni (SSO)
- Mobile app (React Native)
- 3D Model Preview + slicing

---

# ⚠️ Wyzwania

**Opóźnienia:**
- Problem: Polling + monitoring = ~15s
- Rozwiązanie: WebSockets/SignalR

**Vendor lock-in:**
- Problem: Zależność od Azure
- Rozwiązanie: Abstrakcja (Repository Pattern)

**Synchronizacja:**
- Problem: Race conditions
- Rozwiązanie: ETags w Cosmos DB

**Koszty:**
- Free Tier na start
- Optymalizacja: caching, batch operations

---

# 📊 Statystyki projektu

**Infrastruktura:**
- 6 mikrousług backendowych
- 1 frontend (React)
- 1 agent (Raspberry Pi)
- 5 usług Azure

**Kod:**
- ~15,000 linii kodu
- ~180 plików źródłowych
- 9 repozytoriów Git
- ~40 endpointów REST API

**Czas realizacji:** ~4.5 miesiąca

---

# 🎓 Wartość edukacyjna

**Poznane technologie:**
- Frontend: React, TypeScript, Tailwind
- Backend: Node.js, Python, mikrousługi
- Cloud: Azure (5 usług)
- IoT: MQTT, Direct Methods, telemetria
- DevOps: Docker, CI/CD

**Koncepty:**
- Event-driven architecture
- RBAC
- JWT authentication
- Message queuing
- Real-time monitoring

---

# 🎯 Demonstracja

**Scenariusz:**
1. Login do systemu (admin@uwr.edu.pl)
2. Dashboard - widok statystyk
3. Upload pliku test.gcode
4. Schedule na 15:00
5. Monitoring - auto-refresh
6. Panel kontroli - kamera
7. Panel admin - zarządzanie

**Live Demo:** https://addipi.vercel.app

---

# 📝 Wnioski

**Cel: ✅ OSIĄGNIĘTY**

System umożliwia:
- ✅ Zdalne przesyłanie plików
- ✅ Harmonogramowanie druków
- ✅ Monitoring real-time
- ✅ Zarządzanie użytkownikami

**Największe wyzwania:**
- Synchronizacja stanów (rozwiązano przez event-driven)
- Opóźnienia sieciowe (optymalizacja polling)
- Bezpieczeństwo multi-tenant (JWT + RBAC)

**Najlepsze osiągnięcia:**
- Bezproblemowa integracja
- Stabilne działanie
- Intuicyjny UI

---

# 🔮 Perspektywy wdrożenia

**Potencjalne zastosowania:**
- Laboratoria studenckie (uniwersytety)
- Małe firmy prototypowe
- FabLab'y i makerspace'y
- Szkoły (edukacja STEM)

**Wymagania:**
- Drukarka 3D z OctoPrint
- Raspberry Pi 3B+
- Konto Azure (Free Tier)

**Koszty miesięczne:**
- Free Tier: $0
- Small scale: $10-30
- Medium scale: $50-100

---

# 📚 Dokumentacja

**Repozytoria GitHub (9):**
- AddiPi-Frontend
- AddiPi-Auth-Service
- AddiPi-User-Service
- AddiPi-Files-Service
- AddiPi-Queue-Service
- AddiPi-Printer-Service
- AddiPi-Video-Service
- AddiPi-Agent
- AddiPi-Infrastructure

**Zasoby:**
- README w każdym repo
- Kolekcje Postman
- Diagramy architektury
- Instrukcje deployment

---

# 🎓 Podsumowanie

**Projekt AddiPi to:**

✅ Kompleksowy system zarządzania drukarką 3D
✅ Nowoczesna architektura (mikrousługi + cloud)
✅ Praktyczne zastosowanie
✅ Skalowalność i bezpieczeństwo
✅ Intuicyjny interfejs

**Wartość:**
- 📚 Edukacyjna - poznanie nowoczesnych technologii
- 💼 Praktyczna - gotowe rozwiązanie
- 🚀 Rozwojowa - fundament do rozbudowy

---

# DZIĘKUJĘ ZA UWAGĘ!

**Pytania?**

**Kontakt:**
- GitHub: github.com/AddiPii
- Demo: https://addipi.vercel.app

**Oliwer Urbaniak**
**Promotor: Dr. Bartosz Brzostowski**
**Wydział Fizyki i Astronomii**
**Uniwersytet Wrocławski, 2025**

---

# Pytania techniczne

**Najczęściej zadawane:**

Q: Czy system obsługuje wiele drukarek?  
A: Obecnie jedna, architektura pozwala na rozbudowę

Q: Jakie są koszty?  
A: Free Tier wystarczy na start, potem $10-100/mies

Q: Inne drukarki niż Prusa?  
A: Tak, każda z OctoPrint

Q: Czas setup?  
A: ~2-3 godziny (Azure + Pi + deployment)

Q: Bezpieczeństwo?  
A: Wielowarstwowe (JWT, RBAC, weryfikacja email)