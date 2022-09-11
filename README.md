# Codables-and-Networking

## Codables

### Decoder, Encoder
```
let encoder = JSONEncoder()
let decoder = JSONDecoder()
```

### Simple
```
struct User: Codable {
  var id: String
  var name: String
}

let user_data = """
{
  "id": "abcdefg",
  "name": "name of user"
}
""".data(using: .utf8)!

let userDTO = try! decoder.decode(User.self, from: user_data)
```

### Nested Types
```
struct User: Codable {
  var id: String
  var name: String
}

struct Post: Codable {
  var id: String
  var authorUser: User
  var title: String
  var content: String
}

let post_data = """
{
  "id": "12345",
  "title": "post title",
  "content": "post content",
  "authorUser": {
    "id": "abcdefg",
    "name": "name of user"
  }
}
""".data(using: .utf8)!

let postDTO = try! decoder.decode(Post.self, from: post_data)
```

### Snake Case
```
struct User: Codable {
  var id: String
  var name: String
}

struct Post: Codable {
  var id: String
  var authorUser: User
  var title: String
  var content: String
}

let post_data = """
{
  "id": "12345",
  "title": "post title",
  "content": "post content",
  "author_user": {
    "id": "abcdefg",
    "name": "name of user"
  }
}
""".data(using: .utf8)!

decoder.keyDecodingStrategy = .convertFromSnakeCase

let postDTO = try decoder.decode(Post.self, from: post_data)
```

### Coding Keys
```
struct User: Codable {
  var id: String
  var name: String
}

struct Post: Codable {
  var id: String
  var author: User
  var title: String
  var content: String

  enum CodingKeys: String, CodingKey {
    case id, author = "author-user", title, content
  }
}

let post_data = """
{
  "id": "12345",
  "title": "post title",
  "content": "post content",
  "author-user": {
    "id": "abcdefg",
    "name": "name of user"
  }
}
""".data(using: .utf8)!

let postDTO = try decoder.decode(Post.self, from: post_data)
```

### Flat JSON
```
struct User: Codable {
  var id: String
  var name: String
}

struct Post: Decodable {
  var id: String
  var author: User
  var title: String
  var content: String

  enum CodingKeys: String, CodingKey {
    case id, title, content, author_id, author_name
  }

  init(from decoder: Decoder) throws {
    let container = try decoder.container(keyedBy: CodingKeys.self)

    self.id = try container.decode(String.self, forKey: .id)
    self.title = try container.decode(String.self, forKey: .title)
    self.content = try container.decode(String.self, forKey: .content)

    let author_id = try container.decode(String.self, forKey: .author_id)
    let author_name = try container.decode(String.self, forKey: .author_name)

    self.author = User(id: author_id, name: author_name)
  }
}

let post_data = """
{
  "id": "12345",
  "author_id": "asdfg",
  "author_name": "user's name",
  "title": "post title",
  "content": "post content"
}
""".data(using: .utf8)!

let postDTO = try decoder.decode(Post.self, from: post_data)
```

### Nested JSON
```
struct User: Codable {
  var id: String
  var name: String
}

struct Post: Decodable {
  var id: String
  var author: User
  var title: String
  var content: String

  enum CodingKeys: String, CodingKey {
    case id, title, content, author
  }

  enum AuthorKeys: String, CodingKey {
    case user
  }

  init(from decoder: Decoder) throws {
    let container = try decoder.container(keyedBy: CodingKeys.self)

    self.id = try container.decode(String.self, forKey: .id)
    self.title = try container.decode(String.self, forKey: .title)
    self.content = try container.decode(String.self, forKey: .content)

    let authorContainer = try container.nestedContainer(keyedBy: AuthorKeys.self, forKey: .author)

    self.author = try authorContainer.decode(User.self, forKey: .user)
  }
}

let post_data = """
{
  "id": "12345",
  "title":"post title",
  "content":"post content",
  "author": {
    "user": {
      "id": "asdf",
      "name": "user's name"
    }
  }
}
""".data(using: .utf8)!

let postDTO = try decoder.decode(Post.self, from: post_data)
```

## Networking / Alamofire

### request / responseData
```
AF.request("https://jsonplaceholder.typicode.com/posts")
  .responseData { response in
    if let data = response.data,
      let posts = try? JSONDecoder().decode([Post].self, from: data)
    {
      posts.forEach { post in
        print(post)
      }
    } else {
      print("ERROR")
    }
}
```
```
struct Post: Decodable {
  let id: Int
  let userId: Int
  let title: String
  let body: String
}
```

### responseDecable
```
AF.request("https://jsonplaceholder.typicode.com/posts")
  .responseDecodable(of: [Post].self) { response in
    if let posts = response.value {
      posts.forEach { post in
        print(post)
      }
    } else {
      print("ERROR")
    }
  }
```

### Auth Adapter
```
class AuthAdapter: RequestAdapter {
  var getToken : () -> String?
      
  init(getToken: @escaping () -> String?) {
    self.getToken = getToken
  }
  
  func adapt(_ urlRequest: URLRequest, for session: Session, completion: @escaping (Result<URLRequest, Error>) -> Void) {
    var request = urlRequest
              
    if let token = self.getToken() {
      request.setValue(token, forHTTPHeaderField: "token")
    }
      
    completion(.success(request))
  }
}
```

### Auth Retrier
```
class AuthRetryAdapter : RequestRetrier {
  
  var getRefreshToken : () -> String?
  var onUpdateToken : (Token) -> Void
  
  
  init(getRefreshToken: @escaping () -> String?, onUpdateToken: @escaping (Token) -> Void) {
    self.getRefreshToken = getRefreshToken
    self.onUpdateToken = onUpdateToken
  }
  
  func retry(_ request: Request, for session: Session, dueTo error: Error, completion: @escaping (RetryResult) -> Void) {
    if request.response?.statusCode == 401 {
      self.requestNewToken(session) { token in
        self.onUpdateToken(token)
        completion(.retry)
      } fail: {
        completion(.doNotRetry)
      }
    } else {
      completion(.doNotRetry)
    }
  }
  
  func requestNewToken(_ session: Session, then: @escaping (Token) -> Void, fail: @escaping () -> Void) {
    guard let refreshToken = self.getRefreshToken() else {
      fail()
      return
    }
    session.request(
      NetworkController.baseURL + "/refresh-token",
      headers: ["refresh-token": refreshToken]
    ).responseDecodable(of: Token.self) { response in
      if let token = response.value {
        then(token)
      } else {
        fail()
      }
    }
  }
}
```
