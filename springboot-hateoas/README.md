# SpringBoot3.1使用HATEOAS增强API的活力

> 示例项目：
> 
> JDK 17
> SpringBoot 3.1.4 (当前最新版本)
> 
> Spring HATEOAS 2.1.2
> 
> Kotlin 1.8.22

# 初始化项目

使用[Spring Initializr](https://start.spring.io/) 创建示例项目，使用Gradle管理项目，语言选择Kotlin，JDK 17，依赖选择Spring Web 、Spring HATEOAS ，其他都会自动引入。

生成的示例项目的build.gradle内容如下：
```java
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile  
  
plugins {  
    id 'org.springframework.boot' version '3.1.4'  
    id 'io.spring.dependency-management' version '1.1.3'  
    id 'org.jetbrains.kotlin.jvm' version '1.8.22'  
    id 'org.jetbrains.kotlin.plugin.spring' version '1.8.22'  
}  
  
group = 'cn.myplus.examples'  
version = '0.0.1-SNAPSHOT'  
  
java {  
    sourceCompatibility = '17'  
}  
  
repositories {  
    mavenCentral()  
}  
  
dependencies {  
    implementation 'org.springframework.boot:spring-boot-starter-hateoas'  
    implementation 'org.springframework.boot:spring-boot-starter-web'  
    implementation 'com.fasterxml.jackson.module:jackson-module-kotlin'  
    implementation 'org.jetbrains.kotlin:kotlin-reflect'  
    testImplementation 'org.springframework.boot:spring-boot-starter-test'  
}  
  
tasks.withType(KotlinCompile) {  
    kotlinOptions {  
       freeCompilerArgs += '-Xjsr305=strict'  
       jvmTarget = '17'  
    }  
}  
  
tasks.named('test') {  
    useJUnitPlatform()  
}
```

上面代码中引入了Log4j2做为日志输出框架，所以把spring-boot-starter-logging 做全局排除了。

使用Ideaj 打开项目后，会自动进行load gradle 项目，并下载依赖包。如果下载比较慢，可以参考[Gradle 国内加速](#Gradle国内加速)

# 编写代码

## 实体类

我们使用User对象做为示例，只有4个属性，id、name、loginName是必填属性，放到构造方法中，phone 是非必填属性，可以放到类中。在本示例项目没有做Controller之外的代码编写，不涉及保存数据库等操作。

```kotlin
# User.kt

package cn.myplus.examples.springboothateoas.user  
  
import java.io.Serializable  
  
/**  
 * @project springboot-hateoas  
 * @description 用户信息实体类  
 * @author libo  
 * @date 2023-10-03 10:26:32  
 */data class User(val id: String, val name: String, val loginName: String) : Serializable{  
    val phone: String? = null  
}

```

## Controller类

创建UserController.kt的文件，基本的注解与Java代码编写一样。
```kotlin

package cn.myplus.examples.springboothateoas.user
  
import org.slf4j.Logger  
import org.slf4j.LoggerFactory  
import org.springframework.beans.factory.annotation.Autowired  
import org.springframework.web.bind.annotation.RestController

@RestController  
class UserController(@Autowired val userService: UserService) {  
  
    private val logger: Logger = LoggerFactory.getLogger(UserController::class.java)

}

```

### 获取单个对象接口

hateoas风格的api主要是api的返回值中有相关的链接，使用者能够更容易使用。
在获取单个对象的api中，最常见的就是给出自身（self）的访问链接。

希望通过如下的请求地址
```
GET http://localhost:9999/user/admin  
Accept: application/json
```

返回单个对象(admin用户)的信息
```json
{
  "id": "admin",
  "name": "系统管理员",
  "loginName": "admin",
  "phone": null,
  "links": [
    {
      "rel": "self",
      "href": "http://localhost:9999/user/admin"
    }
  ]
}

```

在UserController 中加入获取单个对象的接口

```kotlin
@GetMapping("/user/{id}")  
fun getUser(@PathVariable("id") id: String): ResponseEntity<User> {  
    val user = getUserById(id)  
    return ResponseEntity<User>(user, HttpStatus.OK)  
}  
  
/**  
 * 获取单个实体，可以在Service类中实现。  
 */  
fun getUserById(id: String): User {  
    if (id.isBlank()) {  
        throw IllegalArgumentException("查询用户,参数错误.")  
    }  
    return User("admin", "系统管理员", "admin")  
}
```

访问 http://localhost:9999/user/admin 这个地址，返回值如下：

```json
{
  "id": "admin",
  "name": "系统管理员",
  "loginName": "admin",
  "phone": null
}
```

接口正常返回单个实体对象，但是没有HATEOAS 的内容，接下来我们就要加入HATEOAS的格式的返回值。

1. 首先创建一个UserModel类，这个类的属性与返回JSON的属性对应（多数情况下与User类属性相同）
```kotlin
package cn.myplus.examples.springboothateoas.user  
  
import org.springframework.hateoas.RepresentationModel  
  
/**  
 * @project myplus5  
 * @description 用记信息接口返回类  
 * @author libo  
 * @date 2023-10-03 15:32:58  
 */
 class UserModel : RepresentationModel<UserModel>() {  
  
    var id: String = ""  
  
    var name: String = ""  
  
    var loginName: String = ""  
  
    var phone: String? = null

}
```

这个类继承org.springframework.hateoas.RepresentationModel类，这个类是Spring Hateoas 提供的包装links的基本model类。继承这个类就可以为模型（UserModel）增加link信息了。

2. 修改getUser方法
```kotlin

    @GetMapping("/user/{id}")  
    fun getUser(@PathVariable("id") id: String): ResponseEntity<UserModel> {  
        val user = getUserById(id)  
        val userModel: UserModel = UserModel()  
        //加入自身链接  
        userModel.add(  
            WebMvcLinkBuilder.linkTo(  
                WebMvcLinkBuilder.methodOn(UserController::class.java).getUser(user.id)  
            ).withSelfRel()  
        )  
        // 以下为属性赋值  
        userModel.id = user.id  
        userModel.name = user.name  
        userModel.loginName = user.loginName  
        userModel.phone = user.phone  
        return ResponseEntity<UserModel>(userModel, HttpStatus.OK)    
    }
    
```

-  getUser方法返回值由ResponseEntity(User) 改为ResponseEntity(UserModel)
-  userModel 加入链接，是通UserController.getUser()方法自动生成链接

再次访问 http://localhost:9999/user/admin 这个地址，返回值如下：

```json
{
  "id": "admin",
  "name": "系统管理员",
  "loginName": "admin",
  "phone": null,
  "links": [
    {
      "rel": "self",
      "href": "http://localhost:9999/user/admin"
    }
  ]
}

```
已经加入self 的链接，只是在Controller 的返回值中加入了一些代码，其他都没变化。

### 代码优化

Spring HATEOAS 还提供了RepresentationModelAssemblerSupport这样的模型转类，只需要继承这个抽象类就可以快速的实现模型与实体的转换操作。
```kotlin
package cn.myplus.examples.springboothateoas.user  
  
import org.springframework.hateoas.CollectionModel  
import org.springframework.hateoas.server.mvc.RepresentationModelAssemblerSupport  
import org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.linkTo  
import org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.methodOn  
import org.springframework.stereotype.Component  
  
/**  
 * @project springboot-hateoas  
 * @description 用户实体与模型转换  
 * @author libo  
 * @date 2023-10-03 18:06:58  
 */@Component  
class UserModelAssembler :  
    RepresentationModelAssemblerSupport<User, UserModel>(UserController::class.java, UserModel::class.java) {  
    override fun toModel(entity: User): UserModel {  
        val userModel: UserModel = instantiateModel(entity)  
        userModel.add(linkTo(methodOn(UserController::class.java).getUser(entity.id)).withSelfRel())  
        userModel.id = entity.id  
        userModel.name = entity.name  
        userModel.loginName = entity.loginName  
        userModel.phone = entity.phone  
  
        return userModel;  
    }  
  
    override fun toCollectionModel(entities: MutableIterable<User>): CollectionModel<UserModel> {  
        val userModels: CollectionModel<UserModel> = super.toCollectionModel(entities)  
        userModels.add(methodOn(UserController::class.java).getUsers(null)?.let { linkTo(it).withSelfRel() })  
        return userModels  
    }  
  
}

```

这个类把UserController和UserModel 做为构造参数，创建一个类，实现toModel方法，就可以实现model转换的操作。

```kotlin
@RestController  
class UserController(@Autowired val userService: UserService) {  
  
    @Autowired  
    private lateinit var userModelAssembler: UserModelAssembler  
    
    /**  
     * @return 用户信息.  
     * @param id 用户id  
     * @author libo  
     */    @GetMapping("/user/{id}")  
    fun getUser(@PathVariable("id") id: String): ResponseEntity<UserModel> {  
        val user = getUserById(id)  
        return ResponseEntity<UserModel>(userModelAssembler.toModel(user), HttpStatus.OK)  
    }

```

这样即可以达到代码的复用，Controller类看来也不是那么乱了。

完整代码已放到[gitee仓库](https://gitee.com/dadaoziran/examples/tree/master)，github仓库

# Gradle国内加速
因为使用Gradle 管理项目，在下Gradle 及依赖的jar还比较慢。可以配置国内加速的办法。

1. 先配置GRADLE_USER_HOME环境变量到Maven的仓库地址，注意是仓库地址（里面全是jar包的那个），不是maven的主目录

```shell
export GRADLE_USER_HOME="/env/repository" 
```

2. 下载Gradle到本地（/env/gradle-8.2.1），解压后修改init.d目录下的init.gradle 文件，没有创建即可。

```groovy
allprojects {
    repositories {
        maven { url '/env/repository'}
        mavenLocal()
        maven { name "Alibaba" ; url "https://maven.aliyun.com/nexus/content/groups/public/" }
        mavenCentral()
    }

    buildscript { 
        repositories { 
            maven { name "Alibaba" ; url 'https://maven.aliyun.com/nexus/content/groups/public/' }
        }
    }
}
```

3. 在没有打开任何项目的情况下（如果打开项目只能设置当前项目的gradle，不是全局的）打开ideaj-->settings--> 搜索gradle，在Gradle user home 选择前面下载的Gradle解压后的地址(/env/gradle-8.2.1)，这样在后续每个项目都不用单独下载Gradle了，而且能够使用aliyun的镜像了。
