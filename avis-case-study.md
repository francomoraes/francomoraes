# Avis-app (Vehicle reminder messaging service)

A WhatsApp-based reminder service that turns vehicle maintenance and tax obligations into natural-language conversations, no app to install, no notifications to manage.

## The issue

Owning and maintaining a vehicle often leads to several non obvious obligations that are often forgotten or left aside;
Checking tyre pressure, for instance, is recommended once a month as good practice, but most people do it once a year or only after actually seeing the flat tire;
If we go ahead and detail every other aspect, from taxes (IPVA and licensing in Brazil), insurance, car loan installments, car washing, and so on.
And the obvious solution is: let's create an app to manage that, except we didn't, quite, because that leads to one more app to download, manage notifications and use storage.

## Central decisions

So what we did was create a platform where users register themselves and their vehicles. An AI agent then uses the vehicle information to generate personalized reminders, which are delivered directly through WhatsApp (widely used in Brazil). The idea is simple: no learning curve, no extra downloads, and no additional app to manage. Since people already use WhatsApp every day for work and personal communication, they can now also rely on it to receive vehicle reminders generated through natural language.

- "Avis, create a reminder to wash my car next friday"
- "Ok, i'll remind you next friday"

And then the user will receive a WhatsApp message to remind him. 

### Technologies

- **WhatsApp**: Widely used in Brazil, no learning curve, no new app to download. The downside is we depend on third party services, and are subject to their pricing, which recently removed a 24h no-cost window, and we have to pay 4 cents per message.
- **Twilio**: Comprehensive documentation, clear pricing, sandbox to enable early development without cost, is an official Meta Business Partner.
- **Gemini**: Considerable free tier, comprehensive documentation, enabled natural language conversations, advanced models available in case of future needs.


## Result

The app has been working for a couple of months, since May 2026, and currently covers the tax methods of Santa Catarina, and soon will be extended to more states.
We already have reports of users that only went to check tire pressure because they were reminded.

---

## Architecture

user -> web platform -> register user (consents to privacy policy and terms of use, validates whatsapp) -> register vehicle (informs license plate, mileage, date of last maintenance, the rest is fetched via api: owner, state, model and manufacturer, year model, etc) -> gemini uses vehicle information to generate custom events -> user receives whatsapp message saying reminders were set -> from this point the user can actively consult their reminders, or passively wait for the messages.

## Tech decisions

### 1. Tax dates calculation by state and license plate

**Issue:** Each state has its own schedule of tax collecting depending on the end char of the plate; Some distribute the tax collecting throughout the year, some condense everything in January.

**Decision:** Since most of the users are from Santa Catarina, we started implementing the core logic of this state, and will progressively expand to more states;

**Result:** Structure is ready to expand to other states, but the main result is now, at least for users with cars from Santa Catarina, the app uses the logic to come with dates that are as close as possible to the real logic, still not definitive because there's an official calendar released in late December, but will also run a cron job to check for that official information and then update users reminders with the official dates;

### 2. WhatsApp/Twilio as interface

**Issue:** There was little to no discussion for this decision, because WhatsApp is really part of most people's lives in Brazil, we didn't want anything the user would have to install, because lots of users have no more memory available on their phones, one more notification to manage is a bummer, and one more WhatsApp notification is just a part of daily life. As for Twilio, there are not too many options of official BSP's that send messages to brazilian numbers, the interface and the sandbox are great.

**Decision:** Meta block any non-templated messages starting from the service, so unless the user initiated the conversation, the messages that are app initiated must have a template approved by meta. That's the main functionality of the app, so if there's an event with scheduled messages for 15, 7, 1 and 0 days before the event, the app will proactively send a user the message with one of the approved templates. If the user initiates the conversation, then Gemini is used to interpret the user's intentions and perform an action, be it show the current events, or scheduling a new one, or cancelling or editing an existing one.

We implemented a queue of messages to the user with TTL context based, to prevent hallucination or stale conversations, or missing data;

The main flows are:
- User initiates conversation -> twilio redirects the message to the backend -> backend runs it against gemini to catch intention and trigger the appropriate action -> gemini or appropriate template message are sent to twilio -> twilio responds the user with the message;
- Every day at 9h00pm the app checks the queue and sends the messages to the users -> user might ignore or respond -> enters the first flow;

We also have rate limits so users don't use too much the natural language conversation, and from a point on, the template messages are the way to go while the cool down is running.

**Result:** Users can perform natural language conversations asking to see their reminders, and updating, deleting or creating new ones. If the rate limit is reached, there's still a pretty much straightforward interface with button oriented actions, such as: 1 - Create, 2 - Edit, and so on.

### 3. Tech stack

#### Monorepo (pnpm + turbo)

**Issue:** API and web share types/schemas and need to build/run together

**Decision:** monorepo with workspaces (apps/api, apps/web, packaged/shared)

**Result:** shared types, parallel builds with turbo and only one repo to manage;

#### Fastify

**Issue:** Low latency API and available plugins (JWT, cookie, CORS, rate-limit, Oauth2)

**Decision:** Fastify over Express/Nest

**Result:** light setup, quick run with tsx watch and access to fastify ecosystem covering auth, cors and form;

#### Postgres + Drizzle

**Issue:** Relational data (users, vehicles, etc) with the need of typed migrations

**Decision:** Postgres + Drizzle ORM/Kit

**Result:** type-safe queries with TS, versioned migrations with db:generate / db:migrate


#### BullMQ + Redis (ioredis)

**Issue:** each event has reminder messages for 15, 7, 1 and 0 days, user decisions involved, context timeouts, official tax calendar only live in late December, need to reconcile with real dates after that.

**Decision:** 6 dedicated queues (notifications, overdue-check, token-cleanup, date-pending-check, conversation-timeout, ipva-reconcile), each one with its own worker. Redis also keeps the conversation state with TTL, not only backend queue.

**Result:** Each worker resolves one isolated business rule; jobs resolved on their own if the event is already done before triggering messages.

#### Zod

**Issue:** payload, dto and form data validation without duplicating rules

**Decision:** Zod as a shared package as the single source of truth

**Result:** input errors caught before reaching business rules

#### Next

**Issue:** frontend needs public routes (landing page, login, register) separate from authenticated routes, mutations (create vehicle, edit event) without mounting a separate rest client, and session protection with silent refresh token

**Decision:** App Router with route groups (`auth` and `app`), Server Actions for mutations and middlewares guarding: blocks protected routes without `access_token`, tries rotation with `refresh_token` before redirecting to login.

**Result:** transparent session renewal, without auth logic spread out throughout code base

---

## Stack

Language: TypeScript (full-stack)
Backend: Fastify, Drizzle ORM, PostgreSQL, BullMQ + Redis (ioredis), Zod
Frontend: Next.js (App Router), React, Tailwind CSS, Sonner, Framer Motion
3rd party services: Twilio (WhatsApp Business API), Google Gemini (NLU), Resend (email)
Infra/tooling: pnpm workspaces + Turbo (monorepo), Docker (Postgres), Drizzle Kit (migrations)

## What I would do differently

- **Automated tests from the start for critical workflows** (6 queues/workers) —
  none exist today, and a silent bug in any worker means a wrong or duplicated
  message to a real user.

- **A structured eval set for the NL intent layer.** Intent recognition for
  actions like creating a reminder isn't regex-based — it's fully delegated to
  Gemini via function calling (`handle-nl-message.ts`), with no deterministic
  fallback. If the prompt changes or the model is updated, there's currently no
  way to catch a regression before it reaches production; a versioned set of
  expected messages → expected function calls, run in CI, would close that gap.
  Right now, if Gemini fails or doesn't respond within 10s, the flow falls back
  to a fixed numeric menu without ever re-validating what the user meant.

- **Idempotency at the webhook entrypoint.** Inbound Twilio webhooks are
  processed synchronously against Redis/Postgres, and `MessageSid`, the ID
  Twilio sends to identify each delivery, is never read anywhere in the
  codebase. If Twilio retries a webhook after a timeout, the handler replays
  from scratch. Downstream scheduling already dedupes via BullMQ job IDs
  (`notification:{eventId}:{date}`), but that protects the outbound side, not
  the entrypoint. The fix is small, a short-TTL `SET NX` on `MessageSid` before
  processing, but it isn't there yet.
