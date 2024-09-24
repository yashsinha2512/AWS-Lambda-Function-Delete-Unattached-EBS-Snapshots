# AWS Lambda Function: Delete Unattached EBS Snapshots

This AWS Lambda function automates the process of deleting unused Elastic Block Store (EBS) snapshots that are no longer associated with any volume or whose associated volume is not attached to a running EC2 instance. This helps to optimize storage costs by cleaning up snapshots that are no longer needed.

## How It Works

- **Describe Snapshots**: The function retrieves all EBS snapshots owned by the account.
- **Describe Instances**: It retrieves all currently running EC2 instances to determine active volumes.
- **Snapshot Cleanup**:
  - If a snapshot is not attached to any volume, it gets deleted.
  - If a snapshot is attached to a volume that is no longer attached to a running EC2 instance, it also gets deleted.
  - In each case, the function prints details of the deleted snapshot, including the snapshot ID and creation time.

## Prerequisites

Before running this function, ensure that:
- You have the necessary IAM permissions to describe and delete EBS snapshots and EC2 instances. The required permissions are:
  - `ec2:DescribeSnapshots`
  - `ec2:DeleteSnapshot`
  - `ec2:DescribeInstances`
  - `ec2:DescribeVolumes`
  
- [Boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) is the AWS SDK for Python and is used in this Lambda function for interacting with AWS EC2 services.

## Lambda Code

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')

    # Get all EBS snapshots
    response = ec2.describe_snapshots(OwnerIds=['self'])

    # Get all active EC2 instance IDs
    instances_response = ec2.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    active_instance_ids = set()

    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            active_instance_ids.add(instance['InstanceId'])

    # Iterate through each snapshot and delete if it's not attached to any volume or the volume is not attached to a running instance
    for snapshot in response['Snapshots']:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId')
        start_time = snapshot['StartTime']  # Snapshot creation time

        if not volume_id:
            # Delete the snapshot if it's not attached to any volume
            ec2.delete_snapshot(SnapshotId=snapshot_id)
            print(f"Deleted EBS snapshot {snapshot_id} (created on {start_time}) as it was not attached to any volume.")
        else:
            # Check if the volume still exists
            try:
                volume_response = ec2.describe_volumes(VolumeIds=[volume_id])
                if not volume_response['Volumes'][0]['Attachments']:
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} (created on {start_time}) as the volume {volume_id} was not attached to any running instance.")
            except ec2.exceptions.ClientError as e:
                if e.response['Error']['Code'] == 'InvalidVolume.NotFound':
                    # The volume associated with the snapshot is not found (it might have been deleted)
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} (created on {start_time}) as its associated volume {volume_id} was not found.")
```

## Deployment Steps

### 1. Upload the Code to AWS Lambda

1. Log in to your [AWS Management Console](https://aws.amazon.com/console/).
2. Go to the **Lambda** section.
3. Create a new Lambda function, select Python as the runtime, and set up an execution role with the necessary permissions mentioned above.
4. Copy and paste the provided code into the Lambda function editor.
5. Set up a **trigger** (e.g., scheduled using **CloudWatch Events** to run periodically).

### 2. Set Up Permissions

Make sure that your Lambda function's execution role has the necessary permissions to interact with EC2 and manage snapshots. You can attach an inline policy with the following permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSnapshots",
        "ec2:DeleteSnapshot",
        "ec2:DescribeInstances",
        "ec2:DescribeVolumes"
      ],
      "Resource": "*"
    }
  ]
}
```

### 3. Test the Function

You can create a test event in Lambda (no event payload is required) to manually trigger the snapshot deletion process. Check the Lambda execution logs in **CloudWatch** to see the details of the deleted snapshots.

### 4. Automate the Function with CloudWatch Events

Set up a CloudWatch Event to trigger this Lambda function on a schedule (e.g., daily or weekly) to automatically clean up unattached EBS snapshots.

## Logs

The function prints detailed information about each deleted snapshot, including:
- Snapshot ID
- Snapshot creation time (StartTime)
- Volume ID (if applicable)

Example log output:
```
Deleted EBS snapshot snap-0123456789abcdef0 (created on 2023-09-23T12:34:56Z) as it was not attached to any volume.
Deleted EBS snapshot snap-0987654321fedcba0 (created on 2023-07-19T08:12:34Z) as the volume vol-1234567890abcdef0 was not attached to any running instance.
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Contributing

If you would like to contribute to this project, feel free to fork the repository and submit a pull request.
