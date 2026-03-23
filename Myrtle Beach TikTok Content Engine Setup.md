# Myrtle Beach TikTok Image Content Engine Setup

## Files in this deliverable
- `Myrtle Beach TikTok Content Engine.json` — importable n8n workflow.
- This setup note — implementation details, configurable values, and node behavior summary.

## Configurable values
Set these as n8n environment variables before activating the workflow:

- `BYTEZ_API_KEY` — Bytez API key used for both text and image generation.
- `BYTEZ_TEXT_MODEL` — any Bytez-compatible text/chat model ID for topic and copy generation.
- `BYTEZ_IMAGE_MODEL` — any Bytez-compatible text-to-image model ID.
- `SPREADSHEET_ID` — target Google Sheets spreadsheet ID.
- `SHEET_NAME` — defaults to `ContentQueue`.
- `GOOGLE_DRIVE_FOLDER_ID` — Google Drive folder to receive generated images.
- `CTA_LINK` — defaults to `https://tally.so/r/lb608W`.
- `POSTS_PER_DAY` — defaults to `10`.

## Required sheet tab and columns
Use tab name `ContentQueue` with these exact columns:

1. `content_id`
2. `created_at`
3. `post_date`
4. `category`
5. `topic`
6. `keyword`
7. `hook`
8. `image_style`
9. `image_prompt`
10. `image_url`
11. `caption`
12. `hashtags`
13. `cta`
14. `status`
15. `score`
16. `notes`

## Bytez HTTP node configs

### Topic generation
- Node: `Generate Daily Topics`
- Method: `POST`
- URL: `https://api.bytez.com/models/v2/openai/v1/chat/completions`
- Headers:
  - `Authorization: {{$json.bytezApiKey}}`
  - `Content-Type: application/json`
- Body fields:
  - `model: {{$json.bytezTextModel}}`
  - `temperature: 0.8`
  - `max_tokens: 1400`
  - `response_format: { type: 'json_object' }`
  - `messages`: system + user prompt enforcing strict JSON wrapper `{ "topics": [...] }`

### Copy generation
- Node: `Generate Post Package`
- Method: `POST`
- URL: `https://api.bytez.com/models/v2/openai/v1/chat/completions`
- Headers:
  - `Authorization: {{$json.config.bytezApiKey}}`
  - `Content-Type: application/json`
- Body fields:
  - `model: {{$json.config.bytezTextModel}}`
  - `temperature: 0.9`
  - `max_tokens: 1200`
  - `response_format: { type: 'json_object' }`
  - `messages`: system + user prompt requesting `hook`, `caption`, `hashtags`, `cta`, `image_prompt`, and `score`

### Image generation
- Node: `Generate Image`
- Method: `POST`
- URL: `https://api.bytez.com/models/v2/{{$json.config.bytezImageModel}}`
- Headers:
  - `Authorization: {{$json.config.bytezApiKey}}`
  - `Content-Type: application/json`
- Body:
  - `{ "text": {{$json.final_image_prompt}} }`
- Retry enabled once after initial attempt for stability.

## Function/Code node behavior

### `Parse Topics`
- Extracts `choices[0].message.content` from the Bytez OpenAI-compatible response.
- Removes optional fenced JSON if the model returns it.
- Accepts either a raw array or a wrapper object with `topics`.
- Retries once on JSON parsing failure.

### `Assign Image Style`
- Randomly selects one of the five approved style presets.
- Adds `image_style` and `style_prompt` to each post item.

### `Parse Post Package`
- Parses the copy model JSON.
- Converts `score` to numeric.
- Marks the item `REJECTED` if copy generation fails.

### `Enhance Final Image Prompt`
- Merges the model-generated `image_prompt` with the approved `style_prompt`.
- Enforces overlay text, readability, vertical composition, premium aesthetic, and anti-watermark / anti-gibberish instructions.

### `Prepare Ready/Rejected Row`
- Generates `content_id`.
- Formats `created_at` and `post_date`.
- Preserves `READY` by default.
- Downgrades to `REJECTED` if image generation fails permanently.
- Keeps Google Drive upload failures non-blocking and logs them into `notes` while still allowing `READY` rows.

### `Prepare Rejected Row`
- Builds a sheet row even when copy generation fails before image generation starts.
- Sets `status = REJECTED` and records the failure in `notes`.

## Google Drive setup
- Create or choose a folder in Google Drive.
- Put that folder ID into `GOOGLE_DRIVE_FOLDER_ID`.
- Connect the `Upload Image to Google Drive` node to a Google Drive OAuth2 credential in n8n.
- The node uploads the binary image and attempts to use the returned Drive link as `image_url`.
- If Drive upload fails, the workflow still appends the sheet row with a blank `image_url` and a warning in `notes`.

## Google Sheets setup
- Create or reuse the target spreadsheet.
- Add a `ContentQueue` tab with the exact columns listed above.
- Connect the `Append Row to Google Sheets` node to a Google Sheets OAuth2 credential in n8n.
- Set `SPREADSHEET_ID` to the spreadsheet ID.
- The append node never overwrites existing rows.

## Future TikTok upload branch
The workflow intentionally ends after Google Sheets append so Phase 2 can branch from the READY output and reuse:
- `image_url`
- `caption`
- `hashtags`
- `cta`
- `score`
- `content_id`

A future TikTok Content Posting API draft-upload path can be attached after `Append Row to Google Sheets` or after `Prepare Ready/Rejected Row` if you want approval logic before draft creation.

## Import checklist
1. Import `Myrtle Beach TikTok Content Engine.json` into n8n.
2. Attach Google Sheets OAuth2 credentials.
3. Attach Google Drive OAuth2 credentials.
4. Add the required environment variables.
5. Confirm the `ContentQueue` sheet exists with the exact columns.
6. Run a manual test before enabling the daily schedule.
