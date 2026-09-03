# 5. Building Block View

C4 Container + Component, Logical View, các core service, mô hình dữ liệu, thiết kế DB, ranh
giới service, và Development View — tách thành 5 file dưới `05-building-blocks/` (2026-09-03, vì
file gộp trước đó đã 694 dòng, dài gấp ~4 lần file thứ nhì trong `docs/architectures/`; cùng lý do
+ cùng cách làm `docs/roadmap.md` từng được tách thành `docs/roadmap/*.md` ngày 2026-08-24). Trang
này chỉ còn là mục lục — nội dung thật nằm ở từng file con.

| Mục | Nội dung |
|---|---|
| [5.1 C4 Diagrams (Containers & Components)](../05-building-blocks/01-c4-diagrams.md) | Các layer cấp cao, C4 Level 2 (Containers), C4 Level 3 (Components bên trong API Server) |
| [5.2 Logical View (class-level)](../05-building-blocks/02-logical-view.md) | Các type/class và quan hệ phụ thuộc đứng sau component diagram ở 5.1 |
| [5.3 Whitebox: Core Services](../05-building-blocks/03-core-services.md) | Metadata Registry, Control Plane (Router/Multi-Tenancy), CRUD Service, Permission Service, Query Planner, Workflow Functions, Outbox and EventBus |
| [5.4 Data Model & Database Design](../05-building-blocks/04-data-model.md) | Bảng `records` tổng quát, lộ trình dedicated-table, Database Design (ER diagram) |
| [5.5 Service Boundaries & Development View](../05-building-blocks/05-service-boundaries-and-dev-view.md) | Quy tắc phụ thuộc giữa layer, sơ đồ workspace (Cargo + pnpm) |
