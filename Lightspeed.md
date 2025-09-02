# Lightspeed operator

Prvo trebamo postaviti nešto što servira model, to mogu biti Ollama ili HuggingFace

Instalacija Ollama operatora:

```
oc apply -f https://raw.githubusercontent.com/llamastack/llama-stack-k8s-operator/main/release/operator.yaml
```

Pokretanje quickstart skripte:

```
git clone https://github.com/opendatahub-io/llama-stack-k8s-operator.git
./llama-stack-k8s-operator/hack/deploy-quickstart.sh 
```

Testiranje:

```
oc port-forward -n ollama-dist svc/ollama-server-service 11434:11434
curl http://localhost:11434/api/tags
curl -X POST http://localhost:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3.2:1b",
    "prompt": "Hello, how are you?",
    "stream": false
  }'
> {"model":"llama3.2:1b","created_at":"2025-09-02T08:01:12.695352503Z","response":"I'm doing well, thanks for asking. I'm a large language model, so I don't have feelings or emotions like humans do, but I'm here to help you with any questions or topics you'd like to discuss. How about you? How's your day going so far?","done":true,"done_reason":"stop","context":[...],"total_duration":3182471835,"load_duration":1075095370,"prompt_eval_count":31,"prompt_eval_duration":133426381,"eval_count":59,"eval_duration":1973451533}
```

Početni model koji dobijemo sa *deploy-quickstart.sh* je *llama3.2:1b*, ali kao parametar možemo dati naziv modela. 

Koristit ćemo *gpt-oss-20b* model sa Hugging Face-a. 

```
oc project ollama-dist
oc get pods -n ollama-dist
NAME                            READY   STATUS    RESTARTS   AGE
ollama-server-dd94cfb4b-xkgp6   1/1     Running   0          7m42s
oc exec -it ollama-server-dd94cfb4b-xkgp6 -- /bin/bash
root@ollama-server-dd94cfb4b-xkgp6:/# ollama pull gpt-oss:20b
root@ollama-server-dd94cfb4b-xkgp6:/# ollama list
NAME           ID              SIZE      MODIFIED       
gpt-oss:20b    aa4295ac10c3    13 GB     9 seconds ago     
llama3.2:1b    baf6a787fdff    1.3 GB    26 minutes ago 
```

Sada možemo pokrenuti novi model i exposeati service:

```
./llama-stack-k8s-operator/hack/deploy-quickstart.sh --provider ollama --model gpt-oss:20b
oc expose svc/ollama-server-service -n ollama-dist
```

Budući da je model jako velik, često de dogodi timeout kada pokuša dati odgovor, i zbog toga je potrebno podesiti timeout na service:

```
oc annotate route ollama-server-service   --overwrite   haproxy.router.openshift.io/timeout=10m
```

Potreban nam je i secret za api komunikaciju:

```
apiVersion: v1
data:
  apitoken: SUdOT1JF
kind: Secret
metadata:
  name: openai-api-keys
  namespace: openshift-lightspeed
type: Opaque
```

Sada instaliramo Lightspeed operator na Operator Hubu i radimo OLSConfig:

```
apiVersion: ols.openshift.io/v1alpha1
kind: OLSConfig
metadata:
  name: cluster
spec:
  llm:
    providers:
      - name: ollama
        type: openai
        credentialsSecretRef:
          name: openai-api-keys
        url: https://ollama-server-service-ollama-dist.apps.op2os.lan.croz.net/v1
        models:
          - name: gpt-oss:20b
  ols:
    defaultModel: gpt-oss:20b
    defaultProvider: ollama
```