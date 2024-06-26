# Running a Java EE 7 Application on IBM WebSphere Application Server Traditional Network Deployment V9 with Azure VMs

This project showcases how to configure and deploy a Java EE 7 application on IBM WebSphere Application Server Traditional (tWAS) Network Deployment V9 using Azure VMs. The project is based on the well-known Jakarta EE sample application Cargo Tracker V1.0. For more details about the Cargo Tracker project, please visit: https://eclipse-ee4j.github.io/cargotracker/.

# Table of Contents

1. [Get Started](#get-started)
    - [Environment setup](#environment-setup)
1. [Standup WebSphere Application Server (traditional) Cluster on Azure VMs](#standup-websphere-application-server-traditional-cluster-on-azure-vms)
1. [Sign in Azure](#sign-in-azure)
1. [Create Azure Database for PostgreSQL Flexible Server](#create-azure-database-for-postgresql-flexible-server)
1. [Install PostgreSQL driver](#install-postgresql-driver)
1. [Configure Console Preferences for Node Synchronization](#configure-console-preferences-for-node-synchronization)
1. [Create data source connection in tWAS](#create-data-source-connection-in-twas)
1. [Create JMS queues](#create-jms-queues)
1. [Deploy Cargo Tracker](#deploy-cargo-tracker)
1. [Exercise Cargo Tracker Functionality](#exercise-cargo-tracker-functionality)
1. [Migration summary](#migration-summary)


# Get Started

Environment setup：

* An Azure subscription.
* Obtained the project source code.
* Java SE 8 installed and JAVA_HOME configured.
* Maven properly set up.
* Azure CLI version 2.58.0 or later installed..

Navigate to the project root and run `mvn clean install` to build the application. This will generate:

* A WAR package at `cargotracker/target/cargo-tracker.war`.
* An EAR package at `cargotracker-was-application/target/cargo-tracker.ear`, which will be deployed on tWAS.

## Standup WebSphere Application Server (traditional) Cluster on Azure VMs

To deploy a tWAS cluster using [Azure Marketplace offer](https://aka.ms/websphere-on-azure-portal), follow the instructions in [Deploy WebSphere Application Server (traditional) Cluster on Azure Virtual Machines](https://learn.microsoft.com/azure/developer/java/ee/traditional-websphere-application-server-virtual-machines?tabs=basic). Ensure you create a cluster with 4 members and note down your WebSphere administrator credentials from the deployment outputs.

After deployment, retrieve the administrative console and IHS console URLs from the **Outputs** section.

## Sign in Azure

If you haven't already, sign into your Azure subscription by using the `az login` command and follow the on-screen directions.

```
az login --use-device-code
```

If you have multiple Azure tenants associated with your Azure credentials, you must specify which tenant you want to sign in to. You can do this with the `--tenant` option. For example, `az login --tenant contoso.onmicrosoft.com`.

## Create Azure Database for PostgreSQL Flexible Server

Cargo Tracker requires a database for persistence. This project creates a PostgreSQL Flexible Server for it. 

Run the following commands to create PostgreSQL database.

```
export RESOURCE_GROUP_NAME=<the-resource-group-your-deploy-twas>
export DB_SERVER_NAME=postgresql$(date +%s)
export DB_NAME=cargotracker
export DB_ADMIN_USER_NAME=wasadmin
export DB_ADMIN_PSW=Secret123456
export LOCATION=eastus
```

Create server and database.

```
az postgres flexible-server create \
  --resource-group ${RESOURCE_GROUP_NAME} \
  --name ${DB_SERVER_NAME} \
  --location ${LOCATION} \
  --admin-user ${DB_ADMIN_USER_NAME} \
  --admin-password ${DB_ADMIN_PSW} \
  --version 16 \
  --public-access 0.0.0.0 \
  --tier Burstable \
  --sku-name Standard_B1ms \
  --yes

az postgres flexible-server db create \
  --resource-group ${RESOURCE_GROUP_NAME} \
  --server-name ${DB_SERVER_NAME} \
  --database-name ${DB_NAME}
```

Set server parameters.

```
az postgres server configuration set \
  --resource-group ${RESOURCE_GROUP_NAME} \
  --server-name ${DB_SERVER_NAME} \
  --name max_connections \
  --value 200

az postgres server configuration set \
  --resource-group ${RESOURCE_GROUP_NAME} \
  --server-name ${DB_SERVER_NAME} \
  --name max_prepared_transactions \
  --value 50
```

Configure firewall rule.

```
echo "Allow Access To Azure Services"
az postgres flexible-server firewall-rule create \
  -g ${RESOURCE_GROUP_NAME} \
  -n ${DB_SERVER_NAME} \
  -r "AllowAllWindowsAzureIps" \
  --start-ip-address "0.0.0.0" \
  --end-ip-address "0.0.0.0"

echo "Connection string:"
echo "jdbc:postgresql://${DB_SERVER_NAME}.postgres.database.azure.com:5432/${DB_NAME}?user=${DB_ADMIN_USER_NAME}&password=${DB_ADMIN_PSW}&sslmode=require"
```

## Install PostgreSQL driver

The tWAS installation does not include the PostgreSQL driver. Follow the steps to install in manually.

Download PostgreSQL driver and save it in each managed VM.

```
export POSTGRESQL_DRIVER_URL="https://jdbc.postgresql.org/download/postgresql-42.5.1.jar"
export DRIVER_TARGET_PATH="/datadrive/IBM/WebSphere/ND/V9/postgresql"

vmList=$(az vm list --resource-group ${RESOURCE_GROUP_NAME} --query [].name -otsv | grep "managed")

for vm in ${vmList}
do 
    az vm run-command invoke \
      --resource-group $RESOURCE_GROUP_NAME \
      --name ${vm} \
      --command-id RunShellScript \
      --scripts "sudo mkdir ${DRIVER_TARGET_PATH}; sudo curl ${POSTGRESQL_DRIVER_URL} -o ${DRIVER_TARGET_PATH}/postgresql.jar"
done

echo "PostgreSQL driver path:"
echo ${DRIVER_TARGET_PATH}/postgresql.jar
```

You should find success message for each VM like:

```
{
  "value": [
    {
      "code": "ProvisioningState/succeeded",
      "displayStatus": "Provisioning succeeded",
      "level": "Info",
      "message": "Enable succeeded: \n[stdout]\n\n[stderr]\n  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current\n                                 Dload  Upload   Total   Spent    Left  Speed\n\r  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0\r100 1022k  100 1022k    0     0  1932k      0 --:--:-- --:--:-- --:--:-- 1936k\n",
      "time": null
    }
  ]
}
```

## Configure Console Preferences for Node Synchronization

First, configure the Console to synchronize changes with Nodes. The changes will be applied to all nodes once you save them.

* Open the administrative console in your Web browser and login with WebSphere administrator credentials.
* In the left navigation panel, select **System administration** -> **Console Preference**.
* Select **Synchronize changes with Nodes**.
* Select **Apply**. You will find message saying "Your preferences have been changed."

## Create data source connection in tWAS

In this section, you'll configure the data source using IBM console.

Current tWAS cluster does not ship with PostgreSQL database provider. Follow the steps create a provider:

* In the left navigation panel, select **Resources** -> **JDBC** -> **JDBC providers**.
* In the **Data source** panel, change scope with **Cluster=MyCluster**. Then select **New...** button to create a new data source.
  * In **Step 1**:
    * For **Database type**, select **User-defined**.
    * For **Implementation class name**, fill in value `org.postgresql.xa.PGXADataSource`.
    * For **Name**, fill in `PostgreSQLJDBCProvider`.
    * Select **Next**.
  * In **Step 2**:
    * For **Class path**, fill in `/datadrive/IBM/WebSphere/ND/V9/postgresql/postgresql.jar`, the same value with the printed "PostgreSQL driver path" previously.
    * Select **Next**.
  * In **Step 3**:
    * Select **Finish**.
* Select **Save** to save the configuration.
* To load the drive, you have to restart the cluster.
  * In the left navigation panel, select **Servers** -> **Clusters** -> **WebSphere application server clusters**.
  * Check the box next to **MyCluster**.
  * Select **Stop** to stop the cluster. 
    * Select **OK** in the **Stop cluste** page.
    * Refresh the status by clicking refresh button next to **Status** column.
    * Wait for the cluster changes to **Stop** state.
  * Check the box next to **MyCluster**.
  * Select **Start** to start the cluster. Refresh the status by clicking refresh button next to **Status** column. Do not move on before the status is in **Started** state.

Follow the steps to create data source:

* In the left navigation panel, select **Resources** -> **JDBC** -> **Data sources**.
* In the **Data source** panel, change scope with **Cluster=MyCluster**. Then select **New...** button to create a new data source.
  * In **Step 1**:
    * For **Data source name**, fill in `CargoTrackerDB`. 
    * For **JNDI name**, fill in `jdbc/CargoTrackerDB`.
    * Select **Next**.
  * In **Step 2**:
    * Select **Select an existing JDBC provider**.
    * From the drop down, select `PostgreSQLJDBCProvider`.
    * Select **Next**.
  * In **Step 3**:
    * For **Data store helper class name**, fill in `com.ibm.websphere.rsadapter.GenericDataStoreHelper`.
    * Select **Next**.
  * In **Step 4**:
    * Select **Next**.
  * In **Summary**, select **Finish**.
  * Select **Save** to save the configuration.
  * Select the data source **CargoTrackerDB** in the table, continue to configure URL.
    * Select **Custom properties**. From the table, from column **Name**, find the row whose name is **URL**. If no, select **New...** to create one.
    * Select **URL**. Fill in value with database connection string that is printed previously. 
    * Select **Apply**. 
    * Select **OK**.
    * Select **Save** to save the configuration.

Validate the data source connection:

  * Select hyperlink **CargoTrackerDB** to return back to the **Data source** panel.
  * Select **Test connection**. If the data source configuration is correct, you will find message like "The test connection operation for data source CargoTrackerDB on server nodeagent at node managed6ba334VM2Node01 was successful". If there is any error, resolve it before you move on.

## Create JMS queues

In this section, you'll create a JMS Bus, a JMS Queue Connection Factory, five JMS Queues and 5 Activation specifications. 
Their names and relationship are listed in the table.

| Bean name | Activation spec |Activation spec JNDI | Queue name | Queue JNDI |
|-----------|-----------------|---------------------|------------|------------|
| RejectedRegistrationAttemptsConsumer | RejectedRegistrationAttemptsQueueAS | jms/RejectedRegistrationAttemptsQueueAS | RejectedRegistrationAttemptsQueue| jms/RejectedRegistrationAttemptsQueue |
| HandlingEventRegistrationAttemptConsumer | HandlingEventRegistrationAttemptQueueAS | jms/HandlingEventRegistrationAttemptQueueAS | HandlingEventRegistrationAttemptQueue | jms/HandlingEventRegistrationAttemptQueue |
| CargoHandledConsumer | CargoHandledQueueAS | jms/CargoHandledQueueAS | CargoHandledQueue | jms/CargoHandledQueue |
| DeliveredCargoConsumer | DeliveredCargoQueueAS | jms/DeliveredCargoQueueAS | DeliveredCargoQueue | jms/DeliveredCargoQueue |
| MisdirectedCargoConsumer | MisdirectedCargoQueueAS | jms/MisdirectedCargoQueueAS | MisdirectedCargoQueue | jms/MisdirectedCargoQueue |

You can find the binding definition from `cargotracker/src/main/resources/META-INF/ibm-ejb-jar-bnd.xml`.

Create JMS Bus.

* Open the administrative console in your Web browser and login with WebSphere administrator credentials.
* In the left navigation panel, select **Service integration** -> **Buses**.
* In the **Buses** panel, select **New...**.
* For **Enter the name for your new bus**, fill in `CargoTrackerBus`.
* Uncheck the checkbox next to **Bus security**.
* Select **Next**.
* Select **Finish**. You'll return back to the **Buses** table.
* In the table, select **CargoTrackerBus**.
* In the **Configuration** panel, under **Topology**, select **Bus members**.
* Select **Add** button to open **Add a new bus member** panel.
* For **Select servers, cluster or WebSphere MQ server**, select **Cluster**. 
* Next to **Cluster**, from the dropdown, select **MyCluster**.
* Select **Next**.
* In **Step 1.1.1**, select **File store**.
* Select **Next**.
* In **Step 1.1.2**:
    * For **Log directory path**, fill in `/datadrive/IBM/WebSphere/ND/V9/postgresql` or any path you expect.
    * For **Permanent store directory path**, fill in `/datadrive/IBM/WebSphere/ND/V9/postgresql` or any path you expect.
    * Select **Next**.
* Select **Next**.
* Select **Next**.
* Select **Finish**.
* Select **Save** to save the configuration.

Modify default JMS connection factories.

* In the left navigation panel, select **Resources** -> **JMS** -> **Connection factories**.
* Select **built-in-jms-connectionfactory**.
  * Under **Connection**:
    * For **Bus name**, select `CargoTrackerBus`, the one created previously.
  * Select **Apply**.
  * Select **Save** to save the configuration.

Create JMS queues.

* In the left navigation panel, select **Resources** -> **JMS** -> **Queues**.
* Switch scope to **Cluster=MyCluster**.
* Follow the steps to create 5 queues, queue names and JDNI are listed in above table.
  * Select **New**.
  * For **Select JMS resource provider**, select **Default messaging provider**.
  * Select **OK**.
  * In **General Properties** panel, under **Administration**:
    * For **Name**, fill in one of queue names listed in above table, e.g. `HandlingEventRegistrationAttemptQueue`.
    * For **JNDI name**, fill in corresponding JNDI name, e.g. `jms/HandlingEventRegistrationAttemptQueue`.
  * Under **Connection**:
    * For **Bus name**, select `CargoTrackerBus`, the one created in previously.
    * For **Queue name**, select **Create Service Integration Bus Destination**. The selection causes opening a new panel. Input required value.
        * For **Identity**, input the same value with queue name, e.g. `HandlingEventRegistrationAttemptQueue`.
        * Select **Next**.
        * For **Bus member**, select **Cluster=MyCluster**.
        * Select **Next**.
        * Select **Finish**.
  * Select **Apply**.
  * Select **Save** to save the configuration. 
* After 5 queues are completed, continue to create Activation specifications.

Create Activation specifications.

* In the left navigation panel, select **Resources** -> **JMS** -> **Activation specifications**.
* Switch scope to **Cluster=MyCluster**.
* Follow the steps to create 5 activation specifications, names and JDNI are listed in above table.
  * Select **New**.
  * For **Select JMS resource provider**, select **Default messaging provider**.
  * Select **OK**.
  * In **General Properties** panel, under **Administration**:
    * For **Name**, fill in one of queue names listed in above table, e.g. `HandlingEventRegistrationAttemptQueueAS`.
    * For **JNDI name**, fill in corresponding JNDI name, e.g. `jms/HandlingEventRegistrationAttemptQueueAS`.
  * Under **Connection**:
    * For **Destination type**, select **Queue**.
    * For **Destination lookup**, fill in corresponding queue JNDI name listed in the same row of above table. In this example, value is `jms/HandlingEventRegistrationAttemptQueue`.
    * For **Connection factory lookup**, fill in `jms/built-in-jms-connectionfactory`.
    * For **Bus name**, select `CargoTrackerBus`, the one created in previously.
  * Select **Apply**.
  * Select **Save** to save the configuration.
* After 5 activation specifications are completed, you are ready to deploy application.

## Deploy Cargo Tracker

With data source and JMS configured, you are able to deploy the application.

* Open the administrative console in your Web browser and login with WebSphere administrator credentials.
* In the left navigation panel, select **Applications** -> **Application Types** -> **WebSphere enterprise applications**.
* In the **Enterprise Applications** panel, select **Install**.
  * For **Path to the new application**, select **Local file system**.
  * Select **Choose File**, a wizard for uploading files opens.
  * Locate to `cargotracker-was-application/target/cargo-tracker.ear` and upload the EAR file.
  * Select **Next**.
  * Select **Next**.
  * In **Step 1**, select **Next**.
  * In **Step 2**:
    * Check **cargo-tracker.war** from the table.
    * Select **Apply**. 
    * Select **Next**.
  * In **Step 3**, fill in bind listeners for all the beans.
      | Bean name | Listener Bindings | Target Resource JNDI Name | Destination JNDI name |
      |-----------|-------------------|---------------------------|-----------------------|
      | RejectedRegistrationAttemptsConsumer | Activation Specification | jms/RejectedRegistrationAttemptsQueueAS | jms/RejectedRegistrationAttemptsQueue |
      | CargoHandledConsumer | Activation Specification | jms/CargoHandledQueueAS | jms/CargoHandledQueue |
      | MisdirectedCargoConsumer | Activation Specification | jms/MisdirectedCargoQueueAS | jms/MisdirectedCargoQueue |
      | DeliveredCargoConsumer | Activation Specification | jms/DeliveredCargoQueueAS | jms/DeliveredCargoQueue |
      | HandlingEventRegistrationAttemptConsumer | Activation Specification | jms/HandlingEventRegistrationAttemptQueueAS | jms/HandlingEventRegistrationAttemptQueue |
  * Select all the beans using the select all button.
  * Select **Next**.
  * In **Step 4**, check the box next to **cargo-tracker.war**. Select **Next**.
  * In **Step 5**, select **Next**.
  * In **Step 6**:
    * Check the box next to **cargo-tracker.war,WEB-INF/ejb-jar.xml**.
    * Check the box next to **cargo-tracker.war,WEB-INF/web.xml**.
    * Select **Next**.
* Select **Finish**.
* Select **Save** to save the configuration.
* In the table that lists application, select **cargo-tracker-application**.
* Select **Start** to start Cargo Tracker.

## Exercise Cargo Tracker Functionality

1. On the main page, select **Public Tracking Interface** in new window. 

   1. Enter **ABC123** and select **Track!**

   1. Observe what the **next expected activity** is.

1. On the main page, select **Administration Interface**, then, in the left navigation column select **Live** in a new window.  This opens a map view.

   1. Mouse over the pins and find the one for **ABC123**.  Take note of the information in the hover window.

1. On the main page, select **Mobile Event Logger**.  This opens up in a new, small, window.

1. Drop down the menu and select **ABC123**.  Select **Next**.

1. Select the **Location** using the information in the **next expected activity**.  Select **Next**.

1. Select the **Event Type** using the information in the **next expected activity**.  Select **Next**.

1. Select the **Voyage** using the information in the **next expected activity**.  Select **Next**.

1. Set the **Completion Date** a few days in the future.  Select **Next**.

1. Review the information and verify it matches the **next expected activity**.  If not, go back and fix it.  If so, select **Submit**.

1. Back on the **Public Tracking Interface** select **Tracking** then enter **ABC123** and select **Track**.  Observe that different. **next expected activity** is listed.

1. If desired, go back to **Mobile Event Logger** and continue performing the next activity.

---------------------------

# Migration summary

## Packaging and deploying as enterprise application

To resolve CDI error, the sample packs the original Cargo Tracker source code as enterprise application.

Structure of the application:

```
- cargotracker [cargotracker.war]
- cargotracker-was-application [cargotracker.ear]
```

CDI error example:

```
[5/22/24 14:38:04:957 CST] 00000163 SystemErr     R com.ibm.ws.exception.RuntimeError: java.lang.RuntimeException: com.ibm.ws.cdi.CDIRuntimeException: com.ibm.ws.cdi.CDIDeploymentRuntimeException: org.jboss.weld.exceptions.DeploymentException: incompatible InnerClasses attribute between "org.eclipse.cargotracker.domain.model.cargo.Delivery$1" and "org.eclipse.cargotracker.domain.model.cargo.Delivery"
```

## Activation Specifications for Message-driven beans (MDB)

Cargo Tracker uses message-driven beans in JMS module. In WebSphere Application Server, activation specifications are the standardized way to manage and configure the relationship between an MDB. They combine the configuration of connectivity, the Java Message Service (JMS) destination and the runtime characteristics of the MDB, within a single object. For more information, see [Message-driven beans, activation specifications, and listener ports](https://www.ibm.com/docs/fi/was/9.0.5?topic=retrieval-message-driven-beans-activation-specifications-listener-ports).

The sample creates service bus, queues and activation specifications using the default connection factory:

1. Service Bus: `CargoTrackerBus`

   This represents the message bus used for the communication within the system.

1. Connection Factory: `built-in-jms-connectionfactory`

    The connection factory is used by the message-driven beans (MDBs) to establish connections to the messaging system.
    The **CargoTrackerBus** uses the connection factory to configure the activation specifications.

1. Activation Specifications:

    These specify the configuration for the MDBs and determine how they interact with the queues.
    Examples include RejectedRegistrationAttemptsQueueAS, HandlingEventRegistrationAttemptQueueAS, etc.
    Each activation specification is configured with the connection factory.

1. Queues:

    These are the actual destinations where messages are sent and received.
    Examples include RejectedRegistrationAttemptsQueue, HandlingEventRegistrationAttemptQueue, etc.
    Each activation specification activates a corresponding queue.

The general flow is:

* The CargoTrackerBus uses the connection factory.
* The connection factory is used to configure the activation specifications.
* The activation specifications activate the respective queues.

## XA Data Source

XA Data Source refers to a type of data source in Java EE applications that supports distributed transactions using the XA (Extended Architecture) protocol. XA is a two-phase commit protocol that allows transactions to be distributed across multiple resources, ensuring data integrity and consistency across different systems.

This sample configure XA PostgreSQL, using `org.postgresql.xa.PGXADataSource`.

## FlowScoped Faces refactoring

This sample uses a Java class to declare the flow as following:

```java
public class BookingFlow implements Serializable {

    @Produces
    @FlowDefinition
    public Flow defineFlow(@FlowBuilderParameter FlowBuilder flowBuilder) {
        String flowId = "booking";
        flowBuilder.id("", flowId);
        flowBuilder.viewNode(flowId, "/booking/booking.xhtml").markAsStartNode();
        flowBuilder.viewNode("booking-destination", "/booking/booking-destination.xhtml");
        flowBuilder.viewNode("booking-date", "/booking/booking-date.xhtml");
        flowBuilder.returnNode("returnFromBookingFlow").fromOutcome("/admin/dashboard.xhtml");
        
        return flowBuilder.getFlow();
    }
}
```

Besides, in the method `#{booking.register}`, return the view id but not the exact path.
