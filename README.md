# BankApp

This repository contains the configuration for deploying the `bankapp` Spring Boot application and its MySQL database using Kubernetes in the `test-ns` namespace.

## Environment Variables

### Application Container (`bankapp`)

To run the `bankapp` application, configure the following environment variables for the database connection:

| Variable Name                | Description                                                                 | Example Value                                                                 |
|------------------------------|-----------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| `SPRING_DATASOURCE_URL`      | JDBC URL for the MySQL database                                             | `jdbc:mysql://mysql-service:3306/bankappdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true` |
| `SPRING_DATASOURCE_USERNAME` | Username for the MySQL database                                             | `root`                                                                        |
| `SPRING_DATASOURCE_PASSWORD` | Password for the MySQL database (use Kubernetes secrets for production)      | (Set via `mysql-secret` in Kubernetes or a secure method for local development) |

### MySQL Container

To run the MySQL database, configure the following environment variables:

| Variable Name         | Description                                   | Source                     |
|-----------------------|-----------------------------------------------|----------------------------|
| `MYSQL_ROOT_PASSWORD` | Password for the MySQL `root` user            | `mysql-secret` (key: `MYSQL_ROOT_PASSWORD`) |
| `MYSQL_DATABASE`      | Name of the database to create                | `mysql-configmap` (key: `MYSQL_DATABASE`) |

## Setup Instructions

### Prerequisites
- A Kubernetes cluster with `kubectl` configured.
- The `test-ns` namespace created: `kubectl create namespace test-ns`.

### Creating Secrets and ConfigMap
1. **Create the MySQL Secret**:
   ```bash
   kubectl create secret generic mysql-secret \
     --from-literal=MYSQL_ROOT_PASSWORD=your_secure_password \
     -n test-ns
   ```

2. **Create the MySQL ConfigMap**:
   ```bash
   kubectl create configmap mysql-configmap \
     --from-literal=MYSQL_DATABASE=bankappdb \
     -n test-ns
   ```

### Deploying the Application
1. **Deploy MySQL**:
   Apply the MySQL `Deployment` and `Service`:
   ```bash
   kubectl apply -f k8s/mysql-deployment.yaml -n test-ns
   kubectl apply -f k8s/mysql-service.yaml -n test-ns
   ```

2. **Deploy BankApp**:
   Apply the `bankapp` `Deployment` and `Service`:
   ```bash
   kubectl apply -f k8s/bankapp-dep.yaml -n test-ns
   kubectl apply -f k8s/bankapp-service.yaml -n test-ns
   ```

### Local Development
For local development, you can set the environment variables in a `.env` file or directly in your environment. Example `.env` file:
```bash
SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/bankappdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
SPRING_DATASOURCE_USERNAME=root
SPRING_DATASOURCE_PASSWORD=your_secure_password
```

**Note**: Never commit sensitive data like passwords to version control. Use a secrets manager or Kubernetes secrets for production.

## Additional Notes
- The `bankapp` application connects to the MySQL database using the `root` user by default. For production, consider creating a non-root user with limited privileges by adding `MYSQL_USER` and `MYSQL_PASSWORD` to the MySQL `Deployment` and updating the `bankapp` `Deployment` accordingly.
- Configuration files are located in:
  - `src/main/resources/application.properties` for Spring Boot settings.
  - `k8s/` for Kubernetes manifests (`bankapp-dep.yaml`, `bankapp-service.yaml`, `mysql-deployment.yaml`, `mysql-service.yaml`).