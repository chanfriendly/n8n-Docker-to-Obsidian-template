# Docker-to-Obsidian Automation

Automatically document Docker stack deployments in Obsidian using n8n automation.

## Overview
This project automatically creates and updates Obsidian notes whenever a Docker stack is deployed, providing automatic documentation of your infrastructure.

## Prerequisites
- A system running Docker (The machine using this is running TrueNAS and Portainer)
- n8n instance
- Obsidian with Local REST API plugin
- Docker Compose for stack deployments

## Components
- **n8n**: Handles automation workflow and webhook processing
- **Obsidian**: Document storage with REST API enabled
- **Docker Monitor**: Container that watches for Docker events
- **Docker Socket**: Used to capture Docker events

## Setup
1. Configure Obsidian Local REST API plugin
   - Enable the plugin
   - Save the API key
   - Turn on the HTTP port (default: 27123)
     - You can do this with HTTPS, but I ran into SSL issues. I'm also not exposing it to the internet, so I'm less concerned about security. 

2. Configure n8n
   - Set up webhook endpoint
   - Configure HTTP request node with proper headers
   - Add authorization details

3. Deploy Docker monitor
   Compose file I used below:
   ```
version: '3'
services:
  docker-monitor:
    image: alpine
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    entrypoint: >
      sh -c '
        apk add --no-cache curl jq bash;
        while true; do
          docker events --filter "Type=service" --filter "event=create" --format "{{json .}}" |
          while read event; do
            stack_name=$(echo "$event" | jq -r ".Actor.Attributes.name" | cut -d_ -f1);
            compose_content=$(curl -s --unix-socket /var/run/docker.sock http://localhost/containers/json | jq -r ".[] | select(.Labels.\"com.docker.stack.namespace\"==\"$stack_name\")");
            if [ ! -z "$compose_content" ]; then
              curl -X POST "http://n8n:5678/webhook/docker-stack" \
                -H "Content-Type: application/json" \
                -d "{
                  \"stack_name\": \"${stack_name}\",
                  \"compose_content\": ${compose_content}
                }";
            fi;
          done;
        done
      '
    restart: unless-stopped
    networks:
      - n8n_default

networks:
  n8n_default:
    external: true
```
    
## Configuration
### Obsidian Setup
- Enable Local REST API plugin
- Configure HTTP port (27123)
- Save API key for authentication

### n8n Webhook Configuration
- Method: PUT
- URL Format: `http://[your_ip]:27123/vault/[path]`
- Headers:
  - Authorization: Bearer [API_KEY]
  - Content-Type: text/markdown

## Current Status
- Basic automation pipeline established
- Successfully creating notes in Obsidian
- Webhook communication verified
- Initial template structure created

## Next Steps
- [ ] Implement full Docker event handling
- [ ] Implement note templates
- [ ] Implement JSON file path and names
- [ ] Add error handling
- [ ] Create backup procedures
- [ ] Add additional detail and screenshots to README
