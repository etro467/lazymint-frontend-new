# Firebase Studio Backend Prompt: LazyMint

This prompt is designed to guide Firebase Studio in building the complete backend infrastructure for the LazyMint application. The goal is a robust, scalable, and secure Firebase backend that exposes a clear API contract for the Bolt.new frontend.

## Project Overview: LazyMint

LazyMint is a SaaS web application empowering digital creators to launch limited-edition content campaigns. It provides a simple, secure, and non-crypto solution for creators to generate fan engagement and build their email lists. Fans claim unique, numbered editions of symbolic digital tickets by scanning a custom QR code and verifying their email address.

## Core Backend Requirements

### 1. Firebase Project Setup

*   Initialize a new Firebase project.
*   Enable Firestore, Firebase Authentication, Firebase Cloud Functions, and Firebase Storage.

### 2. Firestore Schema (Modular & Concise)

Design and implement the following Firestore collections and document structures. Each document should include `createdAt` and `updatedAt` timestamps.

*   **`users/{userId}`**
    *   `email`: string (from Firebase Auth)
    *   `subscriptionTier`: string (e.g., "free", "basic", "pro")
    *   `revenueCatCustomerId`: string (ID from RevenueCat)
    *   `campaignCount`: number (total campaigns created by user)
    *   `totalClaimsMade`: number (total claims across all user's campaigns)
    *   `aiQrGenerationsThisMonth`: number (counter for artistic QR code generations)
    *   `aiBackgroundGenerationsThisMonth`: number (counter for AI ticket background generations)
    *   `lastBillingCycleReset`: timestamp (timestamp of the last monthly reset for AI counters)

*   **`campaigns/{campaignId}`**
    *   `creatorId`: string (FK to `users/{userId}`)
    *   `title`: string
    *   `description`: string
    *   `message`: string (Creator's message, can contain plaintext/markdown links. Links can be constructed to include the `ticketHash` for external use, e.g., as a discount code parameter.)
    *   `logoStoragePath`: string (Firebase Storage path to creator's logo for the campaign)
    *   `ticketBackgroundMode`: string (e.g., "default_template", "user_uploaded_image", "ai_generated")
    *   `customTicketBackgroundStoragePath`: string (Firebase Storage path for user-uploaded or AI-generated backgrounds)
    *   `aiTicketBackgroundPrompt`: string (text prompt used for AI generation of background)
    *   `qrCodeOverlayPosition`: object (e.g., `{ x: number, y: number, width: number, height: number }` - defines QR code placement on ticket background)
    *   `claimLimit`: number (max claims allowed)
    *   `currentClaims`: number (current number of claims made)
    *   `expirationDate`: timestamp (optional)
    *   `status`: string (e.g., "active", "paused", "completed")
    *   `qrCodeStoragePath`: string (Firebase Storage path to generated artistic QR code)
    *   `algorandTransactionId`: string (optional, for Pro tier)

*   **`claims/{claimId}`** (Top-level collection for easier querying, `campaignId` as field)
    *   `campaignId`: string (FK to `campaigns/{campaignId}`)
    *   `fanEmail`: string
    *   `editionNumber`: number (unique for campaign)
    *   `ticketHash`: string (Unique hash for the ticket, can be used as a coupon code)
    *   `maxDownloads`: number (Maximum number of times the ticket PNG can be downloaded)
    *   `ticketExpirationTime`: timestamp (Expiration time for the ticket download link/access)
    *   `claimedAt`: timestamp
    *   `verified`: boolean (email verification status)
    *   `assetDelivered`: boolean (asset delivery status)
    *   `downloadCount`: number (how many times asset has been downloaded by this claim)
    *   `verificationToken`: string (for email verification)

*   **`algologs/{algologId}`**
    *   `campaignId`: string (FK to `campaigns/{campaignId}`)
    *   `metadataHash`: string (SHA-256 hash of campaign metadata)
    *   `algorandTransactionId`: string (transaction ID on Algorand testnet)

*   **`legalContent/{docId}`**
    *   `type`: string (e.g., "terms_of_use", "privacy_policy", "eula", "about", "contact")
    *   `title`: string
    *   `content`: string (Markdown or HTML content)
    *   `version`: number
    *   `lastUpdated`: timestamp

### 3. Firestore Security Rules

Implement strict, user-centric security rules for Firestore:

*   **`users`**:
    *   Authenticated users can read their own document.
    *   Users can only update `subscriptionTier` and `revenueCatCustomerId` (via `handleMonetization` Cloud Function).
*   **`campaigns`**:
    *   Authenticated users can create campaigns.
    *   Campaign creators can read, update, and delete their own campaigns.
    *   Public read access for `title`, `description`, `qrCodeStoragePath`, `claimLimit`, `currentClaims`, `status` (for fan claim pages).
*   **`claims`**:
    *   Authenticated users (creators) can read claims for their own campaigns (filtered by `campaignId`).
    *   Unauthenticated users can create claims (via `processClaim` Cloud Function).
    *   No direct update/delete access for claims.
*   **`algologs`**:
    *   Only Cloud Functions can write.
    *   Public read access.

### 4. Firebase Storage Rules

Implement security rules for Firebase Storage:

*   **`qrcodes` bucket**:
    *   Only Cloud Functions can write.
    *   Publicly readable.
*   **`logos` bucket**:
    *   Only authenticated users (creators) can upload their logos (e.g., `logos/{userId}/logo.png`).
    *   Publicly readable.
*   **`ticket_backgrounds` bucket**:
    *   Only Cloud Functions can write (for AI-generated backgrounds).
    *   Authenticated users (creators) can upload their own backgrounds (e.g., `ticket_backgrounds/{userId}/background.png`).
    *   Publicly readable.

### 5. Firebase Cloud Functions (API Endpoints)

Implement the following HTTP-triggered Cloud Functions. Each function should handle authentication, input validation, and error handling.

**General Implementation Notes for Cloud Functions:**
*   **Data Consistency:** For operations involving updates to multiple Firestore documents (e.g., `users` and `campaigns`), ensure atomicity by utilizing **Firestore Transactions**.
*   **Secure Secrets:** Store all API keys (e.g., for Nodely, RevenueCat, email services) securely using **Firebase Secret Manager** and access them within the Cloud Functions.
*   **Standardized Error Handling:** All functions should return consistent error responses, including an appropriate HTTP status code and a JSON body like `{ success: false, error: { code: "ERROR_CODE", message: "Descriptive error message" } }`.
*   **Rate Limiting:** Implement rate limiting for public-facing and resource-intensive endpoints (e.g., `processClaim`, `generateArtisticQr`, `generateTicketBackground`) to prevent abuse and ensure fair usage. Consider using Firebase App Check or a custom rate-limiting mechanism.

**Email Sending Strategy (Modular for MVP):**
For sending emails (e.g., verification links), it is recommended to utilize a Firebase Extension (e.g., "Send Email with SendGrid" or "Trigger Email") for rapid MVP development. This approach simplifies implementation and management. Ensure the email sending logic is encapsulated to allow for easy swapping with a direct email service API integration post-hackathon if more advanced features or control are required.

---

### API Endpoints (for Bolt.new Frontend Integration)

Below are the detailed specifications for each Cloud Function, serving as the API contract for the Bolt.new frontend. All requests to authenticated endpoints must include a Firebase ID Token in the `Authorization: Bearer` header.

#### 4.4.1. `createCampaign`
*   **Method**: `POST`
*   **Path**: `/api/v1/campaigns`
*   **Description**: Creates a new campaign for the authenticated creator.
*   **Request Body (JSON)**:
    ```json
    {
        "title": "string",
        "description": "string",
        "message": "string",
        "claimLimit": "number",
        "expirationDate": "timestamp" (optional),
        "ticketBackgroundMode": "string" (e.g., "default_template", "user_uploaded_image", "ai_generated"),
        "customTicketBackgroundFile": "multipart/form-data" (optional, if user_uploaded_image mode),
        "aiTicketBackgroundPrompt": "string" (optional, if ai_generated mode),
        "qrCodeOverlayPosition": "object" (e.g., `{ x: number, y: number, width: number, height: number }`, optional)
    }
    ```
    *Note: `logoFile` will be handled via a separate `uploadLogo` endpoint or direct Firebase Storage upload from frontend.* 
*   **Logic**:
    *   Authenticate user.
    *   Validate inputs and user's `subscriptionTier` against campaign limits (Free: 1 active campaign, Basic: 30 active campaigns, Pro: 100 active campaigns).
    *   If `customTicketBackgroundFile` is provided, upload it to Firebase Storage (`ticket_backgrounds/{userId}/background.png`) and store its public URL in `customTicketBackgroundStoragePath`.
    *   If `ai_generated` mode is selected and `aiTicketBackgroundPrompt` is provided, trigger `generateTicketBackground` to create and store the background.
    *   Create a new `campaign` document in Firestore, setting `currentClaims` to 0.
    *   Increment `campaignCount` in `users/{userId}`.
*   **Response Body (JSON)**:
    ```json
    {
        "campaignId": "string",
        "success": "boolean",
        "message": "string" (optional, for success/error),
        "ticketBackgroundUrl": "string" (optional, if user_uploaded_image or ai_generated mode)
    }
    ```

#### 5.2. `processClaim`
*   **Method**: `POST`
*   **Path**: `/api/v1/claims`
*   **Description**: Processes a fan's claim for a campaign, generates a symbolic ticket, and sends a verification email.
*   **Request Body (JSON)**:
    ```json
    {
        "campaignId": "string",
        "fanEmail": "string"
    }
    ```
*   **Response Body (JSON)**:
    ```json
    {
        "message": "string",
        "success": "boolean",
        "claimId": "string" (optional)
    }
    ```

#### 5.3. `verifyClaim`
*   **Method**: `GET` or `POST`
*   **Path**: `/api/v1/claims/{claimId}/verify`
*   **Description**: Verifies a fan's email for a claim.
*   **Request Body (JSON - for POST) / Query Params (for GET)**:
    ```json
    {
        "claimId": "string",
        "token": "string"
    }
    ```
*   **Response Body (JSON)**:
    ```json
    {
        "message": "string",
        "success": "boolean",
        "campaignId": "string" (optional),
        "editionNumber": "number" (optional)
    }
    ```

*   **`downloadAsset` (HTTP GET)**
    *   **Input**: `claimId`, `token` (for authentication/authorization).
    *   **Logic**:
        *   Authenticate `claimId` and `token`.
        *   Retrieve `claim` and `campaign` documents.
        *   **Download Limit Enforcement**: Check `claim.downloadCount` against `claim.maxDownloads` and `claim.ticketExpirationTime`. If limit exceeded or expired, deny download.
        *   Increment `claim.downloadCount`.
        *   **Image Composition for Symbolic Ticket**: 
            *   Retrieve the chosen `ticketBackgroundMode` and associated data (`customTicketBackgroundStoragePath`, `aiTicketBackgroundPrompt`) from the `campaign` document.
            *   Fetch the appropriate background image (default template, user-uploaded, or AI-generated via `generateTicketBackground`).
            *   Fetch the artistic QR code image from `campaign.qrCodeStoragePath`.
            *   **Utilize an image processing library (e.g., `sharp` for Node.js) to:**
                *   Load the background image.
                *   Load the artistic QR code and overlay it onto the background at `campaign.qrCodeOverlayPosition`.
                *   Add the creator's logo (from `campaign.logoStoragePath`), campaign title, description, message (from `campaign.message`), edition number (from `claim.editionNumber`), and ticket hash (from `claim.ticketHash`) onto the composed image. **The `ticketHash` should be prominently displayed on the ticket, or embedded in a QR code within the ticket, to facilitate its use as an external discount code.**
            *   Generate the final personalized PNG. **Ensure image optimization (e.g., compression, conversion to WebP) for efficient delivery.**
        *   **Filename Personalization**: Serve the generated PNG with a personalized filename (e.g., `[CampaignTitle]-Edition-[EditionNumber]-DL-[DownloadCount]-of-[MaxDownloads].png`).
    *   **Output**: PNG image stream or `{ message: string, success: boolean }` on error/limit.

#### 5.5. `handleMonetization`
*   **Method**: `POST`
*   **Path**: `/api/v1/webhooks/revenuecat`
*   **Description**: Webhook endpoint for RevenueCat events to update user subscription tiers.
*   **Request Body**: RevenueCat webhook payload (JSON).
*   **Response Body (JSON)**:
    ```json
    {
        "message": "string",
        "success": "boolean"
    }
    ```

#### 4.4.6. `generateArtisticQr`
*   **Method**: `POST`
*   **Path**: `/api/v1/qrcodes/generate`
*   **Description**: Generates an artistic QR code for a given campaign.
*   **Request Body (JSON)**:
    ```json
    {
        "qrData": "string",
        "styleMode": "string" ("prompt" or "image"),
        "stylePrompt": "string" (optional),
        "userId": "string",
        "scannabilityPreference": "string" ("lowest" to "highest"),
        "backgroundImageDataUri": "string" (optional),
        "useImageForStyle": "boolean" (optional)
    }
    ```
*   **Logic**:
    *   Authenticate user.
    *   Retrieve user's `subscriptionTier` and `aiQrGenerationsThisMonth` from `users/{userId}`.
    *   **Enforce Quota (Conservative MVP Limits):**
        *   Free Tier: Default for new users. Limited campaigns (1 active campaign), limited claims (100 claims per month), 1 asset download per claim.
        *   Basic Tier: Max 30 artistic QR code generations per month.
        *   Pro Tier: Max 100 artistic QR code generations per month.
    *   Integrate with Genkit/Gemini (as demonstrated in `src/ai/flows/generate-artistic-qr.ts`).
    *   Generate the artistic QR code image.
    *   Validate scannability.
    *   Store the generated QR code image in Firebase Storage (`qrcodes/{campaignId}/qr.png`) and update `qrCodeStoragePath` in the `campaign` document.
    *   Implement caching of results to Firestore to avoid regenerating identical QR codes.
    *   Include a non-AI fallback if Genkit/Gemini fails or is unavailable.
*   **Response Body (JSON)**:
    ```json
    {
        "artisticQrCodeUrl": "string",
        "isScannable": "boolean",
        "success": "boolean"
    }
    ```

#### 4.4.7. `logCampaignToAlgorand`
*   **Method**: `POST`
*   **Path**: `/api/v1/campaigns/{campaignId}/log-algorand`
*   **Description**: Logs campaign metadata to the Algorand testnet, providing on-chain authenticity and enabling future optional NFT collectibility for fans (Pro tier feature).
*   **Request Body (JSON)**:
    ```json
    {
        "campaignId": "string"
    }
    ```
*   **Response Body (JSON)**:
    ```json
    {
        "algorandTransactionId": "string",
        "success": "boolean"
    }
    ```



#### 5.8. `getCampaignAnalytics`
*   **Method**: `GET`
*   **Path**: `/api/v1/campaigns/{campaignId}/analytics`
*   **Description**: Retrieves analytics data for a specific campaign.
*   **Query Parameters**:
    *   `campaignId`: string
*   **Response Body (JSON)**:
    ```json
    {
        "totalClaims": "number",
        "uniqueClaimants": "number",
        "claimRate": "number",
        "message": "string" (optional, for success/error)
        // ... other analytics data
    }
    ```

#### 5.9. `generateCampaignContent`
*   **Method**: `POST`
*   **Path**: `/api/v1/campaigns/generate-content`
*   **Description**: Generates AI-assisted campaign content (title, description, message) based on user input.
*   **Request Body (JSON)**:
    ```json
    {
        "campaignTheme": "string",
        "keywords": "string" (comma-separated, optional),
        "tone": "string" (e.g., "exciting", "professional", "casual", optional)
    }
    ```
*   **Response Body (JSON)**:
    ```json
    {
        "title": "string",
        "description": "string",
        "message": "string",
        "success": "boolean"
    }
    ```

#### 4.4.10. `generateTicketBackground`
*   **Method**: `POST`
*   **Path**: `/api/v1/tickets/generate-background`
*   **Description**: Generates an AI-assisted image for the symbolic ticket background.
*   **Request Body (JSON)**:
    ```json
    {
        "prompt": "string",
        "userId": "string"
    }
    ```
*   **Logic**:
    *   Authenticate user.
    *   Retrieve user's `subscriptionTier` and `aiBackgroundGenerationsThisMonth` from `users/{userId}`.
    *   **Enforce Quota (Conservative MVP Limits):**
        *   Free Tier: Max 1 AI ticket background generation per month.
        *   Basic Tier: Max 30 AI ticket background generations per month.
        *   Pro Tier: Max 100 AI ticket background generations per month.
    *   Integrate with Genkit/Gemini.
    *   Generate the artistic background image.
    *   Store the generated image in the `ticket_backgrounds` bucket.
*   **Response Body (JSON)**:
    ```json
    {
        "backgroundUrl": "string",
        "success": "boolean"
    }
    ```

---

### 6. Frontend Integration with Firebase SDK

Bolt.new should integrate the Firebase JavaScript SDK for the following functionalities:

*   **Authentication:**
    *   Utilize Firebase Authentication for user sign-up, login, and session management.
    *   Obtain and manage Firebase ID Tokens for authenticated API calls to Cloud Functions.
*   **Direct Firestore Reads:**
    *   **Public Data:** For fan-facing pages (e.g., `/campaign/:id`), directly read public campaign details (`title`, `description`, `qrCodeStoragePath`, `claimLimit`, `currentClaims`, `status`) from the `campaigns` collection using the Firestore SDK.
    *   **User-Specific Data:** For creator dashboards, directly read the authenticated user's `users/{userId}` document and their associated `campaigns` documents (filtered by `creatorId`).
*   **Real-time Data / Firestore Listeners:**
    *   Implement real-time listeners on relevant Firestore collections for dynamic UI updates. Examples:
        *   `campaigns/{campaignId}`: To update `currentClaims` or `status` on the creator dashboard in real-time.
        *   `claims/{claimId}`: To update `verified` status or `downloadCount` for a specific claim.
*   **Frontend-Specific Data Needs (for UI Rendering):**
    *   **Symbolic Ticket Display:** The frontend will need `campaign.title`, `campaign.logoStoragePath`, `campaign.description`, `campaign.qrCodeOverlayPosition`, `campaign.message`, `claim.editionNumber`, `claim.ticketHash`, `claim.ticketExpirationTime`, `campaign.ticketBackgroundMode`, `campaign.customTicketBackgroundStoragePath`, `campaign.aiTicketBackgroundPrompt` to render the symbolic ticket.
    *   **QR Code Scannability Overlay:** The frontend can utilize the `qrData` (original QR code data, typically a data URI) that was used as input for `generateArtisticQr`. This `qrData` represents the plain black QR code, which can be overlaid with variable opacity on the artistic QR code to assist creators in testing scannability across devices.
    *   **Campaign List:** For the creator dashboard, `campaign.title`, `campaign.status`, `campaign.currentClaims`, `campaign.campaignId`, `campaign.expirationDate`.
    *   **Analytics:** Data from `getCampaignAnalytics` for charts and summaries.

### 7. Environment Variables and Configuration

The Bolt.new frontend will require the following Firebase project configuration details as environment variables for proper SDK initialization:

*   `FIREBASE_API_KEY`
*   `FIREBASE_AUTH_DOMAIN`
*   `FIREBASE_PROJECT_ID`
*   `FIREBASE_STORAGE_BUCKET`
*   `FIREBASE_MESSAGING_SENDER_ID`
*   `FIREBASE_APP_ID`
*   `FIREBASE_MEASUREMENT_ID` (optional)

## Deployment

*   The Firebase project should be configured for deployment via standard Firebase CLI commands.
*   Cloud Functions should be deployed to a region close to the expected user base.

This comprehensive prompt should enable Firebase Studio to construct the robust backend required for LazyMint.
