
---


# 🧩 Fixing Auth.js Session Not Detected on First Visit (Next.js 16)

## 🧠 Problem Summary

After a successful login, visiting a protected route (like `/create-event`) immediately after login causes a **redirect back to `/login`**, even though the user is already authenticated.

However, if the page is **refreshed** or revisited later, it shows the correct signed-in content.

### 🔍 Symptoms
- Experimental feature "use cache" **ON**
- Login succeeds.
- Navigating to `/create-event` (client-side) shows the login page again.
- Refreshing `/create-event` makes the form appear (session is now detected).
- Waiting before clicking the link does **not** fix the issue.

---

## 🧩 Root Cause

Next.js 16 (with component caching) prefetches React Server Components (RSC) **before login**, and keeps that snapshot cached in memory.

- When you click `/create-event`, it reuses the **prefetched unauthenticated snapshot**, where `auth()` returned `null`.
- After a refresh, the cache is invalidated and `auth()` runs again on the server → correct session is detected.

This is **not** a race condition — it’s **RSC prefetch caching** combined with **Auth.js cookie session** behavior.

---

## ✅ Working Solution (Recommended)

### Step 1 — Force dynamic rendering using cookies

At the top of your protected page file (e.g. `app/(root)/create-event/page.tsx`):

```tsx
import { cookies } from "next/headers";
import { Suspense } from "react";
import NewEventFormPage from "./form";
import { redirect } from "next/navigation";
import { auth } from "@/auth";

const AuthWrapper = async () => {
  cookies(); // Force Next.js to treat this route as dynamic
  const session = await auth();

  if (!session) redirect("/login");

  return <NewEventFormPage />;
};

export default function Page() {
  return (
    <Suspense
      fallback={<div className="max-w-3xl mx-auto p-6">Loading...</div>}
    >
      <AuthWrapper />
    </Suspense>
  );
}
```

✅ This ensures the route checks the latest cookies on every request.
✅ You don’t need `export const dynamic = "force-dynamic";`, which conflicts with `cacheComponents`.

---

### Step 2 — After login, **force a full reload or refresh** to reset the RSC cache

If your login logic uses `router.push()` or `redirect()`, replace it with one of the following:

#### Option A — Hard reload (most reliable)

```tsx
// After successful login
window.location.href = "/";
```

This ensures the next page load is **server-rendered** with fresh session cookies.

#### Option B — Soft reload with SPA navigation

```tsx
import { useRouter } from "next/navigation";

const router = useRouter();

const handleLogin = async () => {
  const result = await signIn("credentials", { redirect: false });
  if (result?.ok) {
    router.refresh(); // clears the stale RSC cache
    router.push("/"); // navigate to homepage or another route
  }
};
```

---

### Step 3 (Optional) — Disable prefetch on critical links

If your protected routes (like `/create-event`) are linked in navigation menus, add `prefetch={false}`:

```tsx
<Link href="/create-event" prefetch={false}>
  Create Event
</Link>
```

This prevents Next.js from preloading the route with an outdated unauthenticated snapshot.

---

## ✅ TL;DR Summary

| Problem                                                    | Cause                                | Fix                                                 |
| ---------------------------------------------------------- | ------------------------------------ | --------------------------------------------------- |
| Protected page redirects to login after successful sign-in | Prefetched RSC cached before login   | Force reload or call `router.refresh()` after login |
| `export const dynamic` breaks build                        | `cacheComponents` enabled in Next 16 | Use `cookies()` instead                             |
| Refresh fixes issue                                        | Cache invalidation refreshes session | Expected behavior                                   |

---

## 📘 Example File Structure

```
app/
  (root)/
    create-event/
      form.tsx
      page.tsx  <-- contains cookies() + auth() logic
auth/
  auth.ts     <-- contains your Auth.js setup
```

---

## 🧩 Key Takeaways

- This issue isn’t a timing bug — it’s caused by **React Server Component caching**.
- Use `cookies()` or `headers()` to mark a route as dynamic when using Auth.js.
- After login, force a **full reload** (`window.location.href`) or **cache refresh** (`router.refresh()`).
- Avoid `export const dynamic = "force-dynamic"` when `cacheComponents` is enabled.

---

✅ **Final verified setup:**

- `cookies()` used in protected page → always dynamic
- `window.location.href` after login → ensures fresh SSR load
- No stale prefetch snapshots
- No unnecessary config changes
- Works reliably across all devices and browsers

---

**Author:** Wahyu Pratama
**Context:** Fix for Auth.js + Next.js 16 session caching issue after login
**Last verified:** 2025-10-31