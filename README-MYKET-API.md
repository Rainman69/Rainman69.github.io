# Myket API Documentation

This document contains the Myket API integration code extracted from the [APK-DL](https://github.com/Rainman69/APK-DL) project's `worker.js` file.

## 📋 Table of Contents

- [Configuration](#configuration)
- [Authentication](#authentication)
- [Get App Information](#get-app-information)
- [Get Download URL](#get-download-url)
- [Helper Functions](#helper-functions)
- [Usage Instructions](#usage-instructions)
- [API Endpoints](#api-endpoints)
- [Important Notes](#important-notes)

---

## Configuration

### Myket API Configuration Object

```javascript
const MYKET_API_CONFIG = {
  apiServerUrl: "https://apiserver.myket.ir",
  v1BaseUrl: "https://apiserver.myket.ir/v1/applications",
  v2BaseUrl: "https://apiserver.myket.ir/v2/applications",
  authUrl: "https://apiserver.myket.ir/v1/devices/authorize/",
  headers: {
    "Content-Type": "application/json",
    "Accept": "application/json",
    "Myket-Version": "914",
    "User-Agent": "Dalvik/2.1.0 (Linux; U; Android x.x; xxxx Build/xxxxxx)",
    "Host": "apiserver.myket.ir",
    "Connection": "Keep-Alive",
    "Accept-Encoding": "gzip",
  },
  authBody: {
    acId: "",
    acKey: "",
    adId: "82d8810f-e83c-46c0-8bf5-080a83d635c6",
    andId: "82ee24cd7aad5173",
    api: "29",
    brand: "Nokia",
    cpuAbis: ["armeabi-v7a", "armeabi"],
    dens: 2.625,
    deviceModel: "Nokia 7 plus",
    deviceName: "B2N_sprout",
    deviceType: "normal",
    dsize: "300",
    hsh: "4e9118e410d75e2ef80aac45357493e397037940",
    imei: "",
    imsi: "",
    manufacturer: "HMD Global",
    product: "Onyx_00WW",
    supportedAbis: ["arm64-v8a", "armeabi-v7a", "armeabi"],
    uuid: "235f746f-3a40-44b7-97b4-01cac934df6d",
  }
};
```

---

## Authentication

### Get Myket Authentication Token

This function obtains an authentication token from Myket API and caches it for 1 hour.

```javascript
async function getMyketAuthToken(env) {
  try {
    // Check if we have a cached token
    const cachedToken = await env.KV_BINDING.get('myket_auth_token');
    if (cachedToken) {
      console.log('Using cached Myket auth token');
      return cachedToken;
    }

    console.log('Getting new Myket auth token...');
    const response = await fetch(MYKET_API_CONFIG.authUrl, {
      method: 'POST',
      headers: MYKET_API_CONFIG.headers,
      body: JSON.stringify(MYKET_API_CONFIG.authBody)
    });

    if (!response.ok) {
      const errorData = await response.json().catch(() => ({}));
      const errorMessage = errorData.translatedMessage || `HTTP ${response.status}`;
      throw new Error(`Authentication failed: ${errorMessage}`);
    }

    const data = await response.json();
    const token = data.token;
    
    if (!token) {
      throw new Error('No token received from Myket API');
    }

    // Cache the token for 1 hour
    await env.KV_BINDING.put('myket_auth_token', token, { expirationTtl: 3600 });
    console.log('Successfully obtained and cached Myket auth token');
    
    return token;
  } catch (error) {
    console.error('Error getting Myket auth token:', error);
    throw new Error(`Myket authentication failed: ${error.message}`);
  }
}
```

**Key Features:**
- ✅ Token caching for 1 hour to reduce API calls
- ✅ Automatic error handling and logging
- ✅ Returns authentication token for subsequent API calls

---

## Get App Information

### Retrieve Application Details

This function fetches detailed information about an app from Myket.

```javascript
async function getMyketAppInfo(packageName, authToken, env) {
  try {
    const infoUrl = `${MYKET_API_CONFIG.v2BaseUrl}/${packageName}/`;
    
    const headers = {
      ...MYKET_API_CONFIG.headers,
      'Authorization': authToken
    };

    console.log(`Getting Myket app info for: ${packageName}`);
    const response = await fetch(infoUrl, {
      method: 'GET',
      headers: headers
    });

    if (!response.ok) {
      if (response.status === 401) {
        // Token expired, get new one and retry
        console.log('Myket auth token expired, getting new one...');
        await env.KV_BINDING.delete('myket_auth_token');
        const newToken = await getMyketAuthToken(env);
        
        const retryHeaders = {
          ...MYKET_API_CONFIG.headers,
          'Authorization': newToken
        };
        
        const retryResponse = await fetch(infoUrl, {
          method: 'GET',
          headers: retryHeaders
        });
        
        if (!retryResponse.ok) {
          throw new Error(`Myket API error after retry: HTTP ${retryResponse.status}`);
        }
        
        const retryData = await retryResponse.json();
        console.log('Myket app info (retry) raw response:', JSON.stringify(retryData, null, 2));
        return retryData;
      } else {
        throw new Error(`Myket API error: HTTP ${response.status}`);
      }
    }

    const data = await response.json();
    console.log('Myket app info raw response:', JSON.stringify(data, null, 2));
    console.log('Successfully got Myket app info for:', packageName);
    return data;
  } catch (error) {
    console.error('Error getting Myket app info:', error);
    throw new Error(`Failed to get app info from Myket: ${error.message}`);
  }
}
```

**Response Structure:**
```javascript
{
  metadata: {
    applicationName: "App Name"
  },
  version: {
    code: 123
  },
  price: {
    isFree: true
  },
  size: {
    actual: 1234567  // Size in bytes
  }
}
```

**Key Features:**
- ✅ Automatic token refresh on 401 errors
- ✅ Detailed logging for debugging
- ✅ Returns complete app metadata

---

## Get Download URL

### Obtain Direct Download Link

This function retrieves the direct download URL for an app.

```javascript
async function getMyketAppDownloadUrl(version, packageName, authToken, env) {
  try {
    const params = new URLSearchParams({
      action: 'start',
      requestedVersion: version,
      fileType: 'App',
      lang: 'fa'
    });
    
    const downloadUrl = `${MYKET_API_CONFIG.v1BaseUrl}/${packageName}/uri/?${params.toString()}`;
    
    const headers = {
      ...MYKET_API_CONFIG.headers,
      'Authorization': authToken
    };

    console.log(`Getting Myket download URL for: ${packageName}, version: ${version}`);
    const response = await fetch(downloadUrl, {
      method: 'GET',
      headers: headers
    });

    if (!response.ok) {
      if (response.status === 401) {
        // Token expired, get new one and retry
        console.log('Myket auth token expired during download URL request, getting new one...');
        await env.KV_BINDING.delete('myket_auth_token');
        const newToken = await getMyketAuthToken(env);
        
        const retryHeaders = {
          ...MYKET_API_CONFIG.headers,
          'Authorization': newToken
        };
        
        const retryResponse = await fetch(downloadUrl, {
          method: 'GET',
          headers: retryHeaders
        });
        
        if (!retryResponse.ok) {
          throw new Error(`Myket download API error after retry: HTTP ${retryResponse.status}`);
        }
        
        const retryData = await retryResponse.json();
        return buildMyketDownloadUrl(retryData);
      } else {
        throw new Error(`Myket download API error: HTTP ${response.status}`);
      }
    }

    const data = await response.json();
    return buildMyketDownloadUrl(data);
  } catch (error) {
    console.error('Error getting Myket download URL:', error);
    throw new Error(`Failed to get download URL from Myket: ${error.message}`);
  }
}
```

**Key Features:**
- ✅ Version-specific download URL generation
- ✅ Automatic token refresh
- ✅ Multiple server support

---

## Helper Functions

### Build Download URL

Constructs the final download URL from API response.

```javascript
function buildMyketDownloadUrl(data) {
  const { uriPath, uriServers } = data;
  
  if (!uriPath) {
    throw new Error('No download link available from Myket');
  }
  
  if (!uriServers || uriServers.length === 0) {
    throw new Error('No download servers available from Myket');
  }
  
  // Use a random server from the available servers
  const randomServer = uriServers[Math.floor(Math.random() * uriServers.length)];
  const fullUrl = randomServer + uriPath;
  
  console.log('Built Myket download URL:', fullUrl);
  return fullUrl;
}
```

**Key Features:**
- ✅ Random server selection for load balancing
- ✅ Error validation
- ✅ Complete URL construction

---

## Usage Instructions

### Complete Workflow Example

```javascript
// Step 1: Get authentication token
const authToken = await getMyketAuthToken(env);

// Step 2: Get app information
const appInfo = await getMyketAppInfo('com.example.app', authToken, env);

// Step 3: Verify app is free
if (!appInfo.price.isFree) {
  throw new Error('Only free apps can be downloaded');
}

// Step 4: Extract app details
const appName = appInfo.metadata?.applicationName || 'Unknown App';
const versionCode = appInfo.version?.code;
const fileSize = appInfo.size?.actual; // Size in bytes

// Step 5: Get download URL
const downloadUrl = await getMyketAppDownloadUrl(
  versionCode,
  'com.example.app',
  authToken,
  env
);

// Step 6: Use the download URL
console.log('Download URL:', downloadUrl);
```

### URL Pattern Recognition

The bot recognizes Myket URLs in this format:
```
https://myket.ir/app/[PACKAGE_NAME]
```

**Example:**
```
https://myket.ir/app/com.instagram.android
```

**Regex Pattern:**
```javascript
const myketRegex = /https:\/\/myket\.ir\/app\/([a-zA-Z0-9_.]+)/;
const myketMatch = url.match(myketRegex);
if (myketMatch) {
  const packageName = myketMatch[1];
  // Process Myket download
}
```

---

## API Endpoints

### Base URLs

| Endpoint | URL | Purpose |
|----------|-----|----------|
| **API Server** | `https://apiserver.myket.ir` | Base API server |
| **V1 Applications** | `https://apiserver.myket.ir/v1/applications` | V1 API endpoints |
| **V2 Applications** | `https://apiserver.myket.ir/v2/applications` | V2 API endpoints |
| **Authorization** | `https://apiserver.myket.ir/v1/devices/authorize/` | Authentication endpoint |

### Endpoint Details

#### 1. Authentication
```
POST https://apiserver.myket.ir/v1/devices/authorize/
```
**Request Body:** Device information (see authBody in config)
**Response:** `{ token: "AUTH_TOKEN" }`

#### 2. App Information (V2)
```
GET https://apiserver.myket.ir/v2/applications/{packageName}/
```
**Headers:** Authorization token required
**Response:** Complete app metadata

#### 3. Download URL (V1)
```
GET https://apiserver.myket.ir/v1/applications/{packageName}/uri/
  ?action=start
  &requestedVersion={version}
  &fileType=App
  &lang=fa
```
**Headers:** Authorization token required
**Response:** `{ uriPath: "...", uriServers: [...] }`

---

## Important Notes

### ⚠️ Limitations

- **Free Apps Only**: The implementation only supports downloading free apps from Myket
- **Token Expiration**: Auth tokens expire after 1 hour and are automatically refreshed
- **Rate Limiting**: Be mindful of API rate limits to avoid being blocked

### 🔒 Security Considerations

- **Token Storage**: Tokens are cached in KV storage with automatic expiration
- **Error Handling**: All functions include comprehensive error handling
- **Retry Logic**: Automatic retry on 401 (unauthorized) errors

### 📊 Data Extraction

#### File Size
The code attempts to extract file size from multiple possible locations:
```javascript
// Primary approach
if (appInfo.size?.actual) {
  fileSizeBytes = appInfo.size.actual;
}
// Fallback approaches
else if (appInfo.size?.compressed) {
  fileSizeBytes = appInfo.size.compressed;
}
```

#### App Name
Multiple fallback locations for app name:
```javascript
if (appInfo.metadata?.applicationName) {
  appName = appInfo.metadata.applicationName;
} else if (appInfo.name) {
  appName = appInfo.name;
} else if (appInfo.title) {
  appName = appInfo.title;
}
// ... more fallbacks
```

### 🚀 Performance Tips

1. **Token Caching**: Always check for cached tokens before requesting new ones
2. **Parallel Requests**: If downloading multiple apps, reuse the same token
3. **Error Recovery**: Implement retry logic for network failures
4. **Server Selection**: The random server selection helps distribute load

### 🔧 Debugging

Enable detailed logging:
```javascript
console.log('Myket app info full response:', JSON.stringify(appInfo, null, 2));
```

This will show the complete API response structure for debugging.

---

## 📞 Related Projects

- **[APK-DL Bot](https://github.com/Rainman69/APK-DL)**: Full Telegram bot implementation
- **[CafeBazaar-Downloader](https://github.com/Rainman69/CafeBazaar-Downloader)**: Similar implementation for Cafe Bazaar

## 📝 License

This documentation is part of the APK-DL project and follows the same license.

---

**📚 For more information, visit the [APK-DL repository](https://github.com/Rainman69/APK-DL)**
