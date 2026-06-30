# Deploy n8n to Render

This deploys a hosted n8n instance on Render and then imports the LinkedIn automation workflow.

## What Gets Deployed

- Render Web Service running `docker.io/n8nio/n8n:2.27.5`
- Render Postgres database for n8n workflow/execution data
- Public HTTPS URL for approval links and OAuth callbacks
- `GEMINI_API_KEY` stored as a Render environment variable, not inside the workflow JSON

## Important Free-Tier Reliability Note

The included `render.yaml` uses:

- Web service plan: `free`
- Postgres plan: `free`

This deploys without paid Render resources, but it is not a permanent reliable production setup:

- Render free web services spin down after 15 minutes without inbound traffic.
- If n8n is asleep at 8 AM, the internal n8n schedule can be missed.
- Render free Postgres databases expire after 30 days.

For free testing, this is fine. For reliable daily automation, move the web service to a paid always-on plan and Postgres to a paid database.

## Files To Push

Push these to a private GitHub repo:

```text
render.yaml
suleman_linkedin_share_api_workflow.json
README_SETUP.md
README_RENDER_DEPLOY.md
Sheet1 - Sheet1.csv.csv
chat_context_summary.md
.gitignore
```

Do not push:

```text
Credentials.txt
```

## Deploy On Render

1. Push this folder to a private GitHub repo.
2. In Render, click `New` -> `Blueprint`.
3. Connect the repo.
4. Select `render.yaml`.
5. Deploy.
6. Open the new service after deploy.

If Render gives the service a URL different from:

```text
https://suleman-linkedin-n8n.onrender.com
```

update these environment variables in the Render service:

```text
N8N_HOST=your-real-render-host.onrender.com
N8N_EDITOR_BASE_URL=https://your-real-render-host.onrender.com
WEBHOOK_URL=https://your-real-render-host.onrender.com/
```

Then click `Manual Deploy` -> `Clear build cache & deploy`.

## Add Gemini Key

In Render service environment variables, set:

```text
GEMINI_API_KEY=your_google_ai_studio_key
```

Use the key from `Credentials.txt`, but do not commit that file.

## Configure OAuth Redirect URLs

Use this callback URL for Google and LinkedIn:

```text
https://your-real-render-host.onrender.com/rest/oauth2-credential/callback
```

Add it in:

- Google Cloud Console OAuth client authorized redirect URIs
- LinkedIn Developer App `Auth` redirect URLs

## Create n8n Credentials On Render

Open the Render n8n URL and create the n8n owner account.

Create these credentials in n8n:

```text
Google Sheets account
Gmail account
Linkedin
```

For LinkedIn, request:

```text
w_member_social openid profile email
```

Your local n8n credentials do not automatically exist on Render. Reconnect them in the Render n8n UI, then assign them to the workflow nodes if n8n asks.

## Import And Activate Workflow

1. In Render n8n, import `suleman_linkedin_share_api_workflow.json`.
2. Check these nodes have credentials selected:
   - `Read pending topic`
   - `Email approval preview`
   - `LinkedIn - get authenticated profile`
   - `LinkedIn - initialize image upload`
   - `LinkedIn - upload image binary`
   - `LinkedIn - publish image post`
   - `Mark as Posted`
   - `Mark as Skipped`
   - `Success email`
   - `Skip email`
3. Execute the workflow manually once.
4. Confirm the approval email link starts with your Render URL, not `localhost`.
5. Approve the test post.
6. Publish/activate the workflow so the 8 AM schedule runs automatically.

## Free Automation Checklist

- Deploy the Blueprint.
- Set `GEMINI_API_KEY`.
- Reconnect Google, Gmail, and LinkedIn credentials.
- Import and activate the workflow.
- Use an external free uptime ping service to hit your Render n8n URL before 8 AM, or every 10-12 minutes, if you want the free service to stay awake.

## Production Checklist

- Render service is paid/always-on.
- Render Postgres is paid, not free-expiring.
- `WEBHOOK_URL` uses the real Render HTTPS URL.
- Gemini key is set in Render env vars.
- Google and LinkedIn OAuth redirect URLs use the Render callback URL.
- The workflow is active/published in n8n.
