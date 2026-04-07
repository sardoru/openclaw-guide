# Appendix A: Physical Mail Intake Pipeline *(Optional)*

> **This appendix is optional.** It adds a physical-to-digital mail scanning and categorization system. If you don't need physical mail management, skip this entirely.

Most productivity systems stop at the digital boundary. But physical mail — bills, legal notices, insurance, tax documents — still arrives in paper form and creates its own backlog. This appendix bridges the gap: a camera-based scanning system that ingests physical mail into your agent's workflow with automatic OCR categorization.

### What It Does

1. **Snap a photo** of any piece of mail (phone camera, webcam, or automated Pi camera)
2. **OpenAI Vision (gpt-4o)** extracts: sender, description, category, dollar amount, due date, priority
3. **Review & confirm** the extracted data in a UI modal before it's filed
4. **Auto-categorizes** into shelves: Bills/Finance, Bills/Health, Bills/House, Records to File, Personal/Family, To-Do, Marketing, Spam
5. **Archives** scan images and logs entries to an Obsidian-compatible markdown file
6. **Tracks** unpaid amounts, due dates, and action-needed status across all mail

### A.1 Mail Intake Store

Add these types to your app's state management (example uses Zustand):

```typescript
export type MailCategory =
  | 'bills-finance'
  | 'bills-health'
  | 'bills-house'
  | 'records-to-file'
  | 'personal-family'
  | 'todo-non-bills'
  | 'marketing'
  | 'spam'
  | 'unsorted';

export type MailStatus = 'new' | 'opened' | 'action-needed' | 'processed' | 'filed' | 'trashed';

export interface MailItem {
  id: string;
  sender: string;
  description: string;
  category: MailCategory;
  status: MailStatus;
  priority: 'high' | 'medium' | 'low';
  receivedDate: string;
  dueDate?: string;
  amount?: number;
  notes?: string;
  createdAt: string;
}
```

Persist `mailItems` to local storage so the shelf state survives refreshes.

### A.2 OCR Module

Create a server-side module that calls OpenAI Vision to extract structured data from mail photos:

```typescript
// server/mail-ocr.ts
const SYSTEM_PROMPT = `You are a mail scanning assistant. Analyze this photo of physical mail
and extract structured data. Return ONLY valid JSON with these fields:

{
  "sender": "Company or person name",
  "description": "Brief description of the mail content",
  "category": "One of: bills-finance, bills-health, bills-house, records-to-file, personal-family, todo-non-bills, marketing, spam, unsorted",
  "priority": "high (bills with due dates, legal), medium (personal, records), low (marketing, spam)",
  "amount": null or dollar amount as number,
  "dueDate": null or "YYYY-MM-DD",
  "notes": "Brief summary of key content",
  "confidence": 0.0 to 1.0,
  "rawText": "Full text visible in the image"
}`;

export async function analyzeMailImage(
  imageBase64: string,
  hintCategory?: string
): Promise<MailOCRResult> {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
    },
    body: JSON.stringify({
      model: 'gpt-4o',
      messages: [
        { role: 'system', content: SYSTEM_PROMPT },
        {
          role: 'user',
          content: [
            { type: 'text', text: hintCategory
              ? `Category hint: ${hintCategory}. Analyze this mail:`
              : 'Analyze this mail:' },
            { type: 'image_url', image_url: {
              url: `data:image/jpeg;base64,${imageBase64}`,
              detail: 'high'
            }}
          ]
        }
      ],
      max_tokens: 1000,
    }),
  });

  const data = await response.json();
  const content = data.choices[0].message.content;
  return JSON.parse(content); // Validate and coerce fields in production
}
```

### A.3 Ingestion API Endpoint

Add a POST endpoint to your server:

```
POST /api/mail/ingest
Body: { "image": "base64-encoded-jpeg", "category": "bills-finance" (optional) }

Response: {
  "success": true,
  "result": { sender, description, category, priority, amount, dueDate, notes, confidence, rawText },
  "scanPath": "mail-scans/2026-04-03-00-15-30.jpg"
}
```

The endpoint should:
- Save the scan image to `public/mail-scans/` with a timestamp filename
- Run OCR via the module above
- Append to an Obsidian-compatible markdown log (`mail-log.md`)
- Return the structured result for client-side review

### A.4 Camera Upload UI

Build a scan button that opens the phone's rear camera:

```html
<input type="file" accept="image/*" capture="environment" />
```

The flow:
1. User taps "Scan Mail" → phone camera opens
2. Photo captured → resized client-side to max 1024px (saves API tokens)
3. Base64 uploaded to `/api/mail/ingest`
4. OCR results shown in a **review modal** — user can adjust any field
5. On confirm → item added to mail shelf store

### A.5 Category Shelves UI

Build a visual shelf organizer with drag-and-drop between categories:

- Each category is a column/shelf with its own color and icon
- Drag mail items between shelves to re-categorize
- Stats bar shows: total items, new count, action-needed count, unpaid $ total, % processed
- Filter by category and status
- Responsive: phone gets single-column stacked shelves, desktop gets 3-4 column grid

### A.6 Hardware Automation (Future)

Once the software pipeline works with manual phone photos, you can automate with:

- **Raspberry Pi Zero 2W + Camera Module 3** (~$45) mounted over your mail sorting area
- QR code labels on each tray slot → camera detects which category
- Motion detection triggers auto-capture when mail is placed
- Runs the same `/api/mail/ingest` endpoint
- Agent sends Telegram notification: "New mail from AT&T — $142.50 bill due Apr 15"

### A.7 Integration with OpenClaw

Wire mail events into your agent's awareness:

- **Heartbeat check**: scan `mailItems` for overdue bills (status = action-needed, dueDate < today)
- **Daily summary**: include mail stats (new items, unpaid total)
- **Episodic memory**: log mail processing sessions as episodes
- **Obsidian vault**: `mail-log.md` is automatically searchable via QMD

### Benefits

| Without | With |
|---------|------|
| Paper mail piles up unsorted | Every piece scanned and categorized on arrival |
| Bills get missed/late | Due dates tracked, overdue alerts via agent |
| No record of what arrived | Searchable archive with images + OCR text |
| Manual data entry for each bill | Vision AI extracts sender, amount, due date automatically |
| Physical-only filing | Digital + physical — find anything by searching |

---
