Here is a comprehensive summary of all external calls from this repository, organized by category:

---

## External Calls from the Elasticsearch Repository

### 1. Build & Artifact Repositories (Gradle/Maven)

| Endpoint | Purpose |
|---|---|
| `https://repo1.maven.org/maven2/` | Maven Central — primary library downloads |
| `https://artifactory.elastic.dev/artifactory/elasticsearch-native` | Elastic internal Artifactory (native libraries) |
| `https://artifactory.elstc.co/artifactory/shibboleth-releases/` | Shibboleth SAML/SSO library releases |
| `https://s3.amazonaws.com/download.elasticsearch.org/lucenesnapshots/` | Lucene snapshot builds (AWS S3) |
| `https://storage.googleapis.com/elasticsearch-cuvs-snapshots` | CUVS (cuVS vector search) snapshots (GCS) |
| `https://snapshots.elastic.co/maven/` | Elastic snapshot artifacts |
| `https://artifacts-snapshot.elastic.co/elasticsearch/` | Elastic DRA snapshot artifacts |
| `https://artifacts-staging.elastic.co/` | Elastic staging artifacts |
| `https://artifacts.elastic.co/` | Elastic production release artifacts |
| `https://raw.githubusercontent.com/elastic/elasticsearch/main/` | GitHub raw content (e.g. version files, branches.json) |

### 2. JDK Downloads (Build toolchain)

| Endpoint | Purpose |
|---|---|
| `https://api.adoptium.net/v3/binary/version/` | Eclipse Adoptium (Temurin) JDK downloads |
| `https://builds.es-jdk-archive.com/` | Elastic JDK archive (early-access / archived builds) |
| `https://download.oracle.com/java/` | Oracle JDK downloads |
| `https://cdn.azul.com` | Azul Zulu JDK downloads |

### 3. CI/CD External Services

| Endpoint | Purpose |
|---|---|
| `https://gradle-enterprise.elastic.co` / `https://develocity.elastic.co` | Gradle Enterprise build scan server |
| `https://api.buildkite.com/v2/` | Buildkite API (CI pipeline management, build status) |
| `https://es-buildkite-agents.elastic.dev/` | Elastic Buildkite agent metrics/logs |
| `https://snyk.io` | Snyk dependency vulnerability scanning |
| `https://secrets.elastic.co:8200` | Vault secrets management |
| `https://elastic-release-api.s3.us-west-2.amazonaws.com/` | Release manifest fetching (AWS S3) |
| `https://artifacts-snapshot.elastic.co/beats/` | Beats artifacts (CI dependency) |
| `https://artifacts-staging.elastic.co/beats/` `…/ml-cpp/` | Staging beats/ML artifacts (DRA workflow) |
| `https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip` | AWS CLI installer download |

### 4. AI / Machine Learning Inference APIs (x-pack inference plugin)

| External Service | Endpoint |
|---|---|
| **OpenAI** | `api.openai.com` |
| **Anthropic** | `api.anthropic.com` |
| **Azure OpenAI** | `<resource>.openai.azure.com` |
| **Google Vertex AI** | `<location>-aiplatform.googleapis.com` |
| **Google AI Studio** | `generativelanguage.googleapis.com` |
| **Google Discovery Engine** | `discoveryengine.googleapis.com` |
| **Amazon Bedrock** | AWS SDK (region-based endpoints) |
| **AWS SageMaker** | AWS SDK (region-based endpoints) |
| **Cohere** | `api.cohere.ai` |
| **Mistral** | `https://api.mistral.ai/v1/` |
| **Groq** | `api.groq.com` |
| **NVIDIA** | `integrate.api.nvidia.com`, `ai.api.nvidia.com` |
| **HuggingFace** | User-configurable URL |
| **JinaAI** | `api.jina.ai` |
| **VoyageAI** | `api.voyageai.com` |
| **DeepSeek** | `https://api.deepseek.com/chat/completions` |
| **Fireworks AI** | `https://api.fireworks.ai/inference/v1/embeddings` |
| **AI21** | `api.ai21.com` |
| **IBM WatsonX** | User-configurable URL |
| **Alibaba Cloud Search** | User-configurable host |
| **Elastic Inference Service (EIS)** | Configurable via `xpack.inference.elastic.url` / `xpack.inference.eis.gateway.url` |
| **Azure AI Studio** | Azure-hosted |
| **OpenShift AI** | Configurable URL |
| **Contextual AI** | Configurable URL |
| **Llama** | Configurable URL |

### 5. Cloud Storage Repositories (snapshot/backup)

| Service | Endpoint |
|---|---|
| **AWS S3** | AWS SDK (`amazonaws.com`, region-based) |
| **Google Cloud Storage (GCS)** | `https://www.googleapis.com` + OAuth2 at `oauth2.googleapis.com` |
| **Azure Blob Storage** | Azure SDK (`*.blob.core.windows.net`) + `login.microsoft.com` for auth |

### 6. GeoIP / Database Downloads

| Endpoint | Purpose |
|---|---|
| `https://geoip.elastic.co/v1/database` | Elastic GeoIP database endpoint (default) |
| `https://download.maxmind.com/geoip/databases` | MaxMind GeoIP enterprise databases |
| `https://ipinfo.io/data` | IPInfo.io enterprise databases |

### 7. Node Discovery Plugins

| Service | Endpoint |
|---|---|
| **AWS EC2** | `http://ec2.amazonaws.com/doc/2013-02-01/` (AWS SDK) |
| **Azure Classic** | `https://management.azure.com` (Azure SDK) |
| **Google Compute Engine (GCE)** | `https://www.googleapis.com` |

### 8. Watcher Notification Integrations

| Service | Endpoint |
|---|---|
| **PagerDuty** | `events.pagerduty.com` |
| **Slack** | User-configured webhook URL |
| **Jira** | User-configured URL |
| **Email** | User-configured SMTP host |
| **Webhook** | Arbitrary user-configured URLs |

### 9. Security / Auth Integrations

| Protocol | External Dependency |
|---|---|
| **OIDC** | Any OIDC provider (user-configured `op.issuer`) |
| **SAML** | Any SAML IdP (user-configured `idp.entity_id`) |
| **LDAP/AD** | Any LDAP server (user-configured) |
| **Kerberos** | Any KDC (configured externally) |
| **Microsoft Graph** | `https://graph.microsoft.com/v1.0`, `https://login.microsoftonline.com` |

### 10. Docker Base Images

| Registry / Image | Used In |
|---|---|
| `docker.elastic.co/wolfi/chainguard-base:latest` | `wolfi/Dockerfile` |
| `docker.elastic.co/wolfi/chainguard-base-fips:latest` | `cloud_ess_fips/Dockerfile` |
| `redhat/ubi8` | zstd.Dockerfile |
| `${base_image}` (parameterized) | `default/Dockerfile`, `ironbank/Dockerfile` |
