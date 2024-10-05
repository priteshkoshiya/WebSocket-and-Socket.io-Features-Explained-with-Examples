# WebSocket and Socket.io Features Explained with Examples

## Introduction
This guide covers both native WebSockets and Socket.io, explaining their features, differences, and use cases with practical examples.

## WebSockets

### 1. Native WebSocket Basics

#### Server-side (Node.js with 'ws' library)
```javascript
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', function connection(ws) {
  console.log('New client connected');
  
  ws.on('message', function incoming(message) {
    console.log('received: %s', message);
  });

  ws.send('Welcome to the WebSocket server!');
});
```

#### Client-side
```javascript
const ws = new WebSocket('ws://localhost:8080');

ws.onopen = function() {
  console.log('Connected to WebSocket server');
};

ws.onmessage = function(e) {
  console.log('Received:', e.data);
};

ws.onclose = function() {
  console.log('Disconnected from WebSocket server');
};

ws.onerror = function(error) {
  console.error('WebSocket error:', error);
};
```

### 2. WebSocket Heartbeat
Implement a ping-pong mechanism to keep the connection alive:

```javascript
// Server
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

function heartbeat() {
  this.isAlive = true;
}

wss.on('connection', function connection(ws) {
  ws.isAlive = true;
  ws.on('pong', heartbeat);
});

const interval = setInterval(function ping() {
  wss.clients.forEach(function each(ws) {
    if (ws.isAlive === false) return ws.terminate();
    
    ws.isAlive = false;
    ws.ping(noop);
  });
}, 30000);

function noop() {}
```

### 3. WebSocket vs HTTP
```javascript
// HTTP Request/Response
fetch('http://api.example.com/data')
  .then(response => response.json())
  .then(data => console.log(data));

// WebSocket Continuous Connection
const ws = new WebSocket('ws://api.example.com');
ws.onmessage = function(event) {
  console.log('Real-time data:', event.data);
};
```

## Socket.io (Building on WebSockets)

### 1. Key Differences from WebSockets

#### Fallback Support
Socket.io automatically falls back to other techniques when WebSockets aren't available:

```javascript
// Server
const io = require('socket.io')(httpServer, {
  transports: ['websocket', 'polling'] // prioritize WebSocket
});

// Client
const socket = io('http://localhost:3000', {
  transports: ['websocket', 'polling']
});
```

#### Reconnection Logic
```javascript
const socket = io('http://localhost:3000', {
  reconnection: true,
  reconnectionAttempts: 5,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000
});

socket.on('reconnect', (attemptNumber) => {
  console.log('Reconnected after', attemptNumber, 'attempts');
});

socket.on('reconnect_attempt', (attemptNumber) => {
  console.log('Reconnection attempt:', attemptNumber);
});

socket.on('reconnect_error', (error) => {
  console.log('Reconnection error:', error);
});
```

### 2. Protocol Comparison Example

#### Native WebSocket
```javascript
// Server (Node.js with 'ws')
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', function connection(ws) {
  ws.on('message', function incoming(message) {
    // Need to manually parse JSON
    const data = JSON.parse(message);
    
    // Manual broadcast
    wss.clients.forEach(function each(client) {
      if (client !== ws && client.readyState === WebSocket.OPEN) {
        client.send(JSON.stringify(data));
      }
    });
  });
});

// Client
const ws = new WebSocket('ws://localhost:8080');
ws.onmessage = function(event) {
  const data = JSON.parse(event.data);
  console.log('Received:', data);
};

ws.send(JSON.stringify({ type: 'message', content: 'Hello' }));
```

#### Socket.io Equivalent
```javascript
// Server
const io = require('socket.io')(httpServer);

io.on('connection', (socket) => {
  socket.on('message', (data) => {
    // Automatic JSON parsing
    socket.broadcast.emit('message', data);
  });
});

// Client
const socket = io('http://localhost:3000');
socket.on('message', (data) => {
  console.log('Received:', data);
});

socket.emit('message', { content: 'Hello' });
```

### 3. Advanced Features Comparison

#### Binary Data Handling

WebSocket:
```javascript
// Server
wss.on('connection', function(ws) {
  ws.on('message', function(data) {
    if (data instanceof Buffer) {
      console.log('Received binary data');
    }
  });
});

// Client
const ws = new WebSocket('ws://localhost:8080');
const array = new Float32Array(5);
ws.send(array.buffer);
```

Socket.io:
```javascript
// Server
io.on('connection', (socket) => {
  socket.on('binary', (data) => {
    console.log('Received binary data');
  });
});

// Client
const array = new Float32Array(5);
socket.emit('binary', array);
```

### 4. Real-World Example: Collaborative Drawing App

```javascript
// Server (Socket.io)
io.on('connection', (socket) => {
  socket.on('draw', (data) => {
    socket.broadcast.emit('draw', data);
  });
  
  socket.on('clear', () => {
    io.emit('clear');
  });
});

// Client (Socket.io)
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
let isDrawing = false;
let lastX = 0;
let lastY = 0;

canvas.addEventListener('mousedown', startDrawing);
canvas.addEventListener('mousemove', draw);
canvas.addEventListener('mouseup', stopDrawing);

function startDrawing(e) {
  isDrawing = true;
  [lastX, lastY] = [e.offsetX, e.offsetY];
}

function draw(e) {
  if (!isDrawing) return;
  
  const drawData = {
    x0: lastX,
    y0: lastY,
    x1: e.offsetX,
    y1: e.offsetY
  };
  
  drawLine(drawData);
  socket.emit('draw', drawData);
  
  [lastX, lastY] = [e.offsetX, e.offsetY];
}

function drawLine(data) {
  ctx.beginPath();
  ctx.moveTo(data.x0, data.y0);
  ctx.lineTo(data.x1, data.y1);
  ctx.stroke();
}

socket.on('draw', drawLine);
socket.on('clear', () => {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
});
```

## Performance Considerations

1. **Message Batching**
```javascript
// Socket.io automatic batching
socket.on('connect', () => {
  socket.emit('event1', data1);
  socket.emit('event2', data2);
  socket.emit('event3', data3);
});
```

2. **Binary Data**
```javascript
// WebSocket
ws.binaryType = 'arraybuffer';
ws.send(new Uint8Array([1, 2, 3, 4]));

// Socket.io
socket.binary(true).emit('binary', new Uint8Array([1, 2, 3, 4]));
```

## When to Use What

1. **Use WebSocket when:**
   - You need minimal overhead
   - You're implementing a simple protocol
   - You need to support non-JavaScript clients
   - You're working in a restricted environment

2. **Use Socket.io when:**
   - You need automatic reconnection
   - You need fallback options
   - You want built-in multiplexing
   - You need room/namespace support
   - You want event-based communication

## Resources and Tools

1. **Debugging**
```javascript
// Socket.io debugging
localStorage.debug = '*';

// WebSocket debugging
const ws = new WebSocket('ws://localhost:8080');
ws.onmessage = function(event) {
  console.log('Raw message:', event.data);
};
```

2. **Testing**
```javascript
// WebSocket testing with wscat
// Install: npm install -g wscat
// Usage: wscat -c ws://localhost:8080

// Socket.io testing
const io = require('socket.io-client');
const socket = io('http://localhost:3000');

// Simulate events
socket.emit('testEvent', 'test data');
```

This comprehensive guide covers both WebSocket and Socket.io features, helping you understand when and how to use each technology effectively.
