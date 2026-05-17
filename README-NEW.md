# Cymbal Eats: Project Overview for Trainee Software Engineers

Welcome to the **Cymbal Eats** project! This document provides a comprehensive, step-by-step breakdown of the project's problem statement, our technical approach, the underlying architecture, and the design patterns we implemented. It is designed to help you, as a new engineer, understand *what* we built and *why* we built it this way.

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

## 4. Design Patterns Used

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

## 5. Deploy on Windows Laptop

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

## Summary for Trainees

When exploring the `cymbal-eats` codebase, start by looking at a specific feature (like placing an order). Trace the request from the `customer-ui` into the `order-service`, and see how it interacts with Firestore. Notice how the code doesn't worry about servers or scaling—that is handled by the deployment platform (Cloud Run). Understand that the complexity here lies in the distributed nature of the system (how services talk to each other and manage data) rather than complex monolithic code.
