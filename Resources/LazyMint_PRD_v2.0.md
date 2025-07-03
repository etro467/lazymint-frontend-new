# Product Requirements Document (PRD) & Detailed Specifications: LazyMint

**Version:** 2.0
**Date:** June 26, 2025
**Codename:** LazyMint

---

## 1. Executive Summary

**LazyMint** is a SaaS web application that empowers digital creators to launch limited-edition content campaigns. It provides a simple, secure, and non-crypto solution for creators to generate fan engagement and build their email lists. Fans claim unique, numbered editions of **symbolic digital tickets** by scanning a custom QR code and verifying their email address. The platform is designed as a modular, installable Progressive Web App (PWA) with a Firebase backend, built to win the Bolt.new Hackathon.

**Core Value Proposition:** LazyMint provides the thrill of exclusivity and the utility of a powerful promotional tool. It offers a flexible, quasi-crypto approach for enhanced authenticity and optional collectibility, while ensuring a seamless experience for those who prefer to remain off-chain. We empower creators with powerful branding, AI-assisted design, and fan engagement tools, offering fans a unique, verifiable, and shareable digital keepsake with the exciting possibility of future NFT collectibility, or simply a valuable digital coupon/gateway.

---

## 2. Architecture & Tech Stack

*   **Frontend:** Bolt.new (generating a modular Preact/React + Tailwind CSS frontend).
*   **Backend:** Firebase (Firestore, Authentication, Cloud Functions, Storage).
*   **Deployment:** Netlify (for the Deploy Challenge).
*   **Monetization:** RevenueCat SDK (for the Make More Money Challenge), designed to be swappable with Stripe post-hackathon.
*   **Blockchain:** Algorand via Nodely API (for the Blockchain Challenge - Verifiable Data, Authenticity, and Optional NFT Collectibility).
*   **AI:** Genkit/Gemini for artistic QR codes, campaign content generation, and ticket background generation.
*   **Email:** Firebase Extension (e.g., SendGrid, Mailgun) for transactional emails.
*   **Image Processing:** Server-side image processing library (e.g., `sharp` for Node.js) within Cloud Functions for ticket composition.

---

## 3. User Flows

### 3.1. Creator Flow

1.  **Sign Up/Login:** Creator signs up or logs into the LazyMint web app using Firebase Authentication.
2.  **Campaign Creation:** Creator initiates a new campaign, defining:
    *   Campaign title, description, and message (can be AI-assisted).
    *   Claim limit and expiration date.
    *   Uploads their brand logo.
    *   Selects a ticket background mode (default template, user-uploaded image, or AI-generated).
    *   If user-uploaded, uploads the background image.
    *   If AI-generated, provides a prompt for the AI to create the background.
    *   Defines the position and size of the artistic QR code overlay on the ticket background.
3.  **QR Code Generation:** LazyMint generates a unique, artistic QR code for the campaign (AI-assisted).
4.  **Campaign Launch:** Creator launches the campaign.
5.  **Distribution:** Creator shares the artistic QR code on their social media, blog, or at live events.
6.  **Analytics:** Creator views campaign analytics (total claims, unique claimants, claim rate) and can export fan emails from their private dashboard.

### 3.2. Fan Flow

1.  **QR Scan:** Fan scans the creator's artistic QR code.
2.  **Landing Page Access:** Fan is directed to a unique, campaign-specific landing page (dynamically generated based on an encrypted hash).
3.  **Email Gate:** Fan enters their email address to claim their spot.
4.  **Email Verification:** Fan receives a verification email and clicks the unique verification link.
5.  **Symbolic Ticket Display:** Upon successful verification, the fan is taken to a success page displaying their personalized, symbolic digital ticket. This ticket is a dynamic composition including:
    *   Creator's branding (logo, title, description).
    *   A unique edition number (e.g., "You are #999 of 1000!").
    *   The main artistic QR code for the campaign.
    *   A message from the creator (can optionally include their own links).
    *   A unique `ticketHash` (which can double as a discount code).
    *   A countdown to ticket expiration.
6.  **Optional NFT Acquisition:** The fan is presented with the option to acquire an exclusive limited edition NFT, leveraging the unique `ticketHash` as a potential identifier or gateway. This is positioned as a way to further support the creator and gain optional collectibility, without guaranteeing monetary value.
7.  **Ticket Download:** Fan can download their symbolic ticket as a personalized PNG image. Download is limited (e.g., 3 times) and the filename is personalized with campaign and claim details.

---

## 4. Backend Specifications (Firebase)

### 4.1. Firestore Schema (Modular & Concise)

Each document should include `createdAt` and `updatedAt` timestamps.

*   **`users/{userId}`**
    *   `email`: string (from Firebase Auth)
    *   `subscriptionTier`: string (e.g., "free", "basic", "pro")
    *   `revenueCatCustomerId`: string (ID from RevenueCat)
    *   `campaignCount`: number (total campaigns created by user)
    *   `totalClaimsMade`: number (total claims across all user's campaigns)

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

### 4.2. Firestore Security Rules

Implement strict, user-centric security rules for Firestore:

*   **`users`**:
    *   Only authenticated users can read their own document.
    *   Users can only update `subscriptionTier` and `revenueCatCustomerId` (via `handleMonetization` Cloud Function).
*   **`campaigns`**:
    *   Authenticated users can create campaigns.
    *   Campaign creators can read, update, and delete their own campaigns.
    *   Public read access for `title`, `description`, `qrCodeStoragePath`, `claimLimit`, `currentClaims`, `status`, `logoStoragePath`, `ticketBackgroundMode`, `customTicketBackgroundStoragePath`, `aiTicketBackgroundPrompt`, `qrCodeOverlayPosition`, `message` (for fan claim pages).
*   **`claims`**:
    *   Authenticated users (creators) can read claims for their own campaigns (filtered by `campaignId`).
    *   Unauthenticated users can create claims (via `processClaim` Cloud Function).
    *   No direct update/delete access for claims.
*   **`algologs`**:
    *   Only Cloud Functions can write.
    *   Public read access.
*   **`legalContent`**:
    *   Public read access.
    *   Only Cloud Functions (or admin SDK) can write.

### 4.3. Firebase Storage Rules

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

### 4.4. Firebase Cloud Functions (API Endpoints)

Implement the following HTTP-triggered Cloud Functions. Each function should handle authentication, input validation, and error handling.

**General Implementation Notes for Cloud Functions:**
*   **Data Consistency:** For operations involving updates to multiple Firestore documents (e.g., `users` and `campaigns`), ensure atomicity by utilizing **Firestore Transactions**.
*   **Secure Secrets:** Store all API keys (e.g., for Nodely, RevenueCat, email services) securely using **Firebase Secret Manager** and access them within the Cloud Functions.
*   **Standardized Error Handling:** All functions should return consistent error responses, including an appropriate HTTP status code and a JSON body like `{ success: false, error: { code: "ERROR_CODE", message: "Descriptive error message" } }`.

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
    *   Validate inputs and user's `subscriptionTier` against campaign limits.
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

#### 4.4.2. `processClaim`
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

#### 4.4.3. `verifyClaim`
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

#### 4.4.4. `downloadAsset`
*   **Method**: `GET`
*   **Path**: `/api/v1/claims/{claimId}/download`
*   **Description**: Generates and serves the symbolic ticket PNG for a verified claim, respecting download limits.
*   **Query Parameters**:
    *   `claimId`: string
    *   `token`: string
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
            *   Load the artistic QR code.
            *   Overlay the artistic QR code onto the background at the `qrCodeOverlayPosition`.
            *   Add the creator's logo (from `campaign.logoStoragePath`), message (from `campaign.message`), edition number (from `claim.editionNumber`), and ticket hash (from `claim.ticketHash`) onto the composed image. **The `ticketHash` should be prominently displayed on the ticket, or embedded in a QR code within the ticket, to facilitate its use as an external discount code.**
        *   Generate the final personalized PNG.
    *   **Filename Personalization**: Serve the generated PNG with a personalized filename (e.g., `[CampaignTitle]-Edition-[EditionNumber]-DL-[DownloadCount]-of-[MaxDownloads].png`).
*   **Response**: PNG image stream or JSON error.

#### 4.4.5. `handleMonetization`
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
*   **Description**: Logs campaign metadata to the Algorand testnet (Pro tier feature).
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

#### 4.4.8. `getCampaignAnalytics`
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

#### 4.4.9. `generateCampaignContent`
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
*   **Response Body (JSON)**:
    ```json
    {
        "backgroundUrl": "string",
        "success": "boolean"
    }
    ```

#### 4.4.11. `getLegalContent`
*   **Method**: `GET`
*   **Path**: `/api/v1/legal/{docId}`
*   **Description**: Retrieves dynamic legal and informational content.
*   **Query Parameters**:
    *   `docId`: string (e.g., "terms_of_use", "privacy_policy", "eula", "about", "contact")
*   **Response Body (JSON)**:
    ```json
    {
        "type": "string",
        "title": "string",
        "content": "string",
        "version": "number",
        "lastUpdated": "timestamp",
        "success": "boolean"
    }
    ```

---

## 5. Frontend Integration with Firebase SDK

Bolt.new should integrate the Firebase JavaScript SDK for the following functionalities:

*   **Authentication:**
    *   Utilize Firebase Authentication for user sign-up, login, and session management.
    *   Obtain and manage Firebase ID Tokens for authenticated API calls to Cloud Functions.
*   **Direct Firestore Reads:**
    *   **Public Data:** For fan-facing pages (e.g., `/campaign/:id`), directly read public campaign details (`title`, `description`, `qrCodeStoragePath`, `claimLimit`, `currentClaims`, `status`, `logoStoragePath`, `ticketBackgroundMode`, `customTicketBackgroundStoragePath`, `aiTicketBackgroundPrompt`, `qrCodeOverlayPosition`, `message`) from the `campaigns` collection using the Firestore SDK.
    *   **User-Specific Data:** For creator dashboards, directly read the authenticated user's `users/{userId}` document and their associated `campaigns` documents (filtered by `creatorId`).
    *   **Legal Content:** Directly read legal and informational content from the `legalContent` collection.
*   **Real-time Data / Firestore Listeners:**
    *   Implement real-time listeners on relevant Firestore collections for dynamic UI updates. Examples:
        *   `campaigns/{campaignId}`: To update `currentClaims` or `status` on the creator dashboard in real-time.
        *   `claims/{claimId}`: To update `verified` status or `downloadCount` for a specific claim.
*   **Frontend-Specific Data Needs (for UI Rendering):**
    *   **Symbolic Ticket Display:** The frontend will need `campaign.title`, `campaign.logoStoragePath`, `campaign.message`, `claim.editionNumber`, `claim.ticketHash`, `claim.ticketExpirationTime`, `campaign.ticketBackgroundMode`, `campaign.customTicketBackgroundStoragePath`, `campaign.aiTicketBackgroundPrompt`, `campaign.qrCodeOverlayPosition` to render the symbolic ticket.
    *   **QR Code Scannability Overlay:** The frontend can utilize the `qrData` (original QR code data, typically a data URI) that was used as input for `generateArtisticQr`. This `qrData` represents the plain black QR code, which can be overlaid with variable opacity on the artistic QR code to assist creators in testing scannability across devices.
    *   **Campaign List:** For the creator dashboard, `campaign.title`, `campaign.status`, `campaign.currentClaims`, `campaign.campaignId`, `campaign.expirationDate`.
    *   **Analytics:** Data from `getCampaignAnalytics` for charts and summaries.

### 6. Environment Variables and Configuration

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
