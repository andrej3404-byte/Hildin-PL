# Hildin Programming Language

Hildin is an indentation-based programming language with a desktop editor,
interpreter, optional native-code toolchain, GUI helpers, local package
management, and an optional AI assistant. This project is in early alpha; see
[NOTICE.mc](./NOTICE.mc) before relying on a particular release.

## Getting started

### Windows installer

Run `setup.exe`, choose an installation location, and follow the prompts. The
installer checks for Python and can install a private Python runtime if needed.
It adds Hildin Studio and command-line shortcuts. You can also build a new
installer from a checkout with:

```powershell
powershell -ExecutionPolicy Bypass -File .\build_setup.ps1
```

### Run from a checkout

Use Python 3.10 or newer:

```powershell
.\hl.bat .\examples\hello.hlpl
.\hl.bat --check .\examples\features.hlpl
.\hl.bat --transpile .\examples\hello.hlpl
.\hl.bat
```

Run `hl --help` for command-line options. The no-argument command starts the
interactive prompt. Hildin source files normally use the `.hlpl` extension.

## Language capabilities

- Indentation-based blocks, Python-style expressions, functions, loops,
  conditionals, collections, and exception handling.
- Hildin declarations such as `let`, `var`, `const`, `proc`, and `func`.
- An independent Hildin tokenizer and parser, syntax checking, and a typed
  subset with immutable bindings and compile-time checks.
- Python 3 code generation and execution; a supported typed subset can generate
  C and native binaries.
- Standard-library access, project-local package installation, and GUI
  application helpers.
- Example programs in `examples\`, including a GUI calculator.

The language currently runs on Python. Native compilation, type coverage, and
the standard library are evolving; the alpha does not promise full Python
compatibility.

## Hildin Studio

Studio includes a source editor, syntax diagnostics, program output, terminal,
debug console, ports and extensions panels, project explorer, and AI assistant.
The assistant can answer Hildin questions and help create or edit supported
project files. It can use a configured remote AI provider or a local Ollama
model. Configure the provider, model, and your own provider key in Assistant
Settings; the key is not supplied by this repository.

The assistant requires a signed-in Hildin account. Accounts and conversation
sync use the project administrator's Supabase project; users do not run a
Hildin account server on their own computer.

## Configure Supabase account services

The repository contains only a Supabase client. To make account registration
available to users, configure a shared Supabase project and distribute its
**project URL** and **publishable/anon key** with users. These are public client
credentials, not a substitute for database security. **Never put a Supabase
`service_role` key in Studio, the installer, or a public repository.**

1. Create a Supabase project and copy its Project URL and publishable (or legacy
   anon) key from **Project Settings > API**.
2. In **Authentication > Providers > Email**, enable email sign-ups and email
   OTP. In **Authentication > Email Templates**, make the confirmation template
   display the six-digit code using `{{ .Token }}`. Configure SMTP in Supabase
   for production delivery and set the allowed redirect/site URLs for your
   product.
3. Open the Supabase SQL editor and run this schema and policy setup:

```sql
create table if not exists public.profiles (
    id uuid primary key references auth.users(id) on delete cascade,
    username text not null
        check (username ~ '^[A-Za-z0-9_]{3,24}$'),
    avatar_url text,
    created_at timestamptz not null default now()
);

create unique index if not exists profiles_username_ci
    on public.profiles (lower(username));
alter table public.profiles enable row level security;
grant select (id, username, avatar_url) on public.profiles to anon, authenticated;
grant insert (id, username, avatar_url), update (username, avatar_url)
    on public.profiles to authenticated;

drop policy if exists "Profiles are discoverable by username" on public.profiles;
create policy "Profiles are discoverable by username"
    on public.profiles for select to anon, authenticated using (true);
drop policy if exists "Users create their own profile" on public.profiles;
create policy "Users create their own profile"
    on public.profiles for insert to authenticated
    with check (auth.uid() = id);
drop policy if exists "Users update their own profile" on public.profiles;
create policy "Users update their own profile"
    on public.profiles for update to authenticated
    using (auth.uid() = id) with check (auth.uid() = id);

create table if not exists public.chats (
    id uuid primary key,
    user_id uuid not null references auth.users(id) on delete cascade,
    title text not null check (char_length(title) <= 200),
    messages jsonb not null default '[]'::jsonb,
    updated_at timestamptz not null default now()
);

create index if not exists chats_user_updated
    on public.chats (user_id, updated_at desc);
alter table public.chats enable row level security;
grant select, insert, update, delete on public.chats to authenticated;

drop policy if exists "Users manage their own chats" on public.chats;
create policy "Users manage their own chats"
    on public.chats for all to authenticated
    using (auth.uid() = user_id)
    with check (auth.uid() = user_id);

insert into storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
values ('avatars', 'avatars', true, 2097152,
        array['image/png', 'image/jpeg', 'image/gif', 'image/webp'])
on conflict (id) do update set
    public = excluded.public,
    file_size_limit = excluded.file_size_limit,
    allowed_mime_types = excluded.allowed_mime_types;

drop policy if exists "Users upload their own avatar" on storage.objects;
create policy "Users upload their own avatar"
    on storage.objects for insert to authenticated
    with check (
        bucket_id = 'avatars'
        and (storage.foldername(name))[1] = auth.uid()::text
    );
drop policy if exists "Users update their own avatar" on storage.objects;
create policy "Users update their own avatar"
    on storage.objects for update to authenticated
    using (
        bucket_id = 'avatars'
        and (storage.foldername(name))[1] = auth.uid()::text
    )
    with check (
        bucket_id = 'avatars'
        and (storage.foldername(name))[1] = auth.uid()::text
    );
```

4. In Hildin Studio, select the gear beside the account button and enter the
   project URL and publishable/anon key. Alternatively, set
   `HILDIN_SUPABASE_URL` and `HILDIN_SUPABASE_ANON_KEY` before launching Studio.
   Studio keeps manually entered settings in
   `%APPDATA%\HildinStudio\supabase.json`.
5. Share the same project URL and publishable/anon key with every user. All
   user data is stored in the shared Supabase project; row-level security ties
   each account's saved conversations to its Supabase Auth user.

Users register with an email code, then choose a username and password.
Subsequent sign-ins use the confirmed email address and password; username is
the public profile name, not a login alias. Configure Supabase Auth's SMTP
delivery before inviting users. Supabase's publishable/anon key is expected to
be visible in a desktop client; RLS policies, Auth settings, and storage
policies must remain enabled.

The former custom API stored accounts in a local SQLite database. Those old
local accounts are not automatically imported into Supabase; users must create
new accounts in the configured Supabase project. Reinstalling over an older
Hildin installation removes the obsolete account-server Python module but does
not delete that local database file.

## Project files

- `hildin.py` — language runtime and command-line interface.
- `hildin_frontend.py` — Hildin lexical and parser frontend.
- `hildin_native.py` — native-code generation.
- `hildin_panel.py` — Hildin Studio.
- `hildin_supabase.py` — Supabase Auth and data client (no local account API).
- `guide.mt` — extended language reference.
- `examples\` — runnable sample programs.
