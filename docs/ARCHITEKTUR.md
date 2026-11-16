# Architektur und Erweiterung

Diese Seite beschreibt den implementierten Ablauf von `main.py` und die vorhandenen, teilweise noch nicht angeschlossenen Dienste.

## Startreihenfolge

1. `VoskServiceImpl` lädt das Modell aus `models/vosk-model-small-de-0.15`.
2. `GeminiServiceImpl` erzeugt einen API-Client mit `GEMINI_API_KEY`.
3. `CommandService` liefert die Intents `dance` und `sing`. `main.py` ergänzt `smalltalk` als Gemini-Handler.
4. `IntentRouterService` lädt Tokenizer und Qwen-Gewichte aus `models/qwen2_5-0_5b_instruct` auf die CPU.
5. `SSHService` wird nur dann aufgebaut, wenn `PEPPER_SSH_HOST` gesetzt ist. Benutzername und Passwort beziehungsweise Schlüsselpfad werden aus Umgebungsvariablen gelesen. Die Aufnahme-Schleife nutzt den Dienst anschließend nicht.

Fehlende Modellverzeichnisse führen beim Start zu `FileNotFoundError`. Fehlt `GEMINI_API_KEY`, schlägt die Client-Initialisierung fehl.

## Audio und Aktivierung

`AudioProcessorService.record_chunk()` nimmt Mono-Audio als `float32` auf. `VoskServiceImpl.audio_to_pcm()` wandelt es in 16-Bit-PCM um. Ein `KaldiRecognizer` verarbeitet die Chunks fortlaufend. `PartialResult()` wird nur ausgegeben; erst ein erfolgreiches `AcceptWaveform()` mit nicht leerem `Result()` fügt Text zu `recent_texts` hinzu.

`main.py` sucht `pepper` ohne Wortgrenzen im zusammengefügten, kleingeschriebenen Text. Dieser Text kann bis zu acht fertige Fragmente enthalten. Der Audio-Puffer ist auf ungefähr zehn Sekunden begrenzt, steuert die Textsuche aber nicht. Solange Vosk keinen fertigen Text liefert, gibt es keine Aktivierung. Nach jeder Aktivierung beginnt ein neuer Erkenner.

## Intent-Auswahl und Antworten

`IntentRouterService` baut aus den registrierten Funktionen eine deutsche Systemnachricht. Das Modell soll ausschließlich ein JSON-Objekt wie `{"function": "dance"}` erzeugen. Die Ausgabe wird als JSON gelesen; falls davor oder danach Text steht, wird zusätzlich der Bereich zwischen der ersten öffnenden und letzten schließenden geschweiften Klammer versucht. Nur registrierte Funktionsnamen werden akzeptiert. `none`, ungültiges JSON oder ein unbekannter Name führen zu `None`.

| Intent | Handler | Aktuelle Wirkung |
| --- | --- | --- |
| `dance` | `CommandService.dance_action()` | Gibt „Ich tanze jetzt!“ zurück und schreibt eine Platzhalter-Meldung. |
| `sing` | `CommandService.sing_action()` | Gibt „La la laaaa!“ zurück und schreibt eine Platzhalter-Meldung. |
| `smalltalk` | `GeminiServiceImpl.generate_response()` | Sendet den gesamten erkannten Text an `gemini-2.5-flash` und gibt die API-Antwort zurück. |
| kein gültiger Intent | Direkter Gemini-Aufruf in `main.py` | Sendet denselben Text als Fallback an Gemini. |

Die Handler erhalten den vollständigen Text inklusive Aktivierungswort. Die Antwort wird von `main.py` ausgegeben, aber nicht per `SSHService.execute_talk()` an Pepper übertragen.

## Vorhandene Erweiterungspunkte

- Neue Aktionen lassen sich als `IntentFunction` mit Beschreibung und Handler registrieren. Der Funktionsname muss mit der erwarteten Qwen-Ausgabe übereinstimmen.
- `SSHService.execute_talk(text)` ruft auf dem entfernten Gerät `ALTextToSpeech.say` über `qicli` auf. `execute_wave()` steuert den rechten Arm über `ALMotion`. Beide Methoden sind nur bei bestehender SSH-Verbindung nutzbar und derzeit nicht Teil des Hauptablaufs.
- `CommandService.check_command()` enthält reguläre Ausdrücke für Singen und Tanzen, wird aber vom Qwen-basierten Routing nicht aufgerufen.
- `WhisperServiceImpl` implementiert dieselbe Transkriptionsschnittstelle wie Vosk, erwartet jedoch ein von außen übergebenes Modell und ist in `main.py` nicht konfiguriert.
- `AudioProcessorService.start_recording()` bietet eine ältere Aufnahme per gedrückter Strg-Taste. Der aktive Ablauf nutzt stattdessen kontinuierliche `record_chunk()`-Aufrufe.

## Betriebsgrenzen

Die Hauptschleife ist synchron: Aufnahme, Transkription, Qwen-Inferenz und Gemini-Anfrage laufen nacheinander. Während Intent-Auswahl oder API-Anfrage werden keine weiteren Audio-Chunks aufgenommen. Für einen dauerhaften Robotereinsatz wären eine Anbindung der Handler an Pepper, abgesicherte SSH-Zugangsdaten und Host-Schlüsselprüfung sowie eine robustere Fehlerbehandlung nötig.
