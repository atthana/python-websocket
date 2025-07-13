# Building a Simple Bidirectional WebSocket Chat Application with Python

## Introduction

WebSockets provide a powerful protocol for real-time, bidirectional communication between clients and servers. Unlike traditional HTTP requests, WebSockets maintain a persistent connection, allowing both the server and client to send messages at any time without the overhead of establishing a new connection each time.

In this article, we'll build a simple bidirectional chat application using Python's `websockets` library. This project serves as an excellent introduction to WebSockets and can be expanded into more complex real-time applications.

## Prerequisites

- Python 3.7 or higher
- Basic understanding of asynchronous programming in Python
- The `websockets` library (`pip install websockets`)

## The WebSocket Server

Our server needs to:
1. Accept and manage client connections
2. Receive messages from clients
3. Send messages back to clients
4. Broadcast messages to all connected clients
5. Allow the server to send messages to clients

Here's our server implementation:

```python
import asyncio
import websockets

connected_clients = set()

async def handler(websocket):
    connected_clients.add(websocket)
    print("✅ A client connected.")
    try:
        async for message in websocket:
            # Send back to sender
            await websocket.send(f"You said: {message}")
            print(f"📩 Received from client: {message}")
            # Broadcast to others
            await asyncio.gather(*[
                client.send(f"Someone says: {message}")
                for client in connected_clients if client != websocket
            ])
    finally:
        connected_clients.remove(websocket)
        print("❌ A client disconnected.")

async def server_input():
    while True:
        try:
            # Use asyncio.to_thread to make input non-blocking
            msg = await asyncio.to_thread(input, "[Server] Type message to broadcast: ")
            if connected_clients:  # Check if there are any connected clients
                await asyncio.gather(*[
                    client.send(f"🟢 Server: {msg}")
                    for client in connected_clients
                ])
                print(f"📤 Server broadcast: {msg}")
            else:
                print("❗ No clients connected to receive message")
        except Exception as e:
            print(f"Error in server input: {e}")

async def main():
    server = await websockets.serve(handler, "localhost", 8765)
    print("✅ Server is running on ws://localhost:8765")
    print("Waiting for client connections...")
    
    # Run both server input handling and keep the server running
    await asyncio.gather(
        server_input(),
        asyncio.Future()  # This future never completes, keeping the server running
    )

if __name__ == "__main__":
    asyncio.run(main())
```

## The WebSocket Client

Our client needs to:
1. Connect to the server
2. Send messages to the server
3. Receive messages from the server
4. Handle connection errors gracefully

Here's our client implementation:

```python
import asyncio
import websockets
import sys

async def chat():
    uri = "ws://localhost:8765"
    try:
        print(f"Connecting to server at {uri}...")
        async with websockets.connect(uri) as websocket:
            print("✅ Connected to server!")
            print("Start chatting (type messages and press Enter)")
            
            async def receive():
                try:
                    async for message in websocket:
                        print(f"\n📩 {message}")
                        print("You: ", end="", flush=True)  # Prompt again after receiving
                except websockets.exceptions.ConnectionClosed:
                    print("\n❌ Connection to server closed")
                    return
                except Exception as e:
                    print(f"\n❌ Error receiving message: {e}")
                    return

            async def send():
                try:
                    while True:
                        # Use asyncio.to_thread to make input non-blocking
                        msg = await asyncio.to_thread(input, "You: ")
                        if msg.lower() == "exit":
                            print("Exiting chat...")
                            sys.exit(0)
                        await websocket.send(msg)
                except websockets.exceptions.ConnectionClosed:
                    print("\n❌ Connection to server closed")
                    return
                except Exception as e:
                    print(f"\n❌ Error sending message: {e}")
                    return

            await asyncio.gather(receive(), send())
    except ConnectionRefusedError:
        print("❌ Could not connect to server. Is the server running?")
    except Exception as e:
        print(f"❌ Error: {e}")

if __name__ == "__main__":
    try:
        asyncio.run(chat())
    except KeyboardInterrupt:
        print("\nExiting chat client...")
        sys.exit(0)
```

## How It Works

### The Server

1. **Connection Management**: We maintain a set of connected clients (`connected_clients`).
2. **Message Handling**: The `handler` function processes incoming messages from clients.
3. **Server Input**: The `server_input` function allows the server to broadcast messages to all connected clients.
4. **Concurrency**: We use `asyncio.gather()` to run multiple asynchronous tasks concurrently.

### The Client

1. **Connection**: The client connects to the server using the WebSocket URI.
2. **Bidirectional Communication**: We use two concurrent tasks:
   - `receive()`: Listens for incoming messages from the server
   - `send()`: Sends user input to the server
3. **Error Handling**: We handle connection errors and provide user-friendly messages.

## Running the Application

1. Start the server:
   ```
   python server.py
   ```

2. In a separate terminal, start one or more clients:
   ```
   python client.py
   ```

3. Start chatting! You can:
   - Send messages from any client
   - See messages from other clients
   - Send messages from the server to all clients

## Key Concepts

1. **WebSocket Protocol**: A persistent connection protocol that enables real-time communication.
2. **Asynchronous Programming**: Using Python's `async`/`await` syntax for non-blocking I/O operations.
3. **Concurrent Tasks**: Running multiple tasks concurrently with `asyncio.gather()`.
4. **Event-Driven Architecture**: Responding to events (messages, connections, disconnections) as they occur.

## Conclusion

This simple chat application demonstrates the power of WebSockets for real-time, bidirectional communication. The code is concise yet functional, providing a solid foundation for more complex applications.

You can extend this project in many ways:
- Add user authentication
- Implement private messaging
- Create chat rooms
- Add a web-based frontend
- Deploy to a public server

WebSockets open up a world of possibilities for real-time applications. Happy coding!
