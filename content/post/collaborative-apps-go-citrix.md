+++
date = "2015-03-17T08:59:18-04:00"
title = "Collaborative Apps with Go at Citrix"
tags = ["Firebase", "go"]
author = "Tim Gossett"
+++

__Reposted from [the Firebase blog](https://www.firebase.com/blog/2015-03-17-collaborative-go-apps-at-citrix.html)__

## Collaborative Apps with Go at Citrix

_[Tim Gossett](https://github.com/MrGossett) and his colleague [Steven Berlanga](https://github.com/zabawaba99) are Backend Developers for the Realtime Platform Group at Citrix. They build services to enable collaborative web and mobile apps._

Our team is developing realtime communications technology, and we’re using Firebase for low-latency data sync. Our platform provides a rally point for mobile and web clients to connect and start collaborating. Part of that collaboration happens in Firebase.

Our platform takes on the responsibility of vending tokens that secure access to Firebase, thereby securing the communication between the clients. To implement this, we knew our platform would need to consume the [Firebase REST Streaming API](https://www.firebase.com/docs/rest/api/#section-streaming) and vend JWTs (JSON Web Tokens).

We write in [Go](https://golang.org/), so we looked for a Go package to handle our interaction with Firebase. As soon as we were tasked to look into integrating Firebase, we quickly checked the Firebase documentation to see if there were any open source Go REST API implementations. Because there were no existing open source projects that utilized Firebase’s REST streaming protocol,  we decided to write our own. We ended up with three libraries.

### Firego

[Firego](https://github.com/CloudCom/firego) wraps the Firebase REST and Streaming APIs. Our goal was to have a comparable interface as Firebase’s iOS and Android SDKs, while still following Go conventions. Example code explains it best:

```go
// create a reference to data in Firebase and authorize it with a JWT token
f := firego.New("https://my-firebase-app.firebaseio.com/path/to/some/data")
f.Auth("some-token")

// fetch the data and hydrate v with the result
var v map[string]interface{}
if err := f.Value(&v); err != nil {
  log.Fatal(err)
}

// update Firebase with new data
newData := map[string]string{"foo":"bar"}
if err := f.Update(newData); err != nil {
  log.Fatal(err)
}

// watch for updates
notifications := make(chan firego.Event)
f.Watch(notifications)

e := <-notifications
fmt.Println("type=%s path=%s data=%v", e.Type, e.Path, e.Data)
```

### Fireauth

To generate secure JWTs for Firebase’s [Custom Login](https://www.firebase.com/docs/web/guide/login/custom.html) we built [Fireauth](https://github.com/CloudCom/fireauth). Using this library, you can pack arbitrary data into a token and use that data to tailor access to Firebase using [security rules](https://www.firebase.com/docs/security/guide/securing-data.html).

```go
data := fireauth.Data{
    "uid":  "1",
    "some": "arbitrary",
    "data": "here",
}

tokenGenerator = fireauth.New("<YOUR_FIREBASE_SECRET>")
token, err := tokenGenerator.CreateToken(data);
if err != nil {
    log.Fata(err)
}
println(“token:”, token)
```

### Fireload

[Fireload](https://github.com/CloudCom/fireload) ties together a pool of Firebase namespaces and acts as a load balancer among them. Since we’re building in the Cloud, we aim to be horizontally scalable so that we can ramp up to handle “web-scale” traffic in very little time. While Firebase can handle rather heavy load, we want to avoid any single point of failure.

```go
// create namespaces
instance1 := fireload.NewNamespace("https://foo.firebaseIO.com")
instance2 := fireload.NewNamespace("https://bar.firebaseIO.com")

// set metadata on namespaces
instance1.Metadata.Set("awesome", true)
instance2.Metadata.Set("secret", "my-awesome-secret")

// create pool
pool, err := fireload.NewPool(instance1, instance2)
if err != nil {
    log.Fatal(err)
}

// choose the strategy you want to use
pool.SetStrategy(fireload.StrategyRoundRobin)

// get a firebase namespace
namespace := pool.Next()

// use your shiny new namespace
f := firego.New(namespace.Domain)
```

### We’d like your feedback!

The open-source repos are at [github.com/CloudCom](https://github.com/CloudCom). We would welcome feature requests, bug reports, pull requests and other feedback from the Firebase and Golang communities.