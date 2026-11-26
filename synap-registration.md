# Synap API – Register a Candidate and Assign an Exam

This documents the current tested approach for creating a candidate in Synap, assigning them to an exam and retrieving their unique exam link.

The process uses one POST request to register the candidate and assign them to the appropriate user group, followed by one GET request to retrieve the automatically generated exam attempt.

## 1. Register the candidate and assign the exam

**Endpoint**

```http
POST https://api.synap.ac/v3/user-registrations
```

**Headers**

```http
Authorization: Bearer YOUR_SYNAP_API_TOKEN
Accept: application/vnd.api+json
Content-Type: application/vnd.api+json
```

**Request body**

```json
{
  "data": {
    "type": "UserRegistration",
    "attributes": {
      "name": "Jane Smith",
      "emailAddress": "jane.smith@example.com",
      "password": "TempPass123!",
      "welcomeEmailDisabled": true,
      "locale": "en-GB",
      "externalId": "joomla-identity-12345"
    },
    "relationships": {
      "userGroups": {
        "data": [
          {
            "type": "UserGroup",
            "id": "doPmofQ9k3l"
          }
        ]
      }
    }
  }
}
```

### Request fields

| Field | Purpose |
|---|---|
| `name` | Candidate's full name |
| `emailAddress` | Candidate's email address |
| `password` | Generated temporary Synap password |
| `welcomeEmailDisabled` | Prevents Synap sending its own welcome email |
| `locale` | Candidate's selected language/locale |
| `externalId` | Joomla Identity account identifier |
| `userGroups` | Assigns the candidate to the relevant exam user group |

The `userGroups` relationship is important. Assigning the candidate to the relevant user group automatically generates their exam attempt. We do not need a separate POST request to create the attempt.

The user group ID in this example (`doPmofQ9k3l`) is from our testing. Production will use the appropriate user group ID for the selected exam.

### Password

For now, Joomla will generate a temporary password and include it in the candidate's email alongside their exam link.

The password used during testing needed at least eight characters, including uppercase, lowercase and a digit.

We have not yet finalised whether Synap can provide passwordless access through a secure, time-limited link.

### Welcome email

`welcomeEmailDisabled: true` prevents Synap from sending its own welcome email. Joomla will handle all candidate communications.

### Registration source

We deliberately omit `registrationSource`. Synap confirmed that this field is optional and defaults to `generated`.

## 2. Retrieve the generated exam attempt

The registration response does not reliably include the generated attempt ID.

We therefore use the Synap user ID returned by the registration request to retrieve the candidate's attempt.

**Endpoint**

```http
GET https://api.synap.ac/v3/attempts?filter={"owner":"SYNAP_USER_ID"}
```

**Headers**

```http
Authorization: Bearer YOUR_SYNAP_API_TOKEN
Accept: application/vnd.api+json
```

Replace `SYNAP_USER_ID` with the ID of the newly registered candidate returned by the POST request.

The response contains the exam attempt, including its unique attempt ID.

## 3. Construct the personalised exam link

Once we have the attempt ID, we construct the candidate's personalised exam URL.

```text
https://joomla.synap.ac/exam-hall/ATTEMPT_ID
```

Replace `ATTEMPT_ID` with the ID returned by the GET request.

The portal hostname above is illustrative and should be replaced with the actual Synap portal hostname.

We tested this process with a real candidate and confirmed that the link opened the correct exam. Synap login was still required.

## 4. Email the candidate

Joomla sends the candidate an email containing:

- Their personalised exam link.
- Their Synap username (email address).
- Their generated temporary password.
- Any relevant instructions for taking the exam.

Synap's own welcome email remains disabled.

## 5. Overall workflow

```text
Joomla receives confirmed registration/payment
                  |
                  v
Generate temporary Synap password
                  |
                  v
POST /v3/user-registrations
                  |
                  | Creates candidate
                  | Assigns exam user group
                  | Automatically generates exam attempt
                  v
Receive Synap user ID
                  |
                  v
GET /v3/attempts?filter={"owner":"USER_ID"}
                  |
                  | Retrieves generated attempt
                  v
Extract attempt ID
                  |
                  v
Construct personalised exam URL
                  |
                  v
Joomla emails candidate:
  - Exam link
  - Username
  - Temporary password
```

## 6. Current implementation decision

The current agreed approach is:

1. Joomla creates the candidate using `POST /v3/user-registrations`.
2. The same POST assigns the candidate to the appropriate exam user group.
3. Synap automatically generates the exam attempt.
4. Joomla retrieves the attempt using `GET /v3/attempts`, filtering by the candidate's Synap user ID.
5. Joomla constructs the personalised exam URL from the attempt ID.
6. Joomla emails the candidate their exam link, username and temporary password.

**Important:** We have confirmed that candidate creation and exam assignment can be handled in a single POST. We have not confirmed a single API call that also returns the unique exam URL.

The only outstanding question in this part of the integration is whether Synap can support passwordless access, allowing candidates to follow their personalised exam link without separately entering a Synap username and password.
