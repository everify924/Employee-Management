# Joypurhat EMS — Supabase লগইন + পুরো ক্লাউড ব্যাকআপ

এই সেটআপে:
- **লগইন/ইউজার** → Supabase Auth + `profiles` টেবিল (সব ডিভাইসে একই)
- **কর্মচারী, পদ, অরগানোগ্রাম, বদলী, ACR** → `ems_kv` টেবিলে JSON (ক্লাউড সিঙ্ক)
- **ব্যাকআপ ইতিহাস** → `ems_backups` টেবিল
- কনফিগ না দিলে পুরনো **localStorage** মোডই চলে

---

## ধাপ ১ — Supabase প্রজেক্ট

1. [supabase.com](https://supabase.com) → **New project**
2. নাম: `jph-ems`, Region: **Singapore** (বা কাছের)
3. Database password সেভ করুন
4. **Project Settings → API**:
   - **Project URL** কপি
   - **anon public** key কপি

---

## ধাপ ২ — SQL স্কিমা

Dashboard → **SQL Editor** → New query →  
রিপোর `supabase-schema.sql` ফাইলের **পুরো SQL** পেস্ট করে **Run**।

টেবিল তৈরি হবে:
| টেবিল | কাজ |
|--------|------|
| `profiles` | ইউজারনেম, রোল, অনুমোদন |
| `ems_kv` | অ্যাপের সব ডেটা (key → JSON) |
| `ems_backups` | ক্লাউড ব্যাকআপ ইতিহাস |

---

## ধাপ ৩ — Auth সেটিংস

**Authentication → Providers → Email**  
- Enable Email  
- **Confirm email → OFF** (অফিস অ্যাপ হলে সুবিধা)

**Authentication → URL Configuration**  
- **Site URL** = আপনার GitHub Pages URL  
  উদাহরণ: `https://USERNAME.github.io/REPO/`

---

## ধাপ ৪ — প্রথম অ্যাডমিন

### Dashboard থেকে

1. **Authentication → Users → Add user**
2. Email: `aamin924@jph-ems.local`  
   (অথবা আসল ইমেইল — তাহলে অ্যাপে সেই ইমেইল দিয়েই লগইন)
3. Password: শক্তিশালী পাসওয়ার্ড
4. **Auto Confirm User**: Yes

### প্রোফাইল অ্যাডমিন করুন (SQL)

```sql
update public.profiles
set
  username = 'aamin924',
  full_name = 'মোঃ আল আমিন',
  full_name_en = 'Md. Al Amin',
  role = 'admin',
  active = true,
  approved = true,
  pending_approval = false
where email = 'aamin924@jph-ems.local'
   or username = 'aamin924';
```

প্রোফাইল না থাকলে:

```sql
insert into public.profiles (id, username, full_name, full_name_en, email, role, active, approved, pending_approval)
select id, 'aamin924', 'মোঃ আল আমিন', 'Md. Al Amin', email, 'admin', true, true, false
from auth.users
where email = 'aamin924@jph-ems.local'
on conflict (id) do update set
  role = 'admin', active = true, approved = true, pending_approval = false, username = 'aamin924';
```

---

## ধাপ ৫ — `index.html` কনফিগ

ফাইলের নিচে `window.EMS_SUPABASE` ব্লকে:

```js
window.EMS_SUPABASE = {
  url: 'https://YOUR_PROJECT.supabase.co',
  anonKey: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...',
  emailDomain: 'jph-ems.local'
};
```

রিপোতে একসাথে রাখুন:
- `index.html`
- `supabase-cloud.js`
- (ঐচ্ছিক) `supabase-schema.sql`, `SUPABASE-SETUP.md`

---

## ধাপ ৬ — GitHub Pages

1. ফাইলগুলো কমিট + পুশ
2. **Settings → Pages** → branch `main`, folder `/` (বা `/docs`)
3. Supabase **Site URL** = Pages URL

---

## কীভাবে কাজ করে

| কাজ | ক্লাউড |
|-----|--------|
| লগইন (`aamin924` + পাস) | Auth → ইমেইল `aamin924@jph-ems.local` |
| অ্যাডমিন ইউজার তৈরি | Auth user + profiles (`approved=true`) |
| পাবলিক রেজিস্টার | pending → অ্যাডমিন অনুমোদন পরে লগইন |
| কর্মচারী সেভ | ~1 সেকেন্ড পর `ems_kv` তে অটো পুশ |
| **☁️ ক্লাউড সেভ** বাটন | সব ডেটা এখনই পুশ |
| **⬇️ ক্লাউড লোড** বাটন | ক্লাউড → লোকাল + স্ক্রিন রিফ্রেশ |
| লোকাল ব্যাকআপ ডাউনলোড | সাথে `ems_backups` তেও স্ন্যাপশট |

অন্য ফোন/পিসি: একই অ্যাডমিন/ইউজার দিয়ে লগইন → অটো **ক্লাউড লোড**।

---

## প্রথমবার ডেটা আপলোড

1. অ্যাডমিন লগইন (যে ব্রাউজারে আগে localStorage ডেটা আছে)
2. **☁️ ক্লাউড সেভ** চাপুন  
   অথবা একবার যেকোনো কর্মচারী এডিট/সেভ করুন
3. অন্য ডিভাইসে লগইন → ডেটা আসবে

---

## টেস্ট চেকলিস্ট

- [ ] অ্যাডমিন লগইন (দুই ব্রাউজারে)
- [ ] কর্মচারী যোগ → অন্য ব্রাউজারে **⬇️ ক্লাউড লোড** / রিলগইন
- [ ] অ্যাডমিন নতুন ইউজার → সেই ইউজার অন্য ফোনে লগইন
- [ ] পাবলিক রেজিস্টার → pending → অ্যাডমিন approve → লগইন
- [ ] লগআউট

---

## ট্রাবলশুটিং

| সমস্যা | সমাধান |
|--------|--------|
| Invalid API key | `anonKey` ঠিক আছে কিনা |
| Email not confirmed | Confirm email **OFF** |
| অনুমোদন নেই | `profiles`: `approved=true`, `active=true` |
| ক্লাউড সেভ ব্যর্থ | SQL স্কিমা রান হয়েছে? লগইন আছে? |
| CORS / Failed to fetch | Site URL = GitHub Pages HTTPS URL |
| `supabase-cloud.js` 404 | রিপো রুটে ফাইল আছে? Pages path ঠিক? |
| ইউজার তৈরি সেশন ভাঙে | অ্যাডমিন দিয়ে আবার লগইন; প্রয়োজনে Dashboard থেকে user add |

---

## নিরাপত্তা নোট

- **anon key** পাবলিক — RLS পলিসিই ডেটা রক্ষা করে
- `service_role` key **কখনো** ফ্রন্টএন্ডে দেবেন না
- প্রোডাকশনে Confirm email চালু + আসল ইমেইল ব্যবহার করা ভালো
