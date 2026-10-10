# Deployment Guide — Hostinger VPS + CloudPanel

This guide takes the project from your computer to a live website on your own domain with HTTPS. No code changes are needed.

**Works for:** macOS and Windows 10/11
**Time needed:** about 45–60 minutes
**What you end up with:** `https://yourdomain.com` running the app 24/7, with a free SSL certificate and an automatic restart after server reboots.

---

## Before you start

Don't have a VPS or domain yet? Get them from Hostinger with this discount link:

```
  WATCH: Hostinger
  Get : Discount 75%
  URL: https://www.hostg.xyz/aff_c?offer_id=6&aff_id=139422
```

👉 [**Get Hostinger VPS + Domain — 75% off**](https://www.hostg.xyz/aff_c?offer_id=6&aff_id=139422)

You need:

| Item | Notes |
|---|---|
| Hostinger VPS | **KVM 1 or higher** (4 GB RAM). Smaller plans may crash during `npm run build`. [Get it here (75% off)](https://www.hostg.xyz/aff_c?offer_id=6&aff_id=139422) |
| A domain | This guide assumes the domain is bought from Hostinger. Domains from other providers work the same way: you just edit DNS at that provider. |
| Supabase project | With the database schema from `supabase/schema.sql` already applied. |
| API keys | OpenAI, Resend, Stripe (see the `.env.local` section below). |
| The project working locally | Run `npm install` and `npm run dev` on your computer first and check that it works. |

Throughout this guide, replace these placeholders with your own values:

| Placeholder | Example |
|---|---|
| `yourdomain.com` | `aicallagent.tech` |
| `YOUR_VPS_IP` | `88.222.241.24` |
| `siteuser` | the Site User you create in CloudPanel (Step 5), for example `aicallagent` |

---

## Step 1 — Prepare `.env.local`

In the project folder, open `.env.local` and fill in every value. The most important one for deployment:

```env
NEXT_PUBLIC_APP_URL=https://yourdomain.com
```

> ⚠️ Next.js bakes every `NEXT_PUBLIC_*` value into the app when you run `npm run build`. If you change any of them later, you must rebuild (see [Updating the site](#updating-the-site-later)).

Variables the app uses:

| Variable | Where to get it |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` | Supabase → Project Settings → API |
| `ADMIN_EMAILS` | Comma-separated emails allowed into the admin dashboard |
| `OPENAI_API_KEY`, `OPENAI_REALTIME_MODEL` | https://platform.openai.com/api-keys |
| `NEXT_PUBLIC_APP_URL` | `https://yourdomain.com` |
| `NEXT_PUBLIC_DEMO_BUSINESS_ID` | Optional. Leave empty to hide the landing-page demo widget |
| `NEXT_PUBLIC_ENABLE_VOICE` | Turns voice on or off |
| `RESEND_API_KEY`, `RESEND_FROM_EMAIL` | https://resend.com/api-keys |
| `RESEND_DEV_OVERRIDE_TO` | **While set, every app email goes to this one inbox.** Clear it once your domain is verified in Resend. |
| `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY`, `STRIPE_SECRET_KEY` | https://dashboard.stripe.com/apikeys (use live keys for real payments) |
| Plan names, prices and limits (`NEXT_PUBLIC_*_PLAN_*`, `*_AGENT_LIMIT`, `*_BOOKING_LIMIT`, `NEXT_PUBLIC_WEBSITE_BUILDER_PRICE_USD`) | Your own pricing |

---

## Step 2 — Set up the VPS on Hostinger

> Need a VPS? Buy **KVM 1 or higher** with this 75% discount link: https://www.hostg.xyz/aff_c?offer_id=6&aff_id=139422

In hPanel, open your VPS and start the setup wizard.

1. **Choose what to install** → open the **Control panel** tab → select **CloudPanel**.
   > Don't pick Plain OS, cPanel, Coolify, Dokploy or "OpenLiteSpeed and Node.js". CloudPanel is the simplest option for this project.
2. **Secure your VPS access**
   - **Panel password:** click **Generate** (or type a strong one) and **save it somewhere safe**. It's used for the CloudPanel `admin` login and for root SSH.
   - **Add SSH key:** skip it (optional).
   - Click **Next**.
3. **Additional features** → leave **Monarx malware scanner** unchecked (it targets PHP sites and uses RAM) → click **Finish setup**.
4. Wait 5–10 minutes. The final screen shows your SSH command, for example `ssh root@YOUR_VPS_IP`. **Note down your VPS IP.**

---

## Step 3 — Point your domain to the VPS

In hPanel → **Domains** → your domain → **DNS / Nameservers** → **DNS records**:

1. Find the **A** record with Name **`@`**. It points to Hostinger's parking IP by default.
2. Click the **pencil (edit)** icon, change **Content** to **`YOUR_VPS_IP`**, and save.
3. Leave the existing **CNAME `www` → `yourdomain.com`** record as it is. It follows the `@` record automatically.

The records should end up like this:

| Type | Name | Content |
|---|---|---|
| A | @ | `YOUR_VPS_IP` |
| CNAME | www | `yourdomain.com` |

> If there is no `www` CNAME record, add an **A** record with Name `www` pointing to `YOUR_VPS_IP` instead. Don't keep both.

**Check that DNS is working** (wait 5–30 minutes). In Terminal on Mac, or Command Prompt or PowerShell on Windows, run:

```bash
ping yourdomain.com
```

When the reply shows `YOUR_VPS_IP`, DNS is ready. Press **Ctrl + C** to stop the ping.

---

## Step 4 — Log in to CloudPanel

1. Open **`https://YOUR_VPS_IP:8443`** in your browser.
2. You'll see **"Your connection is not private"**. This is expected: CloudPanel uses a self-signed certificate for its admin page. Click **Advanced → Proceed to YOUR_VPS_IP (unsafe)**.
3. Log in:
   - **User Name:** `admin`
   - **Password:** the **panel password** from Step 2

<details>
<summary>Can't log in? Reset the admin password</summary>

Connect to the server as root (the password is the panel password):

```bash
ssh root@YOUR_VPS_IP
```

Then run:

```bash
clpctl user:reset:password --userName='admin' --password='YourNewStrongPassword123!'
```

You can also check the username and reset the password from hPanel → VPS → **Manage**.
</details>

---

## Step 5 — Create the Node.js site

In CloudPanel, go to **Sites → + ADD SITE → Create a Node.js Site** and fill in:

| Field | Value |
|---|---|
| Domain Name | `yourdomain.com` |
| Node.js Version | **20 LTS** |
| App Port | **3000** |
| Site User | any name, for example `aicallagent` (this is your `siteuser`) |
| Site User Password | a strong password. **Save it**, because you'll use it for SSH |

Click **Create**. You should see **"Site has been created."**

---

## Step 6 — Install the free SSL certificate (HTTPS)

HTTPS is **required**: browsers only allow microphone access on secure sites, so the voice agent won't work without it.

1. **Sites → Manage** (next to your domain) → **SSL/TLS** tab.
2. **Actions → New Let's Encrypt Certificate**.
3. Both `yourdomain.com` and `www.yourdomain.com` are pre-filled. Keep both.
4. Click **Create and Install**.

> The red box saying *"A DNS record pointing to this server is required…"* is only a reminder, not an error.

You should see **"Certificate has been installed."** with a **Let's Encrypt** row marked **Installed: Yes**. You can ignore the "Self-Signed" row. The certificate renews automatically.

> If this fails, DNS from Step 3 isn't ready yet. Wait a few minutes and try again.

---

## Step 7 — Create `project.zip` on your computer

The zip must **not** include `node_modules` or `.next`, which are huge and get rebuilt on the server. It **must** include the hidden `.env.local` file.

Open a terminal **inside the project folder**. In VS Code, open the project and go to **Terminal → New Terminal**.

### macOS (and Linux)

```bash
zip -r ~/Desktop/project.zip . -x "node_modules/*" ".next/*" ".DS_Store" "tsconfig.tsbuildinfo"
```

This creates `project.zip` on your Desktop. `zip` is built into macOS.

### Windows 10/11 — Option A: command (recommended)

Works in both PowerShell and Command Prompt:

```bash
tar -a -cf ..\project.zip --exclude=node_modules --exclude=.next --exclude=tsconfig.tsbuildinfo .
```

This creates `project.zip` **next to** your project folder (one level up). `tar` is built into Windows 10/11.

### Windows 10/11 — Option B: File Explorer (no commands)

1. Open the project folder.
2. Show hidden files so `.env.local` is visible: **View → Show → Hidden items** (Windows 11) or tick **View → Hidden items** (Windows 10).
3. Select **everything inside** the folder **except** `node_modules` and `.next`.
4. Right-click → **Compress to ZIP file** (Windows 11) or **Send to → Compressed (zipped) folder** (Windows 10).
5. Rename the file to `project.zip`.

> ❌ Renaming a folder to `.zip` does not create a zip. Always use one of the methods above.
> ❌ Don't use `.z` or other formats. Use **`.zip`**.

---

## Step 8 — Upload and extract the code

1. In CloudPanel → **Sites → Manage** → **File Manager** tab.
2. Open **`htdocs`**, then **`yourdomain.com`**.
3. Delete any default files inside it.
4. Click **Upload** and choose `project.zip`. Wait for the upload to finish.
5. Right-click `project.zip` → **Extract**, then delete the zip.
6. Check that `package.json`, `src`, `next.config.mjs`, `supabase` and **`.env.local`** are there.

> 📁 **Note your project path.** Depending on how the zip was made, the files land either:
> - directly in `htdocs/yourdomain.com/`, or
> - inside a subfolder, such as `htdocs/yourdomain.com/project/`.
>
> **Both are fine.** CloudPanel only forwards the domain to port 3000, so the app can run from either folder. Use the folder that contains `package.json` in the next steps. This guide uses `htdocs/yourdomain.com/project`; drop the `/project` part if your files are directly in `yourdomain.com`.

---

## Step 9 — Connect to the server with SSH

Use **Terminal** on Mac or **PowerShell** on Windows. SSH is built into both.

```bash
ssh siteuser@YOUR_VPS_IP
```

- If asked *"Are you sure you want to continue connecting?"*, type `yes` and press Enter.
- Enter the **Site User Password** from Step 5. Nothing appears on screen while you type; that's normal.

Your prompt should change to something like `siteuser@srv1234567:~$`.

---

## Step 10 — Install, build and start the app

Run these **one at a time**:

```bash
cd htdocs/yourdomain.com/project
npm install
npm run build
```

- `npm install` takes about 1–3 minutes.
- `npm run build` takes about 2–5 minutes. Success ends with a table of routes, `○ (Static)` and `ƒ (Dynamic)`, and no red errors.

Then start the app with PM2, which keeps it running after you close the terminal:

```bash
npm install -g pm2
pm2 start npm --name app -- start
pm2 save
```

After `pm2 start`, you should see a table with **app** and status **online**.

> ⚠️ **Run `pm2 start npm --name app -- start` only ONCE.** Running it again starts extra copies that show as **errored** (port 3000 is already in use). If that happens, see [Troubleshooting](#troubleshooting).
>
> ⚠️ Typing `pm2 start` **on its own** gives `File ecosystem.config.js not found`. That's harmless: nothing was started or broken. Use `pm2 restart app` instead.

---

## Step 11 — Auto-start the app after a server reboot

```bash
crontab -e
```

- If asked to choose an editor, type `1` (nano) and press Enter.
- Move to the **bottom** of the file with the **↓ arrow** key.
- Paste this line (Cmd + V on Mac, right-click on Windows), changing the path to match yours:

```
@reboot bash -c 'source ~/.nvm/nvm.sh && cd ~/htdocs/yourdomain.com/project && pm2 resurrect'
```

- Save: **Ctrl + O**, then **Enter**. Exit: **Ctrl + X**.

You should see `crontab: installing new crontab`. If it also says `no crontab for siteuser - using an empty one`, that's normal the first time.

> The `source ~/.nvm/nvm.sh` part is needed because CloudPanel installs Node.js through nvm, and cron doesn't load it on its own.

Check that the line was saved:

```bash
crontab -l
```

---

## Step 12 — Configure Supabase for your domain

Without this step, signup confirmation emails send users to `localhost`.

1. Go to https://supabase.com/dashboard and open your project.
2. **Authentication → URL Configuration**.
3. **Site URL:** `https://yourdomain.com`, then click **Save**.
4. **Redirect URLs → Add URL**, and add both:
   ```
   https://yourdomain.com/**
   https://www.yourdomain.com/**
   ```
   Click **Save**. Keep `http://localhost:3000/**` if you also develop locally.

---

## Step 13 — You're live 🎉

Open **`https://yourdomain.com`**. You should see the padlock 🔒 and the landing page.

Test:

- [ ] Sign up with a new email, click the confirmation email, and land on `/dashboard`
- [ ] Log in and log out
- [ ] Start a voice call (allow microphone access)
- [ ] Book an appointment
- [ ] Admin dashboard, logged in with an email listed in `ADMIN_EMAILS`

---

## Updating the site later

1. Make your changes locally and test them.
2. Create a new `project.zip` (Step 7) and upload and extract it in File Manager over the old files (Step 8). Alternatively, upload only the changed files.
3. On the server:

```bash
ssh siteuser@YOUR_VPS_IP
cd htdocs/yourdomain.com/project
npm install
npm run build
pm2 restart app
```

Always rebuild after changing `.env.local`. `NEXT_PUBLIC_*` values only take effect after `npm run build`.

---

## Useful PM2 commands

| What | Command |
|---|---|
| Check if the app is running | `pm2 status` |
| View live logs and errors | `pm2 logs app` (exit with Ctrl + C) |
| Last 30 log lines | `pm2 logs app --lines 30` |
| Restart the app | `pm2 restart app` |
| Stop the app | `pm2 stop app` |
| Save the current process list | `pm2 save` |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| **"Your connection is not private"** on `https://YOUR_VPS_IP:8443` | Expected for the CloudPanel admin page. Click **Advanced → Proceed**. Your website itself uses a real Let's Encrypt certificate. |
| **Let's Encrypt fails** | DNS isn't pointing to the VPS yet. Check with `ping yourdomain.com`, wait, then retry. |
| **502 Bad Gateway** | The app isn't running. Run `pm2 status`; if it isn't online, run `pm2 logs app --lines 30` and look for the error. |
| **`pm2 status` shows extra `errored` rows** (for example ids 1 and 2) | `pm2 start` was run more than once. Remove the extras with `pm2 delete 1 2` (use the ids shown), then run `pm2 save`. Only one `app` should be **online**. |
| **`File ecosystem.config.js not found`** | You typed `pm2 start` with nothing after it. Harmless; use `pm2 restart app`. |
| **`npm run build` gets "Killed" or runs out of memory** | The VPS has too little RAM. Use KVM 1 (4 GB) or higher. |
| **Site works but shows old content or settings** | You changed `.env.local` or code without rebuilding. Run `npm run build && pm2 restart app`. |
| **Signup email link opens `localhost`** | Do Step 12 (Supabase URL Configuration). |
| **All app emails arrive in one inbox** | `RESEND_DEV_OVERRIDE_TO` is set. Clear it after verifying your domain in Resend, then rebuild and restart. |
| **Microphone or voice doesn't work** | The site must be opened over `https://`. Check the SSL step and allow microphone access in the browser. |
| **`ssh` asks for a password and rejects it** | Use the **Site User** password from Step 5 for `siteuser@…`. Root login uses the panel password. |
| **Lost the CloudPanel login** | See the reset instructions in Step 4. |
