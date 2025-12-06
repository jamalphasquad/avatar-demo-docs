# Avatar API Integration Guide

## Overview

This document provides the technical specifications for integrating with the Avatar API, including authentication, WebRTC setup, and available endpoints.

## Authentication

### Login Endpoint

**POST** `/api/login`

Authenticate and receive an access token.

**Request Body:**
```json
{
  "username": "string",
  "password": "string"
}
```

**Response:**
```json
{
  "success": true,
  "token": "string"
}
```

**Error Response:**
```json
{
  "error": "Invalid username or password"
}
```

### Using the Token

Include the token in all subsequent requests:

```javascript
headers: {
  'Authorization': 'Bearer YOUR_TOKEN',
  'Content-Type': 'application/json'
}
```

## Avatar Session Management

### Start Session

**POST** `/api/avatar/start-session`

Creates and initializes a new avatar streaming session.

**Request Body:**
```json
{
  "quality": "medium"
}
```

**Response:**
```json
{
  "data": {
    "session_id": "string",
    "sdp": {
      "type": "offer",
      "sdp": "string"
    },
    "ice_servers": [
      {
        "urls": "stun:stun.l.google.com:19302"
      }
    ]
  }
}
```

### Stop Session

**POST** `/api/avatar/stop-session`

Terminates an active avatar session.

**Request Body:**
```json
{
  "sessionId": "string"
}
```

**Response:**
```json
{
  "message": "Session stopped successfully"
}
```

### Send Message

**POST** `/api/avatar/send-task`

Send text for the avatar to speak.

**Request Body:**
```json
{
  "sessionId": "string",
  "text": "string"
}
```

**Response:**
```json
{
  "message": "Task sent successfully"
}
```

### Start Streaming

**POST** `/api/avatar/streaming-start`

Start WebRTC streaming with the avatar.

**Request Body:**
```json
{
  "sessionId": "string",
  "sdp": {
    "type": "answer",
    "sdp": "string"
  }
}
```

**Response:**
```json
{
  "message": "Streaming started successfully"
}
```

## WebRTC Implementation

### Basic Flow

1. Call `/api/avatar/start-session` to create session and receive SDP offer
2. Set up WebRTC peer connection with received ICE servers
3. Set remote description from server's SDP offer
4. Create and set local answer
5. Wait for ICE gathering to complete
6. Call `/api/avatar/streaming-start` with your SDP answer
7. Handle incoming media track for video playback

### JavaScript Implementation

```javascript
let peerConnection = null;
let sessionToken = null;

async function startAvatarSession() {
  // Step 1: Create session
  const sessionData = await fetch('**demo_url**/api/avatar/start-session', {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_TOKEN',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ quality: 'high' })
  }).then(r => r.json());

  sessionToken = sessionData.data.session_id;

  // Step 2: Setup WebRTC
  peerConnection = new RTCPeerConnection({
    iceServers: sessionData.data.ice_servers || [
      { urls: 'stun:stun.l.google.com:19302' }
    ]
  });

  // Handle incoming video stream
  peerConnection.ontrack = (event) => {
    if (event.streams && event.streams[0]) {
      document.getElementById('videoElement').srcObject = event.streams[0];
    }
  };

  // Set remote description
  await peerConnection.setRemoteDescription(
    new RTCSessionDescription(sessionData.data.sdp)
  );

  // Create answer
  const answer = await peerConnection.createAnswer();
  await peerConnection.setLocalDescription(answer);

  // Wait for ICE gathering
  await new Promise((resolve) => {
    if (peerConnection.iceGatheringState === 'complete') {
      resolve();
    } else {
      peerConnection.addEventListener('icegatheringstatechange', () => {
        if (peerConnection.iceGatheringState === 'complete') {
          resolve();
        }
      });
    }
  });

  // Step 3: Start streaming
  await fetch('**demo_url**/api/avatar/streaming-start', {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_TOKEN',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      sessionId: sessionToken,
      sdp: {
        type: 'answer',
        sdp: peerConnection.localDescription.sdp
      }
    })
  });
}

async function sendMessageToAvatar(text) {
  await fetch('**demo_url**/api/avatar/send-task', {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_TOKEN',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      sessionId: sessionToken,
      text: text
    })
  });
}

async function stopAvatarSession() {
  await fetch('**demo_url**/api/avatar/stop-session', {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_TOKEN',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      sessionId: sessionToken
    })
  });

  if (peerConnection) {
    peerConnection.close();
    peerConnection = null;
  }
}
```

### HTML Setup

```html
<video id="videoElement" autoplay playsinline></video>
<button onclick="startAvatarSession()">Start</button>
<button onclick="sendMessageToAvatar('Hello!')">Send Message</button>
<button onclick="stopAvatarSession()">Stop</button>
```

## Error Handling

All endpoints return standard HTTP status codes:

- 200: Success
- 400: Bad Request (missing or invalid parameters)
- 401: Unauthorized (invalid or missing token)
- 500: Internal Server Error

Error responses follow this format:

```json
{
  "error": "Error message description"
}
```

## Configuration

Default server URL: `**demo_url**`

Update the `BACKEND_URL` constant in your implementation to match your deployment.

## Rate Limiting

The API implements rate limiting: 100 requests per 15 minutes per IP address.

## Notes

- Sessions must be properly closed using `/api/avatar/stop-session`
- Only one active session per client is recommended
- WebRTC requires HTTPS in production environments
- ICE gathering completion is critical before starting streaming
