
CheckPlease API
=================
CheckPlease is a SaaS platform for paperless onboarding and employment compliance.

We provide a simple, secure API for third party developers to use our platform to order and manage pre-employment checks, background checks and online forms, including:
- MoJ Check (NZ Criminal Record check) from the [NZ Ministry of Justice](https://www.justice.govt.nz/criminal-records/)
- Police Vetting Check
- Credit Check
- Qualifications Check
- ACC Claims History

Concepts
======
A **Check** represents a single check, such as an MoJ Check, IR330 Form, etc., performed on an **Individual**.

Some checks, like IR330, are a form that needs to be completed and signed.

Others, like MoJ Check, are complex workflows that move the form between parties including the individual, the customer, and an **Identity Verifier**.

Some checks require identity verification of the individual, which can be:
- less rigorous (for an MoJ Check, sighting their driver licence and comparing to their online signature); or
- more rigorous (for Police Vetting, 2 forms of ID, physical eyeball of the individual by an **Identity Referee**); or
- not required (for a Qualifications Check, the identity verification is made by the awarding institution.

A **Bundle** is a collection of checks, which will all share a single (or none) identity verification process on the individual.

Bundling checks makes for a much better candidate experience. The individual can complete all of their company onboarding paperwork steps with a single, cohesive, mobile-friendly branded web flow and sign online once.

This can replace a poor and insecure candidate experience of paper and online forms, ID photocopying, printing, emailing, scanning, etc., and a fragmented and potentially insecure identity verification process.

For example, an MoJ Check, a Credit Check and a Qualifications Check can be performed with a few web pages for the individual:
- grant consent to each of the 3 checks
- enter the **profile** data required by the 3 checks
- upload ID documentation
- sign online (once)

An individual's profile is made up of:
- their title, first, middle and last names, residential and postal addresses, contact details, gender
- previous names and addresses
- academic qualifications
- employment history
- NZ driver licence number
- any check-specific information, e.g. whether candidate wants postal copy for MoJ Checks.

When identity verification starts, a **profile snapshot** is recorded for audit purposes (i.e. this is the structured data that will be used to generate forms/check requests).

Once identity verification is complete, each of the checks on the bundle are completed and possibly used to start a workflow (e.g. with the Ministry of Justice for an MoJ Check).

For checks that have a result (e.g. MoJ Check), when the workflow produces a result, it is attached to the check.

At every stage, when the check or its bundle are changed, the system calls the webhook (if any).

API Endpoints
----------
The following endpoints are available:

| API        | Purpose   |
| -------- |--------|
| POST&nbsp;/checks | Order or update a bundle of checks on a new or existing individual. |
| GET&nbsp;/checks/byExternalRef/{externalRef} | Get the current state for a check and its bundle and individual.  |
| Your web hook | Be alerted when a check or its bundle is updated. You pass your web hook's url in when you create a check. |
| GET&nbsp;/checks/byClientKey/{clientKey}/request | Fetch the request pdf for a check. | 
| GET&nbsp;/checks/byClientkey/{clientKey}/result | Fetch the result pdf for a check. |
| GET&nbsp;/account | Fetch your account details. |
 
To use the CheckPlease API, you'll need a free account at https://www.checkplease.co.nz.
 
We're here to help you with your integration - for help using our API, email us at info@checkplease.co.nz.   


API authentication
----------
For APIs calls that you make to CheckPlease, include your account's API key in the Authorization header. You can get your API key from your Account settings page in CheckPlease.

For API calls that CheckPlease makes in to you (i.e. web hook calls), CheckPlease will attach your account's API key in the Authorization header. You should reject any incoming requests that don't have the API key present.

   
Ordering checks
====
Use the **POST /checks** API to order checks.

For example, to order 2 checks on Fred Flintstone - an MoJ Check with GOLD day priority and a Credit Check - from the command line, using curl:

````
curl -X POST https://api.checkplease.co.nz/api/checks \
-H 'Content-Type: application/json' \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK' \
-d @- << EOF
  {
    "individual": {
      "firstName": "Fred",
      "lastName": "Flintstone",
      "email": "fred@flintstone.com"
    },
    "checks": [
      {
        "type": "MOJ",
        "priority": "GOLD",
        "externalRef": "100233",
        "webHook": "https://myserver.com/checkWebHook"
      },
      {
        "type": "CREDITCHECK",
        "externalRef": "1033",
        "webHook": "https://myserver.com/checkWebHook2"
      }
    ]
  }
EOF

````
The system processes the request as follows.

First the system sets up the individual:
- If no externalRef or guid is passed for the individual, AND there is not an existing individual within the realm with matching first name, last name and email, then a new individual is created
- otherwise the existing individual will be used (but not updated, since the data is under the control of the individual via CheckPlease).

Next the system sets up the bundle:
-  If a bundle was passed, that bundle is used.
- Otherwise, if there is an existing bundle in ORDERED, I_DETAILS or BOUNCED or any I_% status, then that bundle is used. (In these statuses the individual still has a chance to edit their profile if the new check types require it and to provide consent).
- Otherwise a new bundle is created.

Finally the system creates or updates the checks, for each one:
- if externalRef was not passed, create a check
- otherwise find and update the matching check on the same bundle, or if none, create a check

The API is transactional, so if there is any failure, the system remains unchanged (e.g. we won't have an individual created, but with no attached bundle).

Once complete, the API response contains the same objects, but now with the server-assigned fields included for the check, it's bundle and for the individual:

** can we remove GUIDs? **

````
{
  "individual": {
    "guid": "225f03a6-0e1e-6a2c-ab08-ba9461a091ff",
    "firstName": "Fred",
    "lastName": "Flintstone",
    "email": "fred@flintstone.com",
    "previousNames": {},
    "residentialAddress": {},
    "previousEmployments": {},
    "academicQualifications": {},
    "nzDriverLicence": ""
  },
  "bundle": {
    "guid": "4e5f03a6-0e1e-6a2c-ab08-ba9461a09122",
    "reference": 46387621,
    "individualUrl": "https://personal.checkplease.co.nz/for/bb5f03a6-0e1e-6a2c-ab08-ba9461a09137",
    "status": "ORDERED",
    "statusLabel": "Sent to the individual",
    "statusImage": "https://example.com/imageblah?x=1",
    "msInStatus": 203331
  },
  "checks": [
    {
      "type": "MOJ",
      "priority": "GOLD",
      "canChangePriority": true,
      "externalRef": "100233",
      "clientKey": "bb5f03a6-0e1e-6a2c-ab08-ba9461a09137",
      "plan": "BUSINESS",
      "webHook": "https://myserver.com/checkWebHook",
      "eta": "2018-10-17T12:03:44.05Z",
      "status": "WHITE",
      "statusLabel": "New check",
      "statusImage": "https://example.com/imageblah?x=1",
      "msInStatus": "230435",
      "requestComplete": false,
      "resultComplete": false,
      "vendorReference": null
    },
    {
      "type": "CREDITCHECK",
      "priority": "GOLD",
      "canChangePriority": true,
      "externalRef": "1033",
      "clientKey": "595f03a6-0e1e-6a2c-ab08-ba9461a091e3",
      "plan": "BUSINESS",
      "webHook": "https://myserver.com/checkWebHook2",
      "status": "WHITE",
      "statusLabel": "New check",
      "statusImage": "https://example.com/imageblah?x=5",
      "msInStatus": "230435",
      "requestComplete": false,
      "resultComplete": false,
      "vendorReference": null
    }
  ]
}
````

The POST /checks endpoint can also be used to update the priority on MoJ Checks, e.g.:

````
curl -X POST https://api.checkplease.co.nz/api/checks \
-H 'Content-Type: application/json' \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK' \
-d @- << EOF
  {
    "checks": [
      {
        "externalRef": "100233",
        "priority": "SILVER"
      }
    ]
  }
EOF

````

CheckPlease may return one of the following http statuses:
- 200 if the operation was successful
- 400 if the operation could not be completed

400 errors may be due to:
- firstName or lastName are missing
- a bundle ID was specified, but that bundle already has a check of those types
- a bundle ID was specified, but that bundle is not in one of the statuses listed above or did not pull in enough profile data for the new check(s) to be performed.
- a check has an externalRef which is already in use by some other check, not on this bundle.
- the check's priority cannot be changed once it is sent to the MoJ

Getting the current state of a check
====
Use the **GET /checks/byExternalRef/{externalRef}** API to get a check's current state.

For example, to get the details for the MoJ Check created earlier, from the command line using curl:

````
curl https://api.checkplease.co.nz/api/mojChecks/byExternalRef/100233 \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK'
````
The response contains the check object, as shown above (in the response to POST /checks), along with its bundle and individual.

CheckPlease may return one of the following http statuses:
- 200 if the check was successfully fetched


Polling vs web hooks
-------
If at all possible, avoid calling this GET API repeatedly to see if your check has changed (i.e. polling). Instead, pass a web hook in when you create the check. Then your integration will be notified in real time as soon as your check changes.

Using a web hook instead of polling is:
- more real-time: your code will be instantly alerted when your checks change their state, and your integration will be always up to date
- more efficient: since checks tend to change slowly (e.g. many days may elapse before the MoJ provides the result), web hooks require far less network traffic than polling
- more reliable: your API calls will never fail due to rate limiting (the API is not tightly rate limited but may be in the future)

API schema
===========
Individual
-----
| Field        | Set by         | Mandatory? | Notes  |
| ------------- |:-------------:|:-----:|---|
| firstName | client | Y | First name for the individual (person having the check done on them).|
| lastName | client | Y ||
| email | client | Y ||
| middleNames | client | N | When there are multiple middle names then comma separate them. |
| nzDriverLicence | client | N | |
| residentialAddress | client | N | See Address object |
| postalAddress | client | N | See Address object |
| previousAddresses | client | N | Array of Address objects |
| previousNames | client | N | Array of Name object |

Bundle
-----
| Field        | Set by         | Mandatory? | Notes  |
| ------------- |:-------------:|:-----:|---|
| reference | server | Y | A unique 8 or more digit numeric reference code allocated by CheckPlease. |
| id | server | Y | A long generated by CheckPlease |
| individualKey | server | Y | A GUID automatically generated by CheckPlease that can be used to reach the individual's UI for the bundle. |
| status | server | Y | The stage the bundle is currently at (as opposed to the status of checks within the bundle).  |
| statusLabel | server | Y | A human-readable statement of the bundle's current status. |
| statusImage | server | Y | The url of an image that conveys the status visually. |
| msInStatus | server | Y | The number of millseconds that the bundle has been in its current status. |


Check
-----
| Field        | Set by         | Mandatory? | Notes  |
| ------------- |:-------------:|:-----:|---|
| type | client | Y | What type of check this is, i.e. one of: <ul><li>MOJ (NZ Criminal Record check)<li>EQUICREDIT (Credit check via Equifax)<li>DRIVER (Comprehensive Driver Check)<li>QUAL (Qualifications Check)<li>ACC (ACC Claim History)</ul> |
| priority | client | N | Priority of the check. The possible values depend on the check type. For MoJ Checks, the values are are: <ul><li>GOLD (3 working days turnaround)<li>SILVER (10 working days turnaround)<li>BRONZE (15 working days turnaround)</ul> If you don't provide a value for priority, then the check will be created using your CheckPlease account's default priority. |
| externalRef | client | N | The optional externalRef  field can be used to store your own reference (up to 512 characters) on the check. This is useful if you intend to update the check's priority after it has been created. |
| webHook | client | N | The optional webHook field can be used to store a url (up to 1024 characters) to your own web hook endpoint. This is useful if you want to be updated as the check progresses. |
| canChangePriority | server | Y | true if the check's priority can be changed with the PATCH /checks/byExternalRef/{ref} API. A check's priority can not usually be changed once payment is complete or the request has been submitted to the vendor. |    
| plan | server | Y | The plan field indicates the check's plan. This is copied from the account's plan when the check is created, and never subsequently updated. The possible values are:<ul><li>BUSINESS (CheckPlease handles verification, payment in advance with credit card)<li>CREDIT (CheckPlease handles verification, monthly invoicing)<li>MEMBER (Account owner provides credentials and handles verification, monthly invoicing)</ul> |
| clientKey | server | Y | Automatically generated by CheckPlease and can be used to fetch the result and request pdfs. |
| eta | server | Y | The date and time (as defined by [RFC 3339, section 5.6.](http://tools.ietf.org/html/rfc3339#section-5.6)) that the check is expected to be complete.<br /><br />This can change up until the request is made to the vendor, e.g. due to delays in verification.<br /><br />Null when the check is complete.|
| status | server | Y | The check is currently at (as opposed to the bundle's status).  |
| statusLabel | server | Y | A human-readable statement of the check's current status. |
| statusImage | server | Y | The url of an image that conveys the status visually. |
| msInStatus | server | Y | The number of millseconds that the check has been in its current status. |
| requestComplete | server | Y | true once the request pdf (i.e. the form that CheckPlease sends to the vendor) has been built (but not necessarily sent). The request is built once verification is complete. When true, you can fetch the request pdf with **GET /checks/requests/{clientKey}/request ???**. | 
| resultComplete | server | Y | true once the result pdf (i.e. the document that the vendor provides) has been received. | 
| vendorReference | server | N | The reference code (if any) allocated by the vendor once the request has been submitted to them - until then the field is null. |

Fetching the request and result documents
==========================
A check is performed by sending a pdf to the vendor (e.g. the Ministry of Justice for an MoJ Check). Later, the vendor sends back a result pdf. Both of these documents (when available) can be accessed via API.

Use the **GET /checks/byClientkey/{clientKey}/result** API to fetch the result pdf for a check.

Use the matching **GET /checks/byClientKey/{clientKey}/request** API to get the request pdf. 

You can obtain the client key from the check object.

For example, to stream the result pdf for the check with client key bb5f03a6-0e1e-6a2c-ab08-ba9461a09137:  

````
curl https://api.checkplease.co.nz/api/checks/byClientkey/bb5f03a6-0e1e-6a2c-ab08-ba9461a09137/result \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK'

````

Use the check's **requestComplete** (true or false) field to learn whether the request document is available.  

Use the check's **resultComplete** (true or false) field to learn whether the result document is available.

**Security:** For best security, don't save these documents, but just pass them through to the end user's browser. However accessing these links will return 400 unless the Authorization header includes the API key, so they can't be used directly from a browser. Instead, your app should present its own links for the documents to the end user (protected by your own authentication) and then proxy incoming requests to these APIs, with your API key in the Authorization header. This protects these sensitive documents from any unauthorized access. 


Using web hooks to track changes to your checks
====
When you create a check with a web hook specified, whenever the check's status is modified, CheckPlease will POST to your web hook endpoint, passing the updated check, bundle and individual in the request body.

This lets your server stay updated as the check progresses through each stage, eventually leading up to a result. For example, your server might change its own progress indicator on each incoming web hook call, to keep your end user informed.

The web hook is not called for the very first transition (i.e. when a check is created) - instead the full check is available to you in the response to your POST /api/checks API call.  

**Response status:** When your web hook endpoint succeeds, you should return http status 200. However CheckPlease currently only calls your web hook once per update, regardless of the response - there is no e.g. retry with exponential backoff if your endpoint returns a non-200 response.  

**Security:** CheckPlease attaches your API key in the Authorization header in its requests to your web hook. For security reasons, you should always check that the incoming Authorization header matches your own API key, otherwise an attacker with knowledge of your urls could theoretically call your server with false updates on your checks.  


Fetching your account details
===========
Use the **GET /account** API to fetch your account details.

This simple call does not modify CheckPlease, and is useful when testing connectivity, e.g. that you have the correct API key.
 
For example, to fetch your account details, from the command line using curl:

````
curl https://api.checkplease.co.nz/api/account \
-H 'Authorization: OmV81mPqcQ_5vb3R9UjTQPelTmcvfeAbgqO5ptkK'
````
The response contains your account details.

````
{
  "companyName": "Big Store Enterprises",
  "shortCode": "bigstore",
  "accountOwnerFirstName": "Stephanie",
  "accountOwnerLastName": "Higgins",
  "accountOwnerEmail": "stephanie.higgins@bigstore.com",
  "defaultMoJCheckPriority": "BRONZE",
  "demo": true,
  "plan": "BUSINESS"
}
````


Bundle statuses
==========
The following shows the statuses that a bundle can pass through as:
- the check is ordered
- the individual consents and provides their personal data, ID document and online signature
- the verifier checks the individual's ID document
- the check is possibly returned to the individual to remedy problems (e.g illegible ID document)
- eventually the identity is verified
 
Your web hook is called each time your check's bundle transitions to a new status (as well as when the individual is updated, e.g. takes their married name).

````
* Diagram does not show all possible transitions

        POST /checks triggers a bundle creation
             |
       +---------------+
       | ORDERED       |
       +---------------+
             |
       +-----|---------+   +----------+
       |     \/        |-->| DECLINED |
  +----->I_DETAILS     |   +----------+
  |    |     \/        |
  |    | I_DOCUMENT    | 
  |    |     \/      Individual
  |    | I_SIGNATURE  experience
  |    |     |         |      +----------
  |    |     |         |----->|
  |    +-----|---------+      | CANCELLED_BY_CLIENT
+----------+ |            +-->|
| BOUNCED  | |            |   +---------
+----------+ |            |
  /\         |            |
  |    +-----|-----------------+
  |    |    \/                 |
  |    | V_ID_TYPE             |
  +----|     \/                |   +--------+
       | V_ID_NAME             |-->| BAD_ID |
       |     \/                |   +--------+
       | V_DOB                 |
       |     \/                |
       | V_EXPIRY              |
       |     \/             Verify
       | V_SIGNATURE        experience
       |     \/                |
       | V_3RD_PARTY_SIGNATURE |
       |     |                 |
       +-----|-----------------+
             |
             \/
     +---------------+
     | VERIFIED      |
     +---------------+
```` 
