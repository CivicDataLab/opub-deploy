# NextJS Auto Deployment

This guide will help you set up a Next.js project to automatically deploy to an EC2 instance using GitHub Actions.

## GitHub Repository Setup

Create a GitHub repository, for example: `opub-deploy`.

## Local Development Setup

1.  Generate a Next.js project:

```bash
npx create-next-app@latest opub-deploy
cd opub-deploy
npm install
```

2.  Add the following code to `.github/workflows/deploy.yml` in your project:

```yaml
name: Deploy Next.js to EC2

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Install dependencies
        run: npm install

      - name: Build
        run: npm run build

      - name: Rename .next to .next2
        run: mv .next .next2

      - name: Send .next2 to EC2
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_PRIVATE_KEY }}
          source: .next2
          target: opub-deploy
      - name: Update with new Build
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USERNAME }}
          key: ${{ secrets.EC2_PRIVATE_KEY }}
          script: rm -rf opub-deploy/.next; mv opub-deploy/.next2 opub-deploy/.next; pm2 restart opub-deploy
```

> Replace all instances `opub-deploy` with your project name in EC2.

3. Initialize a Git repository:

```bash
git init
git add .
git branch -M main
git remote add origin git@github.com:CivicDataLab/opub-deploy.git
git commit -m "first commit"
git push -u origin main
```

## Server Setup

### EC2 Setup

1.  Create an EC2 instance with Ubuntu 20.04.
2.  Create a new key pair and download the `.pem` file.
3.  Create a new security group with the following inbound rules:
    1.  SSH (22) from anywhere
    2.  HTTP (80) from anywhere
    3.  HTTPS (443) from anywhere
    4.  Custom TCP (3000) from anywhere
4.  Connect to the EC2 instance using SSH:

```bash
ssh -i "path/to/key.pem" username@publicIP
```

### Add packages

Install Node.js and npm using [NVM](https://github.com/nvm-sh/nvm):

```bash
sudo apt update

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

Install PM2 globally:

```bash
npm install pm2 -g
```

Create symbolic links

```bash
sudo ln -s "$NVM_DIR/versions/node/$(nvm version)/bin/node" "/usr/local/bin/node"
sudo ln -s "$NVM_DIR/versions/node/$(nvm version)/bin/npm" "/usr/local/bin/npm"
sudo ln -s "$NVM_DIR/versions/node/$(nvm version)/bin/pm2" "/usr/local/bin/pm2"
```

### Add Repository

Clone your GitHub repository and navigate to its directory:

```bash
git clone https://github.com/CivicDataLab/opub-deploy.git
cd opub-deploy
```

Install the dependencies and start the server:

```bash
npm install
npm run build
pm2 start npm --name opub-deploy -- run start -- -p 3000
```

## Adding SSH Key to GitHub

1. Generate a SSH key:

```bash
ssh-keygen -m PEMs
```

2. Move the public key to authorized_keys:

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

3. Copy the private key content:

```bash
cat ~/.ssh/id_rsa
// Copy the content manually
```

4. In GitHub, navigate to Repository > Settings > Secrets and variables > Actions

5. Create the following three secrets to allow GitHub to SSH into EC2:

   - `EC2_HOST`: Public IP of your EC2 instance

   - `EC2_USERNAME`: Username to SSH into EC2

   - `EC2_PRIVATE_KEY`: Paste the private key content generated earlier.

## Conclusion

Now, every time you push to the main branch, the GitHub Actions workflow will build the project and deploy it to the EC2 instance without any manual intervention and downtime.
