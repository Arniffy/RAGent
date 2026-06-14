
# Architektura systemu Debat Analitycznych w bazach wektorowych

Orkiestracja wielu agentów oraz system **RAG (Retrieval-Augmented Generation)** do prowadzenia merytorycznych debat i zarządzania wiedzą w czasie rzeczywistym.

## Główne Cechy
*   **Architektura Multi-Agent**: System pozwala na interakcję z różnymi rolami agentów w ramach jednej debaty.
*   **Pamięć Epizodyczna (RAG)**: Wykorzystanie bazy wektorowej do przechowywania i przeszukiwania kontekstu historycznego wszystkich wypowiedzi.
*   **Mechanizm Multi-Tenancy**: Pełna separacja danych pomiędzy różnymi debatami dzięki filtrowaniu metadanych (Metadata Filtering).
*   **Asynchroniczność**: Budowa oparta na `asyncio`, zapewniająca płynną obsługę wielu użytkowników jednocześnie.

## Architektura Systemu

### 1. Zarządzanie Stanem (SQLite)
Moduł odpowiada za trwałość sesji i strukturę biznesową projektów:
*   **Tabela `debaty`**: Przechowuje UUID4 debaty, jej nazwę oraz znaczniki czasu.
*   **Tabela `bot_state`**: Mapuje użytkowników Telegrama do ich aktywnych sesji.

### 2. Silnik Wyszukiwania Semantycznego (ChromaDB)
System pamięci długoterminowej bota .
*   **Model Wektorowy**: `text-embedding-3-small` (1536 wymiarów), zoptymalizowany pod kątem polskiej terminologii technicznej.
*   **Algorytm**: HNSW (Hierarchical Navigable Small World) dla błyskawicznego przeszukiwania poddrzew wektorów spełniających warunek `debata_id` .
*   **Chunking**: Każda zwięzła odpowiedź agenta stanowi jeden integralny dokument (chunk).

### 3. Orkiestracja Agentów i Prompt Engineering
Sercem systemu jest model `gpt-4o-mini`, który przetwarza hybrydowy prompt użytkownika :
*   **Hybrydowa Pamięć**: Łączy kontekst długoterminowy ([CONTEXT_LONG_TERM] z bazy RAG) z kontekstem krótkoterminowym ([CONTEXT_SHORT_TERM] – ostatnia wiadomość) 
*   **Izolacja XML**: Dane w prompcie są separowane znacznikami XML, co zwiększa precyzję odpowiedzi

### 4. Interfejs Telegram (python-telegram-bot)
Zapewnia intuicyjne sterowanie za pomocą komend i przycisków:
*   `/start` – inicjalizacja i instrukcja.
*   `/nowa <nazwa>` – tworzenie nowej sesji debaty.
*   `/list` – dynamiczna lista debat z interaktywnymi przyciskami `InlineKeyboardMarkup`.
*   **Przepływ CallbackQuery**: Pozwala na dynamiczną zmianę widoku z listy debat na listę dostępnych agentów w ramach wybranego projektu

## 🛠 Technologia
*   **Language**: Python (Asyncio) 
*   **LLM**: OpenAI GPT-4o-mini
*   **Embeddings**: text-embedding-3-small
*   **Vector DB**: ChromaDB
*   **Relational DB**: SQLite 
*   **Deployment**: Docker & Docker Compose


#Diagram przepływu danych:

USER<-->TelegramBot<-->DBs<-->LLM API
```mermaid
sequenceDiagram
    autonumber
    actor U as Użytkownik / Moderator
    participant TG as Telegram API
    participant B as Bot Core (Python)
    participant SQL as SQLite (Meta DB)
    participant CH as ChromaDB (Vector DB)
    participant AI as OpenAI API (gpt-4o-mini)

    %% Sekcja 1: Zarządzanie Debatami
    Note over U, SQL: FAZA A: Zarządzanie Sesjami i Menu
    U->>TG: Wpisanie komendy /list
    TG->>B: Przekazanie Update (Command)
    B->>SQL: SELECT id, nazwa FROM debaty
    SQL-->>B: Lista aktywnych projektów
    B->>TG: Wyświetlenie Inline Keyboards (Menu)
    U->>TG: Kliknięcie przycisku "Debata X"
    TG->>B: CallbackQuery (wybierz_X)
    B->>SQL: UPDATE debaty SET updated_at = NOW() WHERE id = X
    B-->>B: Zapisz stan: context.user_data['aktywna_debata'] = X

    %% Sekcja 2: Wywołanie Agenta i RAG
    Note over U, AI: FAZA B: Orkiestracja Hybrydowa RAG
    U->>TG: Wywołanie agenta (np. Reply do wiadomości + /agent1)
    TG->>B: Przekazanie wiadomości + Kontext konwersacji
    B-->>B: Pobierz kontekst krótkoterminowy (Wiadomość z Reply)
    
    %% Retrieval
    B->>AI: Zmień pytanie użytkownika na wektor (text-embedding-3-small)
    AI-->>B: Zwróć wektor (Embedding)
    B->>CH: Szukaj najbliższych sąsiadów WHERE metadata.debata_id == X
    CH-->>B: Zwróć pasujące dokumenty (Pamięć Długoterminowa)
    
    %% Augmentation & Generation
    B-->>B: Buduj potrójny Prompt (System + RAG + Short-Term History)
    B->>AI: Wywołanie ChatCompletion (gpt-4o-mini)
    AI-->>B: Zwróć wygenerowaną odpowiedź Agenta
    
    %% Ingest
    B->>AI: Zamień nową odpowiedź agenta na wektor
    AI-->>B: Zwróć wektor
    B->>CH: Zapisz dokument z metadata: {"debata_id": X, "autor": "Agent1"}
    
    %% Odpowiedź do użytkownika
    B->>TG: Wyślij sformatowaną odpowiedź (Markdown)
    TG-->>U: Wizualizacja odpowiedzi w bąbelku czatu
```
