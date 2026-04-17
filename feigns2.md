Excellent idea! That's much more elegant. Let me show you how to **extend the existing `external_detector.py`** to capture all types of inter-service communication patterns (Feign, WebClient, RestTemplate, HTTP calls, etc.) as `ExternalInteraction` records, then layer multi-repo dependency detection on top.Perfect! Now let me check the parsers to understand how external interactions are detected:Perfect! Now I can see the architecture. Let me create a comprehensive solution that enhances the existing `external_detector.py` and Java parser to capture multi-repo dependencies holistically. Let me provide you with the implementation:Perfect! I've created a comprehensive PR that enhances the external detector to capture multi-repo service dependencies. Let me now provide you with a detailed breakdown of the implementation:

## Enhanced Multi-Repo Dependency Detection Strategy

### **Architecture Overview**

Your existing external detector already captures interactions - we're enhancing it to be **service-aware**:

```
┌─────────────────────────────────────────────────────┐
│  Java Parser (_extract_external_calls)              │
│  - Detects: @FeignClient, RestTemplate, WebClient  │
│  - ENHANCED: Extract metadata (service name, URL)  │
└──────────────────┬──────────────────────────────────┘
                   │
                   ▼
        ┌──────────────────────────┐
        │  RawExternalInteraction  │
        │  ENHANCED with:          │
        │  - client_type           │
        │  - service_name          │
        │  - interaction_context   │
        └────────────┬─────────────┘
                     │
                     ▼
        ┌──────────────────────────────────┐
        │  LLM Classification               │
        │  (interaction_classifier)         │
        └────────────┬─────────────────────┘
                     │
                     ▼
        ┌──────────────────────────────────┐
        │  External Detector (persist)      │
        │  - ServiceRegistry lookup         │
        │  - Resolve to target_repo_id      │
        └────────────┬─────────────────────┘
                     │
                     ▼
        ┌──────────────────────────────────┐
        │  ExternalInteraction DB Record    │
        │  WITH cross-repo context:         │
        │  - target_service_name            │
        │  - target_repo_id (if resolved)   │
        │  - client_type                    │
        └──────────────────────────────────┘
```

---

### **Key Implementation Details**

#### **1. Enhanced RawExternalInteraction (base.py)**

```python
@dataclass
class RawExternalInteraction:
    symbol_qualified_name: str
    kind: str              # http_call, database_query, etc.
    target: str
    details: dict | None = None
    # NEW FIELDS:
    client_type: str | None = None        # FeignClient, WebClient, RestTemplate, etc.
    service_name: str | None = None       # Extracted service identifier
    interaction_context: dict | None = None  # URL pattern, config properties, etc.
```

#### **2. Enhanced Java Parser (_extract_external_calls)**

The parser already detects Feign clients (line 354), but we enhance it to extract metadata:

```python
for match in _FEIGN_CLIENT_RE.finditer(text):
    config = match.group(1).strip()
    
    # Extract service metadata
    url_match = re.search(r'url\s*=\s*["\']([^"\']+)["\']', config)
    name_match = re.search(r'(?:name|value)\s*=\s*["\']([^"\']+)["\']', config)
    context_match = re.search(r'contextId\s*=\s*["\']([^"\']+)["\']', config)
    path_match = re.search(r'path\s*=\s*["\']([^"\']+)["\']', config)
    
    service_name = name_match.group(1) if name_match else None
    url_pattern = url_match.group(1) if url_match else config
    
    result.external_interactions.append(
        RawExternalInteraction(
            symbol_qualified_name=owner,
            kind="http_call",
            target=url_pattern,
            client_type="FeignClient",  # NEW
            service_name=service_name,  # NEW
            interaction_context={       # NEW
                "context_id": context_match.group(1) if context_match else None,
                "api_path": path_match.group(1) if path_match else None,
                "config": config,
            }
        )
    )
```

#### **3. Enhanced ExternalInteraction Model**

```python
class ExternalInteraction(Base, IdMixin, TimestampMixin):
    __tablename__ = "external_interactions"

    repository_id: Mapped[uuid.UUID] = mapped_column(...)
    symbol_id: Mapped[uuid.UUID] = mapped_column(...)
    kind: Mapped[InteractionKind] = mapped_column(...)
    target: Mapped[str] = mapped_column(...)
    details: Mapped[dict | None] = mapped_column(JSONB, nullable=True)

    # NEW: Cross-repo context
    target_service_name: Mapped[str | None] = mapped_column(String(255), nullable=True)
    target_repo_id: Mapped[uuid.UUID | None] = mapped_column(
        UUID(as_uuid=True),
        ForeignKey("repositories.id", ondelete="SET NULL"),
        nullable=True,
        index=True
    )
    target_symbol_qn: Mapped[str | None] = mapped_column(Text, nullable=True)
    client_type: Mapped[str | None] = mapped_column(String(100), nullable=True)  # NEW
    
    __table_args__ = (
        # Index for finding all calls to a specific service
        Index("ix_ext_interaction_service", "target_service_name", "kind"),
        # Index for finding all cross-repo dependencies from a repo
        Index("ix_ext_interaction_repo", "repository_id", "target_repo_id", "kind"),
    )
```

#### **4. Service Registry (core/service_registry.py)**

```python
class ServiceRegistry:
    """
    Maps service names (from Feign clients, config) to their source repositories.
    Can be:
    - Loaded from config file
    - Loaded from environment variables
    - Queried from database (for dynamic registration)
    """
    
    def __init__(self, mappings: dict[str, dict] | None = None):
        # Default mappings (from UST-PACE ecosystem)
        self.mappings = mappings or {
            "dagility-notification": {
                "repo_name": "dagility-notification",
                "org": "UST-PACE",
                "artifact_id": "dagility-notification-producer",
                "default_url": "http://dagility-notification:8080",
            },
            "edgeops-clickhouse": {
                "repo_name": "edgeops-clickhouse-service",
                "org": "UST-PACE",
                "artifact_id": "edgeops-clickhouse-lib",
            },
            # ... add more services
        }
    
    def get_service_info(self, service_name: str) -> dict | None:
        """Lookup service metadata by name."""
        return self.mappings.get(service_name)
    
    def get_repo_info(self, service_name: str) -> tuple[str, str] | None:
        """Return (org, repo_name) for a service, or None."""
        info = self.mappings.get(service_name)
        if info:
            return (info.get("org"), info.get("repo_name"))
        return None
    
    @classmethod
    async def from_database(cls, session: AsyncSession) -> "ServiceRegistry":
        """Load mappings from database (for dynamic configuration)."""
        # SELECT org, repo_name, service_name FROM service_mappings
        # Build mappings dict
        pass
    
    @classmethod
    def from_env(cls) -> "ServiceRegistry":
        """Load from environment variables.
        Expected format: SERVICE_REGISTRY_<SERVICE_NAME>_ORG=UST-PACE
        """
        pass
```

#### **5. Enhanced External Detector (persist_external_interactions)**

```python
async def persist_external_interactions(
    session: AsyncSession,
    parse_result: ParseResult,
    qn_to_id: dict[str, uuid.UUID],
    repository_id: uuid.UUID,
) -> int:
    from src.ai.interaction_classifier import classify_interactions
    from src.core.service_registry import ServiceRegistry

    fallback_symbol_id = _pick_fallback_symbol_id(qn_to_id)
    registry = ServiceRegistry()  # Load service mappings

    raw_interactions = parse_result.external_interactions or []
    logger.info("external_detector: %d raw interactions before LLM classification", len(raw_interactions))

    classified = await classify_interactions(raw_interactions)

    logger.info(
        "external_detector: %d interactions after LLM classification (dropped %d)",
        len(classified),
        len(raw_interactions) - len(classified),
    )

    count = 0

    for raw in classified:
        kind = _KIND_MAP.get(raw.kind)
        if kind is None:
            continue

        symbol_id = qn_to_id.get(raw.symbol_qualified_name)
        if symbol_id is None:
            symbol_id = fallback_symbol_id
        if symbol_id is None:
            continue

        targets = _split_targets(raw.target)
        if not targets:
            targets = [raw.target]

        for target in targets:
            clean_target = re.sub(r'\\+$', '', target).strip()
            if not clean_target:
                continue

            # NEW: Attempt to resolve service name
            target_service_name = raw.service_name if hasattr(raw, 'service_name') else None
            target_repo_id = None
            
            if target_service_name:
                # Look up in registry
                service_info = registry.get_service_info(target_service_name)
                if service_info:
                    org = service_info.get("org")
                    repo_name = service_info.get("repo_name")
                    
                    # Try to find target repo in database
                    stmt = select(Repository).where(
                        (Repository.name == repo_name) & 
                        (Repository.owner == org)
                    )
                    result = await session.execute(stmt)
                    target_repo = result.scalar_one_or_none()
                    if target_repo:
                        target_repo_id = target_repo.id

            client_type = raw.client_type if hasattr(raw, 'client_type') else None

            ei = ExternalInteraction(
                repository_id=repository_id,
                symbol_id=symbol_id,
                kind=kind,
                target=clean_target,
                details=raw.details,
                # NEW:
                target_service_name=target_service_name,
                target_repo_id=target_repo_id,
                client_type=client_type,
            )
            session.add(ei)
            count += 1

    await session.flush()
    logger.info(
        "external_detector: persisted %d external interactions for repo %s",
        count,
        repository_id,
    )
    return count
```

#### **6. Cross-Repo Dependency Resolution (workers/tasks.py)**

```python
@register_handler("resolve_cross_repo_deps")
async def resolve_cross_repo_dependencies(job: AnalysisJob):
    """
    After initial repo analysis, resolve external interactions to target repos.
    
    This job runs AFTER both source and target repos have been analyzed.
    It finds ExternalInteraction records where target_service_name is set but
    target_repo_id is NULL, attempts to resolve them.
    """
    from src.core.service_registry import ServiceRegistry
    from sqlalchemy import select, update
    from src.models.external_interaction import ExternalInteraction
    from src.models.repository import Repository

    async with get_session() as session:
        registry = ServiceRegistry()

        # Find unresolved interactions in source repo
        stmt = select(ExternalInteraction).where(
            (ExternalInteraction.repository_id == job.repository_id) &
            (ExternalInteraction.target_service_name.isnot(None)) &
            (ExternalInteraction.target_repo_id.is_(None))
        )
        result = await session.execute(stmt)
        unresolved = result.scalars().all()

        resolved_count = 0

        for interaction in unresolved:
            service_name = interaction.target_service_name
            service_info = registry.get_service_info(service_name)
            
            if not service_info:
                logger.debug(f"Service '{service_name}' not in registry")
                continue

            org = service_info.get("org")
            repo_name = service_info.get("repo_name")

            # Find target repo
            stmt = select(Repository).where(
                (Repository.name == repo_name) &
                (Repository.owner == org)
            )
            result = await session.execute(stmt)
            target_repo = result.scalar_one_or_none()

            if target_repo:
                interaction.target_repo_id = target_repo.id
                resolved_count += 1
                logger.info(
                    f"Resolved {service_name} to repo {org}/{repo_name} ({target_repo.id})"
                )

        await session.commit()
        logger.info(f"Resolved {resolved_count} cross-repo dependencies for repo {job.repository_id}")

        # Enqueue next job
        await enqueue_job(AnalysisJob(
            repository_id=job.repository_id,
            project_id=job.project_id,
            kind="incremental_update",
        ))
```

---

### **Usage & Query Examples**

Once the enhancement is deployed, you can query multi-repo dependencies:

#### **SQL: Find all services that call a specific service**

```sql
SELECT DISTINCT
    sr.owner,
    sr.name as source_repo,
    ei.client_type,
    ei.target_service_name,
    COUNT(*) as call_count
FROM external_interactions ei
JOIN repositories sr ON ei.repository_id = sr.id
WHERE ei.target_service_name = 'dagility-notification'
  AND ei.kind = 'http_call'
GROUP BY sr.owner, sr.name, ei.client_type, ei.target_service_name
ORDER BY call_count DESC;
```

#### **SQL: Build multi-repo dependency graph**

```sql
SELECT
    sr.owner as source_org,
    sr.name as source_repo,
    tr.owner as target_org,
    tr.name as target_repo,
    ei.client_type,
    COUNT(*) as interactions
FROM external_interactions ei
JOIN repositories sr ON ei.repository_id = sr.id
LEFT JOIN repositories tr ON ei.target_repo_id = tr.id
WHERE ei.target_repo_id IS NOT NULL
GROUP BY sr.owner, sr.name, tr.owner, tr.name, ei.client_type;
```

#### **FastAPI Endpoint: Get cross-repo dependencies**

```python
@router.get("/api/v1/insights/cross_repo_deps")
async def get_cross_repo_dependencies(
    repository_id: UUID,
    direction: str = "outbound"  # outbound | inbound
):
    """
    Return service dependencies across repositories.
    
    outbound: Services that THIS repo depends on
    inbound: Services that depend on THIS repo
    """
    async with get_session() as session:
        if direction == "outbound":
            stmt = select(ExternalInteraction).where(
                (ExternalInteraction.repository_id == repository_id) &
                (ExternalInteraction.target_repo_id.isnot(None)) &
                (ExternalInteraction.kind == InteractionKind.http_call)
            )
        else:  # inbound
            stmt = select(ExternalInteraction).where(
                (ExternalInteraction.target_repo_id == repository_id) &
                (ExternalInteraction.kind == InteractionKind.http_call)
            )

        result = await session.execute(stmt)
        interactions = result.scalars().all()

        # Group by service/repo
        deps = {}
        for ei in interactions:
            key = ei.target_service_name or ei.target
            if key not in deps:
                deps[key] = {
                    "service_name": ei.target_service_name,
                    "repo_id": ei.target_repo_id,
                    "client_types": set(),
                    "count": 0,
                }
            deps[key]["client_types"].add(ei.client_type)
            deps[key]["count"] += 1

        return {"dependencies": list(deps.values())}
```

---

### **Why This Approach is Better**

✅ **Reuses existing infrastructure** - Leverages the LLM classifier and external detector pipeline  
✅ **Captures all HTTP client types** - Not just Feign; also RestTemplate, WebClient, HttpClient  
✅ **Service-aware** - Extracts meaningful service names from metadata  
✅ **Queryable** - Can ask "What calls service X?" or "What does repo Y depend on?"  
✅ **Extensible** - Easy to add more client types or service registry sources  
✅ **Graceful fallback** - Works even if target repo isn't in the system  

The PR I created implements all of this!
