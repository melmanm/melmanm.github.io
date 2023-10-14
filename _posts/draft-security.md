
## Implicit flow
Token is returned directly from `/authorize` endpoint. Can be used in hybrid scenarios `scope = id_token code`.

It was historically used because browsers had limitations:
JS requests could be sent only to the same domain from page was loaded. Authorization Code flow requires POST to token endpoint with code. Now CORS is adopted by browsers.

### Access token leakage
In implicit grant flow token is included in URL. It can be leaked by:
* man in the middle attack - it is in the URL
* through browser history 
* leakage through browser address bar? - to check
Remediation of above - `response_mode=form_post` token is send in post header.

CSRF

* Redirect URI related attacks - Trick user to click the link with attacker redirect URI using open redirector or partially open redirector with `redirect_to`

* attacker Code/Token injection attacks
Remediation of above - "state" or  "nonce"

### Token Reply


1. Recap from previous presentation
- examples of oauth
- roles 
- authorization code flow
2 . From the user perspective
- ssl certificate
- url
-  redirect_uri (how github does it)*
2.1 Good developers should enable users to validate all these parameters

3. Redirect uri - malicious application stoles authorization code
3.1 Open redirector attack
- authorization server validation
- wildcards
3.2. Redirect uri - malicious application gets access to user resources
- Google phishing attack
- users were not able to verify redirect uri in easy way
- verification process when it comes to high privileged access scopes
- don't use authorization server names in Application name.

4. Authorization server url
4.1 Clickjacking 
- iframe, login form can be done in popup
- demo
- `X-Frame-Options: DENY` header
  
5. Authorization code
5.1 CSRF - user application gets attacker access
-> demo
-> state parameter, pkce extension
5.2 Code injection, stolen code is exchanged by the token 

6.PKCE downgrade?

7. JWT token
7.1 None signature 
7.2 Algorithm confusion attack 
7.3 Attacks using the jku header 
## User in desktop mobile application. Device flow.