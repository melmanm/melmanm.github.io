
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

* Code/Token injection attacks
Remediation of above - state parameter "nonce"

### Token Reply

### Google phishing attack