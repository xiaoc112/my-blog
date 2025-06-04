# WebSocket 实时通信指南

基于 nest-admin 项目的 WebSocket 实时通信最佳实践

## 📚 目录

1. [WebSocket配置](#websocket配置)
2. [Socket.IO 集成](#socketio集成)
3. [Redis 适配器](#redis适配器)
4. [网关设计](#网关设计)
5. [事件处理](#事件处理)
6. [身份认证](#身份认证)
7. [消息广播](#消息广播)
8. [错误处理](#错误处理)
9. [性能优化](#性能优化)

## ⚙️ WebSocket配置

### 1. Socket.IO 适配器配置

```typescript
// common/adapters/socket.adapter.ts
import { INestApplication } from '@nestjs/common'
import { IoAdapter } from '@nestjs/platform-socket.io'
import { createAdapter } from '@socket.io/redis-adapter'
import { Server, ServerOptions } from 'socket.io'

import { REDIS_PUBSUB } from '~/shared/redis/redis.constant'

export const RedisIoAdapterKey = 'nest-admin-socket'

export class RedisIoAdapter extends IoAdapter {
  constructor(private readonly app: INestApplication) {
    super(app)
  }

  createIOServer(port: number, options?: ServerOptions): Server {
    // 创建 Socket.IO 服务器
    const server = super.createIOServer(port, {
      ...options,
      cors: {
        origin: '*',
        methods: ['GET', 'POST'],
        credentials: true,
      },
      allowEIO3: true,
      transports: ['websocket', 'polling'],
      pingTimeout: 60000,
      pingInterval: 25000,
    })

    // 获取 Redis 发布订阅客户端
    const { pubClient, subClient } = this.app.get(REDIS_PUBSUB)

    // 创建 Redis 适配器
    const redisAdapter = createAdapter(pubClient, subClient, {
      key: RedisIoAdapterKey,
      requestsTimeout: 10000,
    })

    // 设置 Redis 适配器
    server.adapter(redisAdapter)

    return server
  }
}
```

### 2. 主应用配置

```typescript
// main.ts - 配置 WebSocket 适配器
import { RedisIoAdapter } from './common/adapters/socket.adapter'

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    fastifyApp,
  )

  // ... 其他配置

  // 设置 WebSocket 适配器
  app.useWebSocketAdapter(new RedisIoAdapter(app))

  await app.listen(port, '0.0.0.0')
}
```

### 3. Socket 模块配置

```typescript
// socket/socket.module.ts
import { forwardRef, Module, Provider } from '@nestjs/common'
import { AuthModule } from '../modules/auth/auth.module'
import { SystemModule } from '../modules/system/system.module'

import { AdminEventsGateway } from './events/admin.gateway'
import { WebEventsGateway } from './events/web.gateway'
import { SocketAuthMiddleware } from './middleware/socket-auth.middleware'
import { SocketService } from './socket.service'

const providers: Provider[] = [
  AdminEventsGateway,
  WebEventsGateway,
  SocketService,
  SocketAuthMiddleware,
]

@Module({
  imports: [forwardRef(() => SystemModule), AuthModule],
  providers,
  exports: [...providers],
})
export class SocketModule {}
```

## 🔌 Socket.IO 集成

### 1. 基础网关实现

```typescript
import { Logger, UseGuards, UsePipes, ValidationPipe } from '@nestjs/common'
import { JwtService } from '@nestjs/jwt'
// socket/events/admin.gateway.ts
import {
  ConnectedSocket,
  MessageBody,
  OnGatewayConnection,
  OnGatewayDisconnect,
  OnGatewayInit,
  SubscribeMessage,
  WebSocketGateway,
  WebSocketServer,
} from '@nestjs/websockets'
import { Server, Socket } from 'socket.io'

import { JwtAuthGuard } from '~/modules/auth/guards/jwt-auth.guard'
import { AdminSocketAuthGuard } from '../guards/admin-socket-auth.guard'
import { SocketService } from '../socket.service'

@WebSocketGateway({
  namespace: '/admin',
  cors: {
    origin: '*',
    credentials: true,
  },
  transports: ['websocket', 'polling'],
})
@UseGuards(AdminSocketAuthGuard)
@UsePipes(new ValidationPipe({ transform: true }))
export class AdminEventsGateway
implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server

  private readonly logger = new Logger(AdminEventsGateway.name)

  constructor(
    private readonly socketService: SocketService,
    private readonly jwtService: JwtService,
  ) {}

  // 网关初始化
  afterInit(server: Server) {
    this.logger.log('Admin WebSocket Gateway 初始化完成')
    this.socketService.setAdminServer(server)
  }

  // 客户端连接
  async handleConnection(client: Socket) {
    try {
      // 验证 Token
      const token = this.extractTokenFromClient(client)
      if (!token) {
        this.logger.warn(`连接被拒绝 - 缺少令牌: ${client.id}`)
        client.disconnect()
        return
      }

      // 验证用户身份
      const user = await this.validateUser(token)
      if (!user) {
        this.logger.warn(`连接被拒绝 - 无效令牌: ${client.id}`)
        client.disconnect()
        return
      }

      // 存储用户信息到 socket
      client.data.user = user

      // 加入用户房间
      await client.join(`user:${user.id}`)

      // 记录连接
      await this.socketService.addConnection(client.id, user.id, 'admin')

      this.logger.log(`管理员用户 ${user.username} 已连接: ${client.id}`)

      // 发送连接成功消息
      client.emit('connected', {
        message: '连接成功',
        user: { id: user.id, username: user.username },
      })

      // 广播用户上线通知
      this.server.emit('user:online', {
        userId: user.id,
        username: user.username,
        timestamp: new Date(),
      })
    }
    catch (error) {
      this.logger.error(`连接处理错误: ${error.message}`)
      client.disconnect()
    }
  }

  // 客户端断开连接
  async handleDisconnect(client: Socket) {
    const user = client.data.user

    if (user) {
      // 移除连接记录
      await this.socketService.removeConnection(client.id)

      this.logger.log(`管理员用户 ${user.username} 已断开连接: ${client.id}`)

      // 广播用户离线通知
      this.server.emit('user:offline', {
        userId: user.id,
        username: user.username,
        timestamp: new Date(),
      })
    }
  }

  // 提取客户端令牌
  private extractTokenFromClient(client: Socket): string | null {
    const token = client.handshake.auth?.token
      || client.handshake.headers?.authorization?.replace('Bearer ', '')
    return token || null
  }

  // 验证用户
  private async validateUser(token: string) {
    try {
      const payload = this.jwtService.verify(token)
      // 这里可以添加更多验证逻辑，比如检查用户是否仍然有效
      return payload
    }
    catch {
      return null
    }
  }

  // 处理心跳
  @SubscribeMessage('ping')
  handlePing(@ConnectedSocket() client: Socket): string {
    return 'pong'
  }

  // 处理私聊消息
  @SubscribeMessage('private:message')
  async handlePrivateMessage(
    @MessageBody() data: { toUserId: number, message: string },
    @ConnectedSocket() client: Socket,
  ) {
    const fromUser = client.data.user

    // 发送给目标用户
    this.server.to(`user:${data.toUserId}`).emit('private:message', {
      fromUserId: fromUser.id,
      fromUsername: fromUser.username,
      message: data.message,
      timestamp: new Date(),
    })

    // 确认发送
    client.emit('message:sent', { messageId: Date.now() })
  }

  // 处理加入房间
  @SubscribeMessage('room:join')
  async handleJoinRoom(
    @MessageBody() data: { roomId: string },
    @ConnectedSocket() client: Socket,
  ) {
    await client.join(data.roomId)
    client.emit('room:joined', { roomId: data.roomId })

    // 通知房间其他用户
    client.to(data.roomId).emit('room:user:joined', {
      userId: client.data.user.id,
      username: client.data.user.username,
    })
  }

  // 处理离开房间
  @SubscribeMessage('room:leave')
  async handleLeaveRoom(
    @MessageBody() data: { roomId: string },
    @ConnectedSocket() client: Socket,
  ) {
    await client.leave(data.roomId)
    client.emit('room:left', { roomId: data.roomId })

    // 通知房间其他用户
    client.to(data.roomId).emit('room:user:left', {
      userId: client.data.user.id,
      username: client.data.user.username,
    })
  }

  // 处理房间消息
  @SubscribeMessage('room:message')
  async handleRoomMessage(
    @MessageBody() data: { roomId: string, message: string },
    @ConnectedSocket() client: Socket,
  ) {
    const user = client.data.user

    // 发送给房间所有用户
    this.server.to(data.roomId).emit('room:message', {
      roomId: data.roomId,
      fromUserId: user.id,
      fromUsername: user.username,
      message: data.message,
      timestamp: new Date(),
    })
  }
}
```

### 2. Socket 服务实现

```typescript
// socket/socket.service.ts
import { Injectable, Logger } from '@nestjs/common'
import { Server } from 'socket.io'
import { RedisService } from '~/shared/redis/redis.service'

@Injectable()
export class SocketService {
  private adminServer: Server
  private webServer: Server
  private readonly logger = new Logger(SocketService.name)

  constructor(private readonly redisService: RedisService) {}

  setAdminServer(server: Server) {
    this.adminServer = server
  }

  setWebServer(server: Server) {
    this.webServer = server
  }

  // 连接管理
  async addConnection(socketId: string, userId: number, namespace: string) {
    const connectionKey = `socket:connections:${namespace}:${userId}`
    await this.redisService.sadd(connectionKey, socketId)
    await this.redisService.expire(connectionKey, 86400) // 24小时过期

    // 记录 socket 到用户的映射
    const socketKey = `socket:user:${socketId}`
    await this.redisService.setObject(socketKey, { userId, namespace }, 86400)
  }

  async removeConnection(socketId: string) {
    // 获取用户信息
    const socketKey = `socket:user:${socketId}`
    const socketData = await this.redisService.getObject<{
      userId: number
      namespace: string
    }>(socketKey)

    if (socketData) {
      const connectionKey = `socket:connections:${socketData.namespace}:${socketData.userId}`
      await this.redisService.srem(connectionKey, socketId)
      await this.redisService.del(socketKey)
    }
  }

  // 获取用户的所有连接
  async getUserConnections(userId: number, namespace: string): Promise<string[]> {
    const connectionKey = `socket:connections:${namespace}:${userId}`
    return await this.redisService.smembers(connectionKey)
  }

  // 检查用户是否在线
  async isUserOnline(userId: number, namespace: string = 'admin'): Promise<boolean> {
    const connections = await this.getUserConnections(userId, namespace)
    return connections.length > 0
  }

  // 获取在线用户列表
  async getOnlineUsers(namespace: string = 'admin'): Promise<number[]> {
    const pattern = `socket:connections:${namespace}:*`
    const keys = await this.redisService.keys(pattern)

    const userIds: number[] = []
    for (const key of keys) {
      const userId = Number.parseInt(key.split(':').pop()!)
      const connections = await this.redisService.smembers(key)
      if (connections.length > 0) {
        userIds.push(userId)
      }
    }

    return userIds
  }

  // 发送消息给特定用户
  async sendToUser(userId: number, event: string, data: any, namespace: string = 'admin') {
    const server = namespace === 'admin' ? this.adminServer : this.webServer
    if (!server) {
      this.logger.warn(`服务器未初始化: ${namespace}`)
      return
    }

    server.to(`user:${userId}`).emit(event, data)
  }

  // 发送消息给所有用户
  async sendToAll(event: string, data: any, namespace: string = 'admin') {
    const server = namespace === 'admin' ? this.adminServer : this.webServer
    if (!server) {
      this.logger.warn(`服务器未初始化: ${namespace}`)
      return
    }

    server.emit(event, data)
  }

  // 发送消息给房间
  async sendToRoom(roomId: string, event: string, data: any, namespace: string = 'admin') {
    const server = namespace === 'admin' ? this.adminServer : this.webServer
    if (!server) {
      this.logger.warn(`服务器未初始化: ${namespace}`)
      return
    }

    server.to(roomId).emit(event, data)
  }

  // 强制断开用户连接
  async disconnectUser(userId: number, namespace: string = 'admin') {
    const connections = await this.getUserConnections(userId, namespace)
    const server = namespace === 'admin' ? this.adminServer : this.webServer

    if (!server)
      return

    connections.forEach((socketId) => {
      const socket = server.sockets.sockets.get(socketId)
      if (socket) {
        socket.disconnect(true)
      }
    })
  }
}
```

## 🔐 身份认证

### 1. Socket 认证守卫

```typescript
// socket/guards/admin-socket-auth.guard.ts
import { CanActivate, ExecutionContext, Injectable, Logger } from '@nestjs/common'
import { JwtService } from '@nestjs/jwt'
import { WsException } from '@nestjs/websockets'
import { Socket } from 'socket.io'

@Injectable()
export class AdminSocketAuthGuard implements CanActivate {
  private readonly logger = new Logger(AdminSocketAuthGuard.name)

  constructor(private readonly jwtService: JwtService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    try {
      const client: Socket = context.switchToWs().getClient()

      // 提取令牌
      const token = this.extractToken(client)
      if (!token) {
        throw new WsException('缺少认证令牌')
      }

      // 验证令牌
      const payload = this.jwtService.verify(token)

      // 将用户信息附加到 socket
      client.data.user = payload

      return true
    }
    catch (error) {
      this.logger.error(`Socket 认证失败: ${error.message}`)
      throw new WsException('认证失败')
    }
  }

  private extractToken(client: Socket): string | null {
    return (
      client.handshake.auth?.token
      || client.handshake.headers?.authorization?.replace('Bearer ', '')
      || null
    )
  }
}
```

### 2. Socket 认证中间件

```typescript
// socket/middleware/socket-auth.middleware.ts
import { Injectable, Logger } from '@nestjs/common'
import { JwtService } from '@nestjs/jwt'
import { Socket } from 'socket.io'

@Injectable()
export class SocketAuthMiddleware {
  private readonly logger = new Logger(SocketAuthMiddleware.name)

  constructor(private readonly jwtService: JwtService) {}

  async use(socket: Socket, next: (err?: any) => void) {
    try {
      // 提取令牌
      const token = this.extractToken(socket)

      if (!token) {
        return next(new Error('认证令牌缺失'))
      }

      // 验证令牌
      const payload = this.jwtService.verify(token)

      // 检查用户权限（可选）
      const hasPermission = await this.checkUserPermission(payload)
      if (!hasPermission) {
        return next(new Error('权限不足'))
      }

      // 附加用户信息
      socket.data.user = payload

      this.logger.log(`用户 ${payload.username} 认证成功`)
      next()
    }
    catch (error) {
      this.logger.error(`Socket 认证中间件错误: ${error.message}`)
      next(new Error('认证失败'))
    }
  }

  private extractToken(socket: Socket): string | null {
    return (
      socket.handshake.auth?.token
      || socket.handshake.headers?.authorization?.replace('Bearer ', '')
      || null
    )
  }

  private async checkUserPermission(user: any): Promise<boolean> {
    // 这里可以添加更复杂的权限检查逻辑
    // 比如检查用户是否被禁用、是否有特定权限等
    return user && user.id && user.status === 1
  }
}
```

## 📡 消息广播

### 1. 系统通知服务

```typescript
// socket/services/notification.service.ts
import { Injectable } from '@nestjs/common'
import { EventEmitter2 } from '@nestjs/event-emitter'
import { SocketService } from '../socket.service'

export interface NotificationData {
  title: string
  message: string
  type: 'info' | 'success' | 'warning' | 'error'
  data?: any
}

@Injectable()
export class NotificationService {
  constructor(
    private readonly socketService: SocketService,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  // 发送系统通知给所有用户
  async sendSystemNotification(notification: NotificationData) {
    await this.socketService.sendToAll('system:notification', {
      ...notification,
      timestamp: new Date(),
    })

    // 发出内部事件
    this.eventEmitter.emit('notification.sent', { type: 'system', notification })
  }

  // 发送通知给特定用户
  async sendUserNotification(userId: number, notification: NotificationData) {
    await this.socketService.sendToUser(userId, 'user:notification', {
      ...notification,
      timestamp: new Date(),
    })

    // 发出内部事件
    this.eventEmitter.emit('notification.sent', {
      type: 'user',
      userId,
      notification
    })
  }

  // 发送通知给多个用户
  async sendMultiUserNotification(userIds: number[], notification: NotificationData) {
    const promises = userIds.map(userId =>
      this.sendUserNotification(userId, notification)
    )

    await Promise.all(promises)
  }

  // 发送消息给房间
  async sendRoomNotification(roomId: string, notification: NotificationData) {
    await this.socketService.sendToRoom(roomId, 'room:notification', {
      ...notification,
      roomId,
      timestamp: new Date(),
    })

    this.eventEmitter.emit('notification.sent', {
      type: 'room',
      roomId,
      notification
    })
  }
}
```

### 2. 实时数据推送

```typescript
// socket/services/realtime-data.service.ts
import { Injectable } from '@nestjs/common'
import { Cron, CronExpression } from '@nestjs/schedule'
import { SocketService } from '../socket.service'

@Injectable()
export class RealtimeDataService {
  constructor(private readonly socketService: SocketService) {}

  // 推送系统状态
  async pushSystemStatus() {
    const systemStatus = await this.getSystemStatus()
    await this.socketService.sendToAll('system:status', systemStatus)
  }

  // 推送在线用户统计
  async pushOnlineUserStats() {
    const onlineUsers = await this.socketService.getOnlineUsers()
    const stats = {
      total: onlineUsers.length,
      users: onlineUsers,
      timestamp: new Date(),
    }

    await this.socketService.sendToAll('stats:online-users', stats)
  }

  // 定时推送数据
  @Cron(CronExpression.EVERY_30_SECONDS)
  async scheduledDataPush() {
    await this.pushSystemStatus()
    await this.pushOnlineUserStats()
  }

  // 推送业务数据更新
  async pushDataUpdate(dataType: string, data: any, targetUsers?: number[]) {
    const payload = {
      type: dataType,
      data,
      timestamp: new Date(),
    }

    if (targetUsers && targetUsers.length > 0) {
      // 推送给特定用户
      const promises = targetUsers.map(userId =>
        this.socketService.sendToUser(userId, 'data:update', payload)
      )
      await Promise.all(promises)
    }
    else {
      // 推送给所有用户
      await this.socketService.sendToAll('data:update', payload)
    }
  }

  private async getSystemStatus() {
    // 获取系统状态信息
    return {
      cpu: await this.getCpuUsage(),
      memory: await this.getMemoryUsage(),
      activeConnections: await this.getActiveConnections(),
      timestamp: new Date(),
    }
  }

  private async getCpuUsage(): Promise<number> {
    // 实现CPU使用率获取逻辑
    return Math.random() * 100
  }

  private async getMemoryUsage(): Promise<number> {
    // 实现内存使用率获取逻辑
    return Math.random() * 100
  }

  private async getActiveConnections(): Promise<number> {
    const onlineUsers = await this.socketService.getOnlineUsers()
    return onlineUsers.length
  }
}
```

## 🎯 事件处理

### 1. Socket 事件监听器

```typescript
// socket/listeners/socket-event.listener.ts
import { Injectable } from '@nestjs/common'
import { OnEvent } from '@nestjs/event-emitter'
import { NotificationService } from '../services/notification.service'
import { RealtimeDataService } from '../services/realtime-data.service'

@Injectable()
export class SocketEventListener {
  constructor(
    private readonly notificationService: NotificationService,
    private readonly realtimeDataService: RealtimeDataService,
  ) {}

  // 用户创建事件
  @OnEvent('user.created')
  async handleUserCreated(payload: { user: any }) {
    await this.notificationService.sendSystemNotification({
      title: '新用户注册',
      message: `用户 ${payload.user.username} 已成功注册`,
      type: 'info',
      data: { userId: payload.user.id },
    })
  }

  // 用户登录事件
  @OnEvent('user.login')
  async handleUserLogin(payload: { user: any, ip: string }) {
    // 通知管理员
    await this.notificationService.sendSystemNotification({
      title: '用户登录',
      message: `用户 ${payload.user.username} 从 ${payload.ip} 登录`,
      type: 'info',
      data: { userId: payload.user.id, ip: payload.ip },
    })

    // 推送在线用户统计更新
    await this.realtimeDataService.pushOnlineUserStats()
  }

  // 数据更新事件
  @OnEvent('data.updated')
  async handleDataUpdated(payload: {
    type: string
    data: any
    targetUsers?: number[]
  }) {
    await this.realtimeDataService.pushDataUpdate(
      payload.type,
      payload.data,
      payload.targetUsers,
    )
  }

  // 系统警告事件
  @OnEvent('system.warning')
  async handleSystemWarning(payload: {
    message: string
    level: 'low' | 'medium' | 'high'
    data?: any
  }) {
    await this.notificationService.sendSystemNotification({
      title: '系统警告',
      message: payload.message,
      type: 'warning',
      data: payload.data,
    })
  }

  // 系统错误事件
  @OnEvent('system.error')
  async handleSystemError(payload: {
    message: string
    error?: any
  }) {
    await this.notificationService.sendSystemNotification({
      title: '系统错误',
      message: payload.message,
      type: 'error',
      data: payload.error,
    })
  }
}
```

### 2. 业务事件集成

```typescript
// modules/user/user.service.ts
import { Injectable } from '@nestjs/common'
import { EventEmitter2 } from '@nestjs/event-emitter'
import { SocketService } from '~/socket/socket.service'

@Injectable()
export class UserService {
  constructor(
    private readonly eventEmitter: EventEmitter2,
    private readonly socketService: SocketService,
  ) {}

  async createUser(userData: CreateUserDto): Promise<UserEntity> {
    const user = await this.userRepository.save(userData)

    // 发出用户创建事件
    this.eventEmitter.emit('user.created', { user })

    // 实时推送新用户信息
    await this.socketService.sendToAll('user:created', {
      user: {
        id: user.id,
        username: user.username,
        createdAt: user.createdAt,
      },
    })

    return user
  }

  async updateUser(userId: number, updateData: UpdateUserDto): Promise<UserEntity> {
    const user = await this.userRepository.save({ id: userId, ...updateData })

    // 发出用户更新事件
    this.eventEmitter.emit('user.updated', { userId, updateData })

    // 通知该用户信息已更新
    await this.socketService.sendToUser(userId, 'user:profile:updated', {
      user: {
        id: user.id,
        username: user.username,
        updatedAt: user.updatedAt,
      },
    })

    return user
  }

  async deleteUser(userId: number): Promise<void> {
    await this.userRepository.delete(userId)

    // 发出用户删除事件
    this.eventEmitter.emit('user.deleted', { userId })

    // 强制断开该用户的连接
    await this.socketService.disconnectUser(userId)

    // 通知其他用户
    await this.socketService.sendToAll('user:deleted', { userId })
  }
}
```

## ⚠️ 错误处理

### 1. WebSocket 异常过滤器

```typescript
// socket/filters/ws-exception.filter.ts
import { ArgumentsHost, Catch, Logger } from '@nestjs/common'
import { BaseWsExceptionFilter, WsException } from '@nestjs/websockets'
import { Socket } from 'socket.io'

@Catch()
export class WsAllExceptionsFilter extends BaseWsExceptionFilter {
  private readonly logger = new Logger(WsAllExceptionsFilter.name)

  catch(exception: unknown, host: ArgumentsHost) {
    const client: Socket = host.switchToWs().getClient()
    const data = host.switchToWs().getData()

    this.logger.error(
      `WebSocket异常 - 客户端: ${client.id}, 数据: ${JSON.stringify(data)}`,
      exception instanceof Error ? exception.stack : String(exception),
    )

    if (exception instanceof WsException) {
      // 处理已知的 WebSocket 异常
      client.emit('error', {
        message: exception.message,
        timestamp: new Date(),
      })
    }
    else {
      // 处理未知异常
      client.emit('error', {
        message: '服务器内部错误',
        timestamp: new Date(),
      })
    }
  }
}
```

### 2. 连接错误处理

```typescript
// socket/events/admin.gateway.ts 中的错误处理示例
export class AdminEventsGateway {
  // ... 其他代码

  @SubscribeMessage('test:error')
  handleTestError(@ConnectedSocket() client: Socket) {
    try {
      // 模拟错误
      throw new Error('测试错误')
    }
    catch (error) {
      this.logger.error(`处理测试错误: ${error.message}`)

      // 发送错误响应
      client.emit('error', {
        event: 'test:error',
        message: error.message,
        timestamp: new Date(),
      })
    }
  }

  // 处理无效的消息格式
  @SubscribeMessage('*')
  handleInvalidMessage(@MessageBody() data: any, @ConnectedSocket() client: Socket) {
    if (!this.validateMessageFormat(data)) {
      client.emit('error', {
        message: '无效的消息格式',
        receivedData: data,
        timestamp: new Date(),
      })
    }
  }

  private validateMessageFormat(data: any): boolean {
    // 实现消息格式验证逻辑
    return data && typeof data === 'object'
  }
}
```

## ⚡ 性能优化

### 1. 连接池管理

```typescript
// socket/services/connection-pool.service.ts
import { Injectable, Logger } from '@nestjs/common'
import { Cron, CronExpression } from '@nestjs/schedule'
import { SocketService } from '../socket.service'

@Injectable()
export class ConnectionPoolService {
  private readonly logger = new Logger(ConnectionPoolService.name)
  private readonly maxConnectionsPerUser = 5 // 每个用户最大连接数

  constructor(private readonly socketService: SocketService) {}

  // 检查连接数限制
  async checkConnectionLimit(userId: number, namespace: string): Promise<boolean> {
    const connections = await this.socketService.getUserConnections(userId, namespace)

    if (connections.length >= this.maxConnectionsPerUser) {
      this.logger.warn(`用户 ${userId} 连接数超限: ${connections.length}`)
      return false
    }

    return true
  }

  // 清理无效连接
  @Cron(CronExpression.EVERY_5_MINUTES)
  async cleanupInvalidConnections() {
    try {
      const adminOnlineUsers = await this.socketService.getOnlineUsers('admin')
      this.logger.log(`管理端在线用户数: ${adminOnlineUsers.length}`)

      // 清理已断开但未正确移除的连接记录
      await this.cleanupStaleConnections('admin')
      await this.cleanupStaleConnections('web')
    }
    catch (error) {
      this.logger.error(`清理连接失败: ${error.message}`)
    }
  }

  private async cleanupStaleConnections(namespace: string) {
    // 实现清理陈旧连接的逻辑
    const pattern = `socket:connections:${namespace}:*`
    const keys = await this.socketService.redisService.keys(pattern)

    for (const key of keys) {
      const connections = await this.socketService.redisService.smembers(key)
      // 检查连接是否仍然有效，移除无效连接
      // 这里需要访问 Socket.IO 服务器实例来验证连接
    }
  }
}
```

### 2. 消息限流

```typescript
// socket/middleware/rate-limit.middleware.ts
import { Injectable, Logger } from '@nestjs/common'
import { Socket } from 'socket.io'
import { RedisService } from '~/shared/redis/redis.service'

@Injectable()
export class SocketRateLimitMiddleware {
  private readonly logger = new Logger(SocketRateLimitMiddleware.name)

  constructor(private readonly redisService: RedisService) {}

  async checkRateLimit(
    socket: Socket,
    event: string,
    limit: number = 10, // 每分钟最大请求数
    window: number = 60, // 时间窗口（秒）
  ): Promise<boolean> {
    const userId = socket.data.user?.id
    if (!userId)
      return true // 未认证用户不限流

    const key = `rate:limit:${userId}:${event}`
    const currentCount = await this.redisService.get(key)

    if (currentCount && Number.parseInt(currentCount) >= limit) {
      this.logger.warn(`用户 ${userId} 事件 ${event} 触发限流`)

      socket.emit('rate:limit:exceeded', {
        event,
        limit,
        window,
        message: '请求过于频繁，请稍后再试',
      })

      return false
    }

    // 增加计数
    if (currentCount) {
      await this.redisService.getCacheClient().incr(key)
    }
    else {
      await this.redisService.set(key, '1', window)
    }

    return true
  }
}
```

### 3. 消息批处理

```typescript
// socket/services/message-batch.service.ts
import { Injectable, Logger } from '@nestjs/common'
import { SocketService } from '../socket.service'

interface BatchMessage {
  userId?: number
  roomId?: string
  event: string
  data: any
  namespace?: string
}

@Injectable()
export class MessageBatchService {
  private readonly logger = new Logger(MessageBatchService.name)
  private readonly batchQueue: BatchMessage[] = []
  private readonly batchSize = 100
  private readonly batchInterval = 1000 // 1秒

  constructor(private readonly socketService: SocketService) {
    // 定时处理批量消息
    setInterval(() => {
      this.processBatch()
    }, this.batchInterval)
  }

  // 添加消息到批处理队列
  addToBatch(message: BatchMessage) {
    this.batchQueue.push(message)

    // 如果达到批处理大小，立即处理
    if (this.batchQueue.length >= this.batchSize) {
      this.processBatch()
    }
  }

  // 处理批量消息
  private async processBatch() {
    if (this.batchQueue.length === 0)
      return

    const messages = this.batchQueue.splice(0, this.batchSize)

    try {
      // 按类型分组处理
      const userMessages = messages.filter(m => m.userId)
      const roomMessages = messages.filter(m => m.roomId)
      const broadcastMessages = messages.filter(m => !m.userId && !m.roomId)

      // 批量发送用户消息
      await this.batchSendUserMessages(userMessages)

      // 批量发送房间消息
      await this.batchSendRoomMessages(roomMessages)

      // 批量广播消息
      await this.batchSendBroadcastMessages(broadcastMessages)

      this.logger.log(`批量处理消息完成: ${messages.length} 条`)
    }
    catch (error) {
      this.logger.error(`批量处理消息失败: ${error.message}`)
    }
  }

  private async batchSendUserMessages(messages: BatchMessage[]) {
    const promises = messages.map(msg =>
      this.socketService.sendToUser(
        msg.userId!,
        msg.event,
        msg.data,
        msg.namespace,
      )
    )

    await Promise.all(promises)
  }

  private async batchSendRoomMessages(messages: BatchMessage[]) {
    const promises = messages.map(msg =>
      this.socketService.sendToRoom(
        msg.roomId!,
        msg.event,
        msg.data,
        msg.namespace,
      )
    )

    await Promise.all(promises)
  }

  private async batchSendBroadcastMessages(messages: BatchMessage[]) {
    const promises = messages.map(msg =>
      this.socketService.sendToAll(
        msg.event,
        msg.data,
        msg.namespace,
      )
    )

    await Promise.all(promises)
  }
}
```

## 🚀 最佳实践总结

### 1. 架构设计原则
- **分离关注点**：将认证、消息处理、错误处理分开
- **可扩展性**：使用 Redis 适配器支持多实例部署
- **可维护性**：清晰的模块结构和事件驱动架构

### 2. 性能优化策略
- **连接管理**：限制单用户连接数，及时清理无效连接
- **消息限流**：防止恶意用户发送大量消息
- **批量处理**：减少网络开销，提高吞吐量

### 3. 安全考虑
- **身份认证**：每个连接都必须通过JWT验证
- **权限控制**：基于用户角色控制WebSocket功能访问
- **输入验证**：验证所有客户端消息格式

### 4. 错误处理
- **全局异常捕获**：统一处理WebSocket异常
- **优雅降级**：连接失败时的重连机制
- **日志记录**：详细记录连接和消息处理日志

### 5. 监控与调试
- **连接状态监控**：实时监控在线用户和连接状态
- **性能指标**：监控消息延迟和处理速度
- **错误追踪**：记录和分析WebSocket相关错误

这份指南涵盖了 nest-admin 项目中 WebSocket 实时通信的核心实现，帮助你构建稳定、高性能的实时应用。
