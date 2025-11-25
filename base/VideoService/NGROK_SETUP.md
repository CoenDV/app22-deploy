# Ngrok Setup for VideoService Webhooks

## Overview
This setup creates an ngrok tunnel to allow external webhooks (like Mux) to reach the VideoService running in OpenShift.

## Prerequisites
1. Create a free ngrok account at https://ngrok.com
2. Get your authtoken from https://dashboard.ngrok.com/get-started/your-authtoken

## Setup Steps

### 1. Create the ngrok secret in OpenShift

```bash
# Replace YOUR_AUTHTOKEN with your actual ngrok authtoken
oc create secret generic ngrok-secret \
  --from-literal=authtoken=YOUR_AUTHTOKEN \
  -n projectgroup1-test

# Optional: If you have a paid ngrok plan with a static domain
oc create secret generic ngrok-secret \
  --from-literal=authtoken=YOUR_AUTHTOKEN \
  --from-literal=domain=your-domain.ngrok-free.app \
  -n projectgroup1-test
```

### 2. Deploy the ngrok tunnel

The ngrok deployment is included in the kustomization and will be deployed automatically with:

```bash
oc apply -k overlays/test
# or
oc apply -k overlays/prod
```

### 3. Get the ngrok public URL

**Option A: Via ngrok UI Route**
- Access the ngrok UI route: https://videoservice-ngrok-ui-projectgroup1-test.apps.your-cluster.com
- The public URL will be displayed in the UI

**Option B: Via CLI**
```bash
# Get the ngrok pod name
oc get pods -n projectgroup1-test | grep ngrok

# Check the logs for the public URL
oc logs videoservice-ngrok-XXXXX -n projectgroup1-test

# Or port-forward to access the local API
oc port-forward videoservice-ngrok-XXXXX 4040:4040 -n projectgroup1-test
# Then visit http://localhost:4040
```

### 4. Configure your webhook provider

Use the ngrok public URL for your webhooks:
- Example: `https://abc123.ngrok-free.app/webhooks/mux`

### 5. Update Mux webhook URL

In the Mux dashboard:
1. Go to Settings → Webhooks
2. Update the webhook URL to: `https://YOUR_NGROK_URL/webhooks/mux`
3. The webhook secret should match `MUX_WEBHOOK_SECRET` in your deployment

## Architecture

```
Internet (Mux Webhooks)
    ↓
ngrok Cloud Service
    ↓
ngrok tunnel (in OpenShift)
    ↓
videoservice-service:8085
    ↓
VideoService Pod
```

## Notes

- **Free Plan Limitations**: ngrok free plan URLs change on restart. For production, consider:
  - Paid ngrok plan with static domains
  - OpenShift routes with proper firewall configuration
  - API Gateway or external ingress solution

- **Security**: The ngrok authtoken is stored as a Kubernetes secret and should be protected

- **Monitoring**: Access the ngrok UI route to monitor incoming webhook requests and troubleshoot issues

## Troubleshooting

### ngrok pod not starting
```bash
# Check pod status
oc get pods -n projectgroup1-test | grep ngrok

# Check logs
oc logs videoservice-ngrok-XXXXX -n projectgroup1-test

# Verify secret exists
oc get secret ngrok-secret -n projectgroup1-test
```

### Webhook not received
1. Check ngrok UI for incoming requests
2. Verify VideoService logs for webhook processing
3. Ensure webhook URL in Mux matches the ngrok URL
4. Check webhook secret configuration

### URL changes on restart
- This is expected with free ngrok accounts
- Update the webhook URL in Mux after each restart
- Consider upgrading to paid ngrok for static domains
