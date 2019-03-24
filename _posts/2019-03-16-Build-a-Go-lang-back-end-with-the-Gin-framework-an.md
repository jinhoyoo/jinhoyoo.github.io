---
layout: post
title: "Build a Go-lang back-end with the Gin framework and authenticate with JWT and React front-end"
date: 2019-03-16
excerpt: "Today, I finished the prototyping very small web service in Go."
tags: [Go, Gin, React, JavaScript, ]
comments: false
---


# Build a Go-lang back-end with the Gin framework and authenticate with JWT  and React front-end.

Today, I finished  [the prototyping very small web service](https://github.com/jinhoyoo/golang-gin) in Go.  It's the practice the Go lang and React. It was the journey to the unexplored platform. The bible said, "There is nothing new under the sun". So I started from this [article](https://hakaselogs.me/2018-04-20/building-a-web-app-with-go-gin-and-react/). 

# Back-end side

 This example uses [Gin](https://github.com/gin-gonic/gin) framework in Go.  It's very [simple and fast web framework](https://gin-gonic.com/). See [this document](https://gin-gonic.com/docs/benchmarks/). 

![image](/assets/img/-6fd0b5de-f967-4826-aa75-b9be7b66d7f2untitled)

Let's start with drinking cool water, not gin. 

The code following shows 3 URIs and how you route the request to the handler function. It seems like [flask](http://flask.pocoo.org/) in python for me.  

```go
func main() {

	// Set the router as the default one shipped with Gin
	router := gin.Default()

	// Serve the frontend
	router.Use(static.Serve("/", static.LocalFile("./views", true)))

	api := router.Group("/api")
	{
		api.GET("/", func(c *gin.Context) {
			c.JSON(http.StatusOK, gin.H{
				"message": "pong",
			})
		})
		api.GET("/jokes", authMiddleware(), JokeHandler) // GET /api/jokes
		api.POST("/jokes/like/:jokeID", authMiddleware(), LikeJoke) // POST /api/jokes/like
		api.POST("/auth", auth) // POST /api/auth
	}
	// Start the app
	router.Run(":3000")
}

// JokeHandler returns a list of jokes available (in memory)
func JokeHandler(c *gin.Context) {
	c.Header("Content-Type", "application/json")
	c.JSON(http.StatusOK, jokes)
}
.......
```

 

 Then something I got expression is 'using middleware.' For example, "/api/jokes" handler  has authMiddleware() function. It means, "I wanna call the handler function if the authMiddleware() succeed".  So you can implement authMiddleware() function like following. It's very interesting. Such a [middleware architecture](https://gin-gonic.com/docs/examples/custom-middleware/) allow making the flexible request handler effectively e.g.logging the event. 

```go
func authMiddleware() gin.HandlerFunc {
	return func(c *gin.Context) {

		// Get the JWT token sting in Authorization header.
		var header = c.Request.Header.Get("Authorization")
		fmt.Println("auth : " + header)
		var tokenString = header[7:]

		// Parse and validate JWT token.
		token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
			// Don't forget to validate the alg is what you expect:
			if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("Unexpected signing method: %v", token.Header["alg"])
			}

			// hmacSampleSecret is a []byte containing your secret, e.g. []byte("my_secret_key")
			return []byte(authPassword), nil
		})

		if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
			fmt.Println(claims["user"], claims["timestamp"])
		} else {
			fmt.Println(err)
			c.Abort()
			c.Writer.WriteHeader(http.StatusUnauthorized)
			c.Writer.Write([]byte("Unauthorized"))
			return
		}
	}
}
```

In my project, I implemented the authentication functionality in this middle ware function. 

# Front-end side

Front-end side is comparatively simple.  App component rendered the <Home> component if no access token. And it renders <LoggedIn> component if there is the access token. 


```react
class App extends React.Component {

  setup() {
    $.ajaxSetup({
      beforeSend: (r) => {
        if (localStorage.getItem("access_token")) {
          r.setRequestHeader(
            "Authorization",
            "Bearer " + localStorage.getItem("access_token")
          );
        }
      }
    });
  }

  setState() {
    let idToken = localStorage.getItem("access_token");
    if (idToken) {
      this.loggedIn = true;
    } else {
      this.loggedIn = false;
    }
  }

  componentWillMount() {
    this.setup();
    this.setState();
  }

  render() {
    if (this.loggedIn) {
      return <LoggedIn />;
    }
    return <Home />;
  }
}
```


​    

In <Home> component, it calls `/api/auth` to get access token with ID and password and stores it in local storage. It renders the form for ID and password.

```react
class Home extends React.Component {
  constructor(props) {
    super(props);
    this.authenticate = this.authenticate.bind(this);
  }
  authenticate() {
    this.serverRequest();
  }

  serverRequest() {
    var id = document.getElementById("id").value;
    var password = document.getElementById("password").value;

    fetch('http://localhost:3000/api/auth/', {
      method: 'POST',
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        id: id,
        password: password,
      })
    })
    .then( (response) => {
      if (response.status != 200) {
        console.log('Looks like there was a problem. Status Code: ' +
          response.status);
        alert(response.status)
        return null;
      }
      return response.json() })
    .then( (responseJson) => {
      if (responseJson != null) {
        localStorage.setItem("access_token", responseJson.token);
      }
      location.reload();
      return;
    } )
  }
  
  render() {
    return (
      <div className="container">
        <div className="row">
          <div className="col-xs-4 col-xs-offset-4 jumbotron text-center">
            <h1>Mbears</h1>
            <br /><br /><br />
            <p>Sign in to get access </p>
            <br />
            <div className="form-group has-feedback">
              <input type="text" name="id" id="id" size="36" placeholder="ID"/>
              <span className="glyphicon glyphicon-envelope form-control-feedback"></span>
            </div>
            <div className="form-group has-feedback">
              <input type="password" name="password" id="password" size="36" placeholder="Password"/>
              <span className="glyphicon glyphicon-lock form-control-feedback"></span>
            </div>
            <a
              onClick={this.authenticate}
              className="btn btn-primary btn-lg btn-login btn-block"
            >
              Sign In
            </a>
          </div>
        </div>
      </div>
    );
  }
}
```

In <LoggedIn> component, it calls `/api/jokes` with the access token to get the joke lists. and renders them. 

```react
class LoggedIn extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      jokes: []
    };

    this.serverRequest = this.serverRequest.bind(this);
    this.logout = this.logout.bind(this);
  }

  logout() {
    localStorage.removeItem("access_token");
    location.reload();
  }

  serverRequest() {
    $.get("http://localhost:3000/api/jokes", res => {
      this.setState({
        jokes: res
      });
    });
  }

  componentDidMount() {
    this.serverRequest();
  }

  render() {
    return (
      <div className="container">
        <br />
        <span className="pull-right">
          <a onClick={this.logout}>Log out</a>
        </span>
        <h2>Jokeish</h2>
        <p>Let's feed you with some funny Jokes!!!</p>
        <div className="row">
          <div className="container">
            {this.state.jokes.map(function(joke, i) {
              return <Joke key={i} joke={joke} />;
            })}
          </div>
        </div>
      </div>
    );
  }
}
```

# Conclusion

 Go with Gin seems to work well. React is still hard to follow the code follow. Gin has a very flexible structure with middleware. It's good to extend the action with the handler.  [This repository](https://github.com/jinhoyoo/golang-gin)  has all code I revised.

![](/assets/img/-c7d2c490-4261-4911-b281-65db1e1fd74cuntitled)

OK, let's have break. 

# Tip

 I didn't know jsx works without compiling by embedding babel.js.  In this [HTML code](https://github.com/jinhoyoo/golang-gin/blob/master/views/index.html) shows it but NEVER USE IT IN THE PRODUCTION. But seems useful for debugging.