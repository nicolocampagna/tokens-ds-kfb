# FlowCaster — piano di scaling 480×272 → 800×480

Documento di lavoro. Vive nel repo (non in chat) così non si perde.
Target: portare FlowCaster da 480×272 a 800×480, con risultato
pixel-perfect utilizzabile da firmware in C.

**File di design:** Project FlowCaster — Design v 1.2
<https://www.figma.com/design/FgxDMboNXUKXzwAa0k1S4T/Project-FlowCaster---Design-v-1.2?node-id=49-297>
- file key: `FgxDMboNXUKXzwAa0k1S4T`
- nodo di riferimento: `49:297`

---

## 0. Numeri di partenza

| | 480×272 | 800×480 |
|---|---|---|
| Aspect ratio | 1,765 (≈16:9) | 1,667 (5:3) |
| Fattore orizzontale | — | 800/480 = **1,667×** |
| Fattore verticale | — | 480/272 = **1,765×** |
| Area in pixel | 130.560 | 384.000 (**×2,94**) |

Scalando uniformemente sul fattore minore: 272 × 1,667 = 453,3
→ **residuo verticale di ~27px** da redistribuire.

---

## 1. BLOCCO APERTO — dimensione fisica del pannello

Il fattore di scala corretto **non** deriva dai pixel, ma dalla densità
fisica (PPI). Assumendo il vecchio 480×272 su 4,3" ≈ 128 PPI:

| Nuovo pannello | PPI | Fattore per mantenere la dimensione fisica |
|---|---|---|
| 4,3" 800×480 | 217 | **≈ 1,69** → 1,667× è corretto |
| 5,0" 800×480 | 187 | **≈ 1,45** → 1,667× ingrandisce la UI del ~15% |
| 7,0" 800×480 | 133 | **≈ 1,04** → **non scalare**, si guadagna solo area |

> Su un 7" scalare ×1,667 è un errore netto: UI enorme e guadagno di
> area sprecato.

**Nota:** la densità *non* raddoppia. Raddoppia (×2,94) l'area in pixel;
la densità *lineare*, quella che governa leggibilità e target touch,
sale di ~1,45–1,69× a seconda del pannello.

**Da decidere prima di qualunque altra cosa. Tutto il resto dipende da qui.**

Pannello scelto: _______________  → fattore adottato: _______________

---

## 2. Correzioni al piano iniziale

Il metodo di base è giusto: **scala uniforme sul fattore minore +
redistribuzione del residuo via auto-layout**. Tre correzioni:

### a) Separare geometria e valori atomici
Sono due problemi distinti, non uno:

- **Geometria** (posizioni, larghezze contenitori): deve chiudere esatta
  a 800×480. Si gestisce con auto-layout `fill`/`hug`, **non**
  moltiplicando coordinate.
- **Valori atomici** (font, icone, stroke, radius, spacing): si
  **ri-derivano** su una rampa intera, **non** si moltiplicano e
  arrotondano.

Motivo: "moltiplica e arrotonda" accumula errore e deforma la
progressione. Esempio: `space-1` 2→3,33→3 è +50%, mentre `space-6`
12→20 è +67%. La rampa perde armonia.

### b) La rampa font va potata, non tradotta
`tokens.json` ha 12 `fontSize` + 12 `lineHeights`. Tradotti 1:1
diventano 12 font bitmap da generare e tenere in flash — costo reale su
firmware C. **Ridurre a 4–5 taglie effettive prima** di riesportare.

### c) Icone: optical size nativo, non rasterizzazione scalata
`size-4` = 24 → 40 conferma Material 24→40. Ma 40/24 = 1,667 non è un
rapporto pulito per icone line-based disegnate su griglia 24:
rasterizzare l'SVG a 1,667× dà stroke da 1,67px → sfocati.
→ Usare **Material Symbols all'optical size 40 nativo**, disegnato per
quella griglia.

---

## 3. Remap dei token (fattore 1,667× — da rivedere se il pannello cambia)

Colonna "Snap" = valori proposti, multipli di 2.

| Token | Attuale | ×1,667 | Snap |
|---|---|---|---|
| `space-1…7` | 2, 4, 6, 8, 10, 12, 16 | 3.3, 6.7, 10, 13.3, 16.7, 20, 26.7 | **4, 6, 10, 14, 16, 20, 26** |
| `size-0…7` | 8, 10, 12, 16, 24, 36, 54, 80 | 13.3, 16.7, 20, 26.7, 40, 60, 90, 133 | **14, 16, 20, 26, 40, 60, 90, 132** |
| `border-radius-1…6` | 2, 4, 6, 8, 10, 12 | 3.3, 6.7, 10, 13.3, 16.7, 20 | **4, 6, 10, 14, 16, 20** |
| `border-0…3` | 1, 2, 3, 4 | 1.7, 3.3, 5, 6.7 | **2, 3, 5, 6** |
| `fontSize` (potata) | 12, 16, 20, 24, 32 | 20, 26.7, 33.3, 40, 53.3 | **20, 26, 34, 40, 54** |

**Leva chiave:** i 13 stili tipografici compositi (`display`, `hero`,
`h1…h4`, `title`, `body-1/2`, `caption-1/2/3`) **referenziano** i
primitivi via `{fontSize.9}`, `{lineHeights.2}` ecc. Cambiando la rampa
primitiva, tutta la tipografia si aggiorna in un colpo.

### Residuo verticale (~27px)
Non distribuirlo a pioggia. Un rigo di lista scalato passa da ~40 a
~67px: 27px **non** comprano una riga in più. Investirli su **header e
footer**, dove migliorano i target touch senza toccare la griglia dei
contenuti.

Target touch minimo: ~9mm fisici → **≈66px** a 187 PPI.

---

## 4. Bug in `tokens.json` da correggere (verificati)

1. `border-radius-2…6` usano `$border-radius-1 *2`, mentre `space-2…7`
   usano `{space-1} *2`. Tokens Studio risolve `{...}`, **non** `$...`
   → quei 5 radius con ogni probabilità **non stanno risolvendo**.
2. `size-0` vale `"8px"` (stringa con unità), `size-1…7` sono numeri
   nudi → rompe la codegen.
3. `space-7` è `{space-1} *8` (=16), salta il `*7` → buco a 14 nella rampa.

Stato attuale del file: 123 token, **un solo set** (`global`),
`$themes` vuoto. Buona notizia: aggiungere un secondo set è pulito.

---

## 5. Checklist operativa

- [ ] **P0** Decidere la dimensione fisica del pannello (§1)
- [ ] **P0** Audit del file Figma: i valori sono *applicati come
      token/variabili* o *digitati a mano*? ← discriminante della stima
- [ ] Correggere i 3 bug di `tokens.json` (§4)
- [ ] Potare la rampa `fontSize`/`lineHeights` a 4–5 taglie
- [ ] Generare il token set `density/800x480` + voce `$themes`
- [ ] Importare in Tokens Studio e fare lo switch di tema
- [ ] Normalizzare auto-layout sugli schermi (fill/hug, no coordinate)
- [ ] Decidere destinazione dei 27px verticali
- [ ] Arrotondare i residui decimali a interi
- [ ] Riesportare le icone all'optical size nativo 40
- [ ] Generare gli asset per il firmware (formato da definire, §7)
- [ ] Revisione visiva schermo per schermo

---

## 6. Automatizzabile vs. manuale

### Automatizzabile — senza accesso a Figma
- Generazione del token set `density/800x480` + `$themes` in `tokens.json`
- Fix dei 3 bug del token file
- Pipeline asset locale: da SVG → rasterizzazione a dimensioni esatte →
  array C (1-bit o A8) + manifest. Nessun mezzo pixel, nessuno scaling
  a runtime.

### Automatizzabile — dentro Figma, via plugin one-off
Scaling batch, arrotondamento a interi, binding valori hardcoded →
variabili, normalizzazione auto-layout, resize istanze icona, export
settings in blocco.

> **Vincolo tecnico da tenere presente:** nessun MCP può *scrivere* in
> un file Figma. Il Dev Mode MCP ufficiale è read-only e le REST API non
> mutano i nodi del documento. L'unico modo di modificare un file Figma
> in automatico è un **plugin** (Plugin API) lanciato da dentro Figma.

### Necessariamente manuale
- Destinazione dei 27px verticali (scelta di prodotto)
- Schermi in posizionamento assoluto o con istanze scollegate →
  ricostruzione a mano, nessuno script li salva
- Revisione visiva finale

### Plugin di terze parti utili
Scale (nativo, tasto `K` — scala anche stroke, radius, testo) ·
Pixel Perfect / Integerizer (rounding) · Batch Styler (stili testo) ·
Select Same / Similayer (selezione massiva) · Design Lint (stanare
valori hardcoded)

---

## 7. Stima tempi (~10–15 schermi)

| Stato del file | Tempo | Note |
|---|---|---|
| Auto-layout ovunque + token applicati + componenti/varianti | **1–2 gg** | Il grosso è swap del token set + 27px + riesporto asset |
| Misto (auto-layout parziale, valori hardcoded) | **3–5 gg** | Il plugin one-off taglia ~40% |
| Posizionamento assoluto / istanze scollegate | **6–10 gg** | Ricostruire da zero sui token nuovi è più veloce che convertire |

Audit iniziale: 1–2 ore.

**Il vero discriminante:** se i valori sono token/variabili, lo swap del
set fa ~70% del lavoro. Se sono hardcoded, Tokens Studio non tocca nulla
→ terza riga della tabella.

---

## 8. Decisioni ancora da prendere

1. **Dimensione fisica del pannello** → fattore di scala (§1)
2. **Stack grafico C**: LVGL (font via `lv_font_conv`) / TouchGFX /
   custom bare-metal (array C grezzi) → determina il formato di export
   degli asset
3. **Come intervenire su Figma**: plugin one-off scritto ad hoc /
   MCP read-only per l'audit / export SVG lavorati in locale

---

## 9. Accesso al file — stato (verificato)

In sessione Claude Code **web/remota** il file di design non è
raggiungibile. Due muri indipendenti:

1. `figma.com` e `api.figma.com` sono **bloccati dalla network policy**
   dell'ambiente remoto (`EGRESS_BLOCKED`, 403 sul tunnel CONNECT).
2. Nessun connettore Figma esiste sull'account, e nessun token Figma è
   configurato nell'ambiente.

Nota: sbloccare solo `figma.com` non basta — un file di design non è
leggibile come HTML, è un'app JS dietro autenticazione.

### Opzioni per dare accesso in lettura

| | Cosa serve | Cosa ottengo |
|---|---|---|
| **A. REST API Figma** | `api.figma.com` in allowlist + PAT come env var (`file_read`, `variables:read`) | Albero completo del documento: frame, dimensioni, auto-layout, variabili collegate, componenti/varianti → **audit vero** |
| **B. Claude Code in locale** | Girare in locale invece che web | Nessun blocco di rete + Figma Dev Mode MCP contro l'app desktop |
| **C. Screenshot** | Niente | Sblocco immediato, precisione minore |

Il token va messo come **variabile d'ambiente**, non incollato in chat.
In ogni caso resta **sola lettura**: per *scrivere* nel file serve un
plugin (§6).

### Screenshot minimi utili (opzione C)
- 3–4 schermi rappresentativi
- **pannello destro di Figma con un elemento selezionato** ← il dato che
  conta: dice se padding/corner radius sono token o numeri hardcoded
