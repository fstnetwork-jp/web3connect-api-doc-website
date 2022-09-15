---
title: Web3Connect RESTful API Doc

language_tabs: # must be one of https://git.io/vQNgJ
  - shell
  - javascript

toc_footers:
  - <a href='https://github.com/slatedocs/slate'>Documentation Powered by Slate</a>

includes:
  - errors

search: true

code_clipboard: true

meta:
  - name: description
    content: Documentation for the Web3Connect RESTful API for Organization Accounts
---

# Overview

## HTTP End Point (Base URL)

```
https://dev.web3connect.jp/rest/v1
```

## HTTP Headers required in the requests

> Especially for the APIs that need to know the identity of the client (requester)

```
Authorization: Bearer <ACCESS_TOKEN>
```

> ACCESS_TOKEN is in JWT format.

## `ACCESS_TOKEN` v.s. `API_KEY_SECRET`

`ACCESS_TOKEN` is used for common APIs, which must be used in `Authorization` header in HTTP request.

`API_KEY_SECRET` is used for critical APIs, such as refreshing `ACCESS_TOKEN` and generating one-time-use javascript code.

`API_KEY_SECRET` is never used in the header in HTTP request.

`API_KEY_SECRET` can be obtained from Web3Connect, and it should be stored in a safe place by the organization.

**DO NOT USE API KEY SECRET IN PUBLIC ZONE OR ANY FRONTEND.**

## Common Response Body Format

```js
{
  "status": "ok", // "error"
  "result": {}, // null, Object, Array, Boolean, String or Number
  "error": null // null, Error Message in String or Error Message in Object
}
```

# Auth API

> **Please obtain your API Key Secret from Web3Connect beforehand.**

An API Key Secret can be used to obtain the access token and confirmation token.

## Refresh (Obtain) Access Token with API Key Secret

- `POST`
- `/auth/account/access_token/refresh`
- **No Authorization Header required**

> Request Body JSON Sample (`Content-Type: application/json`)

```js
{
  "api_key_secret": "<SECRET>"
}
```

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": "<ACCESS_TOKEN>",
  "error": null
}
```

# Account Association & End-user Import

## Generate Web3Connect Button OnClick JavaScript function code body

> **This API must be called in the backend, but not in the frontend.**

- `POST`
- `/auth/account/association/generate_web3connect_button_js`
- **No Authorization Header required**

> Request Body JSON Sample (`Content-Type: application/json`)

```js
{
  "api_key_secret": "<SECRET>" // YOUR API KEY SECRET,
  "redirect_url": "https://company.com/abc/def?g=h&userId=xxx" // For the redirection after the end-user finishes the association process
}
```

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": "function globalWeb3ConnectButtonOnClick() {...}", // JavaScript function code body
  "error": null
}
```

In `redirect_url` of the request body, the organization should include the identifier of the end-user in order to associate the organization's end-user with an ID Wallet Address in the organization's internal system.

For example, `userId=xxx` in the redirect url's query param (any query param is allowed for the organization).

1. Embed the JavaScript code in to the organization's web page

The organization needs to embed the function code body in the `<script>...</script>` section in the header of the web page,  
and bind the onclick event to the button like :

```html
<button onclick="event.preventDefault();globalWeb3ConnectButtonOnClick();">
  Web3Connect
</button>
```

2. Let the end-user walk through the association process

Once the end-user clicks the button, the end-user will walk through the association process in a newly opened web page (association page), and Web3Connect backend will associate the end-user account (will be created if needed) with the organization account (the reason why this API needs the `API_KEY_SECRET`).

3. The organization process the callback http request (`redirect_url`)

And finally, a url (the adjusted redirect url) :

```
GET
https://company.com/abc/def?g=h&userId=xxx&id_wallet_address=0x1234567890123456789012345678901234567890
```

will be opened by the end-user's web browser (a query param `id_wallet_address` is appended to the url) at the end of the association process, the organization should process this HTTP request to finish the organization's internal system process.

4. The organization should store the `id_wallet_address` of the end-user

The organization will need to store the `id_wallet_address` of the end-user inside their database.

> Through this process, the organization will only know the `id_wallet_address` of an end-user, but not its email.

> An ID Wallet will be automatically created if the end-user's info does not exist in Web3Connect.

## Import Existing End-user with its email (Not recommended)

> This implies that the end-user already knows which email is used for login.

- `POST`
- `/account/import`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Request Body JSON Sample (`Content-Type: application/json`)

```js
{
  "email": "abc@def.com"
}
```

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": {
    "email": "abc@def.com",
    "id_wallet_address": "0x...."
  }
  "error": null
}
```

# Verify API (for Organization Account)

Web3Connect will support the verification process during the creation of the organization account.

## Verify Phone Step 1

> This will send a code to the mobile.

- `POST`
- `/verify/phone/step1`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Request Body JSON Sample (`Content-Type: application/json`)

```js
{
  "phone_number": "+8860912345678" // a valid mobile phone number in E.164 format
}
```

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": {
    "tmp_token": "token string", // this is critical for step 2
    "code_sent": true
  },
  "error": null
}
```

## Verify Phone Step 2

> **Please take the `tmp_token` from the response in step 1.**

- `POST`
- `/verify/phone/step2`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Request Body JSON Sample (`Content-Type: application/json`)

```js
{
  "tmp_token": "token string", // from step 1
  "code": "123456" // the verification code received by the phone
}
```

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": true, // phone verified
  "error": null
}
```

## Verify Domain Step 1

> **Only Organization Account can perform this action.**

> `domain` in the response is from the value during the sign-up process of the Organization Account.

> Please set the value of `txt_record` under the domain's TXT record (the domain is already set during the sign-up process)

- `GET`
- `/verify/domain/step1`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": {
    "txt_record": "web3connect-domain-verification=XXXXXXXXXX",
    "domain": "company.com"
  },
  "error": null
}
```

## Verify Domain Step 2

> **Only Organization Account can perform this action.**

> This step checks the txt record of the domain.

> Please set the value of `txt_record` under the domain's TXT record in step 1.

- `GET`
- `/verify/domain/step2`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": true, // domain verified
  "error": null
}
```

# Account API

## Get Public Account Info by ID Wallet Address

> Please replace the `<id_wallet_address>` section in the url with the ID Wallet Address (e.g. `0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaabcdef`)

- `GET`
- `/account/id_wallet/<id_wallet_address>/public_info`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": {
    "id_wallet_address": "0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaabcdef",
    "id_wallet_public_key_jwk": {
      "kty": "EC",
      "crv": "secp256k1",
      "x": "base64url string",
      "y": "base64url string"
    }
  },
  "error": null
}
```

## Get My Info

- `GET`
- `/account/my_info`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": {
    "account_id": "6aecfd05-4de5-31c3-052c-ce3dae83057d",
    "oidc_issuer_url": null,
    "sub": "6aecfd05-4de5-31c3-052c-ce3dae83057d",
    "name": null,
    "given_name": null,
    "family_name": null,
    "middle_name": null,
    "nickname": null,
    "preferred_username": null,
    "profile": null,
    "picture": null,
    "website": null,
    "email": "My Email",
    "email_verified": true,
    "gender": null,
    "birthdate": null,
    "zoneinfo": null,
    "locale": null,
    "phone_number": null,
    "phone_number_verified": false,
    "address": null,
    "account_type": "end_user",
    "domain": null,
    "domain_verified": false,
    "id_wallet": {
      "address": "0x5d2c54024c8f9d58059da8c47ba7e5bbdab6b249",
      "jwk_encrypted": {
        "kty": "EC",
        "crv": "secp256k1",
        "x": "zP1RAll6_fSP83YulysNF3L3YVef6DGa0d0Dlh794K4",
        "y": "ON2BME5Sptnptg970Bnuz4qk9tZqSfaHuHg2W-jaKLE"
      },
      "owner": "62da2c56ec8145256f37f3fc",
      "created_date_string": "2022-07-22T04:49:26.105Z",
      "created_at": "2022-07-22T04:49:26.134Z",
      "updated_at": "2022-07-22T04:49:26.134Z",
    },
    "created_at": "2022-07-22T04:49:26.125Z",
    "updated_at": "2022-07-22T04:49:26.125Z",
  },
  "error": null
}
```

# NFT API

## Mint NFT by Organization Account

> **Only Organization Account can perform this action.**

> Organization Account's domain must be verified beforehand.

> Please replace `<holder_id_wallet_address>` section in the url to the id_wallet_address of the end-user account (who will become the holder of the newly minted nft).  
> (e.g. `0x1234567890123456789012345678901234567890`)

- `POST`
- `/nft/mint/to/<holder_id_wallet_address>`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Request Body JSON Sample (`Content-Type: application/json`)

```js
{
  "metadata": {
    "static": {
      // all required, except "expired_at"
      "issuer": "string",
      "contents_provider": "name",
      "name": "name",
      "description": "text",
      "image": "url",
      "link": "url",
      "serial_number": 123, // integer
      "total_issued": 321, // integer
      "animation_url": "url",
      "expired_at": "2030-12-31T12:34:56.000Z" // optional, format is Date ISO String, by default it is 9999-12-31T23:59:59.000Z
    },
    "dynamic": {
      // all optional
      "external_url": "string",
      "youtube_url": "string",
      "background_color": "string",
      "category": "string",
      "id": "string",
      "points_earned": 12, // number
      "unit_of_points": "string",
      "program_name": "string",
      "organization": "string",
      "brand": "string",
      "branch": "string",
      "department": "string",
      "address": "string",
      "sns1": "string",
      "sns2": "string",
      "sns3": "string",
      "role": "string",
      "numbered_position": 1, // integer
      "email": "string",
      "tel": "string",
      "reference_name": "string",
      "license": "string",
      "competency": "string",
      "credential_category": "string",
      "level": "string",
      "status": "string",
      "date_created": 1784576401, // integer
      "date_modified": 1784576401, // integer
      "expires": 1784576401, // integer
      "valid_latitude": "string",
      "valid_longitude": "string",
      "valid_radius": 5.2, // number
      "unit_of_length": "string",
      "ticketed_seat": "string",
      "total_value": 100, // number
      "consumable_value": 123, // number
      "price_currency": "string",
      "discount_rate": 10, // number
      "gtin": "string",
      "gtin12": "string",
      "gtin13": "string",
      "gtin14": "string",
      "gtin8": "string",
      "has_adult_consideration": "string",
      "rating": "string",
      "copyright_notice": "string",
      "copyright_year": 2022, // integer
      "credit_text": "string",
      "version": "string",
      "policy": "string",
      "keywords": "string",
      "attributes": [], // array
      "properties": {}, // object
      "localization": {} // object
      // and any other keys and values
    }
  }
}
```

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": {
    "token_id": "string",
    "token_uri": "string",
    "minter_digital_signature": "string",
    "created_at": "2022-12-31T12:34:56.000Z",
    "metadata": {
      "static": {
        "issuer": "string",
        "contents_provider": "name",
        "name": "name",
        "description": "text",
        "image": "url",
        "link": "url",
        "serial_number": 123, // integer
        "total_issued": 321, // integer
        "animation_url": "url"
      },
      "dynamic": {
        "external_url": "string",
        "youtube_url": "string",
        "background_color": "string",
        "category": "string",
        "id": "string",
        "points_earned": 12, // number
        "unit_of_points": "string",
        "program_name": "string",
        "organization": "string",
        "brand": "string",
        "branch": "string",
        "department": "string",
        "address": "string",
        "sns1": "string",
        "sns2": "string",
        "sns3": "string",
        "role": "string",
        "numbered_position": 1, // integer
        "email": "string",
        "tel": "string",
        "reference_name": "string",
        "license": "string",
        "competency": "string",
        "credential_category": "string",
        "level": "string",
        "status": "string",
        "date_created": 1784576401, // integer
        "date_modified": 1784576401, // integer
        "expires": 1784576401, // integer
        "valid_latitude": "string",
        "valid_longitude": "string",
        "valid_radius": 5.2, // number
        "unit_of_length": "string",
        "ticketed_seat": "string",
        "total_value": 100, // number
        "consumable_value": 123, // number
        "price_currency": "string",
        "discount_rate": 10, // number
        "gtin": "string",
        "gtin12": "string",
        "gtin13": "string",
        "gtin14": "string",
        "gtin8": "string",
        "has_adult_consideration": "string",
        "rating": "string",
        "copyright_notice": "string",
        "copyright_year": 2022, // integer
        "credit_text": "string",
        "version": "string",
        "policy": "string",
        "keywords": "string",
        "attributes": [], // array
        "properties": {}, // object
        "localization": {} // object
        // and other properties
      }
    }
  },
  "error": null
}
```

## Fetch NFT info

> If the NFT's visibility is `private`, then only the holder or minter can check it.

> If the NFT's visibility is `public`, then any account can check it.

> Please replace `<token_id>` section in the url with the real value (a hex string, e.g. `0xabcdef123123123123...`).

> Please check the response body json in [Mint NFT by Organization Account](#mint-nft-by-organization-account) for the schema of the NFT object.

- `GET`
- `/nft/<token_id>`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": {}, // the NFT object, please refer to the response of mint nft
  "error": null
}
```

## List NFTs being held by an ID Wallet (End-user Account)

> Please replace the value `<ID_WALLET_ADDRESS>` to the actual one in the url.

> `perPageItemNum` query param is for setting how many NFT objects per page. Default value is `10`.

> `pageNum` query param starts from `1`, for listing the NFT objects of the n-th page. Default value is `1`.

> `order` query param is for controlling sort order based on `created_at` of the NFTs, it can be `desc` or `asc`, default value is `asc`.

> Please check the response body json in [Mint NFT by Organization Account](#mint-nft-by-organization-account) for the schema of the NFT object.

- `GET`
- `/nft/held_by_id_wallet/<ID_WALLET_ADDRESS>?perPageItemNum=10&pageNum=1&order=asc`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": [{}, {}, {}, {}, {}, {}, {}, {}, {}, {}], // the Array of NFT object. For each NFT object, please refer to the response of mint nft
  "error": null
}
```

**Only the NFTs visible by the requester (http client) will be returned.**

## List NFTs minted by Organization Account (Minter)

> Only the minter (organization account) can perform this action.

> `perPageItemNum` query param is for setting how many NFT objects per page.

> `pageNum` query param starts from `1`, for listing the NFT objects of the n-th page.

> `order` query param is for controlling sort order based on `created_at` of the NFTs, it can be `desc` or `asc`, default value is `asc`.

> Please check the response body json in [Mint NFT by Organization Account](#mint-nft-by-organization-account) for the schema of the NFT object.

- `GET`
- `/nft/minted_by_me?perPageItemNum=10&pageNum=1&order=asc`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": [{}, {}, {}, {}, {}, {}, {}, {}, {}, {}], // the Array of NFT object. For each NFT object, please refer to the response of mint nft
  "error": null
}
```

## Toggle NFT visibility

> **Only the holder (end-user account) and the minter (organization account) can perform this action**.

> Please replace `<token_id>` section in the url with the real value (a hex string, e.g. `0xabcdef123123123123...`).

- `PUT`
- `/nft/<token_id>/visibility`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Request Body JSON Sample (`Content-Type: application/json`)

```js
{
  "to": "public" // or "private"
}
```

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": "public", // or "private"
  "error": null
}
```

## Search Public NFTs by field-matching search

> This can only find public NFTs.

> `perPageItemNum` query param is for setting how many NFT objects per page.

> `pageNum` query param starts from `1`, for listing the NFT objects of the n-th page.

> `order` query param is for controlling sort order based on `created_at` of the NFTs, it can be `desc` or `asc`, default value is `asc`.

> Please check the response body json in [Mint NFT by Organization Account](#mint-nft-by-organization-account) for the schema of the NFT object.

- `POST`
- `/nft/search?perPageItemNum=10&pageNum=1&order=asc`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Request Body JSON Sample (`Content-Type: application/json`)

```js
// each field can be set to undefined for ignoring the field matching
{
  "issuer": "string match value",
  "contents_provider": "string match value",
  "name": "string match value",
  "description": "string match value",
  "category": "string match value",
  "program_name": "string match value",
  "organization": "string match value",
  "brand": "string match value",
  "competency": "string match value",
  "credential_category": "string match value",
  "level": "string match value",
  "status": "string match value",
  "gtin": "string match value",
  "gtin12": "string match value",
  "gtin13": "string match value",
  "gtin14": "string match value",
  "gtin8": "string match value",
  "has_adult_consideration": "string match value",
  "rating": "string match value",
  "keywords": "string match value"
}
```

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": [{}, {}, {}, {}, {}, {}, {}, {}, {}, {}], // the Array of NFT object. For each NFT object, please refer to the response of mint nft
  "error": null
}
```

## Search NFTs minted by me (Organization Account) by field-matching-search

- `POST`
- `/nft/minted_by_me/search?perPageItemNum=10&pageNum=1&order=asc`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Please refer to the previous one for the request/response body format.

## Generate QR Code for NFT verification by the holder (NFT Verification Step 1)

> **Only the holder (end-user account) can perform this action**.

> Please replace `<token_id>` section in the url with the real value (a hex string, e.g. `0xabcdef123123123123...`).

- `POST`
- `/nft/<token_id>/verification/qr_code/step1`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> The holder only needs to post the current gps location of its device to the web3connect api.

> The QR Code in the result is base64 encoded image.

> The QR Code will be only available for next 5 minutes, the scanner must scan this QR Code ASAP.

> Request Body JSON Sample (`Content-Type: application/json`)

```js
{
  // this should be the current location of the holder
  "gps": {
    "latitude": "26.583193",
    "longitude": "128.053881"
  }
}
```

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": {
    "code": "1234567890",
    "qr_code_base64": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAABAgAAAQICAYAAACAmekaAAAAAXNSR0IArs4c6QAAIABJREFUeF7snUGW3TgOBF33P7TnTXvtLz5nRQFJxaxFKBlIgBT8u+br9+/fv3/5PwlIQAISkIAEJCABCUhAAhKQgAReTeDLAcGr8+/mJSABCUhAAhKQgAQkIAEJSEAC/xFwQKARJCABCUhAAhKQgAQkIAEJSEACEnBAoAckIAEJSEACEpCABCQgAQlIQAIS8BcEekACEpCABCQgAQlIQAISkIAEJCAB/xMDPSABCUhAAhKQgAQkIAEJSEACEpDA/wn4Nwj0gQQkIAEJSEACEpCABCQgAQlIQAIOCPSABCQgAQlIQAISkIAEJCABCUhAAv6CQA9IQAISkIAEJCABCUhAAhKQgAQk4H9ioAckIAEJSEACEpCABCQgAQlIQAIS+D8B/waBPpCABCQgAQlIQAISkIAEJCABCUjAAYEekIAEJCABCUhAAhKQgAQkIAEJSMBfEOgBCUhAAhKQgAQkIAEJSEACEpCABPxPDPSABCQgAQlIQAISkIAEJCABCUhAAv8n4N8g0AcSkIAEJCABCUhAAhKQgAQkIAEJOCDQAxKQgAQkIAEJSEACEpCABCQgAQn4CwI9IAEJSEACEpCABCQgAQlIQAISkID/iYEekIAEJCABCUhAAhKQgAQkIAEJSOD/BPwbBPpAAhKQgAQkIAEJSEACEpCABCQgAQcEekACEpCABCQgAQlIQAISkIAEJCABf0GgByQgAQlIQAISkIAEJCABCUhAAhLwPzHQAxKQgAQkIAEJSEACEpCABCQgAQn8n4B/g0AfSEACEpCABCQgAQlIQAISkIAEJOCAQA9IQAISkIAEJCABCUhAAhKQgAQk8AO/IPj6+pKzBNYS+P3791ptJ8Lo+mrnc8IweYbmn2g7WdueX5o/zUf9Jy71mb8RoP3ZTp6ur3Y..."
  },
  "error": null
}
```

## Verify QR Code for NFT verification by the scanner (NFT Verification Step 2)

> The scanner scanned the qrcode.

> The query param **`CODE`** is exactly from the qrcode image, the decoded content of the qrcode image is this **`CODE`**.

> **And the requester (scanner) must add location information into the request body**

- `POST`
- `/nft/verification/qr_code/step2?c=<CODE>`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Request Body JSON Sample (`Content-Type: application/json`)

```js
// this should be the current location of the scanner
{
  "gps": {
    "latitude": "26.583193",
    "longitude": "128.053881"
  }
}
```

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": {
    "token_id": "<VERIFIED_NFT_TOKEN_ID>"
  }
  "error": null
}
```

## Poll NFT Verification result

> Put the `code` from the response of step1 into the query param

- `GET`
- `/nft/verification/qr_code/result?c=<CODE>`
- **Authorization Header required (`Authorization: Bearer <ACCESS_TOKEN>`)**

> Response Body JSON Sample (`Content-Type: application/json`)

```js
{
  "status": "ok",
  "result": {
    "token_id": "<VERIFIED_NFT_TOKEN_ID>" // null if not verified yet
  }
  "error": null
}
```
