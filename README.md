# AWS 'Projects'

Unlike [Google Cloud](https://cloud.google.com/storage/docs/projects), AWS does not have the concept of a "project".  For individuals or small companies this may not be a big deal, but as the number of distinct projects grows within a single AWS account, it gets increasingly difficult to keep track of all of the Resources (e.g. EC2 instances, S3 buckets) that have been created, what project they belong to, and who has responsibility for their maintenance and cost.  It also becomes difficult to ensure that developers don't accidentally mess up other projects that are in the same account.  

One strategy to address this is to use multiple AWS accounts and/or multiple AWS VPCs or subnets to segment project work.  Multiple AWS accounts can even be linked together for billing purposes using AWS Organizations.  There are downsides to this approach.  Creating an entirely new AWS account, VPC, or subnet for each project, regardless of how small, can lead to management and cost overhead, particularly if you employ a set of non-zero-cost boilerplate Resources for each AWS account (e.g. AWS Config Rules).  

Another service to consider is AWS Resource Groups, which do let you group Resources by a tag value.  If you create a Project tag and an associated Resource Group for each Project tag, you can get some insight into what Resources belong to what Projects.  However, there is no detailed per-Resource cost breakdown associated with Resource Groups, and Resource Groups can only track Resources within a single region within a single AWS account, so you can't get a single view into all Resources that belong to a project unless your project happens to reside in a single region in a single account.

In this post, we'll look at how to deploy a solution that aims to create a framework for managing projects on AWS.  This solution includes:

- Setting up a project request process and database/spreadsheet to keep track of our projects.
- Creating budgets for each project using AWS Budgets.
- Enforcing a project tagging scheme using AWS Config.
- Enabling the AWS Cost and Usage Report along with AWS Athena and AWS QuickSight to provide per-resource cost reports for project owners.

Together, these pieces force all AWS Resources to be associated with an approved project and give each project owner feedback about what Resources they own and how much they cost.  **This creates a feedback loop that drives down cost and confusion.**

## Create a Project Approval Process and Database

This step is not presciptive; you can accomplish this however you'd like, but the important part is that you end up having a database or spreadsheet that contains an updated list of all of the approved projects on your AWS account(s).  For the approval process, you might consider setting up a ServiceNow-or-equivalent ticket type.  Once the approval process is complete, the end result should be a new entry in your database or spreadsheet of choice.  I recommend collecting and storing at least the following pieces of information as part of this process:

- Project name (something simple that conforms to the limits of what you can use as the AWS tag value and Budget name as shown in subsequent sections)
- Project description
- Project owner
- Project monthly budget

## Create an AWS Budget for each Project

For every project you have in your project database, you'll want to create an AWS Budget.  If you have a single AWS account, create the AWS Budget in that account.  If you use AWS Organizations to manage multiple AWS accounts, it is recommended to put all Budgets into the parent account of AWS Organizations, because this account has access to the billing information across all of its child accounts.  There are two main reasons we want to set up an AWS Budget for each project:

- It's good practice to create budgets, and in general setting up alerts for budget thresholds helps prevent runaway and/or unexpected cost.
- AWS Budgets is one of the few services in AWS that can see across both AWS accounts and regions, giving us a central place to store a list of our AWS projects on AWS.  As you'll see when we get to the AWS Config step, this is important because we want to be able to check that all AWS Resources contain a Project tag with a value equal to the name of an AWS Budget.

For each AWS Budget that you create for each AWS project, use the following settings:

- For Budget type, select `Cost budget`
- For Budget name, ensure that this value is exactly equal to the `name` of the project
- For Budgeted amount, set this equal to the value given by the new project requestor
- For Budget Filters, choose 'Tag' and then search for 'Project'.  Set the value of the 'Project' equal to the name of your project.  If you don't see 'Project' when you search for Tag, you must first add 'Project' to your [user-defined cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/custom-tags.html).  Changes to cost allocation tags can take roughly 24 hours to implement and see as available in AWS Budgets.
- For all other values, you can use defaults or edit them as you see fit.

## Create an AWS Config Rule to Ensure Resources Have Valid Project Tags

![AWS Config Rule Checks that Project Tag Exists and Matches Budget Name](/aws-config-project-tag-matches-budget.png)

AWS Config can be used to monitor and/or enforce rules within your AWS account.  For this solution, we'll create an AWS Config Rule that checks that all taggable AWS Resources have a Project tag with a value equal to the name of an AWS Budget.  Because each of your approved projects now has an associated AWS Budget, and the name of the Budget is equal to the name of the project, this Config Rule will help track down any Resources that don't have a Project tag and make sure that all Resources are associated with a project.  

Lucky for you, the custom AWS Config Rule required to perform this check is codified as a CloudFormation template in this project.  You will need to deploy this AWS Config CloudFormation template in every region within every account that you want to enforce.  Thankfully, if you do use AWS Organizations, there is a way to [centrally manage your AWS Config Rule deployments across accounts and regions](https://docs.aws.amazon.com/config/latest/developerguide/config-rule-multi-account-deployment.html).  Also note that unlike many other AWS services, AWS Config requires some initial service setup before you can use it.  If you haven't already set up and enabled Config in your AWS account and the region(s) you are using, you'll need to go through the [initial setup instructions](https://docs.aws.amazon.com/config/latest/developerguide/gs-console.html).

Note that the custom AWS Config Rule is not currently set up to perform any automated remediation of non-compliant Resources, so it will still be up to you to enforce tagging of Resources, but at least with the AWS Config Rule you know which Resources are non-compliant, and if you are so inclined you can look into extending the Rule to add automated remediation.

Also note that you'll likely have some boilerplate AWS Resources that were created for you when you created your account that will not have tags and that don't belong to any particular project, e.g. default VPC and subnet, default security group, etc.  You can either choose to ignore these or go ahead and create a project/budget for these default Resources to ensure 100% compliance coverage for our Config Rule.

## Project Resource Cost Insights

Within a crowded AWS account, particularly one crowded with relative AWS newcomers, it's easy to lose track of Resources and their associated cost.  Now that we've set up a solution to define projects, give them budgets, and make sure they are tagged, let's give project owners a report that gives them better insight into Resources associated with their projects and their cost.

One simple solution to tracking cost by Project tag is to use the AWS Cost Explorer along with [cost allocation tags](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html). This can be useful, but the primary downside is that this can only show you a breakdown of cost per project per Resource *type* (e.g. S3, EC2, etc.), not per Resource.  A per-Resource cost report is more immediately understandable and actionable than a per-Resource-type report.  To get per-Resource cost data, you must instead create an AWS [Cost and Usage Report](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/billing-reports-costusage.html).  

At the time of this writing you cannot create a Cost and Usage Report using CloudFormation, so we'll do this step manually via the AWS Management Console.  

- Navigate to the `Billing` section.
- Find the `Cost and Usage Reports` section from the left panel.
- Click the `Create Report` button.
- Give your report a name.
- Check the `Include resurce IDs` option.
- Leave the default settings and go to the next page.
- Configure an S3 bucket by following the instructions.
- Set your `Report path prefix` to `athena` (or whatever you'd like, really).
- Change your `Time granularity` to `Daily`.
- Set `Enable report data integration for`  to `Amazon Athena`.
- Review and Complete to finish creating your Cost and Usage Report.

Cost and Usage Reports can take half a day or more before you get your first report. 

### Prepare Cost and Usage Data for Athena

Because we chose to set up our Cost and Usage Report to output data formatted for Amazon Athena, the next step is to employ the solution detailed by AWS [here](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/setting-up-athena.html) by grabbing and deploying the CloudFormation template that is placed in your Cost and Usage Report S3 bucket after the first Cost and Usage Report is finished running.  Follow the instructions in the link carefully and once Glue has a chance to crawl the data for the first time, you should see your Tables in Athena and be ready for the next step.

#### A note on CUR with AWS Organizations

In general, when using AWS Organizations, you want to keep as little as possible in the master AWS account.  At the same time, when using Organizations, you can *only* create CUR reports in the master account.  For this reason, you may want to consider using S3 replication to create the CUR report inside the master account but replicate its contents to another AWS account to perform Athena queries, etc.  If you decide to do so, make sure to scrub through the CloudFormation template from the previous step and replace the S3 bucket name with the replicated bucket name before applying the template.  If you're looking for an easy way to set up cross-account S3 replication, check out [this](https://github.com/ethanfahy/aws-s3-replication) project on GitHub which provides CloudFormation templates to create buckets in two separate accounts that replication from one to the other.

### QuickSight Dashboard Setup

Now that our Cost and Usage Report data is available for query using AWS Athena we have numerous paths forward.  We could make ad hoc queries against Athena, create automated report emails by calling Athena from Lambda and sending the results via SES, or we could use QuickSight to create dashboards for end users to explore their cost and usage data.  For now we'll focus on creating a simple QuickSight dashboard that shows a per-resource cost breakdown for a user-defined Project, Year, and Month.

![QuickSight Dashboard for per-Resource Cost breakdown](/quicksight-cur-dashboard.png)

- From the Management Console, navigate to QuickSight.
- If you haven't used QuickSight before you will be asked if you want to enable QuickSight for your AWS account.  Choose yes, and then choose either the Standard or Enterprise option.
- On the next screen, select your AWS region (I advise using the same region that you have chosen for the rest of this guide).
- Give your QuickSight account a name and email address.
- Make sure that the `Amazon Athena` and `Enable autodiscovery of data and users in your Amazon Redshift, Amazon RDS, and AWS IAM services` options are checked since we'll need them later.  Also add permissions for `S3` and select the bucket(s) that have CUR data in them (and Athena query results).
- Once you've set up your QuickSight account, log in to it and click the `Manage data` button and then the `New data set` button.
- Choose `Athena` and for the `Data source name` pick any name you'd like.  Once you validate the connection click the `Create data source` button.
- On the next screen, select your Athena database and table that has your Cost and Usage Report content in it that was created automatically as part of the CloudFormation template above.
- The last step will ask if you want to import the data to SPICE or directly query your data.  If you choose SPICE it will effectively cache your data and your dashboard and queries will be faster, but if your CUR data is too large it won't fit and it also may incur additional expense depending on your usage patterns. The solution will work either way and you can change this setting later if you'd like.  If you do use Spice, you'll want to set up a scheduled refresh so that you pull in new cost and usage data periodically.  You can do this by selecting you data set and choosing the `Schedule refresh` button and setting up a refresh schedule based on whether you chose hourly or daily updates in your Cost and Usage Report setup.
- Once you create your dataset, go back to the main page for QuickSight and navigate to the `New analysis` button.
- Choose your newly created cost and usage data set and hit the `Create analysis` button.
- In the `Visualize` section, select `Table` from the `Visual types` section. 
- In the `Visualize` section, find and select `line_item_blended_cost`, which should populate the `Value` section in your `Field wells`.
- In the `Visualize` section, find and select `line_item_resource_id`, which should populate in the `Group by` section in your `Field wells`.
- In the `Parameters` section, create a new parameter. `Name`: `Project`.  `Data type`: `String`. `Values`: `Single value`. `Static default value`: leave this empty.  Select the `Create` button.  On the next screen, select `Create a new control for a filter or a calculated field.`  `Display name`: `Project`.  `Style`: `Single select drop down`.  `Values`: `Link to a data set field`. `Select a data set`: The name of your CUR dataset that you created in a previous step.  `Select a column`: `resource_tags_user_project`.
- In the `Filter` section, create a new filter and select `resource_tags_user_project`.  Click on your newly created filter to expand the options.  Set `Filter type`: `Custom filter`. Check `Use parameters` and click `Yes` when prompted. Set `Select a parameter` dropdown to `Project`.  Click the `Apply` button then the `Close` button.  We now have a Control that can filter the rows in our table by Project (via the Project tag values).
- Repeat the previous steps for creating a parameter and filter, but this time do this for the `year` item.
- Repeat the previous steps for creating a parameter and filter, but this time do this for the `month` item.
- We now have a control that can help filter results by year and month.
- Rename your Sheet from `Sheet 1` to `Project Resource Cost` or something to that effect by double clicking its name and typing a new value.
- In the upper right corner select the `Share` option and then `Publish dashboard` and publish our analysis as a dashboard that others can use.

At this point we now have a nice example QuickSight dashboard that project owners can use to investigate per-source cost data for their projects.  With some experimentation there are many more useful dashboards you can create.  If using AWS Organizations, you may also consider configuring QuickSight access using a single sign-on solution along with [row-level security](https://docs.aws.amazon.com/quicksight/latest/user/restrict-access-to-a-data-set-using-row-level-security.html) to gate access to the cost and usage report data to particular users and groups based on AWS account numbers and/or other criteria.

## Wrapping Up

We've seen how you can create a framework to support Projects on AWS.  There's more to this concept to explore, like how you might go about restricting access to Project Resources based on Project tags combined with per-Project IAM permissions (see [here](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_tags.html) for more details).  Hope this helps, feedback is always welcome!