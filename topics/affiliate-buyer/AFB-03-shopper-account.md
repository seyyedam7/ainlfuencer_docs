---
id: AFB-03
title: Shopper account
domain: Buyer site (shoppers)
audience: [buyer]
apps: [buyer-site, email]
related: [AFB-01, AFB-02, AFB-04]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-buyer@v1.3.04, affiliate/ainflu-affiliate-backend@v1.5.27
---

# AFB-03 · Shopper account

> Shoppers can optionally create a free account on the Ainfluencer shopper site. Today its only real benefit is **seeing coupon codes** on product pages. Shopping itself never needs an account.

## Who and where
- **Shoppers** on the buyer site. The account pop-ups open from the header buttons on product pages and storefronts, from "Login to See" on a coupon, or from "Login/Register" in the welcome banner.
- Pop-ups: sign in (`?modal=signin`), sign up (`?modal=signup`), email code (`?modal=code`), forgot password (`?modal=forgotpassword`), complete profile (`?modal=completeprofile`).
- The header shows sign-up/sign-in buttons only on product pages and storefronts.

## How it works
1. **Sign up** ("Create your new account for free"): "Full Name", "Email", "Password", accept the terms, reCAPTCHA → "Signup". A verification code is emailed.
2. **Confirm email:** the shopper enters the code → "Confirm" ("Resend now" sends a new code). They are then logged in.
3. **Sign in:** "Email" + "Password" + reCAPTCHA → "Login".
4. **Social sign-in** ("Or Continue With" / "Or Signup With"): **Google, Facebook or Amazon** in a small popup window. After a first social sign-up the shopper is meant to see "complete profile" to set a name.
5. **Once logged in**, coupon codes on product pages become visible and can be copied (AFB-01). The header shows an avatar menu with "Logout".

## Rules and limits
- Optional. Buying, prices and links work the same without an account.
- No orders, carts, wishlists or messages exist on the shopper site.
- A shopper account is **not** an affiliate account. To earn commission, a person needs to register in the affiliate app (AFA-01). The same email can hold both.
- The email code expires after **1 hour** *(based on the shared verification setting; to verify for buyers)*.

## Statuses
None visible to shoppers.

## Timers and automatic actions
None.

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Sign-up / "Resend now" | Shopper | Email with verification code |
| Forgot password | Shopper | Email with a reset link (**doesn't work**, see Known issues) |

## Common questions

### "Why should I create an account?"
**Answer for the user:** It's free and lets you see the coupon codes creators share, plus future offers. You can shop without it.
**What to check:** —
**Escalate when:** —

### "I forgot my password and the reset link doesn't work."
**Answer for the user:** Sorry, password reset on the shopper site isn't working at the moment. Please sign in with Google, Facebook or Amazon using the same email, or create a new account with another email.
**What to check:** Nothing to fix on our side; known issue.
**Escalate when:** Report the case to dev (known issue) if the user can't use social sign-in.

### "I didn't get the verification code."
**Answer for the user:** Check your spam folder, then use "Resend now" in the code window.
**What to check:** The email address typed.
**Escalate when:** Several shoppers report missing codes.

### "I'm logged in but still see 'Login to See'."
**Answer for the user:** Please refresh the page. If it persists, sign out and in again.
**What to check:** Whether the session is still valid.
**Escalate when:** It persists after signing in again.

### "Can I track my order here?"
**Answer for the user:** No. Orders are placed on Amazon or the brand's store; please track them there.
**What to check:** —
**Escalate when:** —

## Known issues
- **Password reset can't be completed.** The emailed link points to a page that doesn't exist on the shopper site. Workaround: social sign-in. Escalate to dev.
- **"Dashboard" and "Saved products"** in the avatar menu lead to "page not found".
- **Logout** only signs out in the browser; the server session isn't ended.
- After a first social sign-up, the "complete profile" pop-up probably never opens (the page reloads first) *(to verify)*.

## Related topics
- AFB-01 · Product page and coupons.
- AFB-02 · Storefront.
- AFB-04 · Applying for partnership on brand pages (a shopper session there causes an error).
- AFA-01 · Affiliate sign-up (a different account type).

## Technical reference
- **Endpoints:** `POST client/buyer/v1/auth/register`, `POST client/buyer/v1/auth/verification-code`, `POST client/buyer/v1/auth/email-verify`, `POST client/buyer/v1/auth/login`, `POST client/buyer/v1/auth/login/{google|facebook|amazon}` (callbacks `GET web/v1/buyer/login/*/callback` → buyer `/auth?token=&is_registered=`), `GET buyer/v1/auth/auth-status`, `POST buyer/v1/profile`, `POST client/buyer/v1/auth/forget-password`.
- Reset email URL: config `sanctum.buyer.reset_password.url` = `BUYER_BASE_URL/user/reset-password` (no such page; `modals/ResetPassword.tsx` is never used).
- **Buyer code:** `aff-buyer/managers/AuthManager.tsx`, `modals/*`, `lib/api/nextAuthOptions` (NextAuth credentials; `session.accessToken`), `components/Header/UserProfile.tsx`.
- **Data:** `users`, `user_emails`, `user_types` (buyer type; every affiliate/seller user also gets a buyer type).
