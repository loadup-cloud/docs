# 命令参考手册

> 常用的构建、测试、格式化、质量检查命令，可直接复制执行。

---

## 构建

```bash
# 全量构建并运行所有测试
mvn clean verify

# 全量构建，跳过测试（仅打包）
mvn clean install -DskipTests

# 全量构建，跳过测试和 Spotless 检查（最快，仅验证编译）
mvn clean install -DskipTests -Dspotless.check.skip=true

# 单模块构建（-pl 指定模块，-am 构建其所有上游依赖）
mvn clean verify -pl modules/loadup-modules-config/loadup-modules-config-app -am

# 强制刷新依赖（SNAPSHOT 更新后执行）
mvn clean install -U -DskipTests

# 仅编译，不打包
mvn compile
```

---

## 测试

```bash
# 运行单模块所有测试
mvn test -pl modules/loadup-modules-config/loadup-modules-config-test

# 运行指定测试类
mvn test -pl modules/loadup-modules-config/loadup-modules-config-test \
    -Dtest=ConfigItemServiceIT

# 运行指定测试方法
mvn test -pl modules/loadup-modules-config/loadup-modules-config-test \
    -Dtest="ConfigItemServiceIT#create_shouldPersist_whenValidCommand"

# 只运行单元测试（文件名不含 IT）
mvn test -pl modules/loadup-modules-config/loadup-modules-config-test \
    -Dtest="*Test"

# CI 模式（禁用 Testcontainers 复用）
mvn test -Dspring.profiles.active=ci
```

---

## 代码格式化（Spotless）

```bash
# 检查格式（只读，CI 使用，不修改文件）
mvn spotless:check

# 自动修复格式（本地开发使用）
mvn spotless:apply

# 单模块格式修复
mvn spotless:apply -pl modules/loadup-modules-config/loadup-modules-config-app

# 跳过格式检查（临时调试用，不建议）
mvn verify -Dspotless.check.skip=true
```

---

## 静态分析

```bash
# SpotBugs 检查
mvn spotbugs:check -DskipTests

# PMD 检查
mvn pmd:check -DskipTests

# Checkstyle 检查
mvn checkstyle:check -DskipTests

# 一次运行所有质量门
mvn verify -P github
```

---

## 依赖管理

```bash
# 查看某模块的完整依赖树
mvn dependency:tree -pl modules/loadup-modules-config/loadup-modules-config-app

# 查找某依赖在哪个模块中被引入
mvn dependency:tree -pl modules/loadup-modules-config/loadup-modules-config-app \
    -Dincludes=com.mybatis-flex:*

# 检查依赖冲突
mvn dependency:analyze -pl modules/loadup-modules-config/loadup-modules-config-app

# 检查 BOM 有效 POM
mvn help:effective-pom -pl loadup-dependencies
```

---

## MyBatis-Flex APT（代码生成）

```bash
# 触发 APT 生成（DO 修改后执行）
mvn generate-sources -pl modules/loadup-modules-config/loadup-modules-config-infrastructure

# 完整重新生成（清除旧 target）
mvn clean generate-sources -pl modules/loadup-modules-config/loadup-modules-config-infrastructure
```

> APT 生成文件位于 `target/generated-sources/annotations/`，包含 `Tables.java` 和各 `XxxDOTableDef.java`。

---

## License 头管理

```bash
# CI 自动插入 License 头（本地不需要手动执行）
mvn license:format

# 检查 License 头是否完整
mvn license:check
```

> ⚠️ 不要在 Java 文件中手写 `/*- #%L ... #L% */`，CI 会重复插入导致编译失败。

---

## 发布

```bash
# 发布到 OSSRH SNAPSHOT
mvn clean deploy -P github -DskipTests

# Release 正式版（需要 GPG 签名配置）
mvn clean deploy -P release -DskipTests
```

---

## Git 工作流

```bash
# 标准提交格式（Conventional Commits）
git commit -m "feat(config): add config item expiry support"
git commit -m "fix(upms): correct role permission cascade delete"
git commit -m "refactor(infra): extract GatewayImpl base class"
git commit -m "test(config): add integration test for ConfigItemService"
git commit -m "docs: update component integration quick ref"

# 类型前缀速查
# feat     新功能
# fix      Bug 修复
# refactor 重构（不改变功能）
# test     测试相关
# docs     文档变更
# build    构建脚本变更
# ci       CI/CD 配置变更
# chore    其他维护性修改

# 创建 feature 分支
git checkout -b feat/modules-xxx-add-yyy

# Rebase 到最新 develop
git fetch origin && git rebase origin/develop

# 安装 Git Hooks（首次 clone 后执行）
./install-git-hooks.sh
```

---

## 常见组合场景

```bash
# 新模块开发完成后的完整验证
mvn spotless:apply && mvn clean verify

# 修改 DO 后重新生成并运行测试
mvn clean generate-sources -pl <infra-module> -am && \
mvn test -pl <test-module>

# 本地模拟 CI 全量检查
mvn com.diffplug.spotless:spotless-maven-plugin:check -B -P github && \
mvn clean verify -B -P github
```
