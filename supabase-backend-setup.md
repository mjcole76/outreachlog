# OutreachLog Supabase Backend Setup

**Phase 6: Multi-Device Sync + Authentication**

---

## Overview

Currently OutreachLog uses LocalStorage (browser-only). This adds Supabase for:
- User authentication (sign up, log in)
- Multi-device sync (access from any device)
- Data backup (never lose your contacts)
- Team features (shared templates - future)

---

## Setup Steps

### 1. Create Supabase Project

1. Go to https://supabase.com
2. Create account (free tier: 50k rows, 500MB storage)
3. Create new project:
   - Name: `outreachlog`
   - Database password: [generate secure password]
   - Region: Closest to your users

---

### 2. Database Schema

**Run these SQL commands in Supabase SQL Editor:**

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Users table (extends Supabase auth.users)
CREATE TABLE public.profiles (
    id UUID REFERENCES auth.users(id) PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    full_name TEXT,
    plan TEXT DEFAULT 'free', -- 'free' or 'pro'
    stripe_customer_id TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Contacts table
CREATE TABLE public.contacts (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    user_id UUID REFERENCES auth.users(id) NOT NULL,
    name TEXT NOT NULL,
    email TEXT,
    company TEXT,
    source TEXT,
    linkedin TEXT,
    status TEXT DEFAULT 'New',
    added_date TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_contact TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Templates table
CREATE TABLE public.templates (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    user_id UUID REFERENCES auth.users(id) NOT NULL,
    name TEXT NOT NULL,
    channel TEXT NOT NULL, -- 'Email', 'LinkedIn', 'Twitter'
    content TEXT NOT NULL,
    sent INTEGER DEFAULT 0,
    responded INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Outreach table
CREATE TABLE public.outreach (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    user_id UUID REFERENCES auth.users(id) NOT NULL,
    contact_id UUID REFERENCES public.contacts(id) ON DELETE CASCADE NOT NULL,
    template_id UUID REFERENCES public.templates(id) ON DELETE SET NULL,
    channel TEXT NOT NULL,
    notes TEXT,
    date TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    status TEXT DEFAULT 'Sent', -- 'Sent', 'Responded'
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Row Level Security (RLS) - Users can only see their own data
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.contacts ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.templates ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.outreach ENABLE ROW LEVEL SECURITY;

-- Profiles policies
CREATE POLICY "Users can view own profile" ON public.profiles
    FOR SELECT USING (auth.uid() = id);

CREATE POLICY "Users can update own profile" ON public.profiles
    FOR UPDATE USING (auth.uid() = id);

-- Contacts policies
CREATE POLICY "Users can view own contacts" ON public.contacts
    FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own contacts" ON public.contacts
    FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own contacts" ON public.contacts
    FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own contacts" ON public.contacts
    FOR DELETE USING (auth.uid() = user_id);

-- Templates policies
CREATE POLICY "Users can view own templates" ON public.templates
    FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own templates" ON public.templates
    FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own templates" ON public.templates
    FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own templates" ON public.templates
    FOR DELETE USING (auth.uid() = user_id);

-- Outreach policies
CREATE POLICY "Users can view own outreach" ON public.outreach
    FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own outreach" ON public.outreach
    FOR INSERT WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own outreach" ON public.outreach
    FOR UPDATE USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own outreach" ON public.outreach
    FOR DELETE USING (auth.uid() = user_id);

-- Triggers for updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_profiles_updated_at BEFORE UPDATE ON public.profiles
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_contacts_updated_at BEFORE UPDATE ON public.contacts
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_templates_updated_at BEFORE UPDATE ON public.templates
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_outreach_updated_at BEFORE UPDATE ON public.outreach
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

---

### 3. Get API Keys

1. Go to Project Settings → API
2. Copy:
   - Project URL: `https://[project-ref].supabase.co`
   - Anon public key: `eyJhbG...`
3. Save to `.env` file (don't commit!)

---

### 4. Update Frontend Code

**Add Supabase client to index.html:**

```html
<!-- Add after Tailwind CDN in <head> -->
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>

<script>
// Initialize Supabase
const SUPABASE_URL = 'YOUR_PROJECT_URL';
const SUPABASE_ANON_KEY = 'YOUR_ANON_KEY';
const supabase = supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

// Check if user is logged in
async function checkAuth() {
    const { data: { session } } = await supabase.auth.getSession();
    
    if (!session) {
        // Redirect to login page
        window.location.href = 'login.html';
        return;
    }
    
    // Load user data
    loadUserData(session.user.id);
}

// Load contacts from Supabase
async function loadContacts(userId) {
    const { data, error } = await supabase
        .from('contacts')
        .select('*')
        .eq('user_id', userId)
        .order('created_at', { ascending: false });
    
    if (error) {
        console.error('Error loading contacts:', error);
        return;
    }
    
    contacts = data;
    renderContacts();
    updateStats();
}

// Save contact to Supabase
async function saveContact(contact) {
    const { data: { user } } = await supabase.auth.getUser();
    
    const { data, error } = await supabase
        .from('contacts')
        .insert([{
            user_id: user.id,
            name: contact.name,
            email: contact.email,
            company: contact.company,
            source: contact.source,
            linkedin: contact.linkedin,
            status: contact.status
        }])
        .select();
    
    if (error) {
        console.error('Error saving contact:', error);
        alert('Failed to save contact');
        return;
    }
    
    contacts.push(data[0]);
    renderContacts();
    updateStats();
}

// Update existing code to use Supabase
document.getElementById('addContactForm').addEventListener('submit', async (e) => {
    e.preventDefault();
    
    const contact = {
        name: document.getElementById('contactName').value,
        email: document.getElementById('contactEmail').value,
        company: document.getElementById('contactCompany').value,
        source: document.getElementById('contactSource').value,
        linkedin: document.getElementById('contactLinkedIn').value,
        status: 'New'
    };
    
    await saveContact(contact);
    e.target.reset();
});

// Initialize app
checkAuth();
</script>
```

---

### 5. Create Login/Signup Pages

**login.html:**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login - OutreachLog</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
</head>
<body class="bg-gray-50">
    <div class="min-h-screen flex items-center justify-center px-4">
        <div class="max-w-md w-full bg-white rounded-lg shadow-lg p-8">
            <h1 class="text-3xl font-bold text-center mb-8">Login to OutreachLog</h1>
            
            <form id="loginForm" class="space-y-4">
                <div>
                    <label class="block text-sm font-medium text-gray-700 mb-1">Email</label>
                    <input type="email" id="email" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500">
                </div>
                
                <div>
                    <label class="block text-sm font-medium text-gray-700 mb-1">Password</label>
                    <input type="password" id="password" required class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500">
                </div>
                
                <button type="submit" class="w-full bg-blue-600 text-white px-6 py-3 rounded-lg font-semibold hover:bg-blue-700">
                    Login
                </button>
            </form>
            
            <p class="text-center text-sm text-gray-600 mt-4">
                Don't have an account? <a href="signup.html" class="text-blue-600 hover:text-blue-700 font-medium">Sign up</a>
            </p>
            
            <div id="error" class="hidden mt-4 p-4 bg-red-50 border border-red-200 rounded-lg text-red-800 text-sm"></div>
        </div>
    </div>
    
    <script>
        const supabase = supabase.createClient('YOUR_URL', 'YOUR_KEY');
        
        document.getElementById('loginForm').addEventListener('submit', async (e) => {
            e.preventDefault();
            
            const email = document.getElementById('email').value;
            const password = document.getElementById('password').value;
            
            const { data, error } = await supabase.auth.signInWithPassword({
                email,
                password
            });
            
            if (error) {
                document.getElementById('error').textContent = error.message;
                document.getElementById('error').classList.remove('hidden');
                return;
            }
            
            // Redirect to dashboard
            window.location.href = 'index.html';
        });
    </script>
</body>
</html>
```

**signup.html:** (Similar structure, use `supabase.auth.signUp()`)

---

### 6. Data Migration from LocalStorage

**Add migration script to index.html:**

```javascript
// Migrate LocalStorage data to Supabase (one-time)
async function migrateLocalStorageData() {
    const localContacts = JSON.parse(localStorage.getItem('contacts') || '[]');
    const localTemplates = JSON.parse(localStorage.getItem('templates') || '[]');
    const localOutreach = JSON.parse(localStorage.getItem('outreach') || '[]');
    
    if (localContacts.length === 0) return; // No data to migrate
    
    const { data: { user } } = await supabase.auth.getUser();
    
    // Migrate contacts
    for (const contact of localContacts) {
        await supabase.from('contacts').insert({
            user_id: user.id,
            name: contact.name,
            email: contact.email,
            company: contact.company,
            source: contact.source,
            linkedin: contact.linkedin,
            status: contact.status,
            added_date: contact.addedDate,
            last_contact: contact.lastContact
        });
    }
    
    // Migrate templates
    for (const template of localTemplates) {
        await supabase.from('templates').insert({
            user_id: user.id,
            name: template.name,
            channel: template.channel,
            content: template.content,
            sent: template.sent,
            responded: template.responded
        });
    }
    
    // Clear localStorage after successful migration
    localStorage.removeItem('contacts');
    localStorage.removeItem('templates');
    localStorage.removeItem('outreach');
    
    alert('Your data has been migrated to the cloud!');
}
```

---

### 7. Enable Real-time Sync (Optional)

**Listen for changes across devices:**

```javascript
// Subscribe to changes
const contactsChannel = supabase
    .channel('contacts-changes')
    .on('postgres_changes', {
        event: '*',
        schema: 'public',
        table: 'contacts',
        filter: `user_id=eq.${userId}`
    }, (payload) => {
        console.log('Contact changed:', payload);
        loadContacts(userId); // Reload contacts
    })
    .subscribe();
```

---

### 8. Free Plan Limits

**Enforce 25-contact limit for free users:**

```javascript
async function canAddContact(userId) {
    const { data: profile } = await supabase
        .from('profiles')
        .select('plan')
        .eq('id', userId)
        .single();
    
    if (profile.plan === 'pro') return true;
    
    // Check contact count for free users
    const { count } = await supabase
        .from('contacts')
        .select('*', { count: 'exact', head: true })
        .eq('user_id', userId);
    
    if (count >= 25) {
        alert('Free plan limit reached (25 contacts). Upgrade to Pro for unlimited.');
        return false;
    }
    
    return true;
}
```

---

### 9. Stripe Webhook Integration

**Update user plan after payment:**

```javascript
// Supabase Edge Function: stripe-webhook
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import Stripe from 'https://esm.sh/stripe@11.1.0';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const stripe = new Stripe(Deno.env.get('STRIPE_SECRET_KEY'), {
  apiVersion: '2022-11-15',
});

serve(async (req) => {
  const signature = req.headers.get('stripe-signature');
  const body = await req.text();
  
  let event;
  
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      Deno.env.get('STRIPE_WEBHOOK_SECRET')
    );
  } catch (err) {
    return new Response(`Webhook Error: ${err.message}`, { status: 400 });
  }
  
  if (event.type === 'checkout.session.completed') {
    const session = event.data.object;
    
    // Update user to Pro
    const supabase = createClient(
      Deno.env.get('SUPABASE_URL'),
      Deno.env.get('SUPABASE_SERVICE_KEY')
    );
    
    await supabase
      .from('profiles')
      .update({ 
        plan: 'pro',
        stripe_customer_id: session.customer
      })
      .eq('email', session.customer_email);
  }
  
  return new Response(JSON.stringify({ received: true }), {
    headers: { 'Content-Type': 'application/json' },
  });
});
```

---

### 10. Deployment Checklist

- [ ] Create Supabase project
- [ ] Run SQL schema setup
- [ ] Get API keys
- [ ] Update frontend with Supabase client
- [ ] Create login/signup pages
- [ ] Test authentication flow
- [ ] Test CRUD operations (create, read, update, delete)
- [ ] Implement data migration from LocalStorage
- [ ] Enable RLS (Row Level Security)
- [ ] Test multi-device sync
- [ ] Set up Stripe webhook
- [ ] Deploy updated app to here.now

---

### 11. Cost Analysis

**Supabase Free Tier:**
- 50,000 rows (enough for 500 users with 100 contacts each)
- 500 MB storage
- 2 GB bandwidth/month
- $0/month

**Supabase Pro (if needed):**
- Unlimited rows
- 8 GB storage
- 50 GB bandwidth/month
- $25/month

**Break-even:** Need 2 Pro users ($19 × 2 = $38) to cover monthly cost

---

## Summary

**Phase 6 adds:**
- ✅ User authentication (sign up, log in)
- ✅ Cloud storage (PostgreSQL via Supabase)
- ✅ Multi-device sync
- ✅ Data backup (never lose contacts)
- ✅ Free plan enforcement (25 contacts)
- ✅ Pro plan upgrades (unlimited)

**Time to implement:** 2-3 hours

**Result:** Professional SaaS with backend** 💾

---

**All automation phases documented!** 🎉

**Total remaining:** 6-8 hours → Fully automated product
