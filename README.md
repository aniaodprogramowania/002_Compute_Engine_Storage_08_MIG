<p align="center">
Google Cloud Create MIG

</p>

## Step 01: Introduction

### What is an Instance Group?
An **Instance Group (IG)** is a logical collection of **Compute Engine VM instances** that you can manage as a single resource.  
It enables centralized configuration, scaling, and health management for multiple VMs.

---

### Managed Instance Group (MIG) — *Stateless Mode*
A **Managed Instance Group (MIG)** automates the lifecycle of identical VM instances based on a shared instance template.  
In **stateless** mode, all instances are treated as interchangeable and do not retain local state after recreation.

**Key Features:**
1. **Autoscaling** — Automatically adjusts the number of VM instances based on CPU utilization, load, or custom metrics.  
2. **Autohealing** — Automatically recreates failed or unhealthy instances using a defined health check.  
3. **Rolling Updates** — Enables phased updates of instance templates with minimal or zero downtime.  
4. **Multi-Zone Deployment** — Distributes instances across multiple zones to improve availability and resilience.  
5. **Load Balancing** — Integrates seamlessly with backend services to distribute traffic across healthy instances.

---

### Focus of This Demo
In this demo, we will focus on configuring and managing a **Stateless Managed Instance Group (MIG)**, including:

- Creating an instance template  
- Deploying and scaling the instance group  
- Integrating a health check  
- Configuring load balancing

## Step 02: Create Instance Template

Follow the steps below to create an instance template that will be used by the Managed Instance Group (MIG).

---

### Navigation
**Google Cloud Console → Compute Engine → Virtual Machines → Instance Templates → Create Instance Template**

---

### Basic Configuration
- **Name:** `mig-it-stateless-v1`  
- **Location:** `Regional`  
- **Region:** `us-central1`

---

### Machine Configuration
- **Series:** `E2`  
- **Machine Type:** `e2-micro`

---

### Availability Policies
- **VM Provisioning Model:** `Standard`

---

### Additional Settings
- **Display Device:** *Disabled (leave default)*  
- **Confidential VM Service:** *Disabled (leave default)*  
- **Container:** *Disabled (leave default)*  
- **Boot Disk:** *Use default settings*

---

### Identity and API Access
- **Service Account:** `Compute Engine default service account`  
- **Access Scopes:** `Allow default access`

---

### Firewall
- **Allow HTTP traffic**

---

### Advanced Options

#### Management
- **Description:** `startupscript`  
- **Reservations:** *Automatically use created reservation (default)*  

#### Automation
- **Startup Script:**  
  Copy and paste the content from the file:  
  ```t
  wgrywka2.sh
  ```
### Final Step
Click **Create** to deploy the instance template.

---

### Tip
Ensure your startup script (`wgrywka.sh`) installs and configures the web server correctly.  
This script will automatically execute on every VM created from this template.

# Set Project
gcloud config set project PROJECT_ID
gcloud config set project xxx

# Create Instance Template - V1
 ```t
gcloud compute instance-templates create mig-it-stateless-v1 \
  --machine-type=e2-micro \
  --network-interface=network=default,network-tier=PREMIUM \
  --instance-template-region=us-central1 \
  --tags=http-server \
  --metadata-from-file=startup-script=wgrywka2.sh 
```

## Step 03: Create Health Check

Before attaching a health check to the backend service or Managed Instance Group (MIG), ensure that a valid **HTTP health check** exists in your project.  
If no health check is configured yet, create one using the following command:

```t
gcloud compute health-checks create http app1-health-check \
   --project=xxxx \
   --port=80 \
   --request-path=/index.html \
   --proxy-header=NONE \
   --response=Warsztat \
   --region=us-central1 \
   --no-enable-logging \
   --check-interval=10 \
   --timeout=5 \
   --unhealthy-threshold=3 \
   --healthy-threshold=2
```


### Important Configuration Note
If you already have a **different health check** or your application uses a **custom health endpoint** or **response text**,  
make sure to **update** the following parameters in the command:

- **`--request-path`** → Use the correct endpoint path (for example: `/healthz`, `/status`, or `/`).  
- **`--response`** → Match the expected response string returned by your application (for example: `OK`, `Healthy`, or `App2`).  

This ensures that the health check accurately detects healthy instances according to your specific application configuration.


## Step 04: Create a Firewall Rule to Allow Health Check Probes

Google Cloud **health check probes** originate from the following IP address ranges:

- `130.211.0.0/22`  
- `35.191.0.0/16`

Ensure that your firewall rules allow traffic from these ranges to reach your application instances.  
In this example, the Managed Instance Group (MIG) uses the **default network**, and the VM instances listen on **port 80**.

---

### Check if a Firewall Rule Already Exists
Before creating a new rule, verify whether an existing rule already allows health check traffic:

```
gcloud compute firewall-rules list --filter="name~allow-health-check"
```

### Create the Firewall Rule
```
gcloud compute firewall-rules create allow-health-check \
    --allow tcp:80 \
    --source-ranges 130.211.0.0/22,35.191.0.0/16 \
    --network default
```

### Note
If port **80** is already open or covered by a broader firewall rule, you do **not** need to create this rule again.  
This rule ensures that **Google Cloud Load Balancer health check probes** can reach your backend instances on **port 80**.

## Step 05: Create a Managed Instance Group (MIG)

A **Managed Instance Group (MIG)** automatically manages identical virtual machine (VM) instances created from a shared instance template.  
This setup provides scalability, resilience, and simplified management for your application.

---

### 1. Create the Managed Instance Group
```
gcloud compute instance-groups managed create mig-stateless \
   --size=2 \
   --template=projects/xxxx/regions/us-central1/instanceTemplates/mig-it-stateless-v1 \
   --zone=us-central1-c \
   --health-check=projects/xxxx/regions/us-central1/healthChecks/app1-health-check
```

### 2. Set Named Ports
```
gcloud compute instance-groups set-named-ports mig-stateless \
   --zone=us-central1-c \
   --named-ports=webserver-port:80
```

### 3. Configure Autoscaling

**Autoscaling** is a feature of Managed Instance Groups (MIGs) that automatically adjusts the number of VM instances based on real-time workload metrics — such as CPU utilization, load balancing capacity, or custom Cloud Monitoring metrics.  
This helps maintain **optimal performance** while minimizing **costs** by running only as many instances as needed.

---

```
gcloud compute instance-groups managed set-autoscaling mig-stateless \
   --project=xxxx \
   --zone=us-central1-c \
   --cool-down-period=60 \
   --max-num-replicas=6 \
   --min-num-replicas=2 \
   --mode=on \
   --target-cpu-utilization=0.6 \
   --scale-in-control=max-scaled-in-replicas=1,time-window=600
```

## Step 07: Create a Load Balancer

In this step, you will create an **HTTP Load Balancer** to distribute incoming traffic across VM instances in your **Managed Instance Group (MIG)**.

---

### Navigation
Go to:  
**Google Cloud Console → Network Services → Load Balancing → Create Load Balancer → HTTP(S) Load Balancing → START CONFIGURATION**

---

### Basic Setup
- **Internet facing or internal only:** `From internet to my VMs`  
- Click **Continue**  
- **Name:** `lb-for-mig-stateless`

---

### Frontend Configuration
- **New Frontend IP and Port**
  - **Name:** `fip-lb-mig-stateless`
  - **Protocol:** `HTTP`
  - **Network Service Tier:** `Premium`
  - **IP Version:** `IPv4`
  - **IP Address:** Click **CREATE IP ADDRESS**
    - **Name:** `sip-lb-mig-stateless`
    - **Description:** `sip-lb-mig-stateless`
  - **Port:** `80`
- Click **Done**

---

### Backend Configuration
Click **Create a Backend Service**, then provide the following details:

- **Name:** `lb-backend-for-mig-stateless`  
- **Description:** `lb-backend-for-mig-stateless`  
- **Backend Type:** `Instance Group`  
- **Protocol:** `HTTP`  
- **Named Port:** `webserver-port`  
- **Timeout:** `30` seconds  
- **New Backend:** `mig-stateless` *(Managed Instance Group)*  
- **Port Numbers:** `80` *(select the named port `webserver-port` from the instance group)*  
- **Balancing Mode:** `Utilization`  
- **Maximum Backend Utilization:** `80`%  
- **Maximum RPS:** *(leave empty — use defaults)*  
- **Scope:** `Per Instance` *(leave default)*  
- **Capacity:** `100`  

Click **Done** → then **Create**

---

### Logging & Security
- **Logging:** *Unchecked (use defaults)*  
- **Security:** *Unselected (use defaults)*

---

### Review and Finalize
1. **Frontend** — Verify IP, protocol, and port settings  
2. **Backend** — Ensure the correct backend service and health check are attached  
- Click **CREATE**  
- The provisioning process takes approximately **3–5 minutes** to complete.

---

### Final Step
Click **Create** to deploy the **HTTP Load Balancer**.  

Once created, Google Cloud will automatically provision the external IP address and route traffic through the load balancer to your healthy backend instances.

---

## Step 08: Verify Load Balancer Properties After Creation
- Navigate to **Network Services → Load Balancing → lb-for-mig-stateless**  
- Confirm that the **backend instances are marked as Healthy** under the backend service section.  
- Review the **frontend IP address** — this will be used to access your application.

---

### Tip
Once the load balancer is active, test your deployment by visiting its external IP address in a web browser.

## Step 09: Access the Sample Application Using the Load Balancer IP

Once the HTTP Load Balancer has been created and the backend instances are marked as **Healthy**, you can access your sample web application through the **external IP address** assigned to the load balancer.

---

### Access the Application
```
# Access the sample application
http://<LB-IP-ADDRESS>
```

### Observation
- Open the URL in your browser and **refresh the page multiple times**.  
- You should notice that the **response** (for example, the hostname or VM name) **changes between requests**.  
- This confirms that **traffic is being load balanced** across multiple VM instances in the **Managed Instance Group (MIG)**.  
- Each refresh routes your request to a different backend VM instance, demonstrating the **HTTP Load Balancer’s round-robin distribution** and proper operation of your **stateless MIG setup**.

---

### Tip
If you see responses alternating between two different VM hostnames (e.g., `vm-1` and `vm-2`), it verifies that:
- Both instances are **healthy** and actively serving traffic.  
- The **load balancer** is correctly distributing requests between them.  
- Your **startup script** (`wgrywka2.sh`) successfully deployed a functional web server on each VM.

## Step 10: Configure and Manage Schedules

You can configure **instance schedules** in a Managed Instance Group (MIG) to automatically adjust capacity during predictable traffic patterns — for example, scaling up during business hours or high-traffic periods.

---

### Navigation
Go to:  
**Compute Engine → Instance Groups → mig-stateless → DETAILS TAB → MANAGE SCHEDULES → CREATE SCHEDULE**

---

### Schedule Configuration
- **Name:** `high-traffic-tuesday`  
- **Description:** `high-traffic-tuesday`  
- **Minimum Required Instances:** `3`  
  *(Default minimum is 2, but this schedule increases it to 3 on Tuesdays to handle expected high traffic.)*  
- **Start Time:** `7:00 AM`  
- **Time Zone:** `India Standard Time (IST)`  
- **Duration:** `10 hours`  
- **Recurrence:** `Every Week`  
- **Day(s) of Week:** `Tuesday`  

---

### Finalize
1. Click **Save** to create the schedule.  
2. Click **Refresh** to review the applied configuration.  
3. Click **Done** to complete the setup.

---

### Tip
By using **Managed Instance Group schedules**, you can:
- Automatically **increase instance capacity** during known traffic peaks.  
- **Reduce costs** by scaling down outside of scheduled hours.  
- Combine schedules with **autoscaling** to maintain performance efficiency during both predictable and dynamic load changes.

# Create  Schedules
```
gcloud compute instance-groups managed update-autoscaling mig-stateless \
   --project=xxx \
   --zone=us-central1-c \
   --set-schedule=high-traffic-tuesday \
   --schedule-cron='0 7 * * Tue' \
   --schedule-duration-sec=36000 \
   --schedule-time-zone='xxxx/xxxxx' \
   --schedule-min-required-replicas=3 \
   --schedule-description='high-traffic-tuesday'
```
## Step 11: Predictive Autoscaling and Auto-healing

### Step 11-01: Understanding and Configuring Predictive Autoscaling

**Predictive Autoscaling** uses historical usage patterns and machine learning models to **forecast future load** and **proactively scale** your Managed Instance Group (MIG) before the traffic increases.  
Unlike standard (reactive) autoscaling, which adds instances only *after* the load rises, predictive autoscaling launches new VMs *in advance* — ensuring higher availability and smoother performance during traffic spikes.

---

### Navigation
Go to:  
**Compute Engine → Instance Groups → mig-stateless → Edit → Predictive Autoscaling → See if predictive autoscaling can optimize your availability**

---

### How It Works
- Google Cloud analyzes **CPU utilization trends** and **autoscaler metrics** over time.  
- Based on learned patterns (for example, recurring daily or weekly traffic peaks), it **pre-scales** your instances a few minutes before demand increases.  
- Predictive autoscaling combines with **reactive autoscaling** — predictions guide when to scale up, while reactive rules ensure scaling down as load decreases.

---

### Benefits
- **Improved performance** — new VMs are ready before users arrive.  
- **Reduced cold start time** — avoids delays caused by VM provisioning during sudden load increases.  
- **Balanced cost and availability** — scales efficiently based on actual patterns rather than constant overprovisioning.  

---

### How to Enable
1. Go to your **Managed Instance Group** (e.g., `mig-stateless`).  
2. Click **Edit**.  
3. Scroll to the **Autoscaling** section.  
4. Locate **Predictive Autoscaling** and click:  
   **“See if predictive autoscaling can optimize your availability.”**  
5. Select **Enabled (Optimize for Availability)**.  
6. Click **Save** to apply changes.

---

### Tip
Predictive autoscaling works best for **applications with consistent and repeating traffic patterns**, such as:
- Business-hour traffic surges  
- Daily API load fluctuations  
- Periodic batch or scheduled processing  

For unpredictable workloads, **standard autoscaling** may be sufficient, but enabling predictive mode adds proactive scaling capability without removing reactive scaling logic.

### Step 11-02: Auto-healing

**Auto-healing** is a key feature of Managed Instance Groups (MIGs) that automatically detects and replaces **unhealthy VM instances** based on the configured **health check**.  
If an instance stops serving traffic correctly (for example, due to application or OS failure), the MIG automatically recreates that instance to restore availability.

---

### Test Auto-healing Behavior

Connect to one of the VM instances in the MIG and simulate an application failure by removing the web server’s main page.

```
# Connect to one VM instance
gcloud compute ssh --zone "us-central1-c" "mig-stateless-xxxx" --project "xxxx"

# Remove the web server's index file
cd /var/www/html
sudo rm index.html
sudo rm index.nginx-debian.html
```

### Verify Application Response
Access the VM’s **public IP address** in a browser:

---

### Observation
- You should see an **HTTP 403 Forbidden** or **404 Not Found** error from **NGINX**, confirming that the web application content has been removed.  
- The **health check** will soon detect this instance as **unhealthy**.

---

### Auto-healing Observation
- The **Managed Instance Group (MIG)** will automatically **recreate** the unhealthy VM instance using the original **instance template**.  
- The newly created instance will have the **same instance name** and will automatically **re-run the startup script**, restoring the web server and the default index page.

---

### Tip
Auto-healing ensures that:
- Your application remains **highly available**, even if individual VM instances fail.  
- Recovery happens **automatically**, without manual intervention.  
- The newly created instance is **identical** to others — same configuration, template, and startup behavior.

# Step 12: Update VM Instances in the Managed Instance Group (MIG)

As your application evolves, you may need to update the **instance configuration** — for example, deploying a new version of the web server, adjusting machine types, or applying security patches.  
To roll out such changes safely in a **Managed Instance Group (MIG)**, you first create a **new instance template** and then update the group to use it.

---

### Step 12-01: Create a New Instance Template (Version 2)

In this step, we create a **v2 instance template** that includes updated configuration and startup script.

```
# Create Instance Template - V2
gcloud compute instance-templates create mig-it-stateless-v2 \
  --machine-type=e2-small \
  --network-interface=network=default,network-tier=PREMIUM \
  --instance-template-region=us-central1 \
  --tags=http-server \
  --metadata-from-file=startup-script=wgrywka3.sh
```
### Why Create a Second Template?

Creating a new instance template (**v2**) allows you to:

- Roll out **application updates** or **configuration changes** without downtime.  
- Use **rolling updates** in the MIG — replacing VMs gradually, one at a time.  
- Preserve the ability to **rollback** to the previous version (**v1**) if needed.  
- Maintain **version control** and consistent deployment history across environments.

---

###  Tip
After creating the new template, update the **Managed Instance Group (MIG)** to use it:

```
gcloud compute instance-groups managed set-instance-template mig-stateless \
  --template=projects/xxxx/regions/us-central1/instanceTemplates/mig-it-stateless-v2 \
  --zone=us-central1-c
```

## Step 12-02: Updating VMs in the Managed Instance Group (MIG)

When you update an instance template, you need to apply it to your existing VMs within the **Managed Instance Group (MIG)**.  
This can be done in two ways — **Automatic (Proactive)** or **Selective (Opportunistic)** — depending on your deployment strategy.

---

### Update Type: Automatic (Proactive)

When **automatic updates** are enabled, the MIG proactively updates all instances to use the new instance template.  
Google Cloud automatically replaces existing VMs according to the defined rollout strategy.

**API Type:** `PROACTIVE`

**Example Scenario:**
- You are deploying template **`mig-it-stateless-v1`** to **50%** of instances and template **`mig-it-stateless-v2`** to the remaining **50%** in the instance group **`mig-stateless`**.  
- One instance will be **taken offline at a time**, and **one temporary instance** will be added to maintain capacity during the rolling update.

*Command:**
```
# Update VMs Automatically (Proactive)
gcloud compute instance-groups managed rolling-action start-update mig-stateless \
   --project=xxxx \
   --type='proactive' \
   --max-surge=1 \
   --max-unavailable=1 \
   --min-ready=0 \
   --minimal-action='replace' \
   --most-disruptive-allowed-action='replace' \
   --replacement-method='substitute' \
   --version=template=projects/xxxxx/regions/us-central1/instanceTemplates/mig-it-stateless-v2 \
   --zone=us-central1-c
```

### Update Type: Selective (Opportunistic)

**Selective updates** allow you to control **when and how** updates are rolled out.  
You can manually trigger the update for the group, and the MIG replaces instances **gradually** or only when natural opportunities occur (such as recreation or healing events).

**API Type:** `OPPORTUNISTIC`

---

### Command
```
# Update VMs Selectively (Opportunistic)
gcloud compute instance-groups managed rolling-action start-update mig-stateless \
   --project=xxxx \
   --type='opportunistic' \
   --max-surge=1 \
   --max-unavailable=1 \
   --min-ready=0 \
   --minimal-action='replace' \
   --most-disruptive-allowed-action='' \
   --replacement-method='substitute' \
   --version=template=projects/xxxxx/regions/us-central1/instanceTemplates/mig-it-stateless-v2 \
   --zone=us-central1-c
```

### Tip
After starting a **rolling update**, you can monitor its progress using the following command:
```
gcloud compute instance-groups managed list-instances mig-stateless --zone=us-central1-c
```

You’ll see VMs gradually being replaced by instances built from the new template (mig-it-stateless-v2), ensuring zero downtime and a consistent configuration across the Managed Instance Group (MIG).

### Step 12-03: Verify the VM Instances

After initiating the rolling update, verify that the **Managed Instance Group (MIG)** has successfully replaced old instances with the new configuration defined in **`mig-it-stateless-v2`**.

---

### Navigation
Go to:  
**Compute Engine → VM Instances**

**You should observe:**
- One or more VMs created using the new **machine type** `e2-small`  
- Eventually, the group stabilizes with **2 VM instances** — both using `e2-small`  
- The new **version (v2)** of your web application is deployed and serving traffic  

**Observation:**  
The update process may take several minutes (approximately **10 minutes**) to complete and reach the desired state.

---

### Test Application Access
Use the load balancer’s external IP address to verify that the updated VMs are serving traffic correctly.

```
# Access the application using the Load Balancer IP
curl http://35.209.128.93/
```

### Summary
At this stage:

- The **Managed Instance Group (MIG)** has completed the **rolling update** using the new instance template **`mig-it-stateless-v2`**.  
- All instances are now running on the **`e2-small`** machine type with the updated **v2 web application**.  
- The **Load Balancer** continues to distribute traffic seamlessly, ensuring **zero downtime** throughout the update process.


## Step 13: Explore the DELETE INSTANCE Setting

In a Managed Instance Group (MIG), deleting an instance manually does **not** reduce the group size.  
The MIG automatically **recreates the deleted instance** using the assigned instance template to maintain the desired group size.

---

### Steps
1. Go to **Compute Engine → Instance Groups → mig-stateless**.  
2. Select any instance and click **DELETE**.  
3. Wait for a few minutes.  
4. Observe that a **new instance** is automatically created to replace the deleted one.

---

## Step 14: Explore the REMOVE FROM GROUP Setting

Removing an instance from a Managed Instance Group **detaches it** from group management — it will no longer be monitored, autohealed, or managed by the MIG.  
The MIG will still create a **new instance** to maintain the target size.

---

### Steps
1. Go to **Compute Engine → Instance Groups → mig-stateless**.  
2. Select an instance and click **REMOVE FROM GROUP**.  
3. Wait for a few minutes.  
4. Observe that a **new instance** is created to maintain group capacity.  
5. Then, go to **Compute Engine → VM Instances**, and once the **Delete** option is enabled, **delete** the detached instance manually.

---

## Step 15: Explore RESTART and REPLACE VM Settings

You can manually restart or replace instances in the MIG to apply updates, restart applications, or refresh instance health.

---

###  Step 15-01: Restart Option
This restarts instances in the group without recreating them.

**Configuration:**
- **Maximum unavailable:** `1 instance`  
- **Minimum wait time:** `0 seconds`  

Click **RESTART** to initiate the restart operation.

---

### Step 15-02: Replace Option
This option recreates VM instances using the same instance template — useful for testing autohealing or refreshing configurations.

**Configuration:**
- **Maximum surge:** `1 instance`  
  *(Maximum number or percentage of temporary instances added during replacement.)*  
- **Maximum unavailable:** `1 instance`  
- **Minimum wait time:** `0 seconds`  

Click **REPLACE** to begin the replacement process.

---

## Step 16: Review the Monitoring Tab

The **Monitoring** tab provides insights into MIG operations such as scaling, instance health, and lifecycle activities.

---

### Steps
1. Go to **Compute Engine → Instance Groups → mig-stateless**.  
2. Click the **MONITORING** tab.  
3. Review the **Instances graph** — it shows how the group scaled **out** and **in** over time based on autoscaling policies and load patterns.

## Step 20: Create a Multi-Zone Managed Instance Group (Stateless)

In this step, we will create a **Managed Instance Group (MIG)** that spans **multiple zones** within a region.  
Multi-zone MIGs improve **availability**, **resilience**, and **fault tolerance** by distributing VM instances evenly across zones.

---

### ⚙️ Configuration Details

- **Name:** `mig-stateless-multiple-zones`  
- **Description:** `mig-stateless-multiple-zones`  
- **Location:** `Multiple Zones`  
- **Region:** `us-central1`  
- **Zones:** `us-central1-a`, `us-central1-b`, `us-central1-c`, `us-central1-f`  
- **Target Distribution Shape:** `Even`  
  *(Distributes instances evenly across all selected zones.)*

---

### Port Mapping
- **Specify Port Name Mapping:**  
  - **Port Name:** `webserver-port`  
  - **Port Number:** `80`

---

### Instance Template
- **Instance Template:** `xxxxx`

---

### Autoscaling Configuration
- **Autoscaling Mode:** `Autoscale`  
  *(There are three modes — discussed below.)*  
- **Autoscaling Policy:**  
  - **CPU Utilization:** 60%  
  *(Other autoscaling policies include **HTTP Load Balancer Metrics** and **Cloud Monitoring (Stackdriver) Metrics** — see discussion below.)*  
- **Predictive Autoscaling:** `Off`  
- **Cool-down Period:** `60 seconds`  
- **Minimum Number of Instances:** `3`  
- **Maximum Number of Instances:** `9`

#### Scale-in Controls
- **Enable Scale-in Controls:** Enabled  
- **Don’t scale in by more than:** `1 instance`  
- **Over the course of:** `5 minutes`

---

### Autohealing
- **Health Check:** *None (for this demo)*  
  *(In production, a health check is recommended to allow automatic instance recovery.)*

---

### Advanced Creation Options
Leave all other settings as **default**.  
You can optionally review settings such as **Statefulness**, **Instance Redistribution**, and **Maintenance Policy** for advanced configurations.

---

### Final Step
Click **CREATE** to deploy the **multi-zone stateless Managed Instance Group**.  
The group will automatically distribute instances across multiple zones for higher availability.

---

## 🔍 Step 21: Review Managed Instance Group Properties

After creation, review the MIG configuration and confirm instances have been deployed correctly across zones.

---

### Navigation
Go to:  
**Compute Engine → Instance Groups → mig-stateless-multiple-zones**

Review the following tabs:

1. **Overview** — Summary of group settings, size, and status.  
2. **Details** — Template version, autoscaling configuration, and distribution.  
3. **Monitoring** — Metrics showing scaling behavior and performance.  
4. **Errors** — Check for any provisioning or configuration issues.

Then, go to:  
**Compute Engine → VM Instances**

- Verify that VM instances were created by the MIG.  
- Confirm that instances are **spread across multiple zones** within the **us-central1** region.

---

## Step 22: Delete the Managed Instance Group

Once you have completed reviewing the multi-zone MIG, you can delete it to clean up resources.

---

### Steps
1. Go to **Compute Engine → Instance Groups → mig-stateless-multiple-zones**.  
2. Click **DELETE GROUP**.  
3. Confirm the deletion when prompted.

---

### Tip
Deleting the MIG will:
- Automatically delete all VM instances created by it.  
- Release underlying resources associated with those instances.  
- Not affect your instance template or other unrelated MIGs.
