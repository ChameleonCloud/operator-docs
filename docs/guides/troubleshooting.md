
### Getting a reservation for a node to maintain

Example: We need to replace SSDs in nodes P3-SSD-009 and P3-SSD-010, but they are currently reserved. Therefore, we'll need to make our own reservation for a future time.

#### Step 1: Find out when the nodes are currently reserved.

We go to https://chi.uc.chameleoncloud.org/project/leases/calendar/host/ , then select the node type of "storage\_nvme"

![image](https://gist.github.com/assets/18199349/2793a967-d092-42f6-ae04-fb6da9dc54ff) The bars go off the right side of the screen, so we zoom out to 30 days. ![image](https://gist.github.com/assets/18199349/e8902663-04ff-40ff-a95f-9553c45d8dec)

It's not clear what date the lease ends visually, but we can hover over the tooltips and get timestamps.

*   Current lease:

    ![image](https://gist.github.com/assets/18199349/c5c27990-42b5-4f60-973d-acebd3c18049)
*   Blocking lease:

    ![image](https://gist.github.com/assets/18199349/ad02a76a-5817-4857-b48b-e2514393d412)

So we can make a lease either between 2024/02/21 and 2024/02/23, or after 2024/03/01.

We don't know the timezone, so we'll try both UTC and Central Time (Chicago)

#### Step 2, make a lease(s)

We want to reserve two different nodes by name, and Blazar doesn't support "OR" parameters. We could specify an array of reservations, but the syntax is difficult via CLI, and not supported in horizon.

So, instead we'll make two leases.

Here, we fill in parameters to try and fit a lease in on the 21st-23rd. ![image](https://gist.github.com/assets/18199349/4a1c456c-1ed4-4427-b140-f3df9a612d6a)

It tells us not to specify seconds, even though the lease calendar shows them. ![image](https://gist.github.com/assets/18199349/7e3dd4ec-b130-4c4d-a46e-f0dbc611fd51)

We specify the node by name, and set quantity min=max=1 ![image](https://gist.github.com/assets/18199349/26bc1dce-186b-405b-b49c-e0357dd1ef7e)

We click create lease, and hooray, it worked! If we guessed wrong, we would have started this step over.

Now we do it a second time for node p3-ssd-009.

#### Using the leases

Once the leases start, we'll know that users do NOT have the nodes. At that time, we'll use the openstack CLI to:

1. set the nodes to maintenance mode in ironic
   1. `openstack baremetal node maintenance set p3-ssd-009`
   2. `openstack baremetal node maintenance set p3-ssd-010`
2. view current maintenance status with `openstack baremetal node show p3-ssd-009`
3. Now that the node have maintenance mode set in ironic, it will no longer enforce power state. This means that powering the node on and off with idrac or the physical power button will work as expected. However, openstack related operations (node cleaning, inspection, provisioning), will be blocked until maintenance mode is unset.

## Inspecting a Node

To run some basic health checks on a node (that isn't in use), we can use ironic's inpection feature. This will network boot a node into the ironic agent ramdisk, and then report a bunch of information about the node to ironic. Besides being used to populate the reference-api, it's a good sanity check for "Can ironic successfully boot and manage this node", e.g. if inspection fails, then deploying instances will also fail. (However, if inspection succeeds, this does not necessarily guarantee that deploying an instance will too).

1. Check the current state of the node: `openstack baremetal node show <node_name_or_uuid>`
2. If the `provision_state` is _not_ `active`, then we can proceed, and set the node to the manageable state via `openstack baremetal node manage <node_name_or_uuid>`
3. Start the inspection by running `openstack baremetal node inspect <node_name_or_uuid>`
4. After about 15 minutes, the node will have either finished, and moved back to state `manageable`, or failed and moved to state `inspect failed`. If it failed, look into the logs and see why, similar to failed deployments. If it succeeded, move it back to available via `openstack baremetal node provide`

{% hint style="warning" %}
If the node inspected successfully, make sure to move it back to `available`! Otherwise, it won't be available for users.&#x20;
{% endhint %}

## Wiping a Node's disks (cleaning)

To wipe data on a node, you can use ironic's "cleaning" feature.

{% embed url="https://docs.openstack.org/ironic/xena/admin/cleaning.html#manual-cleaning" %}

After verifying that you do indeed want to erase all disks on a node:

```sh
# Discover current "provisioning_state" of a node
openstack baremetal node show \
-c provision_state \
<node_name_or_uuid>

# before cleaning can be triggered, the node must be "manageable"
openstack baremetal node manage <node_name_or_uuid>

# you can choose to either:
# Erase partition tables and similar (quick)
openstack baremetal node clean \
--clean-steps '[{"interface": "deploy", "step": "erase_devices_metadata"}]' \
<node_name_or_uuid>

# or do a full erasure. If devices support hardware erasure 
# (nvme secure erase, etc), this step will use it.
openstack baremetal node clean \
--clean-steps '[{"interface": "deploy", "step": "erase_devices"}]' \
<node_name_or_uuid>

# these steps will reboot a node, pxe boot it into the ironic agent ramdisk, 
# execute these commands, and then shut down.
# the states will transition from manageable->clean-wait->cleaning->manageable

# when done, you will need to make the node available for use again
# setting the status to available by running
openstack baremetal node provide <node_name_or_uuid>
```



## Recovering data from deleted baremetal instances

(We don't do this usually - only in exceptional cases if the instance got deleted due to an unexpected issue in the system) In case an instance on a baremetal node is deleted before the user is able to backup their data (for example in the case of an expired lease), data can be recovered as long as it is not overwritten by the launch of a new instance on the same node.


## Lease Fails to start

### IP address \<ip address> already allocated in subnet \<subnet\_id>

This message is found in blazar logs with the errored LeaseID that failed to start

Get the following information and execute the command in the controller node of the site

```bash
ORIGINAL_LEASE_ID="<lease id of the lease that fails to start>"
# Get reservation ID from horizon or from `openstack reservation lease show` 
ORIGINAL_LEASE_IP_RESERVATION_ID="<reservation id of the IP in the lease>"
IP="<IP address that is stuck in subnet>"
```

If you do not know the IP that is stuck, execute the below commands

```bash
FIP_IDS=$(mysql blazar -e "select floatingip_id from floatingip_allocations where reservation_id = '$ORIGINAL_LEASE_IP_RESERVATION_ID';" | egrep '[0-9a-f]{8}-([0-9a-f]{4}-){3}[0-9a-f]+' -o)
echo $FIP_IDS
# get the ip address
for FIP_ID in $FIP_IDS
do
    mysql blazar -e "select id, floating_ip_address from floatingips where id = '$FIP_ID';"
done
```

We need to get the Lease ID of a reservation with which the IP address got stuck with

```bash
# see if IP exists and get the reservations ID tagged
# get this from 
`openstack floating ip show <IP> | egrep reservation | egrep '[0-9a-f]{8}-([0-9a-f]{4}-){3}[0-9a-f]+' -o`
```

```bash
TAGGED_RESERVATION_ID="<the reservation ID from above command>"
# get the lease id that got the IP stuck with
IP_LEASE_ID=$(mysql blazar -e "select lease_id from reservations where id ='$TAGGED_RESERVATION_ID';" | egrep '[0-9a-f]{8}-([0-9a-f]{4}-){3}[0-9a-f]+' -o)
# check if the lease ended
echo $(mysql -t blazar -e "select * from leases where id ='$IP_LEASE_ID';")
```

<pre class="language-bash"><code class="lang-bash"><strong># Delete the IP
</strong>openstack floating ip delete &#x3C;IP>
</code></pre>

We will need to update the current/original lease's IP reservation status, lease status, and lease event status

```bash
# Lookup the reservation of the Original IP
echo $(mysql blazar -e "select status, id from reservations where id ='$ORIGINAL_LEASE_IP_RESERVATION_ID';")

# Update the reservation as active for the IP
mysql blazar -e "update reservations set status = 'active' where id = '$ORIGINAL_LEASE_IP_RESERVATION_ID' limit 1;"

# look up the lease status
echo $(mysql blazar -e "select * from events where lease_id ='$ORIGINAL_LEASE_ID' and status = 'ERROR';" | egrep '[0-9a-f]{8}-([0-9a-f]{4}-){3}[0-9a-f]+' -o)

# update the errored lease to be active
mysql blazar -e "update leases set status = 'ACTIVE' where id = '$ORIGINAL_LEASE_ID' limit 1;"

# look up the lease event status
mysql blazar -e "select * from events where lease_id ='$ORIGINAL_LEASE_ID';"

EVENT_ID=$(mysql blazar -e "select * from events where lease_id ='$ORIGINAL_LEASE_ID' and status = 'IN_PROGRESS';" | egrep '[0-9a-f]{8}-([0-9a-f]{4}-){3}[0-9a-f]+' -o)

# update the event to be done
mysql blazar -e "update events set status='DONE' where id='$EVENT_ID' limit 1;"
```

### Host not in Free pool

OpenStack Error:

* Failed to execute action on\_start for lease
* Conflict while adding the host
* One or more hosts already in availability zone(s)

Python Exception Class:

* blazar.manager.exceptions.AggregateAlreadyHasHost

Steps

1. Go to [https://chi.tacc.chameleoncloud.org/dashboard/admin/aggregates/](https://chi.tacc.chameleoncloud.org/dashboard/admin/aggregates/) and find UUID of the node that fails to start
   1. or query the database with:\
      `mysql -D nova_api -e 'select * from aggregate_hosts where host =”<host_id>"'`
2. Take note of the reservation ID (aggregate id in MySQL query)
3. Run `nova aggregate-list | grep <reservation_id or aggregate_id from mysql query>`  to find the aggregate ID (should be an integer)
4. From the same web page afs 1., copy the list of hosts in the aggregate and paste it in a file (e.g. in \~/hosts)
5.  Run:

    ```
    for i in `cat ~/hosts`; \
    do nova aggregate-remove-host <aggregate_id> $i; \
        nova aggregate-add-host 1 $i;
        done;\
    nova aggregate-delete <aggregate_id>
    ```

## The lease is remaining intact and is not being deleted.

Some states (start\_lease, end\_lease, before\_end\_lease) might be stuck with 'In progress'

Solution: Update the lease event which is stuck to 'Done' and try deleting the lease.&#x20;

Look up the event status' of the Lease. The below command must be executed in the controller node of the site.

{% code overflow="wrap" %}
```shell-session
LEASE_ID=<LEASE ID WHICH CANNOT BE DELETED>
mysql blazar -e "select * from events where lease_id ='$LEASE_ID';"
```
{% endcode %}

Get the event ID and update the status as Done

<pre class="language-shell-session"><code class="lang-shell-session">EVENT_ID=$(mysql blazar -e "select * from events where
    lease_id ='$LEASE_ID' and status = 'IN_PROGRESS';" | egrep
<strong>    '[0-9a-f]{8}-([0-9a-f]{4}-){3}[0-9a-f]+' -o)
</strong>mysql blazar -e "update events set status='DONE' where
    id='$EVENT_ID' limit 1;"
</code></pre>

## Moving node to another lease

Host reallocate will find a new host based on previous lease parameters if one is available. Running \`host-reallocate\` without lease ID will reallocate the node for all leases. This may be useful if a node needs to be added to a maintenance lease.

`openstack reservation host reallocate <hypervisor_hostname> --lease-id <lease_id>` \
\
If you need to keep the current lease intact but want to know what upcoming leases are on the node

`openstack reservation allocation list`&#x20;

Find using the `blazar_host_id` to get the allocations for the host&#x20;

`blazar_host_id` is not the same as \<hypervisor\_hostname>, this ID can be acquired from `openstack reservation host show <hypervisor_hostname>` and the `id` from it will be `blazar_host_id`
