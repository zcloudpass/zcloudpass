<div align="center">
    <img src="https://github.com/zcloudpass/zcloudpass/blob/e54b4da8d868b07fa1028a7f9e874d5657a6fa80/assets/rounded/logo.png" width="128"/>
    <h1>ZCloudPass</h1>
</div>

# System Design and Diagrams – zCloudPass

This section explains the system design diagrams prepared for Sprint 1 of the **zCloudPass** project. These diagrams describe the structure, behavior, and interaction flow of the system. They provide a clear understanding of how the password manager is organized and how security is enforced at each level.

## 1. High-Level Architecture Diagram

The **High-Level Architecture Diagram** represents the overall structure of the zCloudPass system. It shows how the main components interact with each other and defines the boundaries of the application.

The system is divided into three primary layers:

### Client Layer (Frontend)
The client layer is responsible for user interaction. It allows users to:
- Register and log in
- Add, edit, delete, and view stored passwords
- Synchronize vault data

The frontend also performs encryption **before** transmitting sensitive information to the server. This ensures that plaintext passwords are never exposed during communication.

### Application Layer (Backend Server)
The backend server handles the core business logic of the system. Its responsibilities include:
- User authentication and validation
- Session management using secure tokens
- Secure storage and retrieval of encrypted vault data
- Digital signature verification

The backend operates under a **zero-knowledge principle**, meaning it never has access to plaintext vault data.

### Data Layer (Database)
The database stores:
- Hashed user credentials
- Encrypted vault data
- Metadata such as timestamps and version information

Since vault data is encrypted before storage, even in the case of database compromise, the actual passwords remain protected.

---

## 2. Use Case Diagram

The **Use Case Diagram** illustrates how users interact with the system and defines the functional scope of the application.

The primary actor in the system is the **User**.

The system supports the following major use cases:

### Authentication and Account Management
- Register Account
- Login
- Logout
- Change Master Password

These use cases ensure that only authorized users can access their personal vault.

### Vault Management
- Create Vault
- Add Password Entry
- Edit Password Entry
- Delete Password Entry
- View Stored Credentials
- Sync Vault

This diagram clearly establishes the features implemented in the system and defines the boundary between authentication functions and vault operations.

---

## 3. Authentication Sequence Diagram

The **Authentication Sequence Diagram** explains the step-by-step interaction between the client, server, and database during the login process.

The flow is as follows:
1. The user enters login credentials in the client interface.
2. The client securely transmits the credentials to the backend server.
3. The backend validates the credentials against stored hashed values in the database.
4. If the credentials are valid, a session token is generated.
5. The token is sent back to the client.
6. The user session is established.

This sequence ensures secure authentication while protecting user credentials.

---

## 4. Vault Operation Sequence Diagram

The **Vault Operation Sequence Diagram** demonstrates how password entries are securely stored and retrieved.

For example, when adding a new password entry:
1. The user enters credential details in the client.
2. The client encrypts the data using **AES encryption**.
3. The encrypted data is transmitted to the backend server.
4. The backend stores the encrypted vault data in the database.
5. A confirmation response is returned to the client.

At no stage does the server access plaintext password data, ensuring end-to-end security.

---

## 5. Security Design Representation

The diagrams collectively highlight the **security-focused architecture** of zCloudPass. The system incorporates:
- **AES encryption** for vault protection
- **Secure hashing** for user authentication
- **Base64 encoding** for safe data transmission
- **Digital signatures** for integrity verification
- **Token-based session management**
- **Zero-knowledge architecture principles**

These security mechanisms are integrated into the system design to protect user data from unauthorized access and common attack scenarios.
