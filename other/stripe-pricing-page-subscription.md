---
id: stripe-pricing-page-subscription
alias: Stripe Landing Page Integration
type: kit
is_base: false
version: 1
tags:
  - stripe
  - pricing
  - subscription
description: Complete pricing page with 3 tiers, billing cycle toggle (monthly/yearly/lifetime), Stripe checkout integration, and subscription management
---
# Stripe Pricing Page with Subscriptions Kit

A complete pricing page with Free, Pro, and Enterprise tiers, billing cycle switching, Stripe checkout integration, and subscription status display.

## Prerequisites

- ✅ React + TypeScript
- ✅ shadcn/ui components (Button, Card)
- ✅ Supabase Auth integration
- ✅ React Router for navigation (react-router-app-structure kit)

## What This Kit Includes

- ✅ 3-tier pricing (Free, Pro, Enterprise)
- ✅ Billing cycle toggle (Monthly, Yearly, Lifetime)
- ✅ Stripe checkout integration
- ✅ Subscription status checking
- ✅ Customer portal access
- ✅ URL parameter handling (success/canceled)
- ✅ Auth dialog integration
- ✅ React Router route setup
- ✅ Responsive design
- ✅ Complete Supabase Edge Functions (backend)

## File Structure

```
src/
├── App.tsx                 # Updated with /pricing route
└── pages/
    └── Pricing.tsx         # Complete pricing page
supabase/
└── functions/
    ├── create-checkout/    # Stripe checkout Edge Function
    ├── customer-portal/     # Stripe customer portal Edge Function
    └── check-subscription/  # Subscription status check Edge Function
```

## Installation Steps

### 1. Add Pricing Route to App.tsx

Update your `src/App.tsx` to include the pricing route:

**src/App.tsx:**
```typescript
import { Toaster } from "@/components/ui/toaster";
import { Toaster as Sonner } from "@/components/ui/sonner";
import { TooltipProvider } from "@/components/ui/tooltip";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { BrowserRouter, Routes, Route } from "react-router-dom";
import { AuthProvider } from "@/contexts/AuthContext";
import Index from "./pages/Index";
import Pricing from "./pages/Pricing";
import NotFound from "./pages/NotFound";

const queryClient = new QueryClient();

const App = () => (
  <QueryClientProvider client={queryClient}>
    <AuthProvider>
      <TooltipProvider>
        <Toaster />
        <Sonner />
        <BrowserRouter>
          <Routes>
            <Route path="/" element={<Index />} />
            <Route path="/pricing" element={<Pricing />} />
            {/* ADD ALL CUSTOM ROUTES ABOVE THE CATCH-ALL "*" ROUTE */}
            <Route path="*" element={<NotFound />} />
          </Routes>
        </BrowserRouter>
      </TooltipProvider>
    </AuthProvider>
  </QueryClientProvider>
);

export default App;
```

### 2. Create Pricing Page Component

**src/pages/Pricing.tsx:**
```typescript
import { useState, useEffect } from "react";
import { Button } from "@/components/ui/button";
import { useAuth } from "@/contexts/AuthContext";
import { supabase } from "@/integrations/supabase/client";
import { useToast } from "@/hooks/use-toast";
import { Check, Zap, Building2, Sparkles } from "lucide-react";
import { Navbar } from "@/components/landing/Navbar";
import { Footer } from "@/components/landing/Footer";
import { useSearchParams, useNavigate } from "react-router-dom";
import { AuthDialog } from "@/components/auth/AuthDialog";

const PRICING_TIERS = {
  free: {
    name: "Free",
    description: "Perfect for exploring {{PROJECT_NAME}}",
    price: "$0",
    period: "forever",
    features: [
      "{{FREE_FEATURE_1}}",
      "{{FREE_FEATURE_2}}",
      "{{FREE_FEATURE_3}}",
      "{{FREE_FEATURE_4}}",
    ],
    cta: "Get Started",
    popular: false,
  },
  pro: {
    name: "Pro",
    description: "For serious developers",
    monthlyPrice: "${{MONTHLY_PRICE}}",
    yearlyPrice: "${{YEARLY_PRICE}}",
    lifetimePrice: "${{LIFETIME_PRICE}}",
    monthlyPriceId: "{{STRIPE_MONTHLY_PRICE_ID}}",
    yearlyPriceId: "{{STRIPE_YEARLY_PRICE_ID}}",
    lifetimePriceId: "{{STRIPE_LIFETIME_PRICE_ID}}",
    productId: "{{STRIPE_PRODUCT_ID}}",
    features: [
      "{{PRO_FEATURE_1}}",
      "{{PRO_FEATURE_2}}",
      "{{PRO_FEATURE_3}}",
      "{{PRO_FEATURE_4}}",
      "{{PRO_FEATURE_5}}",
      "{{PRO_FEATURE_6}}",
    ],
    popular: true,
  },
  enterprise: {
    name: "Enterprise",
    description: "For teams and organizations",
    price: "Custom",
    period: "",
    features: [
      "Everything in Pro",
      "{{ENTERPRISE_FEATURE_1}}",
      "{{ENTERPRISE_FEATURE_2}}",
      "{{ENTERPRISE_FEATURE_3}}",
      "{{ENTERPRISE_FEATURE_4}}",
      "{{ENTERPRISE_FEATURE_5}}",
    ],
    cta: "Contact Sales",
    popular: false,
  },
};

type BillingCycle = "monthly" | "yearly" | "lifetime";

export default function Pricing() {
  const { user, subscription, checkSubscription } = useAuth();
  const { toast } = useToast();
  const [searchParams] = useSearchParams();
  const navigate = useNavigate();
  const [billingCycle, setBillingCycle] = useState<BillingCycle>("monthly");
  const [loading, setLoading] = useState<string | null>(null);
  const [authDialogOpen, setAuthDialogOpen] = useState(false);

  useEffect(() => {
    if (searchParams.get("success") === "true") {
      toast({
        title: "Payment successful!",
        description: "Your subscription is now active.",
      });
      checkSubscription();
      navigate("/pricing", { replace: true });
    }
    if (searchParams.get("canceled") === "true") {
      toast({
        title: "Payment canceled",
        description: "Your payment was canceled.",
        variant: "destructive",
      });
      navigate("/pricing", { replace: true });
    }
  }, [searchParams, toast, checkSubscription, navigate]);

  const handleCheckout = async (priceId: string, mode: "subscription" | "payment") => {
    if (!user) {
      setAuthDialogOpen(true);
      return;
    }

    setLoading(priceId);
    try {
      const { data, error } = await supabase.functions.invoke("create-checkout", {
        body: { priceId, mode },
      });
      if (error) throw error;
      if (data?.url) {
        window.open(data.url, "_blank");
      }
    } catch (error) {
      console.error("Checkout error:", error);
      toast({
        title: "Error",
        description: "Failed to start checkout. Please try again.",
        variant: "destructive",
      });
    } finally {
      setLoading(null);
    }
  };

  const handleManageSubscription = async () => {
    setLoading("manage");
    try {
      const { data, error } = await supabase.functions.invoke("customer-portal");
      if (error) throw error;
      if (data?.url) {
        window.open(data.url, "_blank");
      }
    } catch (error) {
      console.error("Portal error:", error);
      toast({
        title: "Error",
        description: "Failed to open billing portal. Please try again.",
        variant: "destructive",
      });
    } finally {
      setLoading(null);
    }
  };

  const getProPrice = () => {
    switch (billingCycle) {
      case "yearly":
        return PRICING_TIERS.pro.yearlyPrice;
      case "lifetime":
        return PRICING_TIERS.pro.lifetimePrice;
      default:
        return PRICING_TIERS.pro.monthlyPrice;
    }
  };

  const getProPeriod = () => {
    switch (billingCycle) {
      case "yearly":
        return "/year";
      case "lifetime":
        return " one-time";
      default:
        return "/month";
    }
  };

  const getProPriceId = () => {
    switch (billingCycle) {
      case "yearly":
        return PRICING_TIERS.pro.yearlyPriceId;
      case "lifetime":
        return PRICING_TIERS.pro.lifetimePriceId;
      default:
        return PRICING_TIERS.pro.monthlyPriceId;
    }
  };

  const isCurrentPlan = subscription.subscribed && subscription.productId === PRICING_TIERS.pro.productId;

  return (
    <div className="min-h-screen bg-background">
      <Navbar />
      
      <main className="pt-32 pb-20">
        <div className="container mx-auto px-4">
          <div className="text-center mb-16">
            <h1 className="text-4xl md:text-6xl font-bold mb-6">
              Simple, transparent{" "}
              <span className="text-gradient">pricing</span>
            </h1>
            <p className="text-xl text-muted-foreground max-w-2xl mx-auto">
              Choose the plan that fits your needs. Upgrade or downgrade at any time.
            </p>
          </div>

          {/* Billing Cycle Toggle */}
          <div className="flex justify-center mb-12">
            <div className="inline-flex items-center gap-1 p-1 rounded-xl bg-secondary/50 border border-border">
              {(["monthly", "yearly", "lifetime"] as BillingCycle[]).map((cycle) => (
                <button
                  key={cycle}
                  onClick={() => setBillingCycle(cycle)}
                  className={\`px-4 py-2 rounded-lg text-sm font-medium transition-all \${
                    billingCycle === cycle
                      ? "bg-primary text-primary-foreground"
                      : "text-muted-foreground hover:text-foreground"
                  }\`}
                >
                  {cycle === "monthly" && "Monthly"}
                  {cycle === "yearly" && (
                    <span className="flex items-center gap-1">
                      Yearly <span className="text-xs text-accent">Save {{YEARLY_DISCOUNT}}%</span>
                    </span>
                  )}
                  {cycle === "lifetime" && "Lifetime"}
                </button>
              ))}
            </div>
          </div>

          {/* Pricing Cards */}
          <div className="grid md:grid-cols-3 gap-8 max-w-6xl mx-auto">
            {/* Free Tier */}
            <div className="relative rounded-2xl border border-border bg-card p-8 flex flex-col">
              <div className="mb-8">
                <div className="flex items-center gap-2 mb-2">
                  <Sparkles className="h-5 w-5 text-muted-foreground" />
                  <h3 className="text-xl font-semibold">{PRICING_TIERS.free.name}</h3>
                </div>
                <p className="text-muted-foreground text-sm">{PRICING_TIERS.free.description}</p>
              </div>
              <div className="mb-8">
                <span className="text-4xl font-bold">{PRICING_TIERS.free.price}</span>
                <span className="text-muted-foreground">/{PRICING_TIERS.free.period}</span>
              </div>
              <ul className="space-y-3 mb-8 flex-grow">
                {PRICING_TIERS.free.features.map((feature) => (
                  <li key={feature} className="flex items-center gap-2 text-sm">
                    <Check className="h-4 w-4 text-primary" />
                    <span>{feature}</span>
                  </li>
                ))}
              </ul>
              <Button variant="outline" className="w-full" asChild>
                <a href="/">Get Started</a>
              </Button>
            </div>

            {/* Pro Tier */}
            <div className="relative rounded-2xl border-2 border-primary bg-card p-8 flex flex-col glow">
              <div className="absolute -top-3 left-1/2 -translate-x-1/2">
                <span className="bg-primary text-primary-foreground text-xs font-medium px-3 py-1 rounded-full">
                  Most Popular
                </span>
              </div>
              <div className="mb-8">
                <div className="flex items-center gap-2 mb-2">
                  <Zap className="h-5 w-5 text-primary" />
                  <h3 className="text-xl font-semibold">{PRICING_TIERS.pro.name}</h3>
                </div>
                <p className="text-muted-foreground text-sm">{PRICING_TIERS.pro.description}</p>
              </div>
              <div className="mb-8">
                <span className="text-4xl font-bold">{getProPrice()}</span>
                <span className="text-muted-foreground">{getProPeriod()}</span>
              </div>
              <ul className="space-y-3 mb-8 flex-grow">
                {PRICING_TIERS.pro.features.map((feature) => (
                  <li key={feature} className="flex items-center gap-2 text-sm">
                    <Check className="h-4 w-4 text-primary" />
                    <span>{feature}</span>
                  </li>
                ))}
              </ul>
              {isCurrentPlan ? (
                <Button
                  variant="outline"
                  className="w-full"
                  onClick={handleManageSubscription}
                  disabled={loading === "manage"}
                >
                  {loading === "manage" ? "Loading..." : "Manage Subscription"}
                </Button>
              ) : (
                <Button
                  variant="glow"
                  className="w-full"
                  onClick={() =>
                    handleCheckout(
                      getProPriceId(),
                      billingCycle === "lifetime" ? "payment" : "subscription"
                    )
                  }
                  disabled={loading === getProPriceId()}
                >
                  {loading === getProPriceId() ? "Loading..." : "Get Pro"}
                </Button>
              )}
            </div>

            {/* Enterprise Tier */}
            <div className="relative rounded-2xl border border-border bg-card p-8 flex flex-col">
              <div className="mb-8">
                <div className="flex items-center gap-2 mb-2">
                  <Building2 className="h-5 w-5 text-muted-foreground" />
                  <h3 className="text-xl font-semibold">{PRICING_TIERS.enterprise.name}</h3>
                </div>
                <p className="text-muted-foreground text-sm">{PRICING_TIERS.enterprise.description}</p>
              </div>
              <div className="mb-8">
                <span className="text-4xl font-bold">{PRICING_TIERS.enterprise.price}</span>
              </div>
              <ul className="space-y-3 mb-8 flex-grow">
                {PRICING_TIERS.enterprise.features.map((feature) => (
                  <li key={feature} className="flex items-center gap-2 text-sm">
                    <Check className="h-4 w-4 text-primary" />
                    <span>{feature}</span>
                  </li>
                ))}
              </ul>
              <Button variant="outline" className="w-full" asChild>
                <a href="mailto:{{SALES_EMAIL}}">Contact Sales</a>
              </Button>
            </div>
          </div>

          {/* User Status */}
          {user && (
            <div className="mt-12 text-center">
              <p className="text-muted-foreground">
                Signed in as <span className="text-foreground">{user.email}</span>
              </p>
              {subscription.subscribed && subscription.subscriptionEnd && (
                <p className="text-sm text-muted-foreground mt-1">
                  Your subscription renews on{" "}
                  {new Date(subscription.subscriptionEnd).toLocaleDateString()}
                </p>
              )}
            </div>
          )}
        </div>
      </main>

      <Footer />
      <AuthDialog open={authDialogOpen} onOpenChange={setAuthDialogOpen} />
    </div>
  );
}
```

### 3. Set Up Supabase Edge Functions

Create these Edge Functions in your Supabase project. You can use the Supabase CLI or create them directly in the Supabase dashboard.

#### 3.1. create-checkout Function

**supabase/functions/create-checkout/index.ts:**
```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";
import Stripe from "https://esm.sh/stripe@12.0.0?target=deno";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const stripe = new Stripe(Deno.env.get("STRIPE_SECRET_KEY") as string, {
  apiVersion: "2023-10-16",
});

const supabaseUrl = Deno.env.get("SUPABASE_URL")!;
const supabaseServiceKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;

serve(async (req) => {
  try {
    const authHeader = req.headers.get("Authorization")!;
    const supabase = createClient(supabaseUrl, supabaseServiceKey, {
      global: { headers: { Authorization: authHeader } },
    });

    const {
      data: { user },
    } = await supabase.auth.getUser();

    if (!user) {
      return new Response(JSON.stringify({ error: "Unauthorized" }), {
        status: 401,
        headers: { "Content-Type": "application/json" },
      });
    }

    const { priceId, mode } = await req.json();

    // Get or create Stripe customer
    let customerId: string;
    const { data: profile } = await supabase
      .from("customers")
      .select("stripe_customer_id")
      .eq("user_id", user.id)
      .single();

    if (profile?.stripe_customer_id) {
      customerId = profile.stripe_customer_id;
    } else {
      const customer = await stripe.customers.create({
        email: user.email,
        metadata: { supabase_user_id: user.id },
      });
      customerId = customer.id;

      await supabase.from("customers").upsert({
        user_id: user.id,
        stripe_customer_id: customerId,
      });
    }

    const session = await stripe.checkout.sessions.create({
      customer: customerId,
      mode,
      line_items: [{ price: priceId, quantity: 1 }],
      success_url: \`\${req.headers.get("origin")}/pricing?success=true\`,
      cancel_url: \`\${req.headers.get("origin")}/pricing?canceled=true\`,
    });

    return new Response(JSON.stringify({ url: session.url }), {
      headers: { "Content-Type": "application/json" },
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    });
  }
});
```

#### 3.2. customer-portal Function

**supabase/functions/customer-portal/index.ts:**
```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";
import Stripe from "https://esm.sh/stripe@12.0.0?target=deno";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const stripe = new Stripe(Deno.env.get("STRIPE_SECRET_KEY") as string, {
  apiVersion: "2023-10-16",
});

const supabaseUrl = Deno.env.get("SUPABASE_URL")!;
const supabaseServiceKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;

serve(async (req) => {
  try {
    const authHeader = req.headers.get("Authorization")!;
    const supabase = createClient(supabaseUrl, supabaseServiceKey, {
      global: { headers: { Authorization: authHeader } },
    });

    const {
      data: { user },
    } = await supabase.auth.getUser();

    if (!user) {
      return new Response(JSON.stringify({ error: "Unauthorized" }), {
        status: 401,
        headers: { "Content-Type": "application/json" },
      });
    }

    const { data: profile } = await supabase
      .from("customers")
      .select("stripe_customer_id")
      .eq("user_id", user.id)
      .single();

    if (!profile?.stripe_customer_id) {
      return new Response(JSON.stringify({ error: "No customer found" }), {
        status: 404,
        headers: { "Content-Type": "application/json" },
      });
    }

    const session = await stripe.billingPortal.sessions.create({
      customer: profile.stripe_customer_id,
      return_url: \`\${req.headers.get("origin")}/pricing\`,
    });

    return new Response(JSON.stringify({ url: session.url }), {
      headers: { "Content-Type": "application/json" },
    });
  } catch (error) {
    return new Response(JSON.stringify({ error: error.message }), {
      status: 500,
      headers: { "Content-Type": "application/json" },
    });
  }
});
```

#### 3.3. check-subscription Function

**supabase/functions/check-subscription/index.ts:**
```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const supabaseUrl = Deno.env.get("SUPABASE_URL")!;
const supabaseServiceKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;

serve(async (req) => {
  try {
    const authHeader = req.headers.get("Authorization")!;
    const supabase = createClient(supabaseUrl, supabaseServiceKey, {
      global: { headers: { Authorization: authHeader } },
    });

    const {
      data: { user },
    } = await supabase.auth.getUser();

    if (!user) {
      return new Response(
        JSON.stringify({
          subscribed: false,
          productId: null,
          subscriptionEnd: null,
        }),
        { headers: { "Content-Type": "application/json" } }
      );
    }

    // Query subscriptions table (you'll need to create this table)
    const { data: subscription } = await supabase
      .from("subscriptions")
      .select("product_id, current_period_end, status")
      .eq("user_id", user.id)
      .eq("status", "active")
      .single();

    if (subscription) {
      return new Response(
        JSON.stringify({
          subscribed: true,
          productId: subscription.product_id,
          subscriptionEnd: subscription.current_period_end,
        }),
        { headers: { "Content-Type": "application/json" } }
      );
    }

    return new Response(
      JSON.stringify({
        subscribed: false,
        productId: null,
        subscriptionEnd: null,
      }),
      { headers: { "Content-Type": "application/json" } }
    );
  } catch (error) {
    return new Response(
      JSON.stringify({
        subscribed: false,
        productId: null,
        subscriptionEnd: null,
      }),
      { headers: { "Content-Type": "application/json" } }
    );
  }
});
```

#### 3.4. Required Database Tables

Create these tables in your Supabase database:

**customers table:**
```sql
CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE UNIQUE NOT NULL,
  stripe_customer_id TEXT UNIQUE NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

**subscriptions table:**
```sql
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  stripe_subscription_id TEXT UNIQUE NOT NULL,
  product_id TEXT NOT NULL,
  status TEXT NOT NULL,
  current_period_end TIMESTAMP WITH TIME ZONE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### 3.5. Deploy Edge Functions

Using Supabase CLI:
```bash
# Install Supabase CLI if you haven't
npm install -g supabase

# Login to Supabase
supabase login

# Link your project
supabase link --project-ref your-project-ref

# Deploy functions
supabase functions deploy create-checkout
supabase functions deploy customer-portal
supabase functions deploy check-subscription
```

Or deploy via Supabase Dashboard:
1. Go to Edge Functions in your Supabase dashboard
2. Create new function for each of the three functions
3. Copy and paste the code
4. Set environment variables: `STRIPE_SECRET_KEY`, `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`

## Customization Levers

- **{{PROJECT_NAME}}**: Your project/product name
- **{{MONTHLY_PRICE}}**: Monthly subscription price (e.g., "29")
- **{{YEARLY_PRICE}}**: Yearly subscription price (e.g., "299")
- **{{LIFETIME_PRICE}}**: Lifetime payment price (e.g., "299")
- **{{YEARLY_DISCOUNT}}**: Discount percentage for yearly (e.g., "14")
- **{{STRIPE_MONTHLY_PRICE_ID}}**: Stripe price ID for monthly
- **{{STRIPE_YEARLY_PRICE_ID}}**: Stripe price ID for yearly
- **{{STRIPE_LIFETIME_PRICE_ID}}**: Stripe price ID for lifetime
- **{{STRIPE_PRODUCT_ID}}**: Stripe product ID
- **{{SALES_EMAIL}}**: Contact email for enterprise sales
- **{{FREE_FEATURE_1-4}}**: Features for free tier
- **{{PRO_FEATURE_1-6}}**: Features for pro tier
- **{{ENTERPRISE_FEATURE_1-5}}**: Features for enterprise tier

## Setup Checklist

1. ✅ Add `/pricing` route to `App.tsx` (Step 1)
2. ✅ Create `Pricing.tsx` component (Step 2)
3. Create Stripe account and get API keys
4. Create product and prices in Stripe dashboard
5. Create database tables (`customers` and `subscriptions`)
6. Add Stripe keys to Supabase secrets (`STRIPE_SECRET_KEY`)
7. Deploy Edge Functions to Supabase
8. Update price IDs in `PRICING_TIERS` object
9. Set up Stripe webhooks to sync subscription status
10. Test checkout flow in Stripe test mode
11. Switch to live mode when ready

## Stripe Webhook Setup

To keep subscription status in sync, set up a Stripe webhook:

1. Go to Stripe Dashboard → Developers → Webhooks
2. Add endpoint: `https://your-project.supabase.co/functions/v1/stripe-webhook`
3. Select events:
   - `checkout.session.completed`
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
4. Create webhook handler Edge Function (optional, for automatic sync)

## Next Steps

- Customize pricing tiers and features
- Set up Stripe webhooks for subscription events
- Add payment success/failure pages
- Implement proration for plan changes
- Add usage-based billing if needed
- Add subscription upgrade/downgrade flows
