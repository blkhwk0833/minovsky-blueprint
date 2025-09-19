# **The Definitive Guide to Minovsky (v2.4)**

## **Project Goal**

The ultimate objective of **Minovsky** is to create a deeply personalized, intelligent AI platform that manages the entire lifecycle of your model kit hobby. It will move beyond simple data tracking to become an active partner that discovers products, analyzes market trends, identifies reliable purchasing opportunities, automates inventory management, provides creative assistance, tracks your collection's value, and even helps generate passive income to fund the hobby itself.

## **Core Concept: A Phased, Modular Approach**

This guide is structured in phases. Each phase is a self-contained module that adds new capabilities. This allows you to build the project incrementally and control your costs. This document covers the foundational phases required to get the core system online.

---

### **Phase 1: Foundational Setup (GCP & Local Environment)**

**Goal:** To prepare your cloud infrastructure and local computer correctly.

#### **Detailed Step-by-Step Instructions:**

**1.1: GCP Project Setup**

* **What is a GCP Project?** In Google Cloud, a "Project" is a dedicated container for all your cloud resources. It helps with organization, billing, and security.  
* **Action:** Go to the [GCP Console](https://console.cloud.google.com/). You will be guided to create a billing account (a free trial with credits is available). Once complete, create a **New Project**. Name it minovsky-platform and note its unique **Project ID**.

**1.2: Local Toolkit Setup**

* **Google Cloud SDK (gcloud CLI):** This is a command-line tool that lets you manage GCP from your computer's terminal. Follow the [official installation guide](https://cloud.google.com/sdk/docs/install).  
* **Docker Desktop:** A platform for "containerizing" applications. A container ensures your code runs the same everywhere. A [default installation](https://www.docker.com/products/docker-desktop/) is sufficient.

**1.3: Connecting Your Machine to GCP**

* Open your terminal and run these commands to log in and set your project. Replace YOUR\_PROJECT\_ID.  
  Bash  
  gcloud auth login  
  gcloud config set project YOUR\_PROJECT\_ID

**1.4: Enabling Services (APIs)**

* **What is an API?** An Application Programming Interface is like a power switch for a cloud service. This command turns on all the services we need.  
* **Action:** Run the command below. The us-central1 region is recommended for its balance of low cost and performance.  
  Bash  
  gcloud services enable sqladmin.googleapis.com secretmanager.googleapis.com run.googleapis.com cloudscheduler.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com compute.googleapis.com

---

### **Phase 2: The Data Core (Database & Secrets)**

**Goal:** To build the central memory of **Minovsky**, where all collected data will be securely stored and organized.

#### **Detailed Step-by-Step Instructions:**

**2.1: Creating the Database (Cloud SQL)**

* **What is Cloud SQL?** A fully managed database service. Google handles the hardware, backups, and security. We're using PostgreSQL for its power and reliability.  
* **Action:**  
  * In the GCP Console, navigate to **Cloud SQL** \-\> **Create Instance** \-\> **PostgreSQL**.  
  * Select the **"Enterprise"** edition and the **"Development"** preset. This is the most cost-effective and is powerful enough for our entire project.  
  * Set an **Instance ID** (e.g., minovsky-db) and a strong **password**. **Copy this password somewhere safe temporarily.**  
  * Once created, open the instance, go to the **Databases** tab, and create a database named minovsky-data.

**2.2: Securing Your Database Password (Secret Manager)**

* **What is Secret Manager?** A secure vault for sensitive data. This prevents us from ever writing the password directly in our code.  
* **Action:**  
  1. In the GCP Console's main navigation menu (☰), scroll down to the "Security" section and click on **Secret Manager**.  
  2. Click **"Create secret"** at the top of the page.  
  3. **Name:** Enter db-password. Per GCP best practices, we use a hyphen.  
  4. **Secret value:** Paste the database password you saved in the previous step.  
  5. **Defaults are Sufficient:** You do not need to add any labels or configure advanced settings. The default settings are secure and sufficient. Click **"Create secret"**.

**2.3: The Database Schema**

* **What is a Schema?** The blueprint for your database, defining the tables and their structure.  
* **Action:**  
  1. **Navigate to Cloud SQL** in the GCP Console and click on your instance ID (minovsky-db).  
  2. **Activate Cloud Shell:** In the top-right of the console, click the **Activate Cloud Shell** icon (\>\_). A terminal will open at the bottom of your screen.  
  3. **Connect to the Database:** Paste the following command into Cloud Shell and press Enter.  
     Bash  
     gcloud sql connect minovsky-db \--user=postgres \--database=minovsky-data

  4. **Enter Password:** When prompted, paste the password for the postgres user. The prompt will change to postgres=\>.  
  5. **Execute the Schema Script:** Copy and paste the entire SQL code block below into the Cloud Shell terminal.  
     SQL  
     CREATE TABLE manufacturers ( id SERIAL PRIMARY KEY, name VARCHAR(100) UNIQUE NOT NULL );  
     CREATE TABLE products ( id SERIAL PRIMARY KEY, name VARCHAR(255) UNIQUE NOT NULL, manufacturer\_id INTEGER REFERENCES manufacturers(id), status VARCHAR(50) DEFAULT 'Announced', image\_url TEXT, description TEXT );  
     CREATE TABLE listings ( id SERIAL PRIMARY KEY, product\_id INTEGER REFERENCES products(id), retailer\_name VARCHAR(100), price\_jpy INTEGER, stock\_status VARCHAR(50), url VARCHAR(255), scraped\_at TIMESTAMP WITH TIME ZONE, condition VARCHAR(50) DEFAULT 'New', listing\_type VARCHAR(50) DEFAULT 'Retail', seller\_rating REAL );  
     CREATE TABLE retailers ( id SERIAL PRIMARY KEY, name VARCHAR(100) UNIQUE NOT NULL, status VARCHAR(50) DEFAULT 'Vetted', base\_url VARCHAR(255) );  
     CREATE TABLE inventory ( id SERIAL PRIMARY KEY, product\_id INTEGER REFERENCES products(id), status VARCHAR(50), purchase\_price REAL, purchase\_date DATE, condition VARCHAR(50) );

     INSERT INTO manufacturers (name) VALUES ('Bandai'), ('Kotobukiya'), ('Good Smile Company');  
     INSERT INTO retailers (name, status, base\_url) VALUES ('HobbyLink Japan', 'Vetted', 'https://www.hlj.com'), ('AmiAmi', 'Vetted', 'https://www.amiami.com');

  6. **Validate the Schema:** After the script runs, run this command to verify that your tables were created correctly.  
     SQL  
     \\dt

  7. **Disconnect:** Type \\q and press Enter.

---

### **Phase 3: The Application Blueprint (Code & Configuration)**

**Goal:** To prepare your local Python environment, finalize the project's dependencies, and create the core application files.

#### **Detailed Step-by-Step Instructions:**

**3.1: Prepare Your Local Environment & Finalize Dependencies**

* **What is a Virtual Environment (venv)?** It's a clean sandbox for a project, ensuring its libraries don't conflict with other projects.  
* **Action:** Follow these steps in your terminal, inside your main **Minovsky** project folder.  
  1. **Create an Initial requirements.txt File:**  
     Plaintext  
     requests  
     beautifulsoup4  
     sqlalchemy  
     cloud-sql-python-connector\[pg8000\]

  2. **Create the Virtual Environment:**  
     PowerShell  
     python \-m venv venv

  3. **Activate the Environment:**  
     * **On Windows (PowerShell):** .\\venv\\Scripts\\activate  
     * On Mac / Linux: source venv/bin/activate  
       Your terminal prompt will now show (venv) at the beginning.  
  4. **Install the Libraries:**  
     PowerShell  
     pip install \-r requirements.txt

  5. **Generate the Version-Locked File:** This takes a snapshot of the exact library versions and overwrites your requirements.txt file. This is crucial for reliable builds.  
     PowerShell  
     pip freeze \> requirements.txt

  6. **Deactivate the Environment:** When you're done, run this command to exit the sandbox.  
     PowerShell  
     deactivate

3.2: Create the Main Application Files  
With the requirements.txt file finalized, you will now create the other three application files.

##### **requirements.txt (Final Version)**

*This file should now contain the version-locked output from the pip freeze command. It will look similar to this:*

Plaintext

beautifulsoup4==4.12.3  
certifi==2024.2.2  
cloud-sql-python-connector==1.5.0  
pg8000==1.30.5  
requests==2.31.0  
SQLAlchemy==2.0.25  
\# Note: Your versions may be slightly different, which is perfectly normal.

##### **main.py**

Python

import os  
import logging  
import sqlalchemy  
from bs4 import BeautifulSoup  
import requests  
from datetime import datetime, timezone

\# \--- Basic Logging Setup \---  
logging.basicConfig(level=logging.INFO, format\='%(asctime)s \- %(levelname)s \- %(message)s')

def init\_connection\_pool() \-\> sqlalchemy.engine.Engine:  
    """Initializes a secure connection pool to the Cloud SQL database."""  
    instance\_connection\_name \= os.environ.get("INSTANCE\_CONNECTION\_NAME")  
    db\_user \= os.environ.get("DB\_USER", "postgres")  
    db\_pass \= os.environ.get("DB\_PASS")  
    db\_name \= os.environ.get("DB\_NAME", "minovsky-data")

    if not all(\[instance\_connection\_name, db\_user, db\_pass, db\_name\]):  
        raise ValueError("One or more database environment variables are not set.")

    return sqlalchemy.create\_engine(  
        sqlalchemy.engine.url.URL.create(  
            drivername="postgresql+pg8000",  
            username=db\_user,  
            password=db\_pass,  
            database=db\_name,  
            query={"unix\_sock": f"/cloudsql/{instance\_connection\_name}/.s.PGSQL.5432"},  
        ),  
        pool\_size=5, max\_overflow=2, pool\_timeout=30, pool\_recycle=1800,  
    )

def scrape\_product\_page(url: str) \-\> dict:  
    """Scrapes a single product page from a target retailer."""  
    logging.info(f"Starting scrape for URL: {url}")  
    headers \= {'User-Agent': 'GCP-Scraper-Bot/1.0 (Project: Minovsky)'}  
      
    try:  
        response \= requests.get(url, headers=headers, timeout=15)  
        response.raise\_for\_status()  
        soup \= BeautifulSoup(response.text, 'html.parser')  
          
        name\_tag \= soup.select\_one('.product-name')  
        price\_tag \= soup.select\_one('.price')  
        status\_tag \= soup.select\_one('.stock-status')

        if not all(\[name\_tag, price\_tag, status\_tag\]):  
            raise ValueError("Could not find one or more required HTML elements. Site layout may have changed.")

        product\_name \= name\_tag.text.strip()  
        price\_jpy \= int(price\_tag.text.strip().replace('¥', '').replace(',', ''))  
        stock\_status \= status\_tag.text.strip()  
          
        logging.info(f"Successfully scraped: {product\_name}")  
        return {  
            "name": product\_name, "price\_jpy": price\_jpy, "stock\_status": stock\_status,  
            "url": url, "retailer\_name": "HobbyLink Japan"  
        }  
    except requests.exceptions.RequestException as e:  
        logging.error(f"HTTP request to {url} failed: {e}")  
        raise

def insert\_data(db\_pool: sqlalchemy.engine.Engine, scraped\_data: dict):  
    """Inserts the scraped data into the database."""  
    product\_name \= scraped\_data\['name'\]  
      
    with db\_pool.connect() as conn:  
        with conn.begin():  
            insert\_product\_stmt \= sqlalchemy.text(  
                "INSERT INTO products (name, manufacturer\_id, status) VALUES (:name, 1, 'In-Stock') ON CONFLICT (name) DO NOTHING"  
            )  
            conn.execute(insert\_product\_stmt, {"name": product\_name})  
              
            product\_id\_result \= conn.execute(sqlalchemy.text("SELECT id FROM products WHERE name \= :name"), {"name": product\_name})  
            product\_row \= product\_id\_result.fetchone()  
            if product\_row is None:  
                raise RuntimeError(f"Could not find or create product with name '{product\_name}' in the database.")  
            product\_id \= product\_row\[0\]

            insert\_listing\_stmt \= sqlalchemy.text(  
                "INSERT INTO listings (product\_id, retailer\_name, price\_jpy, stock\_status, url, scraped\_at) "  
                "VALUES (:product\_id, :retailer\_name, :price\_jpy, :stock\_status, :url, :scraped\_at)"  
            )  
            conn.execute(insert\_listing\_stmt, {  
                "product\_id": product\_id, "retailer\_name": scraped\_data\['retailer\_name'\], "price\_jpy": scraped\_data\['price\_jpy'\],  
                "stock\_status": scraped\_data\['stock\_status'\], "url": scraped\_data\['url'\], "scraped\_at": datetime.now(timezone.utc)  
            })  
    logging.info(f"Successfully inserted listing for '{product\_name}' into database.")

db \= init\_connection\_pool()

def main():  
    """Main function to orchestrate the scraping and data insertion."""  
    target\_url \= "https://www.hlj.com/1-100-scale-mg-zeta-gundam-ver-ka-bans65455"  
    try:  
        data \= scrape\_product\_page(target\_url)  
        insert\_data(db, data)  
    except Exception as e:  
        logging.error(f"The main process failed: {e}")

if \_\_name\_\_ \== "\_\_main\_\_":  
    main()  
    logging.info("Minovsky scraper finished.")

##### **Dockerfile**

Dockerfile

\# \--- Stage 1: The "Builder" Stage \---  
FROM python:3.11\-slim as builder  
WORKDIR /usr/src/app  
ENV PYTHONDONTWRITEBYTECODE 1  
ENV PYTHONUNBUFFERED 1  
COPY requirements.txt .  
RUN python \-m venv /opt/venv  
ENV PATH="/opt/venv/bin:$PATH"  
RUN pip install \--no-cache-dir \-r requirements.txt

\# \--- Stage 2: The "Final" Stage \---  
FROM python:3.11\-slim  
WORKDIR /app  
COPY \--from=builder /opt/venv /opt/venv  
COPY main.py .  
ENV PATH="/opt/venv/bin:$PATH"  
RUN addgroup \--system app && adduser \--system \--group app appuser  
USER appuser  
CMD \["python", "main.py"\]

##### **cloudbuild.yaml**

YAML

steps:  
\- name: 'gcr.io/cloud-builders/docker'  
  args: \['build', '-t', 'us-central1-docker.pkg.dev/$PROJECT\_ID/minovsky-repo/scraper:latest', '.'\]  
\- name: 'gcr.io/cloud-builders/docker'  
  args: \['push', 'us-central1-docker.pkg.dev/$PROJECT\_ID/minovsky-repo/scraper:latest'\]  
images:  
\- 'us-central1-docker.pkg.dev/$PROJECT\_ID/minovsky-repo/scraper:latest'

---

### **Phase 4: Build, Deploy, & Schedule**

**Goal:** To bring the core scraper application to life in the cloud and automate it.

**4.1: Building and Deploying**

* **Create Artifact Registry Repository:**  
  Bash  
  gcloud artifacts repositories create minovsky-repo \--repository-format=docker \--location=us-central1

* **Run the Build (Cloud Build):**  
  Bash  
  gcloud builds submit \--config cloudbuild.yaml .

* **Deploy the Service (Cloud Run):** **Replace YOUR\_PROJECT\_ID and YOUR\_INSTANCE\_CONNECTION\_NAME**.  
  Bash  
  gcloud run deploy minovsky-price-scraper \\  
    \--image="us-central1-docker.pkg.dev/YOUR\_PROJECT\_ID/minovsky-repo/scraper:latest" \\  
    \--region="us-central1" \\  
    \--allow-unauthenticated \\  
    \--set-env-vars="DB\_USER=postgres,DB\_NAME=minovsky-data,INSTANCE\_CONNECTION\_NAME=YOUR\_INSTANCE\_CONNECTION\_NAME" \\  
    \--set-secrets="DB\_PASS=db-password:latest"

**4.2: Scheduling the Scraper**

* Go to **Cloud Scheduler** \-\> **Create job**.  
* **Name:** run-minovsky-price-scraper-daily.  
* **Frequency:** 0 8 \* \* \* (8:00 AM daily).  
* **Timezone:** America/Chicago.  
* **Target:** **HTTP**, using the URL from your minovsky-price-scraper service.  
* **Test the Job:** After creating the job, find it in the list and click the **"Run now"** button. Then, go to the **Logs** tab of your minovsky-price-scraper service in Cloud Run to verify it ran successfully.

---

### **Phase 5: Project Management & Future Steps**

**Goal:** To establish good project management practices and provide resources for future expansion.

**5.1: Cost Management & Monitoring**

* **Set a Budget Alert:** Go to **Billing** \-\> **Budgets & alerts** in the GCP Console and create a budget to be notified if your spending is projected to exceed a certain amount.  
* **GCP Pricing Calculator:** For precise estimates, use the official [GCP Pricing Calculator](https://cloud.google.com/products/calculator).

5.2: Future Module Implementation  
The foundational system is now complete. The following advanced modules can be built as separate microservices, following the same Code \-\> Build \-\> Deploy \-\> Schedule pattern.

* **The Intelligence Layer:**  
  * **Features:** "New Release Radar," "Community Watch," "Kit Intelligence Dossier."  
  * **New Functionality:** The **"Meteor Scraper Auto-Repair Module"** is part of this layer. When **Meteor** detects a failing scraper, it will automatically analyze the new website HTML, use an AI model to generate a suggested fix, and present it for your one-click approval. It will only auto-apply fixes with a very high confidence score.  
* **The Automation Layer:**  
  * **Features:** Automated inventory via "Email Ingestion," automated "Shipment Tracking."  
* **The Investment Layer:**  
  * **Features:** "Resale Manager," "Proxy Buyer" integration for exclusives, "Optimal Purchase Engine."  
* **The Enrichment Layer:**  
  * **Features:** "Creative Suite" with paint recommendations, AI rendering, digital manuals, and "Smart Hobby Space" integration.

#### **Cost Analysis (v2.4)**

* **Estimated Monthly Cost (Foundation Only):** **$15 \- $25 USD**  
  * **Primary Cost Driver:** The Cloud SQL instance.  
* **Estimated 1-Year Cost (Full System Growth):** **$500 \- $900+**  
  * **Primary Cost Drivers:** Cloud SQL, third-party API subscriptions (Shipping, Proxies), and potential usage costs for services like Cloud Translation and Vertex AI (for the auto-repair module) as you exceed the free tiers.
