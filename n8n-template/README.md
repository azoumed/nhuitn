# n8n template — Keyword → Unique Shorts → Publish

Overview
- This repo contains an n8n workflow template that: 1) searches Google for a keyword using SerpAPI, 2) generates unique short-video script and captions via OpenAI, 3) fetches images (Pexels), 4) generates voiceover (ElevenLabs), 5) assembles a short video via Cloudinary (placeholder), and 6) uploads to YouTube (placeholder) and other socials.

Files
- keyword_shorts_workflow.json — n8n workflow export (import into n8n).

Required API keys / credentials (set as environment variables in n8n Credentials or system env):
- `SERPAPI_KEY` — SerpAPI key (or replace with Google Custom Search API)
- `OPENAI_API_KEY` — OpenAI API key
- `PEXELS_API_KEY` — Pexels API key (or Unsplash)
- `ELEVENLABS_KEY` — ElevenLabs TTS API key (or any TTS provider)
- `CLOUDINARY_CLOUD_NAME` / `CLOUDINARY_API_KEY` / `CLOUDINARY_API_SECRET` — if using Cloudinary to assemble video
- `googleOAuth2` — n8n Google OAuth2 credentials for YouTube upload (use n8n credentials UI)

How it works (high level)
1. Cron Trigger fires (or run manually) with an input keyword.
2. `Google / SerpAPI Search` gets SERP results for the keyword.
3. `Extract Top Results` prepares a brief summary of top results.
4. `Generate Script & Caption (OpenAI)` asks OpenAI to produce a short video script, caption, hashtags, and a 6-image storyboard. This template requests STRICT JSON output from OpenAI and includes parsing in `Prepare Media Prompts`.
5. `Fetch Images (Pexels)` downloads images matching storyboard prompts.
6. `Generate Voiceover (TTS)` creates an MP3/ogg voiceover from the script.
7. `(Placeholder) Cloudinary Video Assemble` shows where to upload images + audio and request a Cloudinary video transformation that stitches images into a vertical short with audio overlay.
8. `Upload to YouTube (init)` shows the init call for YouTube resumable upload. Complete the multipart upload steps (README links below).

Sample OpenAI prompt (used by the workflow)
```
Return ONLY valid JSON with the following keys: "script": string (30-45s, 3-6 short sentences), "caption": string (<=150 chars), "hashtags": array of 5 strings, "storyboard": array of 6 short image prompts (strings), "title": short title string.
Generate values based on keyword: <KEYWORD> and topResults: <SERP_TOP_RESULTS>. Output must be pure JSON with no commentary.
```

Notes & next steps
- The workflow includes several placeholder steps: Cloudinary assembly and full YouTube multipart upload require configuration and may need helper functions or an external server endpoint for large file streaming.
- For TikTok/Instagram: use their official APIs (often require Business/Creator accounts and OAuth). Because these APIs vary and require app registration, the template includes placeholders where HTTP Request nodes should be configured.
- For unique-content safety: ensure your OpenAI prompts instruct the model to avoid plagiarism. Prefer `temperature: 0.7-0.9` for creativity but include `max_new_tokens` and content-safety checks.

Recommended improvements
- Update the OpenAI prompt to return strict JSON and parse it in `Prepare Media Prompts` for reliable automation. The workflow includes a robust parsing function that attempts `JSON.parse` and falls back to extracting the first JSON-like substring.
- Replace Cloudinary placeholder with a small serverless function (Azure Function, Cloud Run, or AWS Lambda) that accepts images + audio and runs ffmpeg to create a final MP4, then returns the file url.
- Add retry/error handling and a small database (Airtable, Google Sheets) to track published items and avoid duplicates.

Quick import steps
1. Open n8n editor.
2. Click Import → Paste JSON → paste `keyword_shorts_workflow.json` content.
3. Configure credentials in n8n's Credentials screen (OpenAI, HTTP Request credential values in environment or n8n creds).
4. Test step-by-step: run search → run OpenAI → verify script → fetch images → run TTS → assemble video (manual step first).

Want help?
- I can: 1) convert the Cloudinary placeholder into a runnable Cloudinary/ffmpeg function, 2) help wire up YouTube OAuth and multipart upload in n8n, or 3) test-run a full flow with your API keys (you'd provide them securely). Which would you like me to do next?

**Exact n8n node configs & expressions (copy-paste)**

Below are exact node settings and function code you can paste into n8n nodes to match the workflow. Replace credential placeholders with your n8n credentials or env variables.

- **Check Duplicate** (HTTP Request)
	- Method: `GET`
	- URL expression: `={{ ($env.FFMPEG_SERVICE_URL || 'http://localhost:3000') + '/check?key=' + encodeURIComponent($json.keyword || $json.body || 'untitled') }}`

- **Google / SerpAPI Search** (HTTP Request)
	- Method: `GET`
	- URL: `https://serpapi.com/search.json`
	- Query params:
		- `q`: `={{$json["keyword"] || $json["body"] || 'YOUR_KEYWORD'}}`
		- `engine`: `google`
		- `api_key`: `={{$env.SERPAPI_KEY}}`

- **Generate Script & Caption (OpenAI)** (HTTP Request)
	- Method: `POST`
	- URL: `https://api.openai.com/v1/chat/completions`
	- Headers: `Authorization: =Bearer {{$env.OPENAI_API_KEY}}`, `Content-Type: application/json`
	- Body (raw JSON expression):
```
={
	"model": "gpt-4o-mini",
	"messages": [
		{"role": "system", "content": "You are a content creator that writes short, unique scripts and captions for social media shorts (15-60s). Avoid plagiarism and copyrighted text."},
		{"role": "user", "content": `Return ONLY valid JSON with the following keys: "script": string (30-45s, 3-6 short sentences), "caption": string (<=150 chars), "hashtags": array of 5 strings, "storyboard": array of 6 short image prompts (strings), "title": short title string.\nGenerate values based on keyword: ${$json["keyword"]} and topResults: ${JSON.stringify($json["topResults"])}. Output must be pure JSON with no commentary.`}
	],
	"temperature": 0.8
}
```

- **Prepare Media Prompts** (Function)
	- Function code (copy into the Function node):
```
// Parse OpenAI response and extract structured fields
const out = [];
let content = '';
if (items[0].json.choices && items[0].json.choices[0]) {
	content = items[0].json.choices[0].message?.content || items[0].json.choices[0].text || '';
} else if (typeof items[0].json === 'string') {
	content = items[0].json;
} else {
	content = JSON.stringify(items[0].json);
}
let parsed = {};
try {
	parsed = JSON.parse(content);
} catch (e) {
	const m = content.match(/\{[\s\S]*\}/);
	if (m) {
		try { parsed = JSON.parse(m[0]); } catch (e2) { parsed = {}; }
	}
}
const storyboard = Array.isArray(parsed.storyboard) ? parsed.storyboard : [];
const storyboardPrompt = storyboard.join(' ');
const script = parsed.script || parsed.text || content;
const caption = parsed.caption || '';
const hashtags = Array.isArray(parsed.hashtags) ? parsed.hashtags : [];
const title = parsed.title || '';
out.push({ json: { storyboard, storyboardPrompt, script, caption, hashtags, title } });
return out;
```

- **Fetch Images (Pexels)** (HTTP Request)
	- Method: `GET`
	- URL expression: `={{ `https://api.pexels.com/v1/search?query=${encodeURIComponent($json.storyboardPrompt || $json.prompt || 'stock photo')}&per_page=6` }}`
	- Headers: `Authorization: ={{$env.PEXELS_API_KEY}}`

- **Collect Media + Script** (Function)
	- Function code:
```
// Collect image URLs and script for TTS
const images = items[0].json.photos ? items[0].json.photos.map(p => p.src.medium) : [];
const prev = $items("Prepare Media Prompts")[0].json || {};
return [{ json: { images, script: prev.script || '', caption: prev.caption || '', hashtags: prev.hashtags || [], title: prev.title || '' } }];
```

- **Generate Voiceover (TTS)** (HTTP Request)
	- Method: `POST`
	- URL: your TTS provider endpoint (example used ElevenLabs)
	- Example body expression:
```
={ "text": $json.script, "voice": "alloy" }
```
	- Response format: `file` (so the node returns a `url`/file reference)

- **Assembler (HTTP Request)**
	- POST to: `={{ $env.FFMPEG_SERVICE_URL || 'http://localhost:3000/assemble' }}`
	- Body (JSON expression):
```
={ "images": $items("Collect Media + Script")[0].json.images, "audio": $items("Generate Voiceover (TTS)")[0].json.url || $items("Generate Voiceover (TTS)")[0].json.fileUrl || $json.audio }
```

- **Prepare Upload Payload** (Function)
	- Function code (merges metadata + assembler response):
```
const prevCollect = $items('Collect Media + Script')[0]?.json || {};
const prevPrepare = $items('Prepare Media Prompts')[0]?.json || {};
const assembler = items[0].json || {};
const videoUrl = assembler.url || assembler.output || assembler.fileUrl || (assembler.body && assembler.body.url) || '';
const title = prevCollect.title || prevPrepare.title || prevCollect.caption || prevPrepare.storyboard?.[0] || '';
const caption = prevCollect.caption || prevPrepare.caption || '';
const hashtags = Array.isArray(prevCollect.hashtags) ? prevCollect.hashtags : (Array.isArray(prevPrepare.hashtags) ? prevPrepare.hashtags : []);
return [{ json: { videoUrl, title, caption, hashtags } }];
```

- **Upload to YouTube (init)** (HTTP Request)
	- Method: `POST`
	- URL: `https://www.googleapis.com/upload/youtube/v3/videos?uploadType=resumable&part=snippet,status`
	- Headers: `Authorization: =Bearer {{$credentials.googleOAuth2.access_token}}`, `Content-Type: application/json; charset=UTF-8`
	- Body (expression):
```
={
	"snippet": {
		"title": $json.title || ($json.keyword + ' — Short'),
		"description": $json.caption || '',
		"tags": $json.hashtags || []
	},
	"status": {
		"privacyStatus": "public"
	}
}
```

- **Record Published** (HTTP Request)
	- POST to: `={{ ($env.FFMPEG_SERVICE_URL || 'http://localhost:3000') + '/record' }}`
	- Body expression:
```
={ "key": $json.keyword || '', "title": $json.title || '', "url": $json.videoUrl || $json.url || '', "platform": "youtube" }
```

Notes:
- These expressions match the exported workflow nodes in `keyword_shorts_workflow.json` and are safe to paste into equivalent n8n nodes.
- I cannot include image screenshots in this README, but the snippets above are exact copy-paste content for the node fields.

If you want, I can now: 1) update the README with example Airtable/Google Sheets wiring for duplication tracking, or 2) produce a short checklist for running a full end-to-end local test (start assembler, run `test_request.js`, then run the n8n flow). Which next? 

**Google Sheets (gratuit) — configuration & n8n nodes**

1) Créer la feuille
- Créez une Google Sheet nommée `published` (ou autre) et mettez l'en-tête en première ligne exactement :

	key,title,url,platform,publishedAt

- Copiez l'ID du spreadsheet (dans l'URL) — vous en aurez besoin dans n8n.

2) Vérifier le doublon (nœud `Get All` + Function)
- Nœud 1 — `Get Published` (Google Sheets)
	- Resource: `Spreadsheet` (Google Sheets)
	- Operation: `Get All` (ou `Get Many`)
	- Spreadsheet ID: *votre Spreadsheet ID*
	- Sheet Name: `Sheet1` ou `published`
	- Range: `A2:E` (lit toutes les lignes)

- Nœud 2 — `Check Duplicate` (Function)
	- Code (copiez dans un node Function) :

```
const key = ($json.keyword || $json.body || '').toString().trim().toLowerCase();
const rows = $items('Get Published') || [];
const found = rows.find(r => (r.json.key || '').toString().trim().toLowerCase() === key);
return [{ json: { exists: !!found, record: found ? found.json : null } }];
```

3) Enregistrer la publication (nœud `Append`)
- Nœud — `Record Published` (Google Sheets)
	- Resource: `Spreadsheet`
	- Operation: `Append`
	- Spreadsheet ID: *votre Spreadsheet ID*
	- Sheet Name: `Sheet1` (ou `published`)
	- Range: `A:E`
	- Values: utilisez des expressions pour insérer les valeurs, par exemple :

```
={{ [$json.keyword || '', $json.title || '', $json.videoUrl || $json.url || '', 'youtube', new Date().toISOString()] }}
```

4) Exemple d'enchaînement dans le flow
- `Cron Trigger` → `Get Published` → `Check Duplicate` (Function)
	- Si `exists=true` → arrêter / log
	- Sinon → continuer vers `Google / SerpAPI Search` → génération → assembleur → upload
- Après upload réussi → `Record Published` (Append) pour stocker le `key,title,url,platform,publishedAt`

5) Remarques
- Google Sheets est gratuit et rapide à configurer mais devient moins fiable sur de très gros volumes. Ce wiring est idéal pour prototyper sans frais.
- Pour automatiser proprement, créez dans Google Cloud un projet OAuth et enregistrez des identifiants OAuth pour n8n (ou utilisez n8n cloud credentials). Voir la doc n8n pour `Google Sheets` node OAuth setup.

Fichier modèle: `n8n-template/google_sheet_template.csv` (copiez/importe dans Google Sheets si vous voulez démarrer rapidement).
# n8n template — Keyword → Unique Shorts → Publish

Overview
- This repo contains an n8n workflow template that: 1) searches Google for a keyword using SerpAPI, 2) generates unique short-video script and captions via OpenAI, 3) fetches images (Pexels), 4) generates voiceover (ElevenLabs), 5) assembles a short video via Cloudinary (placeholder), and 6) uploads to YouTube (placeholder) and other socials.

Files
- keyword_shorts_workflow.json — n8n workflow export (import into n8n).

Required API keys / credentials (set as environment variables in n8n Credentials or system env):
- `SERPAPI_KEY` — SerpAPI key (or replace with Google Custom Search API)
- `OPENAI_API_KEY` — OpenAI API key
- `PEXELS_API_KEY` — Pexels API key (or Unsplash)
- `ELEVENLABS_KEY` — ElevenLabs TTS API key (or any TTS provider)
- `CLOUDINARY_CLOUD_NAME` / `CLOUDINARY_API_KEY` / `CLOUDINARY_API_SECRET` — if using Cloudinary to assemble video
- `googleOAuth2` — n8n Google OAuth2 credentials for YouTube upload (use n8n credentials UI)

How it works (high level)
1. Cron Trigger fires (or run manually) with an input keyword.
2. `Google / SerpAPI Search` gets SERP results for the keyword.
3. `Extract Top Results` prepares a brief summary of top results.
4. `Generate Script & Caption (OpenAI)` asks OpenAI to produce a short video script, caption, hashtags, and a 6-image storyboard. For robust parsing, customize the prompt to have OpenAI return strict JSON.
5. `Fetch Images (Pexels)` downloads images matching storyboard prompts.
6. `Generate Voiceover (TTS)` creates an MP3/ogg voiceover from the script.
7. `(Placeholder) Cloudinary Video Assemble` shows where to upload images + audio and request a Cloudinary video transformation that stitches images into a vertical short with audio overlay.
8. `Upload to YouTube (init)` shows the init call for YouTube resumable upload. Complete the multipart upload steps (README links below).

Notes & next steps
- The workflow includes several placeholder steps: Cloudinary assembly and full YouTube multipart upload require configuration and may need helper functions or an external server endpoint for large file streaming.
- For TikTok/Instagram: use their official APIs (often require Business/Creator accounts and OAuth). Because these APIs vary and require app registration, the template includes placeholders where HTTP Request nodes should be configured.
- For unique-content safety: ensure your OpenAI prompts instruct the model to avoid plagiarism. Prefer `temperature: 0.7-0.9` for creativity but include `max_new_tokens` and content-safety checks.

Recommended improvements
- Update the OpenAI prompt to return strict JSON and parse it in `Prepare Media Prompts` for reliable automation.
- Replace Cloudinary placeholder with a small serverless function (Azure Function, Cloud Run, or AWS Lambda) that accepts images + audio and runs ffmpeg to create a final MP4, then returns the file url.
- Add retry/error handling and a small database (Airtable, Google Sheets) to track published items and avoid duplicates.

Quick import steps
1. Open n8n editor.
2. Click Import → Paste JSON → paste `keyword_shorts_workflow.json` content.
3. Configure credentials in n8n's Credentials screen (OpenAI, HTTP Request credential values in environment or n8n creds).
4. Test step-by-step: run search → run OpenAI → verify script → fetch images → run TTS → assemble video (manual step first).

Want help?
- I can: 1) convert the Cloudinary placeholder into a runnable Cloudinary/ffmpeg function, 2) help wire up YouTube OAuth and multipart upload in n8n, or 3) test-run a full flow with your API keys (you'd provide them securely). Which would you like me to do next?