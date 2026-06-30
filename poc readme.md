## 6. Summary table

| Item | Google | GitHub |
|---|---|---|
| Provider credential received | ID token (JWT) **or** access token | Authorization `code` |
| Code/token expiry (provider) | ID & access token ~1 hour | Code ~10 min; access token does **not** expire (default) |
| Provider refresh token | No (`offlineAccess: false`) | No (default) |
| Provider token stored by backend? | No | No |
| **App session token issued** | **JWT, HS256, 1 hour** | **JWT, HS256, 1 hour** |
| App refresh token | **None** | **None** |
| Server-side session ID? | No — stateless JWT | No — stateless JWT |
| On expiry | Re-login required | Re-login required |

---
