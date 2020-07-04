# EQ Bank Mobile API

## Base Information
The base URI of the API is `https://mobile-api.eqbank.ca/mobile/v1.1/`

Parameters are sent via a combination of headers, JSON-encoded POST body, and the query string.
Responses are JSON.

Every request includes an authorization and correlation header:

| Header | Value | Description |
| - | - | - |
| Authorization | Basic Njdj...BZjU= | This doesn't appear to change but I don't know if it's deduced from personal information so I have redacted it. |
| correlationId | Generated UUID Version 4 | |

#### Rate Limiting
Rate limiting information is available in the following headers: `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `X-RateLimit-Limit`

## Authentication Flow
### 1. Check for Updates

##### Request
```
GET /compliance/clientApp
```

| Source | Parameter Name | Example | Required | Description |
| - | - | - | - | - |
| Header | clientVersion | 2.2.0 | No | The version of the application running. If it is too outdated, the response will signal that an update is required. |
| Header | clientOS | android | Yes | The operating system of the client. |

##### Response
If the client version is acceptable,
```json
{
  "isForceUpdateRequired": false
}
```
Otherwise,
```json
{
  "isForceUpdateRequired": true,
  "appId": "com.eqbank",
  "downloadURL": "https://play.google.com/store/apps/details?id=com.eqbank.eqbank"
}
```

### 2. Attempt to Log In

##### Request
```
POST /login
```

| Source | Parameter Name | Example | Required | Description |
| - | - | - | - | - |
| Header | clientVersion | 2.2.0 | No | The version of the application running. |
| Header | clientOS | android | No | The operating system of the client. |
| JSON Body | TMSessionId | ffffffffffffffffffffffffffffffff | Yes | UUID Version 4 with no dashes |
| JSON Body | email | me@example.com | Yes | The email of the account |
| JSON Body | password | password | Yes | The password to the account |

##### Response
If the credentials are wrong,
```json
{
  "error": {
    "code": "-1_2_401_1",
    "variant": "AUTHENTICATION_ERROR",
    "message": "The email and password you entered was incorrect. Please double-check and try again, or sign up."
  }
}
```

If the credentials are accepted, the response will be an _access granted_ response, which you can see at the end of the Stepup Section.

If the credentials are accepted, but further authentication is required, then `isStepupRequired` will be `true` and `stepupConfiguration` will contain the relevant details:
```json
{
  "sessionReferenceId": "WTew3F3...dv26tYf",
  "isStepupRequired": true,
  "stepupConfiguration": {
    "challengedQuestion": "What is your favourite animal?",
    "questionCode": "QA_107",
    "stepupType": "CHALLENGED_QUESTION"
  },
  "isTMSessionDetected": false,
  "features": [
    {
      "feature": "transferwise",
      "enabled": true
    },
    {
      "feature": "automatedReferrals",
      "enabled": true
    }
  ]
}
```
If a security question is required, `isStepupRequired` will be `true` and the security question details will be in `stepupConfiguration`.

### 3. Stepup
##### Request
```
PUT /login/stepup
```

| Source | Parameter Name | Example | Required | Description |
| - | - | - | - | - |
| JSON Body | email | me@example.com | Yes | The email of the account |
| JSON Body | sessionReferenceId | WTew3F3...dv26tYf | Yes | `sessionReferenceId` from the stepup response in step 2 |
| JSON Body | stepupConfiguration.questionAnswer | sheep | Yes | The answer to the question posed in `stepupConfiguration.challengedQuestion` |
| JSON Body | stepupConfiguration.questionCode | QA_107 | Yes | `stepupConfiguration.questionCode` |
| JSON Body | steupConfiguration.trustDevice | true | Yes | Boolean whether to ask security questions for this device in a future login |

#### Response
```json
{
    "accessToken": "eyJhbG...ykrczKdYY",
    "data": {
        "dashboard": {
            "accounts": {
                "GIC": [],
                "HISA": [
                    {
                        "accountName": "My HISA",
                        "accountNumber": "100000000",
                        "accountOpeningMonthYear": "072020",
                        "arrangementId": "AA202070VGG2",
                        "availableBalance": 0,
                        "currentBalance": 0,
                        "goalAmount": 0,
                        "goalStatus": "INACTIVE",
                        "postingRestriction": false,
                        "rate": 2
                    }
                ],
                "Joint": [],
                "availableBalance": 0,
                "totalBalance": 0
            }
        },
        "loginDetails": {
            "customerId": "000000",
            "customerName": "John Doe",
            "lastSignInDate": "12:36 PM ET - 3 Jul 2020"
        },
        "sequenceNumber": 420
    },
    "features": [
        {
            "enabled": true,
            "feature": "transferwise"
        },
        {
            "enabled": true,
            "feature": "automatedReferrals"
        }
    ],
    "isReviewRequired": false,
    "isStepupRequired": false
}
```

### 4. Authenticated Request
Authenticated requests require the following headers:

| Header | Example | Description |
| - | - | - |
| accessToken | eyJhbG...ykrczKdYY | The access token from the response in step 3 |
| email | me@example.com | The email of the account |

## Data Endpoints

### Customer Information
```
GET /customer
```

```json
{
    "address1": "123 Test Street NW",
    "address2": null,
    "city": "Town",
    "email": "ME@EXAMPLE.COM",
    "firstName": "John",
    "lastName": "Doe",
    "middleName": null,
    "mobileNumber": null,
    "occupation": "Software Engineer - Information Technology and Communications",
    "otherPhone": "11234567890",
    "postalCode": "A1B2C3",
    "province": "BC"
}
```

### Dashboard
```
GET /dashboard
```

```json
{
    "accounts": {
        "GIC": [],
        "HISA": [
            {
                "accountName": "My HISA",
                "accountNumber": "100000000",
                "accountOpeningMonthYear": "072020",
                "arrangementId": "AA202070AGG2",
                "availableBalance": 0,
                "currentBalance": 0,
                "goalAmount": 0,
                "goalStatus": "INACTIVE",
                "postingRestriction": false,
                "rate": 2
            }
        ],
        "Joint": [],
        "availableBalance": 0,
        "totalBalance": 0
    }
}
```

### Transferwise Dashboard
```
GET /transferwise/dashboard
```
```json
{
    "error": {
        "code": "0_2_13_1",
        "message": "User doesn't have a linked TransferWise account with EQ Bank.",
        "variant": "VALIDATION_ERROR"
    }
}

```

### Messages
```
GET /messages
```
```json
[]
```

### Account Invitation
##### Incoming
```
GET /hisa/jointAcc/invitation/incoming
```
```json
[]
```
##### Outgoing
```
GET /hisa/jointAcc/invitation/outgoing
```
```json
[]
```

### Upcoming Transactions
```
GET /payment/upcoming
```
| Source | Name | Required | Example | Description |
| - | - | - | - | - |
| Query String | pageNo | ? | 0 | The results page to get |
| Query String | individualAccountFlag | ? | false | ? |
| Query String | fromCache | ? | false | Bypass results cache |
| Header | listAccounts | ? | 100000000 | An account number |

```json
{
    "totalNoPages": 0,
    "upcomingTransactions": []
}
```

### Recent Transactions
```
GET /transaction/recent
```
| Source | Name | Required | Example | Description |
| - | - | - | - | - |
| Query String | noTxns | ? | 0 | The results page to get |
| Header | accountId | ? | 100000000 | An account number |

```json
{
    "listEnd": false,
    "transactions": [
        {
            "amount": 100.0,
            "balance": 100.0,
            "date": "2020-07-03",
            "description": "Transaction Description",
            "type": "CREDIT"
        }
    ]
}
```

### Pending Account Links
```
GET /eft/beneficiary/pending
```
```json
[]
```

### Account Links
```
GET /eft/beneficiary
```
```json
[
    {
        "accountNo": "123456",
        "beneficiaryId": "BEN202050ST0D",
        "institutionNo": "004",
        "nickname": "TD",
        "transitNo": "12345"
    }
]
```

### Transferwise Currencies
```
GET /transferwise/currencies
```
```json
{
    "sourceCurrencyList": [
        {
            "code": "CAD",
            "country": "Canada",
            "disabled": false,
            "name": "Canadian dollar",
            "popular": false
        }
    ],
    "targetCurrencyList": [
        {
            "code": "AUD",
            "country": "Australia, Christmas Island, Cocos (Keeling) Islands, Heard Island and McDonald Islands, Kiribati,
Nauru, Norfolk Island, Tuvalu",
            "disabled": false,
            "name": "Australian dollar",
            "popular": false
        }
    ]
}
```
