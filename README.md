# Seba (সেবা): Service Booking & Management App, Master Repository

Umbrella repository for the Flutter technical assessment. It holds the plan, shared documentation and links to the two code repositories.

| Part | Repository | Stack |
|---|---|---|
| Mobile app | [service-booking-app](https://github.com/saad125862/service-booking-app) | Flutter, Riverpod, go_router, dio |
| Backend API | [service-booking-backend](https://github.com/saad125862/service-booking-backend) | Node.js, Express 5, PostgreSQL, Prisma, JWT, Swagger, Gemini |

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
| Backend: auth, users, services, availability, bookings, admin | Done, 23 integration tests |
| Backend: AI assistant (English, Bangla, Banglish) and Swagger docs | Done |
| Deploy to Render + Neon | Done: [live API](https://seba-api-0062.onrender.com), [Swagger docs](https://seba-api-0062.onrender.com/api/docs) |
| Flutter foundation + authentication (persistent session) | Done |
| Flutter: catalog, booking, my bookings, profile, AI chat | In progress |
