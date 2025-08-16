# Cashfree Payment Gateway Integration (React & HTML Website)

This guide will help you integrate the **Cashfree Hosted Checkout** payment gateway into your React or HTML website in a secure, production-ready way.  
**Beginner-friendly, step-by-step!**

---

## Why Do You Need a Backend?

> **Never call payment APIs directly from your frontend.**

- Sensitive API keys (like your Cashfree Client Secret) must NOT be exposed in your React/HTML code.
- All payment operations (order creation, status check) must be performed by a backend server.
- The backend acts as a secure bridge between your website and Cashfree.
- Your frontend will only talk to your backend, never directly to Cashfree.

**Popular backend hosting options:**  
- [Vercel](https://vercel.com/) (serverless functions or Node.js)
- [Render](https://render.com/) (Node.js web service)
- [Railway](https://railway.app/)
- Traditional VPS or any Node.js-compatible platform eg. Hostinger, DigitalOcean, AWS, etc.

---

## How Does It Work?

1. **Frontend:** User clicks "Pay" → Sends request to your backend (e.g., `/api/create-order`).
2. **Backend:** Your backend creates a payment order with Cashfree (using secret keys), receives a `payment_session_id`, and sends it back to the frontend.
3. **Frontend:** Uses Cashfree's JS SDK to open the secure hosted checkout page with the session ID.
4. **User:** Completes payment on Cashfree's hosted page.
5. **Frontend:** (After redirect or popup) calls backend to check payment status (Success or Failed).
6. **Backend:** Verifies payment status with Cashfree, responds to frontend.

---

## Official Documentation

- [Cashfree Hosted Checkout Guide](https://docs.cashfree.com/docs/pg/checkout/web/hosted-checkout)
- [Cashfree JavaScript SDK](https://sdk.cashfree.com/js/v3/cashfree.js)
- [Cashfree Node.js SDK](https://www.npmjs.com/package/cashfree-pg)

---

# Setup Instructions

## Prerequisites

- Node.js (v16+ recommended)
- A [Cashfree Merchant Account](https://merchant.cashfree.com/) (for API keys)

---

## 1. Backend Setup

### a. Create a Backend Folder Structure

You can use the code provided in this repo, or create your own Node.js backend using Express.  
Below is a minimal example for `backend/`:

<details>
<summary>Click to expand <code>package.json</code></summary>

```json
{
  "name": "cashfree-backend",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "cashfree-pg": "^3.0.0",
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2"
  }
}
```
</details>

---

<details>
<summary>Click to expand <code>.env.example</code> (rename to <code>.env</code> and fill values)</summary>

```env
CASHFREE_CLIENT_ID=cf_test_xxxxxxxx
CASHFREE_CLIENT_SECRET=cf_test_xxxxxxxx
CASHFREE_ENV=sandbox
FRONTEND_URL=http://localhost:3000
BACKEND_URL=http://localhost:8080
PORT=8080
```
</details>

---

<details>
<summary>Click to expand <code>src/server.js</code></summary>

```js
import express from "express";
import cors from "cors";
import dotenv from "dotenv";
import { Cashfree, CFEnvironment } from "cashfree-pg";

dotenv.config();

const app = express();
app.use(express.json());
app.use(
  cors({
    origin: process.env.FRONTEND_URL?.split(",") ?? process.env.FRONTEND_URL,
    methods: ["GET", "POST", "OPTIONS"],
  })
);

const ENV =
  (process.env.CASHFREE_ENV || "sandbox").toLowerCase() === "production"
    ? CFEnvironment.PRODUCTION
    : CFEnvironment.SANDBOX;

const cashfree = new Cashfree(
  ENV,
  process.env.CASHFREE_CLIENT_ID,
  process.env.CASHFREE_CLIENT_SECRET
);

function generateOrderId() {
  return "order_" + Date.now();
}

app.post("/api/create-order", async (req, res) => {
  try {
    const { amount, currency = "INR", customer = {} } = req.body || {};
    if (!amount || !customer?.email || !customer?.phone) {
      return res
        .status(400)
        .json({ error: "amount, customer.email, and customer.phone are required" });
    }

    const orderId = generateOrderId();
    const returnUrl = `${process.env.FRONTEND_URL}/payment-return.html?order_id=${orderId}`;

    const request = {
      order_id: orderId,
      order_amount: String(amount),
      order_currency: currency,
      customer_details: {
        customer_id: customer.id || customer.email,
        customer_name: customer.name || "",
        customer_email: customer.email,
        customer_phone: customer.phone,
      },
      order_meta: { return_url: returnUrl },
      order_note: "",
    };

    const response = await cashfree.PGCreateOrder(request);
    const data = response?.data || {};

    return res.json({
      orderId: data?.order_id || orderId,
      paymentSessionId: data?.payment_session_id,
    });
  } catch (err) {
    const msg = err?.response?.data || err?.message || "Unknown error";
    return res.status(500).json({ error: "Failed to create order", details: msg });
  }
});

app.get("/api/order/:orderId", async (req, res) => {
  try {
    const { orderId } = req.params;
    const response = await cashfree.PGFetchOrder(orderId);
    return res.json(response.data);
  } catch (err) {
    const msg = err?.response?.data || err?.message || "Unknown error";
    return res.status(500).json({ error: "Failed to fetch order", details: msg });
  }
});

const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log(`Cashfree backend listening on http://localhost:${port}`);
});
```
</details>

---

### b. Install & Run Backend

```bash
cd backend
npm install
npm run dev
```

---

### c. Deploy Backend

- **Vercel:** Use "serverless" API routes or deploy as a Node.js app. Add env variables in dashboard.
(Push your backend code to a GitHub repo, then add a project in Vercel and connect your repo. This will automatically deploy your backend when there is a push to the main branch.)
- **Render:** (Similar to Vercel) Connect your repo, add env variables, deploy as a web service.
- **Other:** Any Node.js-compatible host.

**After deployment, note your backend's public URL** (example: `https://your-backend.vercel.app`).

---

## 2. Frontend Setup

### a. For HTML/JS Websites

- Add [Cashfree JS SDK](https://sdk.cashfree.com/js/v3/cashfree.js) in your HTML.
- On "Pay" button click, send a POST request to your backend `/api/create-order` with user info and amount.
- Open Cashfree checkout using the returned `paymentSessionId`.

**Example:**

```html
<!-- public/index.html -->
<script src="https://sdk.cashfree.com/js/v3/cashfree.js"></script>
<script>
  const BACKEND_BASE_URL = "http://localhost:8080"; // or your deployed backend

  const cashfree = Cashfree({ mode: "sandbox" }); // change to "production" when live

  async function createOrder() {
    const payload = {
      amount: 1,
      currency: "INR",
      customer: {
        name: "John",
        email: "john@example.com",
        phone: "9999999999"
      }
    };
    const res = await fetch(`${BACKEND_BASE_URL}/api/create-order`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload)
    });
    if (!res.ok) throw new Error("Order creation failed");
    return res.json();
  }

  document.getElementById("payBtn").onclick = async () => {
    try {
      const { paymentSessionId } = await createOrder();
      await cashfree.checkout({ paymentSessionId, redirectTarget: "_self" });
    } catch (e) {
      alert("Payment error: " + e.message);
    }
  };
</script>
```

---

### b. For React Websites

- Install SDK: `npm i @cashfreepayments/cashfree-js`
- Use the `load()` function to initialize SDK.
- Call your backend to create order, then call `cashfree.checkout()`.

**Example:**

```jsx
import { useEffect, useRef } from "react";
import { load } from "@cashfreepayments/cashfree-js";
const BACKEND_BASE_URL = "http://localhost:8080"; // or your deployed backend url

export default function App() {
  const cashfreeRef = useRef(null);
  useEffect(() => {
    load({ mode: "sandbox" }).then(cf => { cashfreeRef.current = cf; });
  }, []);

  async function handlePay() {
    const res = await fetch(`${BACKEND_BASE_URL}/api/create-order`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        amount: 1,
        currency: "INR",
        customer: {
          name: "React User",
          email: "react@example.com",
          phone: "9999999999"
        }
      })
    });
    if (!res.ok) throw new Error("Order creation failed");
    const { paymentSessionId } = await res.json();
    await cashfreeRef.current.checkout({ paymentSessionId, redirectTarget: "_self" });
  }

  return <button onClick={handlePay}>Pay Now</button>;
}
```

---

### c. Handle Payment Result

- After payment, user is redirected to your `return_url` (e.g., `/payment-return.html?order_id=...`)
- On this page, call backend `/api/order/:orderId` endpoint to check status.

**HTML Example:**

```html
<script>
  const BACKEND_BASE_URL = "http://localhost:8080";
  const orderId = new URLSearchParams(window.location.search).get("order_id");
  fetch(`${BACKEND_BASE_URL}/api/order/${orderId}`)
    .then(res => res.json())
    .then(data => {
      if (data.order_status === "PAID") {
        document.getElementById("result").textContent = "Payment Successful!";
      } else {
        document.getElementById("result").textContent = "Payment Pending or Failed.";
      }
    });
</script>
```

---

## 3. Testing

- Use **sandbox** keys and mode until you're ready for production. Change variables to `production` when live.
- Make sure your backend is using the correct environment variables.
- Whitelist your frontend domain in Cashfree dashboard (for SDK).
- Check browser console and network logs if payments are not working.

---

## 4. Move to Production

- Switch to production Cashfree keys.
- Set `CASHFREE_ENV=production` in backend.
- Whitelist your production frontend domain in Cashfree dashboard.
- Update all URLs to your deployed backend and frontend.

---

## Security Tips

- Always keep your Frontend and Backend Seperated in different folders and deploy them seperately. Do not create a /backend folder inside your frontend project.
- Never expose Cashfree Secret Key in frontend code.
- Always verify payment status from your backend, not the client-side.
- Allow CORS only for your frontend domain.
- (Optional) Implement [Cashfree webhooks](https://docs.cashfree.com/docs/pg/webhooks/) for even better reliability.

---

## Useful Links

- [Cashfree Checkout Docs](https://docs.cashfree.com/docs/pg/checkout/web/hosted-checkout)
- [SDK Guide](https://docs.cashfree.com/docs/pg/checkout/web/hosted-checkout#step-2-opening-the-checkout-page)
- [API Reference](https://docs.cashfree.com/docs/pg/api/orders/)
- [React SDK Example](https://www.npmjs.com/package/@cashfreepayments/cashfree-js)

---

**Happy Coding! 🚀**