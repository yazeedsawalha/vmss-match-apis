# Visa VMSS — Simple Integration Guide

## 1. Service overview

**Service name:** Visa Merchant Screening Service (VMSS).  
**Purpose:** Check whether a prospective merchant, sponsored merchant, or agent may match previously terminated records before onboarding.

**Why use it?** It provides termination information for the bank's risk review before enabling Visa card acceptance. It queries Visa's screening records, separately from Mastercard MATCH Pro. A possible match needs investigation before a decision.

## 2. Authentication

**Method:** Mutual TLS (Two-Way SSL) **plus HTTP Basic authentication**.

### What you need

| Item | Mandatory? | Description |
| --- | --- | --- |
| Visa client certificate | Yes | Identifies the application during the TLS connection. |
| Matching private key | Yes | Used with the certificate by the backend HTTP client. |
| Trusted server certificate chain | Yes | Allows your client to verify Visa's server. |
| Visa API username | Yes | Username supplied in your project's Two-Way SSL credentials. |
| Visa API password | Yes | Password supplied with that username. |
| Acquirer BID | Yes | Your institution's authorized eight-digit Visa business identifier. |

### Steps

1. Create a Visa Developer project and enable **VMSS**.
2. Obtain the client certificate, private key, and HTTP Basic credentials for the environment.
3. Configure your HTTP client with the client certificate/key and trusted server certificates.
4. Build `Authorization: Basic BASE64(username:password)` using the Visa API credentials.
5. Send the JSON request through this configured HTTPS client.

The certificate is supplied during the TLS connection, not in the JSON body. Production requires Visa approval and the applicable agreement.

## 3. API — Screen a merchant

**Method:** `POST`  
**Path:** `/vmss/v2/searchTerminatedRequest`  
**Sandbox URL:** `https://sandbox.api.visa.com/vmss/v2/searchTerminatedRequest`

Obtain certification/production URLs and credentials from your approved Visa project.

### Request headers

| Header | Mandatory? | Description / value |
| --- | --- | --- |
| `Authorization` | Yes | `Basic <BASE64(username:password)>`. |
| `Content-Type` | Send for JSON | `application/json`. |
| `Accept` | No; recommended | `application/json`. |
| `request-message-id` | No; recommended | Caller-generated correlation ID; use alphanumeric characters. |
| `request-message-ts` | No; recommended | Current GMT/UTC timestamp with milliseconds. |

```http
POST https://sandbox.api.visa.com/vmss/v2/searchTerminatedRequest
Content-Type: application/json
Accept: application/json
Authorization: Basic <ENCODED_VISA_API_CREDENTIALS>
request-message-id: VMSS202610220001
request-message-ts: 2026-10-22T08:30:00.000Z
```

The timestamp is an example; use the actual current time. No query parameters are declared for this search endpoint.

## 4. Request fields

Required children apply when their parent is supplied. Field names are case-sensitive.

### Request wrapper

| Field | Type | Mandatory? | Description / rule |
| --- | --- | --- | --- |
| `searchTerminatedRequest` | object | Send | Request wrapper; include it in the payload. |
| `searchTerminatedRequest.acquirerBID` | string | Yes | Exactly eight digits; use your authorized Visa BID. |
| `searchTerminatedRequest.acquirerCountryOrRegion` | string | Yes | Two uppercase letters; acquiring institution country. |
| `searchTerminatedRequest.globalSearch` | boolean | Yes | `true` worldwide; `false` regional. Send explicitly even though the schema default is false. |
| `searchTerminatedRequest.retroAlertIfNoMatch` | boolean | No | Default false. If enabled, saves the inquiry for checking against new termination records over 180 days. |
| `searchTerminatedRequest.terminatedRecordSearchCriteria` | object | Yes | Business and owner information to search. |

### Business search criteria

Inside `searchTerminatedRequest.terminatedRecordSearchCriteria`:

| Field | Type | Mandatory? | Description / validation |
| --- | --- | --- | --- |
| `DBAName` | string | Yes | Operating/trading name; 1–70 characters. Preserve uppercase `DBA` in the property name. |
| `address` | object | Yes | Business registered address; see address table. |
| `businessPhoneNumbers` | array of strings | Yes | 1–3 business phone numbers. Each 8–15 characters and must satisfy Visa's phone pattern. |
| `category` | string | Yes | One character `0`–`5`; entity type, not MCC. |
| `principals` | array of objects | Yes | Owner/principal details; each supplied entry requires `name`. Supply actual principals; no array bounds are declared in the retrieved schema. |
| `tradeOverInternet` | boolean | Yes | Whether this business trades online. Send explicitly. |
| `legalName` | string | Conditional | Parent/holding registered name when different from trading name, 4–70 characters; omit for a sole proprietor. |
| `taxID` | string | No | Business tax identifier, 6–30 characters. |
| `webAddresses` | array of strings | Conditional | Required when `tradeOverInternet` is true. Up to 3 URLs, each 4–84 characters. |
| `businessEmailAddress` | string | No | Business email, 4–254 characters, matching email pattern. |
| `financialAccts` | array of objects | No | Account details for matching; see account table. |
| `merchantCategoryCodes` | array of strings | No | Up to 5 MCCs; send four-digit strings and preserve leading zeros. |
| `businessRegistrationNumber` | string | No | Business registration identifier, 2–30 characters. |

### Category values

| `category` | Entity |
| --- | --- |
| `"0"` | Merchant |
| `"1"` | Payment facilitator |
| `"2"` | Independent sales organization |
| `"3"` | Marketplace |
| `"4"` | Staged digital wallet operator |
| `"5"` | Sponsored merchant |

### Business address

Inside `terminatedRecordSearchCriteria.address`:

| Field | Type | Mandatory? | Description / validation |
| --- | --- | --- | --- |
| `city` | string | Yes | City, 2–30 characters. |
| `streetAddress` | string | Yes | Street/location details, 4–100 characters. |
| `countryOrRegion` | string | Yes | Two uppercase letters, e.g. `JO`. |
| `stateOrProvince` | string | No in generic schema | State/province, 2–20 characters when supplied. Confirm local business requirements. |
| `zipOrPostalCode` | string | No in generic schema | Postal code, 3–8 characters when supplied. |

### Principal owners

Inside `terminatedRecordSearchCriteria.principals[]`:

| Field | Mandatory? | Description / validation |
| --- | --- | --- |
| `name` | Yes | Full principal name, 5–35 characters. |
| `businessPhoneNumber` | No | Principal phone, 8–15 characters, Visa phone pattern. |
| `businessEmailAddress` | No | Principal email, 4–254 characters. |
| `residentIDOrNationalID` | Conditional | Required for clients in **CEMEA**, according to the description; maximum 30 characters. Confirm applicability with the bank's Visa onboarding team. |
| `passportNumber` | No | Passport identifier, maximum 30 characters. |
| `driverLicenseNumber` | No | License identifier, maximum 20 characters. |
| `principalID` | No | Legacy VMSS 1.0 principal reference, maximum 30 characters; not a substitute for a national ID. |
| `SSN` | No | Social-security identifier; schema length 11 and a specific pattern. Uppercase property name. Use only when applicable. |

### Optional financial account details

Inside `terminatedRecordSearchCriteria.financialAccts[]`:

| Field | Mandatory? | Description / validation |
| --- | --- | --- |
| `intBankAccountNumber` | No | IBAN, 4–34 characters with IBAN pattern; use an actual valid IBAN, including its checksum. |
| `financialAccountNumber` | No | Numeric domestic account identifier; maximum 20 characters. |
| `financialInstitutionID` | No | Institution reference, such as applicable routing/SWIFT identifier; maximum 10 characters in this schema. |

### Request example

Fictitious Jordan merchant. Replace the BID with your institution’s authorized BID and use actual merchant/owner data. The national ID illustrates the CEMEA requirement.

```json
{
  "searchTerminatedRequest": {
    "acquirerBID": "10033027",
    "acquirerCountryOrRegion": "JO",
    "globalSearch": true,
    "retroAlertIfNoMatch": false,
    "terminatedRecordSearchCriteria": {
      "DBAName": "EXAMPLE STORE",
      "address": {
        "city": "Amman",
        "streetAddress": "10 Example Street",
        "countryOrRegion": "JO",
        "zipOrPostalCode": "11118"
      },
      "businessPhoneNumbers": [
        "+962790000000"
      ],
      "category": "0",
      "principals": [
        {
          "name": "AHMAD ALI",
          "residentIDOrNationalID": "1234567890",
          "businessPhoneNumber": "+962790000001",
          "businessEmailAddress": "ahmad@example.com"
        }
      ],
      "tradeOverInternet": true,
      "webAddresses": [
        "https://example.com"
      ],
      "merchantCategoryCodes": [
        "5732"
      ],
      "businessEmailAddress": "business@example.com"
    }
  }
}
```

## 5. Response

Check HTTP status **and** `status.statusCode`. The documented success is HTTP `200` with code `API000`. Save the search reference and review possible matches.

### Main response fields

“No” means not guaranteed by the published envelope schema. Your application should expect usable status, reference, and match data before treating screening as complete.

| Field | Type | Mandatory in schema? | Description |
| --- | --- | --- | --- |
| `status` | object | No; expected | Application status. |
| `status.statusCode` | string | No; expected | `API000` means success. |
| `status.statusDescription` | string | No | Status explanation. |
| `searchTerminatedResponse` | object | No; expected on success | Search response wrapper. |
| `searchTerminatedResponse.searchRequestRef` | object | No; expected | Search reference and metadata. |
| `searchRequestRef.searchRequestRefID` | string | No; expected | Provider search reference; save it. |
| `searchRequestRef.acquirerBID` | string | Yes within object | Requesting institution BID. |
| `searchRequestRef.acquirerCountryOrRegion` | string | Yes within object | Requesting institution country. |
| `searchRequestRef.globalSearch` | boolean | No | Search scope. |
| `searchRequestRef.terminatedRecordSearchCriteria` | object | No | Echoed request criteria, if returned. |
| `searchTerminatedResponse.possibleMatches` | object | No; expected | Match results. |
| `possibleMatches.totalCount` | integer | No | Possible match count. |
| `possibleMatches.matchedRecords` | array | No | Matching termination records. |

Shorter paths above continue beneath `searchTerminatedResponse`.

### Matched record

Inside `searchTerminatedResponse.possibleMatches.matchedRecords[]`:

| Field | Type | Mandatory in schema? | Description |
| --- | --- | --- | --- |
| `terminatedRefID` | string | No | Termination record reference; different from search reference. |
| `acquirerBID` | string | No | Reporting institution BID. |
| `acquirerName` | string | No | Reporting institution name. |
| `acquirerCountryOrRegion` | string | No | Reporting institution country. |
| `terminatedRecord` | object | No | Reported business and termination details. |
| `terminatedRecordMatch` | object | No | Field-level match indicators. |

### Termination fields used in the example

Inside `matchedRecords[].terminatedRecord`:

| Field | Type | Mandatory within returned object? | Description |
| --- | --- | --- | --- |
| `DBAName` | string | Yes | Reported trading name. |
| `address` | object | Yes | Registered address; same child names as the request. |
| `businessPhoneNumbers` | array | Yes | Business phone numbers. |
| `cardAcceptorIDs` | array | Yes | Card-acceptor identifiers. |
| `category` | string | Yes | Entity type. |
| `contractStartDate` | string | Yes | Agreement start date. |
| `contractEndDate` | string | Yes | Termination date. |
| `incorportationStatus` | string | Yes | Entity classification; preserve this published spelling. |
| `primaryListingReason` | string | Yes | Primary termination reason. |
| `principals` | array | Yes | Reported owners; each requires `name`. |
| `tradeInternationally` | boolean | Yes | International-trading indicator. |
| `tradeOverInternet` | boolean | Yes | Online-trading indicator. |
| `webAddresses` | array | Conditional | Required by descriptive rule when trading online. |
| `acquirerAssignedMerchantID` | string | Conditional | Described as mandatory for merchants. |

Some published Visa examples omit fields marked required in the schema. Confirm actual behavior during testing; incomplete results should be investigated.

### Match indicator fields

| Field used in example | Type | Mandatory? | Description |
| --- | --- | --- | --- |
| `terminatedRecordMatch.DBAName` | string | No | Trading-name match indicator. |
| `terminatedRecordMatch.addressMatch` | object | No | Address match indicators. |
| `addressMatch.city` | string | No | City match indicator. |
| `addressMatch.streetAddress` | string | No | Street match indicator. |
| `addressMatch.countryOrRegion` | string | No | Country match indicator. |
| `terminatedRecordMatch.principalsMatch` | array | No | Owner match indicators. |
| `principalsMatch[].name` | string | No | Owner-name match indicator. |

**`E` = Exact match · `P` = Partial match · `N` = No match for that field.** A field-level `N` does not mean the whole search returned no matches.

### Response example

Illustrative response showing one possible match; additional optional fields may be returned.

```json
{
  "status": {
    "statusCode": "API000",
    "statusDescription": "SUCCESS"
  },
  "searchTerminatedResponse": {
    "searchRequestRef": {
      "searchRequestRefID": "8406523",
      "acquirerBID": "10033027",
      "acquirerCountryOrRegion": "JO",
      "globalSearch": true
    },
    "possibleMatches": {
      "totalCount": 1,
      "matchedRecords": [
        {
          "terminatedRefID": "EXAMPLE_TERMINATED_RECORD_001",
          "acquirerBID": "10048640",
          "terminatedRecord": {
            "DBAName": "EXAMPLE STORE",
            "address": {
              "city": "Amman",
              "streetAddress": "10 Example Street",
              "countryOrRegion": "JO",
              "zipOrPostalCode": "11118"
            },
            "businessPhoneNumbers": [
              "+962790000000"
            ],
            "cardAcceptorIDs": [
              "12345678910111"
            ],
            "category": "0",
            "contractEndDate": "2025-05-01",
            "contractStartDate": "2023-01-15",
            "incorportationStatus": "3",
            "primaryListingReason": "21-Excessive Fraud",
            "principals": [
              {
                "name": "AHMAD ALI"
              }
            ],
            "tradeInternationally": false,
            "tradeOverInternet": true,
            "webAddresses": [
              "https://example.com"
            ],
            "acquirerAssignedMerchantID": "EXAMPLE001"
          },
          "terminatedRecordMatch": {
            "DBAName": "E",
            "addressMatch": {
              "city": "E",
              "streetAddress": "P",
              "countryOrRegion": "E"
            },
            "principalsMatch": [
              {
                "name": "E"
              }
            ]
          }
        }
      ]
    }
  }
}
```

## 6. How to use the result

1. Confirm HTTP `200` and `status.statusCode = "API000"`.
2. Save `searchTerminatedResponse.searchRequestRef.searchRequestRefID`.
3. Review `possibleMatches.matchedRecords` and termination reasons through the bank's risk process.
4. Continue onboarding checks after a completed no-match search. Keep failed or incomplete checks pending/error.

| HTTP status | Application code | Description |
| --- | --- | --- |
| `200` | `API000` | Success. |
| `400` | `API001` | Missing or invalid fields. |
| `401` | `API004` | Authentication failure. |
| `403` | `API007` | Access denied. |
| `404` | `API002` | Resource/data not found; do not assume a clean screening result. |
| `405` | `API008` | Incorrect HTTP method. |
| `406` | `API009` | Unacceptable media type. |
| `500` | `API003` | Processing failure. |

## 7. Official documentation

- [VMSS API reference — complete fields and validation rules](https://developer.visa.com/api-reference/visa-merchant-screening-service)
- [VMSS getting started](https://developer.visa.com/capabilities/visa-merchant-screening-service/docs-getting-started)
- [Visa mutual TLS and Basic authentication setup](https://developer.visa.com/docs/authentication-methods/two-way-ssl)

Examples are illustrative; no authenticated API calls were performed. Based on the published VMSS v2 search contract inspected on 22 October 2026.
