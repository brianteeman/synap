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

## 5. Overall registration workflow

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

## 6. Current registration implementation decision

The current agreed approach is:

1. Joomla creates the candidate using `POST /v3/user-registrations`.
2. The same POST assigns the candidate to the appropriate exam user group.
3. Synap automatically generates the exam attempt.
4. Joomla retrieves the attempt using `GET /v3/attempts`, filtering by the candidate's Synap user ID.
5. Joomla constructs the personalised exam URL from the attempt ID.
6. Joomla emails the candidate their exam link, username and temporary password.

**Important:** We have confirmed that candidate creation and exam assignment can be handled in a single POST. We have not confirmed a single API call that also returns the unique exam URL.

The only outstanding question in this part of the integration is whether Synap can support passwordless access, allowing candidates to follow their personalised exam link without separately entering a Synap username and password.

## 7. Monitor exam results using webhooks

Joomla monitors the candidate's exam progress using two separate Synap webhook subscriptions. Both subscriptions point to the same Joomla webhook endpoint, but they listen for different events.

The two events are:

| Synap event | Purpose |
|---|---|
| `Exam Attempt Submitted` (`exam.submitted`) | Notifies Joomla that the candidate has submitted the exam |
| `Exam Attempt Released` (`exam.released`) | Notifies Joomla that the results have been released and can be processed as final |

### Configure the two webhook subscriptions

In Synap, create two webhook subscriptions using the same Joomla endpoint URL:

1. Subscribe to **Exam Attempt Submitted**.
2. Subscribe to **Exam Attempt Released**.

Both subscriptions use HTTP Basic Authentication. Joomla's endpoint must validate the Basic Authentication credentials before accepting and processing a webhook.

The endpoint receives both event types. It uses `meta.eventName` in the payload to determine which event Synap has sent.

### Webhook 1: Exam Attempt Submitted

The submitted event has:

```json
{
  "meta": {
    "eventName": "exam.submitted"
  },
  "id": "ATTEMPT_ID"
}
```

The attempt ID is at the top level of this payload, in `id`.

This event means that the candidate has submitted their exam. It does **not** mean that the result is final or ready to be communicated as a pass or fail.

In testing, a submitted attempt could have values such as:

- `isCompleted: true`
- `examStatus: "submitted"`
- `resultsStatus: "pending"`
- `resultsMarkingStatus: "readyToFinalise"`
- `proctorRequiresReview: true`
- `proctorStatus: "readyForReview"`

A provisional score or grade may also be present. Joomla must not treat that as the final result.

When Joomla receives this event, it should record that the candidate has submitted the exam and retrieve the attempt from the Synap API if it needs the latest attempt details.

If the attempt requires proctor review, Joomla should notify the administrators and track the review. The candidate should receive an acknowledgement explaining that their result will be available within 24 hours. Joomla must wait for the released event before sending a pass or fail email.

### Webhook 2: Exam Attempt Released

The released event has a different payload structure:

```json
{
  "meta": {
    "eventName": "exam.released"
  },
  "attempt": {
    "id": "ATTEMPT_ID"
  }
}
```

The attempt ID is nested inside `attempt.id`, not at the top level.

This event indicates that Synap has released the result. Joomla can now retrieve the attempt details and process the final outcome.

### Always verify the result using the Synap API

The webhook is the notification that something has happened. Joomla should retrieve the attempt details from Synap and check the current result status before taking action.

The key check is:

```text
resultsStatus === "released"
```

Only when the API confirms that `resultsStatus` is `released` should Joomla treat the result as final and proceed with the pass/fail workflow.

Testing confirmed that an attempt can move through several states before release. For example, after submission it may be pending or awaiting proctor review. Following approval, it may be finalised but still not released. The result is ready for Joomla to process only when its status is `released`.

### Important differences between the two webhook payloads

The attempt ID is in a different place depending on the event:

| Event | Event name | Attempt ID location |
|---|---|---|
| Submitted | `exam.submitted` | `id` |
| Released | `exam.released` | `attempt.id` |

Joomla's webhook handler must account for this difference when extracting the attempt ID.

### Joomla result-processing workflow

```text
Candidate submits exam
          |
          v
Synap sends exam.submitted webhook
          |
          v
Joomla authenticates webhook
          |
          v
Extract attempt ID from payload.id
          |
          v
Retrieve attempt details from Synap API
          |
          v
Is proctor review required?
          |
      +---+---+
      |       |
     Yes      No
      |       |
      v       |
Notify admins |
Acknowledge   |
submission    |
to candidate  |
      |       |
Wait for      |
review        |
      |       |
      +---+---+
          |
          v
Synap releases result
          |
          v
Synap sends exam.released webhook
          |
          v
Joomla authenticates webhook
          |
          v
Extract attempt ID from payload.attempt.id
          |
          v
Retrieve attempt details from Synap API
          |
          v
Check resultsStatus === "released"
          |
          v
Process final pass/fail result
          |
          v
Send candidate result email
          |
          +--> If passed: issue Holopin badge
          |
          +--> If passed: publish to certified-user
               directory if candidate has opted in
```

### Result-processing rules

- Do not process a submitted event as a final result.
- Do not use a provisional score or grade to determine pass or fail.
- Do not assume that a finalised result has been released.
- Retrieve the attempt from Synap and verify `resultsStatus === "released"` before processing the result.
- Handle the two event payloads separately because the attempt ID is in a different location.
- Make result processing idempotent so that receiving the same notification more than once cannot send duplicate emails, issue duplicate badges or create duplicate directory entries.

### Testing observations

The two webhook subscriptions were tested against the same temporary PHP receiver.

Both event types were received successfully:

- `exam.submitted`
- `exam.released`

The submitted payload contained the attempt ID at the top level. The released payload contained it under `attempt.id`.

Testing also confirmed that a submitted attempt can be awaiting proctor review, and that an attempt can be finalised without yet being released. The released status is therefore the important condition for final result processing.

The webhook receiver used for these tests was temporary and is not the production Joomla handler.

### Items still to confirm

The following webhook behaviours have not yet been fully established:

- Synap's webhook retry policy when the receiving endpoint is unavailable or returns an error.
- Whether Synap provides a unique event identifier that Joomla can use for webhook-level deduplication.
- How Synap handles delivery order if events are delayed or arrive out of order.

Joomla should still make its own result-processing logic idempotent and verify the current attempt status through the API rather than relying solely on webhook delivery order.
