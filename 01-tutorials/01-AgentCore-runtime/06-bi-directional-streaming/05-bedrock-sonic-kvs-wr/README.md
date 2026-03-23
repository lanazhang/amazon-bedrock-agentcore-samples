# Nova Sonic WebRTC Agent (KVS TURN)

A bidirectional voice agent using **Nova Sonic** with **WebRTC** audio transport via **Amazon Kinesis Video Streams (KVS)** TURN servers. Unlike the WebSocket-based samples (01–04), this agent uses WebRTC peer connections for audio streaming, providing lower-latency audio with built-in NAT traversal through KVS TURN relay.

## Architecture

```
Browser (WebRTC) ←→ KVS TURN Servers ←→ Agent (aiortc) → Nova Sonic (Bedrock)
```

The browser captures microphone audio via WebRTC and relays it through KVS TURN servers to the agent running on AgentCore Runtime. The agent resamples the audio to 16kHz PCM, streams it to Nova Sonic, and plays back the 24kHz spoken response through the same WebRTC connection.

## Project Structure

```
websocket/                    # Agent (runs on AgentCore Runtime)
  bot.py                      #   FastAPI server, WebRTC offer/answer, ICE handling
  kvs.py                      #   KVS signaling channel and TURN server helpers
  audio.py                    #   Audio resampling (av) and WebRTC output track
  nova_sonic.py               #   Nova Sonic bidirectional streaming session
  requirements.txt
  Dockerfile
  .env.example
client/                       # Browser client
  index.html                  #   WebRTC + optional AgentCore Runtime invocation
  client.py                   #   Static file server
  requirements.txt
kvs-iam-policy.json           # Minimal IAM policy for KVS
bedrock-iam-policy.json       # Minimal IAM policy for Nova Sonic
```

## Requirements

- **Python 3.12+** (required for aws-sdk-bedrock-runtime)
- AWS credentials configured
- **VPC with internet egress** for AgentCore Runtime deployment (see [VPC Setup](#vpc-setup-for-agentcore-runtime))

## Deploy to AgentCore

```bash
# Navigate to the bidirectional streaming tutorial root
cd 01-tutorials/01-AgentCore-runtime/06-bi-directional-streaming

# Create and activate a virtual environment
python3 -m venv .venv
source .venv/bin/activate

# Install deployment dependencies
pip install -r utils/requirements.txt

# Set your AWS account ID
export ACCOUNT_ID=123456789012

# Set AWS credentials
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key
export AWS_REGION=us-east-1

# Deploy
python utils/deploy.py 05-bedrock-sonic-kvs-wr

# Start the web client
./utils/start_client.sh 05-bedrock-sonic-kvs-wr
```

### Manual Deployment (Alternative)

If you prefer to deploy manually using the AgentCore CLI:

```bash
cd 05-bedrock-sonic-kvs-wr/websocket

pip install bedrock-agentcore-starter-toolkit

export SUBNET_IDS=subnet-0123456789abcdef0    # private subnet with NAT gateway
export SECURITY_GROUP_ID=sg-0123456789abcdef0

agentcore configure \
  -e bot.py \
  --deployment-type container \
  --disable-memory \
  --vpc \
  --subnets $SUBNET_IDS \
  --security-groups $SECURITY_GROUP_ID \
  --non-interactive

agentcore deploy --env KVS_CHANNEL_NAME=voice-agent-minimal --env AWS_REGION=us-east-1
```

VPC network mode is required because PUBLIC network mode does not support outbound UDP connectivity needed for WebRTC/TURN.

### Attach IAM Permissions

Update `ACCOUNT_ID` and region in `kvs-iam-policy.json` and `bedrock-iam-policy.json`, then attach them to the execution role from the deploy output:

```bash
ROLE_NAME=AmazonBedrockAgentCoreSDKRuntime-us-east-1-XXXXXXXXXX

aws iam put-role-policy \
  --role-name $ROLE_NAME \
  --policy-name kvs-access \
  --policy-document file://kvs-iam-policy.json

aws iam put-role-policy \
  --role-name $ROLE_NAME \
  --policy-name bedrock-nova-sonic \
  --policy-document file://bedrock-iam-policy.json
```

> **Note:** For `bedrock-iam-policy.json`, the foundation model ARN uses an empty account ID: `arn:aws:bedrock:REGION::foundation-model/amazon.nova-2-sonic-v1:0`

### Try It Out

Enter the agent ARN in the browser client along with AWS credentials. The "Force TURN only" checkbox is automatically enabled when an ARN is provided (required for VPC deployments). Click Connect and speak into your microphone.

### Cleanup

```bash
python ./utils/cleanup.py 05-bedrock-sonic-kvs-wr
```

## Local Testing

```bash
# 1. Install server dependencies
cd 05-bedrock-sonic-kvs-wr/websocket
pip install -r requirements.txt
cp .env.example .env  # Edit with your AWS credentials
python bot.py          # http://localhost:8080

# 2. In another terminal, start the client
cd 05-bedrock-sonic-kvs-wr/client
pip install -r requirements.txt
python client.py       # http://localhost:7860
```

Open `http://localhost:7860` and click "Connect" (no agent ARN needed for local mode).

## VPC Setup for AgentCore Runtime

The agent needs internet egress to reach KVS TURN servers. If you already have a VPC with a private subnet that has NAT gateway access **in a supported availability zone**, skip to deployment.

### Availability Zone Requirements

AgentCore Runtime only supports specific availability zones. AZ name-to-ID mappings vary per account. Check yours:

```bash
aws ec2 describe-availability-zones \
  --query 'AvailabilityZones[*].{Name:ZoneName,ID:ZoneId}' \
  --output table
```

For `us-east-1`, supported AZ IDs are: `use1-az1`, `use1-az2`, `use1-az4`. If you create subnets in an unsupported AZ, deployment will fail with "unsupported availability zones".

### Create VPC

1. Open the [VPC console](https://console.aws.amazon.com/vpc/) → **Create VPC** → **VPC and more**
2. Set name (e.g. `webrtc-bot-example`), keep default CIDR (`10.0.0.0/16`)
3. **1 AZ** (choose one that maps to a supported AZ ID), **1 public subnet**, **1 private subnet**
4. **NAT gateways: In 1 AZ**
5. Click **Create VPC**

### Verify Route Table

The private subnet must have a route to the NAT gateway. Without this, the agent container cannot reach the internet and will time out during initialization (120s limit).

```bash
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=YOUR_PRIVATE_SUBNET_ID" \
  --query 'RouteTables[0].Routes' --output table
```

You should see a `0.0.0.0/0` route pointing to a NAT gateway (`nat-xxx`).

## Key Components

| File | Purpose |
|------|---------|
| `websocket/bot.py` | FastAPI server with `/invocations` endpoint for ICE config, SDP offer/answer, and ICE candidate exchange |
| `websocket/kvs.py` | KVS signaling channel management — creates/finds channels, fetches TURN/ICE credentials |
| `websocket/audio.py` | Audio format conversion (48kHz→16kHz input, 24kHz output) and `OutputTrack` for WebRTC playback |
| `websocket/nova_sonic.py` | Nova Sonic bidirectional streaming — session setup, audio send/receive, barge-in handling |
| `client/index.html` | Browser WebRTC client with optional AgentCore Runtime invocation via `@aws-sdk/client-bedrock-agentcore` |
| `client/client.py` | Static file server for the HTML client |

## How It Works

### Audio Flow

**Browser → Nova Sonic:**
1. WebRTC captures microphone audio (typically 48kHz)
2. `aiortc` receives audio frames on the agent
3. `av.AudioResampler` converts to 16kHz/16-bit/mono PCM
4. Base64-encoded and streamed to Nova Sonic

**Nova Sonic → Browser:**
1. Agent receives audio chunks from Nova Sonic (24kHz PCM)
2. Raw PCM bytes buffered in `av.AudioFifo`
3. `OutputTrack` serves fixed-size 20ms frames to WebRTC
4. Browser plays audio via `<audio>` element

### WebRTC + TURN

The agent runs in a VPC private subnet behind NAT. Direct peer-to-peer connections aren't possible, so both sides use KVS TURN relay:
- Agent fetches TURN credentials with `client_id="server"` and forces `turn_only=True`
- Browser fetches TURN credentials with `client_id="web-client"` and auto-enables TURN relay when an agent ARN is provided
- All audio flows through the KVS TURN server as a relay

### Audio Configuration

| Parameter | Value |
|-----------|-------|
| Input Sample Rate | 16kHz |
| Output Sample Rate | 24kHz |
| Format | 16-bit PCM mono |
| Model | amazon.nova-2-sonic-v1:0 |
| Voice | matthew |

## Troubleshooting

**Deployment fails with "unsupported availability zones":**
Your subnet is in an AZ that AgentCore doesn't support. See [AZ Requirements](#availability-zone-requirements).

**Runtime initialization timeout (120s):**
The agent container cannot reach the internet. Verify the private subnet's route table has a `0.0.0.0/0` route to a NAT gateway, and the route table is explicitly associated with the subnet.

**STUN transaction failed (403 - Forbidden IP):**
The agent must use TURN relay mode in VPC. Ensure `turn_only=True` in `bot.py` and "Force TURN only" is checked in the browser.

**No audio playback:**
- Check microphone permissions in browser
- Ensure "Force TURN only" is checked when using AgentCore Runtime
- Check CloudWatch logs:
  ```bash
  aws logs tail /aws/bedrock-agentcore/runtimes/YOUR_AGENT_ID-DEFAULT \
    --log-stream-name-prefix "$(date -u +%Y/%m/%d)/[runtime-logs]" --since 10m
  ```

## Key Dependencies

| Package | Purpose |
|---------|---------|
| `aws-sdk-bedrock-runtime` | Nova Sonic streaming (requires Python 3.12+) |
| `aiortc` | WebRTC peer connections |
| `av` | Audio resampling and frame buffering (FFmpeg) |
| `boto3` | KVS signaling channel and TURN servers |
| `fastapi` / `uvicorn` | HTTP server |
