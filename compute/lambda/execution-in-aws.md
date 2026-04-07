# Execution in AWS

## 🚀 Create AWS Lambda Function (From Console)

***

### 🔹 Steps (Detailed Explanation)

#### 🔐 Step 1: Login to AWS Console

* Open browser → [https://console.aws.amazon.com](https://console.aws.amazon.com)
* Enter:
  * Email / Account ID
  * Password
* Click **Sign in**

👉 You’ll land on AWS home dashboard (search bar + services)

***

#### 🔍 Step 2: Search → Lambda

* In top search bar → type **Lambda**
* Click\
  👉 AWS Lambda

👉 You will see:

* Left panel → Functions, Layers
* Center → “Create function” button

***

#### ➕ Step 3: Click Create Function

* Click **Create function** (top right)

👉 New page opens with 3 options:

* Author from scratch ✅
* Use blueprint
* Container image

***

#### 🧩 Step 4: Select Author from scratch

* Choose **Author from scratch**

👉 This allows you to create custom Lambda

***

### 🔹 Configuration (Detailed)

***

#### 🏷️ Function Name

* Enter:

```
demo-lambda
```

👉 This becomes your Lambda identifier

***

#### 🐍 Runtime

* Select:

```
Python 3.x (latest available)
```

👉 AWS will:

* Provide Python runtime
* Execute your code inside managed container

***

#### ⚙️ Architecture

* Select:

```
x86_64
```

👉 Standard architecture (default, widely supported)

***

#### 🔐 Permissions (Very Important)

* Select:\
  👉 **Create a new role with basic Lambda permissions**

👉 Behind the scenes:

* AWS creates IAM role
* Attaches policy for:\
  👉 Amazon CloudWatch

👉 This allows:

* Writing logs
* Monitoring execution

***

#### ▶️ Final Step

👉 Click **Create Function**

⏳ Wait few seconds → Lambda is created

***

## 🔹 Add Code (Detailed)

***

#### 🧭 Step 1: Go to Code Tab

* After creation → you land on function page
* Default tab → **Code**

***

#### ✏️ Step 2: Open File

* Click:

```
lambda_function.py
```

***

#### ✏️ Step 3: Replace Code

```
def lambda_handler(event, context):
    print("Hello this is Lambda execution")
    return {
        'statusCode': 200,
        'body': 'Hello from Lambda'
    }
```

👉 Explanation:

* `event` → input data
* `context` → runtime metadata
* `print()` → goes to logs
* return → API response

***

#### 💾 Step 4: Deploy

👉 Click **Deploy** (top right)

⚠️ Important:

* Without Deploy → code will NOT execute

***

## 🔹 Test Lambda (Detailed)

***

#### 🧪 Step 1: Click Test

* Click **Test** button

***

#### 🧾 Step 2: Create Event

Popup opens:

* Event name → `test-event`
* Keep JSON:

```
{}
```

👉 Click **Save**

***

#### ▶️ Step 3: Run Test

👉 Click **Test** again

***

#### ✅ Output (What You’ll See)

**Execution Status:**

```
Succeeded
```

**Response:**

```
{
  "statusCode": 200,
  "body": "Hello from Lambda"
}
```

***

#### 📜 Logs (Very Important)

* Click **View logs**
* Opens\
  👉 Amazon CloudWatch

**Logs show:**

```
Hello this is Lambda execution
```

***

## 🚀 3️⃣ Create Lambda using ZIP (Local Upload)

***

### 🔹 Step 1: Write Code Locally

Create file:

```
test_lambda.py
```

👉 Same function inside it

***

### 🔹 Step 2: Install Dependencies

Run:

```
pip3 install -r requirements.txt -t .
```

👉 This installs packages into current folder\
👉 Required because Lambda doesn’t install dependencies automatically

***

### 🔹 Step 3: Create ZIP

Zip should include:

* test\_lambda.py
* dependencies folder
* requirements.txt

👉 Output:

```
lambda.zip
```

***

### 🔹 Step 4: Upload to Lambda

* Go to Lambda → Code tab
* Click:

```
Upload from → .zip file
```

* Upload `lambda.zip`

***

### 🔹 Step 5: Configure Handler

Go to:

* Configuration → Runtime settings → Edit

Set:

```
test_lambda.lambda_handler
```

👉 Format:

```
filename.function_name
```

***

### 🔹 Step 6: Test

👉 Click **Test**

✔ Verify:

* Output
* Logs

***

## 🚀 4️⃣ Create Lambda using S3

***

### 🔹 Step 1: Upload ZIP to S3

Go to\
👉 Amazon S3

Steps:

* Create bucket (if not exists)
* Click **Upload**
* Upload `lambda.zip`

***

### 🔹 Step 2: Use S3 in Lambda

* Go to Lambda → Code tab
* Click:

```
Upload from → Amazon S3
```

* Paste:

```
s3://bucket-name/lambda.zip
```

👉 Click **Save**

***

### 🔹 Step 3: Test

👉 Click **Test** → Verify output

***

## 🚀 5️⃣ Enable Lambda Function URL

***

### 🔹 Steps

* Go to **Configuration tab**
* Click:

```
Function URL
```

* Click:

```
Create Function URL
```

***

### 🔹 Authentication Types

***

#### 🌐 Public Access

* Select:

```
NONE
```

👉 Anyone with URL can access

***

#### 🌍 Test

* Copy URL
* Open in browser

👉 Output:

```
Hello Lambda from my code
```

***

#### 🔐 Secure Access (IAM)

* Select:

```
AWS_IAM
```

👉 Requires signed request

Use:

* Postman (AWS Signature)

***

## 🚀 6️⃣ Environment Variables in Lambda

***

### 🔹 Step 1: Add Variables

* Go to Configuration → Environment Variables
* Click **Edit → Add**

Add:

```
Key: MY_VAR
Value: test_value
```

***

### 🔹 Step 2: Use in Code

```
import os

def lambda_handler(event, context):
    value = os.environ.get("MY_VAR")
    print(value)
```

***

### 🔹 Step 3: Deploy & Test

👉 Output:

```
test_value
```

***

## 🚀 7️⃣ AWS Lambda Layers

***

### 🔹 What is Lambda Layer?

Reusable shared code across functions

***

### 🔹 Folder Structure

```
python/
 └── lib/
     └── python3.x/
         └── site-packages/
             └── my_module/
```

***

### 🔹 Step 1: Create ZIP

Zip the `python/` folder

***

### 🔹 Step 2: Create Layer

* Go to Lambda → Layers
* Click **Create layer**
* Upload ZIP
* Select runtime

***

### 🔹 Step 3: Attach Layer

* Open Lambda
* Scroll → Layers
* Click **Add layer**

***

### 🔹 Step 4: Use Layer in Code

```
from my_module.my_function import my_function

def lambda_handler(event, context):
    result = my_function()
    print(result)
```

***

### ✅ Output

```
Hello from Lambda Layer
```

***

## 🎯 Final Summary (Clean)

You covered:

* Lambda creation (console)
* ZIP deployment
* S3 deployment
* Function URL
* Environment variables
* Layers

***

## 💡 Pro DevOps Tip (Important for YOU)

#### ❌ Don’t rely on console

#### ✅ Use:

* Terraform
* AWS CLI
* CI/CD (GitHub Actions)

👉 Console = Learning / Debugging\
👉 IaC = Production
