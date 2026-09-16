# Kaggle Ollama → Cloudflare Tunnel → VS Code AI Agent

This guide sets up an Ollama model on a Kaggle GPU notebook and exposes
its OpenAI-compatible API through a temporary Cloudflare Quick Tunnel so
an external AI client, such as a VS Code extension that supports custom
OpenAI-compatible endpoints, can use the model.
------------------------------------------------------------------------

## Architecture

``` text
VS Code AI Extension / External Agent
                |
                | OpenAI-compatible HTTPS API
                v
        Cloudflare Quick Tunnel
                |
                v
          Kaggle Notebook
                |
                v
             Ollama
                |
                v
           qwen3:8b
                |
                v
           Kaggle GPU
```

------------------------------------------------------------------------

# 1. Enable the Kaggle GPU

In your Kaggle Notebook:

**Settings → Accelerator → GPU**

Then verify the GPU:

``` bash
!nvidia-smi
```

------------------------------------------------------------------------

# 2. Install Ollama

Run:

``` bash
!curl -fsSL https://ollama.com/install.sh | sh
```

Verify:

``` bash
!ollama --version
```

------------------------------------------------------------------------

# 3. Start Ollama

Kaggle/Jupyter does not reliably support shell background processes
using `&`, so start Ollama with Python:

``` python
import subprocess
import time

ollama_process = subprocess.Popen(
    ["ollama", "serve"],
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True
)

time.sleep(3)

print("Ollama started")
```

Verify Ollama:

``` python
import requests

r = requests.get("http://127.0.0.1:11434")
print(r.status_code)
print(r.text)
```

Expected:

``` text
200
Ollama is running
```

------------------------------------------------------------------------

# 4. Download an Ollama model

Example:

``` bash
!ollama pull qwen3:8b
```

Check installed models:

``` bash
!ollama list
```

You should see:

``` text
qwen3:8b
```

You can use another Ollama model if it is installed. The model name in
your external client must exactly match the name shown by:

``` bash
!ollama list
```

------------------------------------------------------------------------

# 5. Verify Ollama's OpenAI-compatible API

Ollama exposes an OpenAI-compatible API under `/v1`.

Test the models endpoint:

``` python
import requests

r = requests.get("http://127.0.0.1:11434/v1/models")

print("Status:", r.status_code)
print(r.text)
```

Expected status:

``` text
200
```

and the response should contain your model, for example:

``` json
{
  "object": "list",
  "data": [
    {
      "id": "qwen3:8b"
    }
  ]
}
```

------------------------------------------------------------------------

# 6. Test actual inference locally

Run:

``` python
import requests

r = requests.post(
    "http://127.0.0.1:11434/v1/chat/completions",
    json={
        "model": "qwen3:8b",
        "messages": [
            {
                "role": "user",
                "content": "Say hello from the Kaggle GPU in one sentence."
            }
        ]
    },
    timeout=120
)

print("Status:", r.status_code)
print(r.text)
```

Do not continue to Cloudflare until this returns successfully.

------------------------------------------------------------------------

# 7. Install cloudflared

Run:

``` bash
!wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
!dpkg -i cloudflared-linux-amd64.deb
```

Verify:

``` bash
!cloudflared --version
```

------------------------------------------------------------------------

# 8. Create the Cloudflare Quick Tunnel

For Kaggle, use this command:

``` bash
!cloudflared tunnel --url http://127.0.0.1:11434 --http-host-header localhost:11434
```

The command must remain running.

Cloudflare will print a URL similar to:

``` text
Your quick Tunnel has been created! Visit it at:

https://example-name.trycloudflare.com
```

Copy that URL.

### Do not stop the cell

The Quick Tunnel exists only while `cloudflared` is running. If the
Kaggle session ends, the tunnel ends.

------------------------------------------------------------------------

# 9. Determine your public OpenAI-compatible endpoint

If Cloudflare gives:

``` text
https://example-name.trycloudflare.com
```

your OpenAI-compatible base URL is:

``` text
https://example-name.trycloudflare.com/v1
```

Do not add `/chat/completions` to the Base URL in your VS Code
extension. The client normally adds that path itself.

------------------------------------------------------------------------

# 10. Test the public endpoint

Open a second Kaggle cell while the cloudflared cell is still running:

``` python
import requests

PUBLIC_URL = "https://YOUR-TUNNEL.trycloudflare.com"

r = requests.get(PUBLIC_URL + "/v1/models")

print("Status:", r.status_code)
print(r.text)
```

Replace:

``` text
YOUR-TUNNEL
```

with the hostname Cloudflare actually gave you.

Expected:

``` text
Status: 200
```

and the response should contain:

``` text
qwen3:8b
```

------------------------------------------------------------------------

# 11. Test public inference

``` python
import requests

PUBLIC_URL = "https://YOUR-TUNNEL.trycloudflare.com"

r = requests.post(
    PUBLIC_URL + "/v1/chat/completions",
    json={
        "model": "qwen3:8b",
        "messages": [
            {
                "role": "user",
                "content": "Hello. Confirm that you are running through the Cloudflare tunnel."
            }
        ]
    },
    timeout=120
)

print("Status:", r.status_code)
print(r.text)
```

You want:

``` text
Status: 200
```

with a model response.

------------------------------------------------------------------------

# 12. Connect a VS Code AI extension

The exact settings depend on the extension.

If the extension supports an **OpenAI-compatible provider/custom OpenAI
endpoint**, use:

``` text
Provider:
OpenAI Compatible

Base URL:
https://YOUR-TUNNEL.trycloudflare.com/v1

API Key:
ollama

Model:
qwen3:8b
```

### Example

If your Cloudflare URL is:

``` text
https://example-name.trycloudflare.com
```

configure:

``` text
Base URL:
https://example-name.trycloudflare.com/v1

API Key:
ollama

Model:
qwen3:8b
```

The API key value is commonly accepted as a placeholder for a local
Ollama endpoint; Ollama itself does not require an OpenAI API key for
its local API.

------------------------------------------------------------------------

# 13. OpenAI Python client test from your PC/VPS

Install the OpenAI Python package:

``` bash
pip install openai
```

Then:

``` python
from openai import OpenAI

client = OpenAI(
    base_url="https://YOUR-TUNNEL.trycloudflare.com/v1",
    api_key="ollama"
)

response = client.chat.completions.create(
    model="qwen3:8b",
    messages=[
        {
            "role": "user",
            "content": "Hello from my external computer."
        }
    ]
)

print(response.choices[0].message.content)
```

------------------------------------------------------------------------

# 14. Using it with an AI agent

An agent that supports OpenAI-compatible models can generally be
configured with:

``` text
Base URL:
https://YOUR-TUNNEL.trycloudflare.com/v1

Model:
qwen3:8b

API Key:
ollama
```

The agent can then send normal chat-completion requests to Ollama.

If your agent uses tool/function calling, verify that the particular
Ollama model you selected supports the tool-calling behavior required by
your agent.

------------------------------------------------------------------------

# 15. Important: browser 403 does not prove the API is broken

Do not rely only on opening:

``` text
https://YOUR-TUNNEL.trycloudflare.com/
```

in a browser.

The endpoint you actually care about is:

``` text
https://YOUR-TUNNEL.trycloudflare.com/v1/models
```

and then:

``` text
https://YOUR-TUNNEL.trycloudflare.com/v1/chat/completions
```

Always test those endpoints with an HTTP client.

If `/v1/models` returns `403`, the public tunnel is not successfully
forwarding the request and you should troubleshoot the tunnel before
configuring your VS Code extension.

------------------------------------------------------------------------

# 16. Troubleshooting

## Ollama returns connection refused

Check whether Ollama is running:

``` python
import requests

print(requests.get("http://127.0.0.1:11434").status_code)
```

If it fails, restart Ollama:

``` python
import subprocess
import time

ollama_process = subprocess.Popen(
    ["ollama", "serve"],
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    text=True
)

time.sleep(3)
```

------------------------------------------------------------------------

## Model not found

Check:

``` bash
!ollama list
```

Use the exact model ID.

For example:

``` text
qwen3:8b
```

not:

``` text
qwen3
```

unless that is actually what `ollama list` reports.

------------------------------------------------------------------------

## Cloudflare URL gives 403

First verify Ollama locally:

``` python
import requests

for path in ["/", "/api/tags", "/v1/models"]:
    r = requests.get("http://127.0.0.1:11434" + path)
    print(path, r.status_code, r.text[:300])
```

All required local endpoints should work before debugging Cloudflare.

Then make sure `cloudflared` is still running and that you are using the
newest URL printed by the currently running tunnel.

Restart the Quick Tunnel:

``` bash
!cloudflared tunnel --url http://127.0.0.1:11434 --http-host-header localhost:11434
```

Then test:

``` python
import requests

PUBLIC_URL = "https://YOUR-TUNNEL.trycloudflare.com"

r = requests.get(PUBLIC_URL + "/v1/models")
print(r.status_code)
print(r.text)
```

------------------------------------------------------------------------

# 17. Security warning

A Cloudflare Quick Tunnel URL should be treated as a public endpoint.

Do **not** expose sensitive services through it.

Do not put:

-   passwords
-   private API keys
-   personal files
-   private databases
-   confidential prompts/data

behind an unauthenticated public tunnel.

Also remember that a Quick Tunnel is temporary and does not provide the
reliability or access controls expected from a production API.

For a persistent service, use a Cloudflare-managed/named tunnel and add
authentication/access controls.

------------------------------------------------------------------------

# 18. Final working configuration

Your completed test setup should look like:

``` text
Kaggle GPU
    |
    +-- Ollama
    |     |
    |     +-- qwen3:8b
    |
    +-- 127.0.0.1:11434
              |
              v
       cloudflared Quick Tunnel
              |
              v
https://YOUR-TUNNEL.trycloudflare.com
              |
              v
https://YOUR-TUNNEL.trycloudflare.com/v1
              |
              +---- VS Code extension
              |
              +---- External AI agent
              |
              +---- Python/OpenAI client
```

### VS Code values

``` text
Provider: OpenAI Compatible
Base URL: https://YOUR-TUNNEL.trycloudflare.com/v1
API Key:  ollama
Model:    qwen3:8b
```

### Most important rule

Keep both of these running:

``` text
Ollama
+
cloudflared
```

If either process stops, your external AI client cannot reach the model.
