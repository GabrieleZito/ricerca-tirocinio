# Pipeline STT–LLM–TTS per un avatar assistente in Unity

*Architettura, componenti open source e parti da implementare. Ricerca aggiornata a ottobre 2026.*

## 1. Sintesi

L’avatar ha già percezione (OpenCV) e comportamento (Behaviour Tree). Manca il livello conversazionale e cognitivo. La ricerca porta a tre conclusioni di progetto:

1. **STT, LLM e TTS esistono già come componenti open source utilizzabili in Unity** (whisper.unity, LLMUnity, Piper/Kokoro). La parte difficile non è chiamarli, ma orchestrarli.
2. **Memoria, ruoli, gestione dei compiti e presa di parola non esistono come componente pronto per Unity.** Esistono pattern e paper solidi (Generative Agents, MemGPT/Letta, Mem0, letteratura sulle domande di chiarimento) da cui partire, ma l’integrazione con il tuo avatar va scritta da zero.
3. **L’LLM non deve comandare il Behaviour Tree.** Deve produrre *intenzioni* strutturate (JSON) che un orchestratore deterministico valida e traduce in azioni. Così il comportamento resta prevedibile e debuggabile.

Legenda usata nel documento: **\[Pronto\]** = open source utilizzabile quasi così com’è; **\[Adattare\]** = open source ma con lavoro di integrazione; **\[Da implementare\]** = va scritto da zero.

## 2. Requisiti derivati dalla tua descrizione

- Dialogo vocale in tempo reale (STT, LLM, TTS) con interruzioni (barge-in).
- Più ruoli (assistente, aiutante, moderatore) commutabili.
- Memoria tra turni e tra sessioni.
- Gestione dei compiti: capire la richiesta, riconoscere i dati mancanti, **chiedere** all’utente prendendo la parola.
- Integrazione con rotazione verso il parlante (OpenCV) e Behaviour Tree esistenti.

## 3. Architettura proposta

```
Microfono → VAD → Turn manager → STT ─┐
                                       ▼
 Telecamera → OpenCV (volti) ──►  DIALOGUE ORCHESTRATOR  ◄──► Behaviour Tree
                                  │  Role manager     (blackboard condivisa)
                                  │  Memory manager
                                  │  Task manager (slot filling)
                                  │  Floor manager (presa di parola)
                                  │  LLM client (streaming + JSON/tool)
                                  ▼
                    TTS (per frase) → AudioSource → Lip sync
```

L’orchestratore è il cuore del lavoro ed è **\[Da implementare\]**. È una macchina a stati con stati come Idle, Listening, Thinking, Speaking, AskingQuestion, WaitingForAnswer, e comunica col Behaviour Tree tramite la blackboard (per esempio `speaker_id`, `avatar_state`, `intent`, `gesture_request`).

### Scelta chiave: tutto in Unity o processo esterno?

| Opzione | Pro | Contro |
| --- | --- | --- |
| **A. In-process in Unity** (whisper.unity + LLMUnity + Piper) | Un solo eseguibile, offline, privacy, nessuna rete | Competizione per la GPU con il rendering, più difficile usare librerie Python |
| **B. Sidecar Python via WebSocket** (per esempio Pipecat o LiveKit Agents) | Ecosistema voice-agent maturo con VAD, turn detection, barge-in e plugin già fatti | Due processi da gestire, protocollo audio/eventi da definire |

Pipecat e LiveKit Agents sono framework open source per agenti vocali che integrano Silero VAD e rilevamento semantico di fine turno. Non ho trovato in questa ricerca un client Unity ufficiale: l’integrazione passerebbe da un trasporto WebSocket e andrebbe verificata.

**Consiglio:** parti con A per il prototipo (meno parti mobili) e definisci fin da subito interfacce C# astratte (`ISttService`, `ILlmService`, `ITtsService`, `IMemoryStore`). Se la GPU non regge o servono modelli più grandi, sposti i servizi dietro a un server locale senza toccare l’orchestratore.

## 4. Ingresso audio: VAD e STT

**VAD (Voice Activity Detection).** Silero VAD è un modello open source leggero che gira su CPU ed è il VAD più usato nelle pipeline vocali. Il VAD è lo strato più basso della rilevazione dei turni. Esistono anche porte ONNX utilizzabili con Unity Sentis. **\[Adattare\]**

**STT.**

- **whisper.unity** (Macoron): binding Unity per whisper.cpp, offline, multilingue (circa 60 lingue), licenza MIT, accelerazione GPU via Vulkan (Windows/Linux) e Metal (macOS/iOS). Legge flussi dal microfono. **\[Pronto\]**
- Attenzione: le librerie precompilate sono legate a una versione precisa di whisper.cpp (indicata come v1.7.5 nel README) e il repository risulta con pochi aggiornamenti recenti e diverse issue aperte. Valuta il rischio di manutenzione.
- **Whisper Tiny per Unity Sentis** (pubblicato da Unity su Hugging Face): alternativa 100% dentro Sentis, ma versione tiny e con requisiti di versione di Sentis. **\[Adattare\]**
- Per l’italiano, i modelli molto piccoli sono tipicamente meno accurati: prevedi di provare almeno *base* o *small* e di misurare WER e latenza sulla tua macchina target (dato da misurare, non assumere).

**Da implementare:** un `AudioCapture` che bufferizza il parlato tra inizio e fine turno decisi dal turn manager (sezione 5) e invia a Whisper solo quel segmento, più un eventuale filtro per non trascrivere la voce dell’avatar stesso (vedi echo, sezione 5).

## 5. Turn-taking e barge-in

È la parte che più determina la naturalezza percepita. Le fonti di settore descrivono uno stack a livelli: VAD (c’è parlato?), endpointing (quanto silenzio basta?) e rilevamento semantico (la frase è davvero finita?). Una pausa naturale e una fine turno differiscono di pochi centinaia di millisecondi e sono ambigue anche per gli umani.

- **VAD + silenzio:** Silero VAD con soglia di silenzio configurabile (punto di partenza ragionevole, da tarare). **\[Adattare\]**
- **Fine turno semantica:** Pipecat Smart Turn e il turn detector di LiveKit sono modelli open source che guardano il contenuto oltre al silenzio. Per usarli in Unity servono il sidecar (opzione B) o una porta ONNX. **\[Adattare\]**
- **Barge-in:** se l’utente parla mentre l’avatar parla, bisogna fermare il TTS entro una frazione di secondo. Le implementazioni di riferimento chiedono lo stop quasi immediato. **\[Da implementare\]** come transizione di stato Speaking → Listening con svuotamento della coda audio e annullamento della generazione LLM in corso.
- **Eco:** se l’avatar parla con altoparlanti e microfono aperto, rischia di sentire sé stesso. Soluzioni: cuffie, cancellazione d’eco di sistema, oppure mute del VAD durante il TTS con soglia più alta per il barge-in. **\[Da implementare\]** e da testare nel tuo setup.

## 6. Il livello LLM

**Runtime.**

- **LLMUnity** (undreamai): pacchetto Unity basato su llama.cpp, modelli GGUF, esecuzione locale su PC, mobile e VR, licenza permissiva (Apache-2.0 nel README attuale; verifica la licenza della versione che installi), versione 3.x del 2026. Include `LLMCharacter`/`LLMAgent`, salvataggio della chat, **RAG** con ricerca semantica (usearch) e **grammatiche GBNF** per forzare output strutturato e function calling. **\[Pronto\]**
- **Server locale compatibile OpenAI** (llama.cpp server, Ollama): permette di cambiare modello senza ricompilare e di usare la stessa interfaccia per un LLM cloud. Il client HTTP in C# è **\[Da implementare\]** ma semplice.
- Le licenze dei modelli sono indipendenti da quella del pacchetto: controllale prima di distribuire.

**Uscita strutturata.** Ogni risposta dell’LLM dovrebbe essere un oggetto, non solo testo, per esempio:

```json
{
  "speech": "Per quando vuoi la prenotazione?",
  "intent": "ask_slot",
  "task_id": "t42",
  "slot": "data",
  "emotion": "neutral",
  "gesture": "none"
}
```

Con modelli locali piccoli il function calling nativo è meno affidabile: usa una grammatica GBNF (supportata da LLMUnity) o schema JSON vincolato, e validazione lato codice con un retry.

**Streaming.** Richiedi la risposta a token e taglia a fine frase per passare subito al TTS (vedi sezione 11). **\[Da implementare\]**, parsing incrementale del JSON compreso: un trucco pratico è far generare prima i campi di controllo e poi `speech`, oppure usare due chiamate (decisione, poi testo).

## 7. Ruoli (assistente, aiutante, moderatore)

Non esiste un componente standard. Il pattern consigliato è un **RoleProfile** (ScriptableObject) composto da: system prompt, tono e lingua, tool consentiti, livello di proattività, politica di memoria, voce TTS e set di gesti del Behaviour Tree. **\[Da implementare\]**

- *Assistente:* reattivo, risponde alle richieste, proattività bassa.
- *Aiutante:* esegue compiti, usa il Task manager, fa domande sui dati mancanti.
- *Moderatore:* proattività alta, regola turni e tempi, ha policy esplicite (vedi sezione 10).

Il cambio ruolo avviene per comando esplicito o per contesto (per esempio numero di persone inquadrate), ma sempre tramite l’orchestratore, che ricompone il prompt. Evita che sia l’LLM a decidere liberamente il proprio ruolo.

## 8. Memoria

La letteratura converge su una memoria a più livelli, non su un unico grande contesto.

| Livello | Contenuto | Implementazione |
| --- | --- | --- |
| Lavoro | Ultimi N turni più riassunto progressivo | Storico di LLMUnity più riassunto **\[Da implementare\]** |
| Episodica | Eventi e scambi passati con timestamp | Archivio + embedding **\[Adattare\]** |
| Semantica / profilo | Fatti stabili sull’utente (nome, preferenze) | Blocchi chiave-valore modificabili **\[Da implementare\]** |
| Compiti | Stato dei task aperti | Task manager (sezione 9) |

**Riferimenti di progetto.**

- **Generative Agents** (Park et al., 2023): flusso di memoria in linguaggio naturale; il recupero combina **rilevanza** (similarità di embedding), **recenza** (decadimento esponenziale) e **importanza** (punteggio assegnato dall’LLM); la **riflessione** sintetizza periodicamente le memorie in intuizioni di livello più alto, scatenata quando l’importanza accumulata supera una soglia. È facile da riprodurre in C#.
- **MemGPT / Letta** (Packer et al., 2023): contesto come RAM e archivi esterni come disco, con memoria centrale sempre in prompt, memoria di richiamo e memoria d’archivio; l’agente si auto-modifica la memoria tramite funzioni. Letta è open source (Apache-2.0) ma è un runtime completo, non una libreria leggera.
- **Mem0** (arXiv 2504.19413): livello di memoria con API semplice (`add`, `search`) che estrae i ricordi in modo passivo dalla conversazione. Più facile da integrare, meno autonomo.
- **Nota sui benchmark:** i confronti numerici tra Mem0 e Letta su LongMemEval che circolano online provengono da blog di terze parti e vendor e non li considererei affidabili per decidere.

**Raccomandazione:** per il prototipo, memoria episodica con il RAG di LLMUnity più punteggio stile Generative Agents e un piccolo archivio locale (SQLite o file JSON), nascosta dietro `IMemoryStore`. Se più avanti serve di più, valuta Mem0 o Letta come sidecar senza cambiare l’orchestratore.

**Privacy.** Riconoscimento facciale, voce e memoria personale sono dati sensibili; in UE il GDPR si applica. Elaborare tutto in locale è un vantaggio. Prevedi comandi di consultazione e cancellazione della memoria («dimentica questo»). (Non è consulenza legale: verifica con chi di dovere.)

## 9. Gestione dei compiti e domande sui dati mancanti

È il punto in cui «fare chiamate a un LLM» non basta. La ricerca offre indicazioni chiare ma nessun componente pronto per Unity.

**Cosa dice la letteratura.**

- Gli LLM, per via del modo in cui sono addestrati, tendono a **inventare** gli argomenti mancanti di una chiamata a strumento invece di chiederli (Structured Uncertainty guided Clarification, arXiv 2511.08798); quel lavoro modella la scelta della domanda con un obiettivo di valore dell’informazione e mostra che si possono avere più successi con meno domande. Anche lavori sull’argument filling per API conversazionali (arXiv 2407.12016) concludono che serve un passo di grounding.
- Si possono rappresentare gli slot con tre stati: non riempito, riempito, in conflitto (ClarifyMT-Bench, arXiv 2512.21120).
- Chiedere troppo è un difetto quanto chiedere poco: il rischio è la sovra-richiesta su dettagli irrilevanti (stesso lavoro 2511.08798).
- Altri riferimenti: Proactive information gathering (arXiv 2507.21389), AskToAct (arXiv 2503.01940), benchmark ClarQ-LLM (arXiv 2409.06097).

**Progetto consigliato (Task manager) \[Da implementare\].**

1. **Schema del compito** dichiarato in codice o ScriptableObject: tipo di task, slot con tipo, obbligatorietà, validatore, stato (unfilled / filled / conflict).
2. **Estrazione:** l’LLM, con output vincolato, propone i valori degli slot dall’ultimo turno e dalla memoria. **Il codice valida**: un valore non verificabile resta *non riempito*, non viene dedotto.
3. **Selezione della domanda:** se mancano slot obbligatori, scegli **una sola** domanda per volta (prima gli slot bloccanti o da cui dipendono gli altri), lasciando all’LLM solo la formulazione naturale.
4. **Conferma** prima di azioni irreversibili o con effetti esterni.
5. **Esecuzione** tramite tool registrati; lo stato del task va in memoria e sopravvive a interruzioni.
6. **Più compiti in parallelo** con priorità: se l’utente cambia argomento, il task resta sospeso e viene ripreso.

Macchina a stati del task: Collecting → Confirming → Executing → Done / Failed / Suspended.

### Floor manager: quando l’avatar prende la parola

Oltre alle domande sui dati mancanti, l’avatar può avere altri motivi per parlare (promemoria, timeout, interventi da moderatore). Serve una policy esplicita **\[Da implementare\]**:

- Parla solo a fine turno dell’utente o dopo una pausa sufficiente; mai sopra l’utente.
- Prima di una domanda proattiva fai un segnale non verbale col Behaviour Tree (per esempio girarsi verso la persona, piccolo gesto), poi parla.
- Dopo una domanda l’avatar entra in *WaitingForAnswer*: la risposta successiva è trattata come risposta allo slot richiesto. Questo riprende il principio delle coppie adiacenti (domanda/risposta) usato in *Who Speaks Next?* (arXiv 2412.04937), dove chi è designato dalla domanda ha l’obbligo di rispondere.
- Timeout con una sola ripetizione, poi sospensione del task.

## 10. Moderazione e più persone

Il moderatore è il ruolo più difficile perché richiede di capire **chi** parla e **a chi** si rivolge.

- Un ingresso audio solo con STT non distingue i parlanti. Un sistema di ricerca per conversazioni multi-parte (Abbo et al., arXiv 2503.15496, poi esteso in Frontiers in Robotics and AI) combina direzione di arrivo del suono, diarizzazione e riconoscimento facciale con un LLM, riportando alta accuratezza nell’attribuzione dei destinatari in setup paralleli, ma anche errori dovuti al riconoscimento audio del parlante e latenza.
- Riconoscere il destinatario è difficile: un benchmark su GPT-4o per addressee recognition (arXiv 2501.16643) ottiene risultati poco sopra il caso, anche se un lavoro più recente su riunioni (arXiv 2606.17542) trova che gli LLM prevedono bene il prossimo parlante quando hanno contesto conversazionale.
- Studi sulla moderazione con robot sociali (Interaction Studies, vol. 26) valutano politiche di controllo dei turni, per esempio indirizzare direttamente le persone, con 300 studenti.

**Cosa riusare e cosa fare:**

- Hai già OpenCV: usalo per **associare il parlato a un volto**. Un’idea economica, da sperimentare **\[Da implementare\]**, è correlare VAD e apertura della bocca (landmark facciali) per attribuire il turno a una persona. Non l’ho trovata come soluzione pronta per Unity.
- Diarizzazione audio open source (per esempio pyannote) esiste in Python: richiede il sidecar. **\[Adattare\]**
- Le policy del moderatore (tempo massimo per persona, invito a chi ha parlato poco, richiamo all’ordine del giorno) vanno scritte come **regole nel Floor manager**, con l’LLM solo per la formulazione e per capire il contenuto. **\[Da implementare\]**

## 11. TTS

| Motore | Licenza | Note |
| --- | --- | --- |
| **Piper** | Codice GPL-3.0 (fork Open Home Foundation); voci in un repository MIT con circa 35 lingue | Veloce e leggero, gira anche senza GPU, esiste **piper-unity** (Windows, macOS, Android) **\[Pronto/Adattare\]** |
| **Kokoro** (82M parametri) | Apache-2.0 | Qualità percepita più alta in vari confronti; 9 lingue incluso italiano; tooling in Unity da **\[Adattare\]** (ONNX/Sentis) |

Punti da sapere:

- Piper è GPL-3.0: puoi usarlo anche commercialmente, ma se distribuisci modifiche al suo codice devi rilasciarle sotto GPL. Verifica come lo colleghi al tuo prodotto.
- Nessuno dei due clona la voce (voci preaddestrate fisse).
- Verifica qualità e disponibilità delle voci **italiane** con un ascolto diretto prima di scegliere.
- Non ho dati comparabili e affidabili di latenza tra i due: misura sul tuo hardware.

**Streaming per frase (**\[Da implementare\]**).** Il generatore LLM emette testo; un segmentatore a fine frase invia ogni frase al TTS in una coda; l’audio parte appena la prima frase è pronta mentre le successive si generano. Serve poter annullare la coda per il barge-in.

## 12. Lip sync ed espressività

- **uLipSync** (hecomi): lip sync basato su MFCC con Job System e Burst, senza plugin nativi, calibrabile per personaggio, driver per blendshape. Funziona sull’audio, quindi va bene con qualsiasi TTS e lingua. **\[Pronto\]**
- **OVRLipSync** di Meta è indicato come non più supportato oltre una certa versione: sconsigliato per nuovi progetti.
- Le emozioni e i gesti (`emotion`, `gesture` nel JSON) li traduci tu in blendshape/animazioni tramite blackboard. **\[Da implementare\]**

## 13. Integrazione con il Behaviour Tree

La letteratura recente su LLM e Behaviour Tree si concentra soprattutto sulla **generazione** di alberi o piani da linguaggio naturale (LLM-as-BT-Planner arXiv 2409.10444, BTGenBot-2 arXiv 2602.01870, HBTP arXiv 2406.00965), con la cautela che correttezza e sicurezza della pianificazione restano incerte.

Per un avatar conversazionale è più prudente il contrario: **BT fisso, LLM che sceglie tra azioni predefinite.**

- Il BT espone nodi come `FaceSpeaker`, `Gesture(x)`, `ThinkingPose`, `Idle`.
- L’orchestratore scrive sulla blackboard (`intent`, `gesture_request`, `avatar_state`, `current_speaker`) e legge da essa (`face_detected`, `speaker_id`).
- L’LLM non vede mai il BT: vede solo un elenco di gesti/emozioni consentiti e restituisce un valore.

## 14. Tabella riassuntiva

| Componente | Soluzione | Stato |
| --- | --- | --- |
| VAD | Silero VAD | Open source, adattare (ONNX/Sentis) |
| STT | whisper.unity (whisper.cpp) o Whisper in Sentis | Open source, pronto |
| Turn manager e barge-in | Stati propri; opzionale Smart Turn / LiveKit turn detector | Da implementare |
| Runtime LLM | LLMUnity (llama.cpp) o server locale compatibile OpenAI | Open source, pronto |
| Output strutturato | GBNF di LLMUnity più validazione | Adattare |
| Role manager | RoleProfile + composizione prompt | Da implementare |
| Memoria | RAG di LLMUnity + punteggio Generative Agents; opzionale Mem0/Letta | Adattare / da implementare |
| Task manager e slot filling | Schema, validatori, selezione domanda | Da implementare |
| Floor manager / moderazione | Policy a regole + LLM per la formulazione | Da implementare |
| Attribuzione parlante | Landmark OpenCV + VAD; diarizzazione opzionale | Da implementare / adattare |
| TTS | Piper (piper-unity) o Kokoro | Open source, pronto/adattare |
| Lip sync | uLipSync | Open source, pronto |
| Integrazione BT | Blackboard con azioni predefinite | Da implementare |

## 15. Latenza

La pipeline a cascata somma i ritardi di fine turno, STT, primo token LLM e primo audio TTS. Non ho numeri affidabili per il tuo hardware, quindi **misurali con un log per fase** fin dal primo giorno. Leve principali:

- STT solo sul segmento utile, modello adeguato alla GPU disponibile.
- Streaming LLM → TTS per frase.
- Modello LLM piccolo per decisioni (intent, slot) e uno più grande solo se serve per il testo.
- Feedback visivo immediato: appena finisce il turno, il BT mostra una posa di ascolto/riflessione, così l’attesa non sembra un blocco.

## 16. Roadmap suggerita

1. **Fase 1, loop vocale base:** microfono, VAD, whisper.unity, LLMUnity, Piper o Kokoro, uLipSync. Un solo ruolo, nessuna memoria. Obiettivo: misurare latenza e qualità in italiano.
2. **Fase 2, orchestratore e barge-in:** macchina a stati, interruzioni, streaming per frase, collegamento alla blackboard del BT.
3. **Fase 3, ruoli e memoria:** RoleProfile, memoria di lavoro più episodica più profilo, comandi di cancellazione.
4. **Fase 4, compiti:** Task manager con slot, validazione, domande mirate, conferma.
5. **Fase 5, iniziativa e moderazione:** Floor manager, attribuzione parlante, policy del moderatore, test con più persone.
6. **Fase 6, valutazione:** test con utenti, tasso di domande inutili, numero di turni per completare un compito, latenza percepita.

## 17. Rischi e punti da verificare

- Manutenzione di whisper.unity (dipendenza da una versione specifica di whisper.cpp).
- Qualità di STT, LLM locale e voci TTS in **italiano**: provare prima di fissare le scelte.
- Contesa GPU tra rendering Unity e modelli: potrebbe imporre il sidecar o modelli più piccoli.
- Affidabilità del function calling nei modelli locali piccoli.
- Eco e voce dell’avatar captata dal microfono.
- Licenze (GPL di Piper, licenze dei modelli) e GDPR per volti, voce e memoria.
- Alcuni risultati citati provengono da preprint non ancora sottoposti a revisione paritaria.

## 18. Riferimenti principali

**Software**

- whisper.unity: https://github.com/Macoron/whisper.unity
- Whisper Tiny per Unity Sentis: https://huggingface.co/unity/sentis-whisper-tiny
- LLM for Unity: https://github.com/undreamai/LLMUnity
- piper-unity: https://github.com/unitycoder/piper-unity
- Piper vs Kokoro (confronto): https://www.promptquorum.com/power-local-llm/piper-vs-kokoro-tts
- uLipSync: https://github.com/hecomi/uLipSync
- Letta (ex MemGPT): https://www.letta.com
- Pipecat e LiveKit Agents (documentazione sui turni): https://livekit.com/blog/turn-detection-and-interruption-handling

**Paper**

- Park et al., Generative Agents: https://arxiv.org/abs/2304.03442
- Packer et al., MemGPT: https://arxiv.org/abs/2310.08560
- Mem0: https://arxiv.org/abs/2504.19413
- Structured Uncertainty guided Clarification for LLM Agents: https://arxiv.org/abs/2511.08798
- ClarifyMT-Bench: https://arxiv.org/abs/2512.21120
- Proactive information gathering: https://arxiv.org/abs/2507.21389
- AskToAct: https://arxiv.org/abs/2503.01940
- ClarQ-LLM: https://arxiv.org/abs/2409.06097
- LLM-based argument filling for API: https://arxiv.org/abs/2407.12016
- Who Speaks Next? Multi-party AI discussion: https://arxiv.org/abs/2412.04937
- Fast Multi-Party Open-Ended Conversation with a Social Robot: https://arxiv.org/abs/2503.15496
- Addressee recognition benchmark: https://arxiv.org/abs/2501.16643
- Addressee, turn-change and next speaker prediction in meetings: https://arxiv.org/abs/2606.17542
- LLM-as-BT-Planner: https://arxiv.org/abs/2409.10444
- BTGenBot-2: https://arxiv.org/abs/2602.01870
- HBTP: https://arxiv.org/abs/2406.00965
