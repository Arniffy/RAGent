## 💡 Zastosowania projektu 💡
Większość botów opartych na LLM gubi wątek w długich dyskusjach lub nie potrafi odnieść się do faktów sprzed tygodnia. Ten system rozwiązuje te problemy, oferując:

*   **Wirtualne Panele Eksperckie**: Możesz stworzyć debatę, w której uczestniczą agenci o różnych rolach (np. Architekt IT, Specjalista ds. Bezpieczeństwa), wspólnie analizując Twój problem (np. planowaną migrację bazy danych).
*   **Dynamiczna Baza Wiedzy Projektowej**: Dzięki pamięci epizodycznej RAG, bot staje się "żywym" archiwum projektu. Pamięta każdą kluczową decyzję i fakt z poprzednich sesji, eliminując potrzebę przeszukiwania tysięcy wiadomości.
*   **Weryfikacja Hipotez i Kontrargumentacja**: System pozwala na automatyczne generowanie kontrargumentów przez agentów, co pomaga wykryć luki w planach biznesowych lub technicznych jeszcze przed ich wdrożeniem .
*   **Separacja Wiedzy (Multi-Tenancy)**: Możesz prowadzić wiele niezależnych projektów jednocześnie. Bot gwarantuje, że dane z "Projektu A" nigdy nie wyciekną do odpowiedzi w "Projekcie B" .

---

##  Główne Cechy
*   **Architektura Multi-Agent**: System pozwala na interakcję z różnymi rolami agentów w ramach jednej debaty.
*   **Pamięć Epizodyczna (RAG)**: Wykorzystanie bazy wektorowej do przechowywania i przeszukiwania kontekstu historycznego wszystkich wypowiedzi.
*   **Mechanizm Multi-Tenancy**: Pełna separacja danych dzięki filtrowaniu metadanych (Metadata Filtering).
*   **Asynchroniczność**: Budowa oparta na `asyncio`, zapewniająca płynną obsługę wielu użytkowników jednocześnie.

## Architektura Systemu

### 1. Zarządzanie Stanem (postgreSQL)
Moduł odpowiada za trwałość sesji i strukturę biznesową:
*   **Tabela `debaty`**: Przechowuje UUID4 debaty, jej nazwę (np. "Migracja bazy danych") oraz znaczniki czasu.
*   **Tabela `bot_state`**: Mapuje użytkowników Telegrama do ich aktywnych sesji, zapewniając trwałość po restarcie .

### 2. Silnik Wyszukiwania Semantycznego (ChromaDB)
System pamięci długoterminowej bota:
*   **Model Wektorowy**: `text-embedding-3-small` (1536 wymiarów), zoptymalizowany pod kątem polskiej terminologii technicznej .
*   **Algorytm**: HNSW dla błyskawicznego przeszukiwania poddrzew wektorów spełniających warunek `debata_id` .
*   **Chunking**: Każda zwięzła odpowiedź agenta stanowi jeden dokument (chunk), co upraszcza strukturę danych.

### 3. Orkiestracja Agentów i Prompt Engineering
Sercem systemu jest model `gpt-4o-mini`, który przetwarza hybrydowy prompt użytkownika:
*   **Izolacja XML**: Dane w prompcie są separowane znacznikami (np. `<rag_retrieval>`, `<short_term_memory>`), co zwiększa precyzję odpowiedzi [1].
*   **Instrukcje Ról**: Agenci są instruowani, aby odpowiadać krótko (max 3 zdania) i unikać powtarzania faktów już obecnych w pamięci.

### 4. Interfejs Telegram (python-telegram-bot)
*   **/start** – inicjalizacja i instrukcja.
*   **/nowa <nazwa>** – tworzenie nowej sesji.
*   **/list** – dynamiczna lista debat z interaktywnymi przyciskami .
*   **Przepływ CallbackQuery**: Pozwala na dynamiczną zmianę widoku z listy debat na listę dostępnych agentów .

### Diagram przepływu danych:


```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant TG as Telegram API
    participant B as Bot Core (Python)
    participant SQL as PostgreSQL
    participant CH as ChromaDB (Vector DB)
    participant AI as OpenAI API (gpt-4o-mini)

    Note over U, SQL: FAZA A: Zarządzanie Sesjami i Menu
    U->>TG: Wpisanie komendy /list
    TG->>B: Przekazanie Update (Command)
    B->>SQL: SELECT id, nazwa FROM debaty
    SQL-->>B: Lista aktywnych projektów
    B->>TG: Wyświetlenie Inline Keyboards (Menu)
    U->>TG: Kliknięcie "Debata X"
    TG->>B: CallbackQuery (wybierz_X)
    B->>SQL: UPDATE debaty SET updated_at = NOW() WHERE id = X
    B-->>B: Zapisz stan w cache sesji

    Note over U, AI: FAZA B: Orkiestracja Hybrydowa RAG (Zoptymalizowana)
    U->>TG: Wywołanie agenta (+ /agent1)
    TG->>B: Przekazanie wiadomości + Kontext
    
    %% Pobieranie kontekstu z deduplikacją
    B-->>B: Pobierz historię (Short-Term) i usuń duplikaty z RAG
    B->>AI: Embedding pytania (text-embedding-3-small)
    AI-->>B: Wektor pytania
    B->>CH: Search WHERE metadata.debata_id == X
    CH-->>B: Kontekst z Pamięci Długoterminowej
    
    %% Generowanie odpowiedzi
    B-->>B: Buduj Prompt (System + RAG + Clean History)
    B->>AI: Wywołanie ChatCompletion (gpt-4o-mini)
    AI-->>B: Zwróć wygenerowaną odpowiedź Agenta

    %% KLUCZOWA ZMIANA: Odpowiedź natychmiastowa (UX)
    B->>TG: Wyślij sformatowaną odpowiedź (Markdown)
    TG-->>U: Użytkownik widzi odpowiedź (brak latencji zapisu)

    Note over B, CH: ASYNC: Pamięć Epizodyczna (Indeksowanie po wysłaniu)
    B-->>B: Walidacja jakości odpowiedzi (anty-szum)
    B->>AI: Zamień odpowiedź na wektor
    AI-->>B: Wektor odpowiedzi
    B->>CH: Zapisz do ChromaDB (metadata: debata_id, autor)
```
