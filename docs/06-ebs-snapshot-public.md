# Module 6 - Remediate an Public EBS-Snapshot

This module will show how to setup an automated detection of a EBS Snaspshot that has been made public, with a finding submitted to Security Hub, then use Security Hub Custom action to delete the snapshot.  Then we'll fully automate the remediation by changing the detection policy to perform the delete while still providing notification.

1. Start by reviewing the file "post-ebs-snapshot-public.yml" using the Cloud9 IDE. Observe that the only action type is "post-finding". Also Observe that the following two lines instruct Cloud Custodian to deploy a CloudWatch rule which triggers on a regular schedule, configured to be every 5 minutes.

        type: periodic
        schedule: "rate(5 minutes)"

2. Then deploy the automated detection policy by running the following command:

        ${CLOUDCUSTODIANDOCKERCMD} securityhub-remediations/module6/post-ebs-snapshot-public.yml

3. Next we need to create a new EBS volume, however no data will be stored in it, it will not be attached anywhere.  Note how the VolumeId is saved for the next step.

        export WorkshopVolumeId=$(aws ec2 create-volume --availability-zone $(aws ec2 describe-availability-zones --query AvailabilityZones[0].ZoneName --output text) --size 1 --query VolumeId --output text)


4. Then we Snapshot the volume just created, also saving the SnapshotId for a future step.

        export WorkshopSnapshotId=$(aws ec2 create-snapshot --volume-id $WorkshopVolumeId --query SnapshotId --output text)

5. This step makes the snapshot public.

        aws ec2 modify-snapshot-attribute --snapshot-id $WorkshopSnapshotId --attribute createVolumePermission --operation-type add --group-names all


6. Now navigate to the Findings part of Security Hub's console.  Look for a finding who's title is the same as the policy name from step 1 then click it's checkbox.  If you don't see it, you may need to wait up to 5 minutes (remember the schedule based CloudWatch Rule from Step 1) for it to appear after refreshing the browser.
7. Click the dropdown for Actions then select "Ebs-Snapshot Delete" (This custom action is one of the ones deployed in Module 2).
8. Confirm the snapshot got deleted by running:

        aws ec2 describe-snapshots --snapshot-ids $WorkshopSnapshotId

    then confirming the response is similar to:

        An error occurred (InvalidSnapshot.NotFound) when calling the DescribeSnapshots operation: The snapshot 'snap-0643b6dcd0a6f01f0' does not exist.


9. Now edit the file "post-ebs-snapshot-public.yml" by adding an action to delete the snapshot at time of initial detection, which transform the policy from a detective control to a remediation control.  Append the following line to the file, where the dash should be in column 5.

        - type: delete

10. Save the file in the IDE.
11. Run the following command which redeploys the policy:

        ${CLOUDCUSTODIANDOCKERCMD} securityhub-remediations/module6/post-ebs-snapshot-public.yml

12. Run the following three commands:

        export WorkshopSnapshotId=$(aws ec2 create-snapshot --volume-id $WorkshopVolumeId --query SnapshotId --output text)
        aws ec2 modify-snapshot-attribute --snapshot-id $WorkshopSnapshotId --attribute createVolumePermission --operation-type add --group-names all
        aws ec2 describe-snapshots --snapshot-ids $WorkshopSnapshotId

13. Repeat running the last command, the describe-snapshots one, until either the InvalidSnapshot.NotFound error is received (which means success, move on the the next step)
    or if after 5 minutes it still has not been deleted, the most likely cause is Step 9 did not get done correctly,
    review the CloudWatch logs for the policy, or the file did not get saved (view the file from the terminal to see if it has the type: delete line), or the policy did not get redeployed in step 11.
14. The remainder of this module shows you how to customize the automated remediation by adding an exception filter then provides information on more advanced filters.
    If you have a use case for publicly sharing a snapshot, a change to the policy could be made to filter for only those snapshots which do not have a specific value for a specific tag.
    In this example we use the Tag key of "PublicIntent" and Tag value of "True".  Start by updating the filter section of the policy by inserting the following lines at line 13 or 16,
    as long as it's within the filter section and doesn't overlap the existing filter, with the dash character at column 13.
    The way to read the filter match the condition of not-equal when testing for Tag key of value.
    Thus resources which have this tag value will get filtered out aka excluded, thus the actions will not get invoked on those filtered out resources.

        - type: value
          op: not-equal
          key: "tag:PublicIntent"
          value: "True"

15. Save the file.
16. Deploy the updated policy

        ${CLOUDCUSTODIANDOCKERCMD} securityhub-remediations/module6/post-ebs-snapshot-public.yml

17. Run the following command noticing that this time it's created with a Tag key and value which matches the filter specified.

        export WorkshopSnapshotId=$(aws ec2 create-snapshot --volume-id $WorkshopVolumeId --query SnapshotId --output text --tag-specifications 'ResourceType=snapshot,Tags=[{Key=PublicIntent,Value=True}]')

18. Run the following to make the snapshot public:

        aws ec2 modify-snapshot-attribute --snapshot-id $WorkshopSnapshotId --attribute createVolumePermission --operation-type add --group-names all

19. Run the following which waits for 5 minutes to then show the snapshot did not get deleted, a result of the exception filter.

        sleep 300 ; aws ec2 describe-snapshots --snapshot-ids $WorkshopSnapshotId

20. For good hygene, delete the public snapshot manually.

        aws ec2 delete-snapshots --snapshot-ids $WorkshopSnapshotId

21. To learn more about the types of filters that can be added to any Cloud Custodian Policy, click [generic filters](https://cloudcustodian.io/docs/filters.html).
22. To learn about the filters that can be applied to EBS Snapshots, run the following:

        docker run -it --rm ${SECHUBWORKSHOP_CONTAINER} schema ebs-snapshot.filters

23. To learn more about the attributes of a specific filter, append the filters name to the command, like the following example for the skip-ami-snapshots filter

        docker run -it --rm ${SECHUBWORKSHOP_CONTAINER} schema ebs-snapshot.filters.skip-ami-snapshots

26. To learn about what AWS resource types are supported by Cloud Custodian, run the following:

        docker run -it --rm ${SECHUBWORKSHOP_CONTAINER} schema aws

27. The actions supported by a specific resource can be viewed by using the schema command with the parameter of <resource_type>.actions, like in the following:

        docker run -it --rm ${SECHUBWORKSHOP_CONTAINER} schema ebs-snapshot.actions

28. And just like filters, the attributes for a given action can be viewed by running the schema command specifing the <resource_type>.actions.<action_name>, like the following:

        docker run -it --rm ${SECHUBWORKSHOP_CONTAINER} schema ebs-snapshot.actions.post-finding
