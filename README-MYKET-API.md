# How to Use Myket API
---

## Step 1: Obtain an Authentication Token

Before making requests to Myket API endpoints, you must obtain an authentication token.

```js
const MYKET_API_CONFIG = {
  authUrl: "https://apiserver.myket.ir/v1/devices/authorize/",
  headers: {
    "Content-Type": "application/json",
    "Accept": "application/json"
  },
  authBody: {
    // Use your Android device info here
    api: "29",
    brand: "Nokia",
    deviceModel: "Nokia 7 plus",
    uuid: "your-unique-id"
    // ... Add other device details
  }
};

async function getMyketAuthToken() {
  const response = await fetch(MYKET_API_CONFIG.authUrl, {
    method: 'POST',
    headers: MYKET_API_CONFIG.headers,
    body: JSON.stringify(MYKET_API_CONFIG.authBody)
  });
  const data = await response.json();
  return data.token;
}
```

---

## Step 2: Get Application Information

After obtaining a token, use it as an Authorization header to query app details.

```js
async function getMyketAppInfo(packageName, authToken) {
  const infoUrl = `https://apiserver.myket.ir/v2/applications/${packageName}/`;
  const response = await fetch(infoUrl, {
    method: 'GET',
    headers: {
      "Authorization": authToken,
      "Accept": "application/json"
    }
  });
  return await response.json();
}
```

---

## Step 3: Get APK Download Link

With the app version from Step 2, request the APK URI.

```js
async function getMyketAppDownloadUrl(packageName, versionCode, authToken) {
  const params = new URLSearchParams({
    action: 'start',
    requestedVersion: versionCode,
    fileType: 'App',
    lang: 'fa'
  });
  const url = `https://apiserver.myket.ir/v1/applications/${packageName}/uri/?${params.toString()}`;
  const response = await fetch(url, {
    method: 'GET',
    headers: { "Authorization": authToken }
  });
  const data = await response.json();
  // Compose full download link
  return data.uriServers[0] + data.uriPath;
}
```

---

## Workflow Example

```js
const token = await getMyketAuthToken();
const appInfo = await getMyketAppInfo("com.instagram.android", token);
const downloadUrl = await getMyketAppDownloadUrl("com.instagram.android", appInfo.version.code, token);
console.log("APK download link:", downloadUrl);
```

---

### Myket API Endpoints Reference

| Purpose            | URL/Method                                            |
|--------------------|------------------------------------------------------|
| Obtain Token       | POST https://apiserver.myket.ir/v1/devices/authorize/ |
| Get App Info       | GET  https://apiserver.myket.ir/v2/applications/{packageName}/ |
| Get APK URI        | GET  https://apiserver.myket.ir/v1/applications/{packageName}/uri/?... |

---

**Tip:** All requests require an up-to-date Authorization token. Most requests return JSON responses with app metadata and download URLs. For complete details see [Myket's official documentation](https://myket.ir/).
