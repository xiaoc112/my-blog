# Redis 缓存使用指南

基于 nest-admin 项目的 Redis 缓存最佳实践

## 📚 目录

1. [Redis配置](#redis配置)
2. [缓存模块设计](#缓存模块设计)
3. [基础缓存操作](#基础缓存操作)
4. [高级缓存策略](#高级缓存策略)
5. [分布式缓存](#分布式缓存)
6. [缓存更新机制](#缓存更新机制)
7. [性能优化](#性能优化)
8. [监控和调试](#监控和调试)

## ⚙️ Redis配置

### 1. Redis 连接配置

```typescript
// config/redis.config.ts
import { ConfigType, registerAs } from '@nestjs/config'
import { env, envNumber } from '~/global/env'

export const redisRegToken = 'redis'

export const RedisConfig = registerAs(redisRegToken, () => ({
  host: env('REDIS_HOST', '127.0.0.1'),
  port: envNumber('REDIS_PORT', 6379),
  password: env('REDIS_PASSWORD'),
  db: envNumber('REDIS_DB', 0),

  // 连接池配置
  maxRetriesPerRequest: 3,
  retryDelayOnFailover: 100,
  enableReadyCheck: false,
  lazyConnect: true,

  // 集群配置（如果使用 Redis 集群）
  cluster: {
    enableOfflineQueue: false,
    redisOptions: {
      password: env('REDIS_PASSWORD'),
    },
  },

  // 连接选项
  connectTimeout: 10000,
  commandTimeout: 5000,

  // 键名前缀
  keyPrefix: env('REDIS_KEY_PREFIX', 'nest-admin:'),
}))

export type IRedisConfig = ConfigType<typeof RedisConfig>
```

### 2. Redis 模块配置

```typescript
import { RedisModule as NestRedisModule } from '@liaoliaots/nestjs-redis'
// shared/redis/redis.module.ts
import { Global, Module } from '@nestjs/common'
import { ConfigService } from '@nestjs/config'

import { REDIS_CACHE, REDIS_PUBSUB } from './redis.constant'
import { RedisService } from './redis.service'

@Global()
@Module({
  imports: [
    NestRedisModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => {
        const redisConfig = configService.get('redis')

        return {
          config: [
            {
              namespace: REDIS_CACHE,
              host: redisConfig.host,
              port: redisConfig.port,
              password: redisConfig.password,
              db: redisConfig.db,
              keyPrefix: redisConfig.keyPrefix,
            },
            {
              namespace: REDIS_PUBSUB,
              host: redisConfig.host,
              port: redisConfig.port,
              password: redisConfig.password,
              db: redisConfig.db + 1, // 使用不同的数据库
            },
          ],
        }
      },
    }),
  ],
  providers: [RedisService],
  exports: [RedisService],
})
export class RedisModule {}
```

### 3. Redis 常量定义

```typescript
// shared/redis/redis.constant.ts
export const REDIS_CACHE = 'REDIS_CACHE'
export const REDIS_PUBSUB = 'REDIS_PUBSUB'

// 缓存键常量
export const CACHE_KEYS = {
  USER: 'user',
  USER_PROFILE: 'user:profile',
  USER_PERMISSIONS: 'user:permissions',
  ROLE: 'role',
  DEPT: 'dept',
  MENU: 'menu',
  CAPTCHA: 'captcha',
  TOKEN: 'token',
  SESSION: 'session',
  ONLINE_USERS: 'online:users',
  RATE_LIMIT: 'rate:limit',
} as const

// 缓存过期时间常量
export const CACHE_TTL = {
  ONE_MINUTE: 60,
  FIVE_MINUTES: 300,
  TEN_MINUTES: 600,
  THIRTY_MINUTES: 1800,
  ONE_HOUR: 3600,
  ONE_DAY: 86400,
  ONE_WEEK: 604800,
  ONE_MONTH: 2592000,
} as const
```

## 🏗️ 缓存模块设计

### 1. Redis 服务封装

```typescript
import { InjectRedis } from '@liaoliaots/nestjs-redis'
// shared/redis/redis.service.ts
import { Injectable, Logger } from '@nestjs/common'
import Redis from 'ioredis'

import { REDIS_CACHE, REDIS_PUBSUB } from './redis.constant'

@Injectable()
export class RedisService {
  private readonly logger = new Logger(RedisService.name)

  constructor(
    @InjectRedis(REDIS_CACHE) private readonly cacheClient: Redis,
    @InjectRedis(REDIS_PUBSUB) private readonly pubsubClient: Redis,
  ) {}

  // 基础操作
  async get(key: string): Promise<string | null> {
    try {
      return await this.cacheClient.get(key)
    }
    catch (error) {
      this.logger.error(`Redis GET error for key ${key}:`, error)
      return null
    }
  }

  async set(key: string, value: string, ttl?: number): Promise<void> {
    try {
      if (ttl) {
        await this.cacheClient.setex(key, ttl, value)
      }
      else {
        await this.cacheClient.set(key, value)
      }
    }
    catch (error) {
      this.logger.error(`Redis SET error for key ${key}:`, error)
    }
  }

  async del(key: string | string[]): Promise<number> {
    try {
      return await this.cacheClient.del(key)
    }
    catch (error) {
      this.logger.error(`Redis DEL error for key ${key}:`, error)
      return 0
    }
  }

  async exists(key: string): Promise<boolean> {
    try {
      const result = await this.cacheClient.exists(key)
      return result === 1
    }
    catch (error) {
      this.logger.error(`Redis EXISTS error for key ${key}:`, error)
      return false
    }
  }

  async expire(key: string, ttl: number): Promise<boolean> {
    try {
      const result = await this.cacheClient.expire(key, ttl)
      return result === 1
    }
    catch (error) {
      this.logger.error(`Redis EXPIRE error for key ${key}:`, error)
      return false
    }
  }

  // 对象操作
  async getObject<T = any>(key: string): Promise<T | null> {
    try {
      const value = await this.get(key)
      return value ? JSON.parse(value) : null
    }
    catch (error) {
      this.logger.error(`Redis getObject error for key ${key}:`, error)
      return null
    }
  }

  async setObject(key: string, obj: any, ttl?: number): Promise<void> {
    try {
      await this.set(key, JSON.stringify(obj), ttl)
    }
    catch (error) {
      this.logger.error(`Redis setObject error for key ${key}:`, error)
    }
  }

  // 哈希操作
  async hget(key: string, field: string): Promise<string | null> {
    try {
      return await this.cacheClient.hget(key, field)
    }
    catch (error) {
      this.logger.error(`Redis HGET error for key ${key}, field ${field}:`, error)
      return null
    }
  }

  async hset(key: string, field: string, value: string): Promise<void> {
    try {
      await this.cacheClient.hset(key, field, value)
    }
    catch (error) {
      this.logger.error(`Redis HSET error for key ${key}, field ${field}:`, error)
    }
  }

  async hgetall(key: string): Promise<Record<string, string>> {
    try {
      return await this.cacheClient.hgetall(key)
    }
    catch (error) {
      this.logger.error(`Redis HGETALL error for key ${key}:`, error)
      return {}
    }
  }

  async hdel(key: string, field: string | string[]): Promise<number> {
    try {
      return await this.cacheClient.hdel(key, field)
    }
    catch (error) {
      this.logger.error(`Redis HDEL error for key ${key}, field ${field}:`, error)
      return 0
    }
  }

  // 列表操作
  async lpush(key: string, ...values: string[]): Promise<number> {
    try {
      return await this.cacheClient.lpush(key, ...values)
    }
    catch (error) {
      this.logger.error(`Redis LPUSH error for key ${key}:`, error)
      return 0
    }
  }

  async rpop(key: string): Promise<string | null> {
    try {
      return await this.cacheClient.rpop(key)
    }
    catch (error) {
      this.logger.error(`Redis RPOP error for key ${key}:`, error)
      return null
    }
  }

  async lrange(key: string, start: number, stop: number): Promise<string[]> {
    try {
      return await this.cacheClient.lrange(key, start, stop)
    }
    catch (error) {
      this.logger.error(`Redis LRANGE error for key ${key}:`, error)
      return []
    }
  }

  // 集合操作
  async sadd(key: string, ...members: string[]): Promise<number> {
    try {
      return await this.cacheClient.sadd(key, ...members)
    }
    catch (error) {
      this.logger.error(`Redis SADD error for key ${key}:`, error)
      return 0
    }
  }

  async srem(key: string, ...members: string[]): Promise<number> {
    try {
      return await this.cacheClient.srem(key, ...members)
    }
    catch (error) {
      this.logger.error(`Redis SREM error for key ${key}:`, error)
      return 0
    }
  }

  async smembers(key: string): Promise<string[]> {
    try {
      return await this.cacheClient.smembers(key)
    }
    catch (error) {
      this.logger.error(`Redis SMEMBERS error for key ${key}:`, error)
      return []
    }
  }

  async sismember(key: string, member: string): Promise<boolean> {
    try {
      const result = await this.cacheClient.sismember(key, member)
      return result === 1
    }
    catch (error) {
      this.logger.error(`Redis SISMEMBER error for key ${key}, member ${member}:`, error)
      return false
    }
  }

  // 有序集合操作
  async zadd(key: string, score: number, member: string): Promise<number> {
    try {
      return await this.cacheClient.zadd(key, score, member)
    }
    catch (error) {
      this.logger.error(`Redis ZADD error for key ${key}:`, error)
      return 0
    }
  }

  async zrange(key: string, start: number, stop: number): Promise<string[]> {
    try {
      return await this.cacheClient.zrange(key, start, stop)
    }
    catch (error) {
      this.logger.error(`Redis ZRANGE error for key ${key}:`, error)
      return []
    }
  }

  async zrem(key: string, ...members: string[]): Promise<number> {
    try {
      return await this.cacheClient.zrem(key, ...members)
    }
    catch (error) {
      this.logger.error(`Redis ZREM error for key ${key}:`, error)
      return 0
    }
  }

  // 发布订阅
  async publish(channel: string, message: string): Promise<number> {
    try {
      return await this.pubsubClient.publish(channel, message)
    }
    catch (error) {
      this.logger.error(`Redis PUBLISH error for channel ${channel}:`, error)
      return 0
    }
  }

  async subscribe(channel: string, callback: (message: string) => void): Promise<void> {
    try {
      await this.pubsubClient.subscribe(channel)
      this.pubsubClient.on('message', (receivedChannel, message) => {
        if (receivedChannel === channel) {
          callback(message)
        }
      })
    }
    catch (error) {
      this.logger.error(`Redis SUBSCRIBE error for channel ${channel}:`, error)
    }
  }

  // 模式匹配
  async keys(pattern: string): Promise<string[]> {
    try {
      return await this.cacheClient.keys(pattern)
    }
    catch (error) {
      this.logger.error(`Redis KEYS error for pattern ${pattern}:`, error)
      return []
    }
  }

  async deleteByPattern(pattern: string): Promise<number> {
    try {
      const keys = await this.keys(pattern)
      if (keys.length > 0) {
        return await this.del(keys)
      }
      return 0
    }
    catch (error) {
      this.logger.error(`Redis deleteByPattern error for pattern ${pattern}:`, error)
      return 0
    }
  }

  // 获取客户端实例（用于事务等高级操作）
  getCacheClient(): Redis {
    return this.cacheClient
  }

  getPubSubClient(): Redis {
    return this.pubsubClient
  }
}
```

## 📝 基础缓存操作

### 1. 用户缓存服务

```typescript
// modules/user/user-cache.service.ts
import { Injectable } from '@nestjs/common'
import { CACHE_KEYS, CACHE_TTL } from '~/shared/redis/redis.constant'
import { RedisService } from '~/shared/redis/redis.service'
import { UserEntity } from './user.entity'

@Injectable()
export class UserCacheService {
  constructor(private readonly redisService: RedisService) {}

  // 用户信息缓存
  private getUserKey(userId: number): string {
    return `${CACHE_KEYS.USER}:${userId}`
  }

  private getUserProfileKey(userId: number): string {
    return `${CACHE_KEYS.USER_PROFILE}:${userId}`
  }

  private getUserPermissionsKey(userId: number): string {
    return `${CACHE_KEYS.USER_PERMISSIONS}:${userId}`
  }

  // 缓存用户信息
  async cacheUser(user: UserEntity): Promise<void> {
    const key = this.getUserKey(user.id)
    await this.redisService.setObject(key, user, CACHE_TTL.ONE_HOUR)
  }

  // 获取缓存的用户信息
  async getCachedUser(userId: number): Promise<UserEntity | null> {
    const key = this.getUserKey(userId)
    return await this.redisService.getObject<UserEntity>(key)
  }

  // 删除用户缓存
  async deleteCachedUser(userId: number): Promise<void> {
    const keys = [
      this.getUserKey(userId),
      this.getUserProfileKey(userId),
      this.getUserPermissionsKey(userId),
    ]
    await this.redisService.del(keys)
  }

  // 缓存用户权限
  async cacheUserPermissions(userId: number, permissions: string[]): Promise<void> {
    const key = this.getUserPermissionsKey(userId)
    await this.redisService.setObject(key, permissions, CACHE_TTL.THIRTY_MINUTES)
  }

  // 获取缓存的用户权限
  async getCachedUserPermissions(userId: number): Promise<string[] | null> {
    const key = this.getUserPermissionsKey(userId)
    return await this.redisService.getObject<string[]>(key)
  }

  // 批量缓存用户
  async batchCacheUsers(users: UserEntity[]): Promise<void> {
    const pipeline = this.redisService.getCacheClient().pipeline()

    users.forEach((user) => {
      const key = this.getUserKey(user.id)
      pipeline.setex(key, CACHE_TTL.ONE_HOUR, JSON.stringify(user))
    })

    await pipeline.exec()
  }

  // 更新用户在线状态
  async setUserOnline(userId: number, socketId: string): Promise<void> {
    const key = `${CACHE_KEYS.ONLINE_USERS}:${userId}`
    await this.redisService.hset(key, 'socketId', socketId)
    await this.redisService.hset(key, 'lastSeen', new Date().toISOString())
    await this.redisService.expire(key, CACHE_TTL.ONE_DAY)
  }

  // 检查用户是否在线
  async isUserOnline(userId: number): Promise<boolean> {
    const key = `${CACHE_KEYS.ONLINE_USERS}:${userId}`
    return await this.redisService.exists(key)
  }

  // 获取在线用户列表
  async getOnlineUsers(): Promise<number[]> {
    const pattern = `${CACHE_KEYS.ONLINE_USERS}:*`
    const keys = await this.redisService.keys(pattern)
    return keys.map(key => Number.parseInt(key.split(':').pop()!))
  }

  // 用户离线
  async setUserOffline(userId: number): Promise<void> {
    const key = `${CACHE_KEYS.ONLINE_USERS}:${userId}`
    await this.redisService.del(key)
  }
}
```

### 2. 缓存装饰器

```typescript
// common/decorators/cache.decorator.ts
import { SetMetadata } from '@nestjs/common'

// 缓存拦截器
import { CallHandler, ExecutionContext, Injectable, NestInterceptor } from '@nestjs/common'
import { Reflector } from '@nestjs/core'
import { Observable, of } from 'rxjs'
import { tap } from 'rxjs/operators'
import { RedisService } from '~/shared/redis/redis.service'

// 缓存元数据键
export const CACHE_METADATA_KEY = 'cache'

// 缓存配置接口
export interface CacheConfig {
  key?: string
  ttl?: number
  prefix?: string
  condition?: (result: any) => boolean
}

// 缓存装饰器
export function Cache(config: CacheConfig = {}) {
  return SetMetadata(CACHE_METADATA_KEY, config)
}

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  constructor(
    private readonly redisService: RedisService,
    private readonly reflector: Reflector,
  ) {}

  async intercept(context: ExecutionContext, next: CallHandler): Promise<Observable<any>> {
    const cacheConfig = this.reflector.get<CacheConfig>(
      CACHE_METADATA_KEY,
      context.getHandler(),
    )

    if (!cacheConfig) {
      return next.handle()
    }

    const request = context.switchToHttp().getRequest()
    const cacheKey = this.generateCacheKey(cacheConfig, request)

    // 尝试从缓存获取
    const cachedResult = await this.redisService.getObject(cacheKey)
    if (cachedResult !== null) {
      return of(cachedResult)
    }

    // 执行原方法并缓存结果
    return next.handle().pipe(
      tap(async (result) => {
        if (result !== null && result !== undefined) {
          // 检查缓存条件
          if (!cacheConfig.condition || cacheConfig.condition(result)) {
            await this.redisService.setObject(
              cacheKey,
              result,
              cacheConfig.ttl || 300, // 默认5分钟
            )
          }
        }
      }),
    )
  }

  private generateCacheKey(config: CacheConfig, request: any): string {
    const prefix = config.prefix || 'cache'
    const key = config.key || `${request.method}:${request.url}`
    const params = JSON.stringify(request.query || {})
    return `${prefix}:${key}:${params}`
  }
}
```

### 3. 使用缓存装饰器

```typescript
// modules/user/user.service.ts
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(UserEntity)
    private readonly userRepository: Repository<UserEntity>,
    private readonly userCacheService: UserCacheService,
  ) {}

  // 使用缓存装饰器
  @Cache({
    key: 'user:profile',
    ttl: CACHE_TTL.ONE_HOUR,
    condition: result => result !== null,
  })
  async getUserProfile(userId: number): Promise<UserEntity | null> {
    return await this.userRepository.findOne({
      where: { id: userId },
      relations: ['roles', 'dept'],
    })
  }

  // 手动缓存管理
  async findUserById(userId: number): Promise<UserEntity | null> {
    // 先从缓存获取
    let user = await this.userCacheService.getCachedUser(userId)

    if (!user) {
      // 缓存未命中，从数据库查询
      user = await this.userRepository.findOne({
        where: { id: userId },
        relations: ['roles', 'dept'],
      })

      if (user) {
        // 存入缓存
        await this.userCacheService.cacheUser(user)
      }
    }

    return user
  }

  // 更新用户时清除缓存
  async updateUser(userId: number, updateData: UpdateUserDto): Promise<UserEntity> {
    const user = await this.userRepository.save({ id: userId, ...updateData })

    // 清除相关缓存
    await this.userCacheService.deleteCachedUser(userId)

    // 重新缓存更新后的用户信息
    await this.userCacheService.cacheUser(user)

    return user
  }
}
```

## 🚀 高级缓存策略

### 1. 分层缓存

```typescript
// common/cache/layered-cache.service.ts
import { Injectable } from '@nestjs/common'
import { RedisService } from '~/shared/redis/redis.service'

@Injectable()
export class LayeredCacheService {
  private readonly memoryCache = new Map<string, { value: any, expireAt: number }>()

  constructor(private readonly redisService: RedisService) {}

  async get<T>(key: string): Promise<T | null> {
    // L1: 内存缓存
    const memCached = this.memoryCache.get(key)
    if (memCached && memCached.expireAt > Date.now()) {
      return memCached.value
    }

    // L2: Redis 缓存
    const redisCached = await this.redisService.getObject<T>(key)
    if (redisCached !== null) {
      // 回填到内存缓存
      this.memoryCache.set(key, {
        value: redisCached,
        expireAt: Date.now() + 60000, // 内存缓存1分钟
      })
      return redisCached
    }

    return null
  }

  async set(key: string, value: any, ttl: number = 300): Promise<void> {
    // 同时写入两层缓存
    this.memoryCache.set(key, {
      value,
      expireAt: Date.now() + Math.min(ttl * 1000, 60000), // 内存缓存最多1分钟
    })

    await this.redisService.setObject(key, value, ttl)
  }

  async del(key: string): Promise<void> {
    this.memoryCache.delete(key)
    await this.redisService.del(key)
  }

  // 清理过期的内存缓存
  private cleanupMemoryCache(): void {
    const now = Date.now()
    for (const [key, item] of this.memoryCache.entries()) {
      if (item.expireAt <= now) {
        this.memoryCache.delete(key)
      }
    }
  }
}
```

### 2. 缓存预热

```typescript
// common/cache/cache-warmer.service.ts
import { Injectable, OnModuleInit } from '@nestjs/common'
import { Cron, CronExpression } from '@nestjs/schedule'
import { MenuService } from '~/modules/system/menu/menu.service'
import { UserService } from '~/modules/user/user.service'

@Injectable()
export class CacheWarmerService implements OnModuleInit {
  constructor(
    private readonly userService: UserService,
    private readonly menuService: MenuService,
  ) {}

  async onModuleInit() {
    // 应用启动时预热缓存
    await this.warmupCache()
  }

  @Cron(CronExpression.EVERY_HOUR)
  async scheduleWarmup() {
    // 每小时执行一次缓存预热
    await this.warmupCache()
  }

  private async warmupCache(): Promise<void> {
    try {
      // 预热热门用户数据
      await this.warmupPopularUsers()

      // 预热菜单数据
      await this.warmupMenus()

      // 预热系统配置
      await this.warmupSystemConfig()

      console.log('Cache warmup completed')
    }
    catch (error) {
      console.error('Cache warmup failed:', error)
    }
  }

  private async warmupPopularUsers(): Promise<void> {
    // 获取活跃用户列表
    const activeUsers = await this.userService.getActiveUsers(100)

    // 并发预热用户缓存
    const promises = activeUsers.map(user =>
      this.userService.findUserById(user.id)
    )

    await Promise.all(promises)
  }

  private async warmupMenus(): Promise<void> {
    // 预热菜单树结构
    await this.menuService.getMenuTree()
  }

  private async warmupSystemConfig(): Promise<void> {
    // 预热系统配置
    // await this.configService.getAllConfigs()
  }
}
```

### 3. 缓存穿透防护

```typescript
// common/cache/cache-guard.service.ts
import { Injectable } from '@nestjs/common'
import { RedisService } from '~/shared/redis/redis.service'

@Injectable()
export class CacheGuardService {
  constructor(private readonly redisService: RedisService) {}

  // 布隆过滤器防止缓存穿透
  async bloomFilterCheck(key: string): Promise<boolean> {
    // 简化的布隆过滤器实现
    const bloomKey = `bloom:${key}`
    return await this.redisService.exists(bloomKey)
  }

  async addToBloomFilter(key: string): Promise<void> {
    const bloomKey = `bloom:${key}`
    await this.redisService.set(bloomKey, '1', 86400) // 24小时过期
  }

  // 空值缓存防止缓存穿透
  async cacheNull(key: string, ttl: number = 300): Promise<void> {
    await this.redisService.set(key, 'NULL', ttl)
  }

  async isNullCache(key: string): Promise<boolean> {
    const value = await this.redisService.get(key)
    return value === 'NULL'
  }

  // 分布式锁防止缓存击穿
  async acquireLock(
    key: string,
    timeout: number = 10000,
    retry: number = 3
  ): Promise<boolean> {
    const lockKey = `lock:${key}`
    const lockValue = Date.now().toString()

    for (let i = 0; i < retry; i++) {
      const result = await this.redisService.getCacheClient().set(
        lockKey,
        lockValue,
        'PX',
        timeout,
        'NX'
      )

      if (result === 'OK') {
        return true
      }

      // 等待随机时间后重试
      await new Promise(resolve =>
        setTimeout(resolve, Math.random() * 100)
      )
    }

    return false
  }

  async releaseLock(key: string): Promise<void> {
    const lockKey = `lock:${key}`
    await this.redisService.del(lockKey)
  }
}
```

## 🔄 缓存更新机制

### 1. 缓存更新策略

```typescript
// common/cache/cache-update.service.ts
import { Injectable } from '@nestjs/common'
import { EventEmitter2 } from '@nestjs/event-emitter'
import { RedisService } from '~/shared/redis/redis.service'

@Injectable()
export class CacheUpdateService {
  constructor(
    private readonly redisService: RedisService,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  // Write-Through 策略
  async writeThrough<T>(
    key: string,
    value: T,
    updateFn: () => Promise<T>,
    ttl?: number
  ): Promise<T> {
    // 先更新数据库
    const updatedValue = await updateFn()

    // 再更新缓存
    await this.redisService.setObject(key, updatedValue, ttl)

    // 发出缓存更新事件
    this.eventEmitter.emit('cache.updated', { key, value: updatedValue })

    return updatedValue
  }

  // Write-Behind 策略
  async writeBehind<T>(key: string, value: T, ttl?: number): Promise<void> {
    // 立即更新缓存
    await this.redisService.setObject(key, value, ttl)

    // 异步更新数据库
    this.eventEmitter.emit('cache.write-behind', { key, value })
  }

  // 延迟双删策略
  async delayedDoubleDelete(
    key: string,
    updateFn: () => Promise<void>,
    delay: number = 1000
  ): Promise<void> {
    // 第一次删除缓存
    await this.redisService.del(key)

    // 更新数据库
    await updateFn()

    // 延迟后再次删除缓存
    setTimeout(async () => {
      await this.redisService.del(key)
    }, delay)
  }

  // 版本控制缓存更新
  async updateWithVersion<T>(
    key: string,
    value: T,
    version: number,
    ttl?: number
  ): Promise<boolean> {
    const versionKey = `${key}:version`
    const currentVersion = await this.redisService.get(versionKey)

    if (currentVersion && Number.parseInt(currentVersion) >= version) {
      return false // 版本过低，不更新
    }

    // 使用事务确保原子性
    const pipeline = this.redisService.getCacheClient().pipeline()
    pipeline.set(key, JSON.stringify(value))
    pipeline.set(versionKey, version.toString())

    if (ttl) {
      pipeline.expire(key, ttl)
      pipeline.expire(versionKey, ttl)
    }

    await pipeline.exec()
    return true
  }
}
```

### 2. 缓存事件监听

```typescript
// modules/user/user-cache.listener.ts
import { Injectable } from '@nestjs/common'
import { OnEvent } from '@nestjs/event-emitter'
import { UserCacheService } from './user-cache.service'

@Injectable()
export class UserCacheListener {
  constructor(private readonly userCacheService: UserCacheService) {}

  @OnEvent('user.created')
  async handleUserCreated(payload: { user: UserEntity }) {
    // 用户创建后缓存基本信息
    await this.userCacheService.cacheUser(payload.user)
  }

  @OnEvent('user.updated')
  async handleUserUpdated(payload: { userId: number, updateData: any }) {
    // 用户更新后清除相关缓存
    await this.userCacheService.deleteCachedUser(payload.userId)
  }

  @OnEvent('user.deleted')
  async handleUserDeleted(payload: { userId: number }) {
    // 用户删除后清除所有相关缓存
    await this.userCacheService.deleteCachedUser(payload.userId)
  }

  @OnEvent('user.login')
  async handleUserLogin(payload: { userId: number, socketId: string }) {
    // 用户登录后设置在线状态
    await this.userCacheService.setUserOnline(payload.userId, payload.socketId)
  }

  @OnEvent('user.logout')
  async handleUserLogout(payload: { userId: number }) {
    // 用户登出后设置离线状态
    await this.userCacheService.setUserOffline(payload.userId)
  }

  @OnEvent('cache.write-behind')
  async handleWriteBehind(payload: { key: string, value: any }) {
    // 处理延迟写入数据库的逻辑
    if (payload.key.startsWith('user:')) {
      // 异步更新用户数据到数据库
      // await this.userService.updateUserInDatabase(payload.value)
    }
  }
}
```

## ⚡ 性能优化

### 1. 缓存压缩

```typescript
import { promisify } from 'node:util'
import { gunzip, gzip } from 'node:zlib'
// common/cache/cache-compression.service.ts
import { Injectable } from '@nestjs/common'

const gzipAsync = promisify(gzip)
const gunzipAsync = promisify(gunzip)

@Injectable()
export class CacheCompressionService {
  private readonly compressionThreshold = 1024 // 1KB 以上的数据进行压缩

  async compress(data: string): Promise<Buffer> {
    if (data.length < this.compressionThreshold) {
      return Buffer.from(data, 'utf8')
    }

    return await gzipAsync(data)
  }

  async decompress(buffer: Buffer): Promise<string> {
    try {
      // 尝试解压
      const decompressed = await gunzipAsync(buffer)
      return decompressed.toString('utf8')
    }
    catch {
      // 如果解压失败，说明数据未压缩
      return buffer.toString('utf8')
    }
  }

  async setCompressed(
    redisService: any,
    key: string,
    value: any,
    ttl?: number
  ): Promise<void> {
    const jsonString = JSON.stringify(value)
    const compressed = await this.compress(jsonString)

    if (ttl) {
      await redisService.getCacheClient().setex(key, ttl, compressed)
    }
    else {
      await redisService.getCacheClient().set(key, compressed)
    }
  }

  async getCompressed<T>(redisService: any, key: string): Promise<T | null> {
    const buffer = await redisService.getCacheClient().getBuffer(key)
    if (!buffer) {
      return null
    }

    const decompressed = await this.decompress(buffer)
    return JSON.parse(decompressed)
  }
}
```

### 2. 缓存批量操作

```typescript
// common/cache/batch-cache.service.ts
import { Injectable } from '@nestjs/common'
import { RedisService } from '~/shared/redis/redis.service'

@Injectable()
export class BatchCacheService {
  constructor(private readonly redisService: RedisService) {}

  // 批量获取
  async batchGet<T>(keys: string[]): Promise<Record<string, T | null>> {
    if (keys.length === 0) {
      return {}
    }

    const pipeline = this.redisService.getCacheClient().pipeline()
    keys.forEach(key => pipeline.get(key))

    const results = await pipeline.exec()
    const response: Record<string, T | null> = {}

    results?.forEach((result, index) => {
      const [error, value] = result
      if (!error && value) {
        try {
          response[keys[index]] = JSON.parse(value as string)
        }
        catch {
          response[keys[index]] = value as T
        }
      }
      else {
        response[keys[index]] = null
      }
    })

    return response
  }

  // 批量设置
  async batchSet(items: Array<{ key: string, value: any, ttl?: number }>): Promise<void> {
    if (items.length === 0) {
      return
    }

    const pipeline = this.redisService.getCacheClient().pipeline()

    items.forEach(({ key, value, ttl }) => {
      const serialized = JSON.stringify(value)
      if (ttl) {
        pipeline.setex(key, ttl, serialized)
      }
      else {
        pipeline.set(key, serialized)
      }
    })

    await pipeline.exec()
  }

  // 批量删除
  async batchDelete(keys: string[]): Promise<number> {
    if (keys.length === 0) {
      return 0
    }

    return await this.redisService.del(keys)
  }

  // 批量检查存在性
  async batchExists(keys: string[]): Promise<Record<string, boolean>> {
    if (keys.length === 0) {
      return {}
    }

    const pipeline = this.redisService.getCacheClient().pipeline()
    keys.forEach(key => pipeline.exists(key))

    const results = await pipeline.exec()
    const response: Record<string, boolean> = {}

    results?.forEach((result, index) => {
      const [error, value] = result
      response[keys[index]] = !error && value === 1
    })

    return response
  }
}
```

### 3. 缓存统计监控

```typescript
// common/cache/cache-metrics.service.ts
import { Injectable } from '@nestjs/common'
import { RedisService } from '~/shared/redis/redis.service'

@Injectable()
export class CacheMetricsService {
  private readonly metricsKey = 'cache:metrics'

  constructor(private readonly redisService: RedisService) {}

  // 记录缓存命中
  async recordHit(key: string): Promise<void> {
    const date = new Date().toISOString().split('T')[0]
    const hitKey = `${this.metricsKey}:${date}:hits`
    const keyHitKey = `${this.metricsKey}:${date}:key:${key}:hits`

    const pipeline = this.redisService.getCacheClient().pipeline()
    pipeline.incr(hitKey)
    pipeline.incr(keyHitKey)
    pipeline.expire(hitKey, 86400 * 7) // 保留7天
    pipeline.expire(keyHitKey, 86400 * 7)

    await pipeline.exec()
  }

  // 记录缓存未命中
  async recordMiss(key: string): Promise<void> {
    const date = new Date().toISOString().split('T')[0]
    const missKey = `${this.metricsKey}:${date}:misses`
    const keyMissKey = `${this.metricsKey}:${date}:key:${key}:misses`

    const pipeline = this.redisService.getCacheClient().pipeline()
    pipeline.incr(missKey)
    pipeline.incr(keyMissKey)
    pipeline.expire(missKey, 86400 * 7)
    pipeline.expire(keyMissKey, 86400 * 7)

    await pipeline.exec()
  }

  // 获取缓存统计
  async getMetrics(date?: string): Promise<{
    hits: number
    misses: number
    hitRate: number
    totalRequests: number
  }> {
    const targetDate = date || new Date().toISOString().split('T')[0]
    const hitKey = `${this.metricsKey}:${targetDate}:hits`
    const missKey = `${this.metricsKey}:${targetDate}:misses`

    const [hitsStr, missesStr] = await Promise.all([
      this.redisService.get(hitKey),
      this.redisService.get(missKey),
    ])

    const hits = Number.parseInt(hitsStr || '0')
    const misses = Number.parseInt(missesStr || '0')
    const totalRequests = hits + misses
    const hitRate = totalRequests > 0 ? hits / totalRequests : 0

    return { hits, misses, hitRate, totalRequests }
  }

  // 获取热门缓存键
  async getHotKeys(date?: string, limit: number = 10): Promise<Array<{
    key: string
    hits: number
    misses: number
  }>> {
    const targetDate = date || new Date().toISOString().split('T')[0]
    const pattern = `${this.metricsKey}:${targetDate}:key:*:hits`
    const keys = await this.redisService.keys(pattern)

    const results = await this.redisService.getCacheClient().mget(...keys)
    const keyStats = keys.map((key, index) => {
      const keyName = key.match(/:key:(.+):hits$/)?.[1] || 'unknown'
      const hits = Number.parseInt(results[index] || '0')
      return { key: keyName, hits }
    })

    // 获取对应的 misses
    for (const stat of keyStats) {
      const missKey = `${this.metricsKey}:${targetDate}:key:${stat.key}:misses`
      const misses = await this.redisService.get(missKey)
      stat.misses = Number.parseInt(misses || '0')
    }

    return keyStats
      .sort((a, b) => b.hits - a.hits)
      .slice(0, limit)
  }
}
```

## 🚀 最佳实践总结

### 1. 缓存设计原则
- **合理设置TTL**：根据数据特性设置合适的过期时间
- **键名规范**：使用有意义的键名前缀和层次结构
- **数据序列化**：统一使用JSON序列化，考虑压缩大数据

### 2. 性能优化策略
- **批量操作**：尽量使用批量操作减少网络开销
- **管道技术**：使用Redis Pipeline提高并发性能
- **分层缓存**：结合内存缓存和Redis缓存

### 3. 数据一致性
- **缓存更新策略**：选择合适的缓存更新模式
- **事件驱动**：使用事件机制保证缓存一致性
- **版本控制**：使用版本号避免并发更新问题

### 4. 可靠性保障
- **异常处理**：缓存操作失败不应影响主业务
- **降级策略**：缓存不可用时的备用方案
- **监控告警**：监控缓存命中率和性能指标

### 5. 安全考虑
- **访问控制**：限制Redis访问权限
- **数据加密**：敏感数据缓存前加密
- **防止攻击**：避免缓存穿透、击穿和雪崩

这份指南涵盖了 nest-admin 项目中 Redis 缓存的核心应用场景，帮助你更好地设计和优化缓存系统。
