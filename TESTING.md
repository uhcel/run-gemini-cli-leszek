# Testing Proxy Authentication Locally

This document outlines the steps to manually test the Gemini CLI with the Proxy Authentication method locally on your machine. This mimics the behavior of the GitHub Action step "Authenticate to Proxy".

## Prerequisites

- **Gemini CLI**: Installed and available in your PATH (`npm install -g @google/gemini-cli`).
- **cURL**: For making HTTP requests.
- **jq**: For parsing JSON output (optional, but recommended).
- **Credentials**: A valid `CLIENT_ID` and `CLIENT_SECRET` for the proxy environment.

## Testing Steps

### 1. Retrieve an Access Token

The GitHub Action uses the client credentials flow to obtain a token. You can simulate this using `curl`.

Replace `YOUR_CLIENT_ID` and `YOUR_CLIENT_SECRET` with your actual credentials.

```bash
# Example for the 'st' environment
export PROXY_ENV="st"
export CLIENT_ID="YOUR_CLIENT_ID"
export CLIENT_SECRET="YOUR_CLIENT_SECRET"

# Request the token
TOKEN_RESPONSE=$(curl -s -X POST "https://${PROXY_ENV}.api.genpt.com/oauthv2/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=${CLIENT_ID}" \
  -d "client_secret=${CLIENT_SECRET}")

# Extract the access token (requires jq)
export ACCESS_TOKEN=$(echo "$TOKEN_RESPONSE" | jq -r '.access_token')

# Verify you got a token
echo "Token: ${ACCESS_TOKEN:0:10}..."
```

### 2. Configure Environment Variables

You need to set specific environment variables to force the Gemini CLI to use the proxy and the token you just retrieved. This overrides the default behavior of the Google GenAI SDK.

```bash
# 1. Inject the Proxy Access Token
export GOOGLE_CLOUD_ACCESS_TOKEN="$ACCESS_TOKEN"

# 2. Set the Base URL to the Proxy Endpoint
# Note: The path must end with /v1beta1/publishers/google to match the expected SDK structure
export GEMINI_BASE_URL="https://${PROXY_ENV}.api.genpt.com/genaisp/gas/proxy/gemini/v1beta1/publishers/google"

# 3. Disable Vertex AI auto-configuration
export GOOGLE_GENAI_USE_VERTEXAI="false"

# 4. Clear/Set Dummy Keys to satisfy SDK validation
# The SDK requires *some* key to be present, even if we use the Access Token.
export GOOGLE_API_KEY=""
export GEMINI_API_KEY="DUMMY"

# 5. Clear API Version (Optional, prevents double versioning in URL if needed)
export GEMINI_API_VERSION=""

# 6. Set Project and Location (Required by some SDK paths, can be arbitrary if proxy ignores them)
export GOOGLE_CLOUD_PROJECT="main-leszekw-argolis"
export GOOGLE_CLOUD_LOCATION="global"
```

### 3. Run the Gemini CLI

Now you can run the CLI. Ensure you use a model name that is supported by your proxy.

```bash
# Example using a model verified to work on the proxy
gemini --yolo --model "gemini-2.5-flash" --prompt "Hello, are you functional?"
```

## Successful Test Run

The following command was successfully executed to verify the implementation:

```bash
gemini --yolo --model "gemini-2.5-flash" "Explain why testing is important in one sentence."
```

**Output:**

> Testing is crucial for ensuring software quality, identifying bugs early, and validating that the software meets its requirements.

## Troubleshooting

- **404 Not Found**: This usually means the `GEMINI_BASE_URL` is incorrect or the `model` you requested does not exist on the proxy backend.
  - _Fix:_ Check the URL structure. Ensure you are using a model name provided by your proxy administrator (e.g., `gemini-2.5-flash` instead of `gemini-1.5-flash`).
- **401 Unauthorized**: The `ACCESS_TOKEN` is missing, invalid, or expired.
  - _Fix:_ Re-run the `curl` command to get a new token.
- **SDK Validation Errors**: Ensure `GEMINI_API_KEY` is set to something (like "DUMMY") and `GOOGLE_GENAI_USE_VERTEXAI` is "false".
