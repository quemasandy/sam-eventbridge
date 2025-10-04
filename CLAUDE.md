# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an AWS SAM demo showcasing event-driven microservices using AWS EventBridge. The architecture demonstrates how to build decoupled services where a central order manager publishes events and multiple restaurant services consume them based on EventBridge rules.

## Architecture

**Microservices:**
- `orderManager/`: Central API service that receives orders via REST API (with API Key auth) and publishes events to EventBridge
- `pizzaHat/`: Restaurant service demonstrating CloudFormation-native EventBridge rule definition
- `ThaiLand/`: Restaurant service demonstrating SAM-native EventBridge rule definition
- `blueDragon/`: Restaurant service using EventBridge API Destinations (calls 3rd party API)

**Key Design Pattern:** Each restaurant service owns its EventBridge rule configuration, maintaining decoupling. Services filter events by `detail.restaurantName` field.

## Event Schema

Events published from `orderManager/handler.js` follow this structure:
```javascript
{
  Source: 'custom.orderManager',
  DetailType: 'order',
  Detail: {
    restaurantName: string,  // Routes to specific restaurant
    order: object,
    customerName: string,
    amount: number
  }
}
```

## EventBridge Rule Patterns

**SAM-native approach** (`ThaiLand/template.yml`):
```yaml
Events:
  Trigger:
    Type: CloudWatchEvent
    Properties:
      Pattern:
        source: [custom.orderManager]
        detail-type: [order]
        detail:
          restaurantName: ["thaiLand"]
```

**CloudFormation approach** (`pizzaHat/template.yml`):
```yaml
MyEventsRule:
  Type: AWS::Events::Rule
  Properties:
    EventPattern: {...}
    Targets:
      - Arn: !GetAtt PizzaHatFunction.Arn
```

**API Destinations** (`blueDragon/template.yml`):
Uses `AWS::Events::Connection` and `AWS::Events::ApiDestination` to invoke external HTTP endpoints directly from EventBridge rules.

## Development Commands

### Deploy Services
Each microservice must be deployed independently in this order:
1. Deploy orderManager first (provides EventBridge bus)
2. Deploy restaurant services (pizzaHat, ThaiLand, blueDragon)

```bash
cd <service-directory>
sam deploy -g  # First deployment (guided mode, sets up samconfig.toml)
sam deploy     # Subsequent deployments
```

### Delete Services
```bash
cd <service-directory>
sam delete
```

### Testing
1. After deploying orderManager, retrieve API endpoint and API Key from stack outputs
2. Make POST requests to `/order` endpoint with API Key header
3. Check CloudWatch Logs for each restaurant Lambda to verify event delivery

## Configuration Files

- `samconfig.toml`: Per-service deployment configuration (region, stack name, S3 prefix)
  - Default region: `us-east-2`
  - Each service has its own stack name matching directory name
- `template.yml`: SAM/CloudFormation infrastructure definition per service

## Important Notes

- API Key authentication is required for orderManager endpoints (defined in `orderManager/template.yml`)
- The orderManager uses `ServerlessRestApiProdStage` reference for proper API Gateway stage linking
- EventBridge permissions are granted via IAM policies in each template
- Node.js runtime: `nodejs18.x`
- Only orderManager has npm dependencies (`aws-sdk`)

## Branch Structure

- `main`: Basic EventBridge integration
- `api-destinations`: Adds blueDragon service with API Destinations pattern
- `video3`: Adds Dead Letter Queue (DLQ) pattern for failed event deliveries
