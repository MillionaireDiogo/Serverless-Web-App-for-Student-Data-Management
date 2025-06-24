# Serverless Web App for Student Data Management

This project delivers a fully serverless, scalable, and cost-efficient web application for managing student data. It leverages AWS managed services to reduce operational overhead, improve performance, and provide a seamless user experience.

## Workflow
![architecture](https://github.com/user-attachments/assets/3cc14124-fd3c-4adb-a5f5-082d9ee2b1e2)

### 1. Frontend Hosting (Static Web Application)
- **Service Used:** Amazon S3 (Simple Storage Service) to host static files and act as the entry point for the web application.
- Public access is configured for static web hosting.

### 2. Global Content Delivery
- **Service Used:** Amazon CloudFront (CDN)
- **What it Does:**
  - Distributes static content (from S3) to global users with low latency.
  - Adds security and caching.
  - Can integrate with WAF (optional) for protection.

### 3. Client-Side API Requests
- **Service Used:** Amazon API Gateway acts as the front door for the application’s backend APIs.
- Handles RESTful requests (GET/POST).
- Validates and routes the request to the appropriate Lambda function.

### 4. Backend Logic / Compute
- **Service Used:** AWS Lambda
- Contains the application logic.
- Processes requests like:
  - `GET`: Fetch student data.
  - `POST`: Add/update student records.
- Stateless and runs only when triggered, reducing costs.

### 5. Database
- **Service Used:** Amazon DynamoDB to store structured student data in a NoSQL format.

## Visual Summary (Workflow Steps)
## Visual Summary (Workflow Steps)

User visits app  
↓  
CloudFront  
↓  
S3 (static files served)  

User submits form (e.g., Save Student Data)  
↓  
JS frontend  
↓  
API Gateway  
↓  
Triggers Lambda  
↓  
Processes logic  

Lambda reads/writes data to/from DynamoDB  
↓  
Response returned to frontend via API Gateway

# Prerequisites for the Serverless Student Data Management Project

1. **AWS Account**  
   You need an active AWS account with appropriate permissions to create and manage resources.

2. **IAM Roles and Permissions**  
   Create IAM roles with least privilege policies that allow services to interact securely:
   - **Lambda Execution Role:** Permissions to read/write DynamoDB, log to CloudWatch.  
   - **API Gateway Role:** To invoke Lambda functions.  
   - **S3 Bucket Policy:** To allow CloudFront and users to read static content.  
   - Additional roles for CloudFront if using custom origins or Lambda@Edge.

3. **Lambda Functions**  
   Two Lambda functions:  
   - **GET Lambda:** Handles GET requests, fetches student data from DynamoDB.  
   - **POST Lambda:** Handles POST requests, adds or updates student data in DynamoDB.  
   These functions should be written, packaged, and ready to deploy.

4. **DynamoDB Tables**  
   Create a DynamoDB table to store student data.  
   Design the schema with appropriate partition key (e.g., `studentId`) and attributes for student details.

5. **S3 Bucket**  
   Set up an S3 bucket configured for static website hosting.  
   Upload the web app’s frontend code (HTML, CSS, JS) to this bucket.  
   Configure bucket policies to allow public read access or restrict access via CloudFront origin access identity.

6. **API Gateway**  
   Create an API Gateway REST API or HTTP API.  
   Configure endpoints that integrate with the Lambda functions:  
   - GET endpoint to invoke GET Lambda.  
   - POST endpoint to invoke POST Lambda.
     
7. **CloudFront Distribution**  
   Set up a CloudFront distribution with:  
   - Origin pointing to the S3 bucket (for frontend assets).  
   - Custom behaviors or origins to route API requests via API Gateway.  
   - Optional: Attach AWS WAF for security rules.  
   - Configure HTTPS and custom domain (optional).

# PART 1: Setup IAM Role, DynamoDB Table and Lambda

## Step 1: Create an IAM Role

1. Go to the **IAM** service in the AWS Management Console.  
2. In the sidebar, click on **Roles**, then choose **Create role**.  
3. Under **Trusted entity type**, select **AWS service**.  
4. For the use case, choose **Lambda**, then click **Next**.  
5. In the permissions step, attach the following policies:  
   - **AWSLambdaBasicExecutionRole:** Grants Lambda permission to write logs to Amazon CloudWatch, essential for debugging and monitoring.  
   - **AmazonDynamoDBFullAccess:** Allows full access to DynamoDB, enabling Lambda functions to perform GET, POST, and other operations on the tables.  
   - **AmazonS3FullAccess:** Grants access to S3, which is used to host and serve static content for the serverless web app.  
6. Click **Next**, then:  
   - Provide a role name, e.g., `RoleForLambda`.  
   - Add a short description such as:  
     > IAM role for Lambda function with access to CloudWatch, DynamoDB, and S3 for a serverless web application.
![1](https://github.com/user-attachments/assets/cc624460-6778-4c2f-8ae6-dfa7eca2a98b)
7. Click **Create role** to finish.

---

✅ **Step 2: Create DynamoDB Table**
### 🔹 Goal:
Create a DynamoDB table that will store student data such as ID, name, grade, etc.

### 🔧 Instructions:
1. Log in to the AWS Console.  
2. Navigate to:  
   `Services → DynamoDB → Tables → Create Table`  
3. In the **Create Table** form:  
   - **Table Name:**  
     Example: `ServerlessTable`  
   - **Partition Key (Primary Key):**  
     - **Name:** `studentId`  
     - **Type:** String
![2](https://github.com/user-attachments/assets/95ea6726-815e-49c1-9ba2-bc79ae04a45e)
### ❓ What is the Partition Key?
The Partition Key uniquely identifies each item (record) in the table.  
In this project, `studentId` acts as the unique identifier for each student.

DynamoDB uses the partition key to:  
- Determine data distribution across storage nodes.  
- Ensure fast and scalable lookups.  
- Prevent data collisions, as no two items can have the same partition key.

Think of it like the primary key in a relational database — it ensures each student record is uniquely accessible.

### ⚙️ Leave All Other Settings as Default:
- **Sort Key:** Leave empty (unless you want to allow multiple entries per `studentId`, which we don’t in this case).  
- **Provisioned capacity or On-demand:** Default is fine (On-demand).  
- **Encryption:** Leave default unless the org requires KMS.  
- **TTL (Time to Live):** Optional — not needed for this use case.  
- **Stream:** Disabled — not needed unless using DynamoDB Streams.  
- **Tags:** Optional — add if needed for cost tracking.
![3](https://github.com/user-attachments/assets/f20dfb6f-3619-4ec1-a7de-d10f7281dd47)
✅ **Final Step:**  
Click **Create Table**.

## Step 3: Create Lambda Function For GET Requests
This function will handle all GET requests made to the DynamoDB table.
1. Navigate to **AWS Lambda** in the AWS Management Console.  
2. Click on **Create function**.  
3. Select **Author from scratch** and provide a name for the function, e.g., `LambdaFunctionForServerless`.  
4. For the runtime, choose **Python 3.13** (latest at the time of writing).
5. Under **Change default execution role**, select **Use an existing role** and choose the IAM role `RoleForLambda` created in Step 1.  
6. Click **Create function** to complete the setup.

## Step 4: Configure Lambda Function For GET Requests
### 🔹 1. Paste the Lambda Code into the IDE
Replace the default sample code with the following function.  
Fetch the code here: [https://github.com/ansarshaik965/AWS-SERVERLESS-DEPLOYMENT)

```python
import json
import boto3

def lambda_handler(event, context):
    dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
    table = dynamodb.Table('ServerlessTable')
    response = table.scan()
    data = response['Items']
    while 'LastEvaluatedKey' in response:
        response = table.scan(ExclusiveStartKey=response['LastEvaluatedKey'])
        data.extend(response['Items'])
    return data
```
### 📝 What this code does:
- Connects to DynamoDB in **us-east-1** (make sure to match the region).  
- Targets the table named **ServerlessTable**.  
- Scans the entire table to retrieve all items.  
- Returns the list of records.  

> 🔸 **Note:** This is a read-only operation. `scan()` is useful for testing but not efficient for large tables.

---

### 🔹 2. Click on **Deploy** to save the code and configuration into the Lambda function.
- Deploying creates a new version of the Lambda and makes the function ready for invocation (by API Gateway, test events, etc.).  
- **Deploy** is not the same as executing the function; it just saves and activates the changes.

---

### 🔹 3. Click **Test** → Create a New Test Event
Testing the Lambda verifies that the code works before connecting it to API Gateway or frontend. The Lambda console test simulates an API call or event trigger without external services.

**⚙️ Steps:**  
- Click **Test**  
- Create a new test event → name it e.g., `LambdaEventTest`  
- Leave the default JSON (it’s not used by the code in this case)

---

### 🔹 4. Run the Test
✅ **What happens behind the scenes?**  
- Lambda invokes the function.  
- The function:  
  - Connects to DynamoDB.  
  - Scans the **ServerlessTable**.  
  - Returns the data.  
- The event returns with success.

---
![5](https://github.com/user-attachments/assets/24f14641-d926-428a-ad5c-dd8384298834)
### Important:
- Ensure the region in the code matches the region where the DynamoDB table is created.  
- Ensure the table name in the code matches the DynamoDB table name.  
- Edit the Lambda code if necessary to reflect the actual configuration.

## Step 5: Create Second Lambda Function for POST Requests

### 🧭 Steps in AWS Console:

1. Navigate to **Lambda**.  
2. Click **Create function**.  
3. Fill in the form:  
   - **Function name:** `LambdaPOSTFunction`  
   - **Runtime:** Python 3.13  
   - **Architecture:** Leave as default (`x86_64`)  
4. Under **Permissions → Execution role:**  
   - Select **Use an existing role**  
   - Choose **RoleForLambda** (the role created earlier, with permissions for DynamoDB and CloudWatch)
![6](https://github.com/user-attachments/assets/88be0471-2caa-436d-8a86-7265253a2ee7)
5. Click **Create function**.

## Step 6: Configure Lambda Function for POST Requests

Once the function is created:

### 🔹 Replace the default code with this:

```python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('ServerlessTable')

def lambda_handler(event, context):
    try:
        # Parse JSON body from API Gateway
        body = json.loads(event['body'])
        studentId = body['studentId']
        name = body['name']
        student_class = body['class']
        age = body['age']

        response = table.put_item(
            Item={
                'studentId': studentId,
                'name': name,
                'class': student_class,
                'age': age
            }
        )
        return {
            'statusCode': 200,
            'body': json.dumps('Student data saved successfully!')
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps(f'Error: {str(e)}')
        }
```
![7](https://github.com/user-attachments/assets/5e22f017-09c1-4a02-b609-0e93b47281cb)
This Lambda function handles HTTP POST requests (usually from API Gateway) to add a new student record into a DynamoDB table named `ServerlessTable`.
- Connects to DynamoDB and selects the `ServerlessTable` table.  
- Parses the incoming POST request body (a JSON string) from `event['body']`.  
- Extracts fields: `studentId`, `name`, `class`, and `age`.  
- Inserts the student record into DynamoDB using `put_item()`.  
- Returns a `200 OK` response on success, or a `500` error with details if something fails.

---

### Deploy and Test the Lambda Function
1. Click **Deploy** to save the updated Lambda function code.  
2. To test the function, click **Test** in the Lambda console.  
3. Create a new test event:  
   - Give it a name, e.g., `LambdaEventTest2`.  
   - In the event editor, copy the key-value pairs used in the `Item` section of the Lambda code.  
   - Paste and modify the JSON as needed. For example:

```json
{
  "body": "{\"studentId\": \"12345\", \"name\": \"Millionaire\", \"class\": \"Security101\", \"age\": \"22\"}"
}
```
![8](https://github.com/user-attachments/assets/aec511e1-da55-4e9c-a27d-9554bfdc5e6e)
Save the event configuration, then click **Test**.
Verify the success response or troubleshoot errors as needed.
![9](https://github.com/user-attachments/assets/0bf3c7f2-ef57-4c58-921b-67ff431bd43b)

### What Happens Behind the Scenes
- The test event is passed as input to the Lambda function.  
- Lambda parses the JSON body, extracts the student data, and uses `boto3` to write it to the DynamoDB table (`ServerlessTable`).  
- If successful, you'll get a **200** response confirming the record was added.  
- If something fails (e.g., missing field or table permission issue), Lambda will return a **500** error with a message.

## Step 7: Verify POST Configuration in DynamoDB Table
### 🎯 Objective:
To confirm that the Lambda POST function is correctly writing data into the `ServerlessTable` DynamoDB table.

### 🧭 Steps:
1. Go to the **AWS Console** → Navigate to **DynamoDB**.  
2. Click on **Tables** in the sidebar.  
3. Select the table, for example, `ServerlessTable`.  
4. Click on the **Explore items** tab.

### 🔍 What You Should See:
- This view displays all the items (records) currently stored in the table.  
- If the Lambda POST function ran successfully, you should now see the student record(s) that were added during the test, including fields like `studentId`, `name`, `class`, and `age`.
![10](https://github.com/user-attachments/assets/6b2cc875-4ffa-412a-be1f-8ddcb24eeda2)
✅ This step confirms that the Lambda and DynamoDB integration is working as intended.

# PART 2: Create API Gateway and S3 Bucket
### 🎯 Goal:
Instead of manually triggering the Lambda function (like in Part 1), we'll set up an **API Gateway** that acts as the middleman between the web app and the Lambda function. This way, the user can submit data through the web interface, and it will be automatically sent to DynamoDB via:


**Amazon API Gateway** will be used to create a REST API.
- The API will expose a public HTTP endpoint that the frontend (static website) can call.  
- When a user submits a form, it sends a request to this API endpoint, which then triggers the Lambda POST function.  
- This function adds data to the DynamoDB table (e.g., `ServerlessTable`).

---

### 🔑 What is an API Endpoint?
An API endpoint is a URL exposed by the API Gateway. Think of it as the address where the frontend sends data (via JavaScript `fetch()` or `axios.post()`), and in return, the backend (the Lambda) processes it.

> 📌 This endpoint acts like the **bridge** between the web app and the backend logic.

---

### 🌐 Step 1: Create an API Gateway for POST Requests
1. Go to the **AWS Console** → Navigate to **API Gateway**.  
2. Under API types, choose **REST API**.  
   - 🔹 REST API is used to build traditional HTTP-based APIs, ideal for integrating with Lambda functions using standard methods like POST, GET, etc.  
3. Click **Build** under REST API.

#### 📋 API Details
- **Choose protocol:** Select **New API**.  
- **API name:** Enter a name, e.g., `APIforStudentData`.  
- **Endpoint type:** Select **Edge Optimized**.  

> 🌍 Edge Optimized means the API will be accessible globally, using AWS CloudFront to cache and route requests from edge locations — great for public-facing apps.>
![11](https://github.com/user-attachments/assets/344c875d-e617-411d-9ade-dcc0f7dd757e)
4. Click **Create API**.

## Step 2: Create GET and POST Methods in API Gateway
### 🎯 Objective:
Connect API Gateway to the Lambda functions by defining HTTP methods:  
- **GET** → To retrieve student data from DynamoDB (handled by the Lambda from Part 1, Step 3).  
- **POST** → To submit new student data (handled by the Lambda from Part 1, Step 5).  

This allows external clients (like the web app) to access these functions using standard HTTP calls.

---

### 🧭 Steps to Create the GET Method:
1. In the API Gateway console, select the API you created (e.g., `APIforStudentData`).  
2. In the left menu under **Resources**, click `/` (the root resource).  
3. Click **Actions** → **Create Method**.  
4. Select **GET** from the dropdown and click the checkmark ✅.  
5. Under **Integration type**, choose **Lambda Function**.  
6. Select the AWS region where the Lambda was created (e.g., `us-east-1`).  
7. In the function name field, type the name of the Lambda — e.g., `LambdaFunctionForServerless` (the one from Part 1, Step 3).  
8. Click **Save**, then **OK** to give permission to API Gateway to invoke the Lambda function.
![12](https://github.com/user-attachments/assets/e95d080e-7272-4af0-b060-4ef8a5bb3f24)

### Test the GET Configuration:
1. Click on the **GET** method in the method list.  
2. Click the **Test** button at the top.  
3. Click **Test** again to invoke the Lambda function.  
✅ This should return the list of items stored in the DynamoDB table (`studentData`), confirming that the Lambda integration works.
![13](https://github.com/user-attachments/assets/28248b26-b1b9-44c3-852d-cb7f80423f5d)

### Repeat for POST Method
Repeat the same steps as the GET method for the **POST** method, selecting the Lambda function created in Part 1, Step 5 (e.g., `LambdaPOSTFunction`).

🔧 Once both methods are created, you’ll be able to retrieve and submit student data using HTTP GET and POST requests via the API Gateway endpoints.

---

## Step 3: Deploy API
### 🎯 Objective:
To make the API Gateway live and accessible via a public HTTP endpoint, we need to deploy it to a stage (e.g., Production). This step generates the actual URL (API endpoint) that clients like the frontend can use to call the backend Lambda functions.

### 🧭 Steps:
1. In the API Gateway console, click the **Actions** dropdown.  
2. Select **Deploy API**.  
3. For **Stage**, choose **New Stage**.  
4. **Stage Name:** Enter a name, for example, `Production`.  
5. Click **Deploy**.
![14](https://github.com/user-attachments/assets/07f3dfe2-6cc3-422f-b27f-2c8069f37954)

After deployment, the given URL will look like:
https://yk1fhp0nkg.execute-api.us-east-1.amazonaws.com/Production
This URL is now the **live API endpoint**.
When a request (like a form submission) is made to this URL with the correct path (e.g., `/POST`), API Gateway forwards the request to the corresponding Lambda function.
![15](https://github.com/user-attachments/assets/f52590ba-4581-443d-98c9-ab336dae130e)

## Step 4: Update the `scripts.js` File
### 🎯 Objective:
To connect the frontend web app to the deployed API Gateway endpoint so it can trigger Lambda functions.

### 🧭 Steps:
1. Open the `script.js` file (or whichever JS file is handling API requests).  
2. Locate the placeholder:

```js
var API_ENDPOINT = "API_ENDPOINT_REPLACE_HERE";
```
![16](https://github.com/user-attachments/assets/a97ae022-2504-4a27-8528-cbdb01794b7e)

Replace the placeholder with the actual Invoke URL from the API Gateway deployment, for example:

```js
var API_ENDPOINT = "https://abc123.execute-api.us-east-1.amazonaws.com/Production";
```
### What Happens Next?
When users click a button like **"Get Student Data"** in the web app:  
- JavaScript sends a **GET** request to the API endpoint.  
- API Gateway triggers **GET** Lambda function.  
- Lambda retrieves the data from DynamoDB and sends it back.  
- The data is then displayed in the web app.  

✅ The backend is now fully connected to the frontend via API Gateway.

## Step 5: Enable CORS (Cross-Origin Resource Sharing)
### 🎯 Objective:
To allow the frontend hosted in S3 (or another domain) to communicate with the API Gateway without being blocked by browser security policies.

### 🧭 Steps to Enable CORS in API Gateway:
1. Go to the **API Gateway** console.  
2. Select the API (e.g., `APIforStudentData`).  
3. Under **Resources**, click the `/` root resource.  
4. Click **Actions** → **Enable CORS**.  
5. Under **Access-Control-Allow-Methods**, check the boxes for the **GET** and **POST** methods.  
![17](https://github.com/user-attachments/assets/17611ccc-ca77-4bc8-977b-1e6680d8c3c7)
6. Keep the default settings (you can update them if needed):
   - **Access-Control-Allow-Headers:** `Content-Type,X-Amz-Date,Authorization,X-Api-Key`
   - **Access-Control-Allow-Methods:** `GET,POST,OPTIONS`
   - **Access-Control-Allow-Origin:** `*`
7. Click **Enable CORS and replace existing CORS headers**.
8. Then click **Actions** → **Deploy API** again to apply the changes.

## Step 6: Create an S3 Bucket for Static Website Hosting
The frontend of the web application (HTML, CSS, JavaScript) will be stored in an S3 bucket.  
S3 will serve the static files to users' browsers.  

In the JavaScript (e.g., in a script file like `script.js`), the API endpoint URL created above has been referenced.  
When the user submits a form on the webpage, it calls that endpoint to send the data to DynamoDB.

### Steps:
1. Open the **S3 Dashboard**.  
2. Click **Create bucket**.  
3. Under **Bucket name**, enter a globally unique name, e.g., `webappforstudentidentity.com`.
![18](https://github.com/user-attachments/assets/9c118fcb-db52-4650-abf6-7bd5a3a5fbe4)
4. Leave all default settings as they are (including ACLs and block public access) — you'll adjust permissions later for static hosting.  
5. Scroll down and click **Create bucket**.

## Step 7: Upload Website Files and Enable Static Hosting

🎯 **Objective:**  
To host the static frontend (HTML & JS) in S3 so users can access it via a public URL in their browser.

🧭 **Steps:**  
1. Go to **S3** → Click on the created bucket (e.g., `webappforstudentidentity.com`).  
2. Click **Upload**.  
3. Add the files:  
   - `index.html`  
   - `script.js`
![19](https://github.com/user-attachments/assets/067ff23a-6a6f-43ed-af57-d57ed4c15ef5)
4. Click **Upload** to complete.

### ⚙️ Enable Static Website Hosting
This is enabled to make the bucket act like a web server that can serve HTML pages.
1. Go to the **Properties** tab of the bucket.  
2. Scroll down to **Static website hosting**.  
3. Click **Edit**, then toggle **Enable**.  
4. For **Index document**, type: `index.html` (since that’s the name of the main page).  
5. Click **Save changes**.
![20](https://github.com/user-attachments/assets/a96c4af5-a6b9-4239-bbb8-54f3726d69e5)
🔗 Once saved, you’ll see a **Website endpoint URL** — this is the public link to the hosted site.
![21](https://github.com/user-attachments/assets/a5015b92-bac4-4275-9a51-512838e6a8a4)

## Step 8: Allow Public Access to the Bucket
By default, S3 blocks public access to all buckets. To make the website accessible from the browser, we need to update these settings.

### 🧭 Steps:
1. Go to the **Permissions** tab of the bucket.  
2. Click **Edit** under **Block public access (bucket settings)**.  
3. Uncheck **Block all public access**.  
4. Confirm by checking the acknowledgment box.
![22](https://github.com/user-attachments/assets/512a7048-5e5d-4f88-9082-dca64f767104)
7. Click **Save changes**.

## Step 9: Add a Bucket Policy to Allow Public Access to Web Files
Now that the block is lifted, define who can access the content using a bucket policy.

### 🧭 Steps:
1. In the **Permissions** tab, scroll to **Bucket policy**.  
2. Click **Edit**.  
3. Paste the following policy (update the bucket name in the ARN):  

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Statement1",
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::webappforstudentidentity.com/*"
    }
  ]
}
```
![23](https://github.com/user-attachments/assets/2e0e98ca-a1b3-4cb9-8efc-aebee8cf7636)

Click **Save changes**.  
✅ Now the static website is publicly accessible!
Open the **Website endpoint URL** (found under **Properties > Static website hosting**) in the browser to see the web app live.
The JavaScript inside the `index.html` will call the API Gateway endpoint to interact with the backend Lambda and DynamoDB functions.
![24](https://github.com/user-attachments/assets/6da24198-8d11-4244-89e1-fb5913828c52)

✅ **Test the Web App Workflow**
1. **Open the Web App**  
   Go to S3 website endpoint URL in the browser.  
   You should see the form and any buttons for submitting and viewing data.

2. **Submit (POST) Student Data**  
   Fill out the form fields, for example:  
   - Student ID: `123456`  
   - Name: `John`  
   - Class: `Math101`  
   - Age: `21`
![25](https://github.com/user-attachments/assets/d20b53e6-62b9-404c-afdf-9c7a85ef9142)
Click the **"Save"** or **"Submit"** button (depending on the label).
✅ You should see a success message like:  
`Student Data Saved!`

### 🔧 Behind the Scenes (for Submit):
- JavaScript sends a **POST** request to **API Gateway**.
- API Gateway triggers the **Lambda POST function**.
- Lambda writes the data to the **DynamoDB** table.

---

3. **View (GET) Student Data**  
Click on the **“View all Students”** button.
✅ You should see a list or table displaying the student(s) entered.

### 🔧 Behind the Scenes (for View):

- JavaScript sends a **GET** request to the **API Gateway** endpoint.
- API Gateway triggers the **Lambda GET function**.
- Lambda retrieves data from **DynamoDB** and returns it to the frontend.

## Part 3: Place CloudFront in Front of S3 Bucket

### 🎯 Objective:
To improve the performance, scalability, and security of the static web app, we place **CloudFront**, a global content delivery network (CDN), in front of the S3 bucket.  
CloudFront caches the website content at edge locations worldwide, making it load faster for users regardless of their geographic location.

---

### 🧭 Steps:

#### 1. Access CloudFront in AWS Console
- Go to the **CloudFront** service in the AWS Console.
- Click **Create Distribution**.

#### 2. Configure Origin
- For **Origin Domain Name**, enter the **S3 website endpoint URL**, for example:  
  `http://webappforstudentidentity.com.s3-website-us-east-1.amazonaws.com`  
  *(Alternatively, click "Browse S3" and select the bucket directly.)*
- For **Origin Type**, choose **S3 Origin**.
- ![Screenshot from 2025-06-24 01-53-46](https://github.com/user-attachments/assets/37dceb9c-1a07-47bd-a1f6-8ce4362a2220)


#### 3. Configure Distribution Settings
- Give the distribution a name, e.g., `webappdistribution`.
- Under **Security Settings**, skip WAF (unless the app needs it).
- Click **Next** to review.
- Click **Create Distribution**.

---

### 🔐 Update S3 Bucket Permissions
Once the distribution is created:
1. Go to the **Origins** tab in the CloudFront distribution.
2. Select the **S3 origin** and click **Edit**.
3. Click **Go to S3 bucket permissions**.

In the S3 Console:
4. Enable **Block all public access** to make sure the S3 bucket is only accessible via CloudFront.
> ✅ This ensures the static content is securely served via CloudFront and not directly from the bucket.<

## 🔐 S3 Bucket Policy (After Blocking Public Access)
When blocking public access, AWS automatically updates the **S3 bucket policy** with a **CloudFront-specific policy**.
- ✅ This policy allows **CloudFront** to access the bucket content securely **without making the bucket publicly accessible**.
- 🔒 This means the S3 content is only accessible through the **CloudFront distribution**, protecting it from direct public access.

---

## 💾 Save Changes
- Go back to the **CloudFront console** and save the **origin changes**.

---

## 🌐 Access the CloudFront URL
- In **CloudFront**, under the **General** tab of the distribution, copy the **Distribution Domain Name URL**, for example:  
  `https://dwyhlajrdos2b.cloudfront.net`

- This is the new public URL for the **web application**, served via **CloudFront**.

---

## ✅ Summary
By placing **CloudFront in front of the S3 bucket**:
- 🌍 The website **loads faster globally** through edge caching.
- 🔐 S3 bucket is **secured** by limiting **direct public access**.
- 🔒 Users gain **HTTPS support** and more reliable, scalable delivery.

## 🧹 Cleanup: Delete All Related AWS Services

When you're finished with the project or want to avoid additional charges, you should remove all related AWS resources.

---

### 1. 🗑️ Delete Lambda Functions
- Go to the **AWS Lambda Console**
- Select each Lambda function (e.g., `LambdaFunctionForServerless`, `LambdaPOSTFunction`)
- Click **Actions → Delete**
- Confirm deletion

---

### 2. 🗑️ Delete API Gateway
- Go to the **API Gateway Console**
- Select the API you created (e.g., `APIforStudentData`)
- Click **Actions → Delete API**
- Confirm deletion

---

### 3. 🗑️ Delete DynamoDB Table
- Go to the **DynamoDB Console**
- Select the table (e.g., `ServerlessTable`)
- Click **Actions → Delete Table**
- Confirm deletion

---

### 4. 🗑️ Delete S3 Bucket
- Go to the **S3 Console**
- Select the bucket (e.g., `webappforstudentidentity.com`)
- Delete all objects inside the bucket first (select all files → **Delete**)
- Once the bucket is empty, select it → Click **Delete bucket**
- Confirm deletion by typing the bucket name

---

### 5. 🗑️ Delete CloudFront Distribution
- Go to the **CloudFront Console**
- Select the distribution you created
- Click **Disable distribution** (wait for status to change to `Disabled`)
- Once disabled, select it again → Click **Delete**
- Confirm deletion

---

### 6. 🗑️ Delete IAM Roles (Optional)
- Go to the **IAM Console → Roles**
- Find the Lambda execution role(s) (e.g., `RoleForLambda`)
- Select the role → Click **Delete role**
- Confirm deletion

---

### 7. 🧾 Check for Other Resources
- Go to **CloudWatch Console → Log Groups**
- Delete any leftover log groups related to the Lambda functions
- Review other services for unused resources and delete them if necessary

Project Inspiration: https://www.youtube.com/watch?v=pK52mfm69i0

