# MERN Stack Blog App Deployment on AWS - My Journey

This document details my steps to deploy a MERN (MongoDB, Express, React, Node.js) stack blog application on AWS using free-tier eligible services, as part of the Week 10 Infrastructure Bootcamp Assignment. This is a record of my process, challenges, and solutions.

## Part 1: MongoDB Atlas Configuration (Optional)

I decided to use my existing MongoDB Atlas cluster for this project, so I skipped creating a new account and cluster.

## Part 2: S3 Bucket for Frontend

This part involved setting up the S3 bucket to host the React frontend.

1.  **Created the bucket:** I created a bucket named `mafaz-blogapp-frontend` in the `eu-north-1` region.
2.  **Disabled public access block:** I unchecked the "Block all public access" option in the bucket's permissions settings.
3.  **Enabled static website hosting:** I enabled static website hosting for the bucket, setting the index document to `index.html`.
4.  **Added the bucket policy:** I added the following bucket policy to allow public read access:

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

    I made sure to replace `yourname-blogapp-frontend` with my actual bucket name.

## Part 3: S3 for Media Uploads

Next, I configured the S3 bucket for media uploads.

1.  **Created the bucket:** I created a second bucket named `yourname-blogapp-media` (again, replacing `yourname` with my name).
2.  **Disabled public access block:** I unchecked the "Block all public access" option.
3.  **Configured CORS:** I added the following CORS configuration:

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

4.  **Tested file upload and retrieval:** I uploaded a test file to the bucket and verified that I could retrieve it successfully.

**Deliverables:**

*   **Screenshot of file upload:** I took a screenshot showing the file successfully uploaded to the `yourname-blogapp-media` bucket in the AWS Console.

## Part 4: IAM User and Policy for S3 Media Bucket Access

To control access to the media bucket, I created an IAM user with specific permissions.

1.  **Created the IAM user:** I went to the IAM Console and created a new user named `blog-app-user`.
2.  **Created a custom policy:** I created a custom policy with the following JSON, replacing `yourname-blogapp-media` with my actual bucket name:

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

3.  **Attached the policy to the user:** I attached the custom policy to the `blog-app-user`.
4.  **Saved the credentials:** I carefully saved the Access Key ID and Secret Access Key for the user. **Important: I did not share these credentials in my submission or GitHub repo!**

## Part 5: EC2 Backend Setup

This was the most involved part, setting up the EC2 instance and deploying the backend.

1.  **Launched the EC2 instance:** I launched a `t3.micro` instance in `eu-north-1` using Ubuntu 22.04 LTS.
2.  **Configured security group:** I configured the security group to allow incoming traffic on ports 22 (SSH), 80 (HTTP), 443 (HTTPS), and 5000 (Custom TCP for the backend).
3.  **Ran the User Data script:** I SSHed into the instance and ran the provided User Data script to install necessary software (Git, Node.js, PM2, AWS CLI, etc.).
4.  **Cloned the MERN app:** I cloned the MERN app repository:

    ```bash
    git clone https://github.com/cw-barry/blog-app-MERN.git /home/ubuntu/blog-app
    cd /home/ubuntu/blog-app
    ```

5.  **Configured the backend:** I updated the `.env` file in the `backend` directory with my MongoDB connection string, JWT secrets, and AWS S3 configuration. I made sure *not* to include my actual AWS Access Key ID and Secret Access Key in the file that was committed to the repository.
6.  **Configured the frontend:** I updated the `.env` file in the `frontend` directory with the backend API URL and media base URL.
7.  **Configured AWS CLI:** I ran `aws configure` and entered the IAM user credentials, region, and output format.
8.  **Deployed the backend:** I navigated to the `backend` directory, installed dependencies, and started the server using PM2:

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

9.  **Built and deployed the frontend:** I navigated to the `frontend` directory, installed dependencies, built the frontend, and deployed it to the S3 bucket:

    ```bash
    cd /home/ubuntu/blog-app/frontend
    npm install -g pnpm@latest-10
    pnpm install
    pnpm run build
    aws s3 sync dist/ s3://<your-s3-frontend-bucket-name>/
    ```

**Deliverables:**

*   **Screenshot of backend server running via pm2:** I ran `pm2 list` on my EC2 instance and took a screenshot to show the backend server running.
*   **Screenshot of frontend s3 web page:** I accessed the frontend application via the S3 static website hosting URL and took a screenshot of the running application.

## Cleanup

After completing the assignment, I:

*   Stopped the EC2 instance.
*   Removed the IAM user credentials.
*   Deleted the S3 buckets.

## Troubleshooting

During the process, I encountered a few issues:

*   **Website not loading initially:** This was due to a misconfigured bucket policy. I corrected the policy and the website started loading.
*   **Media uploads failing:** I realized I had not configured the CORS settings correctly on the media bucket. After updating the CORS configuration, media uploads started working.
*   **MongoDB connection errors:** I had forgotten to whitelist the EC2 instance's IP address in the MongoDB Atlas network access settings. Once I added the IP address, the backend was able to connect to the database.

This document reflects my personal experience deploying the MERN stack blog application on AWS. Your experience may vary, but I hope this provides a helpful guide.
