---

## **Полное руководство по Keycloak + Spring Boot 3**

---

### **I. Настройка Keycloak**

#### **1. Установка и запуск**
- **Скачайте Keycloak** с [официального сайта](https://www.keycloak.org/downloads).
- **Запустите** (для разработки):
  ```bash
  bin/kc.sh start-dev --http-port 8080
  ```

---

#### **2. Создание Realm**
1. Откройте админ-панель: `http://localhost:8080/admin`.
2. **Создайте Realm**:
   - Название: `myrealm`.
   - Сохраните.

---

#### **3. Создание клиента (Client)**
1. **Clients -> Create**:
   - Client ID: `myclient`.
   - Client Protocol: `openid-connect`.
2. **Настройки клиента**:
   - Access Type: `confidential` (для client secret).
   - Valid Redirect URIs: `http://localhost:8081/*`.
   - Сохраните.
3. **Получите Client Secret**:
   - Вкладка "Credentials" -> Secret.

---

#### **4. Создание ролей (RBAC)**
1. **Realm Roles -> Create Role**:
   - Название: `USER`, `ADMIN`, `ORDER_MANAGER`.

---

#### **5. Создание пользователей**
1. **Users -> Add User**:
   - Username: `user1`.
   - Email: `user1@example.com`.
2. **Установите пароль**:
   - Вкладка "Credentials" -> Set Password.
3. **Назначьте роли**:
   - Вкладка "Role Mapping" -> Добавьте `USER`.

---

### **II. Интеграция Keycloak с Spring Boot 3**

#### **1. Зависимости (`pom.xml`)**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

---

#### **2. Конфигурация Spring Security**
```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        return http.build();
    }

    // Конвертер ролей из Keycloak
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter converter = new JwtGrantedAuthoritiesConverter();
        converter.setAuthorityPrefix("");
        converter.setAuthoritiesClaimName("roles");

        JwtAuthenticationConverter jwtConverter = new JwtAuthenticationConverter();
        jwtConverter.setJwtGrantedAuthoritiesConverter(converter);
        return jwtConverter;
    }
}
```

---

#### **3. `application.properties`**
```properties
# Keycloak
spring.security.oauth2.resourceserver.jwt.issuer-uri=http://localhost:8080/realms/myrealm
keycloak.auth-server-url=http://localhost:8080
keycloak.realm=myrealm
keycloak.resource=myclient
keycloak.credentials.secret=your-client-secret
```

---

### **III. Модели контроля доступа**

#### **1. RBAC (ролевая модель)**
- **Контроллер**:
  ```java
  @GetMapping("/admin")
  @PreAuthorize("hasRole('ADMIN')")
  public String adminEndpoint() {
      return "Admin access";
  }
  ```

---

#### **2. ABAC (атрибутная модель)**
- **Keycloak Policy (JavaScript)**:
  ```javascript
  // Политика: "Пользователь старше 18 лет"
  var age = user.getAttribute('age');
  if (age >= 18) {
      $evaluation.grant();
  }
  ```
- **Spring Boot**:
  ```java
  @GetMapping("/adult-content")
  @PreAuthorize("@abacService.isAdult(authentication)")
  public String adultContent() { ... }
  ```

---

#### **3. ACL (права на объекты)**
- **Сервис**:
  ```java
  @Service
  public class OrderService {
      public boolean isOrderOwner(Long orderId, String username) {
          Order order = orderRepository.findById(orderId).orElseThrow();
          return order.getOwner().equals(username);
      }
  }
  ```
- **Контроллер**:
  ```java
  @GetMapping("/orders/{id}")
  @PreAuthorize("@orderService.isOrderOwner(#id, authentication.name)")
  public Order getOrder(@PathVariable Long id) { ... }
  ```

---

### **IV. Пример для интернет-магазина**

#### **1. Сценарии**
- **Пользователь**: Чтение своих заказов (ACL).
- **Менеджер**: Чтение всех заказов (RBAC).
- **Админ**: Удаление заказов старше 30 дней (ABAC).

---

#### **2. Конфигурация Keycloak**
1. **Создайте ресурс** `/orders/{id}` в Authorization -> Resources.
2. **Создайте политики**:
   - `Is Order Owner`: Проверка `resource.owner == user.id`.
   - `Order Older Than 30 Days`: JavaScript с проверкой даты.

---

#### **3. Код Spring Boot**
```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @GetMapping("/{id}")
    @PreAuthorize("hasRole('USER') && @orderService.isOrderOwner(#id, authentication.name)")
    public Order getOrder(@PathVariable Long id) { ... }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') && @orderService.isOrderOlderThan30Days(#id)")
    public void deleteOrder(@PathVariable Long id) { ... }
}
```

---

### **V. Генерация сертификатов**

#### **1. TLS/HTTPS для Keycloak**
- **Создание самоподписанного сертификата**:
  ```bash
  openssl req -newkey rsa:2048 -nodes -keyout keycloak.key -x509 -days 365 -out keycloak.crt
  ```
- **Импорт в Keycloak**:
  ```bash
  keytool -import -alias keycloak -file keycloak.crt -keystore truststore.jks
  ```
- **Настройка в `standalone.xml`**:
  ```xml
  <security-realm name="SSLRealm">
      <server-identities>
          <ssl>
              <keystore path="keycloak.jks" password="changeit"/>
          </ssl>
      </server-identities>
  </security-realm>
  ```

---

#### **2. mTLS (двусторонняя аутентификация)**
- **Клиентский сертификат**:
  ```bash
  openssl req -newkey rsa:2048 -nodes -keyout client.key -out client.csr
  openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365
  ```
- **Настройка Keycloak**:
  - Включите "Client Certificate" в настройках Realm.
  - Импортируйте CA-сертификат в truststore.

---

#### **3. Подпись JWT (RSA)**
- **Генерация ключей**:
  ```bash
  openssl genpkey -algorithm RSA -out private-key.pem
  openssl rsa -pubout -in private-key.pem -out public-key.pem
  ```
- **Настройка Keycloak**:
  - Realm Settings -> Keys -> Add RSA Provider.

---

### **VI. Итог**
- **Keycloak** позволяет гибко настраивать политики доступа через RBAC, ABAC, ACL.
- **Spring Boot 3** интегрируется с Keycloak через `spring-security-oauth2`.
- **Сертификаты** (TLS, mTLS, JWT) обеспечивают безопасную аутентификацию.

**Пример полной конфигурации Keycloak**:
- Realm: `myrealm`.
- Client: `myclient` (confidential).
- Пользователи: `user1`, `admin`.
- Роли: `USER`, `ADMIN`.

**Для продакшена**:
- Используйте Let's Encrypt для TLS.
- Настройте HTTPS в Spring Boot через `server.ssl.*`.
