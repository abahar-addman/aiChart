# Azure AD Single Sign-On Setup Guide

This guide explains how to configure Azure AD SSO authentication for ChartBrew.

## Overview

ChartBrew now supports Azure AD Single Sign-On (SSO) as an alternative authentication method alongside the existing email/password login. This implementation:

- **Fully backward compatible** - Works exactly as before if Azure is not configured
- Maintains compatibility with all existing users
- Supports hybrid authentication (users can have both password and Azure login)
- Allows automatic account linking via email matching
- Uses OAuth 2.0 Authorization Code flow for secure authentication

## Backward Compatibility

**Important**: Azure SSO is completely optional. If you don't configure Azure AD credentials:

✅ ChartBrew works exactly as it did before
✅ Email/password authentication continues to work normally
✅ All existing users can log in without any changes
✅ New users can be created with email/password
✅ No Azure-related UI elements will appear
✅ No Azure fields are validated or required

The system automatically detects whether Azure is configured and adjusts accordingly. You only need to follow this guide if you want to enable Azure SSO.

## Prerequisites

Before setting up Azure SSO, you need:

1. An Azure AD tenant
2. Administrative access to Azure AD portal
3. ChartBrew server with database access

## Azure AD Portal Configuration

### Step 1: Create App Registration

1. Sign in to the [Azure Portal](https://portal.azure.com)
2. Navigate to **Azure Active Directory** → **App registrations**
3. Click **New registration**
4. Configure the app:
   - **Name**: ChartBrew (or your preferred name)
   - **Supported account types**: Select "Accounts in this organizational directory only (Single tenant)"
   - **Redirect URI**:
     - Platform: Web
     - URI: `http://localhost:4019/api/azure/auth/callback` (development)
     - For production, use your production server URL: `https://your-domain.com/api/azure/auth/callback`
5. Click **Register**

### Step 2: Configure Authentication

1. In your app registration, go to **Authentication**
2. Under **Implicit grant and hybrid flows**, enable:
   - ✅ ID tokens (used for implicit and hybrid flows)
   - ✅ Access tokens (used for implicit flows)
3. Click **Save**

### Step 3: Generate Client Secret

1. Go to **Certificates & secrets**
2. Click **New client secret**
3. Add a description (e.g., "ChartBrew Production")
4. Select an expiration period (recommended: 24 months)
5. Click **Add**
6. **IMPORTANT**: Copy the secret value immediately - it won't be shown again!

### Step 4: Configure API Permissions

1. Go to **API permissions**
2. Click **Add a permission**
3. Select **Microsoft Graph**
4. Select **Delegated permissions**
5. Add the following permissions:
   - `openid`
   - `profile`
   - `email`
   - `User.Read`
6. Click **Add permissions**
7. (Optional) Click **Grant admin consent** if you want to pre-consent for all users

### Step 5: Collect Configuration Values

From your app registration overview page, collect:

- **Application (client) ID**: Found on the Overview page
- **Directory (tenant) ID**: Found on the Overview page
- **Client secret**: The value you copied in Step 3

## Server Configuration

### Environment Variables

Add the following variables to your `.env` file:

```bash
# Azure AD Configuration
AZURE_AD_CLIENT_ID=<your-application-client-id>
AZURE_AD_CLIENT_SECRET=<your-client-secret>
AZURE_AD_TENANT_ID=<your-tenant-id>
AZURE_AD_AUTHORITY=https://login.microsoftonline.com/<your-tenant-id>
AZURE_AD_REDIRECT_URI=http://localhost:4019/api/azure/auth/callback
```

**Production Example:**
```bash
AZURE_AD_CLIENT_ID=12345678-1234-1234-1234-123456789012
AZURE_AD_CLIENT_SECRET=your-secret-value-here
AZURE_AD_TENANT_ID=87654321-4321-4321-4321-210987654321
AZURE_AD_AUTHORITY=https://login.microsoftonline.com/87654321-4321-4321-4321-210987654321
AZURE_AD_REDIRECT_URI=https://api.yourdomain.com/api/azure/auth/callback
```

### Database Migration

Run the database migration to add Azure authentication fields:

```bash
cd server
npm run db:migrate
```

This will add the following fields to the User table:
- `azureId` - Stores the Azure AD object ID
- `authProvider` - Enum: 'local', 'azure', 'hybrid'
- `azureLinkedAt` - Timestamp of account linking

### Install Dependencies

The required NPM packages have been added. Install them:

```bash
# Server dependencies
cd server
npm install

# Client dependencies
cd ../client
npm install
```

## Client Configuration

### Environment Variables

Add the following variables to `client/.env`:

```bash
# Azure AD Configuration (Client)
VITE_AZURE_AD_CLIENT_ID=<your-application-client-id>
VITE_AZURE_AD_TENANT_ID=<your-tenant-id>
VITE_AZURE_AD_AUTHORITY=https://login.microsoftonline.com/<your-tenant-id>
VITE_AZURE_AD_REDIRECT_URI=http://localhost:4018/azure-callback
```

**Production Example:**
```bash
VITE_AZURE_AD_CLIENT_ID=12345678-1234-1234-1234-123456789012
VITE_AZURE_AD_TENANT_ID=87654321-4321-4321-4321-210987654321
VITE_AZURE_AD_AUTHORITY=https://login.microsoftonline.com/87654321-4321-4321-4321-210987654321
VITE_AZURE_AD_REDIRECT_URI=https://yourdomain.com/azure-callback
```

**Note**: The client redirect URI must match your frontend URL, not the API URL.

## Testing the Integration

### 1. Start the Application

```bash
# Terminal 1: Start server
cd server
npm run start-dev

# Terminal 2: Start client
cd client
npm start
```

### 2. Test New User Flow

1. Navigate to the login page
2. Click **"Sign in with Microsoft"**
3. Sign in with your Azure AD account
4. Verify you're redirected back and logged in
5. Check the database - a new user should be created with:
   - `authProvider: 'azure'`
   - `azureId: '<your-azure-oid>'`
   - `password: null`
   - `active: true`

### 3. Test Account Linking Flow

1. Create a user with email/password (e.g., `test@yourdomain.com`)
2. Log out
3. Click **"Sign in with Microsoft"**
4. Sign in with Azure AD using the same email (`test@yourdomain.com`)
5. Verify account is linked:
   - User should be logged in
   - Check database - user should now have:
     - `authProvider: 'hybrid'`
     - `azureId: '<your-azure-oid>'`
     - `password: <existing-hash>` (preserved)
     - `azureLinkedAt: <timestamp>`

### 4. Test Hybrid Login

For a linked user:
1. Log out
2. Test login with password - should work
3. Log out
4. Test login with Microsoft - should work

### 5. Test Azure-Only User Login Attempt

For an Azure-only user:
1. Try to log in with email/password
2. Should see error: "This account uses Microsoft login. Please use the 'Sign in with Microsoft' button."

## Security Considerations

### Authentication Flow

The implementation uses the **OAuth 2.0 Authorization Code Flow**:

1. User clicks "Sign in with Microsoft"
2. Frontend requests auth URL from backend (`GET /api/azure/auth`)
3. User is redirected to Azure AD login
4. After authentication, Azure redirects to backend callback (`GET /api/azure/auth/callback`)
5. Backend exchanges authorization code for tokens (server-side only)
6. Backend creates/links user account
7. Backend generates JWT token
8. User is redirected to frontend with JWT token
9. Frontend saves token and fetches user data

**Security Benefits:**
- Client secret never exposed to browser
- Authorization code can only be used once
- Tokens are exchanged server-side
- State parameter prevents CSRF attacks

### Rate Limiting

All Azure endpoints are rate-limited:
- Auth endpoints: 10 requests per minute
- Link/unlink endpoints: 5 requests per minute

### Account Security

- **Email matching**: Azure email must match existing user email for account linking
- **Duplicate prevention**: Azure ID can only be linked to one account
- **Password protection**: Hybrid users must verify password before unlinking Azure account
- **Azure-only protection**: Azure-only users cannot unlink (would lose access)

## Troubleshooting

### "Azure AD authentication is not configured"

**Cause**: Environment variables are missing or invalid

**Solution**:
1. Verify all `AZURE_AD_*` variables are set in `.env`
2. Restart the server after adding variables
3. Check server logs for initialization errors

### "No email returned from Azure AD"

**Cause**: Azure AD profile doesn't have an email address or app lacks permissions

**Solution**:
1. Verify the user has an email in Azure AD
2. Check API permissions include `email` and `profile` scopes
3. Grant admin consent for permissions in Azure portal

### "This Azure account is already linked to another user"

**Cause**: Attempting to link an Azure account that's already associated with a different user

**Solution**:
1. Sign in with the originally linked account
2. Unlink the Azure account (if it's a hybrid account)
3. Then link to the new account

### Redirect URI Mismatch

**Cause**: The redirect URI in Azure AD doesn't match the configuration

**Solution**:
1. In Azure portal, go to your app's Authentication settings
2. Verify the redirect URI exactly matches your `AZURE_AD_REDIRECT_URI`
3. Include protocol (`http://` or `https://`)
4. Don't include trailing slashes
5. For development: `http://localhost:4019/api/azure/auth/callback`
6. For production: `https://your-domain.com/api/azure/auth/callback`

### Button Not Appearing

**Cause**: Client-side Azure configuration is missing

**Solution**:
1. Verify all `VITE_AZURE_AD_*` variables are set in `client/.env`
2. Restart the client development server
3. Clear browser cache
4. Check browser console for configuration errors

## Disabling Azure SSO

If you need to temporarily disable Azure SSO:

1. Remove or comment out Azure environment variables
2. Restart the application
3. The "Sign in with Microsoft" button will not appear
4. Existing Azure-linked users can still log in with passwords (if they have hybrid auth)
5. Azure-only users will need to use password reset or contact an administrator

## API Reference

### GET /api/azure/auth
Initiates Azure AD login flow.

**Response:**
```json
{
  "authUrl": "https://login.microsoftonline.com/..."
}
```

### GET /api/azure/auth/callback
Handles OAuth callback from Azure AD.

**Query Parameters:**
- `code` - Authorization code
- `state` - CSRF protection token

**Redirects to:** `/azure-callback?token=<jwt>&new=<boolean>&linked=<boolean>`

### POST /api/azure/link
Links Azure account to logged-in user.

**Headers:**
- `Authorization: Bearer <jwt>`

**Body:**
```json
{
  "code": "authorization_code"
}
```

**Response:**
```json
{
  "id": 123,
  "email": "user@example.com",
  "authProvider": "hybrid",
  "azureLinkedAt": "2026-02-08T10:30:00.000Z"
}
```

### DELETE /api/azure/unlink
Unlinks Azure account from logged-in user.

**Headers:**
- `Authorization: Bearer <jwt>`

**Body:**
```json
{
  "password": "user_password"
}
```

**Response:**
```json
{
  "id": 123,
  "email": "user@example.com",
  "authProvider": "local",
  "azureLinkedAt": null
}
```

## Support

For issues or questions:
1. Check the [ChartBrew documentation](https://docs.chartbrew.com)
2. Open an issue on [GitHub](https://github.com/chartbrew/chartbrew)
3. Contact your Azure AD administrator for tenant-specific issues
