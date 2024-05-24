# 初心者向け　Repostitoryパターンを解説してみた

## はじめに
以下の人向けの記事です

- webアプリケーションのバックエンドをちょっと勉強した人
- MVCをちょっと理解した人

  
## Repostiroryパターンとは

Repostitoryパターンはデザインパターンというものの一種です。デザインパターンとは簡単に言えばよく使える設計の構成パターンをまとめたものです。RepositoryパターンはMVCのC(Controller)にあたる部分をより細分化して使いやすくするものです。

```
  +--------------------+    +--------------------+
  |       Model        |    |       View         |
  +--------------------+    +--------------------+
  | - data             |    |                    |
  +--------------------+    +--------------------+
  | + getData()        |    | + update(data)     |
  | + setData(data)    |    +--------------------+
  +---------+----------+               ^
            ^                          |
            |                          |
            |                          |
  +---------+----------+               |
  |    Controller      +---------------+
  +--------------------+
  | - model : Model    |
  | - view : View      |<---ここを細分化する
  +--------------------+
  | + handleRequest()  |
  +--------------------+

```

## 構成

以下に構成を示します。
Repositoryパターンは従来のMVCでコントローラが担っていたサービスロジックやデータアクセスを細分化していきます。

```
  +--------------------+    +--------------------+
  |       Model        |    |       View         |
  +--------------------+    +--------------------+
  | - data             |    |                    |
  +--------------------+    +--------------------+
  | + getData()        |    | + update(data)     |
  | + setData(data)    |    +--------------------+
  +---------+----------+               ^
            ^                          |
            |                          |
            |                          |
  +---------+----------+               |
  |    Repository      |               |
  +--------------------+               |
  |                    |               |
  +--------------------+               |
  | + fetchData()      |               |
  | + storeData(data)  |               |
  +---------+----------+               |
            ^                          |
            |                          |
            |                          |
  +---------+----------+               |
  |RepositoryInterface |               |
  +--------------------+               |
  |                    |               |
  +--------------------+               |
  | + fetchData()      |               |
  | + storeData(data)  |               |
  +---------+----------+               |
            ^                          |
            |                          |
            |                          |
  +---------+----------+               |
  |      Service       |               |
  +--------------------+               |
  | - repository :     |               |
  | RepositoryInterface|               |
  |                    |               |
  +--------------------+               |
  | + getData()        |               |
  | + saveData(data)   |               |
  +---------+----------+               |
            ^                          |
            |                          |
            |                          |
  +---------+----------+               |
  |    Controller      +---------------+
  +--------------------+
  | - service : Service|
  +--------------------+
  | + handleRequest()  |
  +--------------------+

```

### 従来のMVCのときのControllerとの違い

#### 従来のController

従来のMVCではControllerは以下のような役割を担っていました。

- リクエストの入力処理
- ビジネスロジック
- データベースからのデータの取得

ビジネスロジックと聞くとあまり聞き慣れないかもしれませんが、データを取得してからレスポンスするまでのデータ処理のことだと思っておいてください。詳しく知りたい方は[こちら](https://qiita.com/os1ma/items/25725edfe3c2af93d735)を参考にしてみてください。 


#### この構成のデメリット

このように一つのクラスが複数の役割を持つのは[単一責任の原則](https://qiita.com/conkon326/items/1ce5a4e5b8430c4e26d5)に反していて、コードの拡張や保守がしにくくなってしまいます。要は一つのクラスに多くの処理がまとまっているとデバッグ等がしにくくなるということです。今のコントローラには三つも機能があるのでデバッグもしにくくなるわけです。

```php:UserController.php

@RestController
@RequestMapping("/users")
public class UserController {
    private List<User> users = new ArrayList<>();

    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(@PathVariable int id) {
        User user = users.stream().filter(u -> u.getId() == id).findFirst().orElse(null);
        return ResponseEntity.ok(user);
    }

    @GetMapping
    public ResponseEntity<List<User>> getAllUsers() {
        return ResponseEntity.ok(users);
    }

    @PostMapping
    public ResponseEntity<Void> createUser(@RequestBody User user) {
        users.add(user);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable int id) {
        users.removeIf(user -> user.getId() == id);
        return ResponseEntity.noContent().build();
    }
}
```


#### Repostiryパターンでの改善策
Repositoryパターンではこれを以下のように分けてあげます

- リクエストの入力処理:Controller
- ビジネスロジック:Service
- データベースからのデータの取得:Repository

このように分けてあげることでそれぞれのクラスが責任を一つずつ負うことになり、拡張性、保守性が向上します。

```php:UserController.php

@RestController
@RequestMapping("/users")
@RestController
@RequestMapping("/users")
public class UserController {
    private UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(@PathVariable int id) {
        User user = userService.getUserById(id);
        return ResponseEntity.ok(user);
    }

    @GetMapping
    public ResponseEntity<List<User>> getAllUsers() {
        List<User> users = userService.getAllUsers();
        return ResponseEntity.ok(users);
    }

    @PostMapping
    public ResponseEntity<Void> createUser(@RequestBody User user) {
        userService.createUser(user);
        return ResponseEntity.status(HttpStatus.CREATED).build();
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable int id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }
}

```

```php:UserService.php
public class UserService {
    private UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User getUserById(int id) {
        return userRepository.findById(id);
    }

    public List<User> getAllUsers() {
        return userRepository.findAll();
    }

    public void createUser(User user) {
        userRepository.save(user);
    }

    public void deleteUser(int id) {
        userRepository.deleteById(id);
    }
}

```

```php:UserRepository.php
public interface UserRepository {
    User findById(int id);
    List<User> findAll();
    void save(User user);
    void deleteById(int id);
}

```

```php:UserRepositoryImpl.php
public class UserRepositoryImpl implements UserRepository {
    private List<User> users = new ArrayList<>();

    @Override
    public User findById(int id) {
        return users.stream().filter(user -> user.getId() == id).findFirst().orElse(null);
    }

    @Override
    public List<User> findAll() {
        return users;
    }

    @Override
    public void save(User user) {
        users.add(user);
    }

    @Override
    public void deleteById(int id) {
        users.removeIf(user -> user.getId() == id);
    }
}


```
##### Repository Interfaceの役割

クラス図を見るとRepository Interfaceが定義されています。ただwebアプリケーションをそのまま実装するとこのインターフェースは役割が無い様に見えると思います。しかし、複数のデータベースを実装するときにこのインターフェースはその役割を果たします。
このように複数DBを使ったり切り替える場合にインターフェースを定義しておくと保守性がさらに向上します
```

  +--------------------+
  | RepositoryInterface|
  +--------------------+
  |                    |
  +--------------------+
  | + fetchData()      |
  | + storeData(data)  |
  +---------+----------+
            ^
            |-------------------------------|--------------------------------|                                
            |                               |                                |
  +---------+----------+          +--------------------+          +--------------------+
  |   MySQLRepository  |          | PostgreSQLRepository|          | MongoDBRepository  |
  +--------------------+          +--------------------+          +--------------------+
  |                    |          |                    |          |                    |
  +--------------------+          +--------------------+          +--------------------+
  | + fetchData()      |          | + fetchData()      |          | + fetchData()      |
  | + storeData(data)  |          | + storeData(data)  |          | + storeData(data)  |
  +--------------------+          +--------------------+          +--------------------+
            |                                |                               |
            |                                |                               |
            ・・・それぞれのDBに接続
````

### まとめ
以上、簡単なRepositoryパターンのまとめでした！本来はもっと奥深いものだったりしますがまずはその触りになれればと思います！
