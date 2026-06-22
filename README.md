# **Azure Key Vault Integration in Java (REST API Edition)**

## **Part 1: Azure Portal Configuration**

This section details the step-by-step configuration required in the Azure Portal to store a secret and allow a Java application to access it.

### **Step 1: Access the Azure Portal**

1.  Navigate to [https://portal.azure.com](https://portal.azure.com).
2.  Sign in with your Microsoft account.
3.  If you do not have an account, click **"Create one!"** to set up a free Azure trial account.

### **Step 2: Verify Active Subscription**

1.  In the top search bar, type **"Subscriptions"** and select it.
2.  Ensure you have an active subscription (e.g., "Pay-As-You-Go" or "Azure for Students").
3.  _Note: Without an active subscription, you cannot create resources._

### **Step 3: Create a Resource Group**

A Resource Group is a logical container for your Azure resources.

1.  Search for **"Resource Groups"** in the top bar.
2.  Click **+ Create**.
3.  **Subscription:** Select your active subscription.
4.  **Resource Group:** Enter `demo-rg`.
5.  **Region:** Select the region closest to you (e.g., `East US`).
6.  Click **Review + Create**, then click **Create**.

### **Step 4: Create the Azure Key Vault**

1.  Search for **"Key Vaults"** and click **+ Create**.
2.  **Resource Group:** Select `demo-rg`.
3.  **Key Vault Name:** Enter `settribe-demo-vault`.
4.  **Region:** Same as your Resource Group.
5.  **Pricing Tier:** Select `Standard`.
6.  **Access Configuration:** Ensure **"Azure role-based access control (RBAC)"** is selected (this is the modern recommended standard).
7.  Click **Review + Create**, then click **Create**.

### **Step 5: Add a Secret to the Vault**

1.  Go to your new Key Vault (`settribe-demo-vault`).
2.  On the left-hand menu, under **Objects**, click **Secrets**.
3.  Click **+ Generate/Import**.
4.  **Upload options:** Manual.
5.  **Name:** `db-password`.
6.  **Secret value:** Enter your database password (e.g., `MySecurePassword123`).
7.  Click **Create**.

### **Step 6: Create App Registration (Application Identity)**

To allow Java to "log in" to Azure, we must create an Identity.

1.  Search for **"App Registrations"** in the top bar.
2.  Click **+ New Registration**.
3.  **Name:** `java-keyvault-app`.
4.  **Supported account types:** Accounts in this organizational directory only.
5.  Click **Register**.
6.  **CRITICAL:** Copy the following values to a notepad immediately:
    - **Application (client) ID**
    - **Directory (tenant) ID**

### **Step 7: Generate Client Secret**

This acts as the "password" for your Java application.

1.  Inside your App Registration (`java-keyvault-app`), click **Certificates & secrets** on the left menu.
2.  Click the **Client secrets** tab, then click **+ New client secret**.
3.  **Description:** `java-app-token`.
4.  Click **Add**.
5.  **CRITICAL:** Copy the **Value** column immediately.
    - _Warning: This value will be hidden forever once you refresh the page._

### **Step 8: Assign IAM Permissions (The "Bridge")**

Even though the app exists, it doesn't have permission to look inside the Key Vault yet.

1.  Go back to your Key Vault (**settribe-demo-vault**).
2.  Click **Access Control (IAM)** on the left menu.
3.  Click **+ Add** -> **Add role assignment**.
4.  **Role:** Search for and select **"Key Vault Secrets User"**. Click **Next**.
5.  **Assign access to:** User, group, or service principal.
6.  Click **+ Select members**.
7.  Search for `java-keyvault-app` (the name from Step 6). Select it.
8.  Click **Review + assign**.

---

## **Part 2: Java Spring Boot Implementation**

> **Architecture Note (Why REST API?):** Typically, Spring Boot projects use the `spring-cloud-azure-starter-keyvault-secrets` library. However, we are using direct REST API calls because the standard Azure Key Vault library currently has limitations when compiled into a **GraalVM Native Image**. Using REST calls ensures our application compiles seamlessly to a native executable, allowing for incredibly fast cold start times when deployed to **Azure Container Apps**.

To store sensitive credentials in a `.env` file instead of hardcoding them, we will use the **Dotenv-java** library. This is the standard practice to keep your `tenant_id`, `client_id`, and `client_secret` out of your source code and version control (like Git).

Here is the updated documentation and code.

---

### **Step 1: Add the Dotenv Dependency**

Open your `pom.xml` and add the library that allows Java to read `.env` files.

```xml
<dependencies>
    <!-- Standard Spring Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Library to read .env files -->
    <dependency>
        <groupId>io.github.cdimascio</groupId>
        <artifactId>dotenv-java</artifactId>
        <version>3.0.0</version>
    </dependency>
</dependencies>
```

---

### **Step 2: Create the `.env` File**

In the **root folder** of your project (the same folder where `pom.xml` is located), create a new file named `.env`.

Add your Azure details here:

```text
AZURE_TENANT_ID=your-tenant-id-here
AZURE_CLIENT_ID=your-client-id-here
AZURE_CLIENT_SECRET=your-client-secret-here
AZURE_VAULT_NAME=settribe-demo-vault
AZURE_SECRET_NAME=db-password
```

> **Security Note:** Add `.env` to your `.gitignore` file so you don't accidentally upload these credentials to GitHub.

---

### **Step 3: Update the Java Code**

The code now uses `Dotenv` to load the variables at runtime.

```java
package com.example.demo;

import io.github.cdimascio.dotenv.Dotenv;
import org.springframework.http.*;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.web.client.RestTemplate;
import java.util.Map;

public class KeyVaultRestExample {

    public static void main(String[] args) {
        // Load variables from .env file
        Dotenv dotenv = Dotenv.load();

        String tenantId = dotenv.get("AZURE_TENANT_ID");
        String clientId = dotenv.get("AZURE_CLIENT_ID");
        String clientSecret = dotenv.get("AZURE_CLIENT_SECRET");
        String vaultName = dotenv.get("AZURE_VAULT_NAME");
        String secretName = dotenv.get("AZURE_SECRET_NAME");

        try {
            System.out.println("Connecting to Azure...");

            // 1. Get Access Token (Authentication)
            String token = getAccessToken(tenantId, clientId, clientSecret);

            // 2. Get Secret (Authorization & Retrieval)
            String secretValue = getSecretFromVault(vaultName, secretName, token);

            System.out.println("----------------------------------------");
            System.out.println("SUCCESSFULLY FETCHED FROM KEY VAULT");
            System.out.println("Secret Name: " + secretName);
            System.out.println("Secret Value: " + secretValue);
            System.out.println("----------------------------------------");

        } catch (Exception e) {
            System.err.println("Error occurred: " + e.getMessage());
        }
    }

    // This method mimics: curl -X POST https://login.microsoftonline.com/...
    private static String getAccessToken(String tId, String cId, String cSec) {
        RestTemplate rest = new RestTemplate();
        String url = "https://login.microsoftonline.com/" + tId + "/oauth2/v2.0/token";

        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);

        MultiValueMap<String, String> body = new LinkedMultiValueMap<>();
        body.add("grant_type", "client_credentials");
        body.add("client_id", cId);
        body.add("client_secret", cSec);
        body.add("scope", "https://vault.azure.net/.default");

        HttpEntity<MultiValueMap<String, String>> request = new HttpEntity<>(body, headers);
        Map response = rest.postForObject(url, request, Map.class);

        return (String) response.get("access_token");
    }

    // This method mimics: curl -H "Authorization: Bearer ..." GET https://...
    private static String getSecretFromVault(String vName, String sName, String token) {
        RestTemplate rest = new RestTemplate();
        String url = "https://" + vName + ".vault.azure.net/secrets/" + sName + "?api-version=7.4";

        HttpHeaders headers = new HttpHeaders();
        headers.setBearerAuth(token);

        HttpEntity<String> entity = new HttpEntity<>(headers);
        ResponseEntity<Map> response = rest.exchange(url, HttpMethod.GET, entity, Map.class);

        return (String) response.getBody().get("value");
    }
}
```

---

### **Step 4: Run and Verify**

1. Compile and run the `KeyVaultRestExample` Java application.
2. The console should output:
   ```text
   Connecting to Azure...
   ----------------------------------------
   SUCCESSFULLY FETCHED FROM KEY VAULT
   Secret Name: db-password
   Secret Value: MySecurePassword123
   ----------------------------------------
   ```
3. If you encounter an **authentication error** (e.g., `401 Unauthorized`), double-check that your `tenant_id`, `client_id`, and `client_secret` in the `.env` file match the App Registration in Azure.
4. If you encounter an **authorization error** (e.g., `403 Forbidden`), double-check that the App Registration has been granted the **"Key Vault Secrets User"** role in the Key Vault's Access Control (IAM).
