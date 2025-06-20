# General Technical Review - Fluid / Incubating

- **Project:** Fluid
- **Project Version:** v1.0.5
- **Website:** https://fluid-cloudnative.github.io/
- **Date Updated:** 2025-07-01
- **Template Version:** v1.0
- **Description:** Fluid, elastic data abstraction and acceleration for BigData/AI applications in cloud. 
- **Reviewers:** Rong Gu, Yang Che, Zhihao Xu, Kai Zhang, Lingwei Qiu

## Day 0 - Planning Phase

### Scope

  * Describe the roadmap process, how scope is determined for mid to long term features, as well as how the roadmap maps back to current contributions and maintainer ladder?  

    **Fluid’s roadmap begins with its core vision:** *"Accelerate data-intensive workloads (e.g., AI, analytics) in Kubernetes through efficient data orchestration."* **It is dynamically shaped by community feedback and structured into mid-term (6-18 months) and long-term (2-3 years) phases.** Prioritized initiatives derive from:  
    - **GitHub Issues/PRs**: Feature requests and bug reports (e.g., dataset preloading for AI pipelines).  
    - **User Surveys**: Annual adopter surveys identifying production pain points.  
    - **Community Forums**: Feedback from Slack, WeChat, DingTalk, KubeCon sessions, and community meetings.  
    - **Ecosystem Shifts**: Tracking changes in dependencies (e.g., Kubernetes storage APIs) and integrations (e.g., KServe, Alluxio, JuiceFS).  

    **The roadmap actively supports the maintainer ladder by:**  

    - Providing growth opportunities through high-impact work on governance and prioritized features.  
    - Encouraging new contributors via mentorship on roadmap-aligned tasks.  
    - Recognizing review efforts and feature ownership in contributor promotions.  

  * Describe the target persona or user(s) for the project?
  
    **Distributed Storage Developers**  
    - Build Kubernetes storage plugins based FUSE client with minimal complexity  
    - Develop/Maintain with Docker/shell scripts instead of CSI plugins  
    - Focus only on FUSE mount/unmount operations  
    
    **ML Platform Engineers/MLOps**  
    - Deploy hybrid compute infrastructure (CSI mode + sidecar mode deployment)  
    - Manage diverse caching runtimes (JuiceFS/Alluxio/Vineyard) in unified way  
    - Automate cache operations (warmup/migration/scaling) via CRDs  
    
    **Data Scientists/ML Researchers**  
    - Accelerate data access for faster model iteration  
    - Manage datasets via Python SDK without YAML complexity  
    - Dynamically mount/unmount datasets in running containers  
  
  * Explain the primary use case for the project. What additional use cases are supported by the project?

    - **Unified API and distributed cache Management**: Use the unified API to manage datasets with distributed cache implenementation such as Alluxio, JuiceFS, Vineyard, etc. And use the same manner to automate dataset preloading, scaling, and migration.
    - **Extensible Runtime Plugins**: Support lightweight and extensible runtime plugin framework to enable more distributed cache engines and FUSE based storage clients. These plugins are divided into:
      - **CacheRuntime**: Accelerates data access for storage systems like S3, HDFS, OSS and more via different caching engines such as Alluxio, JuiceFS, and Vineyard and more.
      - **ThinRuntime**: Provides a unified interface for accessing third-party storage, simplifying integration.
    - **Data Access Acceleration**:  Improve data access performance by combining distributed data caching with autoscaling, portability, observability, and affinity scheduling capabilities. This allows for elastic scaling of cache systems and efficient data affinity scheduling. And application side optimizations based on the application’s features.
    - **Runtime Platform Agnostic**: Work across diverse environments, including native, edge, serverless Kubernetes clusters, and multi-cluster setups. It supports various configurations using CSI Plugin and sidecar modes, adapting to different operational environments such as cloud platforms and edge computing.
    
    Additional features supported:
    - **Simplify Distributed Storage Integration**: Allows storage developers to create Kubernetes-integrated plugins using simple **Docker/shell scripts** instead of complex CSI drivers. Developers only implement `mount`/`unmount` operations via FUSE, cutting development time from months to days .  
    - **Zero-Restart Data Source Hot-Swap**: Dynamically mount/unmount datasets in running containers (e.g., JupyterLab), eliminating workflow interruptions .  
    - **Python SDK**: Data scientists manage datasets via Python without YAML/CLI expertise, reducing onboarding friction.  
    - **Dataset Management**:  Unified management of 10K+ datasets in production. It can also support sharing and isolation according to the user's different demand.

* Explain which use cases have been identified as unsupported by the project.  

   - **Persistent Storage Replacement**: Operates caching layers only; requires independent backend storage (S3/HDFS/NFS).  
   - **Transactional Workloads**: Architecturally incompatible with OLTP systems (MySQL/Redis) due to lack of ACID and high write latency.  
   - **Non-Kubernetes Environments**: Exclusively designed for cloud-native K8s ecosystems.  

* Describe the intended types of organizations who would benefit from adopting this project. (i.e. financial services, any software manufacturer, organizations providing platform engineering services)?  
   - Enterprise with ML platform teams running on k8s.  
   - Organizations requiring running on Multi-Cloud/Hybrid Cloud and Multi types of distributed storage.   
   - Mid-Size Companies Scaling Data-Intensive Analytics.  
   - Cloud K8s Providers with AI platform support.  
   - Model as a Service Company.

* Please describe any completed end user research and link to any reports.
    - Fluid has wide range of adopters including cloud providers, end users in finance, AI for Science, game, e-business  industries. According to the [Q3 2024 CNCF Technology Landscape Radar report](https://www.cncf.io/reports/cncf-technology-landscape-radar/) findings,  Fluid has gained widespread recognition in the fields of batch processing, AI, and ML, and have been extensively deployed in end-users' production environments. 

### Usability

  * How should the target personas interact with your project?  

    * Data Scientists prefer a straightforward and easy way to define data access without getting involved in the complexities of infrastructure. They Declaratively define datasets (sources, mounts) via high-level Python SDK. 
    * ML Platform Engineer demands granular control for customizing the configuration in greater detail through K8s CRDs and client go. 
  
  * Describe the user experience (UX) and user interface (UI) of the project.  

    Fluid offers a seamless user experience through the kubectl CLI and Kubernetes CRDs. Data scientists can also enjoy flexibility with the Fluid Python SDK.

  * Describe how this project integrates with other projects in a production environment.

    This project integrates seamlessly into the MLOps stack by enhancing model training and deployment in production environments. Fluid accelerates training by working with Kubeflow and KubeDL Training Operator, allowing data scientists to prefetch datasets into distributed caches. Platform engineers benefit from Fluid's model preloading and cache-affinity scheduling to optimize KServe deployments.

    For observability, Fluid provides detailed metrics like cache hit rates and bandwidth usage to Prometheus/Grafana, helping teams identify and resolve performance issues such as cold starts and I/O contention.   

    Runtime implementations like Vineyard, Alluxio, and JuiceFS ensure efficient data handling. These integrations streamline operations and improve resource utilization in production.  

### Design

  * Explain the design principles and best practices the project is following.   

    - **Kubernetes Native**
      Fluid is designed to use core features of Kubernetes, without reinventing the wheel. The project utilizes a combination of:
      - Kubernetes Controllers
      - Custom Resource Definitions (CRDs)
      - Webhooks (Validation and Mutating)
      - the Control Plane's event notification system
    - **Unified Data AbstractionI and control Plane**
      - Runtime-Agnostic Abstraction: Single API to manage diverse storage backends (S3/HDFS/NAS) via CacheRuntime (Alluxio/JuiceFS) or ThinRuntime (custom FUSE).
      - Consistent Dataset UX: Mount datasets identically across clouds using Dataset CRD – no vendor-specific logic.
    - **Security and Multi-Tenancy**
      - Use Kubernetes namespace for tenant isolation with configured resources limits and RBAC.
    - **Extensibility**
      - Allow users to use Pluggable Runtime Framework to support new Cache Engine.
    - **Cloud and Platform Agnostic**
      - Runs on any cloud provider or on-prem kubernetes cluster.
      - Avoids vendor lock-in while enabling dataset and cache engine management.  

  * Outline or link to the project’s architecture requirements? Describe how they differ for Proof of Concept, Development, Test and Production environments, as applicable.  
  
     The project architecture documentation is available at: [https://fluid-cloudnative.github.io/docs/next/core-concepts/architecture-and-concepts](https://fluid-cloudnative.github.io/docs/next/core-concepts/architecture-and-concepts). Its requirement is Kubernetes (1.18+).

     - Proof of Concept: user can run the quick installation script to install Fluid in local environment.
     - Development/Test: developer can install Fluid on Kind or Minikube for quick development iteration.
     - Fluid is primarily focusing on deploying in the production environment on k8s cluster using the officially released Fluid helm chart.  
  
  * Define any specific service dependencies the project relies on in the cluster.  
     - Kubernetes APIServer for creating/updating/deleting Fluid custom resources.
    
  * Describe how the project implements Identity and Access Management.  

    Fluid leverages Kubernetes' built-in IAM features to manage identity and access control. It uses Kubernetes Role-Based Access Control (RBAC) to define roles and permissions for users and services, ensuring secure access to resources within the cluster. This approach allows administrators to specify detailed access policies based on user roles, namespaces, and resource types.
    
    Fluid also integrates with Kubernetes secrets to manage sensitive information securely, ensuring that credentials and API keys are protected.   

  * Describe how the project has addressed sovereignty.  
    Fluid ensures sovereignty by being cloud and platform agnostic, capable of running on any Kubernetes cluster—whether on-premises or with any cloud provider. It avoids vendor lock-in through the use of open standards and Kubernetes-native APIs. This design ensures that all data remains under the adopter's control. Integrations with external storage or identity providers are optional and fully configurable, empowering users to maintain autonomy over their infrastructure choices.  

  * Describe any compliance requirements addressed by the project.  
    Fluid enhances compliance with industry standards through Kubernetes RBAC and network policies, enabling multi-tenancy and namespace isolation. It supports secure credential storage and is suitable for regulated industries. Users can configure Fluid to meet requirements like data residency, access control, and auditability. Specific compliance certifications, however, depend on the adopter's deployment.

 * Describe the project’s High Availability requirements.  
   Fluid leverages Kubernetes' High Availability (HA) features by deploying core components as Kubernetes Deployments. It utilizes leader election to maintain a single active instance, with others on standby, ensuring seamless failover without downtime. To enhance availability, users should configure multiple replicas for components like the controller and policy servers.  
  
  * Describe the project’s resource requirements, including CPU, Network and Memory.  
    CPU and Memory requirements vary depending on the number of datasets and cache runtimes. 
    You can set `dataset-controller`, `runtime-controller` and `webhook`'s limits and requests using Helm chart values. Default values are here, at [this link](https://github.com/fluid-cloudnative/fluid/blob/master/charts/fluid/fluid/values.yaml5). 

  * Describe the project’s storage requirements, including its use of ephemeral and/or persistent storage.  
    Here's a concise description of Fluid's storage requirements:  

    - **Cache Workers**: Use configurable tiers of *ephemeral* storage (memory, local SSDs, or cloud disks) with user-defined quotas.  
    - **Persistent Backing Storage**: Relies on external systems (S3, HDFS, NAS) for durable data - *not managed by Fluid*.  
    - **Control Plane**: Stateless design requiring *zero persistent storage*; all state managed via Kubernetes APIs.  

  * Please outline the project’s API Design:  
    * Describe the project’s API topology and conventions

    Fluid integrates with Kubernetes by providing a set of CRDs.
    - **Core CRDs**:  
      - `Dataset`: Defines data sources (S3/HDFS/NAS), access policies, and caching requirements  
      - `Runtime` (AlluxioRuntime/JuiceFSRuntime): Configures distributed cache engines  
    - **API Characteristics**:  
      - Versioned APIs (`data.fluid.io/v1alpha1`) with OpenAPI schema validation  
      - Declarative state reconciliation: `spec` defines desired state, `status` reports actual state  
      - Kubernetes-native CRUD via `kubectl`/K8s API server 

    CRDs documentation is at [https://fluid-cloudnative.github.io/docs/next/release-and-API-doc/api-doc](https://fluid-cloudnative.github.io/docs/next/release-and-API-doc/api-doc).  
    The API follows standard Kubernetes extension patterns and is organized in groups and versions.

    * Describe the project defaults  

    The default values for a Fluid Helm installation are available at [https://github.com/fluid-cloudnative/fluid/blob/v1.0.5/charts/fluid/fluid/values.yaml](https://github.com/fluid-cloudnative/fluid/blob/v1.0.5/charts/fluid/fluid/values.yaml)
    These defaults provide a secure deployment configuration.   

    * Outline any additional configurations from default to make reasonable use of the project  

      - **Resource Management**: In production, set resource requests and limits of dataset controller according to the number of datasets and cache runtimes.
      - **Monitoring and Alerting**: Implement observability by setting up Prometheus and Grafana for monitoring and alerting.
      - **Autoscaling**: Configure autoscaling for each CacheRuntime to efficiently manage varying data access traffic.
      - **Scheduled Data Loading**: Establish a schedule for data loading to ensure timely updates.
      - **Distributed Cache System**: The Master Pod component is essential for maintaining file metadata and tracking cache status within the underlying storage system. Ensure it is configured for high availability to support system reliability.  
      - **Controller Replicas**: Deploy multiple controller replicas in production to enhance reliability.
      - **Management Plane**: Enable logging, tracing, and Prometheus metrics on the management plane for comprehensive monitoring and observability.
      
    * Describe any new or changed API types and calls - including to cloud providers - that will result from this project being enabled and used  

      - Fluid introduces the **`Dataset` Custom Resource Definition (CRD)** as the primary API type for abstracting and managing data assets in Kubernetes. This is a new resource type not present in vanilla Kubernetes.  
      - The **`DataOperation` CRD** enables declarative automation of data workflows including `Dataload`, `DataMigrate`, and `DataOperation` operations with trigger-based execution.  
      - Runtime-specific CRDs include **`CacheRuntime`** (for acceleration engines like Alluxio/JuiceFS) and **`ThinRuntime`** (for unified third-party storage access).  
      - Enabling Fluid adds these new CRDs but **does not modify existing Kubernetes core APIs** like Pods, Statefulset, or PersistentVolumeClaims.  
      - For cloud storage integration, Fluid uses **standard storage URIs (s3://, gs://, oss://)** in `Dataset` specs. This requires Kubernetes Secrets for credentials but introduces **no new cloud provider API types**.  
      - Fluid exposes **Prometheus-compatible metrics endpoints** (`/metrics`) for cache observability and **FUSE-based POSIX paths** for data access.   
      - Fluid requires **no changes to cloud provider APIs** and maintains complete backward compatibility with standard Kubernetes storage primitives.  

    * Describe compatibility of any new or changed APIs with API servers, including the Kubernetes API server  

      - All Fluid CRDs (`Dataset`, `DataOperation`, `CacheRuntime`, `ThinRuntime`) are implemented as standard Kubernetes Custom Resource Definitions and maintain full compatibility with the Kubernetes API server.  
      - These CRDs adhere strictly to Kubernetes API conventions for create/update/delete operations and status reporting, working seamlessly with `kubectl`, client libraries, and Kubernetes API tooling.  
      - Fluid CRDs employ versioned schemas validated through OpenAPI definitions, ensuring compatibility with Kubernetes admission controllers and server-side validation.  
      - The project is tested against upstream Kubernetes and supports standard Kubernetes RBAC, admission webhooks, and API aggregation layers without modification.  
      - Fluid introduces **zero changes to existing Kubernetes core APIs**; it exclusively extends the API surface through new CRDs.  
      - All API extensions operate within Kubernetes' standard extension mechanisms, requiring no fork or patching of the API server.  
      - Compatibility is maintained across all environments supported by Kubernetes (cloud, on-prem, edge) without API server adjustments.  

    * Describe versioning of any new or changed APIs, including how breaking changes are handled  

      Fluid follows strict semantic versioning](https://semver.org/) for all releases, including its APIs and CRDs.  
      For CRDs, the stable version in use is `v1`. In line with Kubernetes conventions, breaking changes to the API trigger a new version. Non-breaking enhancements are introduced within the existing version.
      Experimental features are delivered in alpha releases (v1alphaX), which may undergo significant changes or removal.
      Mature but pre-stable features use beta versions (v1betaX), allowing refinements before GA.
      All CRD versions include OpenAPI schema validation and support Kubernetes conversion webhooks for backward compatibility.  

    * Describe the project’s release processes, including major, minor and patch releases.  

      - Fluid follows a **structured release cycle** with documented procedures for major, minor, and patch releases, detailed in the [RELEASE_GUIDE.md](https://github.com/fluid-cloudnative/community/blob/master/operations/release.md).  
      - All releases include:  
        - Versioned artifacts (container images, Helm charts)  
        - Comprehensive release notes with upgrade instructions  
        - CHANGELOG.md detailing features, fixes, and deprecations  
      - **Release candidates (RC)** undergo:  
        1. 2-week community validation period  
        2. Automated conformance testing on upstream Kubernetes (v1.22+)  
      - **CI/CD pipeline** enforces:  
        - End-to-end test pass rate == 100%  
        - CVE scanning via Trivy  
        - Artifact signing. 
      - Deprecated features maintain **backward compatibility for ≥2 minor releases** before removal.  
      - Final artifacts publish to:  
        - Docker Container Registry (`docker.io/fluidcloudnative`)  
        - Helm Hub (`helm install fluid fluid/fluid`)  
        - GitHub Releases Helm package zips  

### Installation

  * Describe how the project is installed and initialized, e.g. a minimal install with a few lines of code or does it require more complex integration and configuration?  

      Please refer to the [Installation Guide](https://fluid-cloudnative.github.io/docs/next/get-started/installation)。 For example, Fluid can be installed via Helm `helm install fluid fluid/fluid`.

  * How does an adopter test and validate the installation?  

      * Each of the installation guide mentioned above also includes validation steps. For example, once the Dataset is created, an adopter can test (details can be found [here](https://fluid-cloudnative.github.io/docs/tutorials/dataset-creation/accelerate-data-accessing-posix)):
          * The status of the Dataset via `kubectl get dataset <dataset-name> -n <namespace-name>`
          * Validate data access:

            ```bash
            kubectl exec <test-pod> -- ls /data
            ```
          * Monitor cache metrics:

            ```bash
            kubectl describe dataset llm-model-fluid  | grep -E 'Cache Capacity|Cached Percentage'
            Cache Capacity:     720.00GiB
            Cached Percentage:  100.0%
            ```

### Security

  * Please provide a link to the project’s cloud native [security self assessment](https://tag-security.cncf.io/community/assessments/): [security/self-assessment.md](https://github.com/fluid-cloudnative/fluid/blob/master/security/self-assessment.md)
  
  * Please review the [Cloud Native Security Tenets](https://github.com/cncf/tag-security/blob/main/security-whitepaper/secure-defaults-cloud-native-8.md) from TAG Security.  
    * How are you satisfying the tenets of cloud native security projects?  

      - Fluid utilizes Kubernetes-native constructs such as Custom Resource Definitions (CRDs) and Role-Based Access Control (RBAC). These allow for fine-grained permission control and secure management of data access and operations within Kubernetes clusters.
      - Network Policies are employed to manage and restrict communication between different components, enhancing security within the cluster.  
      - Fluid's default configurations adhere to Kubernetes security best practices, including running containers with non-root privileges and minimizing granted permissions.  
      - Insecure options (e.g., [useNodeAuthorization: false](https://github.com/fluid-cloudnative/fluid/blob/master/charts/fluid/fluid/values.yaml#L69)) require explicit configuration.
      - Exception handling from provided defaults is possible via modifying the resource definition or through configurations which are exposed in the configmap.  

    * Describe how each of the cloud native principles apply to your project.  

    **1. Containers as Fundamental Units**  
    - Fluid deploys all components (controllers, runtime engines, CSI drivers) as containerized microservices.  
    - Each distributed cache engine (Alluxio/JuiceFS/Vineyard) runs in isolated containers with framework-specific dependencies, ensuring consistent behavior across environments.  
    
    **2. Kubernetes-Native Orchestration**  
    - Fluid extends Kubernetes through CRDs (`Dataset`, `DataOperation`, `Runtime`) for declarative data orchestration.  
    - Deeply integrates with Kubernetes scheduling, storage (PVC/PV), and networking primitives to manage cache lifecycle:  
    **3. Microservices Architecture**  
    | **Component**          | **Responsibility**                     | **Decoupling Mechanism**         |  
    |-------------------------|----------------------------------------|----------------------------------|  
    | Controller              | CRD reconciliation                     | gRPC API to Runtimes             |  
    | Cache Runtime (e.g., Alluxio) | Data acceleration                    | Containerized engine plugins     |  
    | CSI Driver              | Persistent volume attachment           | Standard Kubernetes storage API  |  
    
    **4. Dynamic Autoscaling**  
    - Implements Kubernetes-native elasticity:  
      - **Horizontal Scaling**: Auto-add/remove cache engine replicas based on:  
        ```yaml
       apiVersion: autoscaling/v2beta2
       kind: HorizontalPodAutoscaler
       metadata:
         name: mydataset
       spec:
         scaleTargetRef:
           apiVersion: data.fluid.io/v1alpha1
           kind: AlluxioRuntime
           name: mydataset
         minReplicas: 1
         maxReplicas: 4
         metrics:
         - type: Object
           object:
             metric:
               name: capacity_used_rate
             describedObject:
               apiVersion: data.fluid.io/v1alpha1
               kind: Dataset
               name: mydataset
             target:
               type: Value
               value: "90"
        ```  

    **5. API-Driven Loose Coupling**  
    - **Declarative Data APIs**:  
      ```yaml
      apiVersion: data.fluid.io/v1alpha1
      kind: Dataload  # Preload without knowing storage location
      metadata:
        name: spark
      spec:
        dataset:
          name: spark
          namespace: default
      ```  
    - **Standardized Protocols**:  

      - FUSE for POSIX-compliant data access  
      - Prometheus metrics for observability  
      - S3/GCS/OSS Blob URIs for cloud storage integration  
       
    **6. Self-Healing Operations**  

    - Leverages Kubernetes control loops for:  
      - Automatic cache rebalancing after node failures  
      - Retrying failed data operations (prefetch/migration)  
      - Reconciliation of desired vs. actual dataset states  

  * How do you recommend users alter security defaults in order to "loosen" the security of the project? Please link to any documentation the project has written concerning these use cases.  

     * **defaults to secure node authorization** by setting `useNodeAuthorization: true` in its [values.yaml](https://github.com/fluid-cloudnative/fluid/blob/master/charts/fluid/fluid/values.yaml#L66C30-L69). This leverages Kubernetes [Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/) to restrict CSI Plugin permissions using kubelet's credentials.
     * To loosen this security setting (not recommended for production):
       1. Set `useNodeAuthorization: false` in Helm values:
          ```yaml
          csi:
            useNodeAuthorization: false # NOT RECOMMENDED
          ```
       2. This grants the `fluid-csi` Service Account broad permissions:
          ```yaml
          # Resulting RBAC permissions
          - apiGroups: [""]
            resources: ["nodes"]
            verbs: ["get", "patch"]  # Allows modification of any node
          ```
  
      * **Defaults to Network Isolation** `hostNetwork: false` ensures CSI pods run in isolated network namespaces.  
      *Loosening Risk*: If set to `true`, pods share node network stack → potential port conflicts and reduced network security.  
         ```yaml
         csi:
           config:
             hostNetwork: true  # NOT RECOMMENDED
         ```
  * Security Hygiene  

    * Please describe the frameworks, practices and procedures the project uses to maintain the basic health and security of the project.   

      * **Robust CI/CD Pipeline**: Automated unit, integration, and Kubernetes e2e tests block PR merges unless achieving 100% pass rate.  
      * **Vulnerability Scanning**: Container images scanned daily with Trivy/Grype, Go code analyzed by gosec, and dependencies monitored via Dependabot with auto-PRs for critical CVEs.  
      * **Static Code Analysis**: Code linting and static analysis tools are employed to enforce code quality and detect potential issues before they reach production.  
      * **Dependency Management**: The project uses lock files (e.g., pyproject.toml, go.mod) for reproducibility. Automated generation of Software Bill of Materials (SBOMs) is now integrated into the CI/CD pipeline for all container images and components, supporting supply chain security and transparency.  
      * **Secure Defaults**: Default Kubernetes manifests and Helm charts are configured with security best practices, such as running containers as non-root and enforcing least privilege.  
      * **Code Review Process**: All code changes require peer review and explicit approval before being merged into the master branch.  
      * **Secure Image Builds**: Image builds are validated through CI/CD checks to ensure they are secure, reproducible, and consistent.  
      * **Transparent Governance**: Biweekly public community meetings and Community channels, WeChat Group, DingTalk for raising and discussing security or health concerns.  
      * **Structured Releases**:Releases are managed with clear versioning and changelogs, ensuring traceability and transparency for all changes.  

    * Describe how the project has evaluated which features will be a security risk to users if they are not maintained by the project?  

      * Fluid identifies potential security risks by analyzing the attack surface, particularly focusing on features interacting with user-supplied code, custom containers, and external data sources.
      * Features are discussed and reviewed in the community, with security implications considered during design and code review processes.
      * The project ensures that dependencies are up-to-date and secure, mitigating risks associated with third-party libraries or tools.  

  * Cloud Native Threat Modeling  

    * Explain the least minimal privileges required by the project and reasons for additional privileges.  

      * Fluid requires broader RBAC permissions as it operates cluster-wide rather than being namespace-scoped. This is essential for managing resources across different namespaces.  
      * The Fluid controller needs permissions to watch and reconcile Fluid-specific custom resources Dataset, Runtime and DataOperation and manage associated Kubernetes objects like PersistentVolumes and PersistentVolumeClaims. These actions necessitate cluster-wide privileges.
      * Distributed Cache Pods run as non-root by default, without requiring host access or privileged escalation. The default service account injection is disabled by default as these pods do not need to communicate with the Kubernetes API Server. Secrets/ConfigMaps are mounted directly into Distributed Cache Pods only when explicitly needed, eliminating Fluid controller interpretation of sensitive data.   
      * If users deploy custom containers or runtimes, those may require additional permissions (e.g., access to GPUs, external storage, or specific system capabilities).
      * Even If users provide credentials for external model storage systems (e.g., S3, GCS or Azure Blob), Fluid controller  need no permission to access cluster wide Kubernetes Secrets, because they can be mounted into the Fluid Cache Pod as files.  

    * Describe how the project is handling certificate rotation and mitigates any issues with certificates.  

      **Automated Internal Certificate Rotation**: Fluid has its own certificate rotation mechanism. The CA root and the leaf certificates are both rotated automatically.  The logic is located in [source code](https://github.com/fluid-cloudnative/fluid/blob/master/pkg/ctrl/watch/manager.go#L185).  
      **External Storage Certificate Handling:**: For secure connections to S3/GCS/Azure Blob
      1. Custom CA bundles provided via Kubernetes Secrets:  

      ```yaml
      apiVersion: data.fluid.io/v1alpha1
      kind: Dataset
      metadata:
        name: demo
      spec:
        mounts:
          - mountPoint: s3://spark/fluid-data
            name: spark
            options:
              alluxio.underfs.s3.endpoint: http://{{$demo-minio-addr}}:9000
              alluxio.underfs.s3.disable.dns.buckets: "true"
              alluxio.underfs.s3.inherit.acl: "false"
            encryptOptions:
            - name: aws.accessKeyId
              valueFrom:
                secretKeyRef:
                  name: mysecret
                  key: aws.accessKeyId
      ```
      2. Dynamic certificate reload:  
         - FUSE Pod/Sidecar detect Secret changes via inotify.
         - Certificates reloaded within 5 seconds without pod restart.  

    * Describe how the project is following and implementing [secure software supply chain best practices](https://project.linuxfoundation.org/hubfs/CNCF\_SSCP\_v1.pdf) 

      * **Automated Testing and CI/CD Integration**: Fluid integrates Kubernetes e2e tests, unit tests, and integration tests into GitHub Actions CI/CD pipelines, blocking PR merges until 100% test pass rate is achieved.  
      * **Vulnerability Scanning**: Container images are scanned daily with Trivy and Grype, Go dependencies monitored via Dependabot with auto-PRs for CVSS ≥7 vulnerabilities, and Helm charts validated using Checkov.  
      * **Use of Lock Files**: Go dependencies are pinned via `go.mod` and `vendor/`.  
      * **DCO Check**: Committers are required to sign and comply with the Developer Certificate of Origin (DCO) to affirm the legitimacy and authorship of their contributions, and it's enforced via DCO GitHub Action on every PR.  
      * **Branch Protection**: `master` branch requires:  
        - maintainer/Approver approvals  
        - Successful e2e tests on Kubernetes v1.22 above  
        - DCO/license/secret scans  
      * **License Compliance**: Automated license compliance checks are now integrated into the CI/CD pipeline. All dependencies are scanned and validated for license compatibility as part of every pull request and release build, ensuring transparency and legal compliance.
      * **Peer Review**: Security-sensitive changes require approval from maintainers, with FUSE/CSI modules mandating penetration test reports.  


# Day 1 - Installation and Deployment Phase

### Project Installation and Configuration

  * Describe what project installation and configuration look like.
      Fluid exclusively uses Helm for production-grade installation and lifecycle management.

      A [quick start guide](https://fluid-cloudnative.github.io/docs/next/get-started/quick-start) is available to help users through the initial setup.

      Fluid can be further customized by following [Configuration Reference](https://fluid-cloudnative.github.io/docs/next/get-started/installation#advanced-configuration)

### Project Enablement and Rollback

  * How can this project be enabled or disabled in a live cluster? Please describe any downtime required of the control plane or nodes.  

  Fluid can be enabled or disabled in a live cluster without requiring downtime. To remove Fluid, simply follow the [Uninstall process documentation](https://fluid-cloudnative.github.io/docs/next/get-started/installation#uninstall-fluid). This process does not impact the control plane or nodes, ensuring uninterrupted operations.  

  * Describe how enabling the project changes any default behavior of the cluster or running workloads.  

  ### Zero Default Behavior Changes. 
  - Fluid does not modify existing PVCs, Pods, or APIs; it only adds CRDs like `Dataset` and `Runtime`.
  - The core scheduler, storage layer, and network policies remain unchanged.
  ### Explicit Use of Fluid. 
  - Acceleration is triggered only when Pods mount `Dataset` PVCs, requiring users to use Fluid PVCs.
  - Non-Fluid volumes continue with standard Kubernetes behavior.
  - FUSE sidecars are injected into Pods with the specific label `serverless.fluid.io/inject: "true"`.
  ### Resource Isolation. 
  - Control plane components are isolated within the `fluid-system` namespace.
  - Cache engines are deployed to dedicated node pools.  

  * Describe how the project tests enablement and disablement.  

  Installing and uninstalling Fluid is part of the [Fluid end-to-end testing suite](https://github.com/fluid-cloudnative/fluid/blob/2ded0738d3e9407822f9b4924114c2653da0ffeb/.github/workflows/kind-e2e.yml#L67).  

  * How does the project clean up any resources created, including CRDs?  

  Check the [Uninstall process documentation](https://fluid-cloudnative.github.io/docs/next/get-started/installation#uninstall-fluid) instructions.

### Rollout, Upgrade and Rollback Planning

  * How does the project intend to provide and maintain compatibility with infrastructure and orchestration management tools like Kubernetes and with what frequency?   

  Fluid maintains Kubernetes compatibility by following standard Kubernetes patterns.The supported versions are evaluated and upgraded on every official release.  

  * How can a rollout or rollback fail? Describe any impact to already running workloads.  

  If a rollout or rollback fails, datasets created before the control plane upgrade will continue operating with their original runtime versions. Existing cache runtimes and workload pods remain unaffected. Only operations like Dataload and Cache Scale out/in are impacted. Therefore, already running workloads will not be affected if a rollout fails.  

  * Describe any specific metrics that should inform a rollback.  

  Monitor dataset and runtime status, as well as workload PVC mounting and data access, to ensure the cache system functions correctly and determine if a rollback is necessary.  

  * Explain how upgrades and rollbacks were tested and how the upgrade-\>downgrade-\>upgrade path was tested.  

  At the time of writing, downgrade scenarios have been tested manually Upgrade path testing for each release is conducted using our [end-to-end testing suite](https://github.com/fluid-cloudnative/fluid/blob/2ded0738d3e9407822f9b4924114c2653da0ffeb/.github/workflows/kind-e2e.yml).  

  * Explain how the project informs users of deprecations and removals of features and APIs.  

  All API changes in the Fluid project are designed to be backward compatible. Deprecations and removals are announced in the release notes and on the official website, ensuring users are informed of any changes.  

  * Explain how the project permits utilization of alpha and beta capabilities as part of a rollout.  

  The Fluid project allows users to utilize alpha and beta capabilities during rollouts through the following methods:  

  - **Feature Flags**: Users can enable or disable alpha and beta features using feature flags, ensuring new capabilities can be tested without impacting the stable environment. For more details, refer to the [Feature Gates](https://github.com/fluid-cloudnative/fluid/blob/master/charts/fluid/fluid/values.yaml#L55).  
  - **Documentation**: Comprehensive documentation is available to guide users on accessing and using these [experimental features](https://github.com/fluid-cloudnative/fluid/blob/2ded0738d3e9407822f9b4924114c2653da0ffeb/docs/en/samples/fuse_recover.md).  








    


