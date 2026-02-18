# Notes Database Schema (PostgreSQL)

This schema is for the `notes_database` container.

Connection info is stored in:
- `notes_database/db_connection.txt` (contains a `psql postgresql://...` command)

## Extensions

- `pgcrypto` is enabled to provide `gen_random_uuid()` for UUID primary keys.

## Tables

### `notes`
Stores notes content and metadata.

Columns:
- `id` UUID PRIMARY KEY DEFAULT `gen_random_uuid()`
- `title` TEXT NOT NULL DEFAULT `''`
- `content` TEXT NOT NULL DEFAULT `''`
- `is_pinned` BOOLEAN NOT NULL DEFAULT `FALSE`
- `is_favorited` BOOLEAN NOT NULL DEFAULT `FALSE`
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT `now()`
- `updated_at` TIMESTAMPTZ NOT NULL DEFAULT `now()`
- `search_vector` TSVECTOR (generated/stored): `to_tsvector('simple', coalesce(title,'') || ' ' || coalesce(content,''))`

Behavior:
- A trigger updates `updated_at` automatically on any UPDATE.

Indexes:
- `idx_notes_created_at` on `(created_at DESC)`
- `idx_notes_pinned_created_at` on `(is_pinned DESC, created_at DESC)`
- `idx_notes_favorited_created_at` on `(is_favorited DESC, created_at DESC)`
- `idx_notes_search_vector` GIN on `(search_vector)`

### `tags`
Stores unique tags.

Columns:
- `id` UUID PRIMARY KEY DEFAULT `gen_random_uuid()`
- `name` TEXT NOT NULL
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT `now()`

Indexes/Constraints:
- Case-insensitive uniqueness enforced by unique index:
  - `idx_tags_name_unique_ci` on `(lower(name))`

### `note_tags`
Join table for the many-to-many relationship between notes and tags.

Columns:
- `note_id` UUID NOT NULL REFERENCES `notes(id)` ON DELETE CASCADE
- `tag_id` UUID NOT NULL REFERENCES `tags(id)` ON DELETE CASCADE
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT `now()`

Constraints:
- PRIMARY KEY `(note_id, tag_id)` prevents duplicate tag assignment per note

Indexes:
- `idx_note_tags_note_id` on `(note_id)`
- `idx_note_tags_tag_id` on `(tag_id)`

## Notes on querying

- Full-text search:
  - `WHERE search_vector @@ plainto_tsquery('simple', :query)`
- Tag filtering typically joins via `note_tags` and `tags` (using `idx_note_tags_tag_id` and the tag unique index).
