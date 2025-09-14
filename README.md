# Shop1.0

## Overview

This project is an online game e-commerce platform that supports core functionalities such as game browsing, user registration and login**, **shopping cart management, and order placement. The main technologies used in the project are listed as follows:

- Frontend: JSP, HTML/CSS, JavaScript
- Backend: Spring Framework, MyBatis
- Databases: MySQL (main), Redis (cache)
- Async Processing: Custom Redis-based event queue

## Overall Architecture

This project is built based on the **Monolithic Architecture**. It follows a classic **MVC(Model-View-Controller) architecture** commonly used in JavaWeb applications. 

The fronted is built using **JSP(JavaServer Pages)** framework. Because it allows server-side rendering of dynamic content and enables seamless integration with Java code. It is well-suited for traditional JavaWeb projects where frontend and backend are tightly coupled.

The backend is built on the **Spring Framework**. It provides a robust foundation for building scalable and modular Web app.

- Two types of databases are used in the system:
  - **Redis** is used as an in-memory cache to store frequnetly accessed data, such as shopping cart data, top-picking profucts and configuration values. Besides, it also used to implemented an event-driven mechanism to decouple time-consuming operations (e.g., sending verification emails) from user-facing requests. It can help reduce the load on the primary database and significantly improves the system's response time and user experience .
  - **MySQL** serves as the primary relational database. It is responsible for storing core business data such as user info, product details and order related info. It can guarantee strong consistency and reliable transactions. Since E-commerce systems always face the challenges of dealing with critical operations such as transactions and inventory management, it is important to guarantee ACID principle and data integrity.
- To interact with MySQL database, the project uses **MyBatis** as the persistence framework. MyBatis can simplifies database operations by mapping SQL queries to Java objects and offers a more flexible and fine-grained control over SQL. This is well-suited for achieving complex business logic such as dynamic SQL generation and multiple condition branches. 

<img width="562" height="347" alt="architecture-gameshop drawio" src="https://github.com/user-attachments/assets/28b14bfd-1648-4568-8f2a-2f1044217e9b" />


### Frontend & Backend Interaction

The frontend and the backend communicate via HTTP, using the standard request-response mechanism.

Taking the login logic as an example.

When the user click the login button on the login.jsp page, it will trigger an onclick event handler. This event calls the login() function, which is defined in the linked login.js file.

```jsp
<button type="submit" class="btnys" onclick="login()"><p class="btys">Login</p></button>
```

Inside the login() function, a $.post() request is sent to the backend's /login endpoint, along with the parameters.

```js
$.post("/login", {username: name, password: password, remember: remember}
```

On the backend, the request is handled by a controller method mapped to /login. The controller receives the input parameters and queries the database accordingly. Then it will send an approproate HTTP response back to the frontend.

```java
  @GetMapping(value = "login")
  public String login() {
...
      }
      return "login";
  }
```

### Event-driven mechanism with Redis

#### EventConsumer

When a request triggers an event, the system pushes the event into a Redis queue and returns a response immediately. A background consumer then asynchronously processes the event by dispatching it to the appropriate handler.This approach not only reduces the load on the primary database, but also significantly improves system responsiveness, scalability, and user experience. It has the following basic process:

1. EventConsumer initialization
   - Scans all beans implementing EventHandler via ApplicationContext
   - Call getSupportEvent() on each EventHandler
2. A thread pool is created to consume the events asynchronously. In the loop:
   - redisUtil.lpopObject() will continuously pops EventModel objects from Redis list
   - for each valid event, matching EventHandlers will be retrieved from handlers map
   - for each handler: threadPool.execute(new EventConsumerThread(handler, event))
   - each EventConsumerThread calls: handler.doHandler(event)
   - each EventHandler executes specific business logic

#### EventProducer

EventProducer is responsible for publishing events to the Redis queue. It serializes EventModel objects and pushes them to the "event" list key in Redis, to be later consumed asynchronously by EventConsumer.

## Use Case Demo: Forgot Password Flow

This section uses Forgot Password Flow as a use case demo. It will walk thorugh and demonstrate the password recovery process in the system, including frontend trigger, backend logic, and asynchronous email dispatching via Redis-backed event drive system.

<img width="821" height="437" alt="findPassword drawio" src="https://github.com/user-attachments/assets/a221fcd2-241d-4507-b6b1-1ee99b4bf8e1" />


1. The user click " forget password " in the login page, then click "send mail" button to get the verification code

   - this "href" link will ask the client to automatically sent a GET request to the defined URL

   ```jsp
   // Path: src/main/webapp/WEB-INF/jsp/login.jsp
   <a href="/user/findpassword">忘记密码？</a>
   ```

   - The findPassword() controller returned a "findpassword.jsp"

   ```java
   // Path: src/main/java/cn/cie/controller/UserController.java
   @GetMapping(value = "findpassword")
   public String findPassword() {
       return "findpassword";
   }
   ```

   - the "findpassword.jsp" page gave an "onclick=' ' " method, which will goes to the corresponding function defined by a js file

   ```jsp
   // Path: src/main/webapp/WEB-INF/jsp/findpassword.jsp
   <input id="btna" type="button" value="发送邮件" onclick="sendMail()">
   ```

   - the sendMail() funtion defined a POST request with a path

   ```javascript
   // Path: src/main/webapp/WEB-INF/js/findpassword.js
   function sendMail() {
       ...
       $.post("/user/sendfetchpwdmail", {email: email}, function (result) {
   ...
   }
   ```

   - the sendFetchPwdMail() controller called the function sendFetchPwdMail() defined by userService

   ```java
   // Path: src/main/java/cn/cie/controller/UserController.java
   @PostMapping(value = "sendfetchpwdmail")
   @ResponseBody
   public Result sendFetchPwdMail(String email) {
       return userService.sendFetchPwdMail(email);
   }
   ```

   - the sendFetchPwdMail() will generate the verification code with UUID and save the email and UUID to Redis with a TTL
   - It will also create an EventModel in the eventProducer asynchronously, and it will immediately send a successful result to the user

   ```java
   // Path: src/main/java/cn/cie/services/impl/UserServiceImpl.java
   public Result sendFetchPwdMail(String email) {
       ...// validate the format of the email
       String uuid = UUID.randomUUID().toString();
       // save the data to Redis with a timeout
       redisUtil.putEx(email, uuid, Validatecode.TIMEOUT);
       eventProducer.product(new EventModel(EventType.SEND_FIND_PWD_EMAIL).setExts("mail", user.getEmail()).setExts("code", uuid));
       return Result.success();
   }
   ```

   - the EventConsumer continuously polls the Redis queue for new EventModels. For each event, it looks up the list of handlers that support the event type (via getSupportEvent()), and then delegates the event to each corresponding handler for processing

   ```java
   // Path: src/main/java/cn/cie/event/handler/SendValidateMailHandler.java
   @Service
   public class SendValidateMailHandler implements EventHandler {
       public void doHandler(EventModel model) {
           MailUtil.sendValidateMail(model.getExts("mail"), model.getExts("code"));
       }
       public List<EventType> getSupportEvent() {
           return Arrays.asList(EventType.SEND_VALIDATE_EMAIL);
       }
   }
   ```

   - the senValidateailHandler calls the sendValidateMail() function from the MailUtil.java to execute the sendMail action

   ```java
   // Path: src/main/java/cn/cie/utils/MailUtil.java
   public static void sendMail(String user, String title, String content) {
   ...// the execution logic of send email
   }
   
   public static void sendValidateMail(String user, String code) {
   ...// add validate unique logic on the sendMail()
       sendMail(user, title, content);
   }
   ```

2. The user set new password

   - the "onclicn=' ' " action will goes to the findpassword() function defined in a javascript file

   ```jsp
   // Path: src/main/webapp/WEB-INF/jsp/findpassword.jsp
   // confirm sending the verification code
   <button type="submit" ... onclick="findpassword()">确定(Confirm)</button>
   ```

   - the findpassword() function defined a POST api with the request body {password, email, code}

   ```javascript
   // Path: src/main/webapp/WEB-INF/js/findpassword.js
   function findpassword() {
   ...
           $.post("/user/findpassword", {password: password, email: email, code: code}, function (result) {
   ...
   }
   ```

   - the controller will use the forgetPassword() function defined by userService

   ```java
   // Path: src/main/java/cn/cie/controller/UserController.java
   @PostMapping(value = "findpassword")
   @ResponseBody
   public Result findPassword(String password, String email, String code) {
       return userService.forgetPassword(password, email, code);
   }
   ```

   - userService.forgetPassword() will first validate the format of email and password
   - then it will get verification code from Redis, and validate if the codes matches. 
   - then it will update the user's password with the new password
   - after completing validation, it will delete the code from Redis.
   - Finally, it will send corresponding results to the user

   ```java
   // Path: src/main/java/cn/cie/services/impl/UserServiceImpl.java
   public Result forgetPassword(String password, String email, String code) {
       ...//validate the email and password
       String uuid = redisUtil.get(email);
       if (code != null && code.length() == 36 && code.equals(uuid)) {
           user.setPassword(PasswordUtil.pwd2Md5(password.replaceAll(" ", "")));
           if (1 == userMapper.update(user)) {
               redisUtil.delete(email);        // 验证成功后删除验证码
               return Result.success(user.getUsername());
           } else {
               return Result.fail(MsgCenter.ERROR);
           }
       }
       return Result.fail(MsgCenter.CODE_ERROR);
   }
   ```

## Screenshots

### Login page

<img width="1579" height="1060" alt="LoginPage" src="https://github.com/user-attachments/assets/adf43864-b2d5-4556-abfc-d106759b21ae" />


### Create your account page

<img width="1568" height="1048" alt="AccountPage" src="https://github.com/user-attachments/assets/a244b211-41fb-42ed-824d-11d98c00afee" />


### Home page

<img width="878" height="457" alt="HomePage01" src="https://github.com/user-attachments/assets/924f5b68-a375-4d72-91ed-e287224986fa" />
<img width="1039" height="1223" alt="HomePage02" src="https://github.com/user-attachments/assets/4b7e3b43-efca-4792-91ba-bd96033dac90" />


### Game product detail page
<img width="1261" height="1033" alt="GameProductDetailPage" src="https://github.com/user-attachments/assets/9b76c5e0-c2d6-484e-afb9-7869bbc086c4" />

