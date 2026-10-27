# Data Model

_Living doc. Last updated: 2026-07-05. This is a starter derived from the `supabase/*.sql`
files; replace the table list with a generated schema and ERD (see below)._

## How the schema is currently managed

The schema lives as ~34 loose `.sql` files in `supabase/`, not a versioned migration
folder. There is no ordering guarantee, no down-migrations, and no record of which files
have been applied to the live project. This is [GAP-REPORT.md](../GAP-REPORT.md) G-06 and
is the single biggest data-layer risk, because no one can reproduce the
database from the repo with confidence.

**Recommended fix:** adopt Supabase's migration convention (`supabase/migrations/` with
timestamped files) and record applied state. Until then, `supabase/schema.sql` is the
closest thing to a source of truth.

## Domains and their SQL files

| Domain | Files |
| --- | --- |
| Core / profiles | `schema.sql`, `profiles_migration.sql`, `profiles_rls_fix.sql`, `avatars_bucket.sql`, `push_token_migration.sql` |
| Destinations & vibes | `destinations_table.sql`, `destinations_seed.sql`, `destinations_photos_migration.sql`, `destinations_photos_seed.sql`, `vibes_table.sql`, `vibe_rankings_table.sql`, `vibe_blogs_table.sql`, `caribbean_vibes_expansion.sql`, `card_reactions.sql` |
| Booking & orders | `orders_table.sql`, `booking_in_progress.sql`, `booking_passengers_columns.sql`, `saved_passengers_migration.sql`, `together_booking.sql` |
| Wander Together / groups | `wander_groups_migration.sql`, `wander_groups_vibes_migration.sql`, `friends_migration.sql`, `friends_email_migration.sql` |
| Affiliates | `affiliate_infrastructure.sql`, `affiliate_attribution_trigger.sql`, `affiliate_create_function.sql`, `affiliate_admins.sql`, `affiliate_commissions_expansion.sql`, `affiliate_dashboard_seed.sql` |
| App config & assets | `app_config.sql`, `app_icons_table.sql`, `app_icons_seed.sql`, `dashboard_config.sql`, `dashboard_config_migration.sql`, `dashboard_images.sql` |

## Row-Level Security

RLS is **intentionally disabled** on the Wander Together tables (`trip_participants` and
related), because the dashboard's "Enable RLS" toggle triggers infinite recursion. This
is a deliberate decision with compensating controls, recorded in
[ADR-0004](../decisions/0004-rls-disabled-together.md). It must be presented as a decision,
not discovered as a gap, or it reads as a vulnerability.

## Generate the real schema and ERD

This repo has the Supabase MCP connected. To populate an accurate schema and ERD:

1. List tables and columns via the Supabase MCP (`list_tables`) and paste the structured
   output here.
2. Generate a Mermaid ER diagram from the foreign-key relationships.
3. Commit `types` from `generate_typescript_types` alongside, so the data model and the
   client types stay in sync.

_TODO: run the above and replace this section with the generated schema._
