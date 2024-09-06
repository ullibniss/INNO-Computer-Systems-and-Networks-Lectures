# Computer Systems and Networks

## Lecture 2 Exercises. Done by Joel Okore and Fedorov Alexey. Group: [M24-SNE-01]

## Exercise

Scenario: You are tasked with designing the architecture for a simple e-commerce website
that sells books. The system should support the following functionalities:

1. **User Management**: Allow users to register, log in, and manage their profiles.
2. **Product Catalog**: Display a list of books, including details such as title, author, price,
and availability.
3. **Shopping Cart**: Enable users to add books to a shopping cart, view the cart, and
proceed to checkout.
4. **Order Management**: Process user orders and manage the order history.
5. **Payment Processing**: Integrate with a payment gateway to handle payments.
6. **Administration Panel**: Allow administrators to manage the product catalog, view orders,
and update inventory.

**Expected Output:**
- A list of core components (e.g., User Service, Product Catalog Service, Order Service,
Payment Service, etc.).
- A description of how these components interact (e.g., the User Service interacts with the
Order Service to retrieve a user's order history).
- A basic system architecture diagram (e.g., a layered architecture with a front-end, back-end
services, and a database layer).
- A brief discussion on non-functional requirements.


## Components

There are a bunch on components in system.

### Services

- **User manager** service - provides auth service and managing users profiles
- **Shop** service - manages business logic related to shop. Also provides administration fucntions.
- **Order manager** service - manages cart, ordering and payment logic.
- **Frontend** service - implements frontend.

### Databases

- **Postgresql** - SQL database server for services needs.
- **S3** - file storage for ebooks.
 
### Infrastucture

- **Prometheus**, **Grafana** - monitoring system for reliability.
- **Nginx** - proxy server for frontend and border.
- **Yandex Cloud** - cloud computing for our project.

## Interaction

### Hardware infrastracture

- We decided to use **Yandex Cloud** for computing and storage purposes instead of a local server infrastracture.

- The following cloud services takes a part in infrastructure:
    - **Cloud Computing** - there are 5 servers in use.
    -  **S3** - file storage for ebooks.

### Network traffic

Server's displacement is within the private network. There cannot be accessed directly from internet. To provide it securely, we needed to add L7 proxy server, that will routes requests to the services. This is done using the **Nginx** server.

Nginx serves as an intermediary between the internet (users) and the private network.

Web server will also apply SSL/TLS connection with our services. Certificates are signed by Let's Encrypt.

### Services

#### User manager

- Provides authentication API, that includes register, login and token verification.
- Stores user's profile data.

#### Shop service

- Implements shop's business logic.
- Provides the following fucntionalities:
    - Shopping catalog
    - Detailed information about product (books)
- Interacts with database & s3 to store and access shopping information.
- Interacts with order manager to add products into the cart.

#### Order manager

- Manages user's shopping cart.
- Regulate ordering flow.
- Interacts with database to store and access ordering information.
- Interact with User manager to make decision regarding user info, for example, to make sales.

#### Frontend

- Provide interactive and beautiful web interface for users.
- Interacts with above services by REST API.

### System monitor

#### Prometeus

Collects and Stores hardware, services and business metrics. 

Uses pull model:
  - Node exporters retreive metrics from endpoint user and services.
  - Prometheus collates data using http.

#### Grafana

Visualization system that analyzes data from prometheus and visually represent it on dashboard, making use of graphs and charts.

## Architecture diagram

![image](https://github.com/user-attachments/assets/fa739648-d921-4bbf-92ef-73a0d9201009)

## Non fuctional requirements

1. **Scalability** - The system must be able to handle increasing loads by dynamically scaling up or down based on traffic or user demand.

2. **Replication**
  - **PostgreSQL** databases should support replication to ensure data availability and integrity in case of hardware or service failures. A master-slave or master-master replication model must be employed to minimize downtime and data loss.
  - **S3** storage should replicate files across multiple availability zones in Yandex Cloud to prevent data loss due to storage failure.
  - Prometheus should replicate monitoring data to ensure system monitoring continues in case of a server failure.

3. **Fault Tolerance**
  - The system must remain operational even when individual components fail. This should be achieved through redundancy and high-availability architecture.
  - Nginx should be configured to redirect traffic between multiple instances of backend services in case one instance becomes unresponsive.
  - **Prometheus** and **Grafana** should be deployed in a fault-tolerant configuration to ensure continuous monitoring and alerting even if a node fails.

4. **Security**
- **Authentication and Authorization** - the **User Manager** service must use secure protocols (e.g., OAuth2) to manage user authentication and ensure token validation is implemented securely.
- **Access Control** - аccess to services inside the private network should be restricted and only accessible via the Nginx proxy.

5. **Resilience** - the system should automatically recover from service disruptions. For example, if a service crashes, it should be automatically restarted using container orchestration (e.g., Kubernetes) or cloud-native tools provided by **Yandex Cloud**.

These non-functional requirements help ensure the system operates efficiently, securely, and reliably while meeting business and user demands.

