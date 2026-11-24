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
| Holopin | Issues a verifiable digital badge when a candidate passes. Joomla obtains and includes the individual claim link in the success email. |

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

When the candidate passes, Joomla should initiate two additional actions: issue a Holopin badge and publish the candidate's certification in the public directory (subject to the candidate's directory-publication choice).

#### Issue a Holopin badge

Joomla uses the Holopin API to issue the appropriate regular (static) badge for the certification. The badge represents the achievement; it is not issued for a payment or merely for completing an examination. Only a final passing result should trigger badge issuance.

The proposed API flow is:

1. Joomla receives and validates the final passing result from Synap.
2. Joomla records the pass before attempting external actions.
3. Joomla calls Holopin's regular badge API for the configured badge/sticker ID, supplying the candidate's email address only if the API flow is intended to send an email from Holopin. The preferred flow is to omit the email so that Joomla remains responsible for all candidate communication.
4. Holopin returns a coupon identifier for a unique claim URL.
5. Joomla constructs/stores the claim URL using the documented format, `https://holopin.io/claim/[couponId]`.
6. Joomla includes the claim link in the success email.

The documented regular-badge API endpoint is `POST https://www.holopin.io/api/sticker/share?id=[stickerId]&apiKey=[apiKey]`. The API documentation describes an optional recipient email and a coupon ID that can be used to construct an individual claim URL. The exact request format, response handling, API key permissions and any metadata fields must be confirmed during implementation.

The Holopin API key and badge/sticker ID must be stored securely in Joomla configuration. Badge issuance must be idempotent: retries after a timeout or webhook redelivery must not accidentally issue multiple badges for the same certification achievement. Joomla should store the Holopin coupon ID, claim URL, badge/sticker ID, issuance status and timestamps.

The success email should contain:
- Confirmation that the candidate passed.
- The Holopin badge claim link.
- A short explanation of how to claim the badge.
- A suggested social-media message the candidate can copy or adapt to announce that they passed, with a link to their claimed badge or public verification profile when available.

Joomla should not publish the private claim URL in the public directory. It is an individual claim credential and belongs in the candidate's private success email. Once claimed, the candidate may share the public badge/profile URL provided by Holopin.

Holopin describes regular badges as static digital badges and documents social sharing of badges and badge boards. The integration should use the regular badge flow rather than the experimental custom-image endpoint unless a later requirement specifically needs per-recipient badge artwork.

#### Public certified-user directory

**Joomla's public verification page is the permanent, authoritative record of certification.** Holopin provides a portable digital badge that the candidate can claim and share, but it is supplementary evidence and is not the source of truth.

A third party should be able to verify a person's certification using a stable Joomla URL, whether or not the candidate has claimed their Holopin badge, still has access to their Holopin account, or chooses to share the badge. The Joomla verification page should remain available for as long as the certification remains valid, subject to the site's published retention and certification policies.

Joomla should also provide a public directory of people who have passed the examination, modelled on the [Joomla Certified User Directory](https://certification.joomla.org/certified-user-directory).

The directory is a Joomla feature and its data should come from Joomla's own verified examination records, not from a list of people who have claimed their Holopin badge. Passing the examination is the evidence for certification; claiming a Holopin badge is optional and must not determine whether someone appears in the directory.

The public directory should allow visitors to:
- Browse and search certified candidates.
- See the certification or examination achieved.
- See the date the candidate passed or was certified.
- Open an individual public profile or verification page at a stable, permanent Joomla URL.
- Verify that a named person has a recorded pass for the stated certification, independently of Holopin.

The exact public fields and directory-publication rules must be agreed before implementation. Recommended default: ask candidates for explicit consent to appear in the public directory during registration, and publish only the fields they have agreed to make public. For example, this might include their display name, certification title and date achieved. Email addresses, payment details, Synap identifiers, examination scores, internal registration references and Holopin claim URLs must never be exposed publicly.

Each public profile should have a stable URL that can be shared independently of Holopin. This Joomla page is the canonical verification reference and should show the current certification status and enough information for a third party to verify the claim. The Holopin badge and its public profile may link back to this Joomla page, but must not replace it. If a certification is revoked or corrected, the directory entry must be updated accordingly.

The directory should support pagination and search/filtering as its population grows. Search-engine indexing should be an explicit product/privacy decision, not an accidental consequence of making the pages public.

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
- Holopin badge/sticker ID
- Holopin coupon ID and private claim URL
- Holopin issuance status and timestamp
- Public-directory consent/status and publication timestamp
- Public profile URL or directory record ID

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
- **Data protection:** Store only the candidate and examination data needed for the service, restrict administrator access and define appropriate retention periods. Obtain and record explicit consent before publishing personal details in the public certified-user directory; provide a process for correcting or removing public profile information where appropriate.
- **Public/private separation:** Never expose email addresses, payment data, internal identifiers, examination scores or Holopin claim URLs in public directory pages or public APIs.
- **Holopin issuance:** Issue a badge only for a final passing result, store the returned coupon/claim reference securely, and prevent duplicate issuance when events are retried.
- **Logging:** Log integration failures and useful reference IDs without exposing API tokens, payment credentials or unnecessary personal data.

## Proposed implementation stages

1. **Verify Synap access and API capabilities.** Confirm the API version, authentication, account permissions and availability of the required user, examination, attempt, candidate-link and webhook operations.
2. **Build a standalone API test.** Authenticate, retrieve the six examinations, find or create a test candidate, create a fresh attempt and obtain its candidate-specific link.
3. **Verify result notifications.** Configure a test webhook, inspect its payload and authentication, and confirm when a final pass/fail result is available.
4. **Implement the Joomla integration.** Add configuration for the six language-to-examination mappings, registration/attempt storage, API service and webhook endpoint.
5. **Integrate RSForm!Pro and Stripe.** Connect confirmed payment events to registration and attempt allocation. Keep initial and retake pricing in the form/payment workflow.
6. **Integrate Holopin.** Configure the regular badge/sticker ID and API credentials, issue a badge after a final pass, and verify that Joomla receives and stores the unique claim URL.
7. **Implement Joomla emails.** Add access, congratulations and failure/retake templates, with language-aware content as required. Include the Holopin claim link and suggested social-sharing text in the success email.
8. **Build the public certified-user directory.** Add candidate consent, searchable public listings and individual verification pages using Joomla's verified pass records.
9. **Add administration and recovery tools.** Provide registration/attempt views, badge issuance and directory-publication status, plus safe email/allocation/badge retries.
10. **Test end to end.** Cover each language, successful and failed payments, pass and fail results, retake purchases, duplicate webhooks, duplicate badge prevention, directory consent, email failures and Synap/Holopin API outages.

## Open questions to resolve before implementation

- Which six languages are offered, and what are their corresponding Synap examination IDs?
- Does the Synap account have access to the required API version, candidate management, attempt creation and webhooks?
- Which exact API operation creates a fresh attempt and returns a new candidate-specific examination link?
- Does Synap apply the random-question configuration independently to every attempt created through the API?
- What is the exact webhook event and payload for a final examination result, and how is the webhook authenticated?
- Which RSForm!Pro payment-confirmation hook/event is appropriate for the installed version and Stripe integration?
- Should retake eligibility be restricted to the language of the failed attempt, or may the candidate choose any of the six languages when purchasing again?
- Should candidate emails be available in all six languages?
- What is the Holopin organization badge/sticker ID to issue for a successful certification?
- Should all successful candidates be listed publicly, or should directory publication be opt-in with explicit consent? Which profile fields may be shown?
- Should the directory be indexed by search engines, and should candidates be able to request a correction or removal?
- Should the public directory show the Holopin badge/profile link once claimed, in addition to Joomla's own verification page? The Joomla page remains authoritative regardless.

## Scope boundary

This project integrates Joomla with RSForm!Pro, Stripe, Synap and Holopin. It does not implement examination content, question randomisation or marking; those remain Synap responsibilities. Stripe processes payments, RSForm!Pro manages the purchase form, Holopin issues the digital badge, and Joomla coordinates the workflow, emails and public certification directory.
