# Setting up Kubernetes for deploying LLM Chat models in production.

### Your environment
The commands to set this up were initially executed on a WSL Ubuntu 22.04 LTS instance running on Windows 11. This should work for Windows, MacOS, and Linux, as long as the prerequisites are properly installed and configured.

### Prerequisites
* [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
* [eksctl and kubectl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)

### Setting up the cluster
Use the following command to create a cluster. The following will create a cluster with the name `llm-chat-prod` in the `us-east-1` region. You can change these values as needed. Keep in mind that the AWS account's quotas must be able to support the cluster for the selected region.
```Shell
eksctl create cluster --name llm-chat-prod --region us-east-1
```
This part takes 5-10 minutes, so go grab a coffee or something while you're waiting.

### Setting up LLM Services (AI inference services)

#### Set up Metrics API
The Metrics API is used to monitor the cluster's resource usage. This is required for the HPA (Horizontal Pod Autoscaler) to work.
```Shell
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

#### Llama v2 13b 4-bit quantized model
```Shell
kubectl apply -f services/llama-13b-4bit-quantized/llama-13b-4bit-quantized.yaml
```

### Setting up NGINX Ingress Controller
Start by creating the base NGINX deployment resources for NGINX Ingress in your cluster.
```Shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

Next, apply the Ingress resource with the routing rules for the LLM Chat API.
```Shell
kubectl apply -f ingress/ingress.yaml
```

### Get the public IP address of the NGINX Ingress endpoint
```Shell
kubectl get ingress ingress-llm-production
```

Use the "Address" field from the output to make requests to the LLM endpoint. For example, if the address is `a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6-1234567890.us-east-1.elb.amazonaws.com`, then the LLM endpoint can be accessed at `http://a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6-1234567890.us-east-1.elb.amazonaws.com/llm-chat-api/v1/llm-chat`.


### Adding new models 

- Inside the services folder, add a folder with the name of the model. 
- Add a yaml file - taking example on the other directories.
- Add the path to the ingress.yaml file 

### Example Python client (endpoint up to date as of August 2, 2023)
```Python
from datetime import datetime
import requests

endpoint = "http://a4038e84b9d784ea68e58b325cc447db-193951390.us-east-1.elb.amazonaws.com/llama-v2-13b-quantized/generate_stream" # 

def get_stream(data, headers):
    print("Getting stream")
    s = requests.Session()
    with s.post(endpoint, json=data, headers=headers, stream=True) as r:
        for line in r.iter_lines():
            if line:
                yield line

if __name__ == '__main__':
    print("Running")
    data = {
        'inputs': "What is Deep Learning?",
        'parameters': {
            'max_new_tokens': 300
        }
    }
    headers = {
        'Content-Type': 'application/json'
    }
    
    now = datetime.now()
    for line in get_stream(data, headers):
        print(datetime.now() - now, line.decode('utf-8'))
```

### Cleaning up
To delete the cluster, use the following command:
```Shell
eksctl delete cluster --name llm-chat-prod --region us-east-1
```