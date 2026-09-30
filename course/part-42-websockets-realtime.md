# ตอนที่ 42: WebSockets และ Real-time Communication

## บทนำ

WebSockets ช่วยให้ client และ server สื่อสารกันแบบ two-way real-time TypeScript ช่วยให้การพัฒนา real-time applications ปลอดภัยและมีประสิทธิภาพมากขึ้น

---

## 42.1 WebSocket พื้นฐานกับ TypeScript

### Native WebSocket API

```typescript
// client/websocket-client.ts

interface WebSocketMessage<T = any> {
  type: string;
  payload: T;
  timestamp: number;
}

class TypedWebSocketClient {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelay = 1000;
  private listeners = new Map<string, Set<(payload: any) => void>>();

  constructor(private url: string) {}

  connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(this.url);

      this.ws.onopen = () => {
        console.log('WebSocket connected');
        this.reconnectAttempts = 0;
        resolve();
      };

      this.ws.onmessage = (event: MessageEvent) => {
        try {
          const message: WebSocketMessage = JSON.parse(event.data);
          this.emit(message.type, message.payload);
        } catch (error) {
          console.error('Failed to parse message:', error);
        }
      };

      this.ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        reject(error);
      };

      this.ws.onclose = () => {
        console.log('WebSocket closed');
        this.scheduleReconnect();
      };
    });
  }

  send<T>(type: string, payload: T): void {
    if (this.ws?.readyState !== WebSocket.OPEN) {
      throw new Error('WebSocket is not connected');
    }

    const message: WebSocketMessage<T> = {
      type,
      payload,
      timestamp: Date.now(),
    };

    this.ws.send(JSON.stringify(message));
  }

  on<T>(event: string, callback: (payload: T) => void): () => void {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(callback);

    // Return unsubscribe function
    return () => {
      this.listeners.get(event)?.delete(callback);
    };
  }

  private emit(event: string, payload: any): void {
    this.listeners.get(event)?.forEach(callback => callback(payload));
    this.listeners.get('*')?.forEach(callback => callback({ event, payload }));
  }

  private scheduleReconnect(): void {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error('Max reconnect attempts reached');
      return;
    }

    const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts);
    this.reconnectAttempts++;

    console.log(`Reconnecting in ${delay}ms... (attempt ${this.reconnectAttempts})`);
    setTimeout(() => this.connect().catch(console.error), delay);
  }

  disconnect(): void {
    this.ws?.close(1000, 'Client disconnecting');
    this.ws = null;
  }

  get isConnected(): boolean {
    return this.ws?.readyState === WebSocket.OPEN;
  }
}

// การใช้งาน
const client = new TypedWebSocketClient('ws://localhost:8080');

client.connect().then(() => {
  // Subscribe to events
  client.on<{ message: string; userId: string }>('chat_message', (payload) => {
    console.log(`Message from ${payload.userId}: ${payload.message}`);
  });

  client.on<{ userId: string; online: boolean }>('user_status', (payload) => {
    console.log(`User ${payload.userId} is ${payload.online ? 'online' : 'offline'}`);
  });

  // Send messages
  client.send('send_message', {
    roomId: 'general',
    message: 'สวัสดีครับ!',
  });
});
```

---

## 42.2 ws Library กับ TypeScript

### WebSocket Server

```bash
npm install ws
npm install -D @types/ws
```

```typescript
// server/websocket-server.ts
import WebSocket, { WebSocketServer } from 'ws';
import { IncomingMessage } from 'http';
import { verify } from 'jsonwebtoken';
import { v4 as uuidv4 } from 'uuid';

interface Client {
  id: string;
  ws: WebSocket;
  userId?: string;
  rooms: Set<string>;
  connectedAt: Date;
  lastActivity: Date;
}

interface ServerMessage<T = any> {
  type: string;
  payload: T;
  from?: string;
  to?: string;
  room?: string;
  timestamp: number;
}

class WebSocketServerManager {
  private wss: WebSocketServer;
  private clients = new Map<string, Client>();
  private rooms = new Map<string, Set<string>>(); // roomId -> clientIds
  private pingInterval: NodeJS.Timer;

  constructor(port: number) {
    this.wss = new WebSocketServer({
      port,
      verifyClient: this.verifyClient.bind(this),
    });

    this.wss.on('connection', this.handleConnection.bind(this));

    // Ping clients เพื่อ keep-alive
    this.pingInterval = setInterval(() => {
      this.pingClients();
    }, 30000);

    console.log(`WebSocket Server listening on port ${port}`);
  }

  private verifyClient(
    info: { req: IncomingMessage },
    done: (result: boolean, code?: number, message?: string) => void
  ): void {
    const token = new URL(
      info.req.url || '',
      'ws://localhost'
    ).searchParams.get('token');

    if (!token) {
      done(false, 401, 'Authentication required');
      return;
    }

    try {
      verify(token, process.env.JWT_SECRET || 'secret');
      done(true);
    } catch {
      done(false, 401, 'Invalid token');
    }
  }

  private handleConnection(ws: WebSocket, req: IncomingMessage): void {
    const clientId = uuidv4();
    const url = new URL(req.url || '', 'ws://localhost');
    const token = url.searchParams.get('token');

    let userId: string | undefined;
    if (token) {
      try {
        const decoded = verify(token, process.env.JWT_SECRET || 'secret') as any;
        userId = decoded.userId;
      } catch { /* ignore */ }
    }

    const client: Client = {
      id: clientId,
      ws,
      userId,
      rooms: new Set(),
      connectedAt: new Date(),
      lastActivity: new Date(),
    };

    this.clients.set(clientId, client);
    console.log(`Client connected: ${clientId} (user: ${userId})`);

    // Send welcome message
    this.send(ws, 'connected', { clientId, userId });

    ws.on('message', (data: Buffer) => {
      try {
        const message: ServerMessage = JSON.parse(data.toString());
        client.lastActivity = new Date();
        this.handleMessage(client, message);
      } catch (error) {
        console.error('Invalid message format:', error);
        this.send(ws, 'error', { message: 'Invalid message format' });
      }
    });

    ws.on('close', (code, reason) => {
      this.handleDisconnect(client, code, reason.toString());
    });

    ws.on('error', (error) => {
      console.error(`Client ${clientId} error:`, error);
    });

    ws.on('pong', () => {
      client.lastActivity = new Date();
    });
  }

  private handleMessage(client: Client, message: ServerMessage): void {
    switch (message.type) {
      case 'join_room':
        this.joinRoom(client, message.payload.roomId);
        break;

      case 'leave_room':
        this.leaveRoom(client, message.payload.roomId);
        break;

      case 'send_message':
        this.broadcastToRoom(
          message.payload.roomId,
          'chat_message',
          {
            message: message.payload.message,
            userId: client.userId,
            clientId: client.id,
          },
          client.id
        );
        break;

      case 'direct_message':
        this.sendToUser(message.payload.targetUserId, 'direct_message', {
          message: message.payload.message,
          fromUserId: client.userId,
        });
        break;

      case 'typing':
        this.broadcastToRoom(
          message.payload.roomId,
          'user_typing',
          { userId: client.userId, isTyping: message.payload.isTyping },
          client.id
        );
        break;

      default:
        console.warn(`Unknown message type: ${message.type}`);
    }
  }

  private handleDisconnect(client: Client, code: number, reason: string): void {
    console.log(`Client disconnected: ${client.id} (${code}: ${reason})`);

    // ลบออกจาก rooms ทั้งหมด
    client.rooms.forEach(roomId => {
      const room = this.rooms.get(roomId);
      if (room) {
        room.delete(client.id);
        // แจ้ง room ว่า user ออกไปแล้ว
        this.broadcastToRoom(roomId, 'user_left', {
          userId: client.userId,
          clientId: client.id,
        });
      }
    });

    this.clients.delete(client.id);
  }

  joinRoom(client: Client, roomId: string): void {
    if (!this.rooms.has(roomId)) {
      this.rooms.set(roomId, new Set());
    }

    this.rooms.get(roomId)!.add(client.id);
    client.rooms.add(roomId);

    // แจ้ง client ว่า join สำเร็จ
    this.send(client.ws, 'room_joined', { roomId });

    // แจ้ง room ว่ามี user ใหม่
    this.broadcastToRoom(roomId, 'user_joined', {
      userId: client.userId,
      clientId: client.id,
    }, client.id);

    // ส่งรายชื่อ users ใน room
    const usersInRoom = Array.from(this.rooms.get(roomId)!)
      .map(id => this.clients.get(id))
      .filter(Boolean)
      .map(c => ({ userId: c!.userId, clientId: c!.id }));

    this.send(client.ws, 'room_users', { roomId, users: usersInRoom });
  }

  leaveRoom(client: Client, roomId: string): void {
    this.rooms.get(roomId)?.delete(client.id);
    client.rooms.delete(roomId);

    this.send(client.ws, 'room_left', { roomId });
    this.broadcastToRoom(roomId, 'user_left', {
      userId: client.userId,
      clientId: client.id,
    });
  }

  broadcastToRoom(
    roomId: string,
    type: string,
    payload: any,
    excludeClientId?: string
  ): void {
    const room = this.rooms.get(roomId);
    if (!room) return;

    const message: ServerMessage = {
      type,
      payload,
      room: roomId,
      timestamp: Date.now(),
    };

    room.forEach(clientId => {
      if (clientId === excludeClientId) return;
      const client = this.clients.get(clientId);
      if (client?.ws.readyState === WebSocket.OPEN) {
        client.ws.send(JSON.stringify(message));
      }
    });
  }

  sendToUser(userId: string, type: string, payload: any): void {
    this.clients.forEach(client => {
      if (client.userId === userId && client.ws.readyState === WebSocket.OPEN) {
        this.send(client.ws, type, payload);
      }
    });
  }

  broadcast(type: string, payload: any, excludeClientId?: string): void {
    this.clients.forEach(client => {
      if (client.id !== excludeClientId && client.ws.readyState === WebSocket.OPEN) {
        this.send(client.ws, type, payload);
      }
    });
  }

  private send(ws: WebSocket, type: string, payload: any): void {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify({ type, payload, timestamp: Date.now() }));
    }
  }

  private pingClients(): void {
    const now = Date.now();
    const timeout = 60000; // 1 minute

    this.clients.forEach((client, clientId) => {
      const lastActivity = client.lastActivity.getTime();

      if (now - lastActivity > timeout) {
        console.log(`Client ${clientId} timed out, disconnecting`);
        client.ws.terminate();
        return;
      }

      if (client.ws.readyState === WebSocket.OPEN) {
        client.ws.ping();
      }
    });
  }

  getStats() {
    return {
      totalClients: this.clients.size,
      totalRooms: this.rooms.size,
      rooms: Array.from(this.rooms.entries()).map(([id, clients]) => ({
        id,
        clientCount: clients.size,
      })),
    };
  }

  close(): void {
    clearInterval(this.pingInterval as NodeJS.Timeout);
    this.wss.close();
  }
}

// การเริ่มต้น server
const wsServer = new WebSocketServerManager(8080);
```

---

## 42.3 Socket.IO กับ TypeScript

### การติดตั้ง

```bash
npm install socket.io
npm install -D @types/socket.io
# Client
npm install socket.io-client
```

### Typed Events

```typescript
// shared/socket-events.ts

// Server -> Client events
export interface ServerToClientEvents {
  // Chat
  chat_message: (data: ChatMessage) => void;
  user_joined: (data: UserJoinedData) => void;
  user_left: (data: UserLeftData) => void;
  user_typing: (data: TypingData) => void;
  
  // Notifications
  notification: (data: Notification) => void;
  
  // Real-time updates
  post_updated: (data: PostUpdated) => void;
  order_status_changed: (data: OrderStatus) => void;
  
  // Connection
  error: (data: ErrorData) => void;
  connected: (data: ConnectedData) => void;
}

// Client -> Server events
export interface ClientToServerEvents {
  // Chat
  join_room: (roomId: string, callback: AckCallback) => void;
  leave_room: (roomId: string) => void;
  send_message: (data: SendMessageData, callback: AckCallback) => void;
  typing: (data: TypingData) => void;
  
  // Presence
  set_status: (status: UserStatus) => void;
}

// Inter-server events (cluster mode)
export interface InterServerEvents {
  ping: () => void;
  user_connected: (userId: string) => void;
  user_disconnected: (userId: string) => void;
}

// Socket data (per connection)
export interface SocketData {
  userId: string;
  username: string;
  role: string;
  rooms: Set<string>;
}

// Types
export interface ChatMessage {
  id: string;
  content: string;
  userId: string;
  username: string;
  roomId: string;
  timestamp: Date;
  edited?: boolean;
}

export interface SendMessageData {
  roomId: string;
  content: string;
}

export interface UserJoinedData {
  userId: string;
  username: string;
  roomId: string;
}

export interface UserLeftData {
  userId: string;
  username: string;
  roomId: string;
}

export interface TypingData {
  userId: string;
  username: string;
  roomId: string;
  isTyping: boolean;
}

export interface Notification {
  id: string;
  type: 'info' | 'success' | 'warning' | 'error';
  message: string;
  data?: any;
}

export interface PostUpdated {
  postId: string;
  changes: Record<string, any>;
}

export interface OrderStatus {
  orderId: string;
  status: string;
  updatedAt: Date;
}

export interface ErrorData {
  code: string;
  message: string;
}

export interface ConnectedData {
  socketId: string;
  userId: string;
}

export type AckCallback<T = void> = (error: string | null, data?: T) => void;

export type UserStatus = 'online' | 'away' | 'busy' | 'offline';
```

### Socket.IO Server

```typescript
// server/socket-server.ts
import { Server } from 'socket.io';
import { createServer } from 'http';
import express from 'express';
import { verify } from 'jsonwebtoken';
import {
  ServerToClientEvents,
  ClientToServerEvents,
  InterServerEvents,
  SocketData,
  ChatMessage,
} from '../shared/socket-events';
import { v4 as uuidv4 } from 'uuid';

type TypedServer = Server<
  ClientToServerEvents,
  ServerToClientEvents,
  InterServerEvents,
  SocketData
>;

const app = express();
const httpServer = createServer(app);

const io: TypedServer = new Server(httpServer, {
  cors: {
    origin: process.env.CLIENT_URL || 'http://localhost:3000',
    credentials: true,
  },
  pingTimeout: 60000,
  pingInterval: 25000,
});

// Authentication middleware
io.use((socket, next) => {
  const token = socket.handshake.auth.token || socket.handshake.headers.token;

  if (!token) {
    return next(new Error('Authentication required'));
  }

  try {
    const decoded = verify(token as string, process.env.JWT_SECRET || 'secret') as any;
    socket.data.userId = decoded.userId;
    socket.data.username = decoded.username;
    socket.data.role = decoded.role;
    socket.data.rooms = new Set();
    next();
  } catch {
    next(new Error('Invalid token'));
  }
});

// Connection handler
io.on('connection', (socket) => {
  const { userId, username } = socket.data;
  console.log(`User connected: ${username} (${socket.id})`);

  // Auto-join personal room
  socket.join(`user:${userId}`);

  // Send connected event
  socket.emit('connected', {
    socketId: socket.id,
    userId,
  });

  // Join room handler
  socket.on('join_room', async (roomId, callback) => {
    try {
      await socket.join(roomId);
      socket.data.rooms.add(roomId);

      // แจ้ง room ว่ามี user ใหม่
      socket.to(roomId).emit('user_joined', {
        userId,
        username,
        roomId,
      });

      // ดึงประวัติ messages
      const history = await getMessageHistory(roomId);

      callback(null, history as any);
      console.log(`${username} joined room ${roomId}`);
    } catch (error: any) {
      callback(error.message);
    }
  });

  // Leave room handler
  socket.on('leave_room', (roomId) => {
    socket.leave(roomId);
    socket.data.rooms.delete(roomId);

    socket.to(roomId).emit('user_left', {
      userId,
      username,
      roomId,
    });
  });

  // Send message handler
  socket.on('send_message', async (data, callback) => {
    try {
      const { roomId, content } = data;

      // Validate
      if (!content || content.trim().length === 0) {
        return callback('ข้อความไม่ควรว่างเปล่า');
      }

      if (content.length > 1000) {
        return callback('ข้อความยาวเกิน 1000 ตัวอักษร');
      }

      const message: ChatMessage = {
        id: uuidv4(),
        content: content.trim(),
        userId,
        username,
        roomId,
        timestamp: new Date(),
      };

      // Save to database
      await saveMessage(message);

      // Broadcast to room (including sender)
      io.to(roomId).emit('chat_message', message);

      callback(null);
    } catch (error: any) {
      callback(error.message);
    }
  });

  // Typing indicator
  socket.on('typing', (data) => {
    socket.to(data.roomId).emit('user_typing', {
      userId,
      username,
      roomId: data.roomId,
      isTyping: data.isTyping,
    });
  });

  // Disconnect handler
  socket.on('disconnect', (reason) => {
    console.log(`User disconnected: ${username} (${reason})`);

    // แจ้งทุก room ที่ user อยู่
    socket.data.rooms.forEach(roomId => {
      io.to(roomId).emit('user_left', { userId, username, roomId });
    });
  });
});

// Helper functions
async function getMessageHistory(roomId: string): Promise<ChatMessage[]> {
  // Simulate database query
  return [];
}

async function saveMessage(message: ChatMessage): Promise<void> {
  // Simulate database save
  console.log('Saved message:', message.id);
}

// Utility functions
export function sendNotificationToUser(
  userId: string,
  notification: { type: 'info' | 'success' | 'warning' | 'error'; message: string; data?: any }
): void {
  io.to(`user:${userId}`).emit('notification', {
    id: uuidv4(),
    ...notification,
  });
}

httpServer.listen(4000, () => {
  console.log('🚀 Socket.IO server running on port 4000');
});

export { io };
```

---

## 42.4 Rooms และ Namespaces

### Namespaces

```typescript
// server/namespaces.ts
import { io } from './socket-server';
import { Namespace } from 'socket.io';

// Chat namespace
const chatNamespace: Namespace = io.of('/chat');
chatNamespace.on('connection', (socket) => {
  console.log(`Chat socket connected: ${socket.id}`);

  socket.on('join_channel', (channelId: string) => {
    socket.join(`channel:${channelId}`);
    socket.to(`channel:${channelId}`).emit('user_joined', {
      userId: socket.data.userId,
      channelId,
    });
  });
});

// Game namespace
const gameNamespace: Namespace = io.of('/game');
gameNamespace.on('connection', (socket) => {
  console.log(`Game socket connected: ${socket.id}`);

  socket.on('join_game', (gameId: string) => {
    socket.join(`game:${gameId}`);
    
    const playersInGame = gameNamespace.adapter
      .rooms.get(`game:${gameId}`)?.size || 0;

    gameNamespace.to(`game:${gameId}`).emit('player_joined', {
      playerId: socket.data.userId,
      playerCount: playersInGame,
    });
  });

  socket.on('game_action', (data: { gameId: string; action: any }) => {
    socket.to(`game:${data.gameId}`).emit('game_state_updated', data.action);
  });
});

// Admin namespace (requires admin role)
const adminNamespace: Namespace = io.of('/admin');
adminNamespace.use((socket, next) => {
  if (socket.data.role !== 'ADMIN') {
    return next(new Error('ต้องการสิทธิ์ผู้ดูแลระบบ'));
  }
  next();
});

adminNamespace.on('connection', (socket) => {
  console.log(`Admin connected: ${socket.data.userId}`);

  // Real-time stats
  const statsInterval = setInterval(() => {
    socket.emit('server_stats', {
      connectedUsers: io.engine.clientsCount,
      uptime: process.uptime(),
      memoryUsage: process.memoryUsage(),
    });
  }, 5000);

  socket.on('disconnect', () => {
    clearInterval(statsInterval);
  });
});
```

### Room Management

```typescript
// server/room-manager.ts
import { Server } from 'socket.io';

export interface Room {
  id: string;
  name: string;
  type: 'public' | 'private' | 'direct';
  maxUsers: number;
  createdBy: string;
  createdAt: Date;
  metadata: Record<string, any>;
}

export class RoomManager {
  private rooms = new Map<string, Room>();

  constructor(private io: Server) {}

  async createRoom(config: Omit<Room, 'createdAt'>): Promise<Room> {
    if (this.rooms.has(config.id)) {
      throw new Error(`Room ${config.id} already exists`);
    }

    const room: Room = {
      ...config,
      createdAt: new Date(),
    };

    this.rooms.set(config.id, room);
    return room;
  }

  async joinRoom(socketId: string, roomId: string): Promise<void> {
    const room = this.rooms.get(roomId);
    if (!room) throw new Error('Room not found');

    const currentSize = await this.getRoomSize(roomId);
    if (currentSize >= room.maxUsers) {
      throw new Error('Room is full');
    }

    const socket = this.io.sockets.sockets.get(socketId);
    if (!socket) throw new Error('Socket not found');

    await socket.join(roomId);
  }

  async leaveRoom(socketId: string, roomId: string): Promise<void> {
    const socket = this.io.sockets.sockets.get(socketId);
    socket?.leave(roomId);
  }

  async getRoomSize(roomId: string): Promise<number> {
    const sockets = await this.io.in(roomId).fetchSockets();
    return sockets.length;
  }

  async getRoomUsers(roomId: string): Promise<string[]> {
    const sockets = await this.io.in(roomId).fetchSockets();
    return sockets.map(s => s.data.userId);
  }

  getRoom(roomId: string): Room | undefined {
    return this.rooms.get(roomId);
  }

  getAllRooms(): Room[] {
    return Array.from(this.rooms.values());
  }

  deleteRoom(roomId: string): boolean {
    // Kick all users
    this.io.in(roomId).disconnectSockets(true);
    return this.rooms.delete(roomId);
  }
}
```

---

## 42.5 Authentication กับ WebSockets

```typescript
// server/auth-middleware.ts
import { Socket } from 'socket.io';
import { verify, JwtPayload } from 'jsonwebtoken';

interface AuthenticatedSocket extends Socket {
  data: {
    userId: string;
    username: string;
    role: string;
    rooms: Set<string>;
  };
}

export function createAuthMiddleware(jwtSecret: string) {
  return (socket: AuthenticatedSocket, next: (err?: Error) => void) => {
    // ดึง token จากหลายที่
    const token =
      socket.handshake.auth.token ||
      socket.handshake.headers['x-auth-token'] ||
      socket.handshake.query.token;

    if (!token) {
      return next(new Error('No authentication token provided'));
    }

    try {
      const decoded = verify(token as string, jwtSecret) as JwtPayload & {
        userId: string;
        username: string;
        role: string;
      };

      socket.data.userId = decoded.userId;
      socket.data.username = decoded.username;
      socket.data.role = decoded.role;
      socket.data.rooms = new Set();

      next();
    } catch (error: any) {
      if (error.name === 'TokenExpiredError') {
        next(new Error('Token expired'));
      } else {
        next(new Error('Invalid token'));
      }
    }
  };
}

// Rate limiter middleware
export function createRateLimiter(maxEventsPerSecond: number) {
  const eventCounts = new Map<string, { count: number; resetTime: number }>();

  return (socket: Socket, next: (err?: Error) => void) => {
    const socketId = socket.id;
    const now = Date.now();

    const existing = eventCounts.get(socketId);

    if (!existing || now > existing.resetTime) {
      eventCounts.set(socketId, {
        count: 1,
        resetTime: now + 1000,
      });
      return next();
    }

    if (existing.count >= maxEventsPerSecond) {
      return next(new Error('Rate limit exceeded'));
    }

    existing.count++;
    next();
  };
}
```

---

## 42.6 NestJS WebSockets (Gateways)

### การติดตั้ง

```bash
npm install @nestjs/websockets @nestjs/platform-socket.io
npm install socket.io
```

### Chat Gateway

```typescript
// src/chat/chat.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  OnGatewayInit,
  OnGatewayConnection,
  OnGatewayDisconnect,
  MessageBody,
  ConnectedSocket,
  WsResponse,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { Logger, UseGuards } from '@nestjs/common';
import { JwtWsGuard } from '../guards/jwt-ws.guard';
import { ChatService } from './chat.service';
import { CreateMessageDto } from './dto/create-message.dto';

@WebSocketGateway({
  namespace: '/chat',
  cors: {
    origin: '*',
    credentials: true,
  },
})
export class ChatGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  server!: Server;

  private readonly logger = new Logger(ChatGateway.name);

  constructor(private readonly chatService: ChatService) {}

  afterInit(server: Server) {
    this.logger.log('Chat Gateway initialized');
  }

  handleConnection(client: Socket) {
    this.logger.log(`Client connected: ${client.id}`);
  }

  handleDisconnect(client: Socket) {
    this.logger.log(`Client disconnected: ${client.id}`);
    
    // แจ้ง rooms ที่ client อยู่
    client.rooms.forEach(room => {
      if (room !== client.id) {
        client.to(room).emit('user_left', {
          userId: client.data.userId,
          username: client.data.username,
        });
      }
    });
  }

  @UseGuards(JwtWsGuard)
  @SubscribeMessage('join_room')
  async handleJoinRoom(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { roomId: string }
  ): Promise<WsResponse<any>> {
    const { roomId } = data;

    await client.join(roomId);

    // แจ้ง room ว่ามี user ใหม่
    client.to(roomId).emit('user_joined', {
      userId: client.data.userId,
      username: client.data.username,
      roomId,
    });

    // ดึงประวัติ messages
    const history = await this.chatService.getMessageHistory(roomId);

    return { event: 'room_joined', data: { roomId, history } };
  }

  @UseGuards(JwtWsGuard)
  @SubscribeMessage('send_message')
  async handleMessage(
    @ConnectedSocket() client: Socket,
    @MessageBody() dto: CreateMessageDto
  ): Promise<WsResponse<any>> {
    const message = await this.chatService.createMessage({
      ...dto,
      userId: client.data.userId,
      username: client.data.username,
    });

    // Broadcast ไปยัง room
    this.server.to(dto.roomId).emit('new_message', message);

    return { event: 'message_sent', data: { messageId: message.id } };
  }

  @UseGuards(JwtWsGuard)
  @SubscribeMessage('typing')
  handleTyping(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { roomId: string; isTyping: boolean }
  ): void {
    client.to(data.roomId).emit('user_typing', {
      userId: client.data.userId,
      username: client.data.username,
      isTyping: data.isTyping,
    });
  }

  // Send message to specific user
  sendToUser(userId: string, event: string, data: any): void {
    this.server.to(`user:${userId}`).emit(event, data);
  }

  // Broadcast to all
  broadcast(event: string, data: any): void {
    this.server.emit(event, data);
  }
}
```

### WsGuard

```typescript
// src/guards/jwt-ws.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { verify } from 'jsonwebtoken';
import { Socket } from 'socket.io';
import { WsException } from '@nestjs/websockets';

@Injectable()
export class JwtWsGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const client: Socket = context.switchToWs().getClient();
    const token =
      client.handshake.auth.token ||
      client.handshake.headers['authorization']?.replace('Bearer ', '');

    if (!token) {
      throw new WsException('Authentication required');
    }

    try {
      const decoded = verify(token, process.env.JWT_SECRET || 'secret') as any;
      client.data.userId = decoded.userId;
      client.data.username = decoded.username;
      client.data.role = decoded.role;
      return true;
    } catch {
      throw new WsException('Invalid token');
    }
  }
}
```

---

## 42.7 Chat Application

### Chat Service

```typescript
// src/chat/chat.service.ts
import { Injectable } from '@nestjs/common';
import { v4 as uuidv4 } from 'uuid';

export interface Message {
  id: string;
  roomId: string;
  userId: string;
  username: string;
  content: string;
  type: 'text' | 'image' | 'file' | 'system';
  editedAt?: Date;
  deletedAt?: Date;
  replyTo?: string;
  reactions: Map<string, Set<string>>;
  createdAt: Date;
}

export interface Room {
  id: string;
  name: string;
  type: 'channel' | 'direct';
  members: Set<string>;
  createdAt: Date;
}

@Injectable()
export class ChatService {
  private messages = new Map<string, Message[]>(); // roomId -> messages
  private rooms = new Map<string, Room>();

  async createMessage(data: {
    roomId: string;
    userId: string;
    username: string;
    content: string;
    type?: 'text' | 'image' | 'file';
    replyTo?: string;
  }): Promise<Message> {
    const message: Message = {
      id: uuidv4(),
      roomId: data.roomId,
      userId: data.userId,
      username: data.username,
      content: data.content,
      type: data.type || 'text',
      replyTo: data.replyTo,
      reactions: new Map(),
      createdAt: new Date(),
    };

    const roomMessages = this.messages.get(data.roomId) || [];
    roomMessages.push(message);
    this.messages.set(data.roomId, roomMessages);

    return message;
  }

  async getMessageHistory(
    roomId: string,
    limit = 50,
    before?: Date
  ): Promise<Message[]> {
    let messages = this.messages.get(roomId) || [];

    if (before) {
      messages = messages.filter(m => m.createdAt < before);
    }

    return messages
      .filter(m => !m.deletedAt)
      .slice(-limit);
  }

  async editMessage(
    messageId: string,
    userId: string,
    newContent: string
  ): Promise<Message | null> {
    for (const [, messages] of this.messages.entries()) {
      const message = messages.find(m => m.id === messageId);

      if (message) {
        if (message.userId !== userId) {
          throw new Error('ไม่มีสิทธิ์แก้ไขข้อความนี้');
        }

        message.content = newContent;
        message.editedAt = new Date();
        return message;
      }
    }
    return null;
  }

  async deleteMessage(
    messageId: string,
    userId: string,
    isAdmin = false
  ): Promise<boolean> {
    for (const [, messages] of this.messages.entries()) {
      const message = messages.find(m => m.id === messageId);

      if (message) {
        if (!isAdmin && message.userId !== userId) {
          throw new Error('ไม่มีสิทธิ์ลบข้อความนี้');
        }

        message.deletedAt = new Date();
        message.content = 'ข้อความนี้ถูกลบแล้ว';
        return true;
      }
    }
    return false;
  }

  async addReaction(
    messageId: string,
    userId: string,
    emoji: string
  ): Promise<Map<string, Set<string>> | null> {
    for (const [, messages] of this.messages.entries()) {
      const message = messages.find(m => m.id === messageId);

      if (message) {
        if (!message.reactions.has(emoji)) {
          message.reactions.set(emoji, new Set());
        }

        const emojiReactions = message.reactions.get(emoji)!;

        if (emojiReactions.has(userId)) {
          emojiReactions.delete(userId); // Toggle
        } else {
          emojiReactions.add(userId);
        }

        if (emojiReactions.size === 0) {
          message.reactions.delete(emoji);
        }

        return message.reactions;
      }
    }
    return null;
  }

  createRoom(data: {
    id: string;
    name: string;
    type: 'channel' | 'direct';
    members: string[];
  }): Room {
    const room: Room = {
      id: data.id,
      name: data.name,
      type: data.type,
      members: new Set(data.members),
      createdAt: new Date(),
    };

    this.rooms.set(room.id, room);
    return room;
  }
}
```

---

## 42.8 Live Notifications

```typescript
// src/notifications/notification.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  OnGatewayConnection,
  OnGatewayDisconnect,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { Injectable } from '@nestjs/common';

export interface Notification {
  id: string;
  userId: string;
  type: string;
  title: string;
  message: string;
  data?: any;
  read: boolean;
  createdAt: Date;
}

@WebSocketGateway({ namespace: '/notifications' })
@Injectable()
export class NotificationGateway
  implements OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  private server!: Server;

  private userSockets = new Map<string, Set<string>>(); // userId -> socketIds

  handleConnection(client: Socket) {
    const userId = client.data.userId;
    if (!userId) return;

    if (!this.userSockets.has(userId)) {
      this.userSockets.set(userId, new Set());
    }
    this.userSockets.get(userId)!.add(client.id);

    // Auto-join user room
    client.join(`user:${userId}`);
    console.log(`User ${userId} connected to notifications`);
  }

  handleDisconnect(client: Socket) {
    const userId = client.data.userId;
    if (userId) {
      const sockets = this.userSockets.get(userId);
      sockets?.delete(client.id);

      if (sockets?.size === 0) {
        this.userSockets.delete(userId);
      }
    }
  }

  sendToUser(userId: string, notification: Omit<Notification, 'userId'>): void {
    this.server.to(`user:${userId}`).emit('notification', {
      ...notification,
      userId,
    });
  }

  sendToUsers(userIds: string[], notification: Omit<Notification, 'userId'>): void {
    userIds.forEach(userId => this.sendToUser(userId, notification));
  }

  broadcast(notification: Omit<Notification, 'userId'>): void {
    this.server.emit('broadcast_notification', notification);
  }

  getOnlineUsers(): string[] {
    return Array.from(this.userSockets.keys());
  }

  isUserOnline(userId: string): boolean {
    const sockets = this.userSockets.get(userId);
    return (sockets?.size ?? 0) > 0;
  }
}
```

---

## 42.9 Collaborative Editing พื้นฐาน

```typescript
// src/collab/collaborative.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  ConnectedSocket,
  MessageBody,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

export interface Operation {
  type: 'insert' | 'delete' | 'retain';
  position: number;
  content?: string;
  length?: number;
  userId: string;
  timestamp: number;
  version: number;
}

export interface DocumentState {
  content: string;
  version: number;
  operations: Operation[];
  collaborators: Set<string>;
}

@WebSocketGateway({ namespace: '/collab' })
export class CollaborativeGateway {
  @WebSocketServer()
  server!: Server;

  private documents = new Map<string, DocumentState>();

  @SubscribeMessage('join_document')
  async handleJoinDocument(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { documentId: string }
  ) {
    const { documentId } = data;

    // สร้าง document ถ้ายังไม่มี
    if (!this.documents.has(documentId)) {
      this.documents.set(documentId, {
        content: '',
        version: 0,
        operations: [],
        collaborators: new Set(),
      });
    }

    const doc = this.documents.get(documentId)!;
    doc.collaborators.add(client.data.userId);

    await client.join(`doc:${documentId}`);

    // แจ้ง collaborators คนอื่น
    client.to(`doc:${documentId}`).emit('collaborator_joined', {
      userId: client.data.userId,
      username: client.data.username,
    });

    // ส่ง current state ให้ client
    return {
      event: 'document_state',
      data: {
        content: doc.content,
        version: doc.version,
        collaborators: Array.from(doc.collaborators),
      },
    };
  }

  @SubscribeMessage('apply_operation')
  handleOperation(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { documentId: string; operation: Operation }
  ) {
    const { documentId, operation } = data;
    const doc = this.documents.get(documentId);

    if (!doc) return { event: 'error', data: 'Document not found' };

    // Apply operation
    try {
      doc.content = this.applyOperation(doc.content, operation);
      doc.version++;
      operation.version = doc.version;
      doc.operations.push(operation);

      // Broadcast ไปยัง collaborators อื่น
      client.to(`doc:${documentId}`).emit('operation_applied', {
        operation,
        version: doc.version,
      });

      return {
        event: 'operation_confirmed',
        data: { version: doc.version },
      };
    } catch (error: any) {
      return { event: 'operation_rejected', data: { reason: error.message } };
    }
  }

  @SubscribeMessage('cursor_position')
  handleCursorPosition(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { documentId: string; position: number; selection?: { start: number; end: number } }
  ) {
    client.to(`doc:${data.documentId}`).emit('cursor_updated', {
      userId: client.data.userId,
      username: client.data.username,
      position: data.position,
      selection: data.selection,
    });
  }

  private applyOperation(content: string, op: Operation): string {
    switch (op.type) {
      case 'insert':
        return (
          content.substring(0, op.position) +
          (op.content || '') +
          content.substring(op.position)
        );

      case 'delete':
        return (
          content.substring(0, op.position) +
          content.substring(op.position + (op.length || 0))
        );

      case 'retain':
        return content;

      default:
        throw new Error(`Unknown operation type: ${op.type}`);
    }
  }
}
```

---

## 42.10 Broadcasting Patterns

```typescript
// src/broadcasting/broadcast.service.ts
import { Injectable } from '@nestjs/common';
import { Server } from 'socket.io';
import { InjectServer } from './inject-server.decorator';

@Injectable()
export class BroadcastService {
  constructor(@InjectServer() private io: Server) {}

  // Broadcast ไปทุก client
  toAll(event: string, data: any): void {
    this.io.emit(event, data);
  }

  // Broadcast ไป room
  toRoom(room: string, event: string, data: any): void {
    this.io.to(room).emit(event, data);
  }

  // Broadcast ไป user (ทุก device)
  toUser(userId: string, event: string, data: any): void {
    this.io.to(`user:${userId}`).emit(event, data);
  }

  // Broadcast ไป users หลายคน
  toUsers(userIds: string[], event: string, data: any): void {
    userIds.forEach(id => this.toUser(id, event, data));
  }

  // Broadcast ไปทุกคนยกเว้น socket นี้
  toOthers(socketId: string, event: string, data: any): void {
    this.io.except(socketId).emit(event, data);
  }

  // Broadcast ไป rooms ยกเว้น rooms บางอัน
  toRoomsExcept(
    rooms: string[],
    excludeRooms: string[],
    event: string,
    data: any
  ): void {
    this.io.to(rooms).except(excludeRooms).emit(event, data);
  }

  // Delayed broadcast
  async scheduledBroadcast(
    event: string,
    data: any,
    delayMs: number
  ): Promise<void> {
    await new Promise(resolve => setTimeout(resolve, delayMs));
    this.toAll(event, data);
  }

  // Broadcast พร้อม acknowledgment
  async broadcastWithAck(
    room: string,
    event: string,
    data: any,
    timeout = 5000
  ): Promise<any[]> {
    const responses = await this.io
      .timeout(timeout)
      .to(room)
      .emitWithAck(event, data);

    return responses;
  }
}
```

### Event Throttling

```typescript
// src/utils/throttle.ts
export function throttleEvent<T extends (...args: any[]) => any>(
  fn: T,
  delayMs: number
): T {
  let lastCall = 0;
  let timeout: NodeJS.Timeout | null = null;
  let lastArgs: any[] | null = null;

  return function (this: any, ...args: any[]) {
    const now = Date.now();

    if (now - lastCall >= delayMs) {
      lastCall = now;
      return fn.apply(this, args);
    }

    lastArgs = args;
    if (!timeout) {
      timeout = setTimeout(() => {
        lastCall = Date.now();
        timeout = null;
        if (lastArgs) {
          fn.apply(this, lastArgs);
          lastArgs = null;
        }
      }, delayMs - (now - lastCall));
    }
  } as T;
}

// การใช้งานกับ Socket.IO
const throttledTyping = throttleEvent(
  (socket: Socket, roomId: string) => {
    socket.to(roomId).emit('user_typing', {
      userId: socket.data.userId,
      isTyping: true,
    });
  },
  500
);
```

---

## 42.11 Socket.IO Client TypeScript

```typescript
// client/socket-client.ts
import { io, Socket } from 'socket.io-client';
import {
  ServerToClientEvents,
  ClientToServerEvents,
  ChatMessage,
} from '../shared/socket-events';

type TypedSocket = Socket<ServerToClientEvents, ClientToServerEvents>;

class ChatClient {
  private socket: TypedSocket;
  private currentRoom: string | null = null;

  constructor(serverUrl: string, token: string) {
    this.socket = io(`${serverUrl}/chat`, {
      auth: { token },
      transports: ['websocket'],
      reconnection: true,
      reconnectionAttempts: 5,
      reconnectionDelay: 1000,
    });

    this.setupEventHandlers();
  }

  private setupEventHandlers(): void {
    this.socket.on('connect', () => {
      console.log('Connected to chat server');
    });

    this.socket.on('disconnect', (reason) => {
      console.log(`Disconnected: ${reason}`);
      if (reason === 'io server disconnect') {
        this.socket.connect();
      }
    });

    this.socket.on('connect_error', (error) => {
      console.error('Connection error:', error.message);
    });

    this.socket.on('chat_message', (message) => {
      this.handleIncomingMessage(message);
    });

    this.socket.on('user_joined', (data) => {
      console.log(`${data.username} เข้าร่วมห้อง ${data.roomId}`);
    });

    this.socket.on('user_left', (data) => {
      console.log(`${data.username} ออกจากห้อง ${data.roomId}`);
    });

    this.socket.on('user_typing', (data) => {
      if (data.isTyping) {
        console.log(`${data.username} กำลังพิมพ์...`);
      }
    });
  }

  async joinRoom(roomId: string): Promise<void> {
    return new Promise((resolve, reject) => {
      this.socket.emit('join_room', roomId, (error, data) => {
        if (error) {
          reject(new Error(error));
        } else {
          this.currentRoom = roomId;
          console.log(`Joined room: ${roomId}`);
          resolve();
        }
      });
    });
  }

  sendMessage(content: string): Promise<void> {
    if (!this.currentRoom) {
      return Promise.reject(new Error('ยังไม่ได้เข้าร่วมห้อง'));
    }

    return new Promise((resolve, reject) => {
      this.socket.emit(
        'send_message',
        { roomId: this.currentRoom!, content },
        (error) => {
          if (error) {
            reject(new Error(error));
          } else {
            resolve();
          }
        }
      );
    });
  }

  startTyping(): void {
    if (this.currentRoom) {
      this.socket.emit('typing', {
        roomId: this.currentRoom,
        userId: '',
        username: '',
        isTyping: true,
      });
    }
  }

  stopTyping(): void {
    if (this.currentRoom) {
      this.socket.emit('typing', {
        roomId: this.currentRoom,
        userId: '',
        username: '',
        isTyping: false,
      });
    }
  }

  private handleIncomingMessage(message: ChatMessage): void {
    console.log(`[${message.username}]: ${message.content}`);
    // Update UI here
  }

  disconnect(): void {
    this.socket.disconnect();
  }
}

// React Hook สำหรับ Socket.IO
import { useEffect, useRef, useState, useCallback } from 'react';

export function useChatSocket(token: string) {
  const socketRef = useRef<TypedSocket | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [typingUsers, setTypingUsers] = useState<Map<string, boolean>>(new Map());

  useEffect(() => {
    const socket: TypedSocket = io('/chat', {
      auth: { token },
    });

    socketRef.current = socket;

    socket.on('connect', () => setIsConnected(true));
    socket.on('disconnect', () => setIsConnected(false));

    socket.on('chat_message', (message) => {
      setMessages(prev => [...prev, message]);
    });

    socket.on('user_typing', ({ userId, username, isTyping }) => {
      setTypingUsers(prev => {
        const next = new Map(prev);
        if (isTyping) {
          next.set(userId, true);
        } else {
          next.delete(userId);
        }
        return next;
      });
    });

    return () => {
      socket.disconnect();
    };
  }, [token]);

  const sendMessage = useCallback(
    (roomId: string, content: string) => {
      socketRef.current?.emit(
        'send_message',
        { roomId, content },
        (error) => {
          if (error) console.error('Send failed:', error);
        }
      );
    },
    []
  );

  const joinRoom = useCallback(
    (roomId: string) => {
      socketRef.current?.emit('join_room', roomId, (error) => {
        if (error) console.error('Join failed:', error);
      });
    },
    []
  );

  return { isConnected, messages, typingUsers, sendMessage, joinRoom };
}
```

---

## 42.12 Scaling กับ Redis Adapter

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

async function createRedisAdapter() {
  const pubClient = createClient({ url: process.env.REDIS_URL });
  const subClient = pubClient.duplicate();

  await Promise.all([pubClient.connect(), subClient.connect()]);

  return createAdapter(pubClient, subClient);
}

// gateway-module.ts
import { SocketIoOptions } from '@nestjs/platform-socket.io';

export const socketOptions: SocketIoOptions = {
  cors: {
    origin: '*',
  },
  // ใช้ Redis Adapter สำหรับ horizontal scaling
  adapter: createRedisAdapter as any,
};
```

---

## สรุป

WebSockets และ Real-time Communication ด้วย TypeScript:

1. **ws Library** - Low-level WebSocket สำหรับ Node.js
2. **Socket.IO** - Full-featured real-time library พร้อม fallback
3. **Typed Events** - Type safety สำหรับ events ทั้ง server และ client
4. **NestJS Gateways** - Integration กับ NestJS framework
5. **Chat Application** - ตัวอย่างแอปพลิเคชัน chat สมบูรณ์
6. **Broadcasting** - Pattern ต่างๆ สำหรับส่ง events
7. **Scaling** - Redis Adapter สำหรับ horizontal scaling
8. **Authentication** - JWT-based WebSocket auth

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ TypeScript Tooling และ Configuration
