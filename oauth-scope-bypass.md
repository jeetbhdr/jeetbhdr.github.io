# Hacking OAuth Scopes for fun and profit.
## What Are OAuth Scopes?
When you install or authorize an app on a platform like Shopify, Slack etc, you're redirected to an OAuth authorization URL.
```
https://example.com/v2/oauth?scope=read,write&redirect_url=https://test.com
```
We click "Allow" and move on. But most of us don't check what that `scope` parameter means. The scope parameter defines **what permissions the generated access token will have** on the API.
| Scope Value        | Token Can Do                      |
| ------------------ | --------------------------------- |
| `read`             | Read-only access to API resources |
| `read,write`       | Read and write access             |
| `read,write,admin` | Full administrative access        |


If you request `scope=read`, the token should **only** be able to read data. If you request `scope=read,write`, it should be able to read **and** modify data. That's the theory but implementations break. Few years ago someone on Twitter tweeted about testing on oauth scopes sadly i couldn't find that tweet it was a cool post. Luckily at that time I was already testing an app with OAuth scopes and found few oauth scopes realted vuln. The fact that I found no duplicates confirmed it that people rarely test this stuff, for a few reasons I will explain later.

---
## Finding Targets
Not every OAuth implementation is testable. You need programs that:
1. **Allow creation of OAuth apps via a Developer Console** — platforms like Slack, Shopify etc. that let you register your own OAuth application.
2. **Provide granular permission controls** — the developer console should let you restrict which scopes your app can request. For example, Slack lets you configure exactly which permissions your app is allowed to request from users.

> **Note:** Creating an app and generating tokens is not always straightforward. You will need to manually read the platform's developer documentation. There are nuances around redirect URLs, client secrets, authorization flows, and token exchange endpoints that vary per platform. Take the time to understand the target's OAuth implementation before testing. Because of these hurdles most people don't bother creating OAuth apps and without creating apps you can't test permission realted issues in Oauth scopes.
---

## The Golden Rule: More Scope = More Attack Surface

When setting up your test OAuth app check if there is multiple oauth scopes because more scopes means more permission to play with if your Oauth app has just read and write permission you might find some Priviledge Escalation but the number of vectors will be limited. The more you can enable disable permissions in your token, the more attack surface you have to work with when testing for scope bypass vulnerabilities.

---
## Attack Techniques
### Developer-Side Permission Removal + Client-Side URL Manipulation
1. Create an OAuth app.
2. Generate a token with full permissions (e.g., `read`, `write`, `delete`).
3. Now from the app **remove a permission** (e.g., remove `write`).
4. On the **client side**, manually update the OAuth authorization URL to **include the removed scope**:
```
# Original (after removing write from dev console):
https://example.com/v2/oauth?scope=read&redirect_url=https://yourapp.com
```
```
# Modified (adding write back via URL):
https://example.com/v2/oauth?scope=read,write&redirect_url=https://yourapp.com
```
5. Complete the OAuth flow and obtain a token.
6. Test if the generated token actually has `write` permission** by performing write operations via the API.
If the API allows write operations with this token that's a vulnerability. The server trusted the client-supplied scope instead of enforcing the developer configured restrictions.

### Consent Screen Manipulation
If not all but most OAuth apps display a **consent screen** to the user that lists what the app will be able to do:
```
┌─────────────────────────────────────┐
│  "ExampleApp" wants to:             │
│                                     │
│  ✅ Read your profile               │
│  ✅ Edit your profile       │
│  ✅ Read Write other users profile  │
│                                     │
│         [Allow]    [Deny]           │
└─────────────────────────────────────┘
```
The attack:
1. Intercept the consent request.
2. Remove a scope from the UI/request (e.g: remove "Read Write other users profile" from the consent payload).
3. Complete the flow and obtain the token.
4. Test if the token still has the removed permission by performing CRUD actions against that resource.
If the token retains permissions that the user never explicitly consented to that's a bug. The server should only grant permissions that were displayed and approved in the consent screen.


### Cross-Object Permission Leakage
OAuth scopes are usually object-based each scope corresponds to a specific resource type (e.g: `users.read`, `users.write`, `teams.read` etc).
The attack:
1. Generate a token with multiple scopes.
2. Remove permission for one object (e.g: revoke `teams.read`).
3. Check if data from the revoked object (`team`) leaks into API responses from other permitted endpoints.
For example the generated token has `users.read` permission check whether the `/users` endpoint response includes nested `team` objects or references that should be restricted.
```json
// Response from /users — should NOT include team data after teams scope removal
{
  "contacts": [
    {
      "id": "12345",
      "name": "John Doe",
      "team": {
        // ← This shouldn't be here
        "id": "team_99",
        "name": "Engineering"
      }
    }
  ]
}
```
**If restricted object data appears in other responses that's a vulnerability.**


### No-Permission Token Privilege Escalation
Some OAuth apps might generate tokens without explicitly configuring granular permissions.
1. Create an OAuth app without selecting any specific scopes/permissions.
2. Generate a token.
3. Test what the token can actually do.
**Check if:**
- A token generated by a **view-only user** has **write permissions**.
- A token generated with **no scopes selected** defaults to **full access**.
- A token generated for a **restricted role** can access **admin-level endpoints**.
If a view only user's token can perform high-privilege actions via API. **that's an easy privilege escalation.**.


### IDOR via Object ID Replacement
Standard IDOR testing applies to OAuth-scoped tokens:
1. Generate a token scoped to specific resources.
2. Make API calls and note the object IDs in the responses.
3. Replace IDs with IDs belonging to other users, organizations, or resources.

*If anyone tests for OAuth scopes in a different way than what's covered here and wants to share, please let me know — I'll update and add those methods as well.*

*Tried writing for the first time after being lazy about it for so many years, so if there are any errors or something incorrect let me know and I will correct those mistakes.*

*Ping me on [X/Twitter](https://x.com/jeetbhdr)*

**Happy Hacking and be kind to each other :) :)**

*Used AI to help structure and clean up this writeup.*
