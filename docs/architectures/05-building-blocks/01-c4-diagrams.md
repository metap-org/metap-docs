# 5.1 C4 Diagrams (Containers & Components)

[← 5. Building Block View](00-index.md)

## Các layer cấp cao

```txt
axum routes (crates/metap-http/src/routes/*)
  -> application service (crates/metap-crud/src/crud_service.rs)
    -> platform core (metap-metadata / metap-permission / metap-query / metap-workflow)
      -> PostgreSQL (sqlx::PgPool, injected directly — no repository abstraction; see
         docs/architectures/09-adr.md for why)
      -> outbox (metap-infra::outbox::enqueue) -> RabbitMQ (metap-infra::EventBus)
```

## C4 Level 2: Containers

```mermaid
C4Container
  title Container diagram — Metap

  Person(user, "Người dùng cuối")
  Person(admin, "Admin")
  Person(platformadmin, "Platform Admin", "Superadmin xuyên tenant — tenant sentinel PLATFORM_TENANT_ID + role platform_admin")

  System_Boundary(metap, "Metap") {
    Container(web, "Web Frontend", "React, Vite, TanStack Query", "Dev harness SPA — ../metap-demo-crm/web, dùng ../platform-ui (@metap/platform-ui) qua workspace:*")
    Container(api, "API Server", "Rust, axum", "../metap-demo-crm: một process, router gộp 3 crate — metap-http (/api, /auth, /admin, /metadata), metap-lowcode-http (/admin/lowcode), metap-control-http (/platform/tenants)")
    Container(api2, "API Server (Jira)", "Rust, axum", "../metap-demo-jira, Phase 21+: module nghiệp vụ thứ hai, process/port riêng (3100), entity trên bảng riêng (table-per-entity) thay vì records chung, tenant dedicated_db riêng")
    Container(outboxworker, "Outbox Publisher", "Rust", "crates/outbox-publisher, binary riêng — outbox drain/publish loop của metap-infra")
    Container(cronworker, "Cron Scheduler", "Rust", "crates/cron-scheduler, binary riêng — ticker (poll cron_jobs) + executor (workflow_transition/bulk_query_action gọi lại API Server, webhook gọi ngoài)")
    Container(notifworker, "Notification Worker", "Rust", "crates/notification-worker, binary riêng theo mặc định — hoặc chạy inline trong API Server khi NOTIFICATION_WORKER_INLINE=true; subscribe EventBus, log mọi workflow transition")
  }

  ContainerDb(db, "PostgreSQL", "Postgres 16", "records, control.tenants, policies, outbox_events, workflow_events, user_roles, cron_jobs, low_code_*")
  ContainerQueue(mq, "RabbitMQ", "AMQP 0-9-1", "Reliable event delivery đến các downstream consumer")
  System_Ext(vault, "Vault", "HashiCorp Vault — secret KV v2, resolve DSN cho tenant dedicated_db (tùy chọn, EnvStore là fallback không có Vault)")

  Rel(user, web, "Sử dụng", "HTTPS")
  Rel(admin, web, "Sử dụng", "HTTPS")
  Rel(platformadmin, api, "Provision/suspend/xóa tenant", "REST/JSON, Bearer JWT, PlatformAdminContext")
  Rel(web, api, "Gọi", "REST/JSON, Bearer JWT")
  Rel(api, db, "Đọc/ghi records, metadata, policies, control.tenants; ghi outbox rows trong cùng transaction với business write — mọi transaction tenant-scoped đi qua metap-control::Router", "sqlx/SQL")
  Rel(api, vault, "Resolve DSN cho tenant dedicated_db (VaultStore, token hoặc AppRole)", "HTTPS")
  Rel(api2, db, "Đọc/ghi entity trên bảng riêng (entities schema) qua tenant dedicated_db riêng", "sqlx/SQL")
  Rel(outboxworker, db, "Poll các outbox row đang pending", "SQL, ~1s loop, FOR UPDATE SKIP LOCKED")
  Rel(outboxworker, mq, "Publish", "AMQP")
  Rel(cronworker, db, "Poll cron_jobs đến hạn (FOR UPDATE SKIP LOCKED), ghi cron_job_runs", "SQL")
  Rel(cronworker, api, "Gọi lại /api/:entity/... với service JWT (workflow_transition/bulk_query_action)", "REST/JSON")
  Rel(notifworker, mq, "Subscribe #.workflow.transitioned", "AMQP")
```

API Server, Outbox Publisher, Cron Scheduler, và Notification Worker (khi không chạy inline) là các process tách biệt một cách có chủ ý — khi RabbitMQ gặp sự cố, chỉ các worker bị ngưng trệ, API không bị ảnh hưởng, vì transactional outbox write đã commit xong rồi. `../metap-demo-crm` có thể tùy chọn phục vụ luôn static files đã build của `../metap-demo-crm/web` trên cùng process/port (`pnpm start`, cấu hình `STATIC_DIR`) — đây chỉ là một tiện lợi khi triển khai, không làm thay đổi sự tách biệt này; các worker vẫn luôn là process riêng biệt (Notification Worker là ngoại lệ duy nhất, có thể chạy inline như một background task trong chính API Server qua `NOTIFICATION_WORKER_INLINE=true` — cả hai chế độ gọi chung một hàm `notification_worker::run`, nên không thể lệch hành vi nhau).

## C4 Level 3: Components (inside the API Server)

```mermaid
C4Component
  title Component diagram — API Server

  Container_Boundary(api, "API Server") {
    Component(routes, "HTTP Routes", "axum handlers", "records / metadata / admin / health — crates/metap-http/src/routes")
    Component(crud, "CrudService", "Rust struct", "permission -> validate -> plan -> write -> workflow -> outbox")
    Component(router, "Router", "Rust struct", "metap-control — mở transaction tenant-scoped: SET LOCAL search_path (Schema) hoặc pool riêng theo dsn_secret_ref (DedicatedDb)")
    Component(tenantreg, "TenantRegistry + RegistryCache", "Rust struct", "metap-control — đọc control.tenants, cache TTL 30s (moka)")
    Component(secretstore, "SecretStore", "trait", "metap-control — EnvStore hoặc VaultStore, resolve DSN cho DedicatedDb")
    Component(metadata, "MetadataRegistry", "Rust struct", "Entity definitions; được validate + hash lúc boot (MetadataCompiler); ArcSwap để publish/rollback hot-reload không cần restart")
    Component(perm, "PermissionService", "Rust struct", "RBAC/ABAC, field/record/cross-record enforcement, deny-overrides-allow, PolicyExplainer")
    Component(query, "QueryPlanner", "Rust functions", "Metadata-constrained filter/sort/cursor -> SQL (plan_list)")
    Component(workflow, "Workflow functions", "Rust functions", "State machine transitions + audit log (metap-workflow)")
    Component(outbox, "Outbox", "Rust functions", "Transactional outbox writes (metap-infra::outbox::enqueue)")
    Component(idxr, "IndexReconciler", "Rust functions", "Reconcile indexes từ metadata lúc boot (metap-peripherals)")
    Component(drift, "MetadataDriftService", "Rust functions", "Cảnh báo khi metadata hash drift qua các lần restart (metap-peripherals)")
  }

  ContainerDb(db, "PostgreSQL", "", "")
  System_Ext(vault, "Vault", "")

  Rel(routes, crud, "Gọi")
  Rel(crud, metadata, "Đọc entity definitions")
  Rel(crud, perm, "Kiểm tra permission, load PermissionSnapshot")
  Rel(crud, query, "Lập kế hoạch list query")
  Rel(crud, workflow, "Gán initial status / chạy transitions")
  Rel(crud, outbox, "Enqueue events (cùng DB transaction)")
  Rel(crud, router, "Router::begin(tenant_id) mở mọi transaction thay vì bare PgPool")
  Rel(query, perm, "AND record-level policy WHERE clause")
  Rel(perm, crud, "required_relation_fields(action) — CrudService fetch record liên quan 1-hop, merge vào subject trước khi evaluate")
  Rel(router, tenantreg, "Tra strategy/status của tenant")
  Rel(router, secretstore, "Resolve DSN khi strategy=DedicatedDb")
  Rel(secretstore, vault, "KV v2 read (VaultStore, tùy chọn)")
  Rel(idxr, metadata, "Đọc các flag indexed / unique / searchMode")
  Rel(drift, metadata, "Đọc entity hash (version)")
  Rel(router, db, "SET LOCAL search_path / pool riêng theo dsn_secret_ref", "sqlx")
  Rel(idxr, db, "CREATE INDEX CONCURRENTLY", "DDL, best-effort")
```

