# AI CV Chatbot with Microsoft Azure OpenAI  
Static website (GitHub Pages) + Azure Function backend

This guide reflects the **exact, working setup** discussed in this conversation, aligned with Azure’s current UI (2026), and mirrors the same architecture you previously used with AWS and Google Cloud.

---

## 1) Prerequisites (do once)

You need:

- A GitHub account  
- A Microsoft Azure account with an active subscription  


Prepare two texts (copy–paste later):

### CV Summary
A short, factual summary of:
- professional background  
- work experience  
- skills  
- projects and achievements  

### Assistant Rules (System Message)

Recommended example:

You are an AI assistant for my CV and portfolio.  
Be polite, concise, and factual.  
Answer in French or English depending on the user's language.  
If the question is ambiguous, ask exactly one short clarifying question.  
If the information is not in the CV summary, say you do not know and ask what information is needed.  
Do not invent details.

---

## 2) Create Azure OpenAI (AI model)

### Step 2.1 — Create the Azure OpenAI resource

1. Go to Azure Portal  
   https://portal.azure.com/#home

2. Search for **Azure OpenAI**

3. Click **Create**

4. You will see a choice between:
   - **Foundry (recommended)**  
   - **Azure OpenAI**

   👉 **Choose: Azure OpenAI**

5. Fill in the settings:

   Resource group  
   `chatbot-yourname-resourcegroup`

   Instance name  
   `chatbot-yourname-instance`

   Region  
   `France Central`

   Pricing  
   `Standard S0`

6. Click **Next** until creation (you can skip **Tags**, it is optional)

### Important note (expected behavior)

After creation, the instance page may show a warning like **“Content failed to load”**.  
This is **normal** because **no model is deployed yet**.

To confirm the resource exists:
- Go to **All resources**
- Verify that `chatbot-yourname-instance` appears and is **Running**

---

### Step 2.2 — Deploy a model (VERY IMPORTANT)

1. Once your deployment is complete, click Go to resource 

2. When you are within your resource ' `chatbot-yourname-instance`', got to 'Explore Foundry portal' in the bottom of the page

Alternatively for Step 1 and 2, Open Azure AI Foundry  https://ai.azure.com/ and Select your created instance `chatbot-yourname-instance`

3. In the left menu, click **Model catalog**

4. From the catalog, select:  
   **gpt-4o-mini**

5. Click **Use this model**

6. Deployment settings:
   - select your AI resource
   - Deployment type: **Standard**
   - Deployment name:  
     `gpt-4o-mini-YourName`

8. Click **Deploy**

---

### Step 2.3 — Write down these values (you will need them later)

Use the **exact format below**:

ENDPOINT  
https://chatbot-yourname-instance.cognitiveservices.azure.com/

API KEY  
<your-key-1>

DEPLOYMENT NAME  
gpt-4o-mini

Important:
- The endpoint is the **base URL only**
- Do NOT include `/openai/deployments`
- Do NOT include `/chat/completions`
- Do NOT include `api-version`

---

## 3) Create the Azure Function (backend API)

This function is your backend API, equivalent to AWS Lambda or Google Cloud Function.

### Step 3.1 — Create Function App

1. Go to Azure Portal  
   https://portal.azure.com/#home

2. Search for **Function App**

3. Click **Create**

4. Select hosting option:
   - **Consumption** (serverless)

5. On the next screen:

   Resource group  
    `chatbot-yourname-resourcegroup`

   Function App name  
   `chatbot-yourname-function`

6. Runtime stack:
   - Node.js

7. All other settings:
   - Leave **default**
   - Do **not** configure GitHub deployment
   - Do **not** change authentication
   - Do **not** click through advanced tabs

8. Click **Review + Create**
9. Click **Create**

Wait until the Function App is **Running**

---

## 4) Create the chat function

### Step 4.1 — Create HTTP-triggered function

1. Open **Function App** → `chatbot-yourname-function`
2. Go to **Functions**
3. Click **Create**
4. Choose **Create in Azure portal**
5. Select **HTTP trigger**
6. Configure:

   Authorization level  
   `Anonymous`

   Function name  
   `chat`

7. Click **Create**

Your backend endpoint will be:

https://chatbot-yourname-function.azurewebsites.net/api/chat

---

### Step 4.2 — Paste backend code (copy exactly)

Go to:

Function App → Functions → chat → Code + Test  
Replace **everything** in `index.js` with the code below & your github URL for  "Access-Control-Allow-Origin": "https://yourname.github.io",  

'''
// api/chat/index.js (CommonJS)
// Returns PLAIN TEXT (no JSON braces) so your UI displays cleanly.

module.exports = async function (context, req) {
  // --- CORS (GitHub Pages) ---
  const corsHeaders = {
    "Access-Control-Allow-Origin": "https://juliabaucher.github.io",  
    "Access-Control-Allow-Methods": "POST, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type",
    "Access-Control-Max-Age": "86400"
  };

  // Preflight
  if (req.method === "OPTIONS") {
    context.res = { status: 204, headers: corsHeaders };
    return;
  }

  try {
    // --- Validate input ---
    const userMessage =
      (req.body && req.body.message) ? String(req.body.message).trim() : "";

    if (!userMessage) {
      context.res = {
        status: 400,
        headers: { ...corsHeaders, "Content-Type": "text/plain; charset=utf-8" },
        body: "Missing 'message' in request body."
      };
      return;
    }

    // --- Env vars ---
    const endpoint = process.env.AZURE_OPENAI_ENDPOINT;      // your endpoint name
    const apiKey = process.env.AZURE_OPENAI_API_KEY;
    const deployment = process.env.AZURE_OPENAI_DEPLOYMENT;  // your deployment name
    const system = process.env.SYSTEM_MESSAGE || "You are a helpful CV assistant.";

    if (!endpoint || !apiKey || !deployment) {
      context.res = {
        status: 500,
        headers: { ...corsHeaders, "Content-Type": "text/plain; charset=utf-8" },
        body: "Server misconfigured. Missing AZURE_OPENAI_ENDPOINT, AZURE_OPENAI_API_KEY, or AZURE_OPENAI_DEPLOYMENT."
      };
      return;
    }

    // API version for chat completions
    const apiVersion = process.env.AZURE_OPENAI_API_VERSION || "2024-06-01";

    // IMPORTANT: endpoint must be base only (no /openai/...)
    const base = endpoint.replace(/\/+$/, "");
    const url = `${base}/openai/deployments/${encodeURIComponent(deployment)}/chat/completions?api-version=${encodeURIComponent(apiVersion)}`;

    // --- Call Azure OpenAI ---
    const resp = await fetch(url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "api-key": apiKey
      },
      body: JSON.stringify({
        temperature: 0.2,
        max_tokens: 400,
        messages: [
          { role: "system", content: system },
          { role: "user", content: userMessage }
        ]
      })
    });

    const raw = await resp.text();

    if (!resp.ok) {
      // Return upstream error as plain text (easy to debug)
      context.res = {
        status: 500,
        headers: { ...corsHeaders, "Content-Type": "text/plain; charset=utf-8" },
        body: `Azure OpenAI error ${resp.status}: ${raw}`
      };
      return;
    }

    let data;
    try {
      data = JSON.parse(raw);
    } catch (e) {
      context.res = {
        status: 500,
        headers: { ...corsHeaders, "Content-Type": "text/plain; charset=utf-8" },
        body: "Azure OpenAI returned a non-JSON response."
      };
      return;
    }

    const reply = (data?.choices?.[0]?.message?.content || "").trim();
    const safeReply = reply || "Sorry — I couldn't generate an answer.";

    // ✅ Return PLAIN TEXT (no JSON braces)
    context.res = {
      status: 200,
      headers: { ...corsHeaders, "Content-Type": "text/plain; charset=utf-8" },
      body: safeReply
    };
  } catch (err) {
    context.log("Azure Function error:", err);
    context.res = {
      status: 500,
      headers: { ...corsHeaders, "Content-Type": "text/plain; charset=utf-8" },
      body: err?.message || String(err)
    };
  }
};
'''

Click **Save**  
Save = Deploy (no package.json required)

---

## 5) Add environment variables (VERY IMPORTANT)

Secrets live here. Never put them in HTML.

Go to:

Function App → Settings → Environment variables → App settings

Add:

Name  
AZURE_OPENAI_ENDPOINT  
Value  
https://chatbot-yourname-instance.cognitiveservices.azure.com/

Name  
AZURE_OPENAI_API_KEY  
Value  
<your-key-1>

Name  
AZURE_OPENAI_DEPLOYMENT  
Value  
gpt-4o-mini

Name  
SYSTEM_MESSAGE  
Value  
(Assistant rules + CV summary)

Notes:
- Do NOT check **Deployment slot setting**
- Names are case-sensitive

Click **Save**  
Restart the Function App

---

## 6) Enable CORS (critical step)

Because GitHub Pages ≠ Azure domain.

Go to:

Function App → API → CORS

Add allowed origin:

https://yourname.github.io

Do NOT:
- Enable Access-Control-Allow-Credentials
- Use *

Click **Save**

---

## 7) Get your API URL

In your function:

Functions → chat → Get Function URL

Copy:

https://chatbot-yourname-function.azurewebsites.net/api/chat

This is your permanent backend endpoint.

---

## 8) Insert your API URL in the index.html 

'''


<script>  
const API_URL = "YOUR API URL"; // const API_URL = "https://chatbot-yourname-function.azurewebsites.net/api/chat";  
</script>  

'''

---

## 9) Publish on GitHub Pages

1. Create a GitHub repository
2. Upload **only** `index.html`
3. Go to **Settings → Pages**
4. Enable GitHub Pages
5. Open the public URL

---

## 10) What this setup gives you

- Same pattern as AWS / Google Cloud
- Static frontend only
- Azure-hosted backend
- Secrets securely stored
- No Copilot Studio
- No SDK or package.json
- Save = deploy
- Production-ready architecture

🎉 Done
# MicrosoftChatbot_YourName
