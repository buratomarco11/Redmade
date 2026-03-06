# Template Word - Lettera di Sdoganamento
## File: `LetteraSdoganamento_Template.docx`
## Percorso SharePoint: `/Dogana/Templates/LetteraSdoganamento_Template.docx`

---

## Content Controls (Quick Parts) da inserire nel documento Word

Aprire Word → scheda **Sviluppatore** → **Controllo contenuto testo normale** per ogni campo.
Assegnare il **Tag** (non il titolo visibile) esattamente come indicato sotto.

| Tag del Content Control   | Descrizione                        | Esempio valore             |
|---------------------------|------------------------------------|----------------------------|
| `ImporterCompanyName`     | Ragione sociale importatore        | Redmade S.r.l.             |
| `ImporterVAT`             | Partita IVA importatore            | IT12345678901              |
| `ImporterAddress`         | Indirizzo sede legale              | Via Esempio 1, 20100 Milano|
| `AWB`                     | Air Waybill Number (12 cifre)      | 123456789012               |
| `HSCode`                  | Codice doganale HS                 | 85340000                   |
| `OriginCountry`           | Paese di origine merce             | China                      |
| `InvoiceNumber`           | Numero fattura commerciale         | CI-2024-001                |
| `InvoiceTotal`            | Importo totale fattura             | 647.41                     |
| `Currency`                | Valuta fattura                     | USD                        |
| `SupplierName`            | Nome fornitore/venditore           | Shenzhen Tech Co. Ltd.     |
| `DateToday`               | Data di compilazione lettera       | 2026-03-06                 |

---

## Struttura testo lettera (da replicare nel template)

```
[ImporterCompanyName]
[ImporterAddress]
P.IVA: [ImporterVAT]

Data: [DateToday]

Oggetto: Richiesta documenti sdoganamento – AWB [AWB]

Spett.le FedEx Express Italia,

con la presente trasmettiamo la documentazione necessaria per il
completamento delle operazioni doganali relative alla seguente spedizione:

  Numero AWB:            [AWB]
  Fornitore:             [SupplierName]
  Numero Fattura (CI):   [InvoiceNumber]
  Valore dichiarato:     [InvoiceTotal] [Currency]
  Paese di origine:      [OriginCountry]
  Codice doganale (HS):  [HSCode]

In allegato si trovano:
  1. Fattura Commerciale (Commercial Invoice)
  2. Packing List
  3. Eventuale documentazione aggiuntiva

Si autorizza FedEx a procedere con le operazioni di sdoganamento.

Cordiali saluti,

[ImporterCompanyName]
P.IVA: [ImporterVAT]
```

---

## Note tecniche per n8n (Graph API Populate)

Per popolare il template via **Microsoft Graph API** (endpoint Word Online),
il file DOCX deve contenere Content Controls con i Tag sopra indicati.

Endpoint Graph per populate:
```
POST https://graph.microsoft.com/v1.0/sites/{site-id}/drive/items/{item-id}/workbook
```

Alternativa robusta con **n8n + Azure Function** (docx-templates):
```javascript
// Azure Function - populateTemplate.js
const { createReport } = require('docx-templates');

module.exports = async function(context, req) {
  const templateBuffer = req.body.template; // base64
  const data           = req.body.data;     // oggetto campi

  const report = await createReport({
    template: Buffer.from(templateBuffer, 'base64'),
    data,
    cmdDelimiter: ['{{', '}}'],
  });

  context.res = {
    body: report.toString('base64'),
    headers: { 'Content-Type': 'application/octet-stream' }
  };
};
```

Placeholder nel DOCX con questa modalità: `{{AWB}}`, `{{ImporterCompanyName}}`, etc.

---

## Conversione DOCX → PDF via Graph API

```http
GET https://graph.microsoft.com/v1.0/sites/{site-id}/drive/root:/{path-file.docx}:/content?format=pdf
Authorization: Bearer {token}
```

La risposta è il file PDF come stream binario da salvare su SharePoint.
