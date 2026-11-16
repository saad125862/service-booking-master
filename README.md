# Seba (সেবা): Service Booking & Management App, Master Repository

Umbrella repository for the Flutter technical assessment. It holds the plan, shared documentation and links to the two code repositories.

| Part | Repository | Stack |
|---|---|---|
| Mobile app | [service-booking-app](https://github.com/saad125862/service-booking-app) | Flutter, Riverpod, go_router, dio |
| Backend API | [service-booking-backend](https://github.com/saad125862/service-booking-backend) | Node.js, Express 5, PostgreSQL, Prisma, JWT, Swagger, Gemini |


## Screenshots

<table>
<tr><td align="center" valign="top"><img src="https://raw.githubusercontent.com/saad125862/service-booking-app/main/docs/screenshots/03-home.jpg" width="160" alt="Catalog"><br><sub>Catalog</sub></td><td align="center" valign="top"><img src="https://raw.githubusercontent.com/saad125862/service-booking-app/main/docs/screenshots/07-booking-form.jpg" width="160" alt="Booking with discount"><br><sub>Booking with discount</sub></td><td align="center" valign="top"><img src="https://raw.githubusercontent.com/saad125862/service-booking-app/main/docs/screenshots/11-booking-confirmed-pay.jpg" width="160" alt="Online payment"><br><sub>Online payment</sub></td><td align="center" valign="top"><img src="https://raw.githubusercontent.com/saad125862/service-booking-app/main/docs/screenshots/49-assistant-discount-approved.jpg" width="160" alt="AI discount approval"><br><sub>AI discount approval</sub></td><td align="center" valign="top"><img src="https://raw.githubusercontent.com/saad125862/service-booking-app/main/docs/screenshots/23-admin-dashboard.jpg" width="160" alt="Admin dashboard"><br><sub>Admin dashboard</sub></td><td align="center" valign="top"><img src="https://raw.githubusercontent.com/saad125862/service-booking-app/main/docs/screenshots/37-admin-add-service-photo-and-fields.jpg" width="160" alt="Dynamic service form"><br><sub>Dynamic service form</sub></td></tr>
</table>

More: [customer, AI assistant, payments and admin screens](https://github.com/saad125862/service-booking-app#screenshots).

## Contents
- [PLAN.md](PLAN.md): implementation plan (architecture, database design, API, timeline, deliverables)

## Environments
- **Local:** Docker Compose (API and PostgreSQL)
- **Live:** Render (API) and Neon (PostgreSQL)

## Status

| Phase | State |
|---|---|
| Plan, repos, Docker | Done |
| Database (Prisma, DB-enforced booking conflicts) | Done |
| Backend: auth, users, services, availability, bookings, admin, notifications, discounts, payments | Done |
| Backend: AI assistant (English, Bangla, Banglish) and Swagger docs | Done |
| Deploy to Render + Neon | Done: [live API](https://seba-api-0062.onrender.com), [Swagger docs](https://seba-api-0062.onrender.com/api/docs) |
| Flutter foundation + authentication (persistent session) | Done |
| Flutter catalog: search, filters, featured, details | Done |
| Flutter booking flow, my bookings, profile edit, AI chat, notifications, sidebar, dark / light mode | Done |
| Admin side: dashboard, bookings, dynamic add-service form with custom fields, payment options, discount approvals, sales reports | Done |
| Online card / wallet payments (SSLCommerz, aamarPay, demo checkout), AI sends the exact-amount pay link | Done, 79 backend tests + 130 app tests |
| Dark premium UI and motion, release APK | Done: [Seba v1.0.1 APK](https://github.com/saad125862/service-booking-app/releases/tag/v1.0.1) |
| Emulator testing, demo video | Next (yours) |
