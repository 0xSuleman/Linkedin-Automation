# LinkedIn Automation & Renderer Server Migration Log

This document summarizes the exact context, problems, and final solutions achieved during this debugging session.

## 1. Renderer Server Crashes
**Problem:** The `renderer_server.mjs` was throwing `TypeError: data.readUInt32BE is not a function` and Puppeteer `TargetCloseError`. This happened because the server was attempting to render all 18 frames of the GIF simultaneously using `Promise.all()`. This opened 18 separate Chromium tabs at the exact same moment, causing an Out-Of-Memory (OOM) crash.
**Solution:** Refactored the `renderer_server.mjs` to use a **single sequential loop** on a single Puppeteer page. It now loads the frame, takes a screenshot, and moves to the next frame sequentially. 
*Note: Make sure your terminal running `npm start` is restarted to pull in these new changes!*

## 2. Google Sheets "Wait for Approval" Bug
**Problem:** The workflow was updating the first row regardless of its status, meaning it would overwrite already published rows instead of processing the next `Pending` topic.
**Solution:** We modified the "Read pending topic1" node to filter by `Status = Pending`. We then updated the Google Sheets nodes downstream to match exactly on the `Topic` so that it only marks the correct row as "Posted" or "Skipped".

## 3. LinkedIn API 400 Bad Request (Binary Upload)
**Problem:** The HTTP Request node uploading the GIF binary threw a 400 error. The `n8n` node wasn't actually sending the binary body because the import corrupted the settings.
**Solution:** Enabled `Send Body` -> `Body Content Type: Binary` -> `Input Data Field Name: gif`. Also ensured the `Content-Type: image/gif` header was manually added.

## 4. LinkedIn API Migration (UGC Posts -> Rest Posts)
**Problem:** LinkedIn heavily restricted their APIs. The legacy `/v2/ugcPosts` and `/v2/assets` APIs were throwing regex errors and `403 Forbidden` errors because they do not fully support modern OpenID Connect alphanumeric IDs (`urn:li:person:OiWoXcSK38`) and deprecated the `digitalmediaAsset` URN for posts.
**Solution:** We completely migrated the workflow to the modern 2026 LinkedIn REST API (`/rest/posts` and `/rest/images`).

### Final Modern API Configuration
To ensure the workflow never breaks due to legacy API deprecation, the nodes were reconfigured as follows:

**1. Initialize Image Upload (formerly Register GIF upload)**
- **Endpoint:** `POST https://api.linkedin.com/rest/images?action=initializeUpload`
- **Headers:** `LinkedIn-Version: 202605` and `X-RestLi-Protocol-Version: 2.0.0`
- **Body:**
  ```json
  {
    "initializeUploadRequest": {
      "owner": "urn:li:person:OiWoXcSK38"
    }
  }
  ```

**2. Prepare LinkedIn upload (Code Node)**
- **Code:**
  ```javascript
  const reg = $input.first().json;
  const original = $('Prepare GIF binary').first();
  const uploadUrl = reg.value?.uploadUrl;
  const asset = reg.value?.image;

  if (!uploadUrl || !asset) throw new Error('LinkedIn initializeUpload failed: ' + JSON.stringify(reg));

  return [{ json: { ...original.json, uploadUrl, asset }, binary: { gif: original.binary.gif } }];
  ```

**3. Publish GIF Post**
- **Endpoint:** `POST https://api.linkedin.com/rest/posts`
- **Headers:** `LinkedIn-Version: 202605` and `X-RestLi-Protocol-Version: 2.0.0`
- **Body:**
  ```javascript
  {{ JSON.stringify({
    author: 'urn:li:person:OiWoXcSK38',
    commentary: $('Prepare LinkedIn upload').first().json.post_text,
    visibility: 'PUBLIC',
    distribution: { feedDistribution: 'MAIN_FEED', targetEntities: [], thirdPartyDistributionChannels: [] },
    content: { media: { title: $('Prepare LinkedIn upload').first().json.topic, id: $('Prepare LinkedIn upload').first().json.asset } },
    lifecycleState: 'PUBLISHED',
    isReshareDisabledByAuthor: false
  }) }}
  ```
