# Self-Service Report Generation — Complete Setup Guide

## From Zero to Working Reports with Real Data

This guide takes you step-by-step from "I have the tool running in Preview mode"
to "users click Generate Report and get a real PDF with live data."

Read the whole guide first before making any changes.
There are 8 steps. Each step is small.

---

## Understanding What You're Connecting

Right now your setup looks like this:

```
React UI → Java Backend → Generates HTML with sample data → Back to UI
```

After this guide, it will look like this:

```
React UI → Java Backend → Generates RDL → Uploads to Report Server
                                                    │
                                                    ▼
                                          Report Server connects
                                          to your actual database
                                                    │
                                                    ▼
                                          Runs the SQL query
                                          against real data
                                                    │
                                                    ▼
                                          Renders PDF/Excel/HTML
                                                    │
                                                    ▼
                          Java Backend ← receives rendered bytes
                                │
                                ▼
                          React UI shows the report / downloads PDF
```

You need three things:
1. Your Power BI Report Server URL (you already have this)
2. A service account that can access Report Server (Guide 3 covered this)
3. A shared data source on Report Server pointing to your database


---

## Step 1: Find Your Report Server URLs

You need two URLs. Ask your IT team or check yourself.

**Web Portal URL:**
Open a browser on your company network and go to your Report Server portal.
It looks something like:

```
https://reportserver.yourcompany.com/reports
```

or

```
http://reportserver:80/reports
```

or

```
https://server-name/ReportServer_PBIRS
```

When you open this URL, you should see the Report Server web portal with
folders and reports. If you can see this page, you have the right URL.

Write this down. This is your `portal-url`.


**Execution URL:**
This is usually the same server but with a different path.
If your portal URL is `https://reportserver.yourcompany.com/reports`,
then your execution URL is typically:

```
https://reportserver.yourcompany.com/reportserver
```

To verify: open this URL in your browser. You should see an XML page
or a simple page saying "Microsoft SQL Server Reporting Services" or
"Power BI Report Server." If you see an error, try these variations:

```
https://your-server/ReportServer
https://your-server/ReportServer_PBIRS
http://your-server/reportserver
```

Write this down. This is your `execution-url`.


---

## Step 2: Get or Verify Your Service Account

You need a Windows/AD service account. If you followed Guide 3,
you already have one. If not, ask your Active Directory admin to
create one. You need:

```
Domain:    YOURCOMPANY           (your AD domain name)
Username:  svc-powerbi-app       (or whatever they created)
Password:  [the password]
```

**Quick test:** Try logging into the Report Server web portal
(the portal-url from Step 1) using these credentials.
Open the portal URL in a private/incognito browser window —
it will prompt for credentials. Enter:

```
Username: YOURCOMPANY\svc-powerbi-app
Password: [the password]
```

If you can see the portal, the account works.
If you get "Access Denied," ask the Report Server admin
to grant the Browser role to this account (see Guide 3, Step 2).


---

## Step 3: Create a Folder on Report Server for Self-Service Reports

The tool uploads temporary RDL files to Report Server, renders them,
then deletes them. These temp files need a folder.

1. Open the Report Server web portal in your browser
2. Click **+ New** → **Folder**
3. Name it: `SelfService`
4. Click **Create**

Now grant your service account permission on this folder:

1. Open the `SelfService` folder
2. Click the **gear icon** (top right) or go to folder settings
3. Go to **Security**
4. Click **Add group or user**
5. Enter: `YOURCOMPANY\svc-powerbi-app`
6. Select roles: **Browser** AND **Publisher**
   (Publisher is needed because the tool uploads RDL files)
7. Click **Apply**


---

## Step 4: Create a Shared Data Source on Report Server

This is how Report Server connects to your actual database.
You create it ONCE, and all self-service reports will use it.

1. In the Report Server web portal, go to the root folder
   (or create a `/DataSources/` folder for organization)

2. Click **+ New** → **Data Source**

3. Fill in:

   **Name:** `ProductionDB`
   (Remember this exact name — you'll use it in Step 6)

   **Connection type:** Select your database type:
   - `Microsoft SQL Server` — if your data is in SQL Server
   - `Oracle` — if Oracle
   - `ODBC` — if using ODBC (similar to what WebFocus used)

   **Connection string:**

   For SQL Server:
   ```
   Data Source=your-db-server.company.com;Initial Catalog=YourDatabaseName
   ```

   For SQL Server with a specific instance:
   ```
   Data Source=your-db-server.company.com\INSTANCE1;Initial Catalog=YourDatabaseName
   ```

   For Oracle:
   ```
   Data Source=//oracle-server:1521/SERVICENAME
   ```

   Replace `your-db-server.company.com` with your actual database server
   and `YourDatabaseName` with your actual database name.
   These are the same server and database that WebFocus was connecting to.

   **Credentials:**
   - Select **"Credentials stored securely in the report server"**
   - Enter the database username and password
     (the account that has SELECT permission on your tables)
   - If your database uses Windows Authentication,
     check **"Use as Windows credentials"**

4. Click **Test Connection** — it should say "Connection created successfully"

5. Click **Create**

Write down the FULL PATH of this data source. It will be something like:
```
/DataSources/ProductionDB
```
or if you put it in the root:
```
/ProductionDB
```

You need this path for Step 6.


---

## Step 5: Update application.yml

Open your backend's configuration file:
```
app/backend/src/main/resources/application.properties
```

Replace its ENTIRE contents with this (or create a new file called
`application.yml` in the same folder — Spring Boot reads both):

**If using application.properties (simpler):**

```properties
server.port=8080
spring.application.name=webfocus-powerbi-migration
spring.servlet.multipart.max-file-size=100MB
spring.servlet.multipart.max-request-size=500MB

# Logging
logging.level.com.migration=INFO
logging.pattern.console=%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n

# Actuator
management.endpoints.web.exposure.include=health,info

# ═══════════════════════════════════════════
# POWER BI REPORT SERVER CONNECTION
# ═══════════════════════════════════════════

# The web portal URL (for REST API calls — listing, uploading)
powerbi.report-server.portal-url=https://reportserver.yourcompany.com/reports

# The execution URL (for rendering reports)
powerbi.report-server.execution-url=https://reportserver.yourcompany.com/reportserver

# Service account credentials (the AD account from Step 2)
powerbi.report-server.domain=YOURCOMPANY
powerbi.report-server.username=svc-powerbi-app
powerbi.report-server.password=${POWERBI_SERVICE_PASSWORD}

# Your app server's hostname (for NTLM handshake)
powerbi.report-server.workstation=APP-SERVER-01

# Path to the shared data source you created in Step 4
powerbi.report-server.shared-datasource-path=/DataSources/ProductionDB
```

**IMPORTANT — Replace these values:**

| Placeholder | Replace With | Example |
|---|---|---|
| `https://reportserver.yourcompany.com/reports` | Your portal URL from Step 1 | `https://biserver.acme.com/reports` |
| `https://reportserver.yourcompany.com/reportserver` | Your execution URL from Step 1 | `https://biserver.acme.com/reportserver` |
| `YOURCOMPANY` | Your AD domain | `ACME` |
| `svc-powerbi-app` | Your service account username | `svc-pbi-reports` |
| `APP-SERVER-01` | The hostname of the machine running your Java app | `dev-machine-01` |
| `/DataSources/ProductionDB` | The path of the shared data source from Step 4 | `/ProductionDB` |

**For the password — set it as an environment variable, not in the file:**

On Linux/Mac:
```bash
export POWERBI_SERVICE_PASSWORD=YourActualPasswordHere
```

On Windows Command Prompt:
```
set POWERBI_SERVICE_PASSWORD=YourActualPasswordHere
```

On Windows PowerShell:
```
$env:POWERBI_SERVICE_PASSWORD="YourActualPasswordHere"
```

Then start the backend in the SAME terminal window where you set the variable.


---

## Step 6: Update DynamicRdlGenerator to Use Shared Data Source

This is the one code change needed. Open:
```
app/backend/src/main/java/com/migration/selfservice/DynamicRdlGenerator.java
```

Find this method (around line 30):

```java
private void writeDataSources(XMLStreamWriter w) throws XMLStreamException {
    w.writeStartElement("DataSources");
    w.writeStartElement("DataSource");
    w.writeAttribute("Name", "SelfServiceDS");
    w.writeStartElement("ConnectionProperties");
    el(w, "DataProvider", "SQL");
    el(w, "ConnectString", "Data Source=YOUR_SERVER;Initial Catalog=YOUR_DATABASE");
    el(w, "IntegratedSecurity", "true");
    w.writeEndElement();
    w.writeEndElement();
    w.writeEndElement();
}
```

Replace it with:

```java
private void writeDataSources(XMLStreamWriter w) throws XMLStreamException {
    w.writeStartElement("DataSources");
    w.writeStartElement("DataSource");
    w.writeAttribute("Name", "SelfServiceDS");

    // Reference the shared data source on Report Server
    el(w, "DataSourceReference", "/DataSources/ProductionDB");

    w.writeEndElement();
    w.writeEndElement();
}
```

Change `/DataSources/ProductionDB` to match the EXACT path of
the shared data source you created in Step 4.

This one change means:
- The RDL no longer carries any database connection details
- When Report Server renders the RDL, it looks up the shared data source
- The shared data source has the real database connection and credentials
- All self-service reports automatically use the right database


---

## Step 7: Add Apache HttpClient 5 Dependency (for NTLM Auth)

The current code uses Java's built-in HttpClient which may not handle
NTLM authentication properly with some Report Server configurations.

Open `app/backend/pom.xml` and add this dependency inside `<dependencies>`:

```xml
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
    <version>5.3.1</version>
</dependency>
```

If NTLM doesn't work with the built-in client (you'll know because
you get 401 errors), you'll need to update `ReportRenderer.java` to
use Apache HttpClient 5 with NTCredentials — Guide 3 has the code for this.

For many Report Server setups, the built-in Java client works fine,
so try without the change first.


---

## Step 8: Test It End to End

**Start the backend** (in the terminal where you set the password env var):

```bash
cd app/backend
mvn clean package -DskipTests
mvn spring-boot:run
```

Watch the console. You should see:
```
Started MigrationApplication in X seconds
```

No errors about the powerbi config means the properties loaded correctly.


**Start the frontend:**

```bash
cd app/frontend
npm run dev
```


**Test in the browser:**

1. Open `http://localhost:3000`

2. Switch to **★ Self-Service** mode

3. You should now see the capabilities banner say:
   ```
   Render: ReportServer    Formats: HTML5, PDF, EXCELOPENXML, CSV
   ```
   If it still says "Preview", the Report Server URL is not configured
   correctly — check application.properties.

4. Select **STORE_SALES** as the data source

5. Add measures: click **Revenue** and **Cost of Goods**

6. Add a dimension: click **Region**

7. Select format dropdown: **PDF**

8. Click **▶ Generate Report**

9. What happens:
   - Backend generates RDL with your measures/dimensions
   - Uploads to Report Server's /SelfService/ folder
   - Report Server connects to your database via the shared data source
   - Runs: `SELECT REGION_NAME, REVENUE, COST_OF_GOODS FROM dbo.STORE_SALES`
   - Renders a PDF with real data
   - Backend downloads the PDF and sends to your browser
   - Browser downloads the PDF file

10. Open the PDF — you should see a real report with your actual data,
    grouped by Region, with Revenue and Cost of Goods columns.


---

## Troubleshooting

**Problem: Capabilities banner still says "Preview"**

The `powerbi.report-server.execution-url` property is empty or not being read.
Check that your application.properties file:
- Is in `src/main/resources/` (not somewhere else)
- Has no typos in the property names
- Uses `powerbi.report-server.execution-url` (with hyphens, not underscores)

Restart the backend after any config change.


**Problem: 401 Unauthorized when generating report**

Your service account credentials are wrong or NTLM isn't working.
- Verify you can log into the Report Server portal with these credentials
  (private browser window, enter DOMAIN\username and password)
- Check the domain name is correct (just the short domain like ACME,
  not the full domain like acme.com — unless your setup uses the full name)
- Make sure the password environment variable is set in the same terminal
  where you run `mvn spring-boot:run`
- Try the Apache HttpClient 5 approach from Step 7


**Problem: 403 Forbidden on upload**

The service account doesn't have Publisher role on the /SelfService/ folder.
Go back to Step 3 and verify the permissions.


**Problem: Report renders but shows "Data source not found"**

The shared data source path in DynamicRdlGenerator.java doesn't match
what's on Report Server.
- Open Report Server portal → navigate to where you created the data source
- Click on it → check its Path property
- Make sure the path in your code matches EXACTLY (case-sensitive)


**Problem: Report renders but shows no data / empty table**

The SQL query works but returns no rows with the given filters.
- Click **◎ Preview SQL** in the UI to see the generated SQL
- Copy that SQL and run it directly in your database tool
  (SQL Server Management Studio, SQL Developer, etc.)
- If it returns no rows, adjust your filters
- If the table name is wrong (your actual table might be named differently
  than STORE_SALES), you'll need to update the field catalog


**Problem: "Table dbo.STORE_SALES does not exist"**

The sample field catalog uses example table names. Your actual database
has different table names. Two fixes:

Option A — Import your .mas files to populate the catalog with real table names:
In the Self-Service UI → click **+ Import .mas file** → upload your .mas file.
This creates a catalog entry with the correct table and field names.

Option B — Update the default catalog in SelfServiceService.java:
Change the `initDefaultCatalog()` method to use your actual table and column names.


**Problem: SSL/Certificate error**

Your Report Server uses HTTPS with a certificate your Java app doesn't trust.
Import the certificate:

```bash
keytool -import -alias reportserver \
  -file /path/to/reportserver-cert.cer \
  -keystore $JAVA_HOME/lib/security/cacerts \
  -storepass changeit
```

Get the certificate file from your IT team, or export it from your browser
(click the lock icon → certificate → export).


---

## Quick Verification Checklist

Before testing, confirm each of these:

```
[ ] Report Server portal URL opens in browser
[ ] Report Server execution URL opens in browser (shows XML or SSRS page)
[ ] Service account can log into the portal (test in private browser)
[ ] /SelfService/ folder exists on Report Server
[ ] Service account has Browser + Publisher role on /SelfService/
[ ] Shared data source exists on Report Server (e.g., /DataSources/ProductionDB)
[ ] Shared data source Test Connection succeeds
[ ] application.properties has all powerbi.report-server.* values filled in
[ ] Password environment variable is set in the terminal
[ ] DynamicRdlGenerator.java updated with correct shared data source path
[ ] Backend starts without errors
[ ] Capabilities banner shows "ReportServer" (not "Preview")
```

Once all boxes are checked, the ▶ Generate Report button will produce
real PDF/Excel/HTML reports with live data from your database.


---

## After It Works: What to Do Next

**Import your real .mas files** — In the Self-Service UI, click
"+ Import .mas file" for each master file. This populates the field
catalog with your actual table names, field names, and data types.
Users will then see your real tables and fields when building reports.

**Save commonly used reports** — Build the reports your team uses most often,
save them, and users can re-run them with one click from the Saved Reports panel.

**Add more data sources** — If you have multiple databases, create a shared
data source on Report Server for each one, then update the tool to let users
pick which data source to use (this would be an enhancement).
