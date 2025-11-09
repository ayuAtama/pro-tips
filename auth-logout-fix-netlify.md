
---

# 🧭 NextAuth Logout & Navbar — Netlify vs Local Runtime Fix Guide

> **Goal:** Make `NextAuth` authentication (especially logout) work both locally and on Netlify (Edge + Functions hybrid).

---

## ⚙️ Overview

You had **two versions** of your `Navbar.tsx` and `LogOutLink.tsx` components.

* One worked locally but **failed on Netlify**
* The other worked **on both local and Netlify**

Below is a full comparison with explanations and fixes.

---

## 🔴 Version A — Broken on Netlify

### 🧩 File: `Navbar.tsx`

```tsx
import { auth } from "@/auth";
import { cookies } from "next/headers";
import { Suspense } from "react";
import Image from "next/image";
import Link from "next/link";
import { signOut } from "next-auth/react";
import LogoutLink from "./LogOutLink";

async function Navbar() {
  // 🚨 Forces Edge Runtime (breaks Netlify)
  cookies(); 
  const session = await auth();

  return (
    <header>
      <nav>
        <Link href="/" className="logo">
          <Image src="/icons/logo.png" alt="logo" width={24} height={24} />
          <p>DevEvents</p>
        </Link>

        <ul>
          <Link href="/">Home</Link>
          <Link href="/dashboard">Dashboard</Link>
          <Link href="/dashboard/create-event" prefetch={false}>
            Create Event
          </Link>
          {/* Uses client logout component */}
          {session && <LogoutLink userName={session.user.name} />}
        </ul>
      </nav>
    </header>
  );
}

function SuspenseWrapper() {
  return (
    <Suspense fallback={<p>Loading...</p>}>
      <Navbar />
    </Suspense>
  );
}

export default SuspenseWrapper;
```

### 🧩 File: `LogOutLink.tsx`

```tsx
"use client";

import { signOut } from "next-auth/react";
import Link from "next/link";

interface LogoutLinkProps {
  userName?: string;
}

export default function LogoutLink({ userName }: LogoutLinkProps) {
  const handleLogOut = (e: React.MouseEvent) => {
    e.preventDefault();
    // 🚨 Relies on NextAuth redirect (often breaks in Netlify)
    signOut({ callbackUrl: "/" });
  };

  return (
    <Link href="/" onClick={handleLogOut} className="hover:underline">
      Welcome {userName}, Logout?
    </Link>
  );
}
```

---

### ⚠️ Why It Breaks on Netlify

| Problem                          | Explanation                                                                                                                            |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **`cookies()` call**             | Forces the component to use the **Edge runtime**, but `auth()` and `signOut()` use **Node (serverless functions)** → runtime mismatch. |
| **Netlify redirect system**      | Netlify handles redirects differently, so `signOut({ callbackUrl })` sometimes **skips the cookie deletion step**.                     |
| **Cookies not cleared**          | Browser still holds `next-auth.session-token`, so user stays logged in.                                                                |
| **Redirects to `/` immediately** | Since `/` calls `auth()`, it re-authenticates before the cookies are gone.                                                             |

---

## 🟢 Version B — Works on Netlify

### 🧩 File: `Navbar.tsx`

```tsx
import { auth } from "@/auth";
import { Suspense } from "react";
import Image from "next/image";
import Link from "next/link";
import LogoutLink from "./LogOutLink";

async function Navbar() {
  // ✅ Removed cookies() call → stays in Node runtime
  const session = await auth();

  return (
    <header>
      <nav>
        <Link href="/" className="logo">
          <Image src="/icons/logo.png" alt="logo" width={24} height={24} />
          <p>DevEvents</p>
        </Link>

        <ul>
          <Link href="/">Home</Link>
          <Link href="/dashboard">Dashboard</Link>
          <Link href="/dashboard/create-event" prefetch={false}>
            Create Event
          </Link>
          {/* Client logout component */}
          {session && <LogoutLink userName={session.user.name} />}
        </ul>
      </nav>
    </header>
  );
}

function SuspenseWrapper() {
  return (
    <Suspense fallback={<p>Loading...</p>}>
      <Navbar />
    </Suspense>
  );
}

export default SuspenseWrapper;
```

### 🧩 File: `LogOutLink.tsx`

```tsx
"use client";

import { signOut } from "next-auth/react";

export default function LogoutLink({ userName }: { userName?: string }) {
  const handleLogOut = async () => {
    try {
      // ✅ Tell NextAuth to delete cookies but don't auto-redirect
      await signOut({ redirect: false });

      // ✅ Give browser time to apply Set-Cookie deletions
      await new Promise((r) => setTimeout(r, 300));

      // ✅ Manually clear any leftover cookies (Netlify edge quirk)
      ["__Secure-next-auth.session-token", "next-auth.session-token"].forEach(
        (n) => {
          document.cookie = `${n}=; path=/; expires=Thu, 01 Jan 1970 00:00:00 GMT;`;
        }
      );

      // ✅ Redirect manually to safe page
      window.location.replace("/login");
    } catch (e) {
      console.error("Logout error:", e);
    }
  };

  return (
    <button
      onClick={handleLogOut}
      className="hover:underline text-blue-500 bg-transparent border-none cursor-pointer"
    >
      Welcome {userName}, Logout?
    </button>
  );
}
```

---

## ✅ Why This Works

| Fix                                   | Explanation                                                               |
| ------------------------------------- | ------------------------------------------------------------------------- |
| ❌ Removed `cookies()`                 | Avoids Edge runtime. Keeps both `auth()` and `signOut()` on Node runtime. |
| ✅ Used `signOut({ redirect: false })` | Avoids Netlify’s redirect handler. Lets you handle redirects manually.    |
| ✅ Manual cookie clearing              | Guarantees logout even if Set-Cookie headers are dropped.                 |
| ✅ Manual redirect to `/login`         | Prevents instant re-auth from homepage (`/`).                             |
| ✅ Small delay                         | Lets browser catch up on cookie removal before redirect.                  |

---

## 🧠 Extra Notes

* **Vercel vs Netlify difference:**

  * Vercel = unified runtime for both Edge and Node → `cookies()` + `auth()` works fine.
  * Netlify = Edge and Functions are separate → must pick one runtime.

* **If you need `cookies()`** (e.g., for reading tokens manually):

  ```tsx
  const isNetlify = process.env.NETLIFY === "true";
  const cookieStore = !isNetlify ? cookies() : undefined;
  ```

  This ensures it only runs on supported environments.

---

## 🪄 Future Debugging Checklist

When logout stops working again, check these:

| Step | Check                                             | Fix                                                                                              |
| ---- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| 1️⃣  | Is `cookies()` used in the same file as `auth()`? | Remove or wrap it in a Netlify check                                                             |
| 2️⃣  | Are you using `signOut({ callbackUrl })`?         | Use `signOut({ redirect: false })`                                                               |
| 3️⃣  | Do cookies persist after logout?                  | Manually clear with `document.cookie = ...`                                                      |
| 4️⃣  | Does your redirect go to `/` immediately?         | Redirect to a static page (`/login`)                                                             |
| 5️⃣  | Still broken?                                     | Try setting `"NEXTAUTH_URL"` and `"NEXTAUTH_SECRET"` correctly in Netlify environment variables. |

---

## 🧩 TL;DR

> **The fix:**
>
> * Don’t call `cookies()` in `Navbar`
> * Use `signOut({ redirect: false })`
> * Manually clear cookies
> * Redirect manually after logout

---

