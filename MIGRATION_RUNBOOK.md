# Hiệp Bình Lắng Nghe Migration Runbook

This project has been scaffolded to move away from Base44 while keeping the current React UI.

## Target Architecture

- Frontend: Vite React on Cloudflare Pages
- API: Cloudflare Pages Functions in `functions/api/[[path]].js`
- Database: Cloudflare D1
- File uploads: Cloudflare R2
- Email automation: Resend
- Admin login: Cloudflare Access
- Domain/DNS: IONOS or Cloudflare DNS

Expected monthly cost can stay below USD 10 if traffic and email volume remain moderate.

## What Was Replaced

- `@base44/vite-plugin` was removed from `vite.config.js`.
- `src/api/base44Client.js` now mimics the Base44 client API and calls local `/api/...` endpoints.
- `src/lib/AuthContext.jsx` no longer depends on Base44 app settings.
- Admin authentication now expects Cloudflare Access plus matching rows in the D1 `users` table.
- Base44 functions were ported into Cloudflare Pages Functions.
- Base44 entities were mapped to D1 tables in `migrations/0001_init.sql`.

## Cloudflare Setup

1. Create a Cloudflare account.
2. Create a new Pages project connected to the GitHub repo.
3. Build settings:
   - Build command: `npm run build`
   - Output directory: `dist`
   - Root directory: repository root
4. Create a D1 database:
   - Name: `hiep-binh-lang-nghe`
   - Copy its database ID into `wrangler.toml`.
5. Run the migration:
   - Local/staging: `npx wrangler d1 migrations apply hiep-binh-lang-nghe`
   - Production: `npx wrangler d1 migrations apply hiep-binh-lang-nghe --remote`
6. Create an R2 bucket:
   - Name: `hiep-binh-lang-nghe-files`
7. Bind the D1 database and R2 bucket in the Pages project settings:
   - D1 binding: `DB`
   - R2 binding: `FILES`

## Required Environment Variables

Set these in Cloudflare Pages production environment:

- `APP_ORIGIN=https://hiepbinhlangnghe.com`
- `FROM_EMAIL=Hiệp Bình Lắng Nghe <no-reply@hiepbinhlangnghe.com>`
- `RESEND_API_KEY=...`
- `CRON_SECRET=...` optional but recommended for deadline checks

For local testing only:

- `DEV_AUTH_EMAIL=your-admin-email@example.com`

## Resend Setup

1. Create a Resend account.
2. Add and verify the domain `hiepbinhlangnghe.com`.
3. Add the DNS records Resend provides inside IONOS DNS.
4. Create an API key.
5. Set `RESEND_API_KEY` in Cloudflare Pages.
6. Test:
   - public issue submission confirmation email
   - department notification email
   - overdue reminder email

## Admin Authentication

Recommended: Cloudflare Access.

1. In Cloudflare Zero Trust, create an Access application.
2. Protect:
   - `/admin/*`
   - `/api/entities/*`
   - `/api/functions/notifyDepartment`
   - `/api/functions/checkDeadlines`, unless using `CRON_SECRET`
3. Allow only approved government/admin emails.
4. In D1, replace the seed admin:

```sql
DELETE FROM users WHERE email = 'replace-admin-email@example.com';

INSERT INTO users (id, email, full_name, role, department)
VALUES ('usr_admin_001', 'real-admin@example.com', 'Real Admin Name', 'admin', '');
```

5. Add department staff:

```sql
INSERT INTO users (id, email, full_name, role, department)
VALUES
  ('usr_receive_001', 'receiver@example.com', 'Receiving Staff', 'receiving_staff', 'tiep_nhan'),
  ('usr_kt_001', 'kt@example.com', 'Kinh Te Staff', 'department_staff', 'kinh_te_ha_tang'),
  ('usr_vh_001', 'vh@example.com', 'Van Hoa Staff', 'department_staff', 'van_hoa_xa_hoi'),
  ('usr_ca_001', 'ca@example.com', 'Cong An Staff', 'department_staff', 'cong_an');
```

## IONOS Domain Cutover

1. Keep the Base44 site live.
2. In IONOS, lower DNS TTL if available.
3. Add `hiepbinhlangnghe.com` and `www.hiepbinhlangnghe.com` as custom domains in Cloudflare Pages.
4. Follow Cloudflare's DNS instructions.
5. After SSL is active, test the Cloudflare preview domain first.
6. Switch IONOS DNS records to the Cloudflare Pages target.
7. Keep Base44 unchanged until production tests pass.

## Final Acceptance Tests

Public:

- Home page loads.
- Submit issue with required fields.
- Upload image/PDF/doc attachment.
- Receive tracking code.
- Confirmation email sends.
- Track issue by code.
- Resolve link opens and accepts result note plus attachment.

Admin:

- Admin can access `/admin/dashboard`.
- Unauthorized email cannot access admin.
- Admin can list issues.
- Receiving staff can receive and forward new issue.
- Correct department receives notification email.
- Department staff can view assigned issue.
- Department staff can resolve issue.
- Citizen info page is admin-only.
- User list is admin-only.

Security:

- Public cannot call admin entity APIs.
- Department staff cannot see other department issues.
- Uploaded files are not writable from public URLs.
- Resend API key is not exposed to frontend.
- No Base44 environment variables remain required.

## Known Follow-Ups

- Clean up mojibake Vietnamese text in exported React files.
- Replace Base44-hosted images with local/R2 copies before fully deleting Base44.
- Add a scheduled trigger for `checkDeadlines`.
- Regenerate `package-lock.json` after npm works normally on the machine.
