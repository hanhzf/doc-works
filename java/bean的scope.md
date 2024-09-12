这四种 **Spring Bean 作用域** 各有适用的场景，根据应用程序的需求和类的生命周期，选择适合的作用域是很重要的。下面是这四种作用域的典型使用场景：

### 1. **Singleton（单例，默认）**

#### **使用场景**：
- **无状态服务**：通常用于无状态的服务类或 DAO（数据访问对象）类。这些类不保存特定用户或请求的状态，只是执行业务逻辑或数据访问操作，适合单例模式。例如，服务类、数据库访问类（如 `Service`、`Repository`）。
- **全局共享的资源**：如果某个类的实例需要在整个应用中共享，且类的状态与请求或用户无关，单例是最合适的选择。

#### **典型场景**：
- **服务层类**：`UserService`、`OrderService` 等类，处理业务逻辑，无需为每个请求或会话创建多个实例。
- **数据库访问类**：`UserRepository`、`ProductRepository` 等类，用于数据库操作，不需要多个实例。
  
#### **示例**：
```java
@Service
public class UserService {
    public void performAction() {
        System.out.println("Performing action");
    }
}
```
- Spring 容器只会创建一个 `UserService` 实例，整个应用中共享使用。

---

### 2. **Prototype（原型）**

#### **使用场景**：
- **有状态的 Bean**：如果某个类保存了请求特定的数据或状态，每次使用时需要新的实例，就可以使用原型作用域。
- **需要频繁创建的对象**：例如，大量临时对象，每次请求需要全新实例的情况，适合用原型模式。
- **非线程安全的类**：如果某个类不是线程安全的，最好不要使用单例模式，应该为每个请求生成一个新实例。

#### **典型场景**：
- **临时服务类**：例如生成报表的类，每次调用都需要不同的状态。
- **算法类**：某些复杂的算法类，每次需要一个新的实例进行计算。
  
#### **示例**：
```java
@Service
@Scope("prototype")
public class ReportService {
    public Report generateReport() {
        return new Report();
    }
}
```
- 每次请求 `ReportService` 时，Spring 都会创建一个新的 `ReportService` 实例。

---

### 3. **Request（Web 应用程序中使用）**

#### **使用场景**：
- **与 HTTP 请求相关的 Bean**：当你需要每个 HTTP 请求都生成一个新的 Bean 实例时，使用 `request` 作用域。这个作用域适用于 Web 应用程序，它确保在同一个 HTTP 请求的过程中，Bean 是同一个实例，但在不同请求之间则是不同的实例。
- **请求级别的服务**：适用于需要在同一个 HTTP 请求中共享的服务，比如处理请求参数、计算某些特定数据等。

#### **典型场景**：
- **HTTP 请求处理类**：如处理表单数据、验证请求等类，它们的生命周期只需要在一次 HTTP 请求中有效。
- **与请求相关的用户上下文类**：比如存储请求中的用户数据或身份认证信息。

#### **示例**：
```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestScopedBean {
    public String getData() {
        return "Data specific to this request";
    }
}
```
- 每个 HTTP 请求都会创建一个新的 `RequestScopedBean` 实例，在整个请求中共享该实例。

---

### 4. **Session（Web 应用程序中使用）**

#### **使用场景**：
- **与用户会话相关的 Bean**：当你希望某个对象在一个 HTTP 会话（Session）期间共享，而不同会话之间有不同的实例时，使用 `session` 作用域。适用于用户登录状态、购物车等场景。
- **会话级别的服务**：例如，需要在整个用户会话过程中保留某些状态的类。

#### **典型场景**：
- **用户会话信息类**：比如存储用户登录状态、购物车、浏览历史等对象，这些对象在会话结束后可以销毁。
- **用户首选项管理类**：如记录用户在整个会话中的行为偏好。

#### **示例**：
```java
@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class SessionScopedBean {
    private String userPreferences;

    public String getUserPreferences() {
        return userPreferences;
    }

    public void setUserPreferences(String preferences) {
        this.userPreferences = preferences;
    }
}
```
- `SessionScopedBean` 的实例在整个用户的 HTTP 会话期间共享，不同的会话有不同的实例。

---

### **总结**：

| 作用域       | 使用场景                                                      | 特点                                                 |
|--------------|---------------------------------------------------------------|------------------------------------------------------|
| **Singleton**| 无状态服务类、共享资源类                                        | 整个 Spring 容器中只存在一个实例，全局共享            |
| **Prototype**| 有状态的 Bean，频繁创建的对象，非线程安全的类                    | 每次请求 Bean 时，都会创建一个新的实例                |
| **Request**  | 每个 HTTP 请求都需要新的实例的 Bean，如处理请求参数的类          | 每个 HTTP 请求期间是同一个实例，不同请求则不同实例    |
| **Session**  | 需要在整个用户会话中共享的 Bean，如用户登录状态、购物车等        | 在同一个用户会话中共享实例，不同会话则使用不同实例    |

选择合适的作用域取决于你的类的生命周期和具体应用场景。