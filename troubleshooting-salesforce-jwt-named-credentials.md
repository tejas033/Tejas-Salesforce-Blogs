# Troubleshooting: OAuth 2.0 JWT with Named Credentials and External Client Apps

**Audience:** Salesforce administrators resolving JWT bearer authentication between orgs.

**Context:** Cross-org integrations that use **OAuth 2.0 JWT bearer**, **External Credentials** / **Named Credentials**, and Apex callouts with a **`CALLOUT:`** URL (for example, Configuration Set / Smart Sync–style flows). Failures are usually **Setup and metadata**, not Apex logic.

---

## JWT token exchange: `invalid_client` / invalid client credentials

**Symptom**

```
System.CalloutException: Unable to complete the JWT token exchange.
Error: invalid_client. Error description: invalid client credentials.
```

This appears while Salesforce exchanges the JWT for an access token at the token endpoint—before your REST call completes.

**What to verify (in order)**

1. **Signing algorithm (External Credential)**  
   For Salesforce-to-Salesforce JWT, use **RS256** on the External Named Credential unless product documentation explicitly requires another algorithm.  
   **Team finding:** A **4096-bit** self-signed cert combined with **RS512** produced `invalid_client`; switching the credential to **RS256** fixed authentication. Keep signing algorithm and certificate usage aligned with what the Connected App (External Client App) and JWT flow expect.

2. **Consumer Key (client id)**  
   The value on the credential must match the **Connected App (External Client App)** in the org that issues tokens—no typos, and not a stale key after app recreation. A **“Missing Consumer Key Parameter”** error is in the same bucket: confirm JWT parameters on the External Credential are complete and mapped to the correct app.

3. **Certificate**  
   The cert used to sign the JWT must match what is registered on the Connected App (uploaded cert or linked principal), as in your general Connected App JWT settings.

If the above checks pass, use the **General checklist** below for both orgs.

---

## We couldn't access the credential(s)…

**Example:** We couldn't access the credential(s)… or the external credential `Your_External_Credential_API_Name` might not exist.

**Checks**

- The **External Credential** exists and its **API name** matches what the Named Credential and Apex expect.
- At least one **Principal** is defined on the External Credential.
- That principal is exposed to the running user: add it to a **Permission Set** or **Profile**, then assign that to the user who performs the callout.

---

## The callout couldn't access the endpoint…

**Example:** The callout couldn't access the endpoint… or the named credential `Your_Named_Credential_API_Name` might not exist.

**Checks**

- Open the **Named Credential** used in the `CALLOUT:` URL.
- Under **Allowed Namespaces for Callouts**, allow the namespace where the calling Apex lives (for example **`BMCServiceDesk`**, **`bmcsdf`**, or your **custom** namespace). If the namespace is not allowed, the platform can block the callout before authentication finishes.

---

## Status Code: 401 — `INVALID_SESSION_ID`

**Example:** Body similar to `[{"message":"Session expired or invalid","errorCode":"INVALID_SESSION_ID"}]`.

**Checks**

- On the **External Credential**, set **Authentication Protocol** to **OAuth 2.0** and **Authentication Flow Type** to **JWT Bearer Flow**. Other combinations can yield a session the target org rejects for API use.

---

## Target org: Connected App (External Client App) OAuth scopes

In the **target** org, open the Connected App used for JWT. Under **Selected OAuth Scopes**, include at least:

- **Access and manage your data (api)** — `api`
- **Perform requests at any time (refresh_token, offline_access)** — `refresh_token`, `offline_access`

Narrow scopes only if your security model allows it and the APIs you call still work.

---

## General checklist (both orgs)

Apply as needed to the **calling org** (Named / External Credential) and the **target org** (Connected App that accepts the JWT).

1. **Consumer Secret** — Matches if your credential type sends a client secret; update both sides if the secret was rotated.
2. **Token / login URL** — Matches the environment (production vs sandbox) and **My Domain** host if you use it.
3. **Connected App** — JWT / OAuth settings enabled; certificate matches the signing setup.
4. **Integration user** — Subject username exists; user is **pre-authorized** for the Connected App where required; profiles / permission sets allow the app.
5. **External Credential principal** — Parameters populated; principal assignment matches **Per User** vs **Named Principal** for your Named Credential mapping.

---

## Related code

Apex uses `CALLOUT:<DeveloperName>` on the endpoint (for example in `ConfigurationSet.getResp` and similar). JWT exchange errors indicate configuration on the credential or Connected App side, not typically Apex branching logic.

---

## Developer Console: smoke test

Run **Execute Anonymous** as the same **user** who will use the feature (so principals and permission sets match real usage).

1. Replace `Named_Credential_for_SmartSync` with your Named Credential **API name**.
2. Change the API version in the path (`v61.0`) if required.
3. Inspect the **Debug** log for **Status Code** and response body; map results to the sections above.

```apex
HttpRequest req = new HttpRequest();

req.setMethod('GET');
req.setTimeout(120000);
req.setEndpoint('CALLOUT:Named_Credential_for_SmartSync/services/data/v61.0/query/?q=SELECT+Id,Name+FROM+Organization');

HttpResponse res = new Http().send(req);
System.debug('Status Code: ' + res.getStatusCode());
System.debug('Res Body: ' + res.getBody());
```

- **Success:** HTTP **200** and JSON with `records` including the target org’s `Organization` Id and Name.
- **Failure:** Non-200, empty body, or `CalloutException` — use the log text and this document to narrow the issue.

---

## Revision history

| Date | Notes |
|------|--------|
| 2026-06-08 | Documented `invalid_client` fix: External Named Credential **RS512 → RS256** with **4096-bit** self-signed cert; added error-oriented sections and target-org scopes. |
| 2026-06-09 | Added Developer Console smoke test; consolidated duplicate JWT / RS256 / consumer key guidance for admins. |
