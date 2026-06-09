# Onboarding Wizard: User Journey & Technical Setup

**Participants:** Anna-Lena Winter, Juliana Gratzer

---

## Summary
This document outlines the finalized user journey and technical setup flow for the onboarding wizard. The core decision involves transitioning from intermittent loading screens to a consolidated **"collect all data first, install later"** approach (similar to an SQL Server installation). This ensures a smoother user experience by minimizing interruptions and standardizing terminology across all setup phases.

---

## Process Design: Wizard Flow

### Revised Workflow
The wizard is designed to collect all user input upfront before the actual technical setup process ("SQL Server Principle") begins.

1.  **Selection:** Choice between "Start with new instance" or connect an existing instance.
2.  **Company Data – General:** Input of basic company information.
3.  **Company Data – Payment:** Provision of payment details.
4.  **Shipping Checkbox:** Confirmation of shipping terms (simple language: "Shipping").
5.  **Shop Checkbox + Shop Name:** Agreement to online shop terms and assignment/adjustment of the shop name.
6.  **Start Setup (Progress Screen):**
    * 6.1. Set up shipping
    * 6.2. Boot up shop
    * 6.3. Set up ERP
    * 6.4. "Success" screen opens
7.  **Success - Create articles now:** Final screen providing direct entry into operational work.

---

## Detailed Progress Screen (Setup)
The progress screen visualizes the status of the processes running in the background. It was decided to avoid technical terms like "hosting" or "app" in favor of clear, understandable language:

* **Set up shipping:** Preparation and linking of shipping functions.
* **Boot up shop:** Provisioning and activation of the online shop.
* **Set up ERP:** Background installation and configuration of the JTL-Wawi.
* **Automatic Transition:** Once all three sub-steps are successfully completed, the "Success" screen opens automatically after a brief holding period.

---

## Technical Background & Conclusion

* **Background Installation:** While the progress bar is running, a server is booted up and the JTL-Wawi is installed automatically. The user does not need to perform any manual installations, allowing for a seamless SaaS experience.
* **Centralized Error Management:** Should problems occur in any of the three steps (Shipping, Shop, ERP), they are displayed directly at the corresponding position on the progress screen.
* **Success – Create articles now:** The final screen confirms the successful setup. The "Create articles now" button serves as the primary call-to-action, allowing the user to immediately begin stocking their shop.
