# mayohr-auto-punch

[![npm version](https://img.shields.io/npm/v/mayohr-auto-punch.svg)](https://www.npmjs.com/package/mayohr-auto-punch)
[![npm downloads](https://img.shields.io/npm/dm/mayohr-auto-punch.svg)](https://www.npmjs.com/package/mayohr-auto-punch)
[![Docker Pulls](https://img.shields.io/docker/pulls/justintw/mayohr-auto-punch.svg)](https://hub.docker.com/r/justintw/mayohr-auto-punch)
[![Docker Image Size](https://img.shields.io/docker/image-size/justintw/mayohr-auto-punch)](https://hub.docker.com/r/justintw/mayohr-auto-punch)
[![Docker Stars](https://img.shields.io/docker/stars/justintw/mayohr-auto-punch.svg)](https://hub.docker.com/r/justintw/mayohr-auto-punch)


An automated solution for handling punch in/out operations in MayoHR system. This tool helps streamline your daily attendance tracking by automating the punch process.


## Requirements

- Node.js 18 or higher

## Quick Start

### Environment Setup

First, create and configure your environment file:

```bash
# Download the .env.example file:
mkdir -p ~/.mayohr-auto-punch && \
  wget -O ~/.mayohr-auto-punch/.env \
  https://raw.githubusercontent.com/awesome-oa-tools/mayohr-auto-punch/main/.env.example

# Configure your credentials
vi ~/.mayohr-auto-punch/.env
```

#### Environment Variables Configuration

The .env file requires the following configurations:

```bash
# Browser Mode Configuration
HEADLESS="true"                    # Run in headless mode (recommended for production)

# Microsoft Account Settings
MS_DOMAIN="your_company_domain"    # Your company's domain (e.g., company.com)
MS_USERNAME="your_username"        # Your Microsoft account username (e.g., bob@company.com)
MS_PASSWORD="your_password"        # Your Microsoft account password (e.g., password123)
MS_TOPT_SECRET="otpauth://..."     # Your Microsoft Authenticator TOTP secret

# Telegram Notification Settings (Optional)
TELEGRAM_ENABLED="false"           # Set to "true" to enable Telegram notifications
TELEGRAM_BOT_TOKEN="bot_token"     # Your Telegram bot token
TELEGRAM_CHAT_ID="chat_id"         # Your Telegram chat ID
```

> Note: For MS_TOPT_SECRET, you'll need to extract the TOTP secret from your Microsoft Authenticator QR code. This is typically in the format of "otpauth://...".

## Usage Options

### Option 1: Command Line Interface

Run the script directly using npx:

```bash
npx --yes --quite mayohr-auto-punch@latest
```

Run the script directly using docker:

```bash
docker run --rm -it \
  -v $HOME/.mayohr-auto-punch:/home/pptruser/.mayohr-auto-punch \
  justintw/mayohr-auto-punch:latest
```

### Option 2: Programmatic Usage with Node.js

You can also use the package programmatically in your Node.js applications:

1. Install the package:

```bash
npm install mayohr-auto-punch -S
```

2. Use in your code:

```javascript
import { MayohrService } from 'mayohr-auto-punch';

async function main() {
  // Create MayohrService instance
  const mayohrService = new MayohrService(
    true, // headless mode
    'your-domain.com',
    'your-username@your-domain.com',
    'your-password',
    'your-totp-secret'
  );

  try {
    // Initialize browser
    await mayohrService.init();

    // Login
    const isLoggedIn = await mayohrService.login();
    if (!isLoggedIn) {
      throw new Error('Login failed');
    }

    // Punch
    const isPunched = await mayohrService.punch();
    if (!isPunched) {
      throw new Error('Punch failed');
    } else {
      console.log('Punch successful');
    }
  } catch (error) {
    console.error('Error:', error);
  } finally {
    // Close browser
    await mayohrService.close();
  }
}

main();
```

## Deployment Options

### Option 1: Crontab Setup (Recommended for Linux/macOS)

Set up automated scheduling using crontab:

```bash
# Set up logging
sudo touch /var/log/mayohr-auto-punch.log && \
  sudo chown $USER /var/log/mayohr-auto-punch.log && \
  sudo chmod 644 /var/log/mayohr-auto-punch.log

# Download and configure crontab
mkdir -p ~/.mayohr-auto-punch/cron && \
  wget -O ~/.mayohr-auto-punch/cron/mayohr \
  https://raw.githubusercontent.com/awesome-oa-tools/mayohr-auto-punch/main/examples/cron/mayohr

# Install crontab configuration
crontab -l | cat - ~/.mayohr-auto-punch/cron/mayohr | crontab -

# Verify installation
crontab -l

# To remove the crontab configuration later, use:
# crontab -e
```

### Option 2: AWS Lambda Deployment

> ⚠️ Warning: Deploying to AWS Lambda may trigger security alerts due to foreign IP addresses. Use with caution.

```bash
# Download AWS CloudFormation templates
mkdir -p ~/.mayohr-auto-punch/aws && \
  wget -O ~/.mayohr-auto-punch/aws/ecr-template.yaml \
  https://raw.githubusercontent.com/awesome-oa-tools/mayohr-auto-punch/main/examples/aws/ecr-template.yaml && \
  wget -O ~/.mayohr-auto-punch/aws/lambda-template.yaml \
  https://raw.githubusercontent.com/awesome-oa-tools/mayohr-auto-punch/main/examples/aws/lambda-template.yaml

# Load environment variables
source ~/.mayohr-auto-punch/.env

# Configure AWS SSM parameters
aws ssm put-parameter \
  --no-cli-pager \
  --region ap-northeast-1 \
  --name "/mayohr-auto-punch/ms-password" \
  --value "${MS_PASSWORD}" \
  --type "SecureString" \
  --overwrite

aws ssm put-parameter \
  --no-cli-pager \
  --region ap-northeast-1 \
  --name "/mayohr-auto-punch/ms-totp-secret" \
  --value "${MS_TOPT_SECRET}" \
  --type "SecureString" \
  --overwrite

aws ssm put-parameter \
  --no-cli-pager \
  --region ap-northeast-1 \
  --name "/mayohr-auto-punch/telegram-bot-token" \
  --value "${TELEGRAM_BOT_TOKEN}" \
  --type "SecureString" \
  --overwrite

# Create and configure ECR repository
aws cloudformation create-stack \
  --no-cli-pager \
  --region ap-northeast-1 \
  --stack-name mayohr-auto-punch-ecr \
  --template-body file://~/.mayohr-auto-punch/aws/ecr-template.yaml

# Get ECR repository URI
ECR_URI=$(aws cloudformation describe-stacks \
  --no-cli-pager \
  --region ap-northeast-1 \
  --stack-name mayohr-auto-punch-ecr \
  --query "Stacks[0].Outputs[?OutputKey=='RepositoryUri'].OutputValue" \
  --output text)

# Authenticate with ECR
aws ecr get-login-password \
  --no-cli-pager \
  --region ap-northeast-1 \
  | docker login \
    --username AWS \
    --password-stdin \
    ${ECR_URI}

# Push Docker image to ECR
docker pull --platform linux/amd64 justintw/mayohr-auto-punch:latest && \
  docker tag justintw/mayohr-auto-punch:latest ${ECR_URI}:latest && \
  docker push ${ECR_URI}:latest

# Deploy Lambda function
aws cloudformation create-stack \
  --no-cli-pager \
  --region ap-northeast-1 \
  --stack-name mayohr-auto-punch-lambda \
  --template-body file://~/.mayohr-auto-punch/aws/lambda-template.yaml \
  --parameters \
    ParameterKey=ImageUri,ParameterValue=${ECR_URI}:latest \
    ParameterKey=MsDomain,ParameterValue=${MS_DOMAIN} \
    ParameterKey=MsUsername,ParameterValue=${MS_USERNAME} \
    ParameterKey=TelegramEnabled,ParameterValue=${TELEGRAM_ENABLED} \
    ParameterKey=TelegramChatId,ParameterValue=${TELEGRAM_CHAT_ID} \
  --capabilities CAPABILITY_IAM
```

## Notifications

### Telegram Notification (Optional)

Enable Telegram notifications by following these steps:

1. Create a new bot via [@BotFather](https://t.me/botfather)
2. Create a Telegram channel and add your bot as an administrator
3. Obtain your Channel ID by visiting: https://api.telegram.org/bot<bot_token>/getUpdates
3. Configure your `~/.mayohr-auto-punch/.env` file with:

```bash
TELEGRAM_ENABLED="true"
TELEGRAM_BOT_TOKEN="your_bot_token"
TELEGRAM_CHAT_ID="your_chat_id"
```
