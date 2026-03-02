# ozan.

https://github.com/user-attachments/assets/aae64e30-f1d3-4491-bdf4-81f91d376301


# Flowmind Middleware — Genel Sequence Diagram

## Tam Akış (Uçtan Uca)

```mermaid
sequenceDiagram
    participant User as 📱 iOS App
    participant WS as WebSocket<br/>server.js
    participant STT as Vosk STT<br/>vosk_stt.py
    participant FIL as Filler<br/>filler.js
    participant N8N as n8n<br/>Webhook
    participant TTS as Piper TTS<br/>piper_tts.py
    participant ESP as espeak-ng

    Note over User,ESP: 1️⃣ BAĞLANTI

    User->>WS: WebSocket bağlan (wss://mobilina.net/ws)
    WS-->>User: {"type":"connected"}

    Note over User,ESP: 2️⃣ KULLANICI KONUŞUYOR (STT)

    User->>WS: {"type":"start_listening"}
    WS->>STT: Python subprocess başlat

    loop Her ses chunk'ı
        User->>WS: Binary audio data (16kHz PCM)
        WS->>STT: stdin'e audio yaz
        STT-->>WS: stdout'tan partial text
        WS-->>User: {"type":"partial_transcription","text":"Ahmet'e pa..."}
    end

    User->>WS: {"type":"stop_listening"}
    STT-->>WS: Final text: "Ahmet'e para gönder"
    WS-->>User: {"type":"final_transcription","text":"Ahmet'e para gönder"}

    Note over User,ESP: 3️⃣ MESAJ İŞLEME + FİLLER

    User->>WS: {"type":"message","text":"Ahmet'e para gönder"}

    WS->>FIL: detectIntent("ahmet'e para gönder")
    FIL-->>WS: intent: TRANSFER

    WS->>N8N: POST /webhook {"text","sessionId"} (paralel, await yok)

    WS->>FIL: getACK()
    FIL-->>WS: ack.json (audio + visemes)

    WS-->>User: audio_start {sampleRate: 22050}
    WS-->>User: audio_chunk {ACK audio + visemes}

    Note over WS: ⏳ sleep(ACK süresi + 400ms gap)

    WS->>FIL: getFiller("TRANSFER")
    FIL-->>WS: transfer.json (audio + visemes)
    WS-->>User: audio_chunk {TRANSFER filler + visemes}

    Note over WS: ⏳ sleep(filler süresi)

    alt n8n henüz cevap vermedi
        loop n8n gelene kadar
            Note over WS: ⏳ sleep(1500ms gap)
            WS->>FIL: getKeepalive()
            FIL-->>WS: keepalive.json
            WS-->>User: audio_chunk {keepalive + visemes}
            Note over WS: ⏳ sleep(keepalive süresi)
        end
    end

    N8N-->>WS: ✅ {"speech":"Transfer için IBAN gerekli..."}

    Note over User,ESP: 4️⃣ GERÇEK CEVAP (TTS)

    WS-->>User: {"type":"text","text":"Transfer için IBAN gerekli..."}

    loop Her cümle
        WS->>TTS: echo "cümle" | python3 piper_tts.py
        TTS->>ESP: phonemize(text)
        ESP-->>TTS: IPA phonemes + timing
        TTS-->>WS: PCM audio + phoneme metadata
        Note over WS: phonemes → visemes (visemeMap.js)
        Note over WS: audio → chunks (500ms parçalar)
        WS-->>User: audio_chunk {PCM + visemes}
    end

    WS-->>User: audio_end

    Note over User,ESP: 5️⃣ iOS PLAYBACK

    Note over User: AVAudioPlayerNode buffer kuyruğu:<br/>ACK → gap → Intent → [keepalive] → TTS cümle 1 → TTS cümle 2...<br/><br/>CADisplayLink viseme engine:<br/>playerTime ile gerçek ses pozisyonunu takip eder
```

## Bileşen Haritası

```mermaid
graph TB
    subgraph "📱 iOS"
        A[AvatarChatViewModel] -->|WSS| B
    end

    subgraph "🌐 Nginx"
        B[mobilina.net:443<br/>SSL + Reverse Proxy]
    end

    subgraph "⚙️ VPS - Middleware"
        B -->|ws://127.0.0.1:8080| C[server.js<br/>WebSocket + Orkestrasyon]
        C --> D[filler.js<br/>Intent Detection + Filler Seçimi]
        C --> E[stt.js<br/>Vosk STT Yönetimi]
        C --> F[tts.js<br/>Piper TTS + Chunking]
        C --> G[n8n.js<br/>Webhook İletişimi]

        D --> H[(fillers/<br/>50 JSON dosya)]
        E --> I[vosk_stt.py<br/>Python subprocess]
        F --> J[piper_tts.py<br/>Python subprocess]

        I --> K[Vosk Model<br/>vosk-model-small-tr]
        J --> L[Piper Model<br/>tr_TR-fahrettin-medium]
        J --> M[espeak-ng<br/>Phoneme üretimi]
    end

    subgraph "🔗 Harici"
        G -->|HTTP POST| N[n8n<br/>76.13.144.125:5678]
    end

    style C fill:#4CAF50,color:white
    style D fill:#FF9800,color:white
    style N fill:#2196F3,color:white
```

## Dosya → Sorumluluk Tablosu

| Dosya | Sorumluluk | Bağımlılık |
|-------|-----------|------------|
| [server.js](file:///Users/ozancicek/Documents/projects/flowmind-middleware/server.js) | WebSocket, mesaj yönlendirme, filler orkestrasyon | ws, config, tüm servisler |
| [services/filler.js](file:///Users/ozancicek/Documents/projects/flowmind-middleware/services/filler.js) | Intent detection, filler seçimi, JSON yükleme | config, fillers/ |
| [services/n8n.js](file:///Users/ozancicek/Documents/projects/flowmind-middleware/services/n8n.js) | n8n webhook çağrısı | axios, config |
| [services/tts.js](file:///Users/ozancicek/Documents/projects/flowmind-middleware/services/tts.js) | Piper çağrısı, cümle streaming, chunking | piper_tts.py, visemeMap |
| [services/stt.js](file:///Users/ozancicek/Documents/projects/flowmind-middleware/services/stt.js) | Vosk subprocess yönetimi | vosk_stt.py |
| [utils/visemeMap.js](file:///Users/ozancicek/Documents/projects/flowmind-middleware/utils/visemeMap.js) | IPA phoneme → viseme ID dönüşümü | — |
| [config.js](file:///Users/ozancicek/Documents/projects/flowmind-middleware/config.js) | Tüm sabitler (port, URL, intent, timing) | — |
| [piper_tts.py](file:///Users/ozancicek/Documents/projects/flowmind-middleware/piper_tts.py) | Python: Piper model yükle, sentezle | piper-tts, espeak-ng |
| [vosk_stt.py](file:///Users/ozancicek/Documents/projects/flowmind-middleware/vosk_stt.py) | Python: Vosk model yükle, ses tanı | vosk |
| [generate_fillers.py](file:///Users/ozancicek/Documents/projects/flowmind-middleware/generate_fillers.py) | Bir kere çalış: 50 filler JSON üret | piper-tts |
