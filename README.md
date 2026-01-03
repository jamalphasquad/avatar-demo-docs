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

### Send Message (with pauses)

**POST** `/api/avatar/send-task`

Send text for the avatar to speak. Supports inline pause tags to insert delays between phrases while streaming.

**Request Body:**
```json
{
  "sessionId": "string",
  "text": "Hello there <break time='1000'/> how are you?"
}
```

Behavior:
- Text is split by `<break time="X"/>` tags (milliseconds).
- Each segment is sent to `streaming.task` sequentially; pauses are awaited between segments.

**Response (example):**
```json
{
  "success": true,
  "segments": [
    { "text": "Hello there", "response": { "data": { "task_id": "..." } } },
    { "text": "how are you?", "response": { "data": { "task_id": "..." } } }
  ]
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

### Interrupt Speech

**POST** `/api/avatar/interrupt`

Stops the avatar’s current spoken output immediately (no effect if not speaking).

**Request Body:**
```json
{
  "sessionId": "string"
}
```

**Response:**
```json
{
  "data": { "session_id": "string" }
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
8. Listen for avatar speech completion events (see below)

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

  // Listen for data channel from server (for receiving events)
  peerConnection.ondatachannel = (event) => {
    const dataChannel = event.channel;
    setupDataChannel(dataChannel);
  };

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

// Optional: interrupt current speech
async function interruptAvatar() {
  await fetch('**demo_url**/api/avatar/interrupt', {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_TOKEN',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ sessionId: sessionToken })
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

// Data Channel Setup for Server Events
let dataChannel = null;

function setupDataChannel(channel) {
  dataChannel = channel;
  
  channel.onopen = () => {
    console.log('Data channel opened');
    // Send session registration to associate this channel with the session
    if (sessionToken) {
      channel.send(JSON.stringify({
        type: 'register_session',
        sessionId: sessionToken
      }));
    }
  };

  channel.onmessage = (event) => {
    try {
      let messageData = event.data;
      
      // Handle binary data (ArrayBuffer)
      if (messageData instanceof ArrayBuffer) {
        const decoder = new TextDecoder();
        messageData = decoder.decode(messageData);
      }
      
      // Handle Blob
      if (messageData instanceof Blob) {
        const reader = new FileReader();
        reader.onload = () => {
          const message = JSON.parse(reader.result);
          handleDataChannelMessage(message);
        };
        reader.readAsText(messageData);
        return;
      }
      
      const message = JSON.parse(messageData);
      handleDataChannelMessage(message);
    } catch (error) {
      console.error('Error parsing data channel message:', error, event.data);
    }
  };

  channel.onclose = () => {
    console.log('Data channel closed');
    dataChannel = null;
  };

  channel.onerror = (error) => {
    console.error('Data channel error:', error);
  };
}

function handleDataChannelMessage(message) {
  console.log('Received data channel message:', message);
  
  if (message.type === 'avatar.speech.stopped') {
    onAvatarSpeechStopped(message);
  } else if (message.type === 'avatar.started.speaking') {
    console.log('Avatar started speaking');
  } else if (message.type === 'avatar.error') {
    console.error('Avatar error:', message.data);
  }
}

function onAvatarSpeechStopped(eventData) {
  console.log('Avatar stopped talking', eventData.data);
  // Add your custom logic here:
  // - Re-enable buttons
  // - Update UI state
  // - Trigger next action
  // eventData.data contains: { sessionId, totalDurationMs, segmentCount, timestamp }
}
```

### HTML Setup

```html
<video id="videoElement" autoplay playsinline></video>
<button onclick="startAvatarSession()">Start</button>
<button onclick="sendMessageToAvatar('Hello!')">Send Message</button>
<button onclick="interruptAvatar()">Interrupt</button>
<button onclick="stopAvatarSession()">Stop</button>
```

## Server-to-Client Events via RTCDataChannel

The server sends real-time events to the client through the WebRTC data channel. This provides immediate notification when the avatar completes speaking.

### Event: `avatar.speech.stopped`

Sent automatically when the avatar finishes speaking all segments (including pauses).

**Event Structure:**
```json
{
  "type": "avatar.speech.stopped",
  "data": {
    "sessionId": "string",
    "totalDurationMs": 5000,
    "segmentCount": 2,
    "timestamp": "2026-01-03T10:30:00.000Z"
  }
}
```

**Fields:**
- `sessionId`: The session that completed speaking
- `totalDurationMs`: Total speaking duration across all segments
- `segmentCount`: Number of text segments that were spoken
- `timestamp`: ISO timestamp when the event was generated

**Usage:**
The `handleDataChannelMessage()` function processes incoming events and calls `onAvatarSpeechStopped()` for this event type. Customize `onAvatarSpeechStopped()` to handle your specific use case (e.g., re-enable buttons, trigger next action, update UI state).

### Event: `avatar.started.speaking` (future)

Reserved for future use to notify when avatar begins speaking.

### Event: `avatar.error`

Sent when an error occurs during avatar operations.

**Event Structure:**
```json
{
  "type": "avatar.error",
  "data": {
    "error": "string",
    "details": "..."
  }
}
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
- To add pauses mid-utterance, include `<break time="X"/>` (ms) inside text
- To halt current speech, call `/api/avatar/interrupt`
- The server sends `avatar.speech.stopped` events via RTCDataChannel when the avatar finishes speaking
- Data channel events arrive in real-time with speaking completion; implement `onAvatarSpeechStopped()` to handle UI updates
- `task_mode: 'sync'` is used automatically to track speech duration and ensure accurate event timing
