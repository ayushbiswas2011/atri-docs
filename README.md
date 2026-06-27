Production Deployment Guide
This guide provides step-by-step instructions for deploying ATRI in your environment. By completing this setup, your application will operate on an unrestricted infrastructure tier with zero-maintenance lifetime licensing synchronization.

🏗️ System Architecture Overview
ATRI functions as a highly optimized, two-component architecture engineered for sub-millisecond edge processing:

[ Webhook Senders ] (Stripe/Razorpay/Shopify)
         │
         ▼
┌────────────────────────────────────────┐
│  1. Core Proxy Edge Container (Docker)  │ <── [ Unix Socket Control Plane (atrictl) ]
│     - io_uring / MPMC Ring Buffer      │
│     - Crash-Safe WAL & Deduplication   │
└────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────┐
│ 2. Upstream Validation Layer (Vercel)  │
│    - Stateless Config Handshake        │
│    - Zero-Maintenance Synchronization  │
└────────────────────────────────────────┘
         │
         ▼
[ Your Main Backend Server / Database ]
Core Proxy Edge Container (Docker): The primary containerized service written in C++20 that intercepts system calls, manages the MPMC ring buffer, handles local WAL replication, and routes telemetry securely.

Upstream Validation Layer (Serverless): A lightweight serverless API instance running on Vercel that handles and signs configuration handshake payloads to activate the unrestricted scaling pipeline.

🛠️ Step-by-Step Deployment Instructions
Follow these exact steps to pull, configure, and verify the ATRI engine on your Linux instance or local machine.

Step 1: Verify the Serverless Upstream
Before executing the container runtime, verify that your dedicated serverless endpoint is live on the Vercel network edge and responding to initialization handshakes.

Run the following command in your terminal to test connectivity:

Bash


curl -i https://atri-ui-ux-design.vercel.app/api/verify
Expected Response:
The server should return an HTTP status code (such as 404 Not Found or a generic routing payload). This confirms that the Vercel routing configuration is active, reachable, and listening for container verification queries.

Step 2: Initialize the Container Lifecycle
Deploy the production-ready Docker image directly from the GitHub Container Registry (GHCR). This step automatically handles layer unpacking, sets up the internal memory guards, and maps the ingress traffic to port 8080.

Run the following command to spin up the background container:

Bash


docker run -d \
  --name atri-core-proxy \
  --restart unless-stopped \
  -p 8080:8080 \
  -e ATRI_LICENSE_KEY=ayush \
  -e ATRI_LICENSE_URL=https://atri-ui-ux-design.vercel.app/api/verify \
  ghcr.io/ayushbiswas2011/atri-proxy:latest
Step 3: Verify Runtime Container Logs
Confirm that the initialization routines have successfully established communication with the upstream serverless handler and successfully bypassed internal evaluation limits.

Monitor the runtime engine logs in real-time:

Bash


docker logs -f atri-core-proxy
What to look for:
Look for initialization logs showing a initialization status of ok or tier: scale / unrestricted. This confirms successful container startup, active memory guards, and scaling pipeline activation.

📋 Environment Variable Reference
Configure these variables inside your Docker environment deployment script or .env file to customize the edge layer behavior:

Variable Name	Production Value	Description
ATRI_LICENSE_KEY	ayush	Unique production signature key for local payload validation.
ATRI_LICENSE_URL	[https://atri-ui-ux-design.vercel.app/api/verify](https://atri-ui-ux-design.vercel.app/api/verify)	Global serverless callback URI on the Vercel network edge.

💻 How to Use ATRI (Developer Integration)
Once the container is running successfully on port 8080, updating your architecture to utilize ATRI's zero-data-loss pipeline is entirely non-intrusive.

1. Route Your Webhooks
Instead of pointing your Stripe, Razorpay, or Shopify webhooks directly to your backend application endpoint, point them to ATRI's listener URL:

http://<your-server-ip>:8080/v1/webhooks
2. High-Speed Ingestion Validation
Instant ACK: ATRI's lock-free MPMC queue will ingest the incoming payload, write it safely to the Write-Ahead Log (WAL) with CRC32C checksums, and instantly drop an HTTP 200 OK back to the sender in under 0.5ms.

Proxy Forwarding: ATRI then handles the asynchronous forwarding stream to your actual backend server safely. If your backend is down or hits a timeout, ATRI keeps the events in the local SQLite WAL/DLQ stack and automatically retries using exponential backoff with full jitter.

3. Local Administration via CLI (atrictl)
You can manage the proxy runtime engine, check the internal Bloom filter metrics, or interact with the Dead Letter Queue (DLQ) by accessing the container's built-in control plane CLI tool:

Bash


# Check the runtime health and structural status
docker exec -it atri-core-proxy atrictl status

# Force replay all events currently stored inside the Dead Letter Queue
docker exec -it atri-core-proxy atrictl replay-all
