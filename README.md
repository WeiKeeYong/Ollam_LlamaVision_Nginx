# Ollam_LlamaVision_Nginx
## 
# Setting Up Ollama with brev.dev and NGINX

This guide helps you set up **Ollama** on a GPU-optimized VM using **brev.dev** and configure **NGINX** for secure public access.

---

## Prerequisites
1. **brev.dev** account with a paid subscription for GPU access.
2. Basic knowledge of **Linux commands** and **NGINX configuration**.


I use brev.dev to help configure all the necessary drivers and tools. It also help me create vm with the GPU sizing i need. Once you have a paid subscription, you can creaet gpu via brev.dev and use below command in to enable ollama for public access. For ollama, i use this command on ollama server

```sh
Add Ollama:
    1 brev shell --host <server name>
    2 curl -fsSL https://ollama.com/install.sh | sh
    3 ollama pull llama3.2-vision
    4 sudo systemctl edit ollama.service
      [Service]
      Environment="OLLAMA_HOST=0.0.0.0"
    5 sudo systemctl daemon-reload
    6 sudo systemctl restart ollama

brev.dev:
    1 You need to expose the port 80 out
    2 get the ip after you expose port
```

I use NGINX, and below will wrap localhost ollama to port 80, and check the authorization key. Remember use sudo each command or run "sudo su"

```sh
sudo apt install apache2-utils
sudo apt install nginx
sudo nano /etc/nginx/sites-available/default

server {
    listen 80;

    location / {
        # Check if the Authorization header matches the specific token
        if ($http_authorization != "Bearer keyxx2233hx") {
            return 403;  # Forbidden if the token does not match
        }

        proxy_pass http://localhost:11434;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

sudo systemctl restart nginx
```

Below sample python code to access the exposed ollama to public:
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
image_url = "https://www.zupimages.net/viewer.php?id=17/07/h980.jpg"
local_image_path = "h980.jpg"

# Download the image
downloaded_image_path = download_image(image_url, local_image_path)


# Replace the Client with AuthenticatedClient, http://35.229.78.81 is the public ip replace with yours
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

check your vm gpu memory
```sh
check nvida utilization
Method 1:
- nvidia-smi
or
- watch -n 1 nvidia-smi

Method 2:
- sudo apt install nvtop
- nvtop

```
