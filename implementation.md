# LibreChat & eConnect Implementation Plan

## 1. Authentication

### Option A: LDAP (PC Credentials - Recommended based on new context)
Since employees log into `econnect` using their PC credentials, this indicates your organization uses an Active Directory/LDAP server. LibreChat can connect directly to this server, allowing users to log in with their exact Windows/PC username and password.

#### `.env` Setup (LDAP):
```env
# Disable registration if everyone uses LDAP
ALLOW_REGISTRATION=false
ALLOW_EMAIL_LOGIN=true

# Your Active Directory connection string
LDAP_URL=ldap://your.domain.controller.local:389
LDAP_BIND_DN=CN=ServiceAccount,OU=Users,DC=yourcompany,DC=com
LDAP_BIND_CREDENTIALS=your_service_account_password
LDAP_USER_SEARCH_BASE=OU=Users,DC=yourcompany,DC=com

# Tell LibreChat users will type their username (not email) to login
LDAP_LOGIN_USES_USERNAME=true

# Map Active Directory fields to LibreChat user profiles
LDAP_USERNAME=sAMAccountName
LDAP_EMAIL=mail
LDAP_FULL_NAME=displayName
```

### Option B: Keycloak OIDC Integration
If you prefer to route the login through the `HireHub` Keycloak server (which might also be connected to your LDAP), you use OpenID Connect.

### Keycloak Setup Requirements:
1.  Access the Keycloak Admin Console (`http://192.168.6.6:8080/auth/admin/master/console/`).
2.  Switch to the `HireHub` realm.
3.  Navigate to **Clients** -> **Create New Client**.
4.  Configure the Client:
    *   **Client ID:** `librechat`
    *   **Client Authentication:** Enabled (This generates the Client Secret).
    *   **Valid Redirect URIs:** `https://your-librechat-url.com/oauth/openid/callback`
5.  Copy the generated **Client ID** and **Client Secret** into your `.env` file.

### `.env` Setup (OIDC):
```env
# Enable OpenID Connect Registration/Login
ALLOW_SOCIAL_LOGIN=true
ALLOW_SOCIAL_REGISTRATION=true

# Keycloak Issuer URL from id_token
OPENID_ISSUER=http://192.168.6.6:8080/auth/realms/HireHub

# Client Credentials (from Keycloak)
OPENID_CLIENT_ID=librechat
OPENID_CLIENT_SECRET=your_new_librechat_client_secret

# Make up a random, strict string to encrypt the session
OPENID_SESSION_SECRET=your_random_secure_string_here

# Request standard claims
OPENID_SCOPE="openid email profile"
OPENID_CALLBACK_URL=/oauth/openid/callback

# Map the data from the token to LibreChat's database
OPENID_NAME_CLAIM=name
OPENID_USERNAME_CLAIM=preferred_username

# The button style users will see on the login page
OPENID_BUTTON_LABEL=Login with econnect
OPENID_IMAGE_URL=https://your_econnect_logo_url.com/logo.png
```

---

## 2. API Model Configuration
LibreChat natively supports Anthropic, Google, and OpenAI licenses. Users can select models via a dropdown UI. 

### `.env` Setup (Licenses):
```env
# Anthropic keys
ANTHROPIC_API_KEY=your_anthropic_api_key_here

# Google keys
GOOGLE_KEY=your_google_api_key_here

# OpenAI keys
OPENAI_API_KEY=your_openai_api_key_here
```

### Handling Multiple Licenses & Fallbacks (AI Gateway)
If you have 3 different Anthropic licenses and want to ensure that if one token expires or hits a rate limit, the system automatically tries the next token (or falls back to OpenAI), **LibreChat cannot do this natively.**

You **must** use an open-source AI Gateway like **LiteLLM**.

#### How it works:
1. You run a small LiteLLM docker container next to LibreChat.
2. Inside LiteLLM's `config.yaml`, you provide all 3 Anthropic keys and set up the fallback rules:
   ```yaml
   model_list:
     - model_name: claude-3-5-sonnet
       litellm_params:
         model: claude-3-5-sonnet-20241022
         api_key: ANTHROPIC_KEY_1
     - model_name: claude-3-5-sonnet
       litellm_params:
         model: claude-3-5-sonnet-20241022
         api_key: ANTHROPIC_KEY_2  # Fallback 1
     - model_name: claude-3-5-sonnet
       litellm_params:
         model: claude-3-5-sonnet-20241022
         api_key: ANTHROPIC_KEY_3  # Fallback 2
   ```
3. In LibreChat's `.env`, you remove the direct `ANTHROPIC_API_KEY` and instead tell LibreChat to talk to your LiteLLM server using a custom endpoint in `librechat.yaml`.

This guarantees that your users never see an "Invalid Token" error as long as at least one of your 3 licenses is valid!

---

## 3. Usage Restrictions & Rate Limiting
To prevent abuse and control costs, LibreChat provides extensive configuration options natively.

### `.env` Setup (Restrictions):
```env
# 1. Prevent concurrent requests (stops window spam)
LIMIT_CONCURRENT_MESSAGES=true
CONCURRENT_MESSAGE_MAX=1

# 2. Limit maximum messages per user
LIMIT_MESSAGE_USER=true
MESSAGE_USER_MAX=40     # Max 40 messages
MESSAGE_USER_WINDOW=1   # Per 1 minute

# 3. Limit maximum messages per IP address
LIMIT_MESSAGE_IP=true
MESSAGE_IP_MAX=100
MESSAGE_IP_WINDOW=1
```

### Strict Single Session (Custom Requirement):
LibreChat currently allows a user to be logged in on multiple devices. If strict "Single Session Only" logic is required, it must be developed custom by modifying `api/server/controllers/AuthController.js` to invalidate previous refresh tokens/sessions whenever a new login occurs.

---

## 4. Metrics & Storage
*   **Conversations:** Automatically stored in MongoDB.
*   **User Tracking:** Token usage and cost tracking are available natively.
*   **Analytics:** Add `ANALYTICS_GTM_ID=your_tracking_id` to `.env` for Google Tag Manager analytics.
*   **Observability (Optional):** Integrate with Langfuse for advanced tracking of prompt quality and model latency by configuring the `LANGFUSE_*` variables in the `.env` file.
