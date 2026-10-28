# Visa Merchant Screening Service (VMSS)

## Screen a merchant

Checks whether a merchant, sponsored merchant, or agent may match previously terminated records. Use this during onboarding to identify possible risk. VMSS searches Visa's records separately from Mastercard MATCH Pro.

> **Result:** Possible match → risk review. No match → continue onboarding. Failed or incomplete check → keep screening unresolved.

### Endpoint

| Method | Path | Request / response format |
| --- | --- | --- |
| **POST** | `/vmss/v2/searchTerminatedRequest` | `application/json` |

| Environment | Base URL |
| --- | --- |
| Sandbox | `https://sandbox.api.visa.com` |
| Certification / production | Use the endpoint supplied for your approved Visa project. |

## Authentication

**Mutual TLS (Two-Way SSL) + HTTP Basic**

| Required credential / configuration | Description |
| --- | --- |
| Client certificate and private key | Identify the backend during the TLS connection. |
| Server trust configuration | Verifies Visa's server certificate. |
| Visa API username and password | Project/environment credentials for Basic authentication. |
| Acquirer BID and country | Authorized institution ID and two-letter institution country. |

1. Create a Visa Developer project with **VMSS** and obtain the credentials.
2. Configure the HTTP client with the certificate/private key and server trust.
3. Build `Authorization: Basic <BASE64(username:password)>` using the Visa API credentials.
4. Send the JSON request through this configured HTTPS client.

The certificate is exchanged during TLS, not in the request body. Use API credentials, not the portal login. Production requires Visa approval and the applicable agreement.

## Request

### Headers

| Header | Type | Required | Description / value |
| --- | --- | --- | --- |
| `Authorization` | string | Yes | `Basic <BASE64(username:password)>`. |
| `Content-Type` | string | Send for JSON | `application/json`. |

### Body schema

Include only the fields below and any applicable conditional fields. Names are case-sensitive. All paths are relative to `searchTerminatedRequest` unless shown as the wrapper itself.

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `searchTerminatedRequest` | object | Include | Top-level request wrapper. |
| `acquirerBID` | string | Yes | Authorized Visa BID; exactly eight digits. |
| `acquirerCountryOrRegion` | string | Yes | Institution country; two uppercase letters, e.g. `JO`. |
| `globalSearch` | boolean | Yes | `true` for worldwide search; `false` for regional search. Send explicitly. |
| `terminatedRecordSearchCriteria` | object | Yes | Business and owner information. |
| `terminatedRecordSearchCriteria.DBAName` | string | Yes | Operating/trading name; 1–70 characters. |
| `terminatedRecordSearchCriteria.address` | object | Yes | Business address. |
| `terminatedRecordSearchCriteria.address.city` | string | Yes | City; 2–30 characters. |
| `terminatedRecordSearchCriteria.address.streetAddress` | string | Yes | Street/location; 4–100 characters. |
| `terminatedRecordSearchCriteria.address.countryOrRegion` | string | Yes | Business country; two uppercase letters, e.g. `JO`. |
| `terminatedRecordSearchCriteria.businessPhoneNumbers` | array of strings | Yes | 1–3 phones; each 8–15 characters, matching Visa's format. |
| `terminatedRecordSearchCriteria.category` | string | Yes | `"0"` for an ordinary merchant. Other entity codes are in the API reference. |
| `terminatedRecordSearchCriteria.principals` | array of objects | Yes | Actual business owners. |
| `terminatedRecordSearchCriteria.principals[].name` | string | Yes | Owner's full name; 5–35 characters. |
| `terminatedRecordSearchCriteria.tradeOverInternet` | boolean | Yes | Whether the business trades online. Send the actual value. |

### Conditional fields

Paths below are relative to `terminatedRecordSearchCriteria`.

| Field | Type | Required when | Description |
| --- | --- | --- | --- |
| `principals[].residentIDOrNationalID` | string | CEMEA clients | Owner's resident/national ID; maximum 30 characters. Included in the Jordan example. |
| `webAddresses` | array of strings | `tradeOverInternet` is `true` | Business website; up to 3 URLs, each 4–84 characters. |
| `legalName` | string | Registered parent/holding name differs from trading name | 4–70 characters; omit for a sole proprietor. |

### Request example

Fictitious Jordan merchant that does not trade online. Replace the BID with your authorized institution BID and use actual merchant/owner information. For an online merchant, set the flag to `true` and include its website.

```json
{
  "searchTerminatedRequest": {
    "acquirerBID": "10033027",
    "acquirerCountryOrRegion": "JO",
    "globalSearch": true,
    "terminatedRecordSearchCriteria": {
      "DBAName": "EXAMPLE STORE",
      "address": {
        "city": "Amman",
        "streetAddress": "10 Example Street",
        "countryOrRegion": "JO"
      },
      "businessPhoneNumbers": [
        "+962790000000"
      ],
      "category": "0",
      "principals": [
        {
          "name": "AHMAD ALI",
          "residentIDOrNationalID": "1234567890"
        }
      ],
      "tradeOverInternet": false
    }
  }
}
```

## Responses

### `200` — Search completed

Check HTTP `200` **and** `status.statusCode = "API000"`. The table lists the fields **used by this integration**, not every field returned by Visa.

| Response field | Type | Used for | Description |
| --- | --- | --- | --- |
| `status.statusCode` | string | Success check | `API000` means success. |
| `searchTerminatedResponse.searchRequestRef.searchRequestRefID` | string | Tracking | Provider search reference. Save with the onboarding application. |
| `searchTerminatedResponse.possibleMatches.totalCount` | integer | Decision | Number of possible returned matches. |
| `searchTerminatedResponse.possibleMatches.matchedRecords` | array | Match evidence | Returned termination records, when present. |

The envelope schema does not guarantee all these fields; missing or unusable decision data must not be treated as zero matches.

**Response example — selected fields only:**

```json
{
  "status": { "statusCode": "API000" },
  "searchTerminatedResponse": {
    "searchRequestRef": { "searchRequestRefID": "8406523" },
    "possibleMatches": { "totalCount": 1 }
  }
}
```

### Match decision

| Condition | Return to your business flow |
| --- | --- |
| Successful result with count greater than `0`, or returned match records exist | `POSSIBLE_MATCH` |
| HTTP `200`, `statusCode = "API000"`, complete usable result, and explicitly zero matches | `NO_MATCH` |
| Failed call, missing/unusable result, or inconsistent data | `ERROR` |

**The example returns `POSSIBLE_MATCH`.** Do not default missing counts/arrays to zero. Visa's `E`, `P`, and `N` codes describe individual field matches, not the overall search result. A possible match requires review; it is not an automatic rejection.

### Error responses

| HTTP status | VMSS code | Meaning |
| --- | --- | --- |
| `400` | `API001` | Missing or invalid fields. |
| `401` | `API004` | Authentication failure. |
| `403` | `API007` | Access denied. |
| `404` | `API002` | Resource/data not found; do not assume a successful no-match search. |
| `405` | `API008` | Incorrect HTTP method. |
| `406` | `API009` | Unacceptable media type. |
| `500` | `API003` | Processing failure. |

## References

- [Service overview](https://developer.visa.com/capabilities/visa-merchant-screening-service)
- [API reference — full schemas, category codes, and errors](https://developer.visa.com/api-reference/visa-merchant-screening-service)
- [Getting started and access](https://developer.visa.com/capabilities/visa-merchant-screening-service/docs-getting-started)
- [VMSS authentication method](https://developer.visa.com/capabilities/visa-merchant-screening-service/docs-authentication)
- [Mutual TLS and HTTP Basic setup](https://developer.visa.com/docs/authentication-methods/two-way-ssl)
- [How to use VMSS](https://developer.visa.com/docs/visa-merchant-screening-service/docs-how-to)

*Examples are illustrative; response excerpts omit unrelated fields. No authenticated API calls were performed.*
