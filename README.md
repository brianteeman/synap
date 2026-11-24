# Synap – Joomla Integration

## Project overview

This repository documents a proposed integration between **Joomla**, **RSForm!Pro**, **Stripe** and the **Synap examination platform**.

The aim is to let candidates purchase an examination on a Joomla website, receive a personal link to take the examination in Synap, and receive an email from Joomla when their result is available.

Joomla will manage the purchase, candidate communication and registration history. Synap will deliver and mark the examination.

This is a proposed workflow and technical plan. It is not a description of a completed integration. Exact Synap API endpoints, request/response fields, webhook authentication and attempt-link behaviour must be verified against the API access available to the Synap account before implementation.

## Systems and responsibilities

| System | Responsibility |
| --- | --- |
| Joomla | Orchestrates the workflow, stores registrations and results, and sends candidate emails. |
| RSForm!Pro | Collects candidate details and the selected examination language, and presents the purchase/retake form. |
| Stripe | Processes the payment and provides server-side confirmation of its status. |
| Synap | Manages candidates, examination attempts, random question selection, examination delivery and results. |

The integration should keep the candidate-facing purchase and email experience on the Joomla website. Synap should be used as the examination platform, rather than as the system responsible for payment or candidate email communication.

## Examination languages

There are **six completely separate examinations in Synap**, one for each language. The candidate chooses their examination language in the RSForm!Pro form.

Joomla must use that submitted selection to choose the corresponding Synap examination. The language must be stored with the registration and used throughout the lifecycle of that purchase. It must not be inferred from the candidate's browser, Joomla's default language or email address.

The six Synap examination IDs will be configurable in Joomla. The actual IDs and language labels/codes will be added once supplied.

| Language | Synap examination |
| --- | --- |
| Language 1 | English |
| Language 2 | German |
| Language 3 | Spanish |
| Language 4 | French |
| Language 5 | Italian |
| Language 6 | Dutch |

The table deliberately uses placeholders until the six languages and their Synap examination IDs are confirmed.

## Initial purchase workflow

1. The candidate opens the examination purchase form on the Joomla website.
2. RSForm!Pro collects the candidate's details and their choice of examination language.
3. The candidate pays the examination fee using Stripe.
4. Joomla waits for reliable server-side confirmation that the payment succeeded. A browser redirect or return to a thank-you page is not sufficient proof of payment.
5. Once payment is confirmed, Joomla creates a registration record and selects the Synap examination ID mapped to the chosen language.
6. Joomla uses the Synap API to find or create the candidate as required, and requests access to the selected examination.
7. Synap provides a candidate-specific examination link (subject to confirmation of the exact API operation and behaviour).
8. Joomla stores the relevant Synap identifiers and the access link, then sends the candidate an examination email containing the link and instructions.

A payment must not result in examination access being issued more than once. Payment notifications and the allocation process must therefore be safe to retry.

## Examination attempts and random questions

Synap is configured to generate a random set of questions for each examination attempt.

There is one Synap examination per language. Joomla does not need separate Synap examinations for retakes and does not need to tell Synap whether a purchase is an initial purchase or a retake.

Every successful purchase results in a new examination attempt against the Synap examination corresponding to the language selected for that purchase. Synap is responsible for applying its random-question configuration to the attempt.

The integration must verify that the Synap API can create a fresh attempt and return a new candidate-specific link for each paid registration. In particular, it must confirm that a retake does not simply reopen a previously completed attempt.

## Examination completion and results

Synap should notify Joomla when an examination attempt has been completed and its result is available.

The preferred approach is a Synap webhook to a dedicated Joomla endpoint. The exact event name, payload, delivery behaviour and authentication/signature mechanism must be confirmed against the Synap API version enabled for the account.

When a valid completion notification is received, Joomla should:

1. Authenticate and validate the notification.
2. Identify the relevant registration and attempt using stable Synap identifiers, not just the candidate's email address.
3. Check that the event relates to an attempt known to Joomla.
4. Record the completion date, score and pass/fail result where supplied.
5. Mark the event as processed so that duplicate deliveries cannot trigger duplicate emails.
6. Send the appropriate result email from Joomla.

The integration must distinguish an attempt being submitted from its result being final. Where an examination requires marking or result processing, the candidate email must only be sent once the final result is available.

### Successful result

When the candidate passes, Joomla sends a congratulations email. The email should be based on a Joomla-managed template and sent in the candidate's selected language where translations are available.

### Unsuccessful result

When the candidate fails, Joomla sends a failure email. This email includes information about purchasing a retake and the applicable discount code.

The result and email status should be stored so that an email can be retried after a delivery problem without changing or reprocessing the examination result.

## Retakes and discounted fees

Retakes are a **Joomla and RSForm!Pro pricing matter only**. Synap does not need to know that a candidate is taking a retake.

The retake process is:

1. The candidate fails an attempt.
2. Joomla sends the failure email, including a discount code or retake purchase instructions.
3. The candidate returns to the Joomla website and purchases a retake through RSForm!Pro.
4. The candidate selects the language of the examination they want to take.
5. RSForm!Pro applies the retake price/discount and Stripe processes the payment.
6. After confirmed payment, Joomla creates a new registration and requests a new attempt against the Synap examination for the language selected.
7. Joomla sends a new candidate-specific examination link.

The retake discount must be validated and applied server-side. A price or discount amount supplied only by the browser must never be trusted.

Joomla may record whether a purchase was an initial purchase or a discounted retake for financial and administrative reporting. That information must not alter the Synap API request or examination allocation behaviour.

The language selected on the retake purchase determines which of the six Synap examinations is allocated. If the business rules later require retakes to be restricted to the language of the failed attempt, that restriction should be enforced in Joomla/RSForm!Pro, not in Synap.

## Joomla data and administration

Joomla should retain a record of each paid registration and each examination attempt. A candidate may have multiple registrations over time, and previous registrations and results must not be overwritten by a retake.

At a minimum, the data model should capture:

### Registration

- Joomla registration ID
- Candidate name and email address
- Selected examination language
- RSForm!Pro submission reference
- Stripe payment/session/transaction reference
- Payment status and amount/currency
- Purchase type for reporting (initial or retake, if available)
- Registration status and timestamps

### Examination attempt

- Related Joomla registration ID
- Synap candidate/user ID
- Synap examination ID
- Synap attempt ID or other stable attempt reference
- Candidate-specific examination link, if appropriate to retain
- Attempt status
- Completion timestamp
- Score, where supplied
- Pass/fail result
- Result-email status and timestamp

### Webhook/event processing

- Provider event ID, where supplied
- Event type
- Related registration/attempt reference
- Received and processed timestamps
- Processing status and diagnostic information

An administrator view should make it possible to find a candidate and see their purchases, payment status, selected language, Synap examination/attempt references, result and email status. Resending an email should not create another examination attempt.

## Reliability and security requirements

- **Payment verification:** Only a verified server-to-server Stripe payment confirmation may trigger examination allocation.
- **Webhook authentication:** Validate Synap webhook requests using the authentication mechanism documented for the enabled Synap API version. Do not trust an unauthenticated POST.
- **Idempotency:** Repeated Stripe or Synap notifications must not create duplicate registrations, attempts or emails.
- **Retry handling:** If Synap is unavailable after payment, retain the paid registration as pending allocation and retry safely. The candidate must not have to pay again.
- **Email handling:** Save the result before sending email. Record delivery attempts and allow an administrator to retry email delivery independently.
- **Credentials:** Keep Synap and Stripe secrets on the server, outside public assets and source control. Use least-privilege credentials where supported.
- **Data protection:** Store only the candidate and examination data needed for the service, restrict administrator access and define appropriate retention periods.
- **Logging:** Log integration failures and useful reference IDs without exposing API tokens, payment credentials or unnecessary personal data.

## Proposed implementation stages

1. **Verify Synap access and API capabilities.** Confirm the API version, authentication, account permissions and availability of the required user, examination, attempt, candidate-link and webhook operations.
2. **Build a standalone API test.** Authenticate, retrieve the six examinations, find or create a test candidate, create a fresh attempt and obtain its candidate-specific link.
3. **Verify result notifications.** Configure a test webhook, inspect its payload and authentication, and confirm when a final pass/fail result is available.
4. **Implement the Joomla integration.** Add configuration for the six language-to-examination mappings, registration/attempt storage, API service and webhook endpoint.
5. **Integrate RSForm!Pro and Stripe.** Connect confirmed payment events to registration and attempt allocation. Keep initial and retake pricing in the form/payment workflow.
6. **Implement Joomla emails.** Add access, congratulations and failure/retake templates, with language-aware content as required.
7. **Add administration and recovery tools.** Provide registration/attempt views, processing status and safe email/allocation retries.
8. **Test end to end.** Cover each language, successful and failed payments, pass and fail results, retake purchases, duplicate webhooks, email failures and Synap/API outages.

## Open questions to resolve before implementation

- Which six languages are offered, and what are their corresponding Synap examination IDs?
- Does the Synap account have access to the required API version, candidate management, attempt creation and webhooks?
- Which exact API operation creates a fresh attempt and returns a new candidate-specific examination link?
- Does Synap apply the random-question configuration independently to every attempt created through the API?
- What is the exact webhook event and payload for a final examination result, and how is the webhook authenticated?
- Which RSForm!Pro payment-confirmation hook/event is appropriate for the installed version and Stripe integration?
- Should retake eligibility be restricted to the language of the failed attempt, or may the candidate choose any of the six languages when purchasing again?
- Should candidate emails be available in all six languages?

## Scope boundary

This project is an integration between Joomla and Synap. It does not implement examination content, question randomisation, marking, payment processing or a separate examination engine. Those responsibilities remain with Synap, Stripe and RSForm!Pro respectively.
