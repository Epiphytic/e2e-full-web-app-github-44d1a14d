# SQLite Web Editor - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a web application that allows users to authenticate via JWT and manage SQLite databases (create/delete tables, modify table structures) through an htmx-powered frontend served by a Rust backend.

**Architecture:** Axum web framework serves HTML templates with htmx for dynamic interactions. SQLite is the data store accessed via sqlx. JWT authentication uses RS256 with a local CA, exposing public keys at `.well-known/jwks.json`. Playwright (Node.js) handles E2E tests against the running server.

**Tech Stack:**
- **Backend:** Rust + Axum + sqlx (SQLite) + jsonwebtoken + askama (templates)
- **Frontend:** HTML + htmx + minimal CSS
- **Auth:** RS256 JWT with local CA, JWKS endpoint, CSRF protection via custom header + cookie security flags (`HttpOnly`, `Secure`, `SameSite=Strict`)
- **Testing:** Playwright (Node.js, not Rust) for E2E
- **CI/CD:** GitHub Actions (super-linter, dependency-review-action, Playwright)

---

## Overview

The application is a browser-based SQLite database editor. Users authenticate with JWT tokens (signed by a local CA using RS256). Once authenticated, they can:
- View all tables in the database
- Create new tables with specified columns
- Delete existing tables
- Add columns to existing tables
- Remove columns from existing tables (via table recreation since SQLite has limited ALTER TABLE support)

The backend is a Rust binary using Axum. It serves HTML pages that use htmx for partial page updates (no full-page reloads for CRUD operations). The frontend is server-rendered HTML with htmx attributes - no JavaScript framework.

E2E tests use Playwright (Node.js) to test the full flow: login with a short-lived JWT, table CRUD, and column modifications.

## Risk Areas

1. **SQLite ALTER TABLE limitations** - SQLite cannot drop columns in older versions (pre-3.35.0). The implementation must handle this by recreating tables when removing columns. Need to verify the SQLite version bundled with sqlx.

2. **JWT key management in CI** - The local CA private key must exist at build/test time but must never be committed. CI needs a step to generate ephemeral keys for testing.

3. **Playwright + Rust integration** - Playwright is Node.js-based. The E2E test setup must start the Rust server as a subprocess, wait for it to be ready, then run Playwright tests. This requires careful process lifecycle management.

4. **htmx partial responses** - Each htmx endpoint must return HTML fragments (not full pages). Need clear separation between full-page routes and htmx partial routes.

5. **SQLite concurrent access** - SQLite has limited concurrent write support. The connection pool configuration in sqlx must use WAL mode and appropriate pool sizing.

6. **Super-linter configuration** - Super-linter runs many linters by default. Need to configure it to only run relevant linters (Rust/clippy, HTML, YAML, Markdown) to avoid false positives and long CI times.

7. **Column type handling** - Need to define a supported set of SQLite column types for the UI and validate user input server-side to prevent SQL injection.

8. **Cookie security and CSRF protection** - The `token` cookie MUST always be set via `build_auth_cookie()` (see CRUISE-004), which enforces three critical security flags: `HttpOnly` (prevents XSS-based token theft via JavaScript), `Secure` (ensures HTTPS-only transmission), and `SameSite=Strict` (prevents cross-site cookie transmission). **Never set the token cookie via client-side JavaScript or without these flags.** Additionally, the implementation uses a multi-layer CSRF defense: (a) `SameSite=Strict` prevents cross-site cookie transmission, (b) the `auth_middleware` requires a custom `X-Requested-With: XMLHttpRequest` header on all state-changing requests (POST/PUT/DELETE) when using cookie auth — browsers block custom headers on cross-origin requests without CORS preflight, and (c) the `base.html` template sets `hx-headers='{"X-Requested-With": "XMLHttpRequest"}'` on the `<body>` tag so htmx automatically includes this header. See Task CRUISE-004 Step 5b for CSRF-specific tests.

---

## Task Dependency Graph

```
CRUISE-001 (.gitignore + project skeleton)
    |
    +-- CRUISE-002 (Cargo.toml + dependencies)
    |       |
    |       +-- CRUISE-003 (SQLite DB layer)
    |       |       |
    |       |       +-- CRUISE-005 (Table CRUD handlers)
    |       |       |       |
    |       |       |       +-- CRUISE-007 (htmx table management UI)
    |       |       |       |       |
    |       |       |       |       +-- CRUISE-009 (Playwright E2E: tables)
    |       |       |       |
    |       |       +-- CRUISE-006 (Column modification handlers)
    |       |               |
    |       |               +-- CRUISE-008 (htmx column management UI)
    |       |                       |
    |       |                       +-- CRUISE-009
    |       |
    |       +-- CRUISE-004 (JWT auth + JWKS endpoint)
    |               |
    |               +-- CRUISE-005
    |               +-- CRUISE-006
    |               +-- CRUISE-009 (Playwright E2E: auth)
    |
    +-- CRUISE-010 (GitHub Actions CI/CD)
```

---

## Tasks

### Task CRUISE-001: Project Skeleton and .gitignore

**Files:**
- Create: `.gitignore`
- Create: `src/main.rs` (minimal placeholder)
- Create: `templates/` (empty directory marker)
- Create: `static/` (empty directory marker)

**Step 1: Write the .gitignore file**

```gitignore
# Rust build artifacts
/target/
**/*.rs.bk
*.pdb

# Dependencies
Cargo.lock is committed for binaries, but ignore if library
# Cargo.lock  # Keep for binary projects

# SQLite databases
*.db
*.sqlite
*.sqlite3
*.db-journal
*.db-wal
*.db-shm

# Keys and credentials
*.pem
*.key
*.crt
*.cert
*.p12
*.pfx
*.jks
certs/
keys/
jwt_private*
jwt_public*
ca.*

# Environment files
.env
.env.*
!.env.example

# Log files
*.log
logs/

# Editor/IDE files
.vscode/
.idea/
*.swp
*.swo
*~
.project
.classpath
.settings/
*.sublime-project
*.sublime-workspace
.vim/
tags
TAGS

# OS files - macOS
.DS_Store
.AppleDouble
.LSOverride
._*
.Spotlight-V100
.Trashes

# OS files - Windows
Thumbs.db
ehthumbs.db
Desktop.ini
$RECYCLE.BIN/

# OS files - Linux
*~
.directory

# Node.js (for Playwright tests)
node_modules/
package-lock.json
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Playwright
playwright-report/
test-results/
playwright/.cache/
blob-report/

# Temporary files
*.tmp
*.temp
*.bak
*.orig

# Build artifacts
dist/
build/
out/

# Fork-join directories
.fork-join/

# Coverage
coverage/
*.lcov
tarpaulin-report.html
```

**Step 2: Create minimal src/main.rs placeholder**

```rust
fn main() {
    println!("SQLite Web Editor - placeholder");
}
```

**Step 3: Create directory structure markers**

```bash
mkdir -p templates/partials static/css tests/e2e certs
touch templates/.gitkeep static/.gitkeep static/css/.gitkeep tests/e2e/.gitkeep
```

**Step 4: Commit**

```bash
git add .gitignore src/main.rs templates/ static/ tests/
git commit -m "feat: add project skeleton with .gitignore"
```

---

### Task CRUISE-002: Cargo.toml and Dependencies

**Files:**
- Create: `Cargo.toml`
- Modify: `src/main.rs`

**Step 1: Write Cargo.toml with all dependencies**

```toml
[package]
name = "sqlite-web-editor"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { version = "0.8", features = ["macros"] }
axum-extra = { version = "0.10", features = ["cookie"] }
askama = "0.12"
askama_axum = "0.4"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.8", features = ["runtime-tokio", "sqlite"] }
jsonwebtoken = "9"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tower = "0.5"
tower-http = { version = "0.6", features = ["fs", "cors"] }
base64 = "0.22"
rsa = { version = "0.9", features = ["pem"] }
tracing = "0.1"
tracing-subscriber = "0.3"

[dev-dependencies]
reqwest = { version = "0.12", features = ["json", "cookies"] }
tokio-test = "0.4"
```

**Step 2: Update src/main.rs to verify dependencies compile**

```rust
use axum::Router;
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
    tracing_subscriber::init();

    let app = Router::new();

    let listener = TcpListener::bind("127.0.0.1:3000").await.unwrap();
    tracing::info!("listening on {}", listener.local_addr().unwrap());
    axum::serve(listener, app).await.unwrap();
}
```

**Step 3: Verify it compiles**

```bash
cargo check
```

Expected: compiles with no errors (warnings OK at this stage).

**Step 4: Commit**

```bash
git add Cargo.toml Cargo.lock src/main.rs
git commit -m "feat: add Cargo.toml with all dependencies"
```

---

### Task CRUISE-003: SQLite Database Layer

**Files:**
- Create: `src/db.rs`
- Create: `src/models.rs`
- Modify: `src/main.rs` (add module declarations)

**Step 1: Write the failing test for db module**

Add to `src/db.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use sqlx::SqlitePool;

    #[tokio::test]
    async fn test_list_tables_empty_db() {
        let pool = SqlitePool::connect("sqlite::memory:").await.unwrap();
        let tables = list_tables(&pool).await.unwrap();
        assert!(tables.is_empty());
    }

    #[tokio::test]
    async fn test_create_and_list_table() {
        let pool = SqlitePool::connect("sqlite::memory:").await.unwrap();
        let columns = vec![
            ColumnDef { name: "id".into(), col_type: "INTEGER".into() },
            ColumnDef { name: "name".into(), col_type: "TEXT".into() },
        ];
        create_table(&pool, "users", &columns).await.unwrap();
        let tables = list_tables(&pool).await.unwrap();
        assert_eq!(tables.len(), 1);
        assert_eq!(tables[0].name, "users");
    }

    #[tokio::test]
    async fn test_drop_table() {
        let pool = SqlitePool::connect("sqlite::memory:").await.unwrap();
        let columns = vec![
            ColumnDef { name: "id".into(), col_type: "INTEGER".into() },
        ];
        create_table(&pool, "temp_table", &columns).await.unwrap();
        drop_table(&pool, "temp_table").await.unwrap();
        let tables = list_tables(&pool).await.unwrap();
        assert!(tables.is_empty());
    }

    #[tokio::test]
    async fn test_get_columns() {
        let pool = SqlitePool::connect("sqlite::memory:").await.unwrap();
        let columns = vec![
            ColumnDef { name: "id".into(), col_type: "INTEGER".into() },
            ColumnDef { name: "email".into(), col_type: "TEXT".into() },
        ];
        create_table(&pool, "accounts", &columns).await.unwrap();
        let cols = get_columns(&pool, "accounts").await.unwrap();
        assert_eq!(cols.len(), 2);
        assert_eq!(cols[0].name, "id");
        assert_eq!(cols[1].name, "email");
    }

    #[tokio::test]
    async fn test_add_column() {
        let pool = SqlitePool::connect("sqlite::memory:").await.unwrap();
        let columns = vec![
            ColumnDef { name: "id".into(), col_type: "INTEGER".into() },
        ];
        create_table(&pool, "items", &columns).await.unwrap();
        add_column(&pool, "items", &ColumnDef { name: "price".into(), col_type: "REAL".into() }).await.unwrap();
        let cols = get_columns(&pool, "items").await.unwrap();
        assert_eq!(cols.len(), 2);
        assert_eq!(cols[1].name, "price");
    }

    #[tokio::test]
    async fn test_remove_column() {
        let pool = SqlitePool::connect("sqlite::memory:").await.unwrap();
        let columns = vec![
            ColumnDef { name: "id".into(), col_type: "INTEGER".into() },
            ColumnDef { name: "name".into(), col_type: "TEXT".into() },
            ColumnDef { name: "temp".into(), col_type: "TEXT".into() },
        ];
        create_table(&pool, "people", &columns).await.unwrap();
        remove_column(&pool, "people", "temp").await.unwrap();
        let cols = get_columns(&pool, "people").await.unwrap();
        assert_eq!(cols.len(), 2);
        assert!(cols.iter().all(|c| c.name != "temp"));
    }

    #[tokio::test]
    async fn test_create_table_rejects_sql_injection() {
        let pool = SqlitePool::connect("sqlite::memory:").await.unwrap();
        let columns = vec![
            ColumnDef { name: "id".into(), col_type: "INTEGER".into() },
        ];
        let result = create_table(&pool, "users; DROP TABLE--", &columns).await;
        assert!(result.is_err());
    }
}
```

**Step 2: Run tests to verify they fail**

```bash
cargo test --lib db::tests
```

Expected: FAIL - functions not defined.

**Step 3: Write the models**

Create `src/models.rs`:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TableInfo {
    pub name: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ColumnDef {
    pub name: String,
    pub col_type: String,
}
```

**Step 4: Implement db.rs**

```rust
use crate::models::{ColumnDef, TableInfo};
use sqlx::SqlitePool;

const ALLOWED_NAME_CHARS: &str = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789_";
const ALLOWED_TYPES: &[&str] = &["INTEGER", "TEXT", "REAL", "BLOB", "NUMERIC"];

fn validate_identifier(name: &str) -> Result<(), String> {
    if name.is_empty() {
        return Err("Name cannot be empty".into());
    }
    if name.len() > 128 {
        return Err("Name too long".into());
    }
    if !name.chars().all(|c| ALLOWED_NAME_CHARS.contains(c)) {
        return Err(format!("Invalid characters in identifier: {}", name));
    }
    if name.chars().next().unwrap().is_ascii_digit() {
        return Err("Identifier cannot start with a digit".into());
    }
    Ok(())
}

fn validate_column_type(col_type: &str) -> Result<(), String> {
    if ALLOWED_TYPES.contains(&col_type.to_uppercase().as_str()) {
        Ok(())
    } else {
        Err(format!("Invalid column type: {}. Allowed: {:?}", col_type, ALLOWED_TYPES))
    }
}

pub async fn list_tables(pool: &SqlitePool) -> Result<Vec<TableInfo>, String> {
    let rows = sqlx::query_scalar::<_, String>(
        "SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%' ORDER BY name"
    )
    .fetch_all(pool)
    .await
    .map_err(|e| e.to_string())?;

    Ok(rows.into_iter().map(|name| TableInfo { name }).collect())
}

pub async fn create_table(pool: &SqlitePool, table_name: &str, columns: &[ColumnDef]) -> Result<(), String> {
    validate_identifier(table_name)?;
    if columns.is_empty() {
        return Err("At least one column is required".into());
    }
    for col in columns {
        validate_identifier(&col.name)?;
        validate_column_type(&col.col_type)?;
    }

    let col_defs: Vec<String> = columns
        .iter()
        .map(|c| format!("\"{}\" {}", c.name, c.col_type.to_uppercase()))
        .collect();
    let sql = format!("CREATE TABLE \"{}\" ({})", table_name, col_defs.join(", "));

    sqlx::query(&sql)
        .execute(pool)
        .await
        .map_err(|e| e.to_string())?;

    Ok(())
}

pub async fn drop_table(pool: &SqlitePool, table_name: &str) -> Result<(), String> {
    validate_identifier(table_name)?;
    let sql = format!("DROP TABLE IF EXISTS \"{}\"", table_name);
    sqlx::query(&sql)
        .execute(pool)
        .await
        .map_err(|e| e.to_string())?;
    Ok(())
}

pub async fn get_columns(pool: &SqlitePool, table_name: &str) -> Result<Vec<ColumnDef>, String> {
    validate_identifier(table_name)?;
    let sql = format!("PRAGMA table_info(\"{}\")", table_name);
    let rows = sqlx::query_as::<_, (i64, String, String, i64, Option<String>, i64)>(&sql)
        .fetch_all(pool)
        .await
        .map_err(|e| e.to_string())?;

    Ok(rows.into_iter().map(|(_, name, col_type, _, _, _)| ColumnDef { name, col_type }).collect())
}

pub async fn add_column(pool: &SqlitePool, table_name: &str, column: &ColumnDef) -> Result<(), String> {
    validate_identifier(table_name)?;
    validate_identifier(&column.name)?;
    validate_column_type(&column.col_type)?;

    let sql = format!(
        "ALTER TABLE \"{}\" ADD COLUMN \"{}\" {}",
        table_name, column.name, column.col_type.to_uppercase()
    );
    sqlx::query(&sql)
        .execute(pool)
        .await
        .map_err(|e| e.to_string())?;
    Ok(())
}

pub async fn remove_column(pool: &SqlitePool, table_name: &str, column_name: &str) -> Result<(), String> {
    validate_identifier(table_name)?;
    validate_identifier(column_name)?;

    // Use table recreation pattern for broad SQLite version compatibility
    // (ALTER TABLE DROP COLUMN is only available in SQLite 3.35.0+).
    // Steps: get current columns, filter out the target, recreate the table.
    let current_columns = get_columns(pool, table_name).await?;
    let remaining_columns: Vec<&ColumnDef> = current_columns
        .iter()
        .filter(|c| c.name != column_name)
        .collect();

    if remaining_columns.len() == current_columns.len() {
        return Err(format!("Column '{}' not found in table '{}'", column_name, table_name));
    }
    if remaining_columns.is_empty() {
        return Err("Cannot remove the last column from a table".into());
    }

    let tmp_table = format!("{}_backup", table_name);
    let col_names: Vec<String> = remaining_columns.iter().map(|c| format!("\"{}\"", c.name)).collect();
    let col_defs: Vec<String> = remaining_columns
        .iter()
        .map(|c| format!("\"{}\" {}", c.name, c.col_type.to_uppercase()))
        .collect();
    let col_list = col_names.join(", ");

    // Wrap in a transaction for atomicity
    let sql = format!(
        "CREATE TABLE \"{tmp}\" ({defs}); \
         INSERT INTO \"{tmp}\" ({cols}) SELECT {cols} FROM \"{orig}\"; \
         DROP TABLE \"{orig}\"; \
         ALTER TABLE \"{tmp}\" RENAME TO \"{orig}\";",
        tmp = tmp_table,
        defs = col_defs.join(", "),
        cols = col_list,
        orig = table_name,
    );

    // Execute each statement separately since sqlx doesn't support multi-statement queries
    let create_sql = format!("CREATE TABLE \"{}\" ({})", tmp_table, col_defs.join(", "));
    let insert_sql = format!("INSERT INTO \"{}\" ({}) SELECT {} FROM \"{}\"", tmp_table, col_list, col_list, table_name);
    let drop_sql = format!("DROP TABLE \"{}\"", table_name);
    let rename_sql = format!("ALTER TABLE \"{}\" RENAME TO \"{}\"", tmp_table, table_name);

    sqlx::query(&create_sql).execute(pool).await.map_err(|e| e.to_string())?;
    sqlx::query(&insert_sql).execute(pool).await.map_err(|e| e.to_string())?;
    sqlx::query(&drop_sql).execute(pool).await.map_err(|e| e.to_string())?;
    sqlx::query(&rename_sql).execute(pool).await.map_err(|e| e.to_string())?;

    Ok(())
}
```

**Step 5: Update src/main.rs to declare modules**

```rust
mod db;
mod models;

// ... existing main function
```

**Step 6: Run tests to verify they pass**

```bash
cargo test --lib db::tests
```

Expected: All 7 tests PASS.

**Step 7: Commit**

```bash
git add src/db.rs src/models.rs src/main.rs
git commit -m "feat: add SQLite database layer with table/column CRUD"
```

---

### Task CRUISE-004: JWT Authentication and JWKS Endpoint

**Files:**
- Create: `src/auth.rs`
- Create: `scripts/generate_keys.sh`
- Modify: `src/main.rs`

**Step 1: Write the key generation script**

Create `scripts/generate_keys.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

CERT_DIR="${1:-certs}"
mkdir -p "$CERT_DIR"

# Generate RSA private key (no password for dev/test)
openssl genpkey -algorithm RSA -out "$CERT_DIR/jwt_private.pem" -pkeyopt rsa_keygen_bits:2048

# Generate a self-signed CA certificate (valid for 365 days)
# This satisfies the "local JWT CA with certificate" requirement
openssl req -new -x509 -key "$CERT_DIR/jwt_private.pem" \
  -out "$CERT_DIR/jwt_ca.crt" -days 365 \
  -subj "/CN=Local JWT CA/O=Dev/C=US"

# Extract public key from the certificate (ensures key and cert are consistent)
openssl x509 -in "$CERT_DIR/jwt_ca.crt" -pubkey -noout > "$CERT_DIR/jwt_public.pem"

echo "Keys and certificate generated in $CERT_DIR/"
echo "  Private key:  $CERT_DIR/jwt_private.pem"
echo "  Certificate:  $CERT_DIR/jwt_ca.crt"
echo "  Public key:   $CERT_DIR/jwt_public.pem (extracted from certificate)"
```

**Step 2: Write failing tests for auth module**

Add to `src/auth.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use jsonwebtoken::{encode, EncodingKey, Header, Algorithm};
    use std::time::{SystemTime, UNIX_EPOCH};

    fn test_keys() -> (EncodingKey, DecodingKey) {
        // Generate a test RSA key pair in memory using the rsa crate
        use rsa::RsaPrivateKey;
        use rsa::pkcs8::EncodePrivateKey;
        use rsa::pkcs8::EncodePublicKey;

        let mut rng = rsa::rand_core::OsRng;
        let private_key = RsaPrivateKey::new(&mut rng, 2048).unwrap();
        let private_pem = private_key.to_pkcs8_pem(rsa::pkcs8::LineEnding::LF).unwrap();
        let public_key = private_key.to_public_key();
        let public_pem = public_key.to_public_key_pem(rsa::pkcs8::LineEnding::LF).unwrap();

        let encoding = EncodingKey::from_rsa_pem(private_pem.as_bytes()).unwrap();
        let decoding = DecodingKey::from_rsa_pem(public_pem.as_bytes()).unwrap();
        (encoding, decoding)
    }

    fn make_token(encoding_key: &EncodingKey, exp_offset_secs: i64) -> String {
        let now = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs() as i64;
        let claims = Claims {
            sub: "testuser".into(),
            exp: (now + exp_offset_secs) as usize,
            iat: now as usize,
        };
        let header = Header::new(Algorithm::RS256);
        encode(&header, &claims, encoding_key).unwrap()
    }

    #[test]
    fn test_valid_token() {
        let (encoding, decoding) = test_keys();
        let token = make_token(&encoding, 300); // expires in 5 minutes
        let result = validate_token(&token, &decoding);
        assert!(result.is_ok());
        assert_eq!(result.unwrap().sub, "testuser");
    }

    #[test]
    fn test_expired_token() {
        let (encoding, decoding) = test_keys();
        let token = make_token(&encoding, -10); // expired 10 seconds ago
        let result = validate_token(&token, &decoding);
        assert!(result.is_err());
    }

    #[test]
    fn test_invalid_token() {
        let (_, decoding) = test_keys();
        let result = validate_token("not.a.valid.token", &decoding);
        assert!(result.is_err());
    }
}
```

**Step 3: Run tests to verify they fail**

```bash
cargo test --lib auth::tests
```

Expected: FAIL - types/functions not defined.

**Step 4: Implement auth.rs**

```rust
use axum::{
    extract::{Request, State},
    http::StatusCode,
    middleware::Next,
    response::{IntoResponse, Response, Json},
};
use jsonwebtoken::{decode, DecodingKey, Validation, Algorithm};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,
    pub exp: usize,
    pub iat: usize,
}

pub fn validate_token(token: &str, decoding_key: &DecodingKey) -> Result<Claims, String> {
    let mut validation = Validation::new(Algorithm::RS256);
    validation.validate_exp = true;

    decode::<Claims>(token, decoding_key, &validation)
        .map(|data| data.claims)
        .map_err(|e| format!("Token validation failed: {}", e))
}

#[derive(Clone)]
pub struct AuthState {
    pub decoding_key: DecodingKey,
}

pub async fn auth_middleware(
    State(auth): State<AuthState>,
    mut req: Request,
    next: Next,
) -> Response {
    let token = req
        .headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "));

    // Also check for token in cookie.
    // NOTE: The token cookie MUST be set with HttpOnly, Secure, and SameSite=Strict
    // flags — see build_auth_cookie() below for the canonical cookie builder.
    let cookie_token = req
        .headers()
        .get("Cookie")
        .and_then(|v| v.to_str().ok())
        .and_then(|cookies| {
            cookies.split(';')
                .find_map(|c| {
                    let c = c.trim();
                    c.strip_prefix("token=")
                })
        });

    let token = token.or(cookie_token);

    // CSRF protection: for state-changing requests (POST, PUT, DELETE) that use
    // cookie-based auth, require a custom header (X-Requested-With) to prevent
    // cross-site form submissions. Browsers will not send custom headers in
    // cross-origin requests without CORS preflight approval.
    let method = req.method().clone();
    let is_state_changing = method == "POST" || method == "PUT" || method == "DELETE";
    let has_bearer = req.headers().get("Authorization").is_some();

    if is_state_changing && !has_bearer {
        let has_csrf_header = req
            .headers()
            .get("X-Requested-With")
            .map_or(false, |v| v == "XMLHttpRequest");
        if !has_csrf_header {
            return (StatusCode::FORBIDDEN, "Missing CSRF header (X-Requested-With)").into_response();
        }
    }

    match token {
        Some(token) => match validate_token(token, &auth.decoding_key) {
            Ok(claims) => {
                req.extensions_mut().insert(claims);
                next.run(req).await
            }
            Err(_) => (StatusCode::UNAUTHORIZED, "Invalid or expired token").into_response(),
        },
        None => (StatusCode::UNAUTHORIZED, "Missing authentication token").into_response(),
    }
}

/// JWKS response structure
#[derive(Serialize)]
pub struct JwksResponse {
    pub keys: Vec<JwkKey>,
}

#[derive(Serialize)]
pub struct JwkKey {
    pub kty: String,
    pub r#use: String,
    pub kid: String,
    pub alg: String,
    pub n: String,
    pub e: String,
}

pub fn build_jwks(public_key_pem: &str) -> Result<JwksResponse, String> {
    use rsa::RsaPublicKey;
    use rsa::pkcs8::DecodePublicKey;
    use rsa::traits::PublicKeyParts;
    use base64::engine::general_purpose::URL_SAFE_NO_PAD;
    use base64::Engine;

    let public_key = RsaPublicKey::from_public_key_pem(public_key_pem)
        .map_err(|e| format!("Failed to parse public key: {}", e))?;

    let n = URL_SAFE_NO_PAD.encode(public_key.n().to_bytes_be());
    let e = URL_SAFE_NO_PAD.encode(public_key.e().to_bytes_be());

    Ok(JwksResponse {
        keys: vec![JwkKey {
            kty: "RSA".into(),
            r#use: "sig".into(),
            kid: "key-1".into(),
            alg: "RS256".into(),
            n,
            e,
        }],
    })
}

pub async fn jwks_handler(State(jwks): State<JwksResponse>) -> Json<JwksResponse> {
    Json(jwks)
}

/// Sets the JWT token cookie with security flags.
/// MUST be used by the login handler when setting the `token` cookie.
pub fn build_auth_cookie(token: &str) -> axum_extra::headers::SetCookie {
    // HttpOnly: prevents JavaScript access (mitigates XSS token theft)
    // Secure: cookie only sent over HTTPS (prevents network sniffing)
    // SameSite=Strict: cookie not sent on cross-site requests (mitigates CSRF)
    // Path=/: cookie available for all routes
    format!(
        "token={}; HttpOnly; Secure; SameSite=Strict; Path=/",
        token
    )
    .parse()
    .unwrap()
}
```

Note: `JwksResponse` needs `Clone` derive added for it to work as Axum state. Update the struct derives accordingly.

**Important — Cookie Security:** When setting the `token` cookie (in the login handler or via any response), always use `build_auth_cookie()` above, which enforces:
- **`HttpOnly`** — prevents client-side JavaScript from reading the cookie (mitigates XSS-based token theft)
- **`Secure`** — ensures the cookie is only transmitted over HTTPS
- **`SameSite=Strict`** — prevents the browser from sending the cookie with cross-site requests (mitigates CSRF attacks)

**Step 5: Run tests to verify they pass**

```bash
cargo test --lib auth::tests
```

Expected: All 3 token validation tests PASS.

**Step 5b: Add CSRF protection tests**

Add integration tests (or middleware-level tests) verifying CSRF behavior:

```rust
// Test: POST request with cookie auth but no X-Requested-With header returns 403
// Test: POST request with cookie auth and X-Requested-With: XMLHttpRequest succeeds
// Test: POST request with Bearer token auth (no X-Requested-With) succeeds (no CSRF check for API clients)
// Test: GET request with cookie auth (no X-Requested-With) succeeds (CSRF only applies to state-changing methods)
```

These tests ensure the auth_middleware correctly enforces CSRF protection for cookie-authenticated state-changing requests while allowing API clients using Bearer tokens to operate without the custom header.

**Step 6: Commit**

```bash
chmod +x scripts/generate_keys.sh
git add src/auth.rs scripts/generate_keys.sh
git commit -m "feat: add JWT auth middleware and JWKS endpoint"
```

---

### Task CRUISE-005: Table CRUD HTTP Handlers

**Files:**
- Create: `src/handlers.rs`
- Modify: `src/main.rs` (add routes)

**Step 1: Write the handler module**

Create `src/handlers.rs` with Axum handlers that:
- `GET /api/tables` - list all tables (returns HTML partial for htmx or JSON)
- `POST /api/tables` - create a table (form: table_name, columns as JSON)
- `DELETE /api/tables/:name` - drop a table
- Each handler calls the corresponding `db::` function
- Each handler returns appropriate htmx-compatible HTML fragments

**Important — CSRF Protection for State-Changing Endpoints:** All POST/DELETE handlers (create_table, delete_table, add_column, remove_column) are protected against CSRF attacks by the auth middleware layer (see CRUISE-004, Step 4). The middleware enforces two layers of defense:
1. **`SameSite=Strict` cookie flag** — prevents the browser from sending the `token` cookie on cross-site requests (set via `build_auth_cookie()`).
2. **Custom `X-Requested-With: XMLHttpRequest` header requirement** — the auth middleware rejects state-changing requests (POST/PUT/DELETE) that use cookie-based auth but lack this header (returns 403). Since browsers do not allow cross-origin requests to set custom headers without CORS preflight approval, this blocks CSRF even if `SameSite` is not supported. The htmx frontend includes this header automatically via `hx-headers` on the `<body>` tag (see CRUISE-007).

Integration tests for CSRF enforcement are in CRUISE-004, Step 5b. Handler integration tests below should also include the `X-Requested-With` header on all POST/DELETE requests to pass CSRF validation.

**Step 2: Write integration tests**

Test each handler using `axum::test` helpers or `reqwest` against a running test server:
- Test that listing tables on empty DB returns empty list
- Test creating a table via POST returns success (include `X-Requested-With: XMLHttpRequest` header)
- Test listing tables after creation shows the new table
- Test deleting a table via DELETE removes it (include `X-Requested-With: XMLHttpRequest` header)
- Test that POST/DELETE requests without `X-Requested-With` header are rejected with 403 (CSRF protection)

**Step 3: Implement handlers**

Each handler should:
1. Extract form data or path parameters
2. Call the db function
3. Return an HTML fragment (for htmx `hx-swap`)
4. Include proper error responses (4xx/5xx with error HTML)
5. If setting the `token` cookie (e.g. `post_login`), use `auth::build_auth_cookie()` to ensure `HttpOnly`, `Secure`, and `SameSite=Strict` flags are always present — never construct the `Set-Cookie` header manually

**Step 4: Wire routes into main.rs**

```rust
// In main.rs, after creating the pool and auth state:
let app = Router::new()
    .route("/.well-known/jwks.json", get(auth::jwks_handler))
    .route("/login", get(handlers::get_login).post(handlers::post_login))
    .route("/", get(handlers::index))
    .route("/api/tables", get(handlers::list_tables).post(handlers::create_table))
    .route("/api/tables/:name", delete(handlers::delete_table))
    // Protected routes use auth middleware
    .layer(middleware::from_fn_with_state(auth_state.clone(), auth::auth_middleware))
    .with_state(app_state);
```

**Security Note — Login Handler Cookie:** The `post_login` handler **must** set the `token` cookie using `auth::build_auth_cookie()`, which enforces `HttpOnly` (prevents XSS theft of the token), `Secure` (HTTPS-only transmission), and `SameSite=Strict` (CSRF mitigation). The cookie must **never** be set via client-side JavaScript or without these flags. See CRUISE-004 Step 4 and CRUISE-007 Step 2 for the canonical implementation.

**Step 5: Run tests**

```bash
cargo test
```

Expected: All tests pass.

**Step 6: Commit**

```bash
git add src/handlers.rs src/main.rs
git commit -m "feat: add table CRUD HTTP handlers"
```

---

### Task CRUISE-006: Column Modification Handlers

**Files:**
- Modify: `src/handlers.rs` (add column endpoints)
- Modify: `src/main.rs` (add column routes)

**Step 1: Write failing tests for column handlers**

- Test `GET /api/tables/:name/columns` returns column list
- Test `POST /api/tables/:name/columns` adds a column (include `X-Requested-With: XMLHttpRequest` header for CSRF)
- Test `DELETE /api/tables/:name/columns/:col` removes a column (include `X-Requested-With: XMLHttpRequest` header for CSRF)

**Step 2: Implement column handlers**

- `GET /api/tables/:name/columns` - get columns for a table
- `POST /api/tables/:name/columns` - add a column (form: col_name, col_type)
- `DELETE /api/tables/:name/columns/:col` - remove a column

Each returns an HTML fragment suitable for htmx partial swap.

**Step 3: Wire column routes**

```rust
.route("/api/tables/:name/columns", get(handlers::list_columns).post(handlers::add_column))
.route("/api/tables/:name/columns/:col", delete(handlers::remove_column))
```

**Step 4: Run tests**

```bash
cargo test
```

Expected: All tests pass.

**Step 5: Commit**

```bash
git add src/handlers.rs src/main.rs
git commit -m "feat: add column modification HTTP handlers"
```

---

### Task CRUISE-007: htmx Frontend - Table Management

**Files:**
- Create: `templates/base.html`
- Create: `templates/index.html`
- Create: `templates/login.html`
- Create: `templates/partials/table_list.html`
- Create: `templates/partials/table_row.html`
- Create: `templates/partials/create_table_form.html`
- Create: `static/css/style.css`
- Modify: `src/main.rs` (serve static files)

**Step 1: Create base template**

`templates/base.html` - includes htmx from CDN, basic page structure, nav bar. Use askama template inheritance.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SQLite Web Editor</title>
    <script src="https://unpkg.com/htmx.org@2.0.4"></script>
    <link rel="stylesheet" href="/static/css/style.css">
</head>
<body hx-headers='{"X-Requested-With": "XMLHttpRequest"}'>
    <nav>
        <h1>SQLite Web Editor</h1>
        <span id="user-info">{% block user_info %}{% endblock %}</span>
    </nav>
    <main>
        {% block content %}{% endblock %}
    </main>
</body>
</html>
```

**Step 2: Create login page**

`templates/login.html` - a page that accepts a JWT token (paste or via cookie). The login form POSTs the token to a server-side login handler, which **must** set the cookie via an HTTP response header using `auth::build_auth_cookie()` to enforce security flags (`HttpOnly`, `Secure`, `SameSite=Strict`). **Do NOT set the cookie via client-side JavaScript** — the `HttpOnly` flag means only the server can set and manage it. Example server-side response in the login handler:

```rust
use axum::response::{Redirect, IntoResponse};
use axum::http::header;

pub async fn post_login(Form(form): Form<LoginForm>) -> impl IntoResponse {
    // After validating the token...
    let cookie = auth::build_auth_cookie(&form.token);
    (
        [(header::SET_COOKIE, cookie.to_string())],
        Redirect::to("/"),
    )
}
```

This ensures the token cookie is protected against XSS (HttpOnly), network interception (Secure), and CSRF (SameSite=Strict).

**Step 3: Create index/dashboard page**

`templates/index.html` - extends base. Shows:
- Table list (loaded via htmx `hx-get="/api/tables"` on page load)
- Create table form
- Each table row has a delete button

**Step 4: Create htmx partials**

- `partials/table_list.html` - rendered by `GET /api/tables`, contains the `<tbody>` content
- `partials/table_row.html` - single table row with name + delete button
- `partials/create_table_form.html` - form for creating a table with dynamic column inputs

**Step 5: Add basic CSS**

`static/css/style.css` - minimal styling for tables, forms, buttons, nav.

**Step 6: Configure static file serving in main.rs**

```rust
use tower_http::services::ServeDir;

// Add to router:
.nest_service("/static", ServeDir::new("static"))
```

**Step 7: Verify the app works manually**

```bash
./scripts/generate_keys.sh
cargo run
# In another terminal, generate a test token and visit http://localhost:3000
```

**Step 8: Commit**

```bash
git add templates/ static/ src/main.rs
git commit -m "feat: add htmx frontend for table management"
```

---

### Task CRUISE-008: htmx Frontend - Column Management

**Files:**
- Create: `templates/table_detail.html`
- Create: `templates/partials/column_list.html`
- Create: `templates/partials/column_row.html`
- Create: `templates/partials/add_column_form.html`
- Modify: `src/handlers.rs` (add table detail page handler)
- Modify: `src/main.rs` (add route)

**Step 1: Create table detail page**

`templates/table_detail.html` - extends base. Shows:
- Table name as heading
- Column list (loaded via htmx `hx-get="/api/tables/{name}/columns"`)
- Add column form
- Each column row has a remove button
- Back link to table list

**Step 2: Create column partials**

- `partials/column_list.html` - column table body
- `partials/column_row.html` - single column row with name, type, delete button
- `partials/add_column_form.html` - form with column name + type dropdown

**Step 3: Add table detail handler and route**

```rust
// Handler
pub async fn table_detail(Path(name): Path<String>, ...) -> impl IntoResponse { ... }

// Route
.route("/tables/:name", get(handlers::table_detail))
```

**Step 4: Verify manually**

Navigate to a table detail page, add/remove columns.

**Step 5: Commit**

```bash
git add templates/ src/handlers.rs src/main.rs
git commit -m "feat: add htmx frontend for column management"
```

---

### Task CRUISE-009: Playwright E2E Tests

**Files:**
- Create: `tests/e2e/package.json`
- Create: `tests/e2e/playwright.config.ts`
- Create: `tests/e2e/tests/auth.spec.ts`
- Create: `tests/e2e/tests/tables.spec.ts`
- Create: `tests/e2e/tests/columns.spec.ts`
- Create: `tests/e2e/helpers/jwt.ts`
- Create: `tests/e2e/helpers/server.ts`

**Step 1: Initialize Playwright project**

```bash
cd tests/e2e
npm init -y
npm install -D @playwright/test
npx playwright install chromium
```

**Step 2: Create playwright.config.ts**

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  timeout: 30000,
  retries: 0,
  use: {
    baseURL: 'http://127.0.0.1:3000',
    headless: true,
  },
  reporter: [['html', { open: 'never' }], ['json', { outputFile: 'test-results.json' }]],
  webServer: {
    command: 'cargo run --release',
    cwd: '../../',
    port: 3000,
    timeout: 120000,
    reuseExistingServer: !process.env.CI,
  },
});
```

**Step 3: Create JWT helper**

`tests/e2e/helpers/jwt.ts` - generates short-lived JWT tokens for tests using `jsonwebtoken` npm package and the test private key.

```typescript
import * as jwt from 'jsonwebtoken';
import * as fs from 'fs';
import * as path from 'path';

export function generateTestToken(subject: string = 'testuser', expiresInSeconds: number = 60): string {
  const privateKeyPath = path.resolve(__dirname, '../../../certs/jwt_private.pem');
  const privateKey = fs.readFileSync(privateKeyPath, 'utf-8');

  return jwt.sign(
    { sub: subject, iat: Math.floor(Date.now() / 1000) },
    privateKey,
    { algorithm: 'RS256', expiresIn: expiresInSeconds }
  );
}
```

Install the npm jwt package:

```bash
npm install -D jsonwebtoken @types/jsonwebtoken
```

**Step 4: Write auth E2E test**

`tests/e2e/tests/auth.spec.ts`:

```typescript
import { test, expect } from '@playwright/test';
import { generateTestToken } from '../helpers/jwt';

test.describe('Authentication', () => {
  test('can access app with valid JWT token', async ({ page, context }) => {
    const token = generateTestToken('testuser', 60);
    await context.addCookies([{
      name: 'token',
      value: token,
      domain: '127.0.0.1',
      path: '/',
    }]);
    await page.goto('/');
    await expect(page.locator('h1')).toContainText('SQLite Web Editor');
  });

  test('gets rejected with expired JWT token', async ({ page, context }) => {
    const token = generateTestToken('testuser', -10); // already expired
    await context.addCookies([{
      name: 'token',
      value: token,
      domain: '127.0.0.1',
      path: '/',
    }]);
    const response = await page.goto('/');
    expect(response?.status()).toBe(401);
  });

  test('gets rejected without token', async ({ page }) => {
    const response = await page.goto('/');
    expect(response?.status()).toBe(401);
  });

  test('.well-known/jwks.json returns valid JWKS', async ({ request }) => {
    const response = await request.get('/.well-known/jwks.json');
    expect(response.status()).toBe(200);
    const body = await response.json();
    expect(body.keys).toHaveLength(1);
    expect(body.keys[0].kty).toBe('RSA');
    expect(body.keys[0].alg).toBe('RS256');
  });
});
```

**Step 5: Write table management E2E test**

`tests/e2e/tests/tables.spec.ts`:

```typescript
import { test, expect } from '@playwright/test';
import { generateTestToken } from '../helpers/jwt';

test.describe('Table Management', () => {
  test.beforeEach(async ({ context }) => {
    const token = generateTestToken('testuser', 300);
    await context.addCookies([{
      name: 'token',
      value: token,
      domain: '127.0.0.1',
      path: '/',
    }]);
  });

  test('can create a new table', async ({ page }) => {
    await page.goto('/');
    await page.fill('[name="table_name"]', 'test_users');
    await page.fill('[name="col_name_0"]', 'id');
    await page.selectOption('[name="col_type_0"]', 'INTEGER');
    await page.click('button[type="submit"]');
    await expect(page.locator('text=test_users')).toBeVisible();
  });

  test('can delete a table', async ({ page }) => {
    await page.goto('/');
    // Assumes test_users exists from previous state or setup
    // Create it first
    await page.fill('[name="table_name"]', 'to_delete');
    await page.fill('[name="col_name_0"]', 'id');
    await page.selectOption('[name="col_type_0"]', 'INTEGER');
    await page.click('button[type="submit"]');
    await expect(page.locator('text=to_delete')).toBeVisible();

    // Delete it
    await page.click('[data-table="to_delete"] button.delete-btn');
    await expect(page.locator('text=to_delete')).not.toBeVisible();
  });
});
```

**Step 6: Write column management E2E test**

`tests/e2e/tests/columns.spec.ts`:

```typescript
import { test, expect } from '@playwright/test';
import { generateTestToken } from '../helpers/jwt';

test.describe('Column Management', () => {
  test.beforeEach(async ({ context, page }) => {
    const token = generateTestToken('testuser', 300);
    await context.addCookies([{
      name: 'token',
      value: token,
      domain: '127.0.0.1',
      path: '/',
    }]);
  });

  test('can add a column to an existing table', async ({ page }) => {
    await page.goto('/');
    // Create a table first
    await page.fill('[name="table_name"]', 'col_test');
    await page.fill('[name="col_name_0"]', 'id');
    await page.selectOption('[name="col_type_0"]', 'INTEGER');
    await page.click('button[type="submit"]');

    // Navigate to table detail
    await page.click('text=col_test');

    // Add a column
    await page.fill('[name="col_name"]', 'email');
    await page.selectOption('[name="col_type"]', 'TEXT');
    await page.click('button.add-column-btn');

    await expect(page.locator('text=email')).toBeVisible();
    await expect(page.locator('text=TEXT')).toBeVisible();
  });

  test('can remove a column from an existing table', async ({ page }) => {
    await page.goto('/');
    // Create a table with multiple columns
    await page.fill('[name="table_name"]', 'col_remove_test');
    await page.fill('[name="col_name_0"]', 'id');
    await page.selectOption('[name="col_type_0"]', 'INTEGER');
    // Add second column via the add-column-to-form button
    await page.click('button.add-col-field');
    await page.fill('[name="col_name_1"]', 'temp_col');
    await page.selectOption('[name="col_type_1"]', 'TEXT');
    await page.click('button[type="submit"]');

    // Navigate to table detail
    await page.click('text=col_remove_test');

    // Remove the temp_col column
    await page.click('[data-column="temp_col"] button.delete-col-btn');
    await expect(page.locator('text=temp_col')).not.toBeVisible();
  });
});
```

**Step 7: Run E2E tests locally**

```bash
# Generate keys first
./scripts/generate_keys.sh

# Run tests
cd tests/e2e
npx playwright test
```

Expected: All tests pass.

**Step 8: Commit**

```bash
git add tests/e2e/
git commit -m "feat: add Playwright E2E tests for auth, tables, and columns"
```

---

### Task CRUISE-010: GitHub Actions CI/CD

**Files:**
- Create: `.github/workflows/ci.yml`
- Create: `.github/workflows/e2e.yml`

**Step 1: Create the main CI workflow**

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: read
      statuses: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Super-Linter
        uses: super-linter/super-linter@v7
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          DEFAULT_BRANCH: main
          VALIDATE_RUST_2021: true
          VALIDATE_HTML: true
          VALIDATE_CSS: true
          VALIDATE_YAML: true
          VALIDATE_MARKDOWN: true
          VALIDATE_GITHUB_ACTIONS: true
          FILTER_REGEX_EXCLUDE: "(node_modules|target|playwright-report)/"

  dependency-review:
    name: Dependency Review
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - name: Dependency Review
        uses: actions/dependency-review-action@v4

  build:
    name: Build & Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable

      - name: Cache cargo registry and build
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

      - name: Build
        run: cargo build --release

      - name: Run unit tests
        run: cargo test --lib

      - name: Upload binary
        uses: actions/upload-artifact@v4
        with:
          name: sqlite-web-editor
          path: target/release/sqlite-web-editor
```

**Step 2: Create the E2E test workflow**

`.github/workflows/e2e.yml`:

```yaml
name: E2E Tests

on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  e2e:
    name: Playwright E2E Tests
    runs-on: ubuntu-latest
    needs: []
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable

      - name: Cache cargo
        uses: actions/cache@v4
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

      - name: Build release binary
        run: cargo build --release

      - name: Generate test JWT keys
        run: |
          chmod +x scripts/generate_keys.sh
          ./scripts/generate_keys.sh

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 'lts/*'

      - name: Install Playwright dependencies
        working-directory: tests/e2e
        run: |
          npm ci
          npx playwright install --with-deps chromium

      - name: Run Playwright tests
        working-directory: tests/e2e
        run: npx playwright test
        env:
          CI: true

      - name: Upload Playwright report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: tests/e2e/playwright-report/
          retention-days: 30

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-test-results
          path: tests/e2e/test-results.json
          retention-days: 30
```

**Step 3: Verify workflow YAML is valid**

```bash
# Install actionlint if available, or use yamllint
yamllint .github/workflows/ci.yml .github/workflows/e2e.yml
```

**Step 4: Commit**

```bash
git add .github/
git commit -m "feat: add GitHub Actions workflows for CI and E2E tests"
```

---

## Spawn Instance Configuration

```json
{
  "title": "SQLite Web Editor Implementation",
  "overview": "Build a Rust/Axum web application with htmx frontend for editing SQLite databases, JWT authentication with JWKS endpoint, Playwright E2E tests, and GitHub Actions CI/CD.",
  "spawn_instances": [
    {
      "id": "SPAWN-001",
      "name": "Project Foundation and .gitignore",
      "use_spawn_team": false,
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep --timeout 180",
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "task_ids": ["CRUISE-001"]
    },
    {
      "id": "SPAWN-002",
      "name": "Core Backend Infrastructure",
      "use_spawn_team": true,
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep --timeout 600",
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "task_ids": ["CRUISE-002", "CRUISE-003"]
    },
    {
      "id": "SPAWN-003",
      "name": "Authentication and Security",
      "use_spawn_team": true,
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep --timeout 600",
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "task_ids": ["CRUISE-004"]
    },
    {
      "id": "SPAWN-004",
      "name": "HTTP Handlers",
      "use_spawn_team": true,
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep --timeout 600",
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "task_ids": ["CRUISE-005", "CRUISE-006"]
    },
    {
      "id": "SPAWN-005",
      "name": "Frontend Templates and htmx",
      "use_spawn_team": false,
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep --timeout 480",
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "task_ids": ["CRUISE-007", "CRUISE-008"]
    },
    {
      "id": "SPAWN-006",
      "name": "E2E Testing with Playwright",
      "use_spawn_team": true,
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep --timeout 600",
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "task_ids": ["CRUISE-009"]
    },
    {
      "id": "SPAWN-007",
      "name": "CI/CD Pipeline",
      "use_spawn_team": false,
      "cli_params": "claude --model haiku --allowedTools Read,Write,Edit,Glob,Grep --timeout 180",
      "permissions": ["Read", "Write", "Edit", "Glob", "Grep"],
      "task_ids": ["CRUISE-010"]
    }
  ],
  "tasks": [
    {
      "id": "CRUISE-001",
      "subject": "Project skeleton and .gitignore",
      "description": "Create comprehensive .gitignore excluding keys, credentials, temp files, build artifacts, node_modules, editor/IDE files, OS files, .env files, log files, .fork-join directories. Create minimal src/main.rs placeholder and directory structure (templates/, static/, tests/e2e/, certs/, scripts/).",
      "blocked_by": [],
      "complexity": "low",
      "acceptance_criteria": [
        ".gitignore covers: Rust target/, *.pem, *.key, *.crt, certs/, .env*, *.log, .DS_Store, Thumbs.db, node_modules/, playwright-report/, .vscode/, .idea/, .fork-join/, *.db, *.sqlite",
        "src/main.rs exists with placeholder",
        "Directory structure created: templates/partials/, static/css/, tests/e2e/, certs/, scripts/",
        "git commit succeeds"
      ],
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep",
      "spawn_instance": "SPAWN-001"
    },
    {
      "id": "CRUISE-002",
      "subject": "Cargo.toml and dependency setup",
      "description": "Create Cargo.toml with all required dependencies: axum 0.8 (macros), axum-extra 0.10 (cookie), askama 0.12, askama_axum 0.4, tokio 1 (full), sqlx 0.8 (runtime-tokio, sqlite), jsonwebtoken 9, serde 1 (derive), serde_json 1, tower 0.5, tower-http 0.6 (fs, cors), base64 0.22, rsa 0.9 (pem), tracing 0.1, tracing-subscriber 0.3. Dev-deps: reqwest 0.12, tokio-test 0.4. Update src/main.rs to import axum Router and verify cargo check passes.",
      "blocked_by": ["CRUISE-001"],
      "complexity": "low",
      "acceptance_criteria": [
        "Cargo.toml has all listed dependencies with correct versions and features",
        "cargo check succeeds with no errors",
        "src/main.rs creates an empty Axum Router and binds to 127.0.0.1:3000"
      ],
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep",
      "spawn_instance": "SPAWN-002"
    },
    {
      "id": "CRUISE-003",
      "subject": "SQLite database layer",
      "description": "Create src/db.rs with functions: list_tables, create_table, drop_table, get_columns, add_column, remove_column. Create src/models.rs with TableInfo and ColumnDef structs. All functions validate identifiers (alphanumeric + underscore only, max 128 chars) and column types (INTEGER, TEXT, REAL, BLOB, NUMERIC only) to prevent SQL injection. Use sqlx::SqlitePool. Write comprehensive unit tests using in-memory SQLite including SQL injection rejection test.",
      "blocked_by": ["CRUISE-002"],
      "complexity": "medium",
      "acceptance_criteria": [
        "list_tables returns empty vec for fresh DB",
        "create_table creates table and list_tables shows it",
        "drop_table removes table",
        "get_columns returns correct column names and types",
        "add_column adds column to existing table",
        "remove_column removes column (uses table recreation pattern for broad SQLite version compatibility)",
        "SQL injection in table/column names is rejected",
        "All 7+ unit tests pass with cargo test --lib db::tests"
      ],
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep",
      "spawn_instance": "SPAWN-002"
    },
    {
      "id": "CRUISE-004",
      "subject": "JWT authentication and JWKS endpoint",
      "description": "Create src/auth.rs with: Claims struct (sub, exp, iat), validate_token function using RS256, auth_middleware for Axum (checks Authorization header and cookie, with CSRF protection via X-Requested-With header for state-changing requests using cookie auth), build_jwks function that reads RSA public key PEM and returns JWKS JSON, jwks_handler for GET /.well-known/jwks.json. Create scripts/generate_keys.sh that generates RSA 2048 key pair with a self-signed CA certificate in certs/ directory (jwt_private.pem, jwt_ca.crt, and jwt_public.pem extracted from the certificate). Write unit tests for valid token, expired token, invalid token validation, and CSRF header enforcement. Set token cookie with SameSite=Strict, HttpOnly, and Secure attributes.",
      "blocked_by": ["CRUISE-002"],
      "complexity": "high",
      "acceptance_criteria": [
        "validate_token accepts valid RS256 JWT and returns Claims",
        "validate_token rejects expired tokens",
        "validate_token rejects malformed tokens",
        "auth_middleware extracts token from Authorization: Bearer header",
        "auth_middleware extracts token from cookie named 'token'",
        "auth_middleware returns 401 for missing/invalid token",
        "auth_middleware returns 403 for state-changing requests (POST/PUT/DELETE) via cookie auth without X-Requested-With header (CSRF protection)",
        "auth_middleware allows state-changing requests with Bearer token without CSRF header",
        "token cookie is set with SameSite=Strict, HttpOnly, and Secure attributes",
        "build_jwks returns valid JWKS with RSA key components (n, e)",
        "scripts/generate_keys.sh generates jwt_private.pem, jwt_ca.crt (self-signed CA certificate), and jwt_public.pem (extracted from certificate) in certs/",
        "GET /.well-known/jwks.json returns valid JWKS response (not behind auth)",
        "All unit tests pass"
      ],
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep",
      "spawn_instance": "SPAWN-003"
    },
    {
      "id": "CRUISE-005",
      "subject": "Table CRUD HTTP handlers",
      "description": "Create src/handlers.rs with Axum handlers: index (GET / - full page), list_tables (GET /api/tables - htmx partial), create_table (POST /api/tables - form data), delete_table (DELETE /api/tables/:name). Each handler uses db module functions. Protected routes require auth. Handlers return HTML fragments for htmx swap. Wire routes into main.rs with auth middleware layer.",
      "blocked_by": ["CRUISE-003", "CRUISE-004"],
      "complexity": "medium",
      "acceptance_criteria": [
        "GET / returns full HTML page (behind auth)",
        "GET /api/tables returns HTML table rows partial",
        "POST /api/tables creates table from form data and returns updated table list",
        "DELETE /api/tables/:name drops table and returns updated table list",
        "All endpoints return proper HTTP status codes (200, 201, 404, 500)",
        "Routes are protected by auth middleware (except /.well-known/jwks.json)",
        "POST/DELETE requests without X-Requested-With header are rejected with 403 (CSRF protection via auth middleware)",
        "Integration tests include X-Requested-With: XMLHttpRequest header on state-changing requests",
        "Integration tests pass"
      ],
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep",
      "spawn_instance": "SPAWN-004"
    },
    {
      "id": "CRUISE-006",
      "subject": "Column modification HTTP handlers",
      "description": "Add to src/handlers.rs: list_columns (GET /api/tables/:name/columns), add_column (POST /api/tables/:name/columns), remove_column (DELETE /api/tables/:name/columns/:col), table_detail (GET /tables/:name - full page). Wire column routes into main.rs. Each returns htmx-compatible HTML fragments.",
      "blocked_by": ["CRUISE-003", "CRUISE-004"],
      "complexity": "medium",
      "acceptance_criteria": [
        "GET /api/tables/:name/columns returns column list HTML partial",
        "POST /api/tables/:name/columns adds column and returns updated column list",
        "DELETE /api/tables/:name/columns/:col removes column and returns updated list",
        "GET /tables/:name returns full table detail page",
        "Returns 404 if table doesn't exist",
        "All endpoints protected by auth",
        "POST/DELETE requests without X-Requested-With header are rejected with 403 (CSRF protection via auth middleware)",
        "Integration tests pass"
      ],
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep",
      "spawn_instance": "SPAWN-004"
    },
    {
      "id": "CRUISE-007",
      "subject": "htmx frontend - table management UI",
      "description": "Create askama templates: base.html (includes htmx 2.0.4 CDN, CSS link, nav), login.html (JWT token input form), index.html (extends base, table list with hx-get, create table form, delete buttons). Create partials: table_list.html, table_row.html, create_table_form.html. Create static/css/style.css with minimal styling. Configure static file serving via tower-http ServeDir in main.rs.",
      "blocked_by": ["CRUISE-005"],
      "complexity": "medium",
      "acceptance_criteria": [
        "base.html includes htmx script tag and CSS link",
        "base.html body tag includes hx-headers with X-Requested-With: XMLHttpRequest for CSRF protection",
        "index.html shows table list loaded via htmx hx-get on page load",
        "Create table form submits via hx-post and updates table list without page reload",
        "Delete button uses hx-delete and removes table row without page reload",
        "login.html sets token cookie with SameSite=Strict, HttpOnly, and Secure attributes",
        "Static files served at /static/ path",
        "CSS provides readable, functional (not necessarily beautiful) styling",
        "App works end-to-end in browser manually"
      ],
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep",
      "spawn_instance": "SPAWN-005"
    },
    {
      "id": "CRUISE-008",
      "subject": "htmx frontend - column management UI",
      "description": "Create templates: table_detail.html (extends base, column list with hx-get, add column form, remove buttons, back link). Create partials: column_list.html, column_row.html, add_column_form.html. Column type uses dropdown with allowed types (INTEGER, TEXT, REAL, BLOB, NUMERIC). Wire table_detail handler into routes.",
      "blocked_by": ["CRUISE-006", "CRUISE-007"],
      "complexity": "medium",
      "acceptance_criteria": [
        "Table detail page shows table name and column list",
        "Column list loaded via htmx hx-get",
        "Add column form with name input and type dropdown submits via hx-post",
        "Remove column button uses hx-delete",
        "Column type dropdown shows only allowed types",
        "Back link returns to table list",
        "All htmx interactions work without full page reload"
      ],
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep",
      "spawn_instance": "SPAWN-005"
    },
    {
      "id": "CRUISE-009",
      "subject": "Playwright E2E tests",
      "description": "Create Node.js Playwright test suite in tests/e2e/. Install @playwright/test and jsonwebtoken. Create playwright.config.ts with webServer pointing to cargo run. Create JWT helper that generates tokens from certs/jwt_private.pem. Write tests: auth.spec.ts (valid token access, expired token rejection, missing token rejection, JWKS endpoint validation), tables.spec.ts (create table, delete table), columns.spec.ts (add column, remove column). Configure HTML and JSON reporters.",
      "blocked_by": ["CRUISE-004", "CRUISE-007", "CRUISE-008"],
      "complexity": "high",
      "acceptance_criteria": [
        "npm ci installs all dependencies",
        "Auth test: valid JWT grants access to app",
        "Auth test: expired JWT returns 401",
        "Auth test: missing JWT returns 401",
        "Auth test: /.well-known/jwks.json returns valid JWKS",
        "Table test: can create a new table via the UI",
        "Table test: can delete a table via the UI",
        "Column test: can add a column to existing table",
        "Column test: can remove a column from existing table",
        "HTML report generated in playwright-report/",
        "JSON results generated as test-results.json",
        "All tests pass with npx playwright test"
      ],
      "permissions": ["Read", "Write", "Edit", "Bash", "Glob", "Grep"],
      "cli_params": "claude --model sonnet --allowedTools Read,Write,Edit,Bash,Glob,Grep",
      "spawn_instance": "SPAWN-006"
    },
    {
      "id": "CRUISE-010",
      "subject": "GitHub Actions CI/CD workflows",
      "description": "Create .github/workflows/ci.yml with jobs: lint (super-linter v7 for Rust, HTML, CSS, YAML, Markdown, GitHub Actions), dependency-review (dependency-review-action v4, PR only), build (Rust toolchain, cargo cache, cargo build --release, cargo test --lib, upload binary artifact). Create .github/workflows/e2e.yml with job: e2e (Rust build, generate test keys, Node.js setup, Playwright install, run tests, upload playwright-report and test-results.json as artifacts with retention 30 days). Both triggered on pull_request to main.",
      "blocked_by": ["CRUISE-001"],
      "complexity": "medium",
      "acceptance_criteria": [
        "ci.yml triggers on pull_request to main",
        "Lint job uses super-linter/super-linter@v7 with correct linter env vars",
        "Dependency review job uses actions/dependency-review-action@v4",
        "Build job compiles Rust and runs unit tests",
        "e2e.yml triggers on pull_request to main",
        "E2E job generates ephemeral JWT keys for testing",
        "E2E job installs Playwright with chromium",
        "E2E job uploads playwright-report/ as artifact (if: always())",
        "E2E job uploads test-results.json as artifact (if: always())",
        "Workflow YAML passes yamllint validation"
      ],
      "permissions": ["Read", "Write", "Edit", "Glob", "Grep"],
      "cli_params": "claude --model haiku --allowedTools Read,Write,Edit,Glob,Grep",
      "spawn_instance": "SPAWN-007"
    }
  ],
  "risks": [
    "SQLite ALTER TABLE DROP COLUMN requires SQLite 3.35.0+; the remove_column implementation uses the table recreation pattern (create backup, copy data, drop original, rename) to ensure compatibility with all SQLite versions",
    "Playwright webServer config starts cargo run which compiles from source in CI - this could timeout; may need to use pre-built binary instead",
    "JWT key generation in CI creates ephemeral keys - tests must not depend on specific key material",
    "Super-linter may flag htmx attributes (hx-get, hx-post, etc.) as invalid HTML attributes; may need to configure HTML linter exceptions",
    "askama template compilation happens at Rust compile time - template syntax errors show as Rust compile errors which can be confusing",
    "Concurrent SQLite writes in tests could cause 'database is locked' errors if WAL mode is not enabled",
    "htmx partial responses must set correct Content-Type header (text/html) or htmx may not swap properly",
    "Cookie-based JWT in E2E tests needs correct domain/path matching - 127.0.0.1 vs localhost can cause issues",
    "CSRF protection relies on X-Requested-With custom header and SameSite=Strict cookie; htmx must include hx-headers on body element to send this header with all requests"
  ]
}
```

---

## Execution Notes

**Parallel execution opportunities:**
- CRUISE-003 and CRUISE-004 can run in parallel (both depend only on CRUISE-002)
- CRUISE-005 and CRUISE-006 can run in parallel (both depend on CRUISE-003 + CRUISE-004)
- CRUISE-010 can run in parallel with CRUISE-002 through CRUISE-009 (only depends on CRUISE-001)

**Key files to reference:**
- `src/main.rs` - application entry point, route wiring, state setup
- `src/db.rs` - all SQLite operations
- `src/auth.rs` - JWT validation, middleware, JWKS
- `src/handlers.rs` - HTTP request handlers
- `src/models.rs` - shared data structures
- `templates/` - askama HTML templates
- `tests/e2e/` - Playwright test suite
- `.github/workflows/` - CI/CD pipelines

**Testing strategy:**
- Unit tests: `cargo test --lib` (db and auth modules)
- Integration tests: `cargo test` (handler tests with test server)
- E2E tests: `cd tests/e2e && npx playwright test` (full browser tests)
