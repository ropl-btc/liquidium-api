# Liquidium Instant Loan API – Full Integration Guide

> **Audience**  Developers adding Liquidium’s BTC‑secured lending to **web dApps** or **mobile/desktop Bitcoin wallets** that already handle key‑management (e.g. LaserEyes, Xverse, UniSat, Hiro, Sparrow, etc.).
>
> **Version**  API v1 & OpenAPI 0.0.1 (📄 openapi.json)

---

## 0  Why Liquidium?

- **Peer‑to‑peer, on‑chain loans** — collateral (Runes, BRC‑20, inscriptions) is locked in a multisig escrow controlled by a DLC contract; lender BTC is disbursed instantly.
- **No bridges / L2 risk** — all flows run on Bitcoin L1 using PSBTs + DLCs.
- **Two requests, one signature** — every borrower flow is *prepare → sign → submit*.

---

## 1  Authentication Flow

Liquidium requires **two factors** on every protected call:

1. **Bearer API key** (application‑level)
2. **x‑user‑token** (72 h JWT proving wallet ownership)

### 1.1  Step‑by‑step

| # | Actor  | Action                                             | Endpoint                                      | Notes                      |
| - | ------ | -------------------------------------------------- | --------------------------------------------- | -------------------------- |
| 1 | Client | `POST /auth/prepare`                               | sends `{payment_address, ordinals_address}`   | ordinals can equal payment |
| 2 | Server | 200 → ✉️ returns *two challenge messages*          | `{payment:{msg,nonce}, ordinals:{msg,nonce}}` |                            |
| 3 | Wallet | signs message(s) with private key(s)               | happens in‑app                                |                            |
| 4 | Client | `POST /auth/submit` with signatures                |                                               |                            |
| 5 | Server | 200 → `{user_jwt, is_first_login, vault_address?}` | store JWT for 72 h                            |                            |

> **Header template** for every subsequent call:

```
Authorization: Bearer <API‑Key>
x-user-token: <user_jwt>
Content-Type: application/json
```

*When JWT expires (401), simply re‑run the flow in the background; wallets should auto‑sign the standard message ********************`"Please sign this message to verify your wallet\naddress: {addr}\nnonce: {nonce}\n"`********************.*

---

### 1.2  JWT token schema

The token returned by **/auth/submit** is a standard [HS256 JWT].

```jsonc
Header  // Base64‑decoded first segment
{
  "alg": "HS256",
  "typ": "JWT"
}

Payload // second segment (no user‑id claim exposed)
{  "payment_address": "bc1q…",     // echoed back from /auth/prepare
  "ordinals_address": "bc1p…",    // echoed back from /auth/prepare
  "iat": 1714245123,                // issued‑at (unix‑epoch seconds)
  "exp": 1714504323                 // expiry (+72 h)
}
```

*The signature (third segment) is created with your app’s ****API key**** as the secret, so only Liquidium can issue valid tokens.*

**Validation tip**   Clients generally treat the JWT as opaque, but if you need to inspect it (e.g. to confirm expiration without hitting the API), decode the first two segments with `atob`/`Buffer.from` — no verification key is published.

---

## 2  Browsing Collateral & Offers

### 2.1  Endpoints

| Purpose                    | Method                                                        | URL |
| -------------------------- | ------------------------------------------------------------- | --- |
| List all supported runes   | `GET /borrower/collateral/runes`                              |     |
| Rune detail (valid ranges) | `GET /borrower/collateral/runes/{runeId}`                     |     |
| Quote offers for amount    | `GET /borrower/collateral/runes/{runeId}/offers?rune_amount=` |     |

### 2.2  Sequence

1. **List runes** → UI populates dropdown (floor price + offer counts).
2. **User selects rune** → fetch detail with `/{runeId}` to obtain *valid\_ranges* (amount minima/maxima + valid `loan_term_days`).
3. **User enters amount** → client validates `min ≤ amount ≤ max` for *any* range; if invalid, suggest nearest valid value.
4. **Get offers** → call `/offers?rune_amount=`; server returns top offers for every supported term (1d, 2d, …) plus common data (interest, divisibility).

### 2.3  What the screenshots show

*Left‑most mobile screen* — wallet asset list with **“Borrow”** button. Tapping it jumps to **Borrow BTC** screen.
*Borrow BTC (empty form)* — steps 1‑5 annotated:

1. Dropdown = *list runes* (endpoint 1)
2. User picks LIQUIDIUMTOKEN
3. UI fetches rune detail (endpoint 2) ⇒ populates slider placeholder.
4. User types 1300 (token‑adjusted string e.g. “130000” if divisibility=2).
5. UI GETs best offers (endpoint 3) and renders cards per term.

*Borrow BTC (offers visible)* — item “0.021 BTC – LTV 80%” etc. User taps **Continue**.

Error screenshot (red text) demonstrates **client‑side validation** if amount outside any range; backend fallback: 422 UNPROCESSABLE.

---

## 3  Start Loan Flow

> Two API calls + one PSBT signature.

### 3.1  Prepare

```http
POST /borrower/loans/start/prepare
{
  "instant_offer_id": "<offer_id>",
  "fee_rate": 12,
  "token_amount": "80000",
  "borrower_payment_address": "bc1q…",
  "borrower_payment_pubkey": "02ab…",
  "borrower_ordinal_address": "bc1p…",
  "borrower_ordinal_pubkey": "03cd…"
}
```

Response ⇒ `{prepare_offer_id, base64_psbt, sides[]}`.

*Sequence diagram (pink block)* —

1. Client sends above JSON (headers incl. JWT).
2. Server crafts PSBT locking collateral to  escrow, paying principal to borrower.
3. Server returns PSBT + signing hints.

### 3.2  Sign

*Use your wallet SDK* – iterate over `sides`; sign each input with specified key, `sighash`, tweaks.

### 3.3  Submit

```http
POST /borrower/loans/start/submit
{
  "signed_psbt_base_64": "cHNidP8…",
  "prepare_offer_id": "<id from step 1>"
}
```

Successful 200 ⇒ `{loan_transaction_id}` (txid already broadcast).

### 3.4  What the UI screenshot shows

*Final confirmation screen* (“Due date”, “Repay at”, network fee). User clicks **Start Loan** → behind the scenes step 1‑3 happens, then success ✅ screen (“Loan started”).

---

## 4  Repay Loan Flow

Mirrors start loan but with `offer_id` instead of `instant_offer_id`.

### 4.1  Prepare

`POST /borrower/loans/repay/prepare` → returns `{base64_psbt, sides, utxo_content}` where *utxo\_content* flags if any chosen UTXO contains inscriptions/runes (rare warning overlay in UI).

### 4.2  Sign + Submit

`POST /borrower/loans/repay/submit` with signed PSBT ⇒ `{repayment_transaction_id}`.

### 4.3  Repay screenshot explained

Left panel: **Collectibles → LOANS tab** shows countdown bars per active loan and **Repay** buttons. Selecting a loan opens **Repay loan** screen (right panel) that echoes *due date*, *repayment amount*, editable *network fee* and a **Repay** CTA; this screen corresponds to *prepare* response.

---

## 5  Monitor Portfolio

`GET /borrower/portfolio` returns every offer (active + history) with state machine:

```
OFFERED → ACCEPTED → ACTIVATING → ACTIVE → (REPAYING → REPAID) | (DEFAULTED → CLAIMING → CLAIMED)
```

Front‑end should poll the Bitcoin mempool or use your wallet’s event callbacks to track confirmation status of returned `loan_transaction_id` and `repayment_transaction_id`.

---

## 6  Error Model

Uniform JSON:

```json
{
  "error": "BAD_REQUEST",     // ENUM
  "errorMessage": "Invalid …" // optional human text
}
```

Key enums: BAD\_REQUEST · UNAUTHORIZED · FORBIDDEN · NOT\_FOUND · STATE\_CONFLICT · UNPROCESSABLE · TOO\_MANY\_REQUESTS · INTERNAL\_SERVER\_ERROR.

---

## 7  Implementation Cheatsheet

- **Decimal handling**: `token_amount` is *raw* integer (`displayAmount * 10^divisibility`).
- **Fee rate**: borrower controls sat/vB; show mempool recommended fees.
- **PSBT library**: bitcoinjs-lib (JS/TS), msg‑signed within browser; or PSBTv2 if your wallet supports.
- **Safety**: always display PSBT outputs for user review.
- **Rate limiting**: server returns **429 TOO\_MANY\_REQUESTS** when the per‑IP throttle is exceeded – backoff and retry with exponential delay.

---

## 8  Sandbox & Testing

*Testing environment*: `https://app.liquidium.fi/lend/runes` – toggle the **Instant** switch in the top‑right to use the Instant Loan endpoints with small collateral amounts. Use small fee\_rate (1‑2 sat/vB) on mainnet vaults for cost‑free dev.

---

## 9  FAQ

- **Q Can I batch multiple loans in one PSBT?** – Not yet; 1‑loan‑per‑PSBT keeps collateral isolation simple.
- **Q Do I need separate ordinals & payment addresses?** – No, you can reuse the same taproot for both.

---

### Appendix A  OpenAPI schema

Full machine‑readable spec is embedded in `/openapi.json` (included in this repo) and supports Swagger / Postman import.

