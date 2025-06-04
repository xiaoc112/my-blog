# TypeORM 数据库操作详解

基于 nest-admin 项目的 TypeORM 最佳实践

## 📚 目录

1. [数据库配置](#数据库配置)
2. [实体定义](#实体定义)
3. [数据库连接](#数据库连接)
4. [基础CRUD操作](#基础crud操作)
5. [高级查询](#高级查询)
6. [关系操作](#关系操作)
7. [事务处理](#事务处理)
8. [数据迁移](#数据迁移)
9. [性能优化](#性能优化)

## ⚙️ 数据库配置

### 1. 数据库连接配置

```typescript
// config/database.config.ts
import { ConfigType, registerAs } from '@nestjs/config'
import { DataSource, DataSourceOptions } from 'typeorm'
import { env, envBoolean, envNumber } from '~/global/env'

const dataSourceOptions: DataSourceOptions = {
  type: 'mysql',
  host: env('DB_HOST', '127.0.0.1'),
  port: envNumber('DB_PORT', 3306),
  username: env('DB_USERNAME'),
  password: env('DB_PASSWORD'),
  database: env('DB_DATABASE'),
  synchronize: envBoolean('DB_SYNCHRONIZE', false), // 生产环境设为 false
  multipleStatements: true, // 支持多语句执行（用于数据迁移）

  // 实体路径配置
  entities: [
    'dist/modules/**/*.entity{.ts,.js}',
    'dist/app/**/*.entity{.ts,.js}'
  ],

  // 迁移文件路径
  migrations: ['dist/migrations/*{.ts,.js}'],

  // 订阅者路径
  subscribers: ['dist/modules/**/*.subscriber{.ts,.js}'],

  // 连接池配置
  extra: {
    connectionLimit: 10,
  },

  // 日志配置
  logging: process.env.NODE_ENV === 'development' ? ['error', 'warn'] : false,
}

export const DatabaseConfig = registerAs(
  'database',
  (): DataSourceOptions => dataSourceOptions,
)

// 创建数据源实例（用于迁移）
const dataSource = new DataSource(dataSourceOptions)
export default dataSource
```

### 2. 数据库模块配置

```typescript
// shared/database/database.module.ts
import { Module } from '@nestjs/common'
import { ConfigService } from '@nestjs/config'
import { TypeOrmModule } from '@nestjs/typeorm'

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      useFactory: (configService: ConfigService) => ({
        ...configService.get('database'),
      }),
      inject: [ConfigService],
    }),
  ],
})
export class DatabaseModule {}
```

## 🏗️ 实体定义

### 1. 基础实体类

```typescript
// common/entity/common.entity.ts
import {
  CreateDateColumn,
  PrimaryGeneratedColumn,
  UpdateDateColumn,
} from 'typeorm'

export abstract class CommonEntity {
  @PrimaryGeneratedColumn()
  id: number

  @CreateDateColumn({ name: 'created_at' })
  createdAt: Date

  @UpdateDateColumn({ name: 'updated_at' })
  updatedAt: Date
}
```

### 2. 用户实体示例

```typescript
// modules/user/user.entity.ts
import { Exclude } from 'class-transformer'
import {
  Column,
  Entity,
  JoinColumn,
  JoinTable,
  ManyToMany,
  ManyToOne,
  OneToMany,
  OneToOne,
  Relation,
} from 'typeorm'

import { CommonEntity } from '~/common/entity/common.entity'
import { DeptEntity } from '~/modules/system/dept/dept.entity'
import { RoleEntity } from '~/modules/system/role/role.entity'

@Entity({ name: 'sys_user' })
export class UserEntity extends CommonEntity {
  @Column({ unique: true, comment: '用户名' })
  username: string

  @Exclude() // 序列化时排除密码字段
  @Column({ default: 'WECHAT_USER_NO_PASSWORD', comment: '密码' })
  password: string

  @Column({ length: 32, comment: '密码盐' })
  psalt: string

  @Column({ nullable: true, comment: '昵称' })
  nickname: string

  @Column({ name: 'avatar', nullable: true, comment: '头像' })
  avatar: string

  @Column({ nullable: true, comment: 'QQ号' })
  qq: string

  @Column({ nullable: true, comment: '邮箱' })
  email: string

  @Column({ nullable: true, comment: '手机号' })
  phone: string

  @Column({ nullable: true, comment: '备注' })
  remark: string

  @Column({ type: 'tinyint', nullable: true, default: 1, comment: '状态' })
  status: number

  // 多对多关系：用户-角色
  @ManyToMany(() => RoleEntity, role => role.users)
  @JoinTable({
    name: 'sys_user_roles',
    joinColumn: { name: 'user_id', referencedColumnName: 'id' },
    inverseJoinColumn: { name: 'role_id', referencedColumnName: 'id' },
  })
  roles: Relation<RoleEntity[]>

  // 多对一关系：用户-部门
  @ManyToOne(() => DeptEntity, dept => dept.users)
  @JoinColumn({ name: 'dept_id' })
  dept: Relation<DeptEntity>

  // 一对多关系：用户-访问令牌
  @OneToMany(() => AccessTokenEntity, accessToken => accessToken.user, {
    cascade: true,
  })
  accessTokens: Relation<AccessTokenEntity[]>
}
```

### 3. 实体装饰器详解

```typescript
// 常用装饰器示例
@Entity({
  name: 'custom_table_name', // 自定义表名
  schema: 'public', // 指定schema
  comment: '用户表' // 表注释
})
export class ExampleEntity {
  @PrimaryGeneratedColumn('uuid') // UUID主键
  id: string

  @Column({
    type: 'varchar',
    length: 100,
    unique: true,
    nullable: false,
    comment: '用户名',
    default: 'default_value'
  })
  username: string

  @Column({ type: 'text' })
  description: string

  @Column({ type: 'decimal', precision: 10, scale: 2 })
  price: number

  @Column({ type: 'enum', enum: ['active', 'inactive'] })
  status: 'active' | 'inactive'

  @Column({ type: 'json' })
  metadata: object

  @CreateDateColumn()
  createdAt: Date

  @UpdateDateColumn()
  updatedAt: Date

  @DeleteDateColumn() // 软删除
  deletedAt: Date

  @VersionColumn() // 乐观锁
  version: number

  // 索引定义
  @Index(['username', 'email']) // 复合索引
  @Column()
  email: string
}
```

## 🔌 数据库连接

### 1. Repository 注入

```typescript
// user.service.ts
import { Injectable } from '@nestjs/common'
import { InjectRepository } from '@nestjs/typeorm'
import { EntityManager, Repository } from 'typeorm'
import { UserEntity } from './user.entity'

@Injectable()
export class UserService {
  constructor(
    @InjectRepository(UserEntity)
    private readonly userRepository: Repository<UserEntity>,
    private readonly entityManager: EntityManager, // 实体管理器
  ) {}

  // 基础操作示例...
}
```

### 2. 模块中注册实体

```typescript
// user.module.ts
import { Module } from '@nestjs/common'
import { TypeOrmModule } from '@nestjs/typeorm'
import { UserController } from './user.controller'
import { UserEntity } from './user.entity'
import { UserService } from './user.service'

@Module({
  imports: [TypeOrmModule.forFeature([UserEntity])],
  controllers: [UserController],
  providers: [UserService],
  exports: [UserService],
})
export class UserModule {}
```

## 📝 基础CRUD操作

### 1. 创建操作

```typescript
@Injectable()
export class UserService {
  // 创建单个用户
  async createUser(userData: CreateUserDto): Promise<UserEntity> {
    // 方法1：使用 create + save
    const user = this.userRepository.create(userData)
    return await this.userRepository.save(user)

    // 方法2：直接使用 save
    return await this.userRepository.save(userData)

    // 方法3：使用 insert（返回插入结果）
    const result = await this.userRepository.insert(userData)
    return result.generatedMaps[0] as UserEntity
  }

  // 批量创建
  async createUsers(usersData: CreateUserDto[]): Promise<UserEntity[]> {
    const users = this.userRepository.create(usersData)
    return await this.userRepository.save(users)
  }

  // 使用 upsert（插入或更新）
  async upsertUser(userData: CreateUserDto): Promise<void> {
    await this.userRepository.upsert(userData, ['username'])
  }
}
```

### 2. 查询操作

```typescript
@Injectable()
export class UserService {
  // 查询所有用户
  async findAllUsers(): Promise<UserEntity[]> {
    return await this.userRepository.find()
  }

  // 根据ID查询
  async findUserById(id: number): Promise<UserEntity> {
    return await this.userRepository.findOne({
      where: { id }
    })
  }

  // 查询单个用户（找不到抛异常）
  async findUserByIdOrFail(id: number): Promise<UserEntity> {
    return await this.userRepository.findOneOrFail({
      where: { id }
    })
  }

  // 条件查询
  async findUsersByStatus(status: number): Promise<UserEntity[]> {
    return await this.userRepository.find({
      where: { status },
      order: { createdAt: 'DESC' },
      take: 10, // limit
      skip: 0, // offset
    })
  }

  // 查询指定字段
  async findUsersWithSelect(): Promise<Partial<UserEntity>[]> {
    return await this.userRepository.find({
      select: ['id', 'username', 'email'],
      where: { status: 1 },
    })
  }

  // 关联查询
  async findUsersWithRoles(): Promise<UserEntity[]> {
    return await this.userRepository.find({
      relations: ['roles', 'dept'],
      where: { status: 1 },
    })
  }

  // 计数查询
  async countActiveUsers(): Promise<number> {
    return await this.userRepository.count({
      where: { status: 1 }
    })
  }

  // 分页查询
  async findUsersWithPagination(page: number, limit: number) {
    const [users, total] = await this.userRepository.findAndCount({
      take: limit,
      skip: (page - 1) * limit,
      order: { createdAt: 'DESC' },
    })

    return {
      data: users,
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    }
  }
}
```

### 3. 更新操作

```typescript
@Injectable()
export class UserService {
  // 更新用户
  async updateUser(id: number, updateData: UpdateUserDto): Promise<UserEntity> {
    // 方法1：查询后更新
    const user = await this.userRepository.findOne({ where: { id } })
    if (!user) {
      throw new NotFoundException('用户不存在')
    }

    Object.assign(user, updateData)
    return await this.userRepository.save(user)
  }

  // 批量更新
  async updateUsers(ids: number[], updateData: UpdateUserDto): Promise<void> {
    await this.userRepository.update(
      { id: In(ids) },
      updateData
    )
  }

  // 条件更新
  async updateUsersByStatus(
    oldStatus: number,
    newStatus: number
  ): Promise<void> {
    await this.userRepository.update(
      { status: oldStatus },
      { status: newStatus }
    )
  }

  // 增量更新（数值字段）
  async incrementUserLoginCount(id: number): Promise<void> {
    await this.userRepository.increment(
      { id },
      'loginCount',
      1
    )
  }
}
```

### 4. 删除操作

```typescript
@Injectable()
export class UserService {
  // 物理删除
  async deleteUser(id: number): Promise<void> {
    const result = await this.userRepository.delete(id)
    if (result.affected === 0) {
      throw new NotFoundException('用户不存在')
    }
  }

  // 批量删除
  async deleteUsers(ids: number[]): Promise<void> {
    await this.userRepository.delete(ids)
  }

  // 条件删除
  async deleteInactiveUsers(): Promise<void> {
    await this.userRepository.delete({ status: 0 })
  }

  // 软删除（需要 @DeleteDateColumn）
  async softDeleteUser(id: number): Promise<void> {
    await this.userRepository.softDelete(id)
  }

  // 恢复软删除
  async restoreUser(id: number): Promise<void> {
    await this.userRepository.restore(id)
  }
}
```

## 🔍 高级查询

### 1. QueryBuilder 查询

```typescript
@Injectable()
export class UserService {
  // 使用 QueryBuilder 进行复杂查询
  async findUsersWithComplexQuery(filters: any) {
    const queryBuilder = this.userRepository
      .createQueryBuilder('user')
      .leftJoinAndSelect('user.roles', 'role')
      .leftJoinAndSelect('user.dept', 'dept')

    // 动态添加条件
    if (filters.username) {
      queryBuilder.andWhere('user.username LIKE :username', {
        username: `%${filters.username}%`
      })
    }

    if (filters.status !== undefined) {
      queryBuilder.andWhere('user.status = :status', {
        status: filters.status
      })
    }

    if (filters.deptId) {
      queryBuilder.andWhere('dept.id = :deptId', {
        deptId: filters.deptId
      })
    }

    // 排序和分页
    return await queryBuilder
      .orderBy('user.createdAt', 'DESC')
      .skip((filters.page - 1) * filters.limit)
      .take(filters.limit)
      .getManyAndCount()
  }

  // 聚合查询
  async getUserStatistics() {
    return await this.userRepository
      .createQueryBuilder('user')
      .select([
        'COUNT(user.id) as totalUsers',
        'COUNT(CASE WHEN user.status = 1 THEN 1 END) as activeUsers',
        'COUNT(CASE WHEN user.status = 0 THEN 1 END) as inactiveUsers',
      ])
      .getRawOne()
  }

  // 子查询
  async findUsersWithSubquery() {
    return await this.userRepository
      .createQueryBuilder('user')
      .where((qb) => {
        const subQuery = qb
          .subQuery()
          .select('role.userId')
          .from('sys_user_roles', 'role')
          .where('role.roleId = :roleId')
          .getQuery()
        return `user.id IN ${subQuery}`
      })
      .setParameter('roleId', 1)
      .getMany()
  }

  // 原生SQL查询
  async executeRawQuery(sql: string, parameters: any[] = []) {
    return await this.userRepository.query(sql, parameters)
  }
}
```

### 2. 高级查询条件

```typescript
import { Between, In, IsNull, LessThan, Like, MoreThan, Not } from 'typeorm'

@Injectable()
export class UserService {
  async findUsersWithAdvancedConditions() {
    return await this.userRepository.find({
      where: [
        // OR 条件
        { username: Like('%admin%') },
        { email: Like('%admin%') },

        // AND 条件组合
        {
          status: 1,
          createdAt: MoreThan(new Date('2023-01-01')),
          id: In([1, 2, 3, 4, 5])
        },

        // 更多条件
        {
          age: Between(18, 65),
          deletedAt: IsNull(),
          username: Not(Like('%test%'))
        }
      ]
    })
  }
}
```

## 🔗 关系操作

### 1. 一对多关系

```typescript
// 部门实体
@Entity({ name: 'sys_dept' })
export class DeptEntity extends CommonEntity {
  @Column()
  name: string

  @OneToMany(() => UserEntity, user => user.dept)
  users: Relation<UserEntity[]>
}

// 操作示例
@Injectable()
export class DeptService {
  // 查询部门及其用户
  async findDeptWithUsers(deptId: number) {
    return await this.deptRepository.findOne({
      where: { id: deptId },
      relations: ['users']
    })
  }

  // 创建部门并关联用户
  async createDeptWithUsers(deptData: CreateDeptDto) {
    const dept = this.deptRepository.create(deptData)
    const savedDept = await this.deptRepository.save(dept)

    // 关联用户
    if (deptData.userIds?.length) {
      await this.userRepository.update(
        { id: In(deptData.userIds) },
        { dept: savedDept }
      )
    }

    return savedDept
  }
}
```

### 2. 多对多关系

```typescript
@Injectable()
export class UserService {
  // 为用户分配角色
  async assignRolesToUser(userId: number, roleIds: number[]) {
    const user = await this.userRepository.findOne({
      where: { id: userId },
      relations: ['roles']
    })

    if (!user) {
      throw new NotFoundException('用户不存在')
    }

    const roles = await this.roleRepository.findByIds(roleIds)
    user.roles = roles

    return await this.userRepository.save(user)
  }

  // 移除用户角色
  async removeRolesFromUser(userId: number, roleIds: number[]) {
    const user = await this.userRepository.findOne({
      where: { id: userId },
      relations: ['roles']
    })

    user.roles = user.roles.filter(role => !roleIds.includes(role.id))

    return await this.userRepository.save(user)
  }

  // 使用 QueryBuilder 操作多对多关系
  async addRoleToUserDirectly(userId: number, roleId: number) {
    await this.userRepository
      .createQueryBuilder()
      .relation(UserEntity, 'roles')
      .of(userId)
      .add(roleId)
  }

  async removeRoleFromUserDirectly(userId: number, roleId: number) {
    await this.userRepository
      .createQueryBuilder()
      .relation(UserEntity, 'roles')
      .of(userId)
      .remove(roleId)
  }
}
```

## 💫 事务处理

### 1. 基础事务

```typescript
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(UserEntity)
    private readonly userRepository: Repository<UserEntity>,
    private readonly entityManager: EntityManager,
  ) {}

  // 使用 EntityManager 事务
  async createUserWithTransaction(userData: CreateUserDto) {
    return await this.entityManager.transaction(async (transactionManager) => {
      // 创建用户
      const user = transactionManager.create(UserEntity, userData)
      const savedUser = await transactionManager.save(user)

      // 创建用户配置（示例）
      const userConfig = transactionManager.create(UserConfigEntity, {
        userId: savedUser.id,
        theme: 'default'
      })
      await transactionManager.save(userConfig)

      // 发送欢迎邮件（可能失败）
      await this.emailService.sendWelcomeEmail(savedUser.email)

      return savedUser
    })
  }

  // 使用装饰器事务
  @Transactional()
  async updateUserAndLog(userId: number, updateData: UpdateUserDto) {
    const user = await this.userRepository.findOne({ where: { id: userId } })

    // 更新用户
    Object.assign(user, updateData)
    const updatedUser = await this.userRepository.save(user)

    // 记录操作日志
    await this.logService.createLog({
      action: 'UPDATE_USER',
      userId,
      details: updateData
    })

    return updatedUser
  }
}
```

### 2. 高级事务控制

```typescript
@Injectable()
export class UserService {
  // 手动控制事务
  async complexTransaction() {
    const queryRunner = this.entityManager.connection.createQueryRunner()

    await queryRunner.connect()
    await queryRunner.startTransaction()

    try {
      // 第一步操作
      const user = await queryRunner.manager.save(UserEntity, {
        username: 'test',
        email: 'test@example.com'
      })

      // 第二步操作（可能失败）
      await queryRunner.manager.save(UserProfileEntity, {
        userId: user.id,
        profile: 'test profile'
      })

      // 第三步操作
      await queryRunner.manager.update(UserEntity, user.id, {
        status: 1
      })

      // 提交事务
      await queryRunner.commitTransaction()

      return user
    }
    catch (err) {
      // 回滚事务
      await queryRunner.rollbackTransaction()
      throw err
    }
    finally {
      // 释放查询运行器
      await queryRunner.release()
    }
  }

  // 嵌套事务
  async nestedTransaction() {
    return await this.entityManager.transaction(async (manager) => {
      const user = await manager.save(UserEntity, { username: 'parent' })

      // 嵌套事务（SAVEPOINT）
      await manager.transaction(async (nestedManager) => {
        await nestedManager.save(UserProfileEntity, {
          userId: user.id,
          profile: 'nested'
        })
      })

      return user
    })
  }
}
```

## 🚀 数据迁移

### 1. 创建迁移文件

```bash
# 生成迁移文件
npm run migration:generate

# 创建空迁移文件
npm run migration:create

# 运行迁移
npm run migration:run

# 回滚迁移
npm run migration:revert
```

### 2. 迁移文件示例

```typescript
// migrations/1234567890123-CreateUserTable.ts
import { Index, MigrationInterface, QueryRunner, Table } from 'typeorm'

export class CreateUserTable1234567890123 implements MigrationInterface {
  name = 'CreateUserTable1234567890123'

  public async up(queryRunner: QueryRunner): Promise<void> {
    // 创建用户表
    await queryRunner.createTable(
      new Table({
        name: 'sys_user',
        columns: [
          {
            name: 'id',
            type: 'int',
            isPrimary: true,
            isGenerated: true,
            generationStrategy: 'increment',
          },
          {
            name: 'username',
            type: 'varchar',
            length: '50',
            isUnique: true,
          },
          {
            name: 'email',
            type: 'varchar',
            length: '100',
            isNullable: true,
          },
          {
            name: 'password',
            type: 'varchar',
            length: '255',
          },
          {
            name: 'status',
            type: 'tinyint',
            default: 1,
          },
          {
            name: 'created_at',
            type: 'timestamp',
            default: 'CURRENT_TIMESTAMP',
          },
          {
            name: 'updated_at',
            type: 'timestamp',
            default: 'CURRENT_TIMESTAMP',
            onUpdate: 'CURRENT_TIMESTAMP',
          },
        ],
      }),
    )

    // 创建索引
    await queryRunner.createIndex(
      'sys_user',
      new Index({
        name: 'IDX_USER_EMAIL',
        columnNames: ['email'],
      }),
    )

    // 插入初始数据
    await queryRunner.query(`
      INSERT INTO sys_user (username, email, password, status) VALUES
      ('admin', 'admin@example.com', 'hashed_password', 1),
      ('user', 'user@example.com', 'hashed_password', 1)
    `)
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable('sys_user')
  }
}
```

### 3. 数据种子

```typescript
// seeds/user.seed.ts
import { DataSource } from 'typeorm'
import { UserEntity } from '../modules/user/user.entity'

export class UserSeeder {
  public async run(dataSource: DataSource): Promise<void> {
    const userRepository = dataSource.getRepository(UserEntity)

    const users = [
      {
        username: 'admin',
        email: 'admin@example.com',
        password: 'hashed_password',
        status: 1,
      },
      {
        username: 'moderator',
        email: 'mod@example.com',
        password: 'hashed_password',
        status: 1,
      },
    ]

    for (const userData of users) {
      const existingUser = await userRepository.findOne({
        where: { username: userData.username }
      })

      if (!existingUser) {
        const user = userRepository.create(userData)
        await userRepository.save(user)
      }
    }
  }
}
```

## ⚡ 性能优化

### 1. 查询优化

```typescript
@Injectable()
export class UserService {
  // 使用索引优化查询
  async findUsersByIndexedField(email: string) {
    return await this.userRepository.find({
      where: { email }, // 确保 email 字段有索引
      cache: true, // 启用查询缓存
    })
  }

  // 分页查询优化
  async findUsersOptimized(page: number, limit: number) {
    return await this.userRepository
      .createQueryBuilder('user')
      .select(['user.id', 'user.username', 'user.email']) // 只选择需要的字段
      .where('user.status = :status', { status: 1 })
      .orderBy('user.id', 'DESC') // 使用主键排序更高效
      .skip((page - 1) * limit)
      .take(limit)
      .cache(60000) // 缓存60秒
      .getMany()
  }

  // 批量操作优化
  async bulkUpdateUsers(updates: Array<{ id: number, data: Partial<UserEntity> }>) {
    // 使用原生SQL进行批量更新
    const cases = updates.map(({ id, data }) => {
      const sets = Object.entries(data)
        .map(([key, value]) => `WHEN ${id} THEN '${value}'`)
        .join(' ')
      return `CASE id ${sets} END`
    }).join(', ')

    const ids = updates.map(u => u.id).join(',')

    await this.userRepository.query(`
      UPDATE sys_user SET
        username = ${cases}
      WHERE id IN (${ids})
    `)
  }
}
```

### 2. 连接池优化

```typescript
// database.config.ts
const dataSourceOptions: DataSourceOptions = {
  // ... 其他配置
  extra: {
    connectionLimit: 20, // 连接池大小
    acquireTimeout: 60000, // 获取连接超时时间
    timeout: 60000, // 查询超时时间
    reconnect: true, // 自动重连
    charset: 'utf8mb4_unicode_ci',
  },
}
```

### 3. 缓存策略

```typescript
@Injectable()
export class UserService {
  // 使用Redis缓存查询结果
  async findUserWithCache(id: number): Promise<UserEntity> {
    const cacheKey = `user:${id}`

    // 尝试从缓存获取
    const cached = await this.redisService.get(cacheKey)
    if (cached) {
      return JSON.parse(cached)
    }

    // 从数据库查询
    const user = await this.userRepository.findOne({
      where: { id },
      relations: ['roles', 'dept']
    })

    if (user) {
      // 存入缓存，有效期1小时
      await this.redisService.setex(cacheKey, 3600, JSON.stringify(user))
    }

    return user
  }

  // 更新时清除缓存
  async updateUserAndClearCache(id: number, updateData: UpdateUserDto) {
    const user = await this.userRepository.save({ id, ...updateData })

    // 清除相关缓存
    await this.redisService.del(`user:${id}`)
    await this.redisService.del(`user:list:*`) // 清除列表缓存

    return user
  }
}
```

## 🚀 最佳实践总结

### 1. 实体设计原则
- 使用基础实体类避免重复代码
- 合理使用索引提升查询性能
- 正确定义关系避免N+1问题

### 2. 查询优化策略
- 只查询需要的字段
- 使用适当的缓存策略
- 避免在循环中执行查询

### 3. 事务管理
- 保持事务简短避免长锁
- 正确处理异常和回滚
- 合理使用事务隔离级别

### 4. 性能监控
- 启用慢查询日志
- 监控连接池状态
- 定期分析查询性能

### 5. 安全考虑
- 防止SQL注入攻击
- 敏感数据加密存储
- 实现软删除机制

这份指南涵盖了 nest-admin 项目中 TypeORM 的核心应用场景，帮助你更好地进行数据库操作和优化。
