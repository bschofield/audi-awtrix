# Auth status: BROKEN since mid-October 2026

Last checked: **2026-10-16**. `python3 audi_awtrix.py --config config.json` fails at login and the
display gets no data. The cause is a server-side lockout by Audi/CARIAD. There is no bug in this
repo's code to fix.

## Symptom

```
[...] Got authorization code: eyJraWQiOiI2YWNhYzRl...
Exception: Token exchange failed: 400 - {"error":"invalid assertion headers"}
```

This is raised from the "Step 4: Exchange code for tokens" block of `AudiConnect.login()` in
`audi_connect.py`. Everything before that step still works: the HTML login scrape, the email and
password submission, the consent redirects and getting the authorization code.

## Why: every headless login path is closed

| Path | Status (2026-10-16) | Error |
|---|---|---|
| Password / authorization-code + PKCE (what this repo does) | Blocked since ~July 2026 | `400 invalid assertion headers` on `POST https://emea.bff.cariad.digital/auth/v1/idk/oidc/token` |
| OAuth Device Authorization Grant (RFC 8628), upstream's July workaround | Blocked ~early Oct 2026 | `unauthorized_client: client is not allowed to use the device_code grant` from `https://identity.vwgroup.io/oidc/v1/device_authorization` |
| `refresh_token` grant on an existing token | Blocked since ~2026-10-12 | `invalid assertion headers` |

- The token endpoint now requires a **Google Play Integrity attestation**: a Google-signed verdict
  from a real Android device running the genuine myAudi APK. It cannot be generated in Python and
  cannot be extracted from the APK.
- `X-QMAuth` is **not** the problem. Requests get past the QMAuth check and fail at the assertion
  check. Don't spend time hunting for a new QMAuth secret.
- The hybrid OIDC flow that fixed VW/Cupra connectors gets `invalid client` for the Audi client
  (reported upstream).
- The client ID `09b6cbec-cd19-4589-82fd-363dfa8c24da@apps_vw-dilab_com` is still current. The
  market config (`content.app.my.audi.com/.../market/GB/en`) doesn't supply an
  `idkClientIDAndroidLive` override.
- There is no cached `~/.audi_tokens.json` on this machine, so there's no surviving token to
  limp along on. Refresh is blocked anyway.
- VW Group posted "login-flow changes" notices around 2026-10-06. The VW brand integrations went
  through the same sequence earlier in 2026: new logins blocked first, then existing tokens
  invalidated.

## Where to look for news

- audi_connect_ha [#891](https://github.com/audiconnect/audi_connect_ha/issues/891): refresh rejected, `invalid assertion headers`
- audi_connect_ha [#893](https://github.com/audiconnect/audi_connect_ha/issues/893): device code `unauthorized_client`
- audi_connect_ha [#772](https://github.com/audiconnect/audi_connect_ha/issues/772): original Play Integrity analysis and the device-code workaround
- [robinostlund/homeassistant-volkswagencarnet#977](https://github.com/robinostlund/homeassistant-volkswagencarnet/issues/977): VW Group-wide thread, including EU Data Act discussion
- Recent upstream commits: `gh api repos/audiconnect/audi_connect_ha/commits?per_page=10`

## Quick probes to re-check

```bash
# 1. Does discovery advertise anything new (e.g. device_authorization_endpoint, new grants)?
curl -s https://emea.bff.cariad.digital/auth/v1/idk/oidc/openid-configuration | python3 -m json.tool

# 2. Is the device-code grant allowed again? Success returns device_code/user_code/verification_uri.
curl -s -X POST https://identity.vwgroup.io/oidc/v1/device_authorization \
  -H 'Accept: application/json' -H 'X-App-Version: 4.31.0' -H 'X-App-Name: myAudi' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'client_id=09b6cbec-cd19-4589-82fd-363dfa8c24da%40apps_vw-dilab_com&scope=openid%20mbb%20profile%20badge%20cars%20dealers%20vin'

# 3. Does the full password flow work again?
python3 audi_awtrix.py --config config.json
```

## Options when picking this up

1. **Upstream has a new working flow.** Port it into `audi_connect.py`. Upstream's
   `custom_components/audiconnect/audi_services.py` is the reference (`_discover_endpoints`,
   `request_device_code`, `poll_device_token`, `login_with_refresh_token`, `_finalize_session`).
   If it is device-code again: login needs a one-time browser approval, then runs on the persisted
   refresh token. Note that upstream sends **no** `X-QMAuth` on refresh or device polling, while
   `_refresh_tokens()` here does. Drop it if porting.
2. **EU Data Act portal.** VW Group offers B2C data as periodic file exports (reportedly every
   15 minutes, but the data's median age has been measured at ~2.5 hours). The script could poll
   those files instead of the live API, if staleness is acceptable for the display.
3. **Android bridge.** A real device produces Play Integrity tokens for the script. This would be
   a separate project, not a patch here.
