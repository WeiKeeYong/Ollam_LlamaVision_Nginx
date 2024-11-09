# Ollama Setup Guide with brev.dev and NGINX

A quick guide for setting up Ollama on a GPU-optimized VM using brev.dev and configuring NGINX to add in authentication when exposed to public.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Setting Up Ollama](#setting-up-ollama)
- [NGINX Token Authentication](#nginx-configuration)
  - [Basic Bearer Token Authentication](#basic-bearer-token-authentication)
  - [Client Integration](#client-integration)
    - [Python Client](#python-client)
    - [cURL Examples](#curl-examples)
- [NGINX Basic Authentication](#NGINX-with-Basic-HTTP-Authentication)
- [Monitoring](#monitoring)
- [Additional Resources](#additional-resources)

## Prerequisites

- **brev.dev Account**: Paid subscription required for GPU access
- **Technical Knowledge**: Basic understanding of Linux commands and NGINX configuration
- **Baisc Dev Knowledge**: Some python skill will be great help
- **Tools**: Access to terminal/command line interface

## Setting Up Ollama
I use brev.dev to help configure all the necessary drivers and tools. It also help me create vm with the GPU driver and CUDA I need. Once you have a paid subscription, you can creaet gpu via brev.dev and use below command in to enable ollama for public access. 
1. **Connect to Server**  
Download brev CLI here: https://www.brev.dev/docs/how-to/install-cli
   ```bash
   brev shell --host <server name>
   ```

3. **Install and Configure Ollama**
   ```bash
   # Install Ollama
   curl -fsSL https://ollama.com/install.sh | sh

   # Pull required model
   ollama pull llama3.2-vision

   # Configure Ollama to accept external connections
   sudo systemctl edit ollama.service
   ```

   Add the following configuration:
   ```ini
   [Service]
   Environment="OLLAMA_HOST=0.0.0.0"
   ```

4. **Restart Ollama Service**
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart ollama
   ```

5. **Configure brev.dev**
   - Expose port 80 for external access
   - Note the assigned public IP address

## NGINX Configuration

### Basic Bearer Token Authentication

1. **Install Required Packages**
   ```bash
   sudo apt install nginx apache2-utils
   ```

2. **Configure NGINX**
   ```bash
   sudo nano /etc/nginx/sites-available/default
   ```

   Add the following configuration:
   ```nginx
   server {
       listen 80;

       location / {
           # Bearer token authentication
           if ($http_authorization != "Bearer keyxx2233hx") {
               return 403;
           }

           proxy_pass http://localhost:11434;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
   }
   ```
3. **Restart NGINX**
   ```bash
   sudo systemctl restart nginx
   ```

### Nginx Configuration Summary

This Nginx configuration does the following:

1. **Listening on Port 80**  
   The server listens for incoming HTTP requests on **port 80**.

2. **Bearer Token Authentication**  
   - The `Authorization` header is checked for the value `Bearer keyxx2233hx`.
   - If the token is missing or incorrect, the server returns a **403 Forbidden** response, rejecting the request.
   - If the token is valid, the request proceeds to the next step (proxying).

3. **Reverse Proxy**  
   If the authorization is successful, the request is forwarded to an internal service running on `http://localhost:11434`.

4. **Setting Proxy Headers**  
   The following headers are passed to the upstream service:
   - `Host`: The original `Host` header from the client request.
   - `X-Real-IP`: The client’s IP address.
   - `X-Forwarded-For`: The client’s original IP address for tracking via the reverse proxy.

### Summary
- **Port 80** is used for incoming requests.
- Requests are authenticated via a Bearer token (`Bearer keyxx2233hx`).
- Invalid tokens result in a **403 Forbidden** response.
- Valid requests are proxied to `http://localhost:11434`.
- The original client information (e.g., IP) is forwarded to the upstream service.

### Adding SSL (Optional)
If you want to add SSL encryption to your server, you can configure SSL by including your **.pem** certificate files in the Nginx configuration. For detailed instructions on setting up SSL, you can refer to online resources or documentation (e.g., [Nginx SSL configuration guide](https://nginx.org/en/docs/http/configuring_https_servers.html)).


## Client Integration

### Python Client
Originally ollama library does not support authotization, need to override with subclass to add authotization support. 

```python
from ollama import Client
import requests

#add subclass because ollama no authotization
class AuthenticatedClient(Client):
    def __init__(self, auth_key: str, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._client.headers['Authorization'] = f'Bearer {auth_key}'

# Set the authentication key
auth_key = "keyxx2233hx"

# Function to download the image
def download_image(url: str, local_path: str) -> str:
    response = requests.get(url)
    if response.status_code == 200:
        with open(local_path, 'wb') as f:
            f.write(response.content)
        return local_path
    else:
        raise Exception("Failed to download image")

# URL of the image and local filename
image_url = "https://paultan.org/image/2024/09/2024_Mitsubishi_Xpander_Preview_Malaysia_Int-22-630x420.jpg"
local_image_path = "testimage.jpg"

# Download the image
downloaded_image_path = download_image(image_url, local_image_path)


# http://35.229.78.81 is the public ip replace with your. 
client = AuthenticatedClient(host='http://35.229.78.81:80', auth_key=auth_key)
response = client.chat(model='llama3.2-vision', messages=[
    {
        'role': 'user',
        'content': 'whats in the image, explain in detail to me?',
        'images': [downloaded_image_path]
    },
])
print(response['message']['content'])
```

### cURL Examples

1. **With Bearer Token**
   ```bash
   curl -X POST http://35.229.78.81:80/v1/chat \
     -H "Authorization: Bearer keyxx2233hx" \
     -H "Content-Type: application/json" \
     -d '{
    "model": "llama3.2-vision",
    "messages": [
      {
        "role": "user",
        "content": "why the sky is blue?"
      }
    ]
     }'
   ```


### NGINX with Basic HTTP Authentication
Another method is add in Basic Authenciation, we can use nginx to do that too.

1. **Create Password File**
   ```bash
   sudo htpasswd -c /etc/nginx/.htpasswd userx01
   ```

2. **NGINX Configuration using Basic Auth**
   ```nginx
   server {
       listen 80;

       location / {
           proxy_pass http://localhost:11434;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

           auth_basic "Restricted Access";
           auth_basic_user_file /etc/nginx/.htpasswd;
       }
   }
   ```

1. **With Basic Auth**
   ```bash
   curl -u userx01:password123 http://YOUR_IP/api/chat -d '{
     "model": "llama2",
     "format": "json",
     "stream": true,
     "messages": [
       { "role": "user", "content": "Hey there!" }
     ]
   }'
   ```

## Monitoring

Check GPU utilization using either of these methods:

1. **nvidia-smi**
   ```bash
   nvidia-smi
   # or continuous monitoring
   watch -n 1 nvidia-smi
   ```

2. **nvtop**
   ```bash
   sudo apt install nvtop
   nvtop
   ```

## Additional Resources

- [Ollama OpenAI Compatibility](https://ollama.com/blog/openai-compatibility)

---
