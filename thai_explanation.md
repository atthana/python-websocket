# คำอธิบายโค้ด WebSocket แบบสื่อสารสองทาง

## ภาพรวมของระบบ

เราได้สร้างแอปพลิเคชันแชทอย่างง่ายโดยใช้ WebSocket ซึ่งเป็นโปรโตคอลที่ช่วยให้การสื่อสารระหว่างเซิร์ฟเวอร์และไคลเอนต์เป็นแบบเรียลไทม์และสื่อสารได้ทั้งสองทาง (Bidirectional) โดยไม่ต้องสร้างการเชื่อมต่อใหม่ทุกครั้งที่ต้องการส่งข้อความ

## การแก้ไขโค้ดเดิม

ในโค้ดเดิม มีปัญหาดังนี้:
1. เซิร์ฟเวอร์ไม่สามารถส่งข้อความไปยังไคลเอนต์ได้อย่างถูกต้อง
2. ไคลเอนต์ไม่สามารถรับข้อความจากเซิร์ฟเวอร์ได้อย่างเหมาะสม
3. ไม่มีการจัดการข้อผิดพลาดที่ดี

## การแก้ไขในฝั่งเซิร์ฟเวอร์ (server.py)

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
            # ใช้ asyncio.to_thread เพื่อทำให้ input ไม่บล็อกการทำงาน
            msg = await asyncio.to_thread(input, "[Server] Type message to broadcast: ")
            if connected_clients:  # ตรวจสอบว่ามีไคลเอนต์เชื่อมต่ออยู่หรือไม่
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
    
    # ทำงานทั้งการรับข้อความจากเซิร์ฟเวอร์และรักษาให้เซิร์ฟเวอร์ทำงานตลอดเวลา
    await asyncio.gather(
        server_input(),
        asyncio.Future()  # Future นี้ไม่มีวันเสร็จสิ้น ทำให้เซิร์ฟเวอร์ทำงานตลอดไป
    )

if __name__ == "__main__":
    asyncio.run(main())
```

### สิ่งที่แก้ไขในเซิร์ฟเวอร์:

1. **การใช้ asyncio.to_thread**: เราใช้ `asyncio.to_thread` เพื่อทำให้การรับข้อมูลจากผู้ใช้ (input) ไม่บล็อกการทำงานของโปรแกรม ซึ่งช่วยให้เซิร์ฟเวอร์สามารถรับข้อความจากไคลเอนต์ได้ในขณะที่รอผู้ใช้พิมพ์ข้อความ

2. **การตรวจสอบไคลเอนต์**: เพิ่มการตรวจสอบว่ามีไคลเอนต์เชื่อมต่ออยู่หรือไม่ก่อนที่จะพยายามส่งข้อความ

3. **การจัดการข้อผิดพลาด**: เพิ่ม try-except เพื่อจัดการข้อผิดพลาดที่อาจเกิดขึ้น

4. **การแสดงข้อความที่ชัดเจน**: เพิ่มข้อความแจ้งเตือนที่ชัดเจนเพื่อให้ผู้ใช้ทราบสถานะของเซิร์ฟเวอร์

## การแก้ไขในฝั่งไคลเอนต์ (client.py)

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
                        print("You: ", end="", flush=True)  # แสดงพร้อมต์อีกครั้งหลังจากได้รับข้อความ
                except websockets.exceptions.ConnectionClosed:
                    print("\n❌ Connection to server closed")
                    return
                except Exception as e:
                    print(f"\n❌ Error receiving message: {e}")
                    return

            async def send():
                try:
                    while True:
                        # ใช้ asyncio.to_thread เพื่อทำให้ input ไม่บล็อกการทำงาน
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

### สิ่งที่แก้ไขในไคลเอนต์:

1. **การใช้ asyncio.to_thread**: เช่นเดียวกับเซิร์ฟเวอร์ เราใช้ `asyncio.to_thread` เพื่อทำให้การรับข้อมูลจากผู้ใช้ไม่บล็อกการทำงานของโปรแกรม

2. **การจัดการข้อผิดพลาด**: เพิ่มการจัดการข้อผิดพลาดที่ครอบคลุมมากขึ้น เช่น การจัดการเมื่อการเชื่อมต่อถูกปิด

3. **การแสดงผลที่ดีขึ้น**: ปรับปรุงการแสดงผลให้ดูง่ายขึ้น เช่น การแสดงพร้อมต์ใหม่หลังจากได้รับข้อความ

4. **คำสั่งออกจากโปรแกรม**: เพิ่มความสามารถในการออกจากโปรแกรมโดยพิมพ์ "exit"

## วิธีการทำงานของโปรแกรม

1. **เซิร์ฟเวอร์**: รันไฟล์ `server.py` เพื่อเริ่มเซิร์ฟเวอร์ WebSocket ที่พอร์ต 8765
   ```
   python3 server.py
   ```

2. **ไคลเอนต์**: รันไฟล์ `client.py` ในหน้าต่างเทอร์มินัลใหม่เพื่อเชื่อมต่อกับเซิร์ฟเวอร์
   ```
   python3 client.py
   ```

3. **การสื่อสาร**:
   - ไคลเอนต์สามารถส่งข้อความไปยังเซิร์ฟเวอร์และไคลเอนต์อื่นๆ
   - เซิร์ฟเวอร์สามารถส่งข้อความไปยังไคลเอนต์ทั้งหมด
   - ข้อความจะถูกส่งแบบเรียลไทม์โดยไม่ต้องรีเฟรชหน้าเว็บหรือสร้างการเชื่อมต่อใหม่

## เทคโนโลยีที่ใช้

1. **WebSocket**: โปรโตคอลที่ช่วยให้การสื่อสารระหว่างเซิร์ฟเวอร์และไคลเอนต์เป็นแบบเรียลไทม์และสื่อสารได้ทั้งสองทาง

2. **asyncio**: ไลบรารีสำหรับการเขียนโค้ดแบบ asynchronous ใน Python ช่วยให้โปรแกรมสามารถทำงานหลายอย่างพร้อมกันได้โดยไม่บล็อกการทำงาน

3. **websockets**: ไลบรารี Python สำหรับการใช้งาน WebSocket API

---

## การทดสอบ WebSocket ด้วย Postman

### การทดสอบเซิร์ฟเวอร์ WebSocket ด้วย Postman

1. **ติดตั้ง Postman**:
   - ดาวน์โหลดและติดตั้ง Postman จากเว็บไซต์ [https://www.postman.com/downloads/](https://www.postman.com/downloads/)
   - เปิดโปรแกรม Postman หลังจากติดตั้งเสร็จ

2. **เริ่มเซิร์ฟเวอร์ WebSocket**:
   - รันเซิร์ฟเวอร์ WebSocket ของคุณก่อนทดสอบ
   ```
   python3 server.py
   ```

3. **สร้าง WebSocket Request ใน Postman**:
   - คลิกที่ปุ่ม "New" หรือ "สร้างใหม่" ที่มุมบนซ้าย
   - เลือก "WebSocket Request"
   - ในช่อง URL ให้ใส่ `ws://localhost:8765` (URL ของเซิร์ฟเวอร์ WebSocket)

4. **เชื่อมต่อกับเซิร์ฟเวอร์**:
   - คลิกปุ่ม "Connect" หรือ "เชื่อมต่อ"
   - หากเชื่อมต่อสำเร็จ สถานะจะเปลี่ยนเป็น "Connected" หรือ "เชื่อมต่อแล้ว"
   - คุณจะเห็นข้อความในเทอร์มินัลของเซิร์ฟเวอร์ว่า "A client connected"

5. **ส่งข้อความ**:
   - ในส่วน "Message" หรือ "ข้อความ" ให้พิมพ์ข้อความที่ต้องการส่ง เช่น "สวัสดี จาก Postman"
   - คลิกปุ่ม "Send" หรือ "ส่ง"
   - คุณจะเห็นข้อความตอบกลับจากเซิร์ฟเวอร์ในส่วน "Messages" หรือ "ข้อความ" ด้านล่าง
   - ในเทอร์มินัลของเซิร์ฟเวอร์จะแสดงข้อความที่ได้รับ

6. **ตรวจสอบการตอบกลับ**:
   - Postman จะแสดงข้อความที่ส่งและข้อความตอบกลับในรูปแบบของลำดับเวลา
   - คุณสามารถดูเวลาที่ส่งและรับข้อความได้

7. **ทดสอบการเชื่อมต่อหลายเซสชัน**:
   - เปิด Postman อีกหน้าต่างหรืออีกแท็บ
   - สร้าง WebSocket Request ใหม่และเชื่อมต่อไปยัง `ws://localhost:8765`
   - ส่งข้อความจากเซสชันหนึ่งและดูว่าอีกเซสชันหนึ่งได้รับข้อความหรือไม่

8. **ปิดการเชื่อมต่อ**:
   - คลิกปุ่ม "Disconnect" หรือ "ยกเลิกการเชื่อมต่อ" เมื่อต้องการหยุดการทดสอบ
   - คุณจะเห็นข้อความในเทอร์มินัลของเซิร์ฟเวอร์ว่า "A client disconnected"