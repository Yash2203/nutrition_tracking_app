---
cursor:
  subagentId: "bc-1304bb38-3061-5d72-b75f-958becf9fda5"
---

# Nutrition tracker: recommended tech stack ($0 now, ready to monetize later)

Research date: 2026-10-05. Free-tier numbers change often, so re-check the linked pages before relying on a specific limit.

**Constraints:**
- Yash uses an **iPhone 16 (iOS 26)**.
- **Friends** will use the app too, so it needs accounts and per-user data.
- **$0 today**, without blocking a later **App Store release, subscriptions/paywall, and paid scaling**.

## TL;DR: the recommended stack

| Layer | Pick | Why |
| --- | --- | --- |
| App delivery, now | **PWA** (Vite + React + TypeScript + `vite-plugin-pwa`), added to the iPhone Home Screen | The only $0 way to put an app on his iPhone *and* his friends' iPhones without weekly re-signing |
| App delivery, later | Wrap the same code with **Capacitor** for the App Store and Play Store | Reuses 100% of the PWA code; native camera, barcode, and IAP plugins replace the web fallbacks |
| Accounts + database | **Supabase** (Auth + Postgres with Row Level Security) | Per-user data enforced in the database; free for up to 50K monthly active users; works the same from PWA, Capacitor, or Expo; clear upgrade path to Pro ($25/mo) |
| Server code + secrets | **Supabase Edge Functions** (`/vision`, `/usda`) | Only signed-in users can call them; holds the Gemini and USDA keys; enforces per-user AI quotas |
| Static hosting | **Cloudflare Pages** | Unlimited free static requests; auto-deploys from GitHub |
| Food database | **USDA FoodData Central** (Foundation + SR Legacy bundled into the app) + **Open Food Facts** for barcodes, with **USDA Branded** as fallback | Cronometer-grade micronutrients for whole foods, label data for packaged foods; CC0 and ODbL licenses both allow commercial use |
| Barcode | **`barcode-detector`** (ZXing WebAssembly) in the PWA, then **`@capacitor-mlkit/barcode-scanning`** in the native app | iOS Safari has no working native `BarcodeDetector` |
| AI food photos | **Gemini Flash free tier** with JSON-schema output, mapped to USDA; **Groq Llama 4 Scout** as fallback | $0 now, a fraction of a cent per photo once paid |
| Local data | **IndexedDB via Dexie.js**, local-first, synced to Supabase | Instant and offline; survives iOS storage rules once installed |
| Payments, later | **RevenueCat** (free up to $2.5K monthly tracked revenue) for StoreKit/Play billing + **Stripe** web checkout | One entitlement system across web, iOS, and Android |
| Where to write code | **GitHub Codespaces** (browser IDE) / Cursor cloud agents; iOS builds later on **Codemagic** (free cloud Macs) | No local dev setup and no Mac needed |

## 1. App delivery: PWA vs Expo/React Native, decided by iPhone + friends + App Store

### What the iPhone allows today at $0

- **Installing a PWA.** On iOS 26, *every* site added via Share → Add to Home Screen opens as a web app by default (the "Open as Web App" toggle is on). No manifest or install prompt is required, and there is no `beforeinstallprompt`, so the app has to show its own "how to install" hint ([WebKit: Safari 26.0](https://webkit.org/blog/17333/webkit-features-in-safari-26-0/), [Apple Support](https://support.apple.com/guide/iphone/open-as-web-app-iphea86e5236/ios), [OpenPWA iOS install reference](https://openpwa.net/reference/installation/ios-add-to-home-screen/)). Friends install it the same way from a link.
- **Camera in a PWA.** `getUserMedia` works in Home Screen web apps ([WebKit 185448](https://bugs.webkit.org/show_bug.cgi?id=185448)), but there are known bugs:
  - Some devices get a black stream in standalone mode ([WebKit 273938](https://bugs.webkit.org/show_bug.cgi?id=273938)).
  - The permission prompt can come back every launch ([field notes on WebKit 215884/252465](https://github.com/ClaraVnk/chaudron/blob/main/docs/technical-notes-scanning.md)).
  - A user reports that setting **Safari → Website Settings → Camera → Allow** for the site stops the repeated prompts on iOS 26.2 ([r/PWA thread](https://www.reddit.com/r/PWA/comments/1jciicq/persisting_permissions_for_ios_safari_pwa/)). Put this tip in the onboarding.
- **Barcode in a PWA.** Safari's `BarcodeDetector` is behind a flag and broken even when the flag is on, including on iOS 26 ([WebKit 281848](https://bugs.webkit.org/show_bug.cgi?id=281848)). A WebAssembly decoder is required (Section 3).
- **Offline and storage.** Safari deletes IndexedDB after 7 days without interaction, but **Home Screen web apps are exempt** ([WebKit tracking prevention](https://webkit.org/tracking-prevention/)). `navigator.storage.persist()` is granted heuristically to Home Screen apps ([WebKit storage policy](https://webkit.org/blog/14403/updates-to-storage-policy/), [MDN](https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria)). Server sync (Section 5) is the real safety net.

### Why not Expo/React Native *now*

Without the $99/yr Apple Developer Program:
- Apps signed with a free Apple ID (your own builds or self-signed Expo Go) **expire after about 7 days**.
- The App Store copy of Expo Go stops at SDK 54.
- **TestFlight, the only easy way to give friends an iOS build, needs the paid program** ([Expo FYI: Expo Go usage](https://github.com/expo/fyi/blob/main/expo-go-usage.md), [Expo FYI: Expo Go on TestFlight](https://github.com/expo/fyi/blob/main/deploy-expo-go-testflight.md), [sign.expo.dev / 7-day summary](https://www.muhammadhuzaifa.com/blog/fix-project-incompatible-with-this-version-of-expo-go)).

So at $0, Expo can reach Yash's phone with weekly re-signing, but **not his friends' iPhones at all**.

### Comparison for the App Store and monetization path

| | PWA now → Capacitor later (**recommended**) | Expo / React Native from day 1 |
| --- | --- | --- |
| $0 install on Yash's iPhone | Yes, from a link | Yes, but re-sign every 7 days |
| $0 install on friends' iPhones | Yes, from a link | **No** (TestFlight needs $99/yr) |
| Android friends | Yes (Chrome install) | Yes (APK sideload) |
| App Store later | Wrap with Capacitor (WebView + native plugins). Guideline 4.2 rejects "repackaged websites", so the native build must use real device features ([Apple guidelines](https://developer.apple.com/app-store/review/guidelines/), [4.2 explainer](https://apprejectionfix.com/app-store-guideline-4-2/), [Capacitor readiness checklist](https://manifestx.dev/docs/publishing/app-store-readiness)) | Native UI; 4.2 is a non-issue |
| Rewrite needed for the store | None; same React code (`npx cap add ios`) ([Capacitor PWA guide](https://capgo.app/blog/transform-pwa-to-native-app-with-capacitor/)) | None, but the UI must be React Native views from the start |
| Native barcode/camera later | `@capacitor-mlkit/barcode-scanning`, `@capacitor/camera` ([plugin](https://www.npmjs.com/package/@capacitor-mlkit/barcode-scanning)) | `expo-camera` |
| Subscriptions | RevenueCat Capacitor SDK + RevenueCat Web/Stripe for the PWA ([RevenueCat Capacitor](https://www.revenuecat.com/docs/getting-started/installation/capacitor)) | RevenueCat RN SDK; web via RevenueCat Billing ([RevenueCat Expo](https://production-docs.revenuecat.com/getting-started/installation/expo)) |
| iOS builds without a Mac | Codemagic: 500 free macOS minutes/month ([pricing](https://docs.codemagic.io/billing/pricing/), [Capacitor guide](https://docs.codemagic.io/yaml-quick-start/building-an-ionic-app/)) | EAS Free: 15 iOS + 15 Android builds/month, OTA updates for 1K MAU ([Expo pricing](https://expo.dev/pricing), [EAS plans](https://docs.expo.dev/billing/plans)) |
| Performance and feel | Web UI; fine for forms, lists, and charts | Native; smoother long lists and animations |

**Verdict:** start with a **PWA built so it can be wrapped by Capacitor**. That is the only path that is $0 *and* reaches friends on iPhone today, and it doesn't block the App Store. Ground rules that keep the wrap cheap later:
- Keep the domain logic (nutrient math, sync, food search) in plain TypeScript modules that don't depend on the DOM.
- Put the camera, barcode, storage, and purchases behind small interfaces (`Scanner`, `PhotoCapture`, `Purchases`) with a web implementation now and a native one later.
- Use mobile-style navigation (tab bar, no hamburger menu) and make offline work fully. That also helps with guideline 4.2.

**Switch to Expo if** Yash decides to pay the $99 early *and* wants a fully native feel (heavy animations, widgets, HealthKit). In that case, do it **before** writing much UI. Supabase, the USDA bundle, the AI pipeline, and RevenueCat all carry over unchanged.

## 2. Food data

| Source | Coverage | Barcode | Limits | License |
| --- | --- | --- | --- | --- |
| **USDA FDC: Foundation** | Hundreds of whole foods; richest analytical micronutrient panels; updated each April and October | No | API 1,000 req/hr; `DEMO_KEY` only 30/hr and 50/day | CC0 public domain (attribution requested) |
| **USDA FDC: SR Legacy** | About 7,800 foods with broad micronutrients; frozen since 2018 | No | 6.7 MB zip (CSV) | CC0 |
| **USDA FDC: FNDDS (Survey)** | About 7,000 "as eaten" dishes with portion weights | No | Download | CC0 |
| **USDA FDC: Branded** | About 1.4M+ products; label nutrients only; quality varies | `gtinUpc` | API (the dump is 2.9 GB) | CC0 |
| **Open Food Facts** | Global packaged foods; micronutrients sparse | `GET /api/v2/product/{code}.json` | **15 product reads/min per IP**, 10 searches/min; custom User-Agent; "1 call = 1 real scan" | ODbL (DB) / DbCL (contents) / CC-BY-SA (images) |

Sources: [FDC API guide](https://fdc.nal.usda.gov/api-guide/), [api.data.gov limits](https://api.data.gov/docs/developer-manual/), [FDC data types](https://fdc.nal.usda.gov/data-documentation/), [FDC downloads](https://fdc.nal.usda.gov/download-datasets/), [FDC FAQ](https://fdc.nal.usda.gov/faq), [which FDC type when](https://selfhostednutrition.org/api/usda-foundation-vs-survey-vs-branded/), [OFF API intro](https://openfoodfacts.github.io/documentation/docs/Product-Opener/api/), [OFF data/reuse](https://world.openfoodfacts.org/data), [OFF nutrition refactor](https://blog.openfoodfacts.org/en/news/data-in-open-food-facts).

**Plan:**
- **Bundle** Foundation, SR Legacy, and (optionally) FNDDS into a compact static file in the app. Search becomes instant and offline, and it uses no per-user API quota, which matters once many users share one USDA key.
- **Barcode:** the phone calls OFF directly. Rate limits are per IP, so each user has their own budget, and OFF requires no key. If OFF has no result, call `/usda` (Edge Function) for a Branded `gtinUpc` match.
- **Shared product cache:** store resolved barcodes in a shared `products` table in Supabase, so one friend's scan helps everyone. It also cuts calls to OFF and USDA.
- **Licensing for a paid app:** USDA (CC0) has no restrictions. OFF's ODbL allows commercial use but requires attribution, and any *publicly shared* derived database must stay ODbL. Show "Data: Open Food Facts (ODbL)" and keep OFF-derived rows in their own table. Branded label data has limited micronutrients, the same limitation Cronometer has.

## 3. Barcode scanning (iOS-first)

- **PWA:** use [`barcode-detector`](https://www.npmjs.com/package/barcode-detector) (a `BarcodeDetector` polyfill on [zxing-wasm](https://github.com/sec-ant/zxing-wasm); EAN-13/8 and UPC-A/E; v3.2.1 from July 2026 per its [releases](https://github.com/Sec-ant/barcode-detector/releases/tag/v3.2.1)). Configure it this way:
  - **Self-host the `.wasm` file** (`prepareZXingModule`) so scanning works offline instead of loading it from jsDelivr.
  - Always use the polyfill on iOS, and the native API on Android Chrome when `getSupportedFormats()` includes `ean_13`.
  - Fallbacks: still-photo decode via `<input type="file" accept="image/*" capture="environment">`, which avoids the live-stream bugs, and manual barcode entry.
  - Alternative decoder: [ZBar compiled to WebAssembly](https://eringen.com/blog/barcode-scanning-on-ios-the-missing-web-api-and-a-webassembly-solution/).
- **Native app later:** use [`@capacitor-mlkit/barcode-scanning`](https://github.com/capawesome-team/capacitor-mlkit/tree/main/packages/barcode-scanning) (fast native scanner, torch, and autofocus). Choose the implementation behind the `Scanner` interface at runtime.

## 4. AI food photo recognition

| Provider | Vision model | Free limits (reported) | Notes |
| --- | --- | --- | --- |
| **Google Gemini API** | Current Flash (e.g. `gemini-3.8-flash`) or Flash-Lite | About 10 RPM and 1,000–1,500 RPD per *project* for Flash-class models; check AI Studio for your actual quota | See the terms below. JSON-schema output supported. Keys must be API-restricted. |
| **Groq** | `meta-llama/llama-4-scout-17b-16e-instruct` | 30 RPM, 1,000 RPD, 30K TPM per org | Preview model; scored lower than Gemini/GPT in food benchmarks |
| **OpenRouter** | `:free` models | 20 RPM, **50 RPD** (1,000 after a one-time $10 top-up) | Low daily cap |
| **Cloudflare Workers AI** | `@cf/meta/llama-3.2-11b-vision-instruct` | 10,000 neurons/day | Weaker; emergency fallback only |
| On-device open models | n/a | n/a | Not practical in a phone browser today |

Sources: [Gemini pricing](https://ai.google.dev/gemini-api/docs/pricing), [Gemini rate limits](https://ai.google.dev/gemini-api/docs/rate-limits), [third-party free-tier table](https://pecollective.com/tools/gemini-free-tier-guide/), [restricted-key notice](https://discuss.ai.google.dev/t/free-tier-api-image-generation/177423), [structured output](https://ai.google.dev/gemini-api/docs/structured-output), [regions](https://ai.google.dev/gemini-api/docs/available-regions), [Groq limits](https://console.groq.com/docs/rate-limits), [Groq free table](https://www.eesel.ai/blog/groq-pricing), [OpenRouter free tier](https://klymentiev.com/blog/openrouter-free-tier), [free LLM API comparison](https://openrouter.ai/blog/tutorials/free-llm-apis-compared/), [Workers AI pricing](https://developers.cloudflare.com/workers-ai/platform/pricing/).

**Gemini terms that matter once friends or the public use it** ([Gemini API Additional Terms](https://ai.google.dev/gemini-api/terms), [billing FAQ](https://ai.google.dev/gemini-api/docs/billing), [Google forum clarification](https://discuss.ai.google.dev/t/clarification-on-only-paid-services-for-eea-ch-uk/107860)):
- **Users must be 18+.** The API may not be used in an app "directed towards or likely to be accessed by" under-18s. Add an 18+ statement to sign-up.
- **Users in the EEA, UK, or Switzerland may only be served with Paid Services**, meaning a project with billing enabled. Location is judged primarily by IP. If any friend is there, either route their photos to Groq, or enable billing on the Gemini project, which still costs nearly nothing within the free quotas.
- **Free-tier content is used to improve Google's products.** Disclose this in the app's privacy note, and send only the food photo.
- The **quota is per project and shared by all users**. The Edge Function should enforce a per-user daily cap (e.g. 15 photos/day) using a `ai_usage` table.

**Accuracy:**
- Photo-only estimates are off by about 36% on weight and energy (mean absolute percentage error) for GPT-4o and Claude, and 64–110% for Gemini 1.5 Pro. Every model underestimates large portions ([PMC12513282](https://pmc.ncbi.nlm.nih.gov/articles/PMC12513282/)).
- Supplying the **true weight** cut Gemini 2.5 Flash's carbohydrate error from 56.6% to 20.2% ([ODU benchmark](https://digitalcommons.odu.edu/cgi/viewcontent.cgi?article=1422&context=computerscience_fac_pubs)).
- Choice of model dominates; prompt tricks and multiple photo angles barely help; nutritionists still win by a wide margin ([Sci Rep 2026](https://www.nature.com/articles/s41598-026-58755-w)). Volume cues help ([PMC11990770](https://pmc.ncbi.nlm.nih.gov/articles/PMC11990770/)).

**Pipeline: "identify, then look up, then confirm"**
1. The phone downscales the photo to about 1024 px and calls `supabase.functions.invoke('vision')`. The user's JWT is verified automatically ([Edge Function auth](https://supabase.com/docs/guides/functions/auth)).
2. The function checks the per-user quota, then calls Gemini with a JSON schema such as `{items:[{name, usda_search_term, cooking_method, grams, grams_low, grams_high, confidence}], notes}`. The prompt asks the model to use the plate and cutlery for scale and to list oils and sauces separately.
3. The client fuzzy-matches each term against the bundled USDA database (Foundation and SR Legacy first) and computes nutrients as grams/100 × per-100 g values. **The LLM never supplies nutrient numbers.**
4. A review screen shows each item with a grams slider and lets the user swap the matched food. Nothing is logged until the user confirms.
5. Accuracy boosters: a typed or scale weight, a text hint ("2 eggs, 1 tbsp butter"), and remembered usual portions per user.
6. If Gemini returns 429 or the user is in the EEA/UK, fall back to Groq, then to manual search.

**Paid cost later (rough estimate):** with Gemini 3.8 Flash at $0.75 per 1M input tokens and $3.75 per 1M output tokens through 2026 ([pricing](https://ai.google.dev/gemini-api/docs/pricing)), and assuming about 1.5K input and 0.5K output tokens per photo, each photo costs roughly **$0.003**. 1,000 users × 3 photos/day would cost about **$270/month**. Flash-Lite is cheaper. AI photos are a natural **premium feature** to put behind the paywall.

## 5. Accounts, per-user data, sync, hosting, and keys

### Backend choice: Supabase

- **Auth.** Free plan: 50K monthly active users, social OAuth included, custom SMTP allowed ([pricing](https://supabase.com/pricing)).
  - **Free sign-in today:** **Google OAuth**, plus email/password.
  - The built-in email sender is capped at **2 emails/hour** per project ([auth rate limits](https://supabase.com/docs/guides/auth/rate-limits)). That is fine for a few friends confirming sign-ups, but not for public launch. Later, add custom SMTP; for example Resend's free plan gives 3,000 emails/month, but it needs a domain you own ([Supabase SMTP](https://supabase.com/docs/guides/auth/auth-smtp), [Resend + Supabase](https://www.resend.com/docs/knowledge-base/getting-started-with-resend-and-supabase)).
  - **Sign in with Apple** needs the paid Apple program. Add it when you go to the App Store: guideline **4.8** requires a privacy-preserving login option (Sign in with Apple qualifies) alongside Google sign-in.
  - Guideline **5.1.1(v)** requires **in-app account deletion** ([Apple guidelines](https://developer.apple.com/app-store/review/guidelines/)). Build both from the start.
- **Per-user data.** Every user table has `user_id uuid default auth.uid()` plus an RLS policy `using (user_id = auth.uid())`. Tables: `profiles` (targets, units), `diary_entries`, `custom_foods`, `recipes`, `ai_usage`; plus a shared, read-only-to-users `products` barcode cache, written only by the Edge Function.
- **Local-first sync.** Dexie (IndexedDB) is the UI's source of truth. Each row has `id` (UUID made on the client), `updated_at`, and `deleted`. Sync pushes changed rows with an upsert and pulls rows where `updated_at > last_sync`. The conflict rule is last-write-wins, which is fine for a single person's diary. Ship a JSON/CSV **export** too.
- **Free-tier caveats** ([pricing](https://supabase.com/pricing), [pausing](https://supabase.com/docs/guides/platform/free-project-pausing), [Edge Function limits](https://supabase.com/docs/guides/functions/limits)):
  - The free plan has 500 MB DB, 5 GB egress, and 500K Edge Function invocations, which is ample for friends.
  - **Free projects pause after about 7 days of low DB activity.** Daily use by a few people prevents it; a scheduled GitHub Action that runs a tiny query daily is a backstop.
  - **No automatic backups on Free.** Add a weekly `pg_dump` GitHub Action to a private location, and keep the per-user export.
  - Pro ($25/mo) removes pausing and adds 7-day backups. Upgrade when real users depend on it.
- **Why not the others:**
  - **Firebase** (Spark: Firestore 1 GiB, 50K reads and 20K writes/day, built-in offline cache) is a solid alternative. However, a NoSQL diary is clumsier to query, and Cloud Storage now needs the Blaze plan ([Firebase pricing](https://firebase.google.com/pricing/), [storage change](https://firebase.google.com/docs/storage/faqs-storage-changes-announced-sept-2024)).
  - **Cloudflare D1** (5 GB free; [pricing](https://developers.cloudflare.com/d1/platform/pricing/)) has no auth system built in, so you would have to build accounts yourself.

### Hosting and keys

- **Cloudflare Pages** serves the PWA. Static requests are free and unlimited ([Pages pricing](https://developers.cloudflare.com/pages/functions/pricing/)). GitHub Pages also works; Vercel Hobby is non-commercial only.
- **Secrets live only in Supabase Edge Function secrets:** `GEMINI_API_KEY`, `GROQ_API_KEY`, `USDA_API_KEY`. The browser holds only the Supabase *publishable* key, which is safe to expose because RLS protects the data.
- Functions use `verify_jwt = true`, so anonymous callers can't spend the AI quota.
- In Google Cloud, restrict the Gemini key to the Generative Language API.
- Don't put a service-role/secret key in the client, ever.

## 6. Monetization and scaling path (nothing to pay for until it's needed)

| Step | What changes | Cost |
| --- | --- | --- |
| Friends beta (now) | PWA + Supabase Free + Gemini free tier | $0 |
| Web paywall | RevenueCat Web (RevenueCat Billing on Stripe) or Stripe Checkout; entitlement stored in Supabase | RevenueCat free under $2.5K MTR, then 1%; Stripe 2.9% + 30¢ ([RevenueCat pricing](https://www.revenuecat.com/pricing), [billing](https://www.revenuecat.com/docs/welcome/set-up-revenuecat/account-management), [RevenueCat Web](https://www.revenuecat.com/docs/web/overview)) |
| App Store / Play | `npx cap add ios android`; native scanner/camera plugins; Sign in with Apple; account deletion; IAP via `@revenuecat/purchases-capacitor`; build on Codemagic | Apple $99/yr; Google Play one-time developer fee (not verified here) |
| Production backend | Supabase Pro (no pausing, backups); Gemini billing enabled (needed for EEA/UK anyway) | $25/mo + about $0.003 per AI photo |
| Custom domain + email | Domain + Resend SMTP for auth emails | About $10–15/yr for the domain (typical; not verified here) |

- **iOS payment rules.** Digital subscriptions sold in the iOS app must offer **In-App Purchase** (Apple takes up to 30%) ([fkks summary](https://technologylaw.fkks.com/post/102mp7y/update-your-app-external-purchase-developments-for-apple-google)). On the **US storefront**, the app may *also* link out to a web checkout in the default browser. Apple currently takes no commission on those link-out purchases, but a fee is pending in court ([Apple 3.1.1(a)](https://developer.apple.com/app-store/review/guidelines/), [Stripe guide](https://stripe.com/guides/how-app-developers-can-reduce-fees-and-create-an-optimized-checkout), [MacObserver](https://www.macobserver.com/news/app-store-link-out-commission-rate-not-set-us/)). RevenueCat unifies both under one entitlement.
- **Design now so this is cheap later:** an `entitlements` table keyed by `user_id`, feature flags such as `canUsePhotoAI`, and the paywall UI behind the `Purchases` interface.

## 7. Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| iOS standalone camera bugs or repeated permission prompts | **First milestone: camera + barcode smoke test on his iPhone 16 in Home Screen mode.** Onboarding tip to set Safari → Website Settings → Camera → Allow. Still-photo and manual fallbacks. Native Capacitor scanner later. |
| Friends unsure how to install a PWA (no install prompt on iOS) | An in-app "Share → Add to Home Screen" guide, shown when `navigator.standalone` is false. |
| App Store rejection of the wrapped app (4.2) | Native scanner/camera, offline mode, haptics, and push in the Capacitor build. If 4.2 still blocks it, move the UI to Expo, reusing the backend and logic. |
| Gemini terms (18+, EEA/UK paid-only, training on free data) | 18+ gate, route EEA/UK users to Groq or enable billing, privacy note. |
| Shared free quotas (Gemini per project, USDA 1,000/hr per key) | Per-user AI caps, bundled USDA data, shared barcode cache. |
| Supabase Free pausing or no backups | Daily keep-alive and weekly `pg_dump` via GitHub Actions; move to Pro when it matters. |
| Supabase email limit (2/hour) | Google sign-in as the main option; custom SMTP once there's a domain. |
| Free tiers or terms change | AI provider behind one Edge Function; manual logging always works. |
| Photo estimates are about 35% or more off | Mandatory grams-confirmation step, kitchen-scale entry, AI positioned as a shortcut. |
| Health data privacy once public | Privacy policy, data export and deletion, nothing shared with third parties beyond the photo sent to the AI provider. |

## 8. Setup steps Yash does himself (all $0, no card)

1. **GitHub:** create a private repo. Edit it in **Codespaces** (free: 120 core-hours/month, about 60 hours on 2 cores, plus 15 GB; usage just stops at the quota if no card is on file) ([GitHub docs](https://docs.github.com/en/billing/reference/product-usage-included), [Codespaces billing](https://docs.github.com/en/billing/concepts/product-billing/github-codespaces)), or let an AI coding agent write the code.
2. **Supabase** ([supabase.com](https://supabase.com)): create a Free project in the region closest to him.
   - Authentication → enable Email and **Google**.
   - Copy the project URL and the **publishable** key into the app's environment.
3. **Google Cloud** (free): create an OAuth client (Web) for Supabase Google sign-in. Paste the client ID and secret into Supabase → Auth → Providers → Google.
4. **Google AI Studio** ([aistudio.google.com](https://aistudio.google.com)): create an API key, restrict it to the Generative Language API, and don't enable billing yet. Run `supabase secrets set GEMINI_API_KEY=...`.
5. **Groq** ([console.groq.com](https://console.groq.com)): create an API key and set it as `GROQ_API_KEY`.
6. **api.data.gov** ([signup](https://api.data.gov/signup/)): get the USDA key and set it as `USDA_API_KEY`.
7. **Cloudflare** (free): Pages → Connect to Git → pick the repo (build `npm run build`, output `dist`). Add the `pages.dev` URL to Supabase Auth → URL Configuration (Site URL and redirect URLs).
8. **iPhone 16:** open the URL in Safari, then Share → Add to Home Screen (keep "Open as Web App" on). Launch it from the icon, allow the camera, and set Safari → Website Settings → Camera → Allow for the site. Run the scan test.
9. **Friends:** send them the URL and the same install steps; each signs in with Google.
10. **Later, only when monetizing:** Apple Developer Program ($99/yr), RevenueCat account, Stripe account, Codemagic account (free tier), a domain plus Resend, and Supabase Pro.

## Build order (suggested)

1. Build the PWA shell, Home Screen install guide, and **camera + barcode test on the iPhone 16**. This is the go/no-go check.
2. Add Supabase auth (Google + email), RLS tables, and in-app account deletion.
3. Add the USDA bundle and offline search, manual logging, and daily totals with a micronutrient dashboard.
4. Add Dexie local-first sync and export.
5. Add barcode lookup (OFF, then the `/usda` Edge Function) with the shared products cache.
6. Add the AI photo pipeline with per-user quotas and the confirmation screen.
7. When ready to charge: the entitlements table and paywall, then Capacitor and the App Store.
