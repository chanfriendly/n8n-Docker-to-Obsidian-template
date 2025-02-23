# Docker-to-Obsidian Automation

![image](https://github.com/user-attachments/assets/195763cb-4e0d-49ee-9408-02aba33aa511)

Automatically document Docker stack deployments in Obsidian using n8n automation.

## Overview
This project automatically creates and updates Obsidian notes whenever a Docker stack is deployed, providing automatic documentation of your infrastructure.

## Prerequisites
- A system running Docker (Tested on TrueNAS with Portainer)
- n8n instance
- Obsidian with Local REST API plugin
- Docker Compose for stack deployments
- Optional: AI Agent for enhanced documentation processing
  - Can use a local small model or an API from a GPT provider
  - Google Gemini's free tier can be used as of 02/23/2025
  - Alternative: Interested contributors could help develop a non-AI version

## Components
- **n8n**: Handles automation workflow and webhook processing
- **Obsidian**: Document storage with REST API enabled
- **Docker Monitor**: Container that watches for Docker events
- **Docker Socket**: Used to capture Docker events
- **AI Agent**: Transforms raw data into readable documentation (Optional)

## Setup
### 1. Configure Obsidian Local REST API Plugin
- Enable the plugin
- Save the API key
- Enable HTTP port (default: 27123)
  - Note: HTTPS can be used, but may require additional configuration
  - Local network deployment minimizes security concerns

### 2. Configure n8n
- Download and install n8n
- Create webhook and workflow as per project documentation

### 3. Deploy Docker Monitor
Compose file template (replace placeholders):
   
```version: '3'
version: '3'
services:
  portainer-monitor:
    image: alpine
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - PORTAINER_URL=[YOUR URL]
      - PORTAINER_API_KEY=[YOUR PORTAINER API KEY]
      - N8N_WEBHOOK_URL=[YOUR URL]/webhook/docker-stack
      - WEBHOOK_SECRET=[RANDOMLY GENERATED KEY]
    command: |
      sh -c '
      # Install required packages
      apk add --no-cache curl jq bash ca-certificates
      
      echo "Starting Portainer monitor..."
      
      # Test n8n webhook connectivity
      echo "Testing n8n webhook..."
      curl -v -k -X POST "$$N8N_WEBHOOK_URL" \
        -H "Content-Type: application/json" \
        -H "X-Webhook-Secret: $$WEBHOOK_SECRET" \
        -d "{\"test\": true}"
      
      # Function to get stack update times
      get_stack_updates() {
        curl -s -k -H "X-API-Key: $$PORTAINER_API_KEY" "$$PORTAINER_URL/api/stacks" | \
        jq -c "map({name: .Name, update: .UpdateDate, creation: .CreationDate})"
      }
      
      echo "Getting initial state..."
      previous_state=$(get_stack_updates)
      echo "Initial state: $$previous_state"
      
      while true; do
        echo "Checking for changes..."
        current_state=$(get_stack_updates)
        
        if [ "$$current_state" != "$$previous_state" ]; then
          echo "Change detected!"
          echo "Previous: $$previous_state"
          echo "Current: $$current_state"
          
          # Get full stack info for changed stacks
          changed_stacks=$(curl -s -k -H "X-API-Key: $$PORTAINER_API_KEY" "$$PORTAINER_URL/api/stacks")
          
          echo "Sending webhook..."
          curl -v -k -X POST "$$N8N_WEBHOOK_URL" \
            -H "Content-Type: application/json" \
            -H "X-Webhook-Secret: $$WEBHOOK_SECRET" \
            -d "{
              \"event_type\": \"stack_change\",
              \"stacks\": $$changed_stacks
            }"
            
          previous_state="$$current_state"
        else
          echo "No changes detected"
        fi
        
        sleep 5
      done
      '
```

I also strongly recommend making a test container for troubleshooting. I used this:

```
version: '3'
services:
  web:
    image: nginx:alpine
    ports:
      - "8089:80"
    environment:
      - NGINX_HOST=test.local
      - NGINX_PORT=80
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      replicas: 1
      labels:
        - "test.purpose=webhook_validation"
        - "test.description=This is a test stack for verifying n8n webhooks"
```

### 4. Add Template

- Download template from repository
- Add to new workflow using "Import from File"

### 5. Add Credentials
You will need:

- Portainer API key
- Webhook secret
- AI API credentials (optional)
- Obsidian API key

## Configuration Details
### Obsidian Setup

Enable Local REST API plugin
Configure HTTP port (27123)
Save API key for authentication

### n8n Webhook Configuration

Method: PUT
URL Format: http://[your_ip]:27123/vault/[path]
Headers:

Authorization: Bearer [API_KEY]
Content-Type: text/markdown



## Troubleshooting

Ensure all components are on the same network
Verify API keys and credentials
Check n8n and Docker monitor logs for any errors

## Intended Future Support
- Add non-AI documentation generation option
- Improve error handling
- Create more comprehensive documentation


## License
MIT License
