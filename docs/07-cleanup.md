# Module 7 - Cleanup

If you conducted this workshop with an account provided by an AWS Event, then cleanup will be handled for you by the presentor.  However, if you used your own account, then follow these steps to cleanup your account.

1. Cloud Custodian rules and lambda
2. Delete Module 1 CloudFormation stacks (**ThreatDetectionWksp-Env-Setup** and **ThreatDetectionWksp-Attacks**).
	* Go to the <a href="https://us-west-2.console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks?filter=active" target="_blank">AWS CloudFormation</a> console.
	* Select the appropriate stack.
	* Select **Action**.
	* Click **Delete Stack**.
	* Repeat the steps above for each stack.
3. loggroups
4. Security hub
5. GuardDuty
6. Delete git repo clone
