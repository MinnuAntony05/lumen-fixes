Let me search both repositories to trace where these clients and services are defined or called.Here's a clear breakdown of **how `edgeops-insight-module` depends on `edgeops-core-lib`**, mapped directly to each external interaction you listed:

---

## 🔗 Dependency Map: insight-module → core-lib

### 1. `http_call` — `FeignClient` (source: `feign_scan_package`, package `com.ustri.digital.edgeops`)

The `InsightApplication` uses `@EnableFeignClients(basePackages = "com.ustri.digital.edgeops")` — this means it **scans both its own package AND core-lib's package** for Feign clients.

**Core-lib provides these Feign clients that get picked up:**

| Feign Client | File in core-lib | Target Service |
|---|---|---|
| `UserRoleRestClient` | [`edgeops-common-lib/.../UserRoleRestClient.java`](https://github.com/UST-PACE/edgeops-core-lib/blob/58ea35e788c672316523ba52c3d11c3dc4b13d8a/edgeops-common-lib/src/main/java/com/ustri/digital/edgeops/rest/api/UserRoleRestClient.java) | `dagility-admin` |
| `WorkflowRestClient` | [`edgeops-common-lib/.../WorkflowRestClient.java`](https://github.com/UST-PACE/edgeops-core-lib/blob/58ea35e788c672316523ba52c3d11c3dc4b13d8a/edgeops-common-lib/src/main/java/com/ustri/digital/edgeops/rest/api/WorkflowRestClient.java) | `dagility-conveyor` |
| `DataProcessorDashboardRestClient` | [`edgeops-common-lib/.../DataProcessorDashboardRestClient.java`](https://github.com/UST-PACE/edgeops-core-lib/blob/58ea35e788c672316523ba52c3d11c3dc4b13d8a/edgeops-common-lib/src/main/java/com/ustri/digital/edgeops/rest/api/DataProcessorDashboardRestClient.java) | `dagility-datamorph` |
| `FeatureToggleRestClient` | [`edgeops-common-lib/.../FeatureToggleRestClient.java`](https://github.com/UST-PACE/edgeops-core-lib/blob/58ea35e788c672316523ba52c3d11c3dc4b13d8a/edgeops-common-lib/src/main/java/com/ustri/digital/edgeops/rest/api/FeatureToggleRestClient.java) | `dagility-admin` |
| `EnvironmentRestClient` | [`dagility-habitat-lib/.../EnvironmentRestClient.java`](https://github





Here's the **complete picture** of how `DataProcessorDashboardRestClient` flows from `core-lib` into `insight-module`:

---

## ✅ Answer: Yes — it IS defined in `core-lib`, and here's the full proof

---

### 1. 📍 Where `DataProcessorDashboardRestClient` is DEFINED — in `core-lib`

```java name=DataProcessorDashboardRestClient.java url=https://github.com/UST-PACE/edgeops-core-lib/blob/58ea35e788c672316523ba52c3d11c3dc4b13d8a/edgeops-common-lib/src/main/java/com/ustri/digital/edgeops/rest/api/DataProcessorDashboardRestClient.java
@FeignClient(
    contextId = "dataProcessorDashboards",
    name = "dagility-datamorph",
    url = "${module.url.datamorph:}",
    path = DataProcessorDashboardRestApi.PATH,
    configuration = FeignConfiguration.class
)
public interface DataProcessorDashboardRestClient extends DataProcessorDashboardRestApi {
}
```

- **Package**: `com.ustri.digital.edgeops.rest.api`
- **Module inside core-lib**: `edgeops-common-lib`
- **Calls**: the `dagility-datamorph` service via HTTP

---

### 2. 📍 Where it is USED — in `insight-module`

```java name=WcSelfTestServiceImpl.java url=https://github.com/UST-PACE/edgeops-insight-module/blob/d33bc4a5ca858080dd13fbf0dd9011f0f2f229b0/src/main/java/com/ustri/digital/edgeops/insight/service/wcselftest/WcSelfTestServiceImpl.java#L19-L89
import com.ustri.digital.edgeops.rest.api.DataProcessorDashboardRestClient;
...
private final DataProcessorDashboardRestClient dashboardRestClient;
```

It's injected as a Spring bean and used in `WcSelfTestServiceImpl` — the self-testing feature of the insight module.

---

### 3. 📦 How the dependency is declared — the `pom.xml` chain

This is the key part — **the dependency is NOT declared directly by artifact name "DataProcessorDashboardRestClient"**. Instead, it flows through Maven inheritance/dependencies:

```
insight-module/pom.xml
   │
   ├── <parent> → edgeops-core-lib/pom.xml        ← parent POM (groupId: com.ustri.digital.edgeops.services)
   │
   └── <dependency> edgeops-common-lib             ← explicit dependency in insight-module pom.xml (line 71-73)
            │
            └── contains DataProcessorDashboardRestClient
                (edgeops-common-lib/src/main/java/com/ustri/digital/edgeops/rest/api/)
```

**From `insight-module/pom.xml` (line 70–73):**
```xml name=pom.xml url=https://github.com/UST-PACE/edgeops-insight-module/blob/d33bc4a5ca858080dd13fbf0dd9011f0f2f229b0/pom.xml#L70-L73
<dependency>
    <groupId>com.ustri.digital.edgeops.services</groupId>
    <artifactId>edgeops-common-lib</artifactId>
    <version>${project.version}</version>
</dependency>
```

**And `insight-module` declares `edgeops-core-lib` as its parent (lines 4–9):**
```xml name=pom.xml url=https://github.com/UST-PACE/edgeops-insight-module/blob/d33bc4a5ca858080dd13fbf0dd9011f0f2f229b0/pom.xml#L4-L9
<parent>
    <groupId>com.ustri.digital.edgeops.services</groupId>
    <artifactId>edgeops-core-lib</artifactId>
    <version>${revision}</version>
    <relativePath>../edgeops-core-lib/pom.xml</relativePath>
</parent>
```

---

### 🗺️ Full Dependency Map Summary

| What | Where | How |
|---|---|---|
| `DataProcessorDashboardRestClient` **defined** | `core-lib` → `edgeops-common-lib` sub-module | `@FeignClient` calling `dagility-datamorph` |
| `DataProcessorDashboardRestClient` **used** | `insight-module` → `WcSelfTestServiceImpl` | Injected via `@RequiredArgsConstructor` |
| **Dependency link** | `insight-module/pom.xml` | `edgeops-common-lib` listed as `<dependency>` |
| **Parent link** | `insight-module/pom.xml` | `edgeops-core-lib` as `<parent>` |
| **`@EnableFeignClients`** | `InsightApplication.java` | `basePackages = "com.ustri.digital.edgeops"` — scans ALL packages including core-lib's clients |

---

### 🔑 Key Insight: How to always identify this pattern

> Whenever you see a Feign client in `insight-module` from package `com.ustri.digital.edgeops.rest.api.*` — **it is always defined in `core-lib/edgeops-common-lib`**, not in insight-module itself. The `@EnableFeignClients(basePackages = "com.ustri.digital.edgeops")` in `InsightApplication.java` is what causes Spring to scan and pick up ALL Feign clients across the entire package tree, including those in the `edgeops-common-lib` JAR dependency.




Here is the **complete map of all external calls** used in `edgeops-insight-module`, with their origin and target services:

---

## 🌐 All External Calls in `edgeops-insight-module`

### Category 1 — **HTTP/Feign Clients** (defined IN insight-module itself)

| Client | File | Calls Target Service | Protocol/URL Property |
|---|---|---|---|
| `MetricsClient` | [`insight/client/MetricsClient.java`](https://github.com/UST-PACE/edgeops-insight-module/blob/d33bc4a5ca858080dd13fbf0dd9011f0f2f229b0/src/main/java/com/ustri/digital/edgeops/insight/client/MetricsClient.java) | `dagility-notification` | `${module.url.dagility-notification:}` → `/internal/api/metrics/tags` |

---

### Category 2 — **HTTP/Feign Clients** (defined in `core-lib` → `edgeops-common-lib`, imported via dependency)

| Client | Defined In (core-lib) | Used In (insight-module) | Calls Target Service | URL Property |
|---|---|---|---|---|
| `DataProcessorDashboardRestClient` | [`edgeops-common-lib/.../rest/api/`](https://github.com/UST-PACE/edgeops-core-lib/blob/58ea35e788c672316523ba52c3d11c3dc4b13d8a/edgeops-common-lib/src/main/java/com/ustri/digital/edgeops/rest/api/DataProcessorDashboardRestClient.java) | `WcSelfTestServiceImpl` | `dagility-datamorph` | `${module.url.datamorph:}` |
| `DataProcessorWidgetRestClient` | [`edgeops-common-lib/.../rest/api/`](https://github.com/UST-PACE/edgeops-core-lib/blob/58ea35e788c672316523ba52c3d11c3dc4b13d8a/edgeops-common-lib/src/main/java/com/ustri/digital/edgeops/rest/api/DataProcessorWidgetRestClient.java) | `WcSelfTestServiceImpl` | `dagility-datamorph` | `${module.url.datamorph:}` |

---

### Category 3 — **HTTP/Feign Clients** (defined in `core-lib` → `dagility-habitat-lib`, imported via dependency)

| Client | Defined In (core-lib) | Used In (insight-module) | Calls Target Service | URL Property |
|---|---|---|---|---|
| `EnvironmentRestClient` | [`dagility-habitat-lib/.../rest/api/`](https://github.com/UST-PACE/edgeops-core-lib/blob/58ea35e788c672316523ba52c3d11c3dc4b13d8a/dagility-habitat-lib/src/main/java/com/ustri/digital/edgeops/services/habitat/rest/api/EnvironmentRestClient.java) | `JiraIssueServiceImpl` | `dagility-habitat` | `${module.url.habitat:}` |

---

### Category 4 — **Database Calls** (`NamedParameterJdbcTemplate`) — **PostgreSQL**

These are direct DB calls using JDBC, defined inside insight-module itself:

| DAO / Service | File | DB Target |
|---|---|---|
| `PgTaxonomyTreeDaoImpl` | `dao/taxonomy/PgTaxonomyTreeDaoImpl.java` | PostgreSQL (`edgeops.*` tables) |
| `PgTaxonomySLOPolicyDAOImpl` | `dao/taxonomy/slo/PgTaxonomySLOPolicyDAOImpl.java` | PostgreSQL (`edgeops.taxonomy_slo_*`) |
| `PgIlluminateDashboardWidgetDaoCustomImpl` | `dao/PgIlluminateDashboardWidgetDaoCustomImpl.java` | PostgreSQL (`edgeops.illuminate_*`) |
| `PgCompositeWidgetSettingsCustomDaoImpl` | `dao/PgCompositeWidgetSettingsCustomDaoImpl.java` | PostgreSQL (`composite_widget_settings`) |
| `SelfTestExecutionDaoImpl` | `dao/selftest/SelfTestExecutionDaoImpl.java` | PostgreSQL (`edgeops.self_test_*`) |
| `ClickhouseCleanupHandler` | `service/ClickhouseCleanupHandler.java` | **ClickHouse** (via `clickhouseJdbcTemplate`) |

---

### Category 5 — **Database Calls** — **ClickHouse** (via `BaseDaoImpl` / `getNamedParameterJdbcTemplate()`)

These extend `BaseDaoImpl` and query ClickHouse's `reporting.*` schema:

| DAO | DB Schema/Table |
|---|---|
| `JacocoCoverageDaoImpl` | `reporting.sonar_measures` |
| `JiraStatusMappingDaoImpl` | `reporting.jira_status_mapping` |
| `PipelineDateRangeDaoImpl` | `reporting.jira_sprint_burndown` |
| `TodayCicdActivityDaoImpl` | `reporting.ci_results` |
| `ScmCodeCommitsDaoClickHouseImpl` | `reporting.scm_*` |
| Many others in `dao/clickhouse/` | `reporting.*` |

---

## 🔗 How These All Connect — Dependency Chain

```
insight-module/pom.xml
│
├── <parent> → edgeops-core-lib          ← version management & shared config
│
├── edgeops-common-lib      → DataProcessorDashboardRestClient (dagility-datamorph)
│                           → DataProcessorWidgetRestClient    (dagility-datamorph)
│
├── dagility-habitat-lib    → EnvironmentRestClient            (dagility-habitat)
│
└── (own code)              → MetricsClient                    (dagility-notification)
                            → PostgreSQL (NamedParameterJdbcTemplate)
                            → ClickHouse (NamedParameterJdbcTemplate)
```

> ⚠️ Note: Search results are limited to 10 per query. There may be additional REST clients or DAOs not surfaced above. You can explore further at [GitHub Code Search](https://github.com/search?q=repo%3AUST-PACE%2Fedgeops-insight-module+RestClient&type=code).
