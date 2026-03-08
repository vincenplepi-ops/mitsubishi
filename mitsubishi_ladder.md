# Programma Mitsubishi – Carico/Scarico Lamiera con 24 Ventose + HMI

Documento operativo per impostare un programma PLC (Mitsubishi GX Works / MELSEC) per una macchina di carico-scarico lamiera con 24 ventose, completo di logica sequenziale, gestione vuoto e struttura HMI.

---

## 1) Obiettivo impianto

La macchina deve:

1. Prelevare una lamiera dalla stazione di carico tramite 24 ventose.
2. Verificare il vuoto globale e (opzionalmente) per gruppi.
3. Trasferire la lamiera verso stazione di scarico.
4. Rilasciare la lamiera in sicurezza.
5. Gestire cicli AUTO/MANUALE da HMI con allarmi e diagnostica.

---

## 2) Architettura consigliata

Per semplificare diagnosi e manutenzione, dividi il software in blocchi:

- **Blocco A – Sicurezze e permissivi** (E-Stop, porte, aria, reset allarmi)
- **Blocco B – Comandi operatore** (AUTO, MAN, START, STOP, RESET)
- **Blocco C – Sequenza automatica a passi** (`STEP_0...STEP_N`)
- **Blocco D – Ventose/Vuoto** (comando valvole, consenso presa, verifica sensori)
- **Blocco E – Assi/attuatori** (salita-discesa, traslazioni, pinze se presenti)
- **Blocco F – Allarmi e storico**
- **Blocco G – Interfaccia HMI** (stati, comandi, ricette, manutenzione)

---

## 3) Mappa I/O esempio (adattabile)

> Nota: indirizzi indicativi. Adegua ai moduli reali.

### Ingressi digitali

- `X0` E-Stop OK (catena sicurezza chiusa)
- `X1` Porta protezione chiusa
- `X2` Pressione aria OK
- `X3` Vuoto principale OK (pressostato collettore)
- `X4` Lamiera presente in carico
- `X5` Posizione asse Z alto
- `X6` Posizione asse Z basso (presa)
- `X7` Posizione asse X su carico
- `X10` Posizione asse X su scarico
- `X11` START ciclo
- `X12` STOP ciclo
- `X13` RESET allarmi

### Ingressi analogici / digitali gruppi vuoto (opzionale ma consigliato)

Per diagnosi delle 24 ventose, usare 6 gruppi da 4 ventose:

- `X20..X25` Vuoto gruppo 1..6 OK

### Uscite digitali

- `Y0` Elettrovalvola vuoto ON (abilitazione presa)
- `Y1` Soffio rilascio ON
- `Y2` Comando asse Z GIÙ
- `Y3` Comando asse Z SU
- `Y4` Comando asse X verso CARICO
- `Y5` Comando asse X verso SCARICO
- `Y6` Lampada verde marcia
- `Y7` Lampada rossa allarme
- `Y10` Buzzer allarme

### Memorie interne

- `M0` Macchina pronta (permissivi OK)
- `M1` Auto attivo
- `M2` Ciclo in esecuzione
- `M10..M29` Step sequenza
- `M100` Allarme attivo

### Dati

- `D0` Tempo massimo attesa vuoto (ms)
- `D1` Tempo soffio rilascio (ms)
- `D2` Timeout movimento asse Z (ms)
- `D3` Timeout movimento asse X (ms)

---

## 4) Sequenza automatica (SFC semplificata)

### STEP_0 – Attesa start

Condizioni ingresso:

- Sicurezze OK, nessun allarme, AUTO attivo, posizione iniziale valida.

Transizione a STEP_10 con `START`.

### STEP_10 – Vai su posizione carico

Azioni:

- Comanda asse X su CARICO (`Y4`).

Transizione:

- Quando `X7=1` (in posizione) entro `D3`, passa a STEP_20.
- Se timeout -> allarme movimento X.

### STEP_20 – Discesa testa presa

Azioni:

- Comanda asse Z GIÙ (`Y2`).

Transizione:

- Quando `X6=1` entro `D2`, passa a STEP_30.
- Se timeout -> allarme asse Z.

### STEP_30 – Attiva ventose e verifica vuoto

Azioni:

- Attiva `Y0` (vuoto ON).
- Avvia timer attesa vuoto (`D0`).

Transizione:

- Se `X3=1` (e opzionalmente `X20..X25` tutti validi) -> STEP_40.
- Se timeout -> allarme vuoto insufficiente.

### STEP_40 – Risalita con lamiera

Azioni:

- Comanda asse Z SU (`Y3`).

Transizione:

- Quando `X5=1` entro `D2`, passa a STEP_50.
- Se timeout -> allarme asse Z.

### STEP_50 – Trasferimento su scarico

Azioni:

- Comanda asse X su SCARICO (`Y5`).

Transizione:

- Quando `X10=1` entro `D3`, passa a STEP_60.
- Se timeout -> allarme asse X.

### STEP_60 – Discesa e rilascio

Azioni:

- Z GIÙ (`Y2`) fino a posizione deposito.
- Disattiva vuoto (`Y0=0`), attiva soffio (`Y1=1`) per `D1`.

Transizione:

- Finito `D1`, `Y1=0`, passa a STEP_70.

### STEP_70 – Ritorno posizione sicura

Azioni:

- Z SU (`Y3`) fino `X5=1`.
- Ritorno X su CARICO (`Y4`) oppure posizione home.

Transizione:

- Torna a STEP_0 o ripete da STEP_10 in ciclo continuo.

---

## 5) Logica vuoto per 24 ventose

## Strategia robusta consigliata

- **Livello 1 (obbligatorio):** pressostato vuoto principale (`X3`).
- **Livello 2 (consigliato):** 6 pressostati di gruppo (`X20..X25`).
- **Livello 3 (premium):** monitor singola ventosa (se hardware disponibile).

### Criterio di presa valido

- Modalità standard: `X3=1` e almeno **N gruppi su 6** validi (es. 5/6).
- Parametro N impostabile da HMI (ricette per formati lamiera diversi).

Questo evita scarti per micro-perdite locali non critiche.

---

## 6) HMI – Pagine minime

## Pagina 1: Sinottico produzione

- Stato macchina: PRONTA / CICLO / ALLARME / STOP.
- Step attuale (`STEP_XX`).
- Stato assi (X carico/scarico, Z alto/basso).
- Stato vuoto principale e gruppi.
- Pulsanti: START, STOP, RESET.

## Pagina 2: Manuale/Setup

- Comandi jog protetti da livello utente:
  - X->Carico, X->Scarico, Z su/giù
  - Vuoto ON/OFF, Soffio ON/OFF
- Interlock manuale: movimenti bloccati se sicurezze non OK.

## Pagina 3: Ricette

- `Tempo attesa vuoto (D0)`
- `Tempo soffio (D1)`
- `Timeout asse Z (D2)`
- `Timeout asse X (D3)`
- `Soglia gruppi vuoto minimi validi`

## Pagina 4: Allarmi

- Lista allarmi attivi/storico con timestamp.
- Istruzioni operatore (causa probabile + azione).

---

## 7) Tabella allarmi base

- `AL01` Sicurezza aperta (E-Stop/porta)
- `AL02` Mancanza aria
- `AL03` Timeout asse X
- `AL04` Timeout asse Z
- `AL05` Vuoto non raggiunto
- `AL06` Perdita vuoto durante trasferimento
- `AL07` Lamiera non presente in carico

Regola: in allarme, fermare sequenza, spegnere movimenti, mantenere o rilasciare vuoto secondo analisi rischio macchina.

---

## 8) Esempio logico (pseudo ladder testuale)

### Rete 1 – Macchina pronta

- `M0 = X0 AND X1 AND X2 AND (NOT M100)`

### Rete 2 – Latch AUTO ciclo

- `M2` (ciclo attivo) si setta con `START` se `M0` e `M1`.
- `M2` si resetta con `STOP` o `ALLARME`.

### Rete 3 – Validazione presa vuoto

- `M50 = X3 AND (GruppiVuotoValidi >= SogliaRicetta)`

### Rete 4 – Allarme perdita vuoto in trasporto

- Se step trasferimento attivo e `M50` cade per > tempo filtro -> `M100`.

---

## 9) Pseudo codice Structured Text (sequenza semplificata)

```pascal
CASE Step OF
  0: (* attesa start *)
    IF Auto AND Ready AND StartPB THEN
      Step := 10;
    END_IF;

  10: (* X a carico *)
    CmdXLoad := TRUE;
    IF X_LoadPos THEN
      CmdXLoad := FALSE;
      Step := 20;
    ELSIF TmX.Q THEN AlarmXTimeout := TRUE;
    END_IF;

  20: (* Z giu *)
    CmdZDown := TRUE;
    IF Z_DownPos THEN
      CmdZDown := FALSE;
      Step := 30;
    ELSIF TmZ.Q THEN AlarmZTimeout := TRUE;
    END_IF;

  30: (* presa vuoto *)
    VacOn := TRUE;
    IF VacuumOK THEN
      Step := 40;
    ELSIF TmVac.Q THEN AlarmVacuum := TRUE;
    END_IF;

  40: (* Z su *)
    CmdZUp := TRUE;
    IF Z_UpPos THEN
      CmdZUp := FALSE;
      Step := 50;
    END_IF;

  50: (* X a scarico *)
    CmdXUnload := TRUE;
    IF X_UnloadPos THEN
      CmdXUnload := FALSE;
      Step := 60;
    END_IF;

  60: (* rilascio *)
    CmdZDown := TRUE;
    IF Z_DownPos THEN
      CmdZDown := FALSE;
      VacOn := FALSE;
      BlowOn := TRUE;
      IF TmBlow.Q THEN
        BlowOn := FALSE;
        Step := 70;
      END_IF;
    END_IF;

  70: (* ritorno *)
    CmdZUp := TRUE;
    IF Z_UpPos THEN
      CmdZUp := FALSE;
      Step := 0;
    END_IF;
END_CASE;
```

---

## 10) Checklist avviamento

- Verifica I/O reale vs schema elettrico.
- Test manuale singoli attuatori da HMI (uno per volta).
- Taratura soglie vuoto e tempi ricetta per ogni formato lamiera.
- Test allarmi (forzare mancanza vuoto, timeout assi, ecc.).
- Test ciclo automatico a velocità ridotta.
- Validazione sicurezza con responsabile CE/sicurezza macchina.

---

## 11) Nota pratica

Se vuoi, nel prossimo step posso trasformare questa specifica in:

1. **Tabella I/O completa pronta per GX Works** (CSV),
2. **Template ladder con step M10..M70**,
3. **Lista variabili HMI GOT** (tag + allarmi + popup istruzioni).
