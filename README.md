# 高精度GNSS解算服务

该解算服务采用服务端和客户端两种方式集成使用，服务端使用docker容器化方式镜像部署，客户端使用sdk方式项目依赖使用。

官网：https://resolve-doc.navwhu.com


## 服务端部署

服务端需要依赖mysql数据库和rabbitmq消息队列环境，若项目中无此两个环境需自行安装一下，当然教程后面也会给出相应的安装步骤，如项目中已经有了环境则不需要再安装了。

mysql数据库中需要创建一个nav_resolve的数据库即可，建表容器内部脚本会自动创建。
rabbitmq消息队列需要增加延迟消息的插件。

1、docker-compose部署, 带上全部环境的脚本
```yaml
version: "3"

services:
  resolve-server:
    image: registry.cn-hangzhou.aliyuncs.com/navmg/resolve:1.0.2
    container_name: resolve-server
    hostname: resolve-server
    restart: always
    volumes:
      - ${RESOLVE_LOGBACK}:/home/resolve/logback
      - ${RESOLVE_RAW_PATH}:/app/raw
      - ${RESOLVE_BRDC_PATH}:/app/brdc
      - ${RESOLVE_RTCM_PATH}:/app/rtcm
      - ${RESOLVE_CACHE_PATH}:/app/cache
      - ${RESOLVE_QC_PATH}:/app/qc
      - ${RESOLVE_LICENSE_PATH}:/app/license.lic
    ports:
      - ${RESOLVE_PORT}:9965
    environment:
      - MYSQL_URL=jdbc:mysql://${MYSQL_IP}:${MYSQL_PORT}/nav_resolve?useSSL=false&characterEncoding=utf-8&useTimezone=true&serverTimezone=GMT%2B8&allowPublicKeyRetrieval=true
      - MYSQL_USERNAME=root
      - MYSQL_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - RABBITMQ_SERVER=${RABBITMQ_SERVER}
      - RABBITMQ_PORT=${RABBITMQ_PORT}
      - RABBITMQ_USERNAME=${RABBITMQ_USERNAME}
      - RABBITMQ_PASSWORD=${RABBITMQ_PASSWORD}
      - RABBITMQ_PREFETCH=4
    depends_on:
      - mysql
      - rabbitmq
    networks:
      - iot-net

  mysql:
    container_name: mysql
    hostname: mysql
    image: mysql:8
    ports:
      - ${MYSQL_PORT}:3306
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_ROOT_HOST: '%'
      MYSQL_USER: ${MYSQL_USERNAME}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - ${MYSQL_DATA}:/var/lib/mysql
    command: --default-authentication-plugin=mysql_native_password --character-set-server=utf8mb4 --collation-server=utf8mb4_general_ci --default-time-zone='+8:00' --max_connections=1000 --innodb_lock_wait_timeout=500
    networks:
      - iot-net

  rabbitmq:
    container_name: rabbitmq
    hostname: rabbitmq
    image: rabbitmq:3-management
    ports:
      - ${RABBITMQ_PORT}:5672
      - ${RABBITMQ_WEB_PORT}:15672
      - "4369:4369"
      - "25672:25672"
    restart: always
    environment:
      TZ: Asia/Shanghai
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: 1qaz@WSX
    volumes:
      - ${RABBITMQ_DATA}:/var/lib/rabbitmq
      - ${RABBITMQ_CONF}:/etc/rabbitmq/rabbitmq.conf
      - ${RABBITMQ_COOKIE}:/var/lib/rabbitmq/.erlang.cookie
      - ${RABBITMQ_DELAY_PLUGIN}:/plugins/rabbitmq_delayed_message_exchange-3.11.1.ez
    networks:
      - iot-net

networks:
  iot-net:
    driver: bridge
```

2、docker-compose部署, 如你的项目中已有mysql和rabbitmq环境，只需要解算镜像执行下面即可
```yaml
version: "3"

services:
  resolve-server:
    image: registry.cn-hangzhou.aliyuncs.com/navmg/resolve:1.0.0
    container_name: resolve-server
    hostname: resolve-server
    restart: always
    volumes:
      - ${RESOLVE_LOGBACK}:/home/resolve/logback
      - ${RESOLVE_RAW_PATH}:/app/raw
      - ${RESOLVE_BRDC_PATH}:/app/brdc
      - ${RESOLVE_RTCM_PATH}:/app/rtcm
      - ${RESOLVE_CACHE_PATH}:/app/cache
      - ${RESOLVE_QC_PATH}:/app/qc
      - ${RESOLVE_LICENSE_PATH}:/app/license.lic
    ports:
      - ${RESOLVE_PORT}:9965
    environment:
      - MYSQL_URL=jdbc:mysql://${MYSQL_IP}:${MYSQL_PORT}/nav_resolve?useSSL=false&characterEncoding=utf-8&useTimezone=true&serverTimezone=GMT%2B8&allowPublicKeyRetrieval=true
      - MYSQL_USERNAME=root
      - MYSQL_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - RABBITMQ_SERVER=${RABBITMQ_SERVER}
      - RABBITMQ_PORT=${RABBITMQ_PORT}
      - RABBITMQ_USERNAME=${RABBITMQ_USERNAME}
      - RABBITMQ_PASSWORD=${RABBITMQ_PASSWORD}
      - RABBITMQ_PREFETCH=4
    networks:
      - iot-net

networks:
  iot-net:
    driver: bridge
```

3、环境变量解释说明

| **变量**  | **说明**            |
|---------|-------------------|
| RESOLVE_LOGBACK    | 服务日志文件根路径         |
| RESOLVE_RAW_PATH | 原始观测文件根路径         |
| RESOLVE_BRDC_PATH | 星历存储根路径           |
| RESOLVE_RTCM_PATH | 下载文件存储根路径         |
| RESOLVE_CACHE_PATH | 服务缓存存储根路径         |
| RESOLVE_QC_PATH | 质量分析文件根路径         |
| RESOLVE_LICENSE_PATH | 许可证的绝对路径          |
| MYSQL_URL | mysql数据库连接        |
| MYSQL_USERNAME | mysql数据库用户名       |
| MYSQL_PASSWORD | mysql数据库密码        |
| RABBITMQ_SERVER | rabbitmq的地址       |
| RABBITMQ_PORT | rabbitmq的端口       |
| RABBITMQ_USERNAME | rabbitmq的用户名      |
| RABBITMQ_PASSWORD | rabbitmq的密码       |
| RABBITMQ_PREFETCH | rabbitmq同时消费的最大数量 |

## 客户端部署

1、添加java sdk依赖
```xml
       <dependency>
            <groupId>com.navmg</groupId>
            <artifactId>resolve-lib</artifactId>
            <version>1.0.2</version>
        </dependency>
```

2、项目中配置添加
application.properties文件中做配置，配置信息来自服务端
```properties
resolve.enable=true
resolve.base-url=http://127.0.0.1:9965
resolve.api-key=xxxxxxxxxxx
resolve.rabbitmq-host=127.0.0.1
resolve.rabbitmq-port=5672
resolve.rabbitmq-username=admin
resolve.rabbitmq-password=admin
resolve.rabbitmq-virtual-host=/
```

代码中直接配置引用
```java
    import org.springframework.beans.factory.annotation.Autowired;

    @Autowired
    private ResolveService resolveService;

```

3、调用方法
```java
public interface ResolveService {
    
    /** 设置解算结果的回调 */
    void setSolveCallback(ISolveCallback iSolveCallback);

    /** 设置质量分析结果的回调 */
    void setQcbisCallback(IQcbisCallback iQcbisCallback);

    /** 设置概略坐标结果的回调 */
    void setSppCallback(ISppCallback iSppCallback);

    /** 检查license是否到期信息 */
    LicenseVo checkLicense();

    /** 获取质量分析库的版本 */
    String qcVersion();

    /** 获取滤波库的版本 */
    String filterVersion();

    /** 获取解算库的版本 */
    String rtkconvVersion();

    /** 添加/修改设备 */
    String device(Device device);

    /** 根据guid获取设备 */
    Device getDeviceByGuid(String guid);

    /** 根据guid删除设备 */
    boolean deleteDevice(String guid);

    /** 添加/修改解算 */
    String solve(Solve solve);

    /** 根据guid获取解算 */
    Solve getSolveByGuid(String guid);

    /** 根据guid删除解算 */
    boolean deleteSolve(String guid);

}

```
