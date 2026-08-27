Lab 1: High Availability & Efficient Load Distribution
######################################################

**Business problem** : AI pipelines need consistent, high‑throughput access to S3-compatible storage. Wiring
clients directly to specific storage nodes creates tight coupling and operational risk: a single overloaded node
throttles the entire pipeline.

**Technical problem** : Without a delivery layer, clients must pick a node, handle retries/failover, and live with
uneven utilization (hot spots) and brittle endpoints.

**Solution with BIG‑IP LTM** : Expose a single, resilient virtual endpoint. Behind this VIP, BIG‑IP intelligently
distributes S3 traffic across all healthy MinIO nodes and lets you scale by simply adding/removing pool
members—no client changes required. Use Least Connections to smooth throughput for S3 workloads.

Following the tasks in the prior **Introduction** Section, you should now be able to access the
UDF lab environment.

Task 1: Review the Lab Environment
==================================

These values align with the UDF topology. Keep them unchanged unless your
environment differs.

.. table:: Lab environment

   ========================  =========================================  ==================================
   Component                 Purpose                                    Where to access
   ========================  =========================================  ==================================
   MinIO Cluster‑1 (direct)  Baseline test without BIG‑IP               WARP parameters: 10.1.10.100:9000
   BIG‑IP VIP for Cluster‑1  Single front door with LTM load            WARP parameters: 10.1.40.160:9000
   MinIO Console             Review node-level metrics                  UDF → Cluster1-Node1 → Access → UI
   BIG‑IP TMUI               Verify pools/members and methods           UDF → BIG‑IP → Access → TMUI
   ========================  =========================================  ==================================



Task 2: Baseline: Send traffic directly to MinIO (no BIG‑IP)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following steps will validate access to the application via a web browser, review the
Performance Monitoring dashboard, and gather request details.

**Quick aside, what is Warp?**  It is a high-performance, open-source benchmarking tool designed to measure and
analyze the throughput and latency of S3-compatible object storage systems.

Warp simulates real-world workloads—such as PUT (uploads), GET (downloads), LIST, and DELETE operations—to
evaluate performance, plan capacity, and identify bottlenecks.

The labs offer an easy-to-use graphical front end for Warp, to avoid needing to issue command-line actions.

1. Open MinIO WARP (UDF → Components → Traffic‑Gen → Access → Firefox - you may need to scroll down).
   The credentials are under lab   Documentation tab (admin/admin).
   If presented with Firefox "Restoring Pages" message, simply choose "Restore
   Session" button.   As well, permit the pop-up to allow access to clipboard.

2. Select the cluster‑1 profile.

3. Select all **3** buckets, when selected for use they will appear in bright orange.

4. Set Duration to 3 minutes and Concurrency to 20 threads. Conncurrency refers to parallel S3 transactions.

5. In WARP Parameters, set Endpoint to 10.1.10.100:9000.

6. Click Run Benchmark.

.. image:: ../_static/image_001_fixed_WARP_ui.png



7. Observer there are two clusters of MinIO servers in our lab.
   Open MinIO cluster‑1 Console (UDF → Cluster1‑Node1 → Access → UI). Login: minioadmin / minioadmin

8. Observe that there are 2 AIStor active servers in cluster 1, however data charts require normally
   30 minutes or more to popluate so expect no traffic on the right-hand chart.

   **REMINDER of ABOVE** You will not see the charts below filling for at least the fist 20 minutes, in later
   steps you will notice them filled.

.. image:: ../_static/summary_view_of_data_no_load_balancer.png



9.  Click on the arrow next to **time to first byte**, in the lower right of the screen.

10. Observe that once the charts populate, only traffic will be received by the  first AIStor,
    at address 10.1.10.100 port 9000.  This traffic will task one server, creating a **hot spot** of load.

.. image:: ../_static/detailed_view_of_trafic_without_a_load_balancer.png





Why this matters:  Clients that target a single node are brittle, utilization is uneven and scalability suffers.





Task 3: Baseline: Steer (proxy) the Same Workload through BIG-IP
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~




1. In WARP, switch target to the BIG‑IP VIP profile **BigIP‑cluster‑1**.  Keep all three buckets active.

2. Set Duration: 5 minutes (300 seconds), Concurrency: 20 threads.

3. In WARP Parameters, set Endpoint to 10.1.40.160:9000 (as we are now targeting a virtual server).

4. Click Run Benchmark

5. Log into BIG-IP TMU: Local Traffic → Pools → Cluster‑1.  Confirm 2 pool members are present initially
   Click both Members and Statistics tabs.  **Important:** As an S3 best practice adjust the load
   balancing algorithm to "Least Connections".
.. note::
   Due to short run durations, summary analytics may not have appeared in the AIStor dashboard view yet.

.. image:: ../_static/pool_members.png

.. image:: ../_static/even_traffic.png

6. Confirm the WARP S3 load generator has run to completion, traffic settings can be seen below.  The AISTor
   GUI will likley not populate for a few more minutes, but later when checked will look like the AIStor
   screenshot provided below, set time frame to last 4 hours in second screen.   Data points are updated
   updated every 15 minutes, your first two test runs may be reflected in only one data point unlike the
   provided screenshot below.
.. image:: ../_static/a_warp_setup.png

.. image:: ../_static/before_and_after.png

**Expected outcome**: Traffic is distributed across the **two** nodes behind the VIP.

**Load‑balancing method**: This pool is configured for Least Connections, recommended for S3‑style
workloads to reduce request skew.

Task 4: Scale out easily: add the 3rd and 4th MinIO AIStor nodes to Cluster 1
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In MinIO terminology, a storage pool (often referred to as a server pool) is a set of MinIO server nodes that aggregate their drives and resources to act as a single,
independent unit of storage capacity. It is the fundamental unit for scaling, expanding, and managing capacity in a distributed MinIO deployment.

To re-iterate, a storage pool is a **unit of expansion**:  When you need more capacity, you add a new storage pool to your existing deployment. This allows for horizontal scaling.

Outcome desired:  Expand Cluster 1 from 2 nodes (Storage Pool 1) to 4 nodes (Storage Pool 1 + Storage Pool 2).   We will use the storage prefix to avoid confusion
with BIG-IP **origin pools**, which frequently are just simply referred to as BIG-IP pools just simply pools.

**Prerequsites to this lab task 4**

- Cluster 1 Storage Pool 1 is running on nodes 1 & 2 (10.1.10.100, 10.1.10.101)
- Storage Pool 2 nodes 3 & 4 (10.1.10.102, 10.1.10.103) are pre-configured but **not started**

**Step 1 — Verify current cluster state**

Open a **web shell** session to **cluster1-node1** (equivalent to an SSH session) and confirm the cluster is healthy:

.. code-block:: shell
   mcli admin info cluster1

You should see 2 nodes (MinIO Servers) and 1 pool.  You will also observe details about erasure coding chunks and elements of metadata.

**Step 2 — Update MINIO_VOLUMES on Storage Pool 1 nodes**

Open another web shell session to **cluster1-node2**, so that you have sessions to both nodes. On **both** nodes, edit the MinIO environment file:

.. code-block:: shell
   sudo vi /etc/default/minio

If you are not comfortable with the ``vi`` editor, you can use ``nano`` instead:

.. code-block:: shell
   sudo nano /etc/default/minio

- **Comment out** the single storage pool MINIO_VOLUMES line
- **Uncomment (eg ADD)** the two storage :wpool MINIO_VOLUMES line

The result should look like:

.. code-block:: shell
   # MINIO_VOLUMES="http://10.1.10.{100...101}:9000/mnt/minio"
   MINIO_VOLUMES="http://10.1.10.{100...101}:9000/mnt/minio http://10.1.10.{102...103}:9000/mnt/minio"

*Note: using curly braces in the MINIO_VOLUMES statement allows a simple way to group servers (nodes) when one uses contiguous IP address blocks*

**Step 3 — Start MinIO on Storage Pool 2 nodes**

Open web shell sessions to cluster1-node3 and cluster1-node4.

On both nodes, start the MinIO service:

.. code-block:: shell
   sudo systemctl start minio

These nodes already have the two storage pool MINIO_VOLUMES pre-configured.
The issued command will *not* provide a return to the Linux prompt, the Minio service is activating but we need to restart Cluster1 to accept the new
storage pool.

**Step 4 — Restart Cluster 1 to pick up the new 4 server topology**

Back on cluster1-node1, restart the cluster. Make sure you are using the browser tab for node1. If in doubt, run the following command and confirm the last address shown is ``10.1.10.100/24``:

.. code-block:: shell
   ip address

Then restart the cluster:

.. code-block:: shell
   mcli admin service stop cluster1
   mcli admin service restart cluster1

This restarts all MinIO processes across the cluster, causing storage pool 1 nodes to recognize the new two-pool topology and storage pool 2 nodes to join.

**Step 5 — Verify the expanded cluster**

On cluster1-node1, confirm the expansion:

.. code-block:: shell
   mcli admin info cluster1

You should now see 4 nodes and 2 storage pools.

.. image:: ../_static/mcli_info_cluster1.png

**Brief aside on the MinIO mcli command**

mcli, which until recently was just simply *mc* is MinIO's utility to both administer and monitor AIStor clusters.  It is installed on each
node in the lab, but can equally be installed on any Linux or Windows host, providing a powerful way to undrestand your environment.

To make things easy, mcli allows the setting of an alias for each device, such as an AIStor server using the following syntax:

.. code-block:: shell
   mcli alias set <ALIAS> <ENDPOINT> <ACCESS_KEY> <SECRET_KEY>

*Note: as opposed to using an S3 access key and secret key, one can also utilize user ids and passwords when reaching a node*

For instance, to see the alias values for this lab, run the following command on cluster1-node1:

.. code-block:: shell
   mcli alias list

Since this is a lab, the passwords for nodes that are used are trivial, in production use best practice to set values.

To see the buckets on cluster1 from node1, run:

.. code-block:: shell
   mcli ls cluster1

.. image:: ../_static/mcli_2_commands.png

Many familiar looking Linux/Unix commands, like the ls example above, can be harnessed by simply prefacing mcli to a command and choosing an alias.

The second administrative command in the screenshot above shows an example of tracing where the erasure coded chunks of a given sample object are
actually stored, along with meta data details.

*Note: in the output of our command we see the chunks are stored on storage pool 1 members 10.1.10.100 and 10.1.10.101.   It's worth noting that although
a second storage pool was added, and any cluster member will service S3 read requests, simply expanding the cluster does not re-distribute content already
written previously to storage.*


Task 5: Scale out easily: add the 3rd and 4th MinIO AIStor node to the BIG-IP origin pool
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. In BIG‑IP TMUI: Local Traffic → Pools → Cluster‑1 → Members.  Add a New member and choose Node List menu.

.. image:: ../_static/a_update_pool_members_list.png



2. Click **Add.... -> Node List ->** select cl1-nd3

3. Set **Service Port** to 9000 and click **Finished**

.. note::
     Health checks for the new member will drive the LED from blue (as highlighted in the following
     image) to green (ready) shortly

4. Repeat for node 4 (cl1-nd4).

.. image:: ../_static/pool_list_expanded_to_4_nodes.png



4. Re-run the WARP workload targetting the BIG-IP Virtual IP (VIP 10.1.40.160:9000)

.. note::
   The 3rd and 4th AIStor nodes begin processing traffic **immediately**. All nodes now share the load.

.. image:: ../_static/traffic_across_4_nodes.png

.. image:: ../_static/aistor_4_nodes_working.png

**You will see the presence of the new nodes but data may not yet be reflected in the MinIO screen shown above,
set the time to last 4 hours and data will appear in a few minutes.   By default data points in the AIStor charts
update on the 15-minute mark through the day.**

**Key Takeaway** There were no client changes and S3 applications still continue to talk to the same VIP;
topology changes are absorbed by *BIG‑IP* at the dataplane.


Task 6:  Verify the load‑balancing method & pool health
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following steps will guide you through the rich, visual metrics presented for BIG-IP through the AST
dashboards powered by Grafana.   We will verify the load balancing method and pool health.

1. In BIG‑IP TMUI: Local Traffic → Pools → Cluster‑1.

2. As good practice, confirm Load Balancing Method: Least Connections.  This is seen on the
   Pool Members list screen.

3. Check Members tab:  All members green (up) with active connections.

4. Use the AST tool (to review the Dashboards) UDF -> AST -> Access -> Grafana.
   Login as **admin / admin**, when prompted to change password simply retain the value as **admin** or
   simply click the "skip" hyperlink.

5. In AST, choose Dashboards - BigIP - Device (expand group) - select Device Pools look at the key metrics,
   such as **Active Pool Member Connections**.   For "Pool" in top menu, adjust to "Cluster-1" instead of
   of "All" and change the time range to inspect just the last 15 minutes.


6. Click on the "3 dots" menu → View on any graphical widget to see the full panel.  Click "Refresh" often.




.. image:: ../_static/a_ast_overview_charts.png


The charts indicate an traffic being distributed across all MinIO AIStor nodes, hot spots have been successfully avoided.


Validation
~~~~~~~~~~

MinIO AIStor Console (Main Screen → Data -> arrow next to Time to First Byte): when populated with data points
shows very close to even, per‑node traffic once proxied via VIP (some variability is expected / OK).

BIG‑IP Pool Stats: show all 4 members up with active connections.

End‑user impact: Client endpoint unchanged; backend scaling is transparent.

Troubleshooting
~~~~~~~~~~~~~~~

WARP can't connect
 -> Re‑check endpoint (10.1.10.100:9000 direct vs 10.1.40.160:9000 VIP).
 -> Ensure the http/https scheme matches your setup (WARP must use the correct protocol).

MinIO AIStore UI Metrics don't update
 -> the page isn't real‑time, hit "Refresh" with last 4 hours timerange, towards end of lab to see data

A pool member is down (red)
 -> Verify the MinIO node process.
 -> Review the pool's Monitor and node address/port.

Skew remains after adding cl1‑nd4
 -> Confirm the new member is enabled and passing its monitor.
 -> Ensure Least Connections is applied (not Round Robin).


Clean-Up (Optional)
~~~~~~~~~~~~~~~~~~~

Stop any running WARP tests.

If you temporarily disabled/enabled members, restore their original state.

Leave Cluster‑1 with 4 active members for Labs 2 and 3.


What You Learned - Value of BIG-IP LTM and AIStor
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Decoupling** Clients connect to one VIP; storage topology can evolve freely.

**Efficiency** Least Connections smooths throughput and reduces hotspots.

**Elasticity** Scale by adding/removing pool members without client changes.

**Operational simplicity** Traffic engineering lives at the dataplane, not in every client.



**End of Lab 1:**  This concludes Lab 1.  In this lab you ran high rate S3 loads against a single MinIO
AIStor server, leaving other AIStor instances unused.  You then adjusted the load generator to use a virtual
server with a virtual IP address on the F5 BIG-IP.   An origin pool corresponding to three AIStor instances
allowed traffic to be spread, per the least connections approach, to all healthy nodes.   A fourth AIStor
instance was added to the pool, exercising all nodes equally and not requiring any client side adjustments,
a major benefit in scenarios with hundreds or possibly thousands of clients.

