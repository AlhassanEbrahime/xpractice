# Talabat Application Analysis

A functional breakdown of the **Talabat** food-delivery mobile application, documenting its screens, features, and underlying functions. This document serves as a shared reference point for developers (as a build/scope guide) and clients/stakeholders (as a plain-language summary of app capabilities).

## 📌 Purpose

This analysis was created by reverse-engineering and documenting the existing Talabat app structure, screen by screen. It is intended to be used as:

- A **requirements reference** for building a similar food-delivery/ordering application
- A **scope-of-work document** for aligning developer and client expectations
- A **feature checklist** for QA and development tracking

## 🗂 Document Structure

The source analysis is organized into three columns:

| Column | Description |
|---|---|
| **Layout** | The main screen/module of the app (e.g., Sign Up, Home, Orders) |
| **Feature** | A specific feature or component within that screen |
| **Function** | The individual capability or user action that the feature performs |

---

## 📱 Application Modules

### 1. Sign Up
Handles new user registration and account verification.

| Feature | Function |
|---|---|
| Sign up with phone number | Verify number via WhatsApp code |
| | Verify number via SMS code |
| | Verify number via voice call code |
| Sign up with Google | Register using a Google account |
| Sign up with Facebook | Register using a Facebook account |

**Developer notes:** Requires integration with WhatsApp Business API (or equivalent OTP provider), SMS gateway, voice OTP service, and OAuth for Google/Facebook.

---

### 2. Home
The main landing screen — discovery, offers, and restaurant/cuisine browsing.

| Feature | Function |
|---|---|
| Available options | Lists the main options available for the user to choose from |
| Rewards | Lists places offering gifts/rewards on orders; apply a voucher if available |
| Vouchers | Add a voucher code; view active, used, and expired vouchers |
| Search | Search bar; list item categories; list popular searches |
| In the spotlight | Lists featured/spotlight offers from partner places |
| Dine out (plan your next outing) | Lists popular cuisines and food categories |
| | Lists nearest restaurants with a dedicated search bar |
| | Filter by rating (4.0+) |
| | Sort by: nearest first, highest ratings, highest discounts, cost for two (low–high / high–low) |
| | Filter by "featured" listings |
| | Map view to choose a location/restaurant |

**Developer notes:** Requires geolocation services, a filtering/sorting engine, a voucher/rewards management system, and map integration (e.g., Google Maps SDK).

---

### 3. Orders
Order history and tracking.

| Feature | Function |
|---|---|
| Orders list | Displays a list of all past and current orders |

---

### 4. Talabat Pay
Payment methods, wallet, and support.

| Feature | Function |
|---|---|
| Payment method | Add a payment card |
| Talabat credit transactions | Lists all wallet/credit transactions |
| Talabat reference number | Add an order reference number |
| Talabat help center | Chatbot support for order and payment issues |

**Developer notes:** Requires secure payment gateway integration (PCI-DSS compliant card storage), a wallet/ledger system for credit transactions, and a chatbot or live-support integration.

---

## 👥 Audience Guide

- **Client / Stakeholder:** Use the module tables above to review what the app should be able to do, confirm scope, and request changes before development begins.
- **Developer:** Use the "Developer notes" under each module as a starting point for technical planning (APIs, SDKs, and third-party services required). Each **Function** row can be treated as an individual user story or ticket.

## 🔖 Status

This is a **living analysis document** — as more screens/modules of the app are reviewed (e.g., Checkout, Profile, Notifications), they should be added here following the same Layout → Feature → Function structure.

## 📃 License

Add your license of choice here (e.g., MIT, Proprietary/Internal Use Only) before publishing this repository.
