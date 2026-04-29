# PROMPT PER ANTIGRAVITY — Copia-incolla come primo messaggio

```
Crea da zero una webapp single-file HTML che analizza conferme d'ordine 
fornitori edili in PDF e calcola il costo di acquisto per pezzo fisico.

STACK: HTML/CSS/JS in un unico file index.html.
DESIGN: dark theme, font DM Sans da Google Fonts.
ENCODING: <meta charset="UTF-8"> obbligatorio nell'head.

═══════════════════════════════════════════════
API SUPPORTATE
═══════════════════════════════════════════════

L'app deve supportare sia Anthropic Claude che Google Gemini.
Il dropdown "Modello AI" deve contenere:

  Claude:
  - Claude Sonnet 4.6      → claude-sonnet-4-6
  - Claude Haiku 4.5       → claude-haiku-4-5
  - Claude Opus 4.7        → claude-opus-4-7

  Gemini:
  - Gemini 2.5 Flash       → gemini-2.5-flash
  - Gemini 2.5 Pro         → gemini-2.5-pro
  - Gemini 2.0 Flash       → gemini-2.0-flash

  Default: claude-sonnet-4-6

L'app rileva dal model string se chiamare:
  - API Anthropic: api.anthropic.com/v1/messages
    Headers: Content-Type, x-api-key, anthropic-version: 2023-06-01,
    anthropic-dangerous-direct-browser-access: true
  - API Google: generativelanguage.googleapis.com/v1beta/models/{model}:generateContent

Il system prompt e lo schema JSON richiesto sono IDENTICI per entrambe.
Il PDF va inviato come base64 in entrambi i casi.

API KEY: un unico campo input, salvata in localStorage.
L'utente cambia chiave quando cambia provider.

═══════════════════════════════════════════════
FUNZIONAMENTO
═══════════════════════════════════════════════

1. L'utente carica un PDF (drag & drop o file picker)
2. Il PDF viene convertito in base64
3. Il base64 viene inviato all'API con il system prompt della PARTE 1
4. L'API restituisce un JSON strutturato
5. Il JavaScript (PARTE 2) calcola il costo per pezzo fisico
6. I risultati vengono mostrati in tabella (PARTE 3)
7. Le verifiche automatiche (PARTE 4) segnalano errori

Segui ESATTAMENTE le istruzioni nel documento allegato.
Non inventare logiche alternative. Implementa quanto descritto.
Il documento è il risultato dell'analisi di 13 documenti reali
di 10 fornitori diversi e risolve tutti i casi speciali trovati.
```

---
---

# DOCUMENTO ALLEGATO — Istruzioni tecniche complete

---

## PARTE 1 — SYSTEM PROMPT PER L'API

Questo è il system prompt da inviare all'LLM insieme al PDF in base64:

```
Analizzi documenti commerciali di fornitori edili italiani (conferme d'ordine,
fatture proforma, offerte di vendita, proposte d'ordine, ordini di vendita).
Restituisci SOLO un oggetto JSON valido. Mai markdown, mai backtick, mai testo.

🛑 ATTENZIONE - SINTASSI JSON: Non inserire MAI virgolette doppie (") all'interno dei valori di testo o delle avvertenze. Se devi virgolettare una parola, usa gli apici singoli ('). Qualsiasi virgoletta doppia non escapata romperà il parser JSON!

══════════════════════════════════════════════════════
REGOLA 1 — IDENTIFICAZIONE DEL PEZZO FISICO
══════════════════════════════════════════════════════

L'obiettivo è trovare il PEZZO FISICO che entra in magazzino:
sacco, secchio, latta, bidone, blocco, tavella, tanica, rotolo,
scatola, blister, cartuccia, fustino, fusto, barattolo, ecc.

NON interessa il prezzo per kg, m², lt, TO. Queste sono UM del listino.

FONTE PRIMARIA per quanti pezzi fisici sono stati ordinati:
Il campo nel documento che conta le confezioni. Si chiama:
N.CONF, Conf.Nr., Quantità ordinata (in PZ), Pezzi/Pcs, o simile.

Contenuto per pezzo = qty_totale_UM ÷ num_pezzi_fisici

VERIFICA: il risultato deve corrispondere a quanto descritto nel nome articolo.

ECCEZIONE ISOLANTI: per i prodotti isolanti — lana di vetro, lana di roccia,
lana minerale, polistirene (EPS/XPS), polistirolo, pannelli isolanti,
lastre isolanti, pannelli acustici, pannelli fonoassorbenti —
l'unità di misura del pezzo fisico è SEMPRE il metro quadro (m2),
NON il singolo pannello e NON il pacco/collo.

Quando il prodotto è un isolante:
  um_pezzo_fisico = "m2"
  num_pezzi_fisici = qty_totale_um (la quantità totale in m2 dell'ordine)
  qty_um_per_pezzo = 1
  contenuto_pezzo = "1 m2"
  prezzo_listino_per_um = prezzo per m2 dal documento (invariato)
  um_prezzo = "m2" (invariato)

Il valore_netto_riga resta il totale della riga come da documento.

Esempio ATS ordine 826:
  Pann lana vetro MINERAL WOOL 35, 276,48 mq a 8,00 €/mq sconto 64%:
    um_pezzo_fisico = "m2"
    num_pezzi_fisici = 276.48
    qty_um_per_pezzo = 1
    contenuto_pezzo = "1 m2"
    prezzo_listino_per_um = 8.00
    valore_netto_riga = 796.26

Il calcolo poi darà automaticamente:
  Listino/m2 = 8,00€
  Netto/m2 = 2,88€ (8,00 × 0,36)
  Oneri/m2 = 0,14€ (40€ trasporto ÷ 276,48 proporzione)
  Costo/m2 = 3,02€

Come riconoscere un isolante: cercare nel codice o nella descrizione queste parole chiave:
  lana vetro, lana roccia, lana minerale, mineral wool, rock wool, glass wool,
  polistirene, polistirolo, EPS, XPS, isolante, isolamento, pannello termico,
  pannello acustico, fonoassorbente, fonoisolante, Silence, Acoustic,
  MINERAL WOOL, ROCK WOOL, Styrodur, Neopor, Stiferite, Europlus

══════════════════════════════════════════════════════
REGOLA 2 — PATTERN DA RICONOSCERE NEL NOME ARTICOLO
══════════════════════════════════════════════════════

CONTENITORI SEMPLICI (un numero + unità):
  "fustini 5 kg" → fustino da 5 kg
  "sacchi 25 kg" → sacco da 25 kg
  "bucket 10kg" → secchio da 10 kg
  "Latta 20 KG" → latta da 20 kg
  "taniche 5lt" → tanica da 5 lt
  "Latta cont. 5 l" → latta da 5 L
  "Latta da 5 L (7,1kg)" → latta da 5L, peso 7,1 kg
  "Sacco cont. 25 kg" → sacco da 25 kg

CONFEZIONI MULTIPLE (pattern N×M → moltiplicare):
  "scatole 4x5 kg" → scatola contenente 4 sacchetti da 5 kg = 20 kg/scatola
  "scatole 4X5 kg" → identico (X maiuscola)
  "scatole 10x1lt" → scatola da 10×1 = 10 lt
  "box of 50" → scatola contenente 50 pezzi

  ATTENZIONE: NON confondere con dimensioni fisiche.
  Dimensioni: 3 numeri (LxPxH) in cm/mm → "10x62,5x25" = dimensione blocco
  Confezioni: 2 numeri (N×M) seguiti da kg/lt/pz → "4x5 kg" = 4 sacchetti da 5 kg

BLOCCHI E TAVELLE (pezzi per bancale tra parentesi):
  "sp.10x62,5x25 maschiato (96)" → 96 pezzi per bancale
  "sp.5x62,5x25 liscia(144)" → 144 pezzi per bancale
  "25kg/sac-42sac/pal" → 42 sacchi per bancale
  num_pezzi_fisici = pezzi_per_bancale × num_bancali_ordinati (PAL)

PANNELLI ISOLANTI (m² per pezzo dalle dimensioni):
  "1200x600x40mm" → 1,2 × 0,6 = 0,72 m²/pannello
  "1000x600" → 1,0 × 0,6 = 0,60 m²/pannello
  num_pezzi_fisici = qty_totale_m2 ÷ m2_per_pannello

CARTUCCE:
  "Unipac cont. 600 ml" → pezzo = cartuccia/unipac

══════════════════════════════════════════════════════
REGOLA 3 — SCONTI (estrarre come array ordinato)
══════════════════════════════════════════════════════

  - Un solo sconto: "50%" → sconti: [50]
  - Due a cascata: "45,00" poi "15,00" → sconti: [45, 15]
  - Tre a cascata: "-50% / -10% / -35%" → sconti: [50, 10, 35]
  - "Sconto gruppo 60,500-%" → sconti: [60.5]
  - Nessuno → sconti: []

══════════════════════════════════════════════════════
REGOLA 4 — IVA NON È UNO SCONTO
══════════════════════════════════════════════════════

L'aliquota IVA (22%, 10%, 4%) NON è mai uno sconto commerciale.

L'IVA si trova in:
  - Colonna "IVA" con codice (es. "22", "D22", "IVA22")
  - Sezione riepilogo IVA in calce ("IVA/VAT 22%")
  - Riga separata tipo "IVA vendite - 22%"

Gli sconti si trovano in:
  - Colonna "Sconto/Discount" sulla riga articolo
  - Label "Sconto 1", "Sconto 2", "Sc%", "Sconto gruppo"
  - Righe con percentuali negative sotto il prezzo di listino

Se un valore "22" appare sia come IVA che come possibile sconto,
è IVA se si trova in una colonna IVA, in un riepilogo fiscale,
o dopo una riga "Imponibile". NON estrarlo come sconto.

══════════════════════════════════════════════════════
REGOLA 5 — IMPONIBILE vs TOTALE IVA INCLUSA
══════════════════════════════════════════════════════

Estrarre SEPARATAMENTE:
  totali.imponibile → BASE IVA COMPLETA (senza IVA)
  totali.totale_iva_inclusa → importo CON IVA

IMPORTANTE: totali.imponibile deve essere la BASE IVA COMPLETA,
cioè il valore su cui il fornitore calcola l'IVA.
Comprende articoli + trasporto + bancali + sovrapprezzi — tutto ciò soggetto a IVA.

CONTROLLO: imponibile × (1 + aliquota_IVA) deve essere ≈ totale_iva_inclusa.
Se non quadra, ricavare l'imponibile corretto: totale_iva_inclusa ÷ 1,22.

IMPONIBILE (senza IVA):
  "Imponibile" / "Imponibile al 22%" / "Base IVA"
  "Totale Netto" / "Netto in corso" — usare QUESTO nei documenti Mapei
    perché include il trasporto Tra., a differenza di "Totale Merce"
  "Totale Merce" — SOLO se non esiste anche un "Totale Netto" separato

TRASPORTO FISSO SEPARATO (es. ATS Isolanti):
  Se il documento mostra un "Totale Imponibile" o "Totale Merce" come
  subtotale articoli E le spese trasporto sono elencate a parte sopra
  la riga IVA → SOMMARLI per ottenere la base IVA reale:
    totali.imponibile = subtotale_articoli + spese_trasporto_fisso
  Esempio ATS: "Totale Imponibile 1.111,45" + "Spese Trasporto 40,00"
    → totali.imponibile = 1151.45

TOTALE IVA INCLUSA:
  "Totale fattura" / "Totale da Pagare"
  "Totale documento" / "Importo totale" / "Totale"
  Qualsiasi importo che SEGUE una riga "IVA XX%" con l'importo IVA

══════════════════════════════════════════════════════
REGOLA 6 — ONERI ACCESSORI
══════════════════════════════════════════════════════

Cercare: Tra., Trasporto, Spese Trasporto, Porto, Carburante,
Sovrapprezzo, Bancale, Pallet, CONAI, Addebito, Spese Accessorie,
Sc. Incondiz., EXW, Franco, CPT, DAP.

TIPI DI TRASPORTO (estrarre in trasporto_tipo):

  "incluso" → porto franco, franco destino, CPT, DAP → nessun addebito
  
  "per_um_riga" → colonna Tra. sulla riga articolo (es. Mapei: 0,022 €/kg).
    ATTENZIONE: il valore Tra. è in €/UM (stessa unità del prezzo).
    Va MOLTIPLICATO per il contenuto del pezzo per ottenere l'onere per pezzo.
    Può avere valori DIVERSI per articoli con UM diverse nello stesso ordine
    (es. 0,022 €/kg per solidi, 0,024 €/lt per liquidi).
  
  "riga_separata" → riga "Addebito Spese Trasporto" sotto ogni articolo.
    Il prezzo indicato è GIÀ per pezzo/sacco. NON moltiplicare per nulla.
    Associare alla riga articolo immediatamente precedente.
  
  "fisso_globale" → importo fisso per l'intero ordine (es. "Spese Trasporto 40,00").
    Da ripartire proporzionalmente al VALORE NETTO di ogni riga articolo.
  
  "percentuale_valore" → % sul totale (es. "3% imp. materiale oltre €1.500").
    Estrarre sia la percentuale che l'eventuale soglia.
  
  "soglia_franchigia" → gratis sopra una soglia (es. "franco da 800 euro").
    Se il totale ordine è sotto soglia: avvertenza trasporto non quantificato.
  
  "exw" → ritiro cliente, trasporto a carico dell'acquirente, non addebitato.

RIGHE NON-ARTICOLO:

  "Bancale a rendere" / "BANCALE A RENDERE CONSEGNATO" → bancale_rendere
    COSTO = 0. È una cauzione rimborsabile. MAI includere nel costo.
  
  REGOLA EUROPALLET: I bancali di tipo Europallet / EUR / EPAL / 
  "Bancale Europallet" / "Europallet 1200x800" sono SEMPRE bancale_rendere,
  anche se il documento li segna con [V], "venduto", o mostra un prezzo.
  Sono cauzioni rimborsabili. tipo_riga = "bancale_rendere".
  Solo i "Bancale vendita cliente" o "Bancale a perdere" sono costi reali.

  "Bancale a perdere" → bancale_perdere → includere nel costo
  
  "Bancale vendita cliente" / "Bancale venduto" → bancale_venduto
    Includere nel costo. Distribuire 1 bancale per PAL ordinato:
    costo_bancale_per_pezzo = prezzo_bancale ÷ pezzi_per_bancale
  
  "Addebito Spese Trasporto" → onere_riga
    Associare all'articolo precedente (articolo_padre_pos).
  
  "Sovrapprezzo Carburante" e simili → sovrapp_globale
    Inserire sia in articoli[] (tipo_riga: "sovrapp_globale")
    SIA in oneri_generali.altri_oneri[] (tipo: "sovrapp_globale", importo: valore).
    Questo garantisce la distribuzione anche se Step 6B non è raggiunto.
    Ripartire per valore su SOLO gli articoli principali.
  
  "Sc. Incondiz. Val." → sconto a valore sull'intero ordine

UM MISTE: ogni riga ha la propria UM. Estrarre per singola riga.

VALORE NETTO RIGA: se il documento mostra un totale di riga già scontato
(es. "Valore netto articolo", "Valore netto di posizione", "Importo"),
estrailo in valore_netto_riga. È la fonte più precisa per il calcolo.

REGOLE PER EVITARE ERRORI JSON:
- Il campo "note" deve essere brevissimo (max 10 parole) o null
- NON ripetere la descrizione del prodotto nel campo note
- Il campo "contenuto_pezzo" deve essere solo il valore e l'unità: "25 kg", "5 lt", "4x5 kg"
  MAI frasi come "scatola contenente 4 sacchetti da 5 kg"
- Se il documento ha più di 10 articoli, ometti il campo "note" (metti null)
- NON usare MAI virgolette doppie (") dentro i valori di testo
- NON usare caratteri speciali: ×, °, ², ³ → usa x, gradi, m2, m3

══════════════════════════════════════════════════════
SCHEMA JSON
══════════════════════════════════════════════════════

{
  "fornitore": {"nome": "", "piva": ""},
  "documento": {
    "tipo": "conferma_ordine|fattura_proforma|offerta_vendita|proposta_ordine|ordine_vendita|restituzione",
    "numero": "",
    "data": "YYYY-MM-DD",
    "resa": "",
    "pagamento": ""
  },
  "oneri_generali": {
    "trasporto_tipo": "incluso|per_um_riga|riga_separata|fisso_globale|percentuale_valore|soglia_franchigia|exw|non_specificato",
    "trasporto_importo_fisso": null,
    "trasporto_percentuale": null,
    "trasporto_soglia": null,
    "sconto_incondizionato_valore": null,
    "altri_oneri": [
      {"descrizione": "", "tipo": "", "importo": 0, "base_ripartizione": "valore"}
    ]
  },
  "totali": {
    "imponibile": null,
    "totale_iva_inclusa": null,
    "peso_kg": null
  },
  "articoli": [
    {
      "pos": 1,
      "codice": "",
      "descrizione": "",
      "tipo_riga": "articolo|bancale_rendere|bancale_venduto|bancale_perdere|onere_riga|sovrapp_globale",
      "articolo_padre_pos": null,
      "num_pezzi_fisici": 0,
      "um_pezzo_fisico": "sacco|secchio|latta|blocco|tanica|rotolo|scatola|blister|cartuccia|fustino|pz",
      "contenuto_pezzo": "",
      "um_prezzo": "kg|lt|L|TO|m2|pc|pz|mt",
      "qty_totale_um": 0,
      "qty_um_per_pezzo": 0,
      "prezzo_listino_per_um": 0,
      "sconti": [],
      "onere_per_um_riga": 0,
      "valore_netto_riga": null,
      "pezzi_per_bancale": null,
      "num_pal_ordinati": null,
      "iva_perc": 22,
      "note": null
    }
  ],
  "avvertenze": []
}
```

---

## PARTE 2 — LOGICA DI CALCOLO JAVASCRIPT

Questa è la funzione che l'app deve usare per calcolare i costi dopo aver ricevuto il JSON dall'API.

```javascript
function calcolaCosti(data) {
  const articoli = data.articoli.filter(a => a.tipo_riga === 'articolo');
  const oneri = data.oneri_generali;

  // Base per ripartizioni proporzionali: solo articoli principali
  const totValore = articoli.reduce((s, a) => s + getValoreRiga(a), 0);

  const risultati = articoli.map(art => {

    // ═══ STEP 1: Prezzo netto per UM (sconti a cascata) ═══
    let prezzoNettoUM = art.prezzo_listino_per_um;
    for (const sc of art.sconti) {
      prezzoNettoUM *= (1 - sc / 100);
    }

    // ═══ STEP 2: Contenuto per pezzo fisico ═══
    // Fonte primaria: dal JSON. Fallback: calcolo.
    const qpc = art.qty_um_per_pezzo
      || (art.qty_totale_um / art.num_pezzi_fisici);

    // ═══ STEP 3: Prezzo netto per pezzo ═══
    // METODO PREFERITO: valore_netto_riga ÷ num_pezzi_fisici
    // Questo usa il valore esatto del fornitore, senza arrotondamenti.
    // Fallback: prezzo_netto_per_UM × contenuto_per_pezzo
    let nettoPz;
    if (art.valore_netto_riga && art.num_pezzi_fisici > 0) {
      nettoPz = art.valore_netto_riga / art.num_pezzi_fisici;
    } else {
      nettoPz = prezzoNettoUM * qpc;
    }

    // ═══ STEP 4: Onere trasporto per pezzo ═══
    let onereTrPz = 0;

    if (oneri.trasporto_tipo === 'per_um_riga' && art.onere_per_um_riga > 0) {
      // Colonna Tra.: €/UM → MOLTIPLICARE per contenuto pezzo
      // Es: Tra.=0,022 €/kg, pezzo=25kg → 0,022×25 = 0,55 €/pezzo
      onereTrPz = art.onere_per_um_riga * qpc;
    }
    else if (oneri.trasporto_tipo === 'riga_separata') {
      // Riga separata: prezzo GIÀ per pezzo → NON moltiplicare
      const righeOnere = data.articoli.filter(
        r => r.tipo_riga === 'onere_riga' && r.articolo_padre_pos === art.pos
      );
      onereTrPz = righeOnere.reduce((s, r) => s + r.prezzo_listino_per_um, 0);
    }
    else if (oneri.trasporto_tipo === 'fisso_globale' && oneri.trasporto_importo_fisso > 0) {
      // Fisso globale: ripartire per valore
      const valRiga = getValoreRiga(art);
      const quota = oneri.trasporto_importo_fisso * (valRiga / totValore);
      onereTrPz = quota / art.num_pezzi_fisici;
    }
    else if (oneri.trasporto_tipo === 'percentuale_valore' && oneri.trasporto_percentuale > 0) {
      // Percentuale sul totale (con eventuale soglia)
      const totImp = data.totali.imponibile || totValore;
      if (!oneri.trasporto_soglia || totImp > oneri.trasporto_soglia) {
        const onereStimato = totImp * oneri.trasporto_percentuale / 100;
        const valRiga = getValoreRiga(art);
        onereTrPz = onereStimato * (valRiga / totImp) / art.num_pezzi_fisici;
      }
    }
    // incluso, exw, soglia_franchigia soddisfatta → onereTrPz resta 0

    // ═══ STEP 5: Bancali ═══
    let onereBancPz = 0;

    // 5A: Bancali con padre esplicito
    const bancDiretti = data.articoli.filter(r =>
      ['bancale_venduto', 'bancale_perdere'].includes(r.tipo_riga)
      && r.articolo_padre_pos === art.pos
    );
    for (const b of bancDiretti) {
      const pzBanc = art.pezzi_per_bancale || art.num_pezzi_fisici;
      onereBancPz += b.prezzo_listino_per_um / pzBanc;
    }

    // 5B: Bancali senza padre ma con PAL corrispondenti
    // Distribuire 1 bancale per PAL ordinato per articolo
    if (bancDiretti.length === 0 && art.num_pal_ordinati > 0) {
      const bancSenzaPadre = data.articoli.filter(r =>
        r.tipo_riga === 'bancale_venduto' && !r.articolo_padre_pos
      );
      for (const b of bancSenzaPadre) {
        const bancPerArt = art.num_pal_ordinati;
        onereBancPz += (bancPerArt * b.prezzo_listino_per_um) / art.num_pezzi_fisici;
      }
    }
    // Bancale_rendere → MAI includere (costo = 0)

    // ═══ STEP 6: Altri oneri globali (sovrapprezzi, ecc.) ═══
    // Base di ripartizione: SOLO valore articoli principali (totValore)
    // NON includere bancali o altri oneri nella base
    let altriOnPz = 0;
    for (const ao of (oneri.altri_oneri || [])) {
      if (ao.importo > 0) {
        const valRiga = getValoreRiga(art);
        const quota = ao.importo * (valRiga / totValore);
        altriOnPz += quota / art.num_pezzi_fisici;
      }
    }

    // ═══ STEP 7: Sconto incondizionato a valore ═══
    let scIncPz = 0;
    if (oneri.sconto_incondizionato_valore) {
      const valRiga = getValoreRiga(art);
      scIncPz = oneri.sconto_incondizionato_valore
        * (valRiga / totValore) / art.num_pezzi_fisici;
    }

    // ═══ STEP 8: Totale ═══
    const onereTotPz = onereTrPz + onereBancPz + altriOnPz;
    const costoPz = nettoPz + onereTotPz + scIncPz;

    return {
      codice: art.codice,
      descrizione: art.descrizione,
      um_pezzo: art.um_pezzo_fisico,
      contenuto: art.contenuto_pezzo,
      num_pezzi: art.num_pezzi_fisici,
      listino_pz: art.prezzo_listino_per_um * qpc,
      sconti: art.sconti,
      netto_pz: nettoPz,
      oneri_pz: onereTotPz,
      costo_pz: costoPz,
      tot_riga: costoPz * art.num_pezzi_fisici,
    };
  });

  // ═══ VERIFICHE AUTOMATICHE ═══
  const avvertenze = [...(data.avvertenze || [])];

  // CHECK 1: Quadratura — SEMPRE contro IMPONIBILE, mai totale IVA inclusa
  const totCalc = risultati.reduce((s, r) => s + r.tot_riga, 0);
  let totDoc = data.totali.imponibile; // MAI usare totale_iva_inclusa

  // Correzione imponibile per per_um_riga:
  // Se il trasporto è per_um_riga e l'imponibile estratto sembra essere
  // solo il Totale Merce (senza trasporto), correggerlo automaticamente.
  if (oneri.trasporto_tipo === 'per_um_riga') {
    // Calcola il totale trasporto dalle righe articolo
    const totTra = articoli.reduce((s, a) => {
      const traUM = a.onere_per_um_riga || 0;
      const kgTot = (a.num_pezzi_fisici || 0) * (a.qty_um_per_pezzo || 1);
      return s + traUM * kgTot;
    }, 0);

    // Se la differenza tra totCalc e totDoc corrisponde al totale Tra.,
    // significa che l'AI ha preso Totale Merce invece di Totale Netto
    const diff = totCalc - totDoc;
    if (diff > 0 && Math.abs(diff - totTra) < 2) {
      avvertenze.push('ℹ️ Imponibile corretto da Totale Merce ('
        + totDoc.toFixed(2) + '€) a Totale Netto ('
        + (totDoc + totTra).toFixed(2) + '€) — trasporto Tra. incluso');
      totDoc = totDoc + totTra;
    }
  }

  // [Bug 10] Trasporto percentuale_valore non incluso nell'imponibile
  // (es. Akifix: 3% sul materiale, aggiunto in fattura, non nel documento ordine)
  if (oneri.trasporto_tipo === 'percentuale_valore' && totDoc && totDoc > 0) {
    const perc = oneri.trasporto_percentuale || 0;
    const soglia = oneri.trasporto_soglia || 0;
    if (perc > 0 && (!soglia || totDoc > soglia)) {
      const trasportoCalc = totDoc * (perc / 100);
      const diff = totCalc - totDoc;
      if (diff > 0 && Math.abs(diff - trasportoCalc) < totDoc * 0.005) {
        avvertenze.push('ℹ️ Trasporto ' + perc + '% applicato'
          + (soglia > 0 ? ' (>soglia €' + soglia + ')' : '')
          + ': +€' + trasportoCalc.toFixed(2)
          + ' incluso nel costo/pz, non nell\'imponibile doc');
        totDoc = totDoc + trasportoCalc;
      }
    }
  }

  if (totDoc && totDoc > 0) {
    const scarto = Math.abs(totCalc - totDoc) / totDoc * 100;
    if (scarto > 2) {
      avvertenze.push('❌ Scarto ' + scarto.toFixed(1) + '% — calc. '
        + totCalc.toFixed(2) + '€ vs imponibile ' + totDoc.toFixed(2) + '€');
    } else if (scarto > 0.5) {
      avvertenze.push('⚠️ Scarto ' + scarto.toFixed(1) + '% (arrotondamenti)');
    }
  }

  // CHECK 2: Sanity prezzo — pezzo fisico identificato male?
  for (const r of risultati) {
    if (r.costo_pz < 0.10 && !['pz','blister','cartuccia'].includes(r.um_pezzo)) {
      avvertenze.push('⚠️ Prezzo anomalo ' + r.codice + ': '
        + r.costo_pz.toFixed(2) + '€/pz — probabile errore pezzo fisico');
    }
  }

  // CHECK 3: Coerenza pezzi × contenuto = quantità totale
  for (const art of articoli) {
    const qpc = art.qty_um_per_pezzo || (art.qty_totale_um / art.num_pezzi_fisici);
    const check = art.num_pezzi_fisici * qpc;
    const diffPct = Math.abs(check - art.qty_totale_um) / art.qty_totale_um;
    if (diffPct > 0.01) {
      avvertenze.push('⚠️ Verifica pezzi ' + art.codice + ': '
        + art.num_pezzi_fisici + ' × ' + qpc.toFixed(3) + ' = '
        + check.toFixed(3) + ', atteso ' + art.qty_totale_um
        + ' (stessa UM? non confondere TO con kg)');
    }
  }

  // CHECK 4: IVA confusa con sconto?
  for (const art of articoli) {
    for (const sc of art.sconti) {
      if ([22, 10, 4].includes(sc) && art.iva_perc === sc) {
        avvertenze.push('⚠️ Possibile confusione IVA/sconto: '
          + sc + '% su ' + art.codice + ' — verificare');
      }
    }
  }

  return { risultati, avvertenze, totale_calcolato: totCalc };
}

function getValoreRiga(art) {
  if (art.valore_netto_riga) return art.valore_netto_riga;
  let p = art.prezzo_listino_per_um;
  for (const s of art.sconti) p *= (1 - s / 100);
  return p * art.qty_totale_um;
}
```

---

## PARTE 3 — TABELLA RISULTATI

Colonne da mostrare per ogni articolo principale (tipo_riga = "articolo"):

| Colonna | Contenuto |
|---------|-----------|
| CODICE | Codice prodotto fornitore |
| DESCRIZIONE | Nome prodotto. Sotto in piccolo: tipo pezzo (es. "sacco 25kg") |
| QTÀ | num_pezzi_fisici + tipo (es. "100 sacco", "864 blocco", "5 latta") |
| CONT./PZ | Contenuto pezzo (es. "25 kg", "20 kg (4×5)", "5 lt", "blocco") |
| LISTINO | Prezzo listino per PEZZO FISICO. Sotto in piccolo: prezzo/UM originale |
| SCONTI | Sconti formattati (es. "50%+10%+35%") oppure "—" se nessuno |
| NETTO/PZ | Prezzo netto per pezzo fisico dopo tutti gli sconti |
| ONERI/PZ | Somma trasporto + bancali + sovrapprezzi per pezzo. "—" se zero |
| COSTO/PZ | **Netto + Oneri = costo finale acquisto** (colore evidenziato) |
| TOT. RIGA | Costo/pz × Quantità |

NON mostrare in tabella le righe:
bancale_rendere, onere_riga, sovrapp_globale, bancale_venduto, bancale_perdere
(i loro costi sono già incorporati nella colonna ONERI/PZ)

SOTTO LA TABELLA mostrare:
- Riepilogo oneri applicati (tipo trasporto, importo, metodo ripartizione)
- Semaforo di verifica (verde ✅ / giallo ⚠️ / rosso ❌) con dettaglio
- Lista avvertenze AI se presenti
- Pulsante "Nuovo documento" per ricominciare

---

## PARTE 4 — VERIFICHE AUTOMATICHE

L'app esegue su OGNI documento analizzato, senza valori precaricati:

**CHECK 1 — Quadratura totale**
Somma dei tot_riga calcolati vs IMPONIBILE dichiarato nel documento.
MAI confrontare con il totale IVA inclusa.
  ✅ Scarto < 0,5%
  ⚠️ Scarto 0,5% - 2% (arrotondamenti)
  ❌ Scarto > 2% (errore di calcolo, mostrare dettaglio)

**CHECK 2 — Sanity prezzo**
Se costo/pz < 0,10€ per prodotti che non sono minuteria (viti, tasselli, cartucce):
probabile errore nell'identificazione del pezzo fisico
(es. prezzo/kg usato come prezzo/sacco, prezzo/TO diviso per kg).

**CHECK 3 — Coerenza pezzi**
num_pezzi × qty_um_per_pezzo deve essere ≈ qty_totale_um (stessa UM).
Scarto > 1% → avvertenza. Controllare che non si confrontino TO con kg.

**CHECK 4 — IVA confusa con sconto**
Se tra gli sconti estratti c'è un valore uguale all'aliquota IVA
dell'articolo (tipicamente 22, 10, 4) → avvertenza.

**CHECK 5 — N.CONF**
Se il documento ha un campo N.CONF esplicito e num_pezzi_fisici
non coincide → avvertenza.
