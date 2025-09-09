# AWS Lambda Cheat Sheet

Comprehensive guide for AWS Lambda function development, triggers, layers, performance optimization, and monitoring. Covers production-ready patterns for serverless computing on AWS.

---

## Table of Contents
- [Basic Concepts](#basic-concepts)
- [Function Development](#function-development)
- [Triggers & Events](#triggers--events)
- [Layers & Dependencies](#layers--dependencies)
- [Performance Optimization](#performance-optimization)
- [Monitoring & Debugging](#monitoring--debugging)
- [Security Best Practices](#security-best-practices)
- [Cost Optimization](#cost-optimization)
- [Deployment Strategies](#deployment-strategies)
- [Production Patterns](#production-patterns)

---

## Basic Concepts

| Concept | Description | Example |
|---------|-------------|---------|
| Handler | Entry point for Lambda function | `exports.handler = async (event) => {}` |
| Event | Input data passed to function | API Gateway request, S3 event, etc. |
| Context | Runtime information object | Timeout, memory, request ID |
| Runtime | Execution environment | Node.js 18.x, Python 3.9, Java 11 |
| Cold Start | First invocation after deployment | ~100ms-5s initialization time |
| Warm Start | Subsequent invocations | ~1-10ms startup time |

## Function Development

### Basic Handler Patterns
```javascript
// Simple async handler
exports.handler = async (event, context) => {
  console.log('Event:', JSON.stringify(event, null, 2));
  console.log('Context:', JSON.stringify(context, null, 2));
  
  return {
    statusCode: 200,
    body: JSON.stringify({
      message: 'Hello from Lambda!',
      requestId: context.awsRequestId
    })
  };
};

// API Gateway handler
exports.apiHandler = async (event) => {
  const { httpMethod, path, body, headers } = event;
  
  try {
    const requestBody = body ? JSON.parse(body) : {};
    
    switch (httpMethod) {
      case 'GET':
        return await handleGet(event.pathParameters);
      case 'POST':
        return await handlePost(requestBody);
      default:
        return {
          statusCode: 405,
          body: JSON.stringify({ error: 'Method not allowed' })
        };
    }
  } catch (error) {
    return {
      statusCode: 500,
      body: JSON.stringify({ error: error.message })
    };
  }
};

// DynamoDB Stream handler
exports.streamHandler = async (event) => {
  for (const record of event.Records) {
    console.log('Event Name:', record.eventName);
    console.log('DynamoDB Record:', JSON.stringify(record.dynamodb, null, 2));
    
    switch (record.eventName) {
      case 'INSERT':
        await handleInsert(record.dynamodb.NewImage);
        break;
      case 'MODIFY':
        await handleUpdate(record.dynamodb.NewImage, record.dynamodb.OldImage);
        break;
      case 'REMOVE':
        await handleDelete(record.dynamodb.OldImage);
        break;
    }
  }
};
```

### Error Handling
```javascript
class LambdaError extends Error {
  constructor(message, statusCode = 500) {
    super(message);
    this.statusCode = statusCode;
    this.name = 'LambdaError';
  }
}

const errorHandler = (error) => {
  console.error('Lambda Error:', error);
  
  // Log to CloudWatch
  console.error({
    error: error.message,
    stack: error.stack,
    timestamp: new Date().toISOString()
  });
  
  // Return appropriate response
  return {
    statusCode: error.statusCode || 500,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*'
    },
    body: JSON.stringify({
      error: error.message,
      type: error.name
    })
  };
};

exports.handler = async (event) => {
  try {
    // Business logic
    const result = await processEvent(event);
    return {
      statusCode: 200,
      body: JSON.stringify(result)
    };
  } catch (error) {
    return errorHandler(error);
  }
};
```

### Environment Variables
```javascript
// Reading environment variables
const config = {
  tableName: process.env.TABLE_NAME,
  apiKey: process.env.API_KEY,
  stage: process.env.STAGE || 'dev',
  logLevel: process.env.LOG_LEVEL || 'info'
};

// Validation
const validateConfig = () => {
  const required = ['TABLE_NAME', 'API_KEY'];
  const missing = required.filter(key => !process.env[key]);
  
  if (missing.length > 0) {
    throw new Error(`Missing environment variables: ${missing.join(', ')}`);
  }
};

exports.handler = async (event) => {
  validateConfig();
  // Function logic
};
```

## Triggers & Events

### API Gateway Events
```javascript
// REST API Event
{
  "resource": "/users/{id}",
  "path": "/users/123",
  "httpMethod": "GET",
  "headers": {
    "Accept": "application/json",
    "Authorization": "Bearer token123"
  },
  "pathParameters": {
    "id": "123"
  },
  "queryStringParameters": {
    "include": "profile"
  },
  "body": null,
  "requestContext": {
    "requestId": "c6af9ac6-7b61-11e6-9a41-93e8deadbeef",
    "stage": "prod",
    "apiId": "1234567890"
  }
}

// HTTP API Event (v2.0)
{
  "version": "2.0",
  "routeKey": "GET /users/{id}",
  "rawPath": "/users/123",
  "headers": {
    "authorization": "Bearer token123"
  },
  "pathParameters": {
    "id": "123"
  },
  "requestContext": {
    "http": {
      "method": "GET",
      "path": "/users/123"
    }
  }
}
```

### S3 Events
```javascript
exports.s3Handler = async (event) => {
  for (const record of event.Records) {
    const bucket = record.s3.bucket.name;
    const key = decodeURIComponent(record.s3.object.key.replace(/\+/g, ' '));
    const eventName = record.eventName;
    
    console.log(`S3 Event: ${eventName} on ${bucket}/${key}`);
    
    switch (eventName) {
      case 's3:ObjectCreated:*':
        await processNewFile(bucket, key);
        break;
      case 's3:ObjectRemoved:*':
        await handleFileRemoval(bucket, key);
        break;
    }
  }
};
```

### CloudWatch Events
```javascript
// Scheduled events
exports.scheduledHandler = async (event) => {
  console.log('Scheduled event:', JSON.stringify(event, null, 2));
  
  // Cron job logic
  await performScheduledTask();
  
  return {
    statusCode: 200,
    body: JSON.stringify({
      message: 'Scheduled task completed',
      timestamp: new Date().toISOString()
    })
  };
};
```

### SQS Events
```javascript
exports.sqsHandler = async (event) => {
  const failures = [];
  
  for (const record of event.Records) {
    try {
      const body = JSON.parse(record.body);
      await processMessage(body);
    } catch (error) {
      console.error('Failed to process message:', error);
      failures.push(record.messageId);
    }
  }
  
  // Return partial batch failure
  if (failures.length > 0) {
    return {
      batchItemFailures: failures.map(id => ({ itemIdentifier: id }))
    };
  }
};
```

## Layers & Dependencies

### Creating Layers
```bash
# Create layer structure
mkdir -p layer/nodejs/node_modules
cd layer/nodejs
npm init -y
npm install axios moment

# Package layer
cd ../..
zip -r layer.zip layer/

# Deploy with AWS CLI
aws lambda publish-layer-version \
  --layer-name my-utilities \
  --zip-file fileb://layer.zip \
  --compatible-runtimes nodejs18.x
```

### Using Layers
```yaml
# SAM template
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: index.handler
      Runtime: nodejs18.x
      Layers:
        - !Ref MyLayer
        - arn:aws:lambda:us-east-1:580247275435:layer:LambdaInsightsExtension:21

  MyLayer:
    Type: AWS::Serverless::LayerVersion
    Properties:
      LayerName: MyUtilities
      ContentUri: layer/
      CompatibleRuntimes:
        - nodejs18.x
```

### Dependency Management
```javascript
// Package.json for lightweight functions
{
  "name": "lambda-function",
  "version": "1.0.0",
  "dependencies": {
    "aws-sdk": "*"  // Available in Lambda runtime
  },
  "devDependencies": {
    "webpack": "^5.0.0",
    "webpack-cli": "^4.0.0"
  }
}

// Webpack config for bundling
module.exports = {
  target: 'node',
  mode: 'production',
  entry: './index.js',
  output: {
    filename: 'index.js',
    libraryTarget: 'commonjs2'
  },
  externals: {
    'aws-sdk': 'aws-sdk'  // Exclude from bundle
  }
};
```

## Performance Optimization

### Memory and CPU Allocation
```yaml
# SAM template configuration
Resources:
  FastFunction:
    Type: AWS::Serverless::Function
    Properties:
      MemorySize: 512      # More memory = more CPU
      Timeout: 30          # Reasonable timeout
      ReservedConcurrencyLimit: 10
      
  CpuIntensiveFunction:
    Type: AWS::Serverless::Function
    Properties:
      MemorySize: 3008     # Maximum memory for max CPU
      Timeout: 900         # 15 minutes max
```

### Connection Reuse
```javascript
// AWS SDK v2 - Connection reuse
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient({
  httpOptions: {
    keepAlive: true,
    keepAliveMsecs: 1000
  }
});

// AWS SDK v3 - Connection reuse
const { DynamoDBClient } = require('@aws-sdk/client-dynamodb');
const { DynamoDBDocumentClient } = require('@aws-sdk/lib-dynamodb');

const client = new DynamoDBClient({
  requestHandler: {
    keepAlive: true
  }
});
const docClient = DynamoDBDocumentClient.from(client);

exports.handler = async (event) => {
  // Reuse connection across invocations
  const result = await docClient.get({
    TableName: 'MyTable',
    Key: { id: event.id }
  });
  
  return result.Item;
};
```

### Cold Start Optimization
```javascript
// Initialize outside handler
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient();

// Pre-compile regular expressions
const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

// Cache configuration
let config = null;
const getConfig = async () => {
  if (!config) {
    config = await loadConfiguration();
  }
  return config;
};

exports.handler = async (event) => {
  const config = await getConfig();
  // Handler logic with cached config
};
```

### Provisioned Concurrency
```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      ProvisionedConcurrencyConfig:
        ProvisionedConcurrencyLevel: 10
      AutoPublishAlias: live
      DeploymentPreference:
        Type: Linear50PercentEvery30Seconds
```

## Monitoring & Debugging

### CloudWatch Logging
```javascript
// Structured logging
const logger = {
  info: (message, meta = {}) => {
    console.log(JSON.stringify({
      level: 'INFO',
      message,
      timestamp: new Date().toISOString(),
      requestId: context.awsRequestId,
      ...meta
    }));
  },
  
  error: (message, error, meta = {}) => {
    console.error(JSON.stringify({
      level: 'ERROR',
      message,
      error: {
        name: error.name,
        message: error.message,
        stack: error.stack
      },
      timestamp: new Date().toISOString(),
      requestId: context.awsRequestId,
      ...meta
    }));
  }
};

exports.handler = async (event, context) => {
  logger.info('Function started', { event });
  
  try {
    const result = await processEvent(event);
    logger.info('Function completed successfully', { result });
    return result;
  } catch (error) {
    logger.error('Function failed', error, { event });
    throw error;
  }
};
```

### Custom Metrics
```javascript
const AWS = require('aws-sdk');
const cloudwatch = new AWS.CloudWatch();

const putMetric = async (metricName, value, unit = 'Count') => {
  await cloudwatch.putMetricData({
    Namespace: 'MyApp/Lambda',
    MetricData: [{
      MetricName: metricName,
      Value: value,
      Unit: unit,
      Timestamp: new Date(),
      Dimensions: [
        {
          Name: 'FunctionName',
          Value: process.env.AWS_LAMBDA_FUNCTION_NAME
        }
      ]
    }]
  }).promise();
};

exports.handler = async (event) => {
  const startTime = Date.now();
  
  try {
    const result = await processEvent(event);
    
    // Success metrics
    await putMetric('ProcessedItems', result.count);
    await putMetric('ProcessingTime', Date.now() - startTime, 'Milliseconds');
    
    return result;
  } catch (error) {
    // Error metrics
    await putMetric('Errors', 1);
    throw error;
  }
};
```

### X-Ray Tracing
```javascript
const AWSXRay = require('aws-xray-sdk-core');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));

exports.handler = async (event) => {
  const segment = AWSXRay.getSegment();
  
  // Create subsegment
  const subsegment = segment.addNewSubsegment('database-query');
  
  try {
    const result = await dynamodb.get({
      TableName: 'MyTable',
      Key: { id: event.id }
    }).promise();
    
    subsegment.close();
    return result.Item;
  } catch (error) {
    subsegment.close(error);
    throw error;
  }
};
```

## Security Best Practices

### IAM Roles and Policies
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:region:account:table/MyTable"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:region:account:key/key-id"
    }
  ]
}
```

### Environment Variable Encryption
```javascript
const AWS = require('aws-sdk');
const kms = new AWS.KMS();

let decryptedSecrets = {};

const decrypt = async (encryptedValue) => {
  if (decryptedSecrets[encryptedValue]) {
    return decryptedSecrets[encryptedValue];
  }
  
  const result = await kms.decrypt({
    CiphertextBlob: Buffer.from(encryptedValue, 'base64')
  }).promise();
  
  const decrypted = result.Plaintext.toString();
  decryptedSecrets[encryptedValue] = decrypted;
  
  return decrypted;
};

exports.handler = async (event) => {
  const apiKey = await decrypt(process.env.ENCRYPTED_API_KEY);
  // Use decrypted API key
};
```

### VPC Configuration
```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      VpcConfig:
        SecurityGroupIds:
          - sg-12345678
        SubnetIds:
          - subnet-12345678
          - subnet-87654321
      Environment:
        Variables:
          RDS_ENDPOINT: !GetAtt MyDatabase.Endpoint.Address
```

## Cost Optimization

### Right-sizing Functions
```javascript
// Memory optimization based on function type
const configs = {
  // I/O intensive - low memory
  ioIntensive: {
    memorySize: 128,
    timeout: 30
  },
  
  // CPU intensive - high memory
  cpuIntensive: {
    memorySize: 3008,
    timeout: 900
  },
  
  // Balanced workload
  balanced: {
    memorySize: 512,
    timeout: 60
  }
};
```

### Reserved Concurrency
```yaml
Resources:
  CriticalFunction:
    Type: AWS::Serverless::Function
    Properties:
      ReservedConcurrencyLimit: 100  # Reserve capacity
      
  BackgroundTask:
    Type: AWS::Serverless::Function
    Properties:
      ReservedConcurrencyLimit: 5    # Limit concurrent executions
```

### Cost Monitoring
```javascript
// Cost tracking with tags
const AWS = require('aws-sdk');
const pricing = new AWS.Pricing({ region: 'us-east-1' });

const calculateCost = (memoryMB, durationMs) => {
  const gbSeconds = (memoryMB / 1024) * (durationMs / 1000);
  const pricePerGbSecond = 0.0000166667; // Current pricing
  return gbSeconds * pricePerGbSecond;
};

exports.handler = async (event, context) => {
  const startTime = Date.now();
  
  try {
    const result = await processEvent(event);
    
    const duration = Date.now() - startTime;
    const cost = calculateCost(context.memoryLimitInMB, duration);
    
    console.log(`Function cost: $${cost.toFixed(6)}`);
    
    return result;
  } catch (error) {
    throw error;
  }
};
```

## Deployment Strategies

### Blue/Green Deployment
```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      AutoPublishAlias: live
      DeploymentPreference:
        Type: Linear10PercentEvery1Minute
        Hooks:
          PreTraffic: !Ref PreTrafficHook
          PostTraffic: !Ref PostTrafficHook
        Alarms:
          - !Ref AliasErrorMetricGreaterThanZeroAlarm
          
  PreTrafficHook:
    Type: AWS::Serverless::Function
    Properties:
      Policies:
        - Version: "2012-10-17"
          Statement:
            - Effect: "Allow"
              Action:
                - "codedeploy:PutLifecycleEventHookExecutionStatus"
              Resource: "*"
```

### Canary Testing
```javascript
// Version-aware handler
exports.handler = async (event, context) => {
  const version = context.functionVersion;
  const isCanary = version !== '$LATEST';
  
  if (isCanary) {
    // Additional monitoring for canary
    await recordCanaryMetrics(event);
  }
  
  try {
    const result = await processEvent(event);
    
    if (isCanary) {
      await validateCanaryResult(result);
    }
    
    return result;
  } catch (error) {
    if (isCanary) {
      await recordCanaryError(error);
    }
    throw error;
  }
};
```

## Production Patterns

### Circuit Breaker Pattern
```javascript
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000) {
    this.threshold = threshold;
    this.timeout = timeout;
    this.failures = 0;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.nextAttempt = 0;
  }
  
  async call(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failures = 0;
    this.state = 'CLOSED';
  }
  
  onFailure() {
    this.failures++;
    if (this.failures >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}

const circuitBreaker = new CircuitBreaker();

exports.handler = async (event) => {
  try {
    return await circuitBreaker.call(() => callExternalAPI(event));
  } catch (error) {
    return fallbackResponse(error);
  }
};
```

### Retry Pattern
```javascript
const retry = async (fn, maxAttempts = 3, baseDelay = 1000) => {
  let lastError;
  
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;
      
      if (attempt === maxAttempts) {
        throw error;
      }
      
      // Exponential backoff
      const delay = baseDelay * Math.pow(2, attempt - 1);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
};

exports.handler = async (event) => {
  return await retry(() => processEvent(event), 3, 1000);
};
```

---

## Resources
- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [Serverless Application Lens](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/)
- [AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)

---
*Originally compiled from various sources. Contributions welcome!*