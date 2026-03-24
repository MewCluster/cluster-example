# cluster-example

> 示例仓库，用于展示基于 Flux 的 GitOps 目录结构，仅供工具开发时参考使用，不具备实际部署能力。

## 目录结构

```
├── .nodes/  # 集群节点配置
├── apps/          # 应用层
├── config/        # cluster-vars
├── environments/  # Flux 控制平面入口，按集群分目录
│   ├── production/
│   └── staging/
└── infra/         # 基础设施层
```

## 核心概念

### 部署顺序

每个组件按职责拆分为最多三层，通过 Flux Kustomization 的 `dependsOn` 强制执行顺序：

```
setup/      # 前置资源：Namespace、Secret、ConfigMap、CRD 等
deploy/     # 应用主体：如 HelmRelease
resources/  # 依赖性资源：如必须等控制器就绪后才能创建的 CR
```

层的存在以**是否有部署顺序依赖**为准，而非资源类型。只有 `deploy/` 时直接平铺，无需建子目录。

`deploy/`、`setup/` 等目录下直接平铺 yaml 文件，无需手写 `kustomization.yaml`，Flux 会自动将目录下所有资源纳入管理。

#### 什么时候需要 setup？

Helm 在部署时可能需要某些资源已经存在，例如 redis 的 chart 依赖一个预先创建的 Secret 作为密码来源，此时 Secret 必须在 HelmRelease 之前就绪，单独放入 `setup/`：

```
infra/redis/
├── setup/    # Secret（redis 密码，helm 启动时引用）
└── deploy/   # HelmRelease
```

Namespace 本身不足以单独成层——Kubernetes 在创建资源时会隐式等待 Namespace 就绪，直接随 `deploy/` 一起放置即可。只有当其他资源明确依赖 `setup/` 中的内容时，这一层才有意义。

#### 什么时候需要 resources？

Operator 类应用需要控制器完全就绪后才能处理 CR，例如 cloudnative-pg 的 `Cluster` CR 必须等 Operator 部署完成才能创建：

```
infra/postgresql/
├── deploy/      # cloudnative-pg Operator
└── resources/   # Cluster CR、Database CR
```

### 环境差异

同一组件在不同环境的差异分两类，处理方式不同。

**值差异**用 `cluster-vars` substitution 处理，组件文件只写一份，变量在各环境的 ConfigMap 中定义：

```yaml
instances: ${POSTGRESQL_INSTANCES}  # production=3，staging=1
```

为了避免随组件增多导致单文件膨胀，建议在 `config/` 下按服务拆分 vars 文件，按需引用。

**结构差异**用 Kustomize overlay 处理，仅在该层存在结构差异时才引入，不做预防性拆分。postgresql 的 `resources/` 层是典型例子——不同环境挂载的服务子目录数量不一致，无法用变量表达：

```
resources/
├── base/        # 公共定义
├── production/  # 挂载全部服务
└── staging/     # 只挂载部分服务
```

能用 substitution 解决的绝不引入 overlay。

### 变量替换

Flux 通过 `postBuild.substituteFrom` 引用 ConfigMap 中的变量，在资源 apply 前完成替换：

```yaml
postBuild:
  substituteFrom:
    - kind: ConfigMap
      name: cluster-vars
```

所有 ConfigMap 建议放在 `flux-system` 命名空间下，与 Flux 控制平面保持一致。

#### 数字类型

在某些情况下，参数值可能期待字符串类型数值，但是 Flux 只能将其替换成数值形，在这种情况下我们需要将其使用单引号包裹：

```yaml
value: '"${POSTGRESQL_INSTANCES}"'
```

在某些情况下上述替换方式可能会导致意料之外的问题，可在变量值后加一个空格绕过，但是并不推荐大量使用这种格式：

```yaml
value: "${POSTGRESQL_INSTANCES} "
```

#### 转义

需要保留字面量 `${...}` 不被 Flux 替换时，使用双美元符号：

```yaml
value: $${ORIGINAL_VAR}  # 最终输出 ${ORIGINAL_VAR}
```

### Stack

多个服务组成一个逻辑单元时称为 Stack，以平铺命名区分：

```
apps/
├── lldap-authelia-shared/    # Stack 共用前置资源，等同于单组件的 setup/
├── lldap-authelia-lldap/     # 服务一
└── lldap-authelia-authelia/  # 服务二
```

连续出现相同前缀即表示同一 Stack，单服务组件只有一个目录没有重复前缀，两者一目了然。命名空间约定俗成使用 `<项目>-<stack名>`，例如 `lldap-authelia`。

### 订阅机制

`environments/<集群>/` 下的文件是 Flux Kustomization 对象，负责告知 Flux 监听哪个路径、以什么顺序部署，不承担任何 patch 逻辑，patch 应在组件目录内完成。

同一组件的多层订阅或同一 Stack 的多个服务建议合并到单个文件：

```
environments/production/
├── infra/
│   └── postgresql.yaml      # 包含 deploy、resources 两个 Kustomization 对象
└── apps/
    └── lldap-authelia-stack.yaml  # 包含 shared、lldap、authelia 三个 Kustomization 对象
```

文件内多个对象以 `---` 分隔，顺序即为阅读顺序，`dependsOn` 保证实际执行顺序。