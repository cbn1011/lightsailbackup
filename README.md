# lightsailbackup
## About this:
This is an Automated Lightsail Backup lambda function based on **Python 3.8**.

Data loss can be a nightmare for any business or individual, resulting in lost productivity, revenue, and valuable information. That's why it's critical to have a robust backup and recovery strategy in place. With Amazon Lightsail Automated Backup, you can rest easy knowing that your data is secure and accessible, no matter what happens. In this post, we'll dive into the benefits of Lightsail Automated Backup, how it works, and how to set it up. Whether you're a small business owner, developer, or IT professional, read on to learn how to protect your data with this powerful and easy-to-use backup solution.

## Benefits of this approach

- You can use the single lambda function to create and delete lightsail snapshots of multiple instances in different regions.
- There is no builtin automatic backup option for Windows instance in lightsail, we can overcome this by using this approach.
- All the automatic snapshots associated with the lightsail instance will be deleted when you delete the actual resource, here the snapshots created are like manual once, so the snapshots will not be deleted when we delete the actual resource.

## How to configure it?
- First of all we should have to create a lambda function through the AWS console.
- Add necessay permission to the lambda function's role from **Configraion >> Permissions >> Click on the Role name** and provided the Lightsail full access or you can provide only necessay actions as an inline policy.

Then we can write the code.
<pre><code>
import boto3
import uuid
from datetime import datetime, timedelta, timezone

def lambda_handler(event, context):
    # Create a boto3 client for Lightsail
    region = ["us-east-1", "us-west-2"]
    for x in region:
        client = boto3.client('lightsail', region_name=x)
        # Use the get_instances method to retrieve all instances
        try:
            instances = client.get_instances()
        except Exception as e:
            print(f"Failed to retrieve instances: {e}")
            return

        for instance in instances['instances']:
            # Check if the instance has the "backups.enabled" tag set to "true"
            backups_enabled = False
            retention_period = 7
            for tag in instance['tags']:
                if tag['key'] == 'backups.enabled' and tag['value'] == 'true':
                    backups_enabled = True
                if tag['key'] == 'backup.retention':
                    retention_period = int(tag['value'])
                    
            if backups_enabled:
                instance_name = [instance['name']]
                now = datetime.utcnow().strftime("%Y-%m-%d-%H-%M-%S")
                for name in instance_name:
                    if not name:
                        break
                    snapshot_name = name + '-' + str(uuid.uuid4())
                    try:
                        response = client.create_instance_snapshot(instanceName=name, instanceSnapshotName=snapshot_name)
                    except Exception as e:
                        print(f"Failed to create snapshot for instance {name}: {e}")
                        continue
                    print(snapshot_name)
            if backups_enabled:
                instance_name = instance['name']
                now = datetime.utcnow().strftime("%Y-%m-%d-%H-%M-%S")
                snapshots = []
                next_token = None
                while True:
                    if next_token:
                        response = client.get_instance_snapshots(pageToken=next_token)
                    else:
                        response = client.get_instance_snapshots()
                    snapshots += response['instanceSnapshots']
                    if 'nextPageToken' in response:
                        next_token = response['nextPageToken']
                    else:
                        break
                for snapshot in snapshots:
                    if snapshot['fromInstanceName'] == instance_name:
                        # check the date of the snapshot
                        snapshot_date = snapshot['createdAt'].date()
                        # check if the snapshot is older than the retention period
                        if (datetime.now().date() - snapshot_date).days > retention_period:
                            # delete the snapshot
                            client.delete_instance_snapshot(instanceSnapshotName=snapshot['name'])
                            print(f'Snapshot {snapshot["name"]} deleted')
</code></pre>

> Here we are using python boto3 library, by using this library we can interact with many AWS services. It is Python SDK for AWS.

## How this works?

In the above lambda function we are importing 2 libraries boto3 and uuid ( uuid is for providing Universal Unique Identifier for snapshots ). In the beginning of the code you could see we are specifying a variable called **region** .
 <code><pre>
region = ["us-east-1", "us-west-2"]
</code></pre>
Here, region is assigned to a list containing 2 string elements. You can edit this as per your regions where lightsail instances are located. And the **for** loop next to the **region** variable will iterate through each region specified in the variable region.

In the second **for** loop it checks for the tags in the instance. **We should have to add tags in the instances which snapshot we want to take using this lambda function**, the tag should be **backups.enabled** and the value should be **true**.

If we want to delete the snapshots using a retention we should add another tag that is **backup.retention**, by default it is 7 days, we can change or specify the retention through the tag value.

After the **for** loop we could see a **if** condition in that block the code will take the snapshot of the instances where the tag **backups.enabled** is true and append a unique id (uuid) in the snapshot name.

After exiting from that mail **if** condition the code will move on to the next **if** condition, where the snapshots deletion based on the retention period is happening. 

> You may have to tweak the timeout of the lambda function depends on the resources/instances that you have. Also you need to configure a trigger for the lambda function from Eventbridge rule, you can create an event rule to trigger the lambda function every day at a particular time.

Refer this [link](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html) for creating cron expression for triggering the lambda function.
