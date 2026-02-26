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

### Environment Variable Separation (eConnect vs. LibreChat)
It is crucial to understand that you do **not** copy the `.env` values from `econnect` into LibreChat. They are two completely separate applications with their own databases, ports, and internal configurations.

*   **eConnect's `.env`:** Contains its own database connection (e.g., PostgreSQL/MySQL) and its own unique Keycloak Client ID (e.g., `econnect-web`) and Client Secret.
*   **LibreChat's `.env`:** Contains the MongoDB connection (`MONGO_URI`), its own internal secrets (`JWT_SECRET`), and a **brand new**, uniquely generated Keycloak Client ID (`librechat`) and Client Secret.

The **ONLY** variable the two `.env` files will share in common is the URL of the Keycloak server:
`OPENID_ISSUER=http://192.168.6.6:8080/auth/realms/HireHub`

Because both applications point to the exact same `OPENID_ISSUER`, Keycloak acts as the central bridge. When a user logs into `econnect`, Keycloak sets a session cookie in their browser. When they open LibreChat and click "Login," Keycloak sees the cookie, remembers them from `econnect`, and instantly generates a login token for LibreChat without asking for a password again.

### How LibreChat Extracts User Data (The OIDC Handshake)
When an employee clicks "Login with econnect", the following invisible handshake occurs:
1. **The Redirect:** LibreChat redirects the user's browser back to the Keycloak server (`http://192.168.6.6:8080`).
2. **The Login Recognition:** Keycloak sees that the user is already authenticated via their PC (`econnect` session) and instantly approves the request without prompting for a password.
3. **The Token Handoff:** Keycloak redirects the user back to LibreChat (`/oauth/openid/callback`) and provides an encrypted JWT `id_token`.
4. **Data Extraction:** LibreChat decrypts the token. Based on the `econnect` token payload provided, it sees:
   ```json
   {
     "preferred_username": "manoj mohapatra",
     "given_name": "Manoj",
     "family_name": "Mohapatra",
     "email": "manojm@esspl.com",
     "name": "Manoj Mohapatra"
   }
   ```
By setting `OPENID_NAME_CLAIM=name` and `OPENID_USERNAME_CLAIM=preferred_username` in the `.env` file, LibreChat knows exactly which JSON keys to look at. It extracts the name `"Manoj Mohapatra"` and email `"manojm@esspl.com"`, and seamlessly creates (or logs into) the exact matching account in the MongoDB database. No custom backend code is required!

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

### Adding Groq Support (Custom Endpoint)
Groq delivers incredibly fast inference using Llama and Mixtral models. Because it uses the standard OpenAI API format, you do not need to hack the core code to add it; you simply add it as a "Custom Endpoint" in your `librechat.yaml` file.

#### Step 1: Add the API Key to `.env`
First, add your Groq API key to your `.env` file (you can name the variable whatever you want, but sticking to standard naming is good practice):
```env
GROQ_API_KEY=gsk_your_groq_api_key_here
```

#### Step 2: Configure `librechat.yaml`
Next, add Groq to the `custom` endpoints array in your `librechat.yaml` file. This tells LibreChat where to send the requests and which models to populate in the dropdown.

```yaml
endpoints:
  custom:
    - name: "Groq"
      apiKey: "${GROQ_API_KEY}"
      baseURL: "https://api.groq.com/openai/v1"
      models:
        default: ["llama3-8b-8192", "llama3-70b-8192", "mixtral-8x7b-32768", "gemma-7b-it"]
        fetch: false # Set to true if you want LibreChat to fetch the latest models from Groq automatically
      titleConvo: true
      titleModel: "llama3-8b-8192"
      summarize: false
      summaryModel: "llama3-8b-8192"
      forcePrompt: false
      modelDisplayLabel: "Groq"
```
Once you restart your docker container, "Groq" will appear in the main dropdown menu alongside Anthropic and Google!

### Restricting Specific Models to Specific Users (RBAC)
If you want to allow managers to use expensive models (like `claude-3-opus`) but restrict interns to cheaper models (like `llama3-8b` or `gpt-4o-mini`), you do **not** configure this in the `.env` or `yaml` files. LibreChat has a built-in **Admin Dashboard** that handles this dynamically via Role-Based Access Control (RBAC).

#### How to configure user-specific model access:
1. **The Admin Account:** The very first user to log in to your LibreChat instance is automatically designated as the `ADMIN`. (If you need to manually make someone an admin later, you can do so in the MongoDB database).
2. **Access the Panel:** Log in as the Admin, click on your profile name in the bottom left, and open the **Admin Panel**.
3. **Create Roles:** Navigate to the **Roles & Permissions** tab. Create custom roles (e.g., "Interns", "Marketing", "Engineering").
4. **Configure Endpoint/Model Permissions:** Inside the settings for each Role, you will see a section for **Model Access**. Here, you can uncheck or explicitly deny access to specific endpoints (like "Anthropic") or specific underlying models.
5. **Assign Users:** When a new user (like Manoj) logs in through `econnect`, they are assigned the `USER` role by default. The Admin can click on the **Users** tab in the dashboard and change Manoj's role to "Engineering". 

When Manoj reloads the page, his "Model Selection" dropdown will magically shrink to only show the models you permitted his role to see!

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
