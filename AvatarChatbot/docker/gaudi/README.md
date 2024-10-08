# AvatarChatbot Application

The AvatarChatbot service can be effortlessly deployed on either Intel Gaudi2 or Intel XEON Scalable Processors.

## 🚀 Megaservice flow

```mermaid
flowchart LR
    subgraph AvatarChatbot
        direction LR
        A[User] --> |Input query| B[AvatarChatbot Gateway]
        B --> |Invoke| Megaservice
        subgraph Megaservice["AvatarChatbot Megaservice"]
            direction LR
            subgraph AudioQnA["AudioQnA"]
                direction TB
                C((ASR<br>3001)) -. Post .-> D{{whisper-service<br>7066}}
                E((LLM<br>3007)) -. Post .-> F{{tgi-service<br>3006}}
                G((TTS<br>3002)) -. Post .-> H{{speecht5-service<br>7055}}
            end
            subgraph AvatarAnimation["Avatar Animation"]
                direction TB
                I((AvatarAnime<br>3008)) -. Post .-> J{{animation<br>7860}}
            end
            AudioQnA ==> AvatarAnimation
        end
        Megaservice --> |Output| K[Response]
    end

    subgraph Legend
        direction LR
        L([Microservice]) ==> M([Microservice])
        N([Microservice]) -.-> O{{Server API}}
    end
```

## 🚀 Build Docker images

### 1. Source Code install GenAIComps

```bash
git clone https://github.com/opea-project/GenAIComps.git
cd GenAIComps
git checkout ctao/animation
```

### 2. Build ASR Image

```bash
docker build -t opea/whisper-gaudi:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/asr/whisper/Dockerfile_hpu .


docker build -t opea/asr:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/asr/Dockerfile .
```

### 3. Build LLM Image

```bash
docker build --no-cache -t opea/llm-tgi:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/llms/text-generation/tgi/Dockerfile .
```

### 4. Build TTS Image

```bash
docker build -t opea/speecht5-gaudi:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/tts/speecht5/Dockerfile_hpu .

docker build -t opea/tts:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/tts/Dockerfile .
```

### 5. Build Animation Image

```bash
docker build -t opea/animation:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f comps/animation/Dockerfile_hpu .
```

### 6. Build MegaService Docker Image

To construct the Megaservice, we utilize the [GenAIComps](https://github.com/opea-project/GenAIComps.git) microservice pipeline within the `avatarchatbot.py` Python script. Build the MegaService Docker image using the command below:

```bash
git clone https://github.com/opea-project/GenAIExamples.git
cd GenAIExamples/AvatarChatbot/docker
docker build --no-cache -t opea/avatarchatbot:latest --build-arg https_proxy=$https_proxy --build-arg http_proxy=$http_proxy -f Dockerfile .
```

Then run the command `docker images`, you will have following images ready:

1. `opea/whisper-gaudi:latest`
2. `opea/asr:latest`
3. `opea/llm-tgi:latest`
4. `opea/speecht5-gaudi:latest`
5. `opea/tts:latest`
6. `opea/animation:latest`
7. `opea/avatarchatbot:latest`

## 🚀 Set the environment variables

Before starting the services with `docker compose`, you have to recheck the following environment variables:

```bash
export host_ip=$(hostname -I | awk '{print $1}')
export HUGGINGFACEHUB_API_TOKEN=$your_hf_token

export TGI_LLM_ENDPOINT=http://$host_ip:3006
export LLM_MODEL_ID=Intel/neural-chat-7b-v3-3
export ASR_ENDPOINT=http://$host_ip:7066
export TTS_ENDPOINT=http://$host_ip:7055
export ANIMATION_ENDPOINT=http://$host_ip:3008

export MEGA_SERVICE_HOST_IP=${host_ip}
export ASR_SERVICE_HOST_IP=${host_ip}
export TTS_SERVICE_HOST_IP=${host_ip}
export LLM_SERVICE_HOST_IP=${host_ip}
export ANIMATION_SERVICE_HOST_IP=${host_ip}

export MEGA_SERVICE_PORT=8888
export ASR_SERVICE_PORT=3001
export TTS_SERVICE_PORT=3002
export LLM_SERVICE_PORT=3007
export ANIMATION_SERVICE_PORT=3008
```

```bash
export ANIMATION_PORT=7860
# export INFERENCE_MODE='wav2clip+gfpgan'
export INFERENCE_MODE='wav2clip_only'
export CHECKPOINT_PATH='src/Wav2Lip/checkpoints/wav2lip_gan.pth'
export FACE='assets/avatar1.jpg'
# export AUDIO='assets/eg3_ref.wav' # audio file path is optional, will use base64str as input if is 'None'
export AUDIO='None'
export FACESIZE=96
export OUTFILE='/outputs/result.mp4'
export GFPGAN_MODEL_VERSION=1.3
export UPSCALE_FACTOR=1
export FPS=10
```

## 🚀 Start the MegaService

```bash
cd GenAIExamples/AvatarChatbot/docker/gaudi
docker compose up -d
```

## 🚀 Test MicroServices

```bash
# whisper service
curl http://${host_ip}:7066/v1/asr \
  -X POST \
  -d '{"audio": "UklGRigAAABXQVZFZm10IBIAAAABAAEARKwAAIhYAQACABAAAABkYXRhAgAAAAEA"}' \
  -H 'Content-Type: application/json'

# asr microservice
curl http://${host_ip}:3001/v1/audio/transcriptions \
  -X POST \
  -d '{"byte_str": "UklGRigAAABXQVZFZm10IBIAAAABAAEARKwAAIhYAQACABAAAABkYXRhAgAAAAEA"}' \
  -H 'Content-Type: application/json'

# tgi service
curl http://${host_ip}:3006/generate \
  -X POST \
  -d '{"inputs":"What is Deep Learning?","parameters":{"max_new_tokens":17, "do_sample": true}}' \
  -H 'Content-Type: application/json'

# llm microservice
curl http://${host_ip}:3007/v1/chat/completions\
  -X POST \
  -d '{"query":"What is Deep Learning?","max_new_tokens":17,"top_k":10,"top_p":0.95,"typical_p":0.95,"temperature":0.01,"repetition_penalty":1.03,"streaming":false}' \
  -H 'Content-Type: application/json'

# speecht5 service
curl http://${host_ip}:7055/v1/tts \
  -X POST \
  -d '{"text": "Who are you?"}' \
  -H 'Content-Type: application/json'

# tts microservice
curl http://${host_ip}:3002/v1/audio/speech \
  -X POST \
  -d '{"text": "Who are you?"}' \
  -H 'Content-Type: application/json'

# animation service
curl http://${host_ip}:3008/v1/animation \
  -X POST \
  -d @sample_minecraft.json \
  -H 'Content-Type: application/json'
```

## 🚀 Test MegaService

```bash
curl http://${host_ip}:3009/v1/avatarchatbot \
  -X POST \
  -d @sample_whoareyou.json \
  -H 'Content-Type: application/json'
```

If the megaservice is running properly, you should see the following output:

```bash
"/outputs/result.mp4"
```

The output file will be saved in the current directory, because `${PWD}` is mapped to `/outputs` inside the avatarchatbot-backend-service Docker container.

## Gradio UI

Follow the instructions in [Set the environment variables](#set-the-environment-variables) to set the environment variables. Follow [Start the MegaService](#start-the-megaservice) to start the MegaService. Then run the following command to start the Gradio UI:

```bash
cd GenAIExamples/AvatarChatbot/docker/ui/gradio
python3 app.py
```

## Troubleshooting

```bash
cd GenAIExamples/AvatarChatbot/tests
export IMAGE_REPO="opea"
export IMAGE_TAG="latest"
export HUGGINGFACEHUB_API_TOKEN=<your_hf_token>

bash test_avatarchatbot_on_gaudi.sh
```