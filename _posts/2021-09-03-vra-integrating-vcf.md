---
layout: post
title: vRA 8 - Integrating VMware Cloud Foundation
comments: false
category: blog
---
# vRA 8 – Integrating VMware Cloud Foundation

In this post, today, we’ll talk about how to configure a VMware Cloud Foundation in our vRA 8 environemnt.

As a first step we need to create an SDDC integration:

1. Log into your vRA env with your account (Cloud Assembly administrator as role)
2. Switch to “Infrastructure Tab”
3. Click on “Integrations”
4. Click on “+ Add Integration”

![vRA](/images/20210903/01.png)

As integrations type select “SDDC Manager” and compile the following form using your information:

![vRA](/images/20210903/02.png)

Click on “Validate”, accept the certificate if required and click on “Add” button.

At this point we’re ready to link VMware Cloud Foundation.

In integration page, open the integration created previously:

![vRA](/images/20210903/03.png)

Switch to “Workload Domains” tab and select your WD (in my case is: VI-Workload-01), and then click to “Add account”.

1.	Check the option “use cloud foundation managed service credentials”
2.	Click on “Create and validate service credentials”

After that, the current page will show you all the datacenter that can your provision your workload:

![vRA](/images/20210903/04.png)

In my case, I’ll select “VI-Workload-DC” and “Create a cloud zone for the selected datacenter”. The last option allow me to save time and create automatically a cloud zone that I will use to create my project and my future deployments.

![vRA](/images/20210903/05.png)

The next step will be create a project that will use a cloud zone backed by VMware Cloud Foundation (VCF) account.
Using the left menu, go to “Projects” page:

![vRA](/images/20210903/06.png)

1.	Click to “+ New Project”
2.	Choose a name for your project (in my case will be “VMW-VCF”)

![vRA](/images/20210903/07.png)

Switch to “Users” page and insert the users/groups that will work into this project and then go to “Provisioning” page where add to this project the cloud zone backed by our VCF account:

![vRA](/images/20210903/08.png)

![vRA](/images/20210903/09.png)

As you can see above, you can define some limitations about this cloud zone for this particular project. Click Add and click create.

Now our project is ready, and we can proceed to create a “Flavor mapping” and “Image Mapping” in order to create a cloud template.

Always using the left menu, go to “Flavor Mapping”:

![vRA](/images/20210903/10.png)

Click on “ + NEW FLAVOR MAPPING” and compile the form as follow:

![vRA](/images/20210903/11.png)

Click create to submit the operation. Once your flavor is ready, using the left menu, go to “Image Mapping” and click on “+ NEW IMAGE MAPPING”:

![vRA](/images/20210903/12.png)

And then click “CREATE”.  Now our configuration is ready! The next step will be create a simple cloud template:
   1.	Go to “Design” tab (still remaining in cloud assembly)
   2.	Click +NEW FROM
   3.	Select Blank canvas
   4.	Enter a name like you prefer
   5.	Ensure you’re selecting the project created before (in my case: VMW-VCF)
   6.	Click Create
   7.	Create a simple cloud template as follow (in my case I will use cloud agnostic components):

![vRA](/images/20210903/13.png)

Click on the “TEST” button if you want to check your cloud template and then click to “DEPLOY”.

![vRA](/images/20210903/14.png)

Well done!
The deployment to VCF Workload domain is successful.
