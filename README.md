# Web Identity Federation

## Infrastructure Diagram

- **AWS Infras**
![image](https://drive.google.com/uc?export=view&id=1Bbza8upMxJMIHVctyvWSmDFcX5wpqgXN)


---
## Demo Structures

1. Provision the environment and review tasks
2. Create Google API Project & Client ID
3. Create Cognito Identity Pool
4. Update App Bucket & Test Application
5. Clean up

---
## Provision the environment and review tasks

Click the link to deploy the **Base-infras deployment:** [Stack deployment](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https://learn-cantrill-labs.s3.amazonaws.com/aws-cognito-web-identity-federation/WEBIDF.yaml&stackName=WEBIDF) -> Wait until the stack is in `CREATE_COMPLETE` state

---
## Create Google API Project & Client ID

#### Create Google API Project

Any application that uses OAuth 2.0 to access Google APIs must have authorization credentials that identify the application to Google's OAuth 2.0 server. In this stage we need to create those authorization credentials.

Move to the Google Credentials page [click here](https://console.developers.google.com/apis/credentials) -> Either sign in, or create a google account if you don't have one. -> You will be moved to the `Google API & Services Console` 
> You may have to set your country and agree to some terms and conditions, thats fine go ahead and do that.  

Click the `Project` dropdown right next to the Google Cloud logo, and then click `NEW PROJECT` OR [click here](https://console.cloud.google.com/projectcreate) -> For project name enter `DemoCognitoIDF`  then create the project.

#### Configure Consent Screen

Click `Credentials` -> Click `CONFIGURE CONSENT SCREEN`  
- Choose `External` (Because our application will be usable by any google user, we have to select external users) 
- Then click `CREATE`
- `App Name` = `DemoCognitoIDF`  
- Enter your own email in `user support email` & `Developer contact information`  
- Click `SAVE AND CONTINUE`  
- Click `BACK TO DASHBOARD`

#### Create Google API PROJECT CREDENTIALS

Click `Credentials` on the menu on the left  -> Click `CREATE CREDENTIALS` and then `OAuth client ID`  
- In the `Application type download` select `Web Application`  
- Under Name enter `IDFServerlessApp`
- We need to add the `WebApp URL`, this is the distribution domain name of the cloudfront distribution (making sure it has https:// before it)  
- Click `ADD URI` under `Authorized JavaScript origins`  
- Enter the endpoint URL, you need to enter the `Distribution DNS Name` of your CloudFront distribution (created by the 1-click deployment), you should add https:// at the start, it should look something like this `https://d38sv1tnkmk8i6.cloudfront.net` but you NEED to use your own distributions DNS name **DONT USE THIS ONE**  
- Click `CREATE`

You will be presented with two pieces of information
- `Client ID`
- `Client Secret`

>Note down the `Client ID` you will need it later. You wont need the `Client Secret` again.  

Once noted down safely, click `OK`

---
## Create Cognito Identity Pool

#### Create a Cognito Identity Pool
Move to the Cognito Console, [click here](https://console.aws.amazon.com/cognito/home?region=us-east-1#) -> On the menu on the left, select **Federated Identities**  
- We're going to be creating a new identity pool  
- If this is your first, the creation process will begin immediatly, if you already have any identity pools you'll have to click **federated identities** then click on `Create new identity pool`  
- `Identity pool name` = `DemoIDFIDPool`  
- Expand `Authentication Providers` and click on `Google+`  
- `Google Client ID` = The Google Client ID you noted down in the previous step.  
- Click `Create Pool`

#### Permissions
Expand `View Details`  -> This is going to create two IAM roles (One for `Your authenticated identities` and another for your `Your unauthenticated identities`)  
For now, we're just going to click on `Allow` we can review the roles later. You will be presented with your `Identity Pool ID`, note this down, you will need it later. Click to move back to the dashboard

#### Adjust Permissions
The serverless application is going to read images out of a private bucket created by the initial cloudformation template.  
The bucket is called `patchesprivatebucket`  
Move to the IAM Console, [click here](https://console.aws.amazon.com/iam/home?region=us-east-1#/home)  -> Click **Roles** on the left sidebar 
- Locate and click on `Cognito_DemoIDFIDPoolAuth_Role`  
- Click on `Trust Relationships`  
- See how this is assumable by `cognito-identity.amazonaws.com` with two conditions
	- `StringEquals` `cognito-identity.amazonaws.com:aud` `your congnito ID pool`
	- `ForAnyValue:StringLike` `cognito-identity.amazonaws.com:amr` `authenticated`  
    This means to assume this role - you have to be authenticated by one of the ID providers defined in the cognito ID pool.

When you use WEDIDF with cognito, this role is assumed on your behalf by cognito, and its what generates temporary AWS credentials which are used to access AWS resources.

Click `permissions` .. this defines what these credentials can do.

The cloudformation template created a managed policy which can access the `privatepatches` bucket Click `Add permissions` and then `Attach policies`  
Type `PrivatePatches` in the search box and press `enter`  
Check the box next to `PrivatePatchesPermissions` and click `Attach Policies`

---
## Update App Bucket & Test Application

#### Download index.html and scripts.js from the S3
Move to the S3 Console, [click here](https://s3.console.aws.amazon.com/s3/home?region=us-east-1)  
- Open the `webidf-appbucket-` bucket  
- Select `index.html` and click `Download` & save the file locally  
- Select `scripts.js` and click `Download` & save the file locally

#### Update files with your specific connection information
Open the local copy of `index.html` in a code editor.  
- Locate the `REPLACE_ME_GOOGLE_APP_CLIENT_ID` placeholder  
- Replace this with YOUR CLIENT ID  
- Save `index.html`

Open the local copy of `scripts.js` in a code editor.  
- Locate the IdentityPoolId: `REPLACE_ME_COGNITO_IDENTITY_POOL_ID` placeholder  
- Replace the `REPLACE_ME_COGNITO_IDENTITY_POOL_ID` part with your IDENTITY POOL ID you noted down in the previous step  
- Locate the `Bucket: "REPLACE_ME_NAME_OF_PATCHES_PRIVATE_BUCKET"`  placeholder.  
- Replace `REPLACE_ME_NAME_OF_PATCHES_PRIVATE_BUCKET` with with bucket name of the `webidf-patchesprivatebucket-` bucket 
- Save `scripts.js`

#### Upload files
Back on the S3 console, [click here](https://s3.console.aws.amazon.com/s3/home?region=us-east-1) -> Move inside the `webidf-appbucket-` bucket.  
- Click `Upload`  
- Add the `index.html` and `scripts.js` files and click `Upload`

#### Test application
Open the `WebApp URL` you noted down earlier, the `distribution domain name` of the cloudfront distribution  
This is the web app, with no access to any AWS resources right now  
Open your browser `web developer tools` (firefox tool->browser tools-> web developer tools)  
it might be called browser console in other browsers, it will log any output from javascript running in your web browser.  
With the browser console open, Click `Sign In`  
Sign in with your google account

When you click the Sign In button a few things happen:- (watch the console)

- You authenticate with the Google IDP
- a Google Access token is returned
- This token is provided as part of the API Call to Cognito
- If successful this exchanges this for Temporary AWS credentials
- These are used to list objects in the private bucket
- for all objects, presignedURLs are generated and used to load the images in the browser.

Once signed in you should see 3 cat pictures loaded from a private S3 bucket

Click on each of them, notice the URL which is used? it's a presignedURL generated by the JS running in browser, using the API's which you can access using the cognito credentials.

All of this is done with no self-managed compute.

---
## Clean up

#### Delete the Google API Project & Credentials
[https://console.developers.google.com/cloud-resource-manager](https://console.developers.google.com/cloud-resource-manager) Select `PetIDF` and click `DELETE`  
Type in the ID of the project, which might have a slightly different name (shown above the text box) click `Shut Down`

#### Delete the Cognito ID Pool

Move to the cognito console [https://console.aws.amazon.com/cognito/home?region=us-east-1](https://console.aws.amazon.com/cognito/home?region=us-east-1)  
Click `Federated Identities`  
Click on `PetIDFIDPool`  
Click `Edit Identity Pool`  
Locate and expand `Delete identity pool`  
Click `Delete Identity Pool`  
Click `Delete Pool`

#### Delete the IAM Roles
Move to the IAM Console [https://console.aws.amazon.com/iam/home?region=us-east-1#/home](https://console.aws.amazon.com/iam/home?region=us-east-1#/home)  
Select `Roles`  
Select both `Cognito_PetIDF*` roles  
Click `Delete Role`  
Click `Yes Delete`

#### Delete the CloudFormation Stack

Move to the cloud formation console [https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringText=&filteringStatus=active&viewNested=true&hideStacks=false](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks?filteringText=&filteringStatus=active&viewNested=true&hideStacks=false)  
Select `WEBIDF`, click `Delete` then `Delete Stack`
