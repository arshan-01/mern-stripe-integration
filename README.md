# stripe-integration

In this guide, we'll walk through creating a Stripe checkout session for an online service, using Node.js and MongoDB. We'll cover key steps like calculating prices, creating a Stripe checkout session, and handling webhooks to process orders.

**1. Setting Up Stripe Checkout**

```
npm install stripe

```
Next, initialize Stripe with your secret key:

```
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);
```
**2. Price Calculation Function:**

```
// Sample price calculation based on cart items
async function calculateTotalPrice(cartItems) {
  let totalPrice = 0;
  for (const item of cartItems) {
    totalPrice += item.price * item.quantity;
  }
  return totalPrice;
}
```
**3. Create Checkout Session Function:**
```
export const createCheckoutSession = async (req, res) => {
  try {
    const { cartItems } = req.body; // Expecting cartItems array in request body

    // Sample cart data
    const sampleCartItems = [
      {
        itemId: "1",
        itemName: "Leather Bag",
        quantity: 1,
        price: 50.00, // in USD
        currency: "USD",
        description: "A stylish leather bag",
        image: "https://example.com/images/leather-bag.jpg"
      },
      {
        itemId: "2",
        itemName: "Cotton Shirt",
        quantity: 2,
        price: 25.00, // in USD
        currency: "USD",
        description: "A soft cotton shirt",
        image: "https://example.com/images/cotton-shirt.jpg"
      }
    ];

    // Calculate total price
    const totalPrice = await calculateTotalPrice(sampleCartItems);
    
    // Create a Stripe customer for this checkout session
    const customer = await stripe.customers.create({
      metadata: {
        cartItems: JSON.stringify(sampleCartItems), // Storing cart in metadata
      }
    });

    // Prepare line items for Stripe
    const line_items = sampleCartItems.map(item => ({
      price_data: {
        currency: item.currency,
        product_data: {
          name: item.itemName,
          description: item.description,
          images: [item.image],
        },
        unit_amount: Math.round(item.price * 100),  // Convert price to cents
      },
      quantity: item.quantity,
    }));

    // Create Stripe checkout session
    const session = await stripe.checkout.sessions.create({
      payment_method_types: ["card"],
      line_items,
      mode: "payment",
      customer: customer.id,
      success_url: `${req.headers.origin}/checkout-success?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: `${req.headers.origin}/checkout-cancelled`,
    });

    res.send({ url: session.url });
  } catch (err) {
    console.error("Checkout session creation error:", err);
    res.status(500).send({ error: "Failed to create checkout session." });
  }
};
```

**4. Handle Stripe Webhook:**
```
const createOrder = async (customer, data) => {
  const cartItems = JSON.parse(customer.metadata.cartItems);

  const newOrder = new Order({
    customerId: customer.id,
    cartItems: cartItems,  // Store cart items in the order
    payment_status: data.payment_status,
    payment_intent: data.payment_intent,
    totalAmount: data.amount_total / 100,  // Convert from cents to dollars
    status: "PROCESSING",
    createdAt: new Date(),
  });

  try {
    await newOrder.save();

    // Send a notification after order creation
    const notification = new Notification({
      userId: customer.id,
      orderId: newOrder._id,
      message: `Your order with ID ${newOrder._id} was successfully created.`,
    });
    await notification.save();
  } catch (err) {
    console.error("Failed to save order:", err);
  }
};

export const handleStripeWebhook = async (req, res) => {
  const webhookSecret = process.env.STRIPE_WEBHOOK_SECRET;
  let event;
  
  try {
    const signature = req.headers["stripe-signature"];
    event = stripe.webhooks.constructEvent(req.body, signature, webhookSecret);
  } catch (err) {
    console.log(`⚠️  Webhook signature verification failed: ${err.message}`);
    return res.sendStatus(400);
  }

  const data = event.data.object;
  const eventType = event.type;

  if (eventType === "checkout.session.completed") {
    stripe.customers.retrieve(data.customer)
      .then(async (customer) => {
        try {
          await createOrder(customer, data);
        } catch (err) {
          console.error("Error creating order:", err);
        }
      })
      .catch(err => console.error("Error retrieving customer:", err));
  }

  res.status(200).end();
};
```
**5. Handling routes**
```
// Adjust controllers and path 
import express from "express";
import { createCheckoutSession, handleStripeWebhook } from "../controllers/payment/payment.js";
import { isUser } from "../middleware/auth.js";
import bodyParser from "body-parser";

const router = express.Router();

// Stripe Checkout Session route
router.post("/create-checkout-session",  express.json({ type: "application/json" }),isUser, createCheckoutSession);
  
// Use the raw body parser only for Stripe webhooks
router.post('/webhook', bodyParser.raw({ type: 'application/json' }), handleStripeWebhook);

export default router;
```

```
Put this toute above these middleware
app.use("/payment", payment);

//app.use(express.json());
app.use(express.static("public"));
app.use(express.urlencoded({ extended: true }));
app.use(express.json({ limit: "10mb" }));

```
**6. Frontend **
 

```
 const handleStripeCheckout = async () => {
        setIsLoading(true);
        try {
          const response = await api.post(`${config.BASE_URL}/payment/create-checkout-session`, {
            cartItems,
          });
      
          console.log("Stripe response:", response); // Log the response
          if (response.data.url) {
            window.location.href = response.data.url; // Redirect to Stripe checkout
          } else {
            console.error("No URL returned from Stripe:", response.data);
          }
        } catch (err) {
          console.error("Stripe Checkout Error:", err);
          Swal.fire({
            title: "Error",
            text: "There was an issue processing your payment.",
            icon: "error",
            confirmButtonText: "OK",
          });
        }
        setIsLoading(false);
      };
```
**7. Tailwind Success Page ( optional )**
```
import React from 'react'
import { Link } from 'react-router-dom'

function CheckoutSuccess() {
  return (
    <div className="h-screen flex justify-center items-center bg-gray-100">
      <div className="bg-white p-6 rounded-lg shadow-md text-center max-w-md">
        <svg viewBox="0 0 24 24" className="text-green-600 w-16 h-16 mx-auto my-6">
          <path fill="currentColor"
            d="M12,0A12,12,0,1,0,24,12,12.014,12.014,0,0,0,12,0Zm6.927,8.2-6.845,9.289a1.011,1.011,0,0,1-1.43.188L5.764,13.769a1,1,0,1,1,1.25-1.562l4.076,3.261,6.227-8.451A1,1,0,1,1,18.927,8.2Z">
          </path>
        </svg>
        <h3 className="md:text-2xl text-base text-gray-900 font-semibold">Payment Done!</h3>
        <p className="text-gray-600 my-2">Thank you for completing your secure online payment.</p>
        <p>Have a great day!</p>
        <div className="py-10">
          <Link to="/orders" className="px-12 bg-indigo-600 hover:bg-indigo-500 text-white font-semibold py-3 rounded-lg">
            Go to Orders
          </Link>
        </div>
      </div>
    </div>
  )
}

export default CheckoutSuccess
```

**8. Tailwind Cancel Page ( optional )**
```
import React from 'react'
import { Link } from 'react-router-dom'

function CheckoutCancel() {
  return (
    <div className="h-screen flex justify-center items-center bg-gray-100">
      <div className="bg-white p-6 rounded-lg shadow-md text-center max-w-md">
        <svg viewBox="0 0 24 24" className="text-red-600 w-16 h-16 mx-auto my-6">
          <path fill="currentColor"
            d="M12 2a10 10 0 1 0 10 10A10 10 0 0 0 12 2Zm1 14.93V17a1 1 0 1 1-2 0v-1.07a6.84 6.84 0 0 1-4.6-4.6H7a1 1 0 1 1 0-2H6.4A6.84 6.84 0 0 1 11 6.07V5a1 1 0 1 1 2 0v1.07a6.84 6.84 0 0 1 4.6 4.6H17a1 1 0 1 1 0 2h.6A6.84 6.84 0 0 1 13 15.93Z">
          </path>
        </svg>
        <h3 className="md:text-2xl text-base text-gray-900 font-semibold">Payment Cancelled</h3>
        <p className="text-gray-600 my-2">Your payment has been cancelled. You can continue shopping or try again later.</p>
        <div className="py-10">
          <Link to="/" className="px-12 bg-red-600 hover:bg-red-500 text-white font-semibold py-3 rounded-lg">
            Continue Shopping
          </Link>
        </div>
      </div>
    </div>
  )
}

export default CheckoutCancel
```

**9. Conclusion:**
- This backend example demonstrates how to:
- Create a Stripe checkout session with sample cart items (like a bag and a shirt).
- Handle price calculations based on cart items.
- Use Stripe webhooks to handle completed payments and save orders to the database.

