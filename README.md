# WebSocket Chat Application

A real-time bidirectional chat application built with Python using WebSockets. This application demonstrates how to create a simple chat system where multiple clients can connect to a server and communicate with each other in real-time.

## Features

- Real-time bidirectional communication between server and clients
- Multiple client support with message broadcasting
- Server can broadcast messages to all connected clients
- Proper error handling for connection issues
- Clean terminal UI with emoji indicators
- Graceful disconnection handling

## Requirements

- Python 3.7+
- websockets library

## Installation

1. Clone this repository or download the source code
2. Install the required dependencies:

```bash
pip install -r requirements.txt
```

## Usage

### Starting the Server

Run the server script to start the WebSocket server:

```bash
python server.py
```

The server will start on `ws://localhost:8765`

### Connecting Clients

Open a new terminal window and run the client script:

```bash
python client.py
```

You can open multiple terminal windows and run the client script in each to simulate multiple users chatting.

### Chat Commands

- Type any message and press Enter to send it to all connected clients
- Type `exit` in a client to disconnect that client
- Press Ctrl+C to exit the client or server

## Testing with Postman

This application can be tested using Postman's WebSocket client. For detailed instructions in Thai, see the [thai_explanation.md](thai_explanation.md) file.

## How It Works

### Server (server.py)

The server manages WebSocket connections and handles three main tasks:

1. Accepting new client connections
2. Receiving and broadcasting messages from clients
3. Allowing the server to send messages to all clients

The server uses `asyncio` to handle multiple connections concurrently without blocking.

### Client (client.py)

The client connects to the server and provides two main functions:

1. Sending messages to the server (and through it, to other clients)
2. Receiving and displaying messages from the server and other clients

The client also uses `asyncio` to handle sending and receiving messages concurrently.

## Project Structure

- `server.py` - WebSocket server implementation
- `client.py` - WebSocket client implementation
- `requirements.txt` - Project dependencies
- `thai_explanation.md` - Detailed explanation in Thai

## License

This project is open source and available under the MIT License.