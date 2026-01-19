# Proxy Authentication Implementation Summary

This document summarizes the changes made to the `run-gemini-cli` action to support a new "Authenticate to Proxy" method. This method allows users to authenticate via a custom proxy using a Client ID and Client Secret, specifically tailored for environments like `st` or `dev`.

## Overview

The goal was to enable the Gemini CLI to communicate through a corporate or custom proxy that uses OAuth 2.0 `client_credentials` flow for authentication. This involved adding new configuration inputs, implementing a token retrieval step, and configuring the Gemini CLI environment variables to route requests through the proxy.

## Modified Files

- `action.yml`: Core logic updates.
- `docs/authentication.md`: User documentation for the new method.
- `README.md`: Auto-generated input documentation updates.

## Detailed Changes

### 1. `action.yml`

#### New Inputs

Added the following inputs to support proxy configuration:

- `proxy_environment`: (Optional) A shorthand to automatically configure URLs (e.g., "st", "dev").
- `proxy_url`: (Optional) Explicitly set the Gemini API proxy base URL.
- `proxy_auth_url`: (Optional) Explicitly set the OAuth 2.0 token endpoint.
- `proxy_client_id`: (Required for proxy) The Client ID.
- `proxy_client_secret`: (Required for proxy) The Client Secret.

#### Step: Validate Inputs

Updated the validation logic to:

- Include `proxy_url` in the count of authentication methods (ensuring only one method is used).
- Verify that if `proxy_url` is used, the required companion inputs (`proxy_auth_url`, `proxy_client_id`, `proxy_client_secret`) are also provided.
- _Note: logic was also added to derive URLs from `proxy_environment` if explicit URLs are missing._

#### New Step: Authenticate to Proxy

Added a new step that runs before "Install pnpm":

- **Condition**: Runs if `proxy_url` OR `proxy_environment` is provided.
- **Logic**:
  1.  Determines the `AUTH_URL`. If `proxy_environment` is set (e.g., `st`), it defaults to `https://{env}.api.genpt.com/oauthv2/token`.
  2.  Executes a `curl` command to the auth endpoint with `grant_type=client_credentials`.
  3.  Extracts the `access_token` from the JSON response using `jq`.
  4.  Exports the token as a step output (`access_token`) and masks it in logs.

#### Step: Run Gemini CLI

Updated the environment variable configuration:

- **`GOOGLE_GENAI_USE_VERTEXAI`**: Explicitly set to `'false'` when using proxy to prevent Vertex AI SDK defaults.
- **`GOOGLE_API_KEY`**: Cleared (empty string) when using proxy.
- **`GOOGLE_CLOUD_ACCESS_TOKEN`**: Uses the token from `steps.proxy_auth.outputs.access_token` if available.
- **`GEMINI_BASE_URL`**: Dynamically constructed. If `proxy_environment` is used, it sets the URL to `https://{env}.api.genpt.com/genaisp/gas/proxy/gemini/v1beta1/publishers/google`.
- **`GEMINI_API_VERSION`**: Set to empty string `''` to prevent the SDK from appending default version paths (simulating the monkeypatch behavior).
- **`GEMINI_API_KEY`**: Set to `'DUMMY'` to satisfy SDK validation without sending a real key to the proxy.

### 2. `docs/authentication.md`

Added a new section **Method 4: Authenticating with a Proxy**.

- Lists prerequisites (Proxy URL, Client ID, Client Secret).
- Provides a setup guide for GitHub Secrets and Variables.
- Includes a YAML example showing how to configure the workflow.

### 3. `README.md`

- Regenerated the **Inputs** table using `npm run docs` to include descriptions and default values for the new `proxy_*` inputs.

## Usage Example

```yaml
- uses: 'google-github-actions/run-gemini-cli@v0'
  with:
    # Use the shorthand environment configuration
    proxy_environment: 'st'
    # Credentials stored in secrets
    proxy_client_id: '${{ secrets.GAS_DEV_CLIENT_ID }}'
    proxy_client_secret: '${{ secrets.GAS_DEV_CLIENT_SECRET }}'
    # Specify the model available on the proxy
    gemini_model: 'gemini-2.5-flash'
    prompt: 'Explain this code'
```
