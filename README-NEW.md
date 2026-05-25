# Cymbal Eats: Project Overview for Trainee Software Engineers

Welcome to the **Cymbal Eats** project! This document provides a comprehensive, step-by-step breakdown of the project's problem statement, our technical approach, the underlying architecture, and the design patterns we implemented. It is designed to help you, as a new engineer, understand *what* we built and *why* we built it this way.

---

## Table of Contents
*   [1. Problem Statement](#1-problem-statement)
*   [2. Approach](#2-approach)
*   [3. Architecture Deep-Dive](#3-architecture-deep-dive)
*   [4. Provisioning & Deployment: Scripts, Tools, and GCP Services](#4-provisioning--deployment-scripts-tools-and-gcp-services)
    *   [4.1 Orchestration via setup.sh](#41-orchestration-via-setupsh)
    *   [4.2 Installed & Utilized Tools](#42-installed--utilized-tools)
    *   [4.3 Google Cloud Platform (GCP) Services](#43-google-cloud-platform-gcp-services)
*   [5. Vue.js Deep-Dive: Understanding OrderStatusPage.vue](#5-vuejs-deep-dive-understanding-orderstatuspagevue)
    *   [5.1 Anatomy of OrderStatusPage.vue](#51-anatomy-of-orderstatuspagevue)
    *   [5.2 Core Vue Concepts & How Vue Works](#52-core-vue-concepts--how-vue-works)
*   [6. Cloud Functions Deep-Dive: Understanding process-thumbnails](#6-cloud-functions-deep-dive-understanding-process-thumbnails)
    *   [6.1 Flow of cloud-functions/thumbnail/index.js](#61-flow-of-cloud-functionsthumbnailindexjs)
    *   [6.2 Core Concepts: Event-Driven Architecture, Concurrency, and Ephemeral Storage](#62-core-concepts-event-driven-architecture-concurrency-and-ephemeral-storage)
*   [7. Design Patterns Used](#7-design-patterns-used)
*   [8. Deploy on Windows Laptop](#8-deploy-on-windows-laptop)
*   [9. Summary for Trainees](#9-summary-for-trainees)

---

## 1. Problem Statement

Cymbal Eats is a conceptual food delivery and restaurant management system. The business needed a modern, highly scalable application capable of handling varied and unpredictable traffic (like lunch/dinner spikes). A traditional monolithic application would struggle to scale specific parts of the system independently (e.g., scaling the order taking system without scaling the menu management system). Additionally, different business domains have distinct data requirements—some need strict consistency, while others need flexible schemas or extreme read/write throughput. 

**Goals:**
*   **Scalability:** Auto-scale components independently based on demand.
*   **Reduced Operational Overhead:** Minimize infrastructure management.
*   **Flexibility:** Use the right tools and databases for specific domain problems.
*   **Security:** Ensure internal portals are strictly accessible only by authorized employees.

---

## 2. Approach

To solve these challenges, we adopted a **Cloud-Native, Serverless Microservices** approach built on Google Cloud Platform (GCP). 

**Key Technical Decisions & Reasons:**
*   **Serverless Compute (Cloud Run):** We chose Cloud Run to host our microservices. It abstracts away server management, scales from zero to N instances instantly based on HTTP traffic, and supports containers, which means we can write services in any language (Java, Go, Node.js, etc.).
*   **Polyglot Persistence:** Instead of forcing all data into a single database, we chose specific GCP databases tailored to the needs of each microservice. This ensures optimal performance and cost-efficiency.
*   **Managed Services:** We relied heavily on fully managed services (like Cloud Spanner, AlloyDB, Cloud SQL, and Firestore) to eliminate the need for DBA (Database Administrator) tasks.
*   **Identity-Aware Proxy (IAP):** We implemented IAP for the Employee Portal to provide a Zero Trust security model without building custom login systems for internal tools.

---

## 3. Architecture Deep-Dive

The architecture consists of multiple independent services, each responsible for a distinct business capability. 

### Frontend Applications (Vue.js)
*   **`customer-ui`:** The public-facing web application where customers log in, view the menu, place orders, and check their rewards. It is built as a Single Page Application (SPA) using Vue.js and deployed on Cloud Run.
*   **`employee-ui`:** The internal portal used by staff to process orders, update inventory, and manage menu items. It is secured by **Identity-Aware Proxy (IAP)**, meaning only authenticated corporate users can access it.

### Backend Microservices
*   **`menu-service` (Java / Spring Boot):** 
    *   **Role:** Manages the catalog of food items.
    *   **Database:** **Cloud SQL (PostgreSQL)**. Relational databases are perfect for structured catalog data with relationships.
*   **`inventory-service` (Go):** 
    *   **Role:** Tracks the real-time availability of menu items.
    *   **Database:** **Cloud Spanner**. Inventory requires high availability, global scale, and strict transactional consistency to prevent double-selling. Spanner provides this at a massive scale. Go was chosen for its fast startup times and low resource footprint.
*   **`order-service` (Node.js):** 
    *   **Role:** Handles customer order ingestion and processing.
    *   **Database:** **Firestore (NoSQL)**. Orders often have flexible structures (customizations, special instructions). A document database like Firestore is ideal for this and supports rapid read/writes.
*   **`customer-service` (Java):** 
    *   **Role:** Manages customer profiles and the rewards system.
    *   **Database:** **AlloyDB**. A highly performant, PostgreSQL-compatible database designed for demanding enterprise workloads.
    *   **Orchestration:** Uses **Google Cloud Workflows** to manage the multi-step "Rewards" business logic asynchronously.
*   **`partner-registration-service` (Node.js):**
    *   **Role:** Onboards new delivery and restaurant partners.
    *   **Database:** **Firestore**.

### Event-Driven & Background Processes
*   **`cloud-functions` (Node.js):** 
    *   **Role:** An event-driven function that processes uploaded images for menu items. When an employee uploads an image to Cloud Storage, this function is triggered to generate a thumbnail.
    *   **Reason:** Heavy tasks like image manipulation shouldn't block the HTTP request thread of the Employee UI or Menu Service. Offloading it to an asynchronous Cloud Function keeps the system responsive.
*   **`cleanup-service` (Shell script):** 
    *   **Role:** A scheduled job that periodically runs to clean up menu items that are in a "Failed" status. 

---

## 4. Provisioning & Deployment: Scripts, Tools, and GCP Services

To deploy the entire **Cymbal Eats** ecosystem, the project uses a series of bash scripts that automate infrastructure provisioning, container compilation, and service deployment.

### 4.1 Orchestration via `setup.sh`
The root-level [setup.sh](file:///c:/Data/ShareDataOutside/CanProjects/AI_projects/BuildGoogleAI/cymbal-eats/setup.sh) acts as the central orchestrator. It first sources [config-env.sh](file:///c:/Data/ShareDataOutside/CanProjects/AI_projects/BuildGoogleAI/cymbal-eats/config-env.sh) to set global environment variables (e.g., project ID, region, bucket names, topic IDs) and enable the required GCP APIs. 

It then sequentially invokes directory-specific setup scripts to provision and deploy each microservice:
1.  **`menu-service`**: Connects VPC Peering for Service Networking, provisions a private Cloud SQL (PostgreSQL) database, builds the Java app using Maven, tags/pushes the Docker container, deploys it to Cloud Run with a Serverless VPC Access connector, and seeds initial menu items using `curl`.
2.  **`inventory-service`**: Provisions a Cloud Spanner instance and database, assigns IAM Spanner Admin permissions, and deploys the Go microservice via a source-based Cloud Run deployment.
3.  **`order-service`**: Provisions Firestore in Native mode, downloads the Firebase CLI, deploys security rules/indexes, configures Pub/Sub topics, and deploys the Node.js service to Cloud Run. It also registers a Pub/Sub push subscription `order-points-push-sub` to push order events back to the order service.
4.  **`cloud-functions`**: Provisions GCP buckets for original images and thumbnails (making them public), creates a custom Service Account, deploys the event-driven `process-thumbnails` function (2nd gen on Cloud Run), and configures an Eventarc trigger listening for finalized GCS uploads.
5.  **`cleanup-service`**: Creates an Artifact Registry Docker repository, triggers Cloud Build to compile a Docker container, creates a Cloud Run Job to clean up old failed menu items, and creates a Cloud Scheduler cron job to execute it daily at midnight.
6.  **`customer-service`**: Connects VPC Peering, provisions a private AlloyDB cluster and primary database instance, deploys a Cloud Run Job to run schema migrations, compiles the Java application, deploys the customer-service on Cloud Run (with internal-only ingress), deploys a rewards workflow via Google Cloud Workflows, and creates an Eventarc trigger linking Pub/Sub order events to the rewards workflow.
7.  **`employee-ui`**: Installs Node/NVM, compiles the Quasar (Vue.js) Single Page Application, bundles it inside a Cloud Run container using source-based deployment, and configures permissions for uploading menu images to GCS.
8.  **`customer-ui`**: Registers the web application with Firebase, retrieves Web SDK configuration dynamically, compiles the front-end SPA via Quasar, and deploys it to Cloud Run.

---

### 4.2 Installed & Utilized Tools
The setup scripts dynamically install, configure, or use several developer and system CLI tools:
*   **Firebase CLI (`firebase-tools`)**: Installed via curl; manages Firebase web apps, SDK configurations, and deploys Firestore rules/indexes.
*   **Node Version Manager (NVM) & Node.js (`v16.4.0`)**: Automatically fetched and activated to compile and build front-end web applications.
*   **Quasar CLI (`@quasar/cli`)**: An enterprise-grade Vue.js framework tool installed globally to clean, compile, and package front-end single-page application builds.
*   **envsub**: Used to perform environment variable substitution in `.env.tmpl` files to dynamically inject API keys, auth domains, and service endpoints.
*   **Maven Wrapper (`./mvnw`)**: Runs local builds of Java Spring Boot microservices, downloading required dependencies and compiling JARs.
*   **Docker**: Used by several microservices to build containerized JVM images from local Dockerfiles before pushing to GCP registries.
*   **gcloud CLI**: The primary driver for authenticating, enabling APIs, provisioning resources, managing IAM roles, and deploying serverless applications.
*   **jq**: A lightweight command-line JSON processor used extensively to parse the output of `gcloud` and `firebase` commands to extract URLs, connection strings, IPs, and keys.
*   **curl**: Downloads CLI installation packages, tests endpoints, and issues REST requests to seed initial database contents.

---

### 4.3 Google Cloud Platform (GCP) Services
The microservices architecture is built on a modern, decoupled suite of fully managed GCP services:
*   **Cloud Run (Services & Jobs)**: Used to host the microservices and UI portals, dynamically scaling compute from zero. Also runs serverless database migrations and cleanup scripts as Cloud Run Jobs.
*   **AlloyDB**: High-performance, PostgreSQL-compatible enterprise database powering the customer profile and rewards system.
*   **Cloud Spanner**: Globally scalable, highly available transactional database tracking inventory items without risking double-selling.
*   **Cloud SQL (PostgreSQL)**: Fully managed relational database engine storing structured menu catalogs.
*   **Firestore**: A serverless, flexible NoSQL document database hosting custom order records.
*   **Cloud Functions (2nd Gen)**: Lightweight, event-driven Node.js function triggered automatically to process and generate menu item image thumbnails.
*   **Cloud Storage (GCS)**: Scalable object storage holding uploaded food images and optimized thumbnail files.
*   **Cloud Pub/Sub**: High-performance messaging system facilitating asynchronous communication between services (such as publishing orders to trigger rewards processing).
*   **Google Cloud Workflows**: Serverless workflow orchestrator coordinating complex, multi-step customer rewards calculations.
*   **Eventarc**: Serverless event routing system triggering Cloud Functions from Cloud Storage events and Google Cloud Workflows from Pub/Sub topics.
*   **Cloud Scheduler**: Fully managed cron service orchestrating the daily execution of the cleanup Cloud Run job.
*   **Serverless VPC Access (VPC Connector)**: Safely bridges serverless Cloud Run containers into the VPC default network to privately access databases without public IP exposure.
*   **Service Networking & VPC Peering**: Establishes private service connections within the VPC to secure AlloyDB and Cloud SQL database paths.
*   **Artifact Registry & Container Registry**: Secure, centralized Docker image hosting for all custom-built container files.
*   **Cloud Build**: Manages remote container compilations for source-based Cloud Run deployments and job builders.
*   **Identity-Aware Proxy (IAP)**: Zero-Trust identity manager protecting internal endpoints such as the Employee UI.

---

## 5. Vue.js Deep-Dive: Understanding OrderStatusPage.vue

Front-end components in the Cymbal Eats architecture are structured as modern, reactive, client-side applications. A prime example is the [OrderStatusPage.vue](file:///c:/Data/ShareDataOutside/CanProjects/AI_projects/BuildGoogleAI/cymbal-eats/customer-ui/src/pages/OrderStatusPage.vue) component, which tracks order statuses in real-time.

### 5.1 Anatomy of `OrderStatusPage.vue`
The component consists of three key architectural blocks structured in a Single File Component (SFC) format:

1.  **Structure (`<template>`)**:
    *   Uses **Quasar UI Components** (`<q-layout>`, `<q-header>`, `<q-page-container>`) to build a responsive app container.
    *   Uses dynamic `<Toolbar/>` and `<OrderView/>` child components.
    *   Passes variables and configurations down to child components using Vue’s custom property binding (`:items="orderItems"` and `:allowdelete="false"`).
    *   Renders order information cleanly with fields bound to reactive variables:
        ```html
        <q-input filled v-model="orderNumber" :readonly="true" />
        <q-input filled v-model="status" :readonly="true" />
        ```

2.  **Logic (`<script setup>`)**:
    *   Employs the modern **Vue 3 `<script setup>` Composition API**, which is a compiler-time sugar making code cleaner and more readable by eliminating boilerplate setup code.
    *   Declares reactivity using `ref()` for three local variables:
        *   `orderNumber`: Initialized dynamically from route parameters (`route.params.orderNumber`).
        *   `orderItems`: Initialized as an empty reactive array.
        *   `status`: Initialized as an empty reactive string.
    *   Injects ecosystem utilities:
        *   `useStore()`: Interfaces with the global Vuex state management.
        *   `useRouter()` and `useRoute()`: Accesses the application's route parameters.

3.  **Real-time Synchronization (Firestore Integration)**:
    *   Triggers logic during the lifecycle hook `onMounted`.
    *   Establishes an active subscription using Firestore’s `onSnapshot` on the specific document inside the `orders` collection (`doc(db, 'orders', orderNumber.value)`).
    *   Whenever order data changes in the cloud (e.g., when `order-service` receives points or an employee updates the order status in the `employee-ui`), Firestore fires the snapshot callback.
    *   The callback updates the reactive local state:
        ```javascript
        const order = doc.data();
        orderItems.value = order.orderItems;
        status.value = order.status;
        ```

---

### 5.2 Core Vue Concepts & How Vue Works
Understanding how Vue manages this interaction is fundamental to developer training:

*   **1. The Reactivity System (Vue 3 Proxies)**:
    Under the hood, Vue 3 wraps reactive declarations like `ref()` in JavaScript `Proxy` objects.
    *   **Dependency Tracking (Getter)**: When the page template compiles, it reads the reactive values `status` and `orderNumber`. Vue tracks that the UI depends on these variables.
    *   **Change Notification (Setter)**: When Firestore retrieves new data and modifies `status.value = order.status`, the proxy's setter intercepts the change and notifies the scheduler.
    *   **Re-rendering (Virtual DOM)**: Vue schedules an asynchronous patch of the Virtual DOM, compares the differences, and updates only the text node inside the specific `<q-input>` representing the status. This keeps the page updated instantly without full browser refreshes.

*   **2. Component Lifecycle Hooks (`onMounted`)**:
    Vue lifecycle hooks allow developers to run code at specific stages. `onMounted` fires precisely after the component is compiled and attached to the physical DOM tree. This is the optimal time to establish external connections (like the live Firestore snapshot listener), ensuring no resources are wasted before the UI is actually visible to the user.

*   **3. Unidirectional Data Flow & Props**:
    Vue enforces a strict top-down parent-to-child data flow. The parent component (`OrderStatusPage`) passes its local reactive `orderItems` down to the child component (`OrderView`) using the `:items="orderItems"` prop binding.
    *   Whenever `orderItems.value` updates locally inside the parent snapshot listener, Vue propagates the new array down to `<OrderView>`, triggering an isolated update inside the child component.
    *   By passing `:allowdelete="false"`, the parent controls the capabilities of the child view without duplicating logic.

*   **4. Single File Components (SFC)**:
    Vue files (`.vue`) bundle the markup (`<template>`), behavior (`<script>`), and design (`<style>`) into a single, cohesive module. During compilation, these are transpiled into highly optimized render functions, ensuring high performance.

---

## 6. Cloud Functions Deep-Dive: Understanding process-thumbnails

To handle asynchronous heavy lifting like image scaling and AI-driven content verification, the Cymbal Eats architecture uses Google Cloud Functions. Specifically, the [index.js](file:///c:/Data/ShareDataOutside/CanProjects/AI_projects/BuildGoogleAI/cymbal-eats/cloud-functions/thumbnail/index.js) function processes raw image uploads in the background.

### 6.1 Flow of `cloud-functions/thumbnail/index.js`
The script registers a CloudEvent handler named `process-thumbnails` using the GCP Functions Framework. The execution proceeds as follows:

1.  **Early Validation & Database ID Extraction**:
    *   Extracts the item ID by parsing the base filename:
        ```javascript
        const itemID = parseInt(path.parse(file.name).name);
        ```
        For example, an upload named `15.png` maps directly to the database record with primary key `15`. If the name is non-numeric, the execution exits early before invoking external APIs, preventing billing waste.

2.  **Parallel Execution & Pipeline Setup**:
    *   Instantiates clients for **Cloud Storage** and **Cloud Vision API**.
    *   Initiates an asynchronous request to Google Cloud Vision API (`client.annotateImage(visionRequest)`) to perform **Label Detection** directly on the raw file in GCS (`gs://${file.bucket}/${file.name}`).
    *   **Crucial Performance Optimization**: The Vision call is launched immediately as a JavaScript Promise (`visionPromise`). While the network request travels to the AI backend, the Cloud Function continues execution locally (creating directories and downloading the file) to process tasks in parallel.

3.  **Recursive Local Workspace Setup**:
    *   Generates target paths and creates ephemeral temporary directories under `/tmp` recursively:
        ```javascript
        fs.mkdirSync(path.dirname(originalFile), { recursive: true });
        ```
        Using `{ recursive: true }` ensures that nested GCS paths (e.g. `categories/desserts/15.jpg`) do not cause file download crashes.

4.  **Download File**:
    *   Downloads the original image from the source GCS bucket to the `/tmp/original/` workspace and retrieves its public URL.

5.  **Image Scaling via ImageMagick**:
    *   Promisifies ImageMagick’s legacy `crop` function using `bluebird`.
    *   Crops and resizes the local raw file down to a square `400x400` thumbnail saved in `/tmp/thumbnail/${file.name}`.

6.  **Thumbnail Storage**:
    *   Uploads the compressed thumbnail to the dedicated `BUCKET_THUMBNAILS` and stores its public URL.

7.  **Content Moderation & Approval (with Safety Check)**:
    *   Awaits the pending `visionPromise` to receive the classification labels.
    *   Performs a safety check on `labelAnnotations` to prevent null pointer exceptions if the image lacks labels:
        ```javascript
        const annotations = visionResponse[0] && visionResponse[0].labelAnnotations;
        ```
    *   Loops through returned classifications. If the Vision API detects the label `"Food"`, it flags the item as verified (`"Ready"`); otherwise, it leaves the verification status as `"Failed"`.

8.  **REST Callback API Notification**:
    *   Creates an Axios client pointing to the backend `menu-service` REST URL.
    *   Issues an HTTP GET to `/menu/${itemID}` to load the existing item metadata (name, price, spice level).
    *   Issues an HTTP PUT request updating the menu item with the new `itemImageURL`, `itemThumbnailURL`, and the AI-computed validation `status` ("Ready" or "Failed").

9.  **Ephemeral File Cleanup**:
    *   Guarantees deletion of the temporary local files inside a `finally` block:
        ```javascript
        if (fs.existsSync(originalFile)) fs.unlinkSync(originalFile);
        ```
        This prevents container disk full (OOM) memory exhaustion across warm execution instances.

---

### 6.2 Core Concepts: Event-Driven Architecture, Concurrency, and Ephemeral Storage

*   **1. Event-Driven Architecture (EDA) & Eventarc**:
    By decoupling the UI upload step from thumbnail processing, the front-end remains highly responsive. The Employee UI uploads the file to Cloud Storage and immediately returns a success message to the operator. Eventarc intercepts the `google.cloud.storage.object.v1.finalized` storage event and asynchronously triggers the Cloud Run container running the function, keeping the user experience snappy and decoupled.

*   **2. Concurrency and Asynchronous IO**:
    The Cloud Function maximizes CPU utilization through concurrent IO. Rather than downloading, cropping, and *then* calling the Vision API sequentially, it executes the heavy machine learning labeling requests in parallel with the local download/resize tasks. Only at the very end does it wait for the API promise to resolve, cutting total latency by 30-50%.

*   **3. Ephemeral Disk Space (`/tmp`)**:
    Cloud Run and Cloud Functions deploy a RAM-based overlay directory at `/tmp`. The function downloads the files there to write changes quickly. However, this storage is ephemeral: when the container scales down or restarts, `/tmp` is wiped clean. Developers must avoid writing state here that needs to persist across executions.

*   **4. Automated Verification Loop**:
    This is an automated content moderation pattern. It uses serverless AI inference (Google Cloud Vision API) as a gatekeeper before public catalog inclusion, ensuring that only valid food items are marked as "Ready" for customers.

---

## 7. Design Patterns Used

As a trainee, you will recognize several industry-standard design patterns implemented in this codebase:

1.  **Microservices Architecture Pattern:**
    *   *Implementation:* The system is decomposed into small, independent services (`menu-service`, `order-service`, etc.) organized around business capabilities.
    *   *Benefit:* Enables independent deployment, scaling, and technology choices per service.

2.  **Polyglot Persistence Pattern:**
    *   *Implementation:* Using Cloud SQL (PostgreSQL) for menus, Spanner for inventory, Firestore for orders, and AlloyDB for customers.
    *   *Benefit:* Avoids the "one size fits all" database anti-pattern. Each service uses the data store most appropriate for its data access patterns.

3.  **Event-Driven Architecture (EDA) Pattern:**
    *   *Implementation:* The image processing workflow using Google Cloud Functions triggered by Cloud Storage events.
    *   *Benefit:* Decouples services. The uploader doesn't need to know *how* or *when* the thumbnail is generated, it just happens in the background.

4.  **Orchestration / Workflow Pattern:**
    *   *Implementation:* The `customer-service` uses Google Cloud Workflows to handle complex, multi-step business logic for customer rewards.
    *   *Benefit:* Centralizes the flow of a complex process, making it easier to monitor, retry on failures, and maintain compared to point-to-point microservice communication.

5.  **Stateless Services Pattern:**
    *   *Implementation:* All Cloud Run services maintain no local state. Any required state is pushed to external databases (Firestore, Spanner, etc.).
    *   *Benefit:* Allows Cloud Run to seamlessly spin up or tear down containers to handle fluctuating traffic without losing data.

6.  **Backend for Frontend (BFF) / API Gateway-like interaction:**
    *   *Implementation:* The UIs (`customer-ui`, `employee-ui`) interact with specific backend service REST APIs to aggregate data for the user.

---

## 8. Deploy on Windows Laptop

Since Cymbal Eats heavily relies on Google Cloud Platform and uses bash scripts (`.sh`) for provisioning and deployment, you cannot simply run a single local command like `docker-compose up` natively on Windows command prompt. 

To deploy and test this project from a Windows machine, follow these steps:

### Prerequisites
1.  **Windows Subsystem for Linux (WSL 2)** or **Git Bash**: You need a bash environment to run the deployment scripts. WSL 2 with an Ubuntu distribution is highly recommended.
2.  **Google Cloud CLI (`gcloud`)**: Install the gcloud CLI inside your WSL environment (or ensure the Windows installation is available in your bash path).
3.  **Dependencies**: Depending on the service you are building locally, ensure you have Java (Maven), Node.js, and Go installed in your WSL environment.

### Deployment Steps
1.  **Open your WSL terminal (or Git Bash)**.
2.  **Authenticate with Google Cloud**:
    ```bash
    gcloud auth login
    gcloud auth application-default login
    ```
3.  **Clone the Repository**:
    ```bash
    git clone https://github.com/GoogleCloudPlatform/cymbal-eats.git
    cd cymbal-eats
    ```
4.  **Configure Environment**: 
    Update the `org-admin-setup.sh` and `config-env.sh` scripts with your specific GCP Organization ID, Billing Account, and desired region, as instructed in the main documentation.
5.  **Run the Setup Scripts**:
    ```bash
    # Create the project and assign roles
    ./org-admin-setup.sh
    
    # Deploy all microservices to your GCP project
    ./setup.sh
    ```
    *Note: The `setup.sh` script loops through all subdirectories and runs their individual deployment scripts, provisioning databases and deploying Cloud Run services on your Google Cloud Project.*

---

## 9. Summary for Trainees

When exploring the `cymbal-eats` codebase, start by looking at a specific feature (like placing an order). Trace the request from the `customer-ui` into the `order-service`, and see how it interacts with Firestore. Notice how the code doesn't worry about servers or scaling—that is handled by the deployment platform (Cloud Run). Understand that the complexity here lies in the distributed nature of the system (how services talk to each other and manage data) rather than complex monolithic code.
