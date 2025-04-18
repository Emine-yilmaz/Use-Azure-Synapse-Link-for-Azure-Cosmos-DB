# Use-Azure-Synapse-Link-for-Azure-Cosmos-DB
# Use Azure Synapse Link for Azure Cosmos DB

This document outlines the steps to explore Azure Synapse Link for Azure Cosmos DB, enabling near-real-time analytics over operational data from Azure Synapse Analytics.

## Prerequisites

* An Azure subscription with administrative-level access.

## Steps

### 1. Provision Azure Resources

Provision an Azure Synapse Analytics workspace and an Azure Cosmos DB account using the provided PowerShell script and ARM template.

1.  Sign in to the [Azure portal](https://portal.azure.com).
2.  Open **Cloud Shell** by clicking the **[\>_]** button. Select **PowerShell** if prompted.
3.  Clone the repository:
    ```powershell
    rm -r dp-203 -f
    git clone [https://github.com/MicrosoftLearning/dp-203-azure-data-engineer](https://github.com/MicrosoftLearning/dp-203-azure-data-engineer) dp-203
    ```
4.  Navigate to the lab directory and run the setup script:
    ```powershell
    cd dp-203/Allfiles/labs/14
    ./setup.ps1
    ```
5.  If prompted, choose your subscription and enter a password for the SQL pool.
6.  Wait for the script to complete (approximately 10 minutes).

### 2. Configure Synapse Link in Azure Cosmos DB

Enable Synapse Link and create an analytical store container in your Azure Cosmos DB account.

1.  In the Azure portal, navigate to the `dp203-xxxxxxx` resource group and open your `cosmosxxxxxxxx` Cosmos DB account (the one with the largest number suffix if multiple exist in a deleting state).
2.  In your Cosmos DB account, go to **Data Explorer**. Close any welcome dialog.
3.  Click **Enable Azure Synapse Link** at the top.
4.  In the **Integrations** section, select **Azure Synapse Link** and verify the status is **Enabled**.
5.  Return to **Data Explorer** and click **New Container**. Use the following settings:
    * **Database id**: `AdventureWorks` (Create new)
    * **Share throughput across containers**: Unselected
    * **Container id**: `Sales`
    * **Partition key**: `/customerid`
    * **Container throughput (autoscale)**: Autoscale
    * **Container Max RU/s**: `4000`
    * **Analytical store**: `On`
6.  Expand the `AdventureWorks` database and the `Sales` container, then select **Items**.
7.  Click **New Item** and add the following JSON items, saving each one:
    ```json
    {
        "id": "SO43701",
        "orderdate": "2019-07-01",
        "customerid": 123,
        "customerdetails": {
            "customername": "Christy Zhu",
            "customeremail": "christy12@adventure-works.com"
        },
        "product": "Mountain-100 Silver, 44",
        "quantity": 1,
        "price": 3399.99
    }
    ```
    ```json
    {
        "id": "SO43704",
        "orderdate": "2019-07-01",
        "customerid": 124,
        "customerdetails": {
            "customername": "Julio Ruiz",
            "customeremail": "julio1@adventure-works.com"
        },
        "product": "Mountain-100 Black, 48",
        "quantity": 1,
        "price": 3374.99
    }
    ```
    ```json
    {
        "id": "SO43707",
        "orderdate": "2019-07-02",
        "customerid": 125,
        "customerdetails": {
            "customername": "Emma Brown",
            "customeremail": "emma3@adventure-works.com"
        },
        "product": "Road-150 Red, 48",
        "quantity": 1,
        "price": 3578.27
    }
    ```

### 3. Configure Synapse Link in Azure Synapse Analytics

Connect your Synapse workspace to your Azure Cosmos DB account.

1.  In the Azure portal, return to the `dp203-xxxxxxx` resource group and open your `synapsexxxxxxx` Synapse workspace.
2.  Open **Synapse Studio**.
3.  Go to the **Data** page and select the **Linked** tab.
4.  Click **+ Connect to external data** and select **Azure Cosmos DB for NoSQL**. Click **Continue**.
5.  Create a new Cosmos DB connection with the following settings:
    * **Name**: `AdventureWorks`
    * **Description**: `AdventureWorks Cosmos DB database`
    * **Connect via integration runtime**: `AutoResolveIntegrationRuntime`
    * **Authentication type**: `Account key`
    * **Connection string**: `selected`
    * **Account selection method**: `From subscription`
    * **Azure subscription**: Select your Azure subscription
    * **Azure Cosmos DB account name**: Select your `cosmosxxxxxxxx` account
    * **Database name**: `AdventureWorks`
6.  Click **Create**.
7.  Click the **↻ Refresh** button on the **Data** page until an **Azure Cosmos DB** category appears in the **Linked** pane.
8.  Expand **Azure Cosmos DB** to see the `AdventureWorks` connection and the `Sales` container.

### 4. Query Azure Cosmos DB from Azure Synapse Analytics

Query the Cosmos DB data using both a Spark pool and a serverless SQL pool.

#### Query Azure Cosmos DB from a Spark Pool

1.  In the **Data** pane, select the `Sales` container. In its **...** menu, select **New Notebook** > **Load to DataFrame**.
2.  In the new notebook, attach it to your **sparkxxxxxxx** Spark pool.
3.  Click **▷ Run all** to execute the generated code cell. Review the code, which reads data from the Cosmos DB analytical store into a Spark DataFrame.
4.  Once the code runs, review the output, which should show the three records from your Cosmos DB container.
5.  Add a new code cell and enter:
    ```python
    customer_df = df.select("customerid", "customerdetails")
    display(customer_df)
    ```
    Run the cell and observe the `customerdetails` column containing JSON.
6.  Add another new code cell and enter:
    ```python
    customerdetails_df = df.select("customerid", "customerdetails.*")
    display(customerdetails_df)
    ```
    Run the cell and see the `customername` and `customeremail` as separate columns.
7.  Add another new code cell and enter:
    ```sql
    %%sql
    -- Create a logical database in the Spark metastore
    CREATE DATABASE salesdb;

    USE salesdb;
    -- Create a table from the Cosmos DB container
    CREATE TABLE salesorders using cosmos.olap options (
        spark.synapse.linkedService 'AdventureWorks',
        spark.cosmos.container 'Sales'
    );
    -- Query the table
    SELECT * FROM salesorders;
    ```
    Run the cell to create a Spark metastore database and table.
8.  Add a final new code cell and enter:
    ```sql
    %%sql
    SELECT id, orderdate, customerdetails.customername, product
    FROM salesorders
    ORDER BY id;
    ```
    Run the cell and observe how you can query nested JSON properties using Spark SQL.

#### Query Azure Cosmos DB from a Serverless SQL Pool

1.  In the **Data** pane, select the `Sales` container. In its **...** menu, select **New SQL script** > **Select TOP 100 rows**.
2.  In the new SQL script, replace the `IF (NOT EXISTS(...)` block with the following, replacing `cosmosxxxxxxxx` with your Cosmos DB account name and `<Enter your Azure Cosmos DB key here>` with your Cosmos DB **Primary Key** (found in the **Keys** page of your Cosmos DB account in the Azure portal):
    ```sql
    CREATE CREDENTIAL [cosmosxxxxxxxx]
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
    SECRET = '<Your Azure Cosmos DB Primary Key>'
    GO
    ```
3.  Run the script. Review the results.
4.  Replace all the code in the script with the following, substituting `cosmosxxxxxxxx` with your Cosmos DB account name:
    ```sql
    SELECT *
    FROM OPENROWSET(
        PROVIDER = 'CosmosDB',
        CONNECTION = 'Account=cosmosxxxxxxxx;Database=AdventureWorks',
        OBJECT = 'Sales',
        SERVER_CREDENTIAL = 'cosmosxxxxxxxx'
    )
    WITH (
        OrderID VARCHAR(10) '$.id',
        OrderDate VARCHAR(10) '$.orderdate',
        CustomerID INTEGER '$.customerid',
        CustomerName VARCHAR(40) '$.customerdetails.customername',
        CustomerEmail VARCHAR(30) '$.customerdetails.customeremail',
        Product VARCHAR(30) '$.product',
        Quantity INTEGER '$.quantity',
        Price FLOAT '$.price'
    ) AS sales
    ORDER BY OrderID;
    ```
5.  Run the script and review the structured results.

### 5. Verify Data Modifications in Cosmos DB are Reflected in Synapse

1.  In the Azure portal, go to the **Data Explorer** for your `cosmosxxxxxxxx` Cosmos DB account.
2.  In the `AdventureWorks` > `Sales` > `Items`, create a new item with the following JSON:
    ```json
    {
        "id": "SO43708",
        "orderdate": "2019-07-02",
        "customerid": 126,
        "customerdetails": {
            "customername": "Samir Nadoy",
            "customeremail": "samir1@adventure-works.com"
        },
        "product": "Road-150 Black, 48",
        "quantity": 1,
        "price": 3578.27
    }
    ```
    Save the item.
3.  Return to Synapse Studio and in the **SQL Script 1** tab, re-run the query. Wait a minute or so and re-run again until you see the new record for Samir Nadoy.
4.  Switch to the **Notebook 1** tab and re-run the last code cell to verify the new record is also reflected in the Spark query results.

**Important:** After completing this lab, remember to delete the resource group `dp203-xxxxxxx` in the Azure portal to avoid incurring unnecessary costs.
