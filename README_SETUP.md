# Suleman Ahmed LinkedIn Sheet Image Automation

This n8n workflow creates a LinkedIn system-design post using Gemini for the post text and the `Image` URL from your Google Sheet for the post image.

## Import This Workflow

Import:

```text
suleman_linkedin_share_api_workflow.json
```

The previous generated workflow filenames have also been updated to this same sheet-image Share API flow, but this is the clearest file to use.

## What It Does

1. Runs daily at 8 AM.
2. Reads Google Sheet rows with `Status = Pending`.
3. Selects the first pending row that has a non-empty `Image` URL.
4. Sends only `Topic` and `Description` to Gemini.
5. Gemini returns only the LinkedIn post text.
6. Downloads the sheet `Image` URL as binary field `image`.
7. Emails you the post text and downloaded image for approval.
8. If approved, uploads that same binary image to LinkedIn.
9. Publishes the LinkedIn post with text + image.
10. Marks the sheet row as `Posted` or `Skipped`.

## Google Sheet Columns

Use these exact columns:

```text
Topic | Description | Image | Status | Post Text | Post Date | LinkedIn Asset
```

The `Image` value must be a direct public image URL that returns an image MIME type such as:

```text
image/jpeg
image/png
```

Good examples:

```text
https://miro.medium.com/v2/1*DZ7k_c4ByIkKZh4g1HKa6Q.png
https://cdn.media.amplience.net/i/epammarketplace/core-processes-of-aiops-vs-mlops-vs-llmops?maxW=1200&qlt=80&fmt=jpg
```

Avoid normal webpage URLs, Google Images result pages, private Google Drive links, Pinterest pages, or anything that needs login.

## Credentials

The workflow already preserves your n8n credential references:

- Google Sheets account
- Gmail account
- Linkedin

The Gemini HTTP Request node reads the API key from the `GEMINI_API_KEY` environment variable. On Render, set this in the service environment variables using the key from `Credentials.txt`.

## Test One Post

1. Open n8n.
2. Import `suleman_linkedin_share_api_workflow.json`.
3. Click `Execute workflow`.
4. Check the Gmail approval email.
5. Click Approve to publish, or Skip to mark the row skipped.

## Notes

- No GIF is generated or uploaded.
- No local renderer is used.
- No Gemini image-generation quota is needed.
- External API calls have retry/backoff enabled, but failed LinkedIn/Gmail/Google/Gemini outages will still stop the run instead of falsely marking a row as posted.
- Approval links are validated; only `approve` publishes and only `skip` marks the row skipped.
- The LinkedIn profile is validated from `/v2/userinfo` before image upload or publishing.
- The downloaded image is validated before approval email: it must be a direct JPG/PNG image and should be under 10MB.
- LinkedIn image upload uses `/rest/images?action=initializeUpload`, then binary upload to the returned `uploadUrl`.
- LinkedIn publishing uses `/rest/posts` with the returned `urn:li:image:*` id.
- The workflow sends an escaped copy of the caption to LinkedIn because `/rest/posts` treats characters like `(`, `)`, `[`, `]`, `*`, `_`, `#`, and `@` as reserved text-format characters. The approval email and sheet still show the normal unescaped caption.
- Your LinkedIn credential needs the `w_member_social` scope and the app must have the Share on LinkedIn product enabled.
- Rows with blank `Image` values are ignored until you add an image URL.
