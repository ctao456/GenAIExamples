# AvatarChatbot Application

The AvatarChatbot service can be effortlessly deployed on either Intel Gaudi2 or Intel XEON Scalable Processors.

## 🚀 Megaservice flow

```mermaid
---
config:
  flowchart:
    nodeSpacing: 100
    rankSpacing: 100
    curve: linear
  themeVariables:
    fontSize: 42px
---
flowchart LR
    %% Colors %%
    classDef blue fill:#ADD8E6,stroke:#ADD8E6,stroke-width:2px,fill-opacity:0.5
    classDef thistle fill:#D8BFD8,stroke:#ADD8E6,stroke-width:2px,fill-opacity:0.5
    classDef orange fill:#FBAA60,stroke:#ADD8E6,stroke-width:2px,fill-opacity:0.5
    classDef orchid fill:#C26DBC,stroke:#ADD8E6,stroke-width:2px,fill-opacity:0.5
    classDef invisible fill:transparent,stroke:transparent;
    style AvatarChatbot-Megaservice stroke:#000000

    %% Subgraphs %%
    subgraph AvatarChatbot-Megaservice["AvatarChatbot Megaservice"]
        direction LR
        ASR([ASR<br>3001]):::blue
        LLM([LLM 'text-generation'<br>3007]):::blue
        TTS([TTS<br>3002]):::blue
        animation([Animation<br>3008]):::blue
    end
    subgraph UserInterface["User Interface"]
        direction LR
        invis1[ ]:::invisible
        USER1([User Audio Query]):::orchid
        USER2([User Image/Video Query]):::orchid
        UI([UI server<br>]):::orchid
    end
    subgraph AvatarChatbot GateWay
        direction LR
        invis2[ ]:::invisible
        GW([AvatarChatbot GateWay<br>]):::orange
    end
    subgraph
        direction LR
        X([OPEA Microservice]):::blue
        Y{{Open Source Service}}:::thistle
        Z([OPEA Gateway]):::orange
        Z1([UI]):::orchid
    end

    %% Services %%
    WHISPER{{Whisper service<br>7066}}:::thistle
    TGI{{LLM service<br>3006}}:::thistle
    T5{{Speecht5 service<br>7055}}:::thistle
    WAV2LIP{{Wav2Lip service<br>3003}}:::thistle

    %% Connections %%
    direction LR
    USER1 -->|1| UI
    UI -->|2| GW
    GW <==>|3| AvatarChatbot-Megaservice
    ASR ==>|4| LLM ==>|5| TTS ==>|6| animation

    direction TB
    ASR <-.->|3'| WHISPER
    LLM <-.->|4'| TGI
    TTS <-.->|5'| T5
    animation <-.->|6'| WAV2LIP

    USER2 -->|1| UI
    UI <-.->|6'| WAV2LIP
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