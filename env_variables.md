# Variabili d'Ambiente n8n
## Workflow: FedEx Customs Clearance - Sdoganamento Automatico

Configurare in **Settings → Environment Variables** di n8n o nel file `.env`.

---

## Variabili Obbligatorie

| Variabile               | Esempio                                  | Descrizione                                      |
|-------------------------|------------------------------------------|--------------------------------------------------|
| `SP_SITE_ID`            | `contoso.sharepoint.com,abc123,def456`   | Site ID SharePoint (da Graph API /sites)         |
| `SP_LIST_REGISTRO_ID`   | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`   | ID lista SharePoint "Registro Sdoganamenti"      |
| `TODO_LIST_ID`          | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`   | ID lista Microsoft To-Do per eccezioni           |
| `IMPORTER_COMPANY_NAME` | `Redmade S.r.l.`                         | Ragione sociale importatore                      |
| `IMPORTER_VAT`          | `IT12345678901`                          | Partita IVA importatore                          |
| `IMPORTER_ADDRESS`      | `Via Esempio 1, 20100 Milano (MI)`       | Indirizzo sede legale importatore                |

---

## Credenziali n8n da Configurare

### 1. Microsoft Outlook OAuth2 (`CRED_OUTLOOK`)
- Type: `microsoftOutlookOAuth2Api`
- Scope: `Mail.Read Mail.ReadWrite Mail.Send`
- App Registration: Azure AD → App registrations → Nuova app

### 2. Microsoft Graph API OAuth2 (`CRED_GRAPH`)
- Type: `oAuth2Api` (Generic OAuth2)
- Authorization URL: `https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/authorize`
- Token URL: `https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token`
- Scope: `https://graph.microsoft.com/.default`
- Permessi API richiesti:
  - `Sites.ReadWrite.All`
  - `Files.ReadWrite.All`
  - `Mail.ReadWrite`
  - `Mail.Send`
  - `Tasks.ReadWrite`
  - `User.Read`

### 3. OpenAI API (`CRED_OPENAI`)
- Type: `openAiApi`
- API Key: da platform.openai.com
- Modello usato: `gpt-4o`

### 4. SMTP (`CRED_SMTP`)
- Type: `smtp`
- Host: es. `smtp.office365.com`
- Port: `587`
- Security: `STARTTLS`
- User/Pass: account mittente notifiche eccezioni

---

## Come ottenere SP_SITE_ID

```http
GET https://graph.microsoft.com/v1.0/sites/{hostname}:/sites/{site-name}
```
Esempio con Graph Explorer:
```
https://graph.microsoft.com/v1.0/sites/contoso.sharepoint.com:/sites/Dogana
```
La risposta contiene: `"id": "contoso.sharepoint.com,GUID1,GUID2"` → usare questo come `SP_SITE_ID`.

---

## Come ottenere SP_LIST_REGISTRO_ID

```http
GET https://graph.microsoft.com/v1.0/sites/{SP_SITE_ID}/lists?$filter=displayName eq 'Registro Sdoganamenti'
```
La risposta contiene: `"id": "GUID"` → usare come `SP_LIST_REGISTRO_ID`.

---

## Struttura Cartelle SharePoint attesa

```
📁 Dogana/
  📁 Templates/
    📄 LetteraSdoganamento_Template.docx   ← template Word con Content Controls
  📁 FedEx/
    📁 2026-03/                            ← anno-mese auto-generato
      📁 123456789012/                     ← AWB come nome cartella
        📄 CI_CommercialInvoice.pdf
        📄 PL_PackingList.pdf
        📄 AWB_AirWaybill.pdf              ← se presente
        📄 LetteraSdoganamento_123456789012.docx
        📄 LetteraSdoganamento_123456789012.pdf
```

---

## Categorie Outlook da creare manualmente

In Outlook → Categorize → All Categories → New:
- `DOGANA - IN LAVORAZIONE` (colore: Giallo)
- `DOGANA - COMPLETATO` (colore: Verde)
- `DOGANA - ECCEZIONE` (colore: Rosso)
