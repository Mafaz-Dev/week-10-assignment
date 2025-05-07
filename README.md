# MERN Stack Blog App Deployment on AWS
Infrastructure Bootcamp Assignment

## Part 1: MongoDB Atlas Configuration (Optional)

This step is optional if you want to use your own MongoDB Atlas cluster. You can use the existing MongoDB connection string in the backend `.env` file if you prefer.

1.  **Create a MongoDB Atlas account:** Go to [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) and create an account.
2.  **Set up a free cluster:** Create a free-tier MongoDB cluster.
3.  **Allow EC2 IP in network access settings:** Add your EC2 instance's public IP address to the Atlas cluster's network access settings to allow connections from your EC2 instance.
4.  **Create a database user and obtain the connection string:** Create a database user with appropriate permissions and obtain the MongoDB connection string.

## Part 2: S3 Bucket for Frontend

1.  **Create a bucket named `yourname-blogapp-frontend` in `eu-north-1`:** Replace `yourname` with your actual name.
2.  **Disable "Block all public access":** Uncheck the "Block all public access" option in the bucket's permissions settings.
3.  **Enable static website hosting:** Enable static website hosting for the bucket in the "Static website hosting" section of the bucket's properties. Set the index document to `index.html`.
4.  **Add a bucket policy for public access:** Add the following bucket policy to grant public read access to the bucket's objects:

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Principal": "*",
          "Action": "s3:GetObject",
          "Resource": "arn:aws:s3:::yourname-blogapp-frontend/*"
        }
      ]
    }
    ```

    Replace `yourname-blogapp-frontend` with your actual bucket name.

**Deliverables:**

*   **Screenshot of S3 public URL loading `index.html`:**  Access the S3 static website hosting endpoint in your browser and take a screenshot of the page.
*   **`curl -I` output showing 200 OK:** Run the following command in your terminal (replace the URL with your S3 endpoint) and take a screenshot of the output:

    ```bash
    curl -I http://yourname-blogapp-frontend.s3-website.eu-north-1.amazonaws.com/index.html
    ```

## Part 3: S3 for Media Uploads

1.  **Create a second bucket: `yourname-blogapp-media`:** Replace `yourname` with your actual name.
2.  **Disable "Block all public access":** Uncheck the "Block all public access" option in the bucket's permissions settings.
3.  **Configure CORS for browser upload support:** Add the following CORS configuration to the bucket:

    ```json
    [
      {
        "AllowedHeaders": ["*"],
        "AllowedMethods": ["GET", "PUT", "POST", "DELETE", "HEAD"],
        "AllowedOrigins": ["*"],
        "ExposeHeaders": ["ETag"]
      }
    ]
    ```

4.  **Test uploading and retrieving a file:** Upload a file to the bucket using the AWS CLI or the AWS Management Console, and then verify that you can retrieve it.

**Deliverables:**

*   **Screenshot of file upload:** Take a screenshot showing the file successfully uploaded to the `yourname-blogapp-media` bucket.

## Part 4: IAM User and Policy for S3 Media Bucket Access

1.  **Go to IAM Console > Users > Add users:**
2.  **Username: `blog-app-user`**
3.  **Permissions > Attach existing policies directly**
4.  **Create a custom policy** with the following JSON (replace `yourname-blogapp-media` with your actual bucket name):

    ```json
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": [
            "s3:PutObject",
            "s3:GetObject",
            "s3:DeleteObject",
            "s3:ListBucket"
          ],
          "Resource": [
            "arn:aws:s3:::yourname-blogapp-media",
            "arn:aws:s3:::yourname-blogapp-media/*"
          ]
        }
      ]
    }
    ```

5.  **Attach this policy to the user.**
6.  ***Important:*** Create and save the **Access Key ID** and **Secret Access Key** for this user.  You will not be able to view the secret key again. **DO NOT share these credentials in your submission or solution GitHub repo!**

## Part 5: EC2 Backend Setup

1.  **Launch a `t3.micro` instance in `eu-north-1` using Ubuntu 22.04 LTS:**
2.  **Allow incoming traffic only on necessary ports:** (SSH: 22, HTTP: 80, HTTPS: 443, Custom TCP:5000). Add this Inbound rule:

    | Type       | Protocol | Port Range | Source     | Description          |
    | ---------- | -------- | ---------- | ---------- | -------------------- |
    | Custom TCP | TCP      | 5000       | 0.0.0.0/0 | Allow public traffic |

3.  **SSH into the instance and run the User Data script below:**

    ```bash
    #!/bin/bash
    apt update -y
    apt install -y git curl unzip tar gcc g++ make unzip

    su - ubuntu << 'EOF'
    export NVM_DIR="$HOME/.nvm"
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
    source "$NVM_DIR/nvm.sh"
    nvm install --lts
    nvm use --lts
    npm install -g pm2
    EOF

    # MongoDB Shell
    curl -L https://downloads.mongodb.com/compass/mongosh-2.1.1-linux-x64.tgz -o mongosh.tgz
    tar -xvzf mongosh.tgz
    mv mongosh-*/bin/mongosh /usr/local/bin/
    chmod +x /usr/local/bin/mongosh
    rm -rf mongosh*

    # AWS CLI
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    ./aws/install
    rm -rf aws awscliv2.zip
    ```

4.  **Clone the MERN app from this repo url:**

    ```bash
    git clone https://github.com/cw-barry/blog-app-MERN.git /home/ubuntu/blog-app
    cd /home/ubuntu/blog-app
    ```

5.  **Configure your MERN app Backend:**  Update the `.env` file in the `backend` directory with your MongoDB connection string, JWT secrets, and AWS S3 configuration. **Do not share your AWS Access Key ID and AWS Secret Access Key in your submission or GitHub repo!**

    ```bash
    cd backend
    cat > .env << EOF
    PORT=5000
    HOST=0.0.0.0
    MODE=production
    MONGODB=mongodb+srv://test:qazqwe123@mongodb.txkjsso.mongodb.net/blog-app # Replace with your MongoDB connection string
    JWT_SECRET=$(openssl rand -hex 32)
    JWT_EXPIRE=30min
    JWT_REFRESH=$(openssl rand -hex 32)
    JWT_REFRESH_EXPIRE=3d
    AWS_ACCESS_KEY_ID=<access-key-from-step-6> # Replace with your IAM user's access key
    AWS_SECRET_ACCESS_KEY=<secret-key-from-step-6> # Replace with your IAM user's secret key
    AWS_REGION=eu-north-1
    S3_BUCKET=<your-s3-media-bucket-name> # Replace with your media bucket name
    MEDIA_BASE_URL=https://<your-s3-media-bucket-name>.eu-north-1.amazonaws.com
    DEFAULT_PAGINATION=20
    EOF
    ```

6.  **Configure your MERN app Frontend:** Update the `.env` file in the `frontend` directory with the backend API URL and media base URL.

    ```bash
    cd ../frontend
    cat > .env << EOF
    VITE_BASE_URL=http://<your-ec2-dns>:5000/api # Replace with your EC2 instance's public DNS and backend port
    VITE_MEDIA_BASE_URL=https://<your-s3-media-bucket-name>.eu-north-1.amazonaws.com # Replace with your media bucket name
    EOF
    ```

7.  **Configure AWS CLI:**

    ```bash
    aws configure
    # Enter your AWS Access Key ID when prompted
    # Enter your AWS Secret Access Key when prompted
    # Set default region to eu-north-1
    # Set default output format to json
    ```

8.  **Deploy Backend:**

    ```bash
    cd /home/ubuntu/blog-app/backend
    npm install
    mkdir -p logs
    pm2 start index.js --name "blog-backend"
    pm2 startup
    sudo pm2 startup systemd -u ubuntu
    sudo env PATH=$PATH:/home/ubuntu/.nvm/versions/node/v22.15.0/bin /home/ubuntu/.nvm/versions/node/v22.15.0/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
    pm2 save
    ```

9.  **Build and Deploy Frontend:**

    ```bash
    cd /home/ubuntu/blog-app/frontend
    npm install -g pnpm@latest-10
    pnpm install
    pnpm run build
    aws s3 sync dist/ s3://<your-s3-frontend-bucket-name>/ # Replace with your frontend bucket name
    ```

**Deliverables:**

*   **Screenshot of backend server running via pm2:** Run `pm2 list` on your EC2 instance and take a screenshot.
*   **Screenshot of frontend s3 web page:** Access your frontend application via the S3 static website hosting URL and take a screenshot of the running application.

## Cleanup

*   Stop the EC2 instance to avoid incurring unnecessary charges.
*   Remove the IAM user credentials to prevent unauthorized access.
*   Delete the S3 buckets if they are no longer needed.

## Troubleshooting

*   **Website not loading:** Check the S3 bucket policy, CORS configuration, and ensure that the `index.html` file exists in the bucket.
*   **Backend server not running:** Check the PM2 logs for errors.
*   **Media uploads failing:** Verify the IAM user permissions and CORS configuration on the media bucket.
*   **MongoDB connection errors:** Ensure that your EC2 instance's IP address is whitelisted in the MongoDB Atlas network access settings and that the connection string is correct.

This README provides a comprehensive guide to deploying the MERN stack blog application on AWS. Remember to replace placeholder values with your actual configuration details. Good luck!
