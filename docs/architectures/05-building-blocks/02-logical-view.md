# 5.2 Logical View (class-level)

[← 5. Building Block View](../05-building-blocks.md)

Mô hình object đứng sau component diagram ở [5.1](01-c4-diagrams.md) — các type và cách chúng phụ thuộc lẫn nhau, không phải các đơn vị deploy. (Logical View của Kruchten 4+1.) `metap-query`/`metap-workflow` là các function module chứ không phải struct (không có state cần giữ qua từng call), được thể hiện ở đây như pseudo-class để nhất quán với phần còn lại của diagram.

```mermaid
classDiagram
  class AppState {
    +router: Router
    +metadata: Arc~ArcSwap~MetadataRegistry~~
    +permissions: Arc~PermissionService~
    +decoding_key: DecodingKey
  }
  class Router {
    <<metap-control>>
    +begin(tenant_id) Transaction
  }
  class MetadataRegistry {
    -entities: HashMap~String, EntityDefinition~
    +register(entity)
    +get_entity(name) EntityDefinition
    +list_entities() Vec~EntitySummary~
    +validate_references()
  }
  class EntityDefinition {
    +name: String
    +fields: Vec~EntityField~
    +list_views: Vec~EntityListView~
    +workflow: Option~EntityWorkflow~
  }
  class CrudService {
    -router: Router
    -metadata: Arc~ArcSwap~MetadataRegistry~~
    -permissions: Arc~PermissionService~
    +list(entity, input, context)
    +get(entity, id, context)
    +create(entity, data, context)
    +update(entity, id, version, data, context)
    +transition(entity, id, action, version, payload, context)
    +delete(entity, id, version, context)
    -enrich_record_for_actions(entity, snapshot, actions, tenant_id, record) JsonObject
  }
  class PermissionService {
    +can_read_entity(context, entity)
    +can_create_entity(context, entity)
    +can_update_entity(context, entity)
    +can_transition_entity(context, entity)
    +can_delete_entity(context, entity)
    +load_snapshot(tenant_id, entity) PermissionSnapshot
    +scoped_tenant(context)
  }
  class PermissionSnapshot {
    +filter_readable_fields(context, data)
    +assert_writable_fields(context, fields, existing)
    +can_perform_record_condition(context, record, action) PermissionDecision
    +get_record_policies(action) Vec~PolicyRow~
    +required_relation_fields(action) Vec~String~
  }
  class PolicyEffect {
    <<enum>>
    Allow
    Deny
  }
  class PolicyVerdict {
    <<enum>>
    Allow
    Deny
    NoMatch
  }
  class QueryPlannerFns {
    <<module: metap-query>>
    +plan_list(entity, input, context, policies) PlannedListQuery
  }
  class WorkflowFns {
    <<module: metap-workflow>>
    +get_initial_status(entity, data)
    +find_transition(entity, action, from_state)
    +run_guard(transition, data, context)
    +run_validator(transition, next_data, context)
    +apply_set_fields(transition, data, context)
  }
  class OutboxFns {
    <<module: metap-infra::outbox>>
    +enqueue(executor, event)
  }
  class EventBus {
    <<trait>>
    +publish(topic, payload)
    +subscribe(topic, handler)
  }
  class RabbitEventBus {
    +publish(topic, payload)
    +subscribe(topic, handler)
  }
  class IndexReconciler {
    <<module: metap-peripherals>>
    +reconcile_indexes(pool, entities)
  }
  class MetadataDriftService {
    <<module: metap-peripherals>>
    +check_metadata_drift(pool, entities)
  }

  AppState --> Router
  AppState --> MetadataRegistry
  AppState --> PermissionService
  MetadataRegistry --> EntityDefinition : holds
  CrudService --> Router
  CrudService --> MetadataRegistry
  CrudService --> PermissionService
  CrudService --> QueryPlannerFns
  CrudService --> WorkflowFns
  CrudService --> OutboxFns
  PermissionService --> PermissionSnapshot : creates per call
  PermissionSnapshot ..> PolicyEffect : evaluate_policies decides via
  PermissionSnapshot ..> PolicyVerdict : evaluate_policies decides via
  QueryPlannerFns --> PermissionService
  IndexReconciler --> MetadataRegistry
  MetadataDriftService --> MetadataRegistry
  EventBus <|.. RabbitEventBus : implements
  OutboxFns ..> EventBus : drained by outbox-publisher, publishes through
```

