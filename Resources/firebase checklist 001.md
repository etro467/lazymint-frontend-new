# Firebase Studio

This is a NextJS starter in Firebase Studio.

To get started, take a look at src/app/page.tsx.

# LazyMint Backend - Project Status

This checklist tracks the progress of the LazyMint application based on the project specifications.

## ✅ Accomplished

-   **UI Foundation**: Project set up with Next.js, React, TypeScript, ShadCN UI, and Tailwind CSS.
-   **Styling & Theme**: The visual theme (colors, fonts, icons, card layout) is implemented as per the style guidelines in `src/app/globals.css`.
-   **Core UI Components**:
    -   Dashboard page (`/dashboard`) displaying mock campaigns.
    -   Campaign creation form (`/campaigns/create`) with AI content generation.
    -   Campaign details page (`/campaigns/[campaignId]`) with tabs for analytics, QR code generation, claims, and settings.
    -   Public claim page (`/campaigns/[campaignId]/claim`) for fans.
    -   Digital ticket display page (`/claims/[claimId]/ticket`).
    -   Basic login page UI.
    -   Legal document pages (`/legal/[slug]`).
-   **AI Content Generation**: The Genkit flow `generateCampaignContent` is implemented and connected to the UI for generating a campaign title, description, and message.

## 🟡 In Progress

-   **Artistic QR Code Generation**: A Genkit flow (`generateArtisticQr`) exists. The UI on the campaign details page calls this flow, but currently uses a placeholder for the generated image. Full integration (displaying the real image, saving to storage) is pending.
-   **AI Ticket Background Generation**: A Genkit flow (`generateTicketBackground`) exists. The UI in the campaign creation form calls this flow and displays a preview. Full integration (saving the image and linking it to the campaign) is pending.
-   **Subscription Management**: A webhook endpoint for RevenueCat exists but contains only boilerplate code. It needs to be implemented to handle subscription events and update user data in Firestore.

## ⏳ Not Yet Implemented

-   **Firebase Integration**: The entire backend is pending. The app currently relies entirely on mock data (`src/lib/mock-data.ts`). This includes:
    -   **Firestore Database**: Setup for all collections (users, campaigns, claims, etc.).
    -   **Firebase Authentication**: User login/signup functionality.
    -   **Firebase Storage**: Buckets for hosting logos, QR codes, and ticket backgrounds.
    -   **Firebase Security Rules**: Comprehensive rules for Firestore and Storage.
-   **Ticket Composition**: The backend logic (e.g., using `Sharp`) for combining the logo, QR code, and background into a final ticket image is missing.
-   **Algorand Logging**: The feature to log campaign metadata to the Algorand blockchain is a UI placeholder. The backend implementation is required.
-   **Full End-to-End Functionality**:
    -   Saving new campaigns to the database.
    -   Processing fan claims and sending verification emails.
    -   Tracking download counts and claim limits against real data.
    -   Displaying real data in the analytics dashboard.

## 📝 Deviations / Notes

-   **Focus**: The project is currently a functional frontend prototype with AI integrations. The next major phase will be to build out the Firebase backend to replace all mock data and client-side simulations.
