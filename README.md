## Project Summary

tech: Docker, Kubernetes, react, Nodejs, micro-service architecture,

key points:

1. micro-services challenge is handling data
   ie. understanding passing events and payload to event-bus and letting query service handle incoming events
2. Async communication using events sent to an event bus
3. Async communication means temp service downtime wouldn't break the app, ie. query service on launch will scan through all events saved in event bus.
4. Async communication means each micro service can be self-sufficient
5. Docker packages services for K8s to deploy and scales

Pain Points & possible solutions:

1. Duplicated code, ie. express n packages
   a. central lib as NPM module
2. hard to picture flow of events
   a. define all events in central lib
3. hard to remember which properties an event should have
   a. Typescript 😱😱
4. hard to test some event flows
   a. writing tests
5. concurrency issues. ie. milliseconds after someone makes a post, they comment, order of things might mess up.
   a. code to prevent concurrency

How it works:

1.  client => create posts => API call to posts service, posts => "event-created: Post Created" => event-bus service => keep a copy of said event (event-bus) =>

Routes:
posts (PORT: 4000):
listening: GET: "/posts" | POST: "/posts/create" | POST: "/events"

description: get all posts (dev only) | creating one post | event-bus ReceivedEvent response

API calls: POST: "http://event-bus-srv:4005/events"
description: K8s clusterIP adding PostCreated event

---

comments (PORT:4001):
listening: GET: "/posts/:id/comments" | POST: "/posts/:id/comments" | POST: "/events"

description: finds the post via :id, and return the comments arr | creates the comment with status of pending, assign a commentId, push content into an arr, CommentCreated event sent to event-bus

"POST: /events"
Every event is passed back, but we are listening for the CommentModerated event, which only the moderation service can send to event-bus, we are taking that event and updating our comment status with it and sending the event to event bus again updating event-bus and indirectly the query service with CommentUpdated event.

API calls: POST: "http://event-bus-srv:4005/events"
description: K8s clusterIP adding CommentCreated event, and CommentUpdated after moderation service CommentModerated event trigger

---

moderation (PORT:4003):
listening: POST: "/events"

description: listens to every CommentCreated event passed from event-bus, has an filter rule and either approve or reject the comment content. Then makes a call to pass the CommentModerated that the comment service listens to. Possible change is to have the moderation service change the status of the comment obj once its moderated.

---

query service (PORT:4002):
pseudo database service, listens to PostCreated, CommentCreated, CommentUpdated coming for the event bus and updates our posts object. The real "/posts" route that gives the most up to date data, such as moderated comments.

---

event-bus service (PORT: 4005):

listening: GET: "/events" | POST: "/events"
description: sends out all events in the events arr | takes the incoming event, push it into events arr, and calls every service to pass off the event for appropriate services to handle it.

# Updating Docker Container and Images

docker build -t <name> .
docker push <name>

# K8s apply and restart

kubectl apply -f <yaml>
kubectl rollout restart deployments <names>

# Cluster IP within index.js (posts, event-bus ...)

instead of localhost:4005, we are using exact name of k8s services names and its open port, ie. event-bus-srv:4005

# Publishing Services

1. ClusterIP: Exposes the Service on a cluster-internal IP. Choosing this value makes the Service only reachable from within the cluster. This is the default ServiceType.

2. NodePort(DEV): Exposes the Service on each Node's IP at a static port (the NodePort). A ClusterIP Service, to which the NodePort Service routes, is automatically created. You'll be able to contact the NodePort Service, from outside the cluster, by requesting <NodeIP>:<NodePort>.

3. LoadBalancer: Exposes the Service externally using a cloud provider's load balancer. NodePort and ClusterIP Services, to which the external load balancer routes, are automatically created.

4. ExternalName: Maps the Service to the contents of the externalName field (e.g. foo.bar.example.com), by returning a CNAME record with its value. No proxying of any kind is set up

# React-dev-server Pod

React dev pod is only for giving the clients html, JS and CSS, other than that. It does not make any network requests to our services.

The Client makes all requests from front-end.

# Load balancing and ingress controllers

In order for the Ingress resource to work, the cluster must have an ingress controller running. Supports and maintains AWS, GCE.

We will be using any load balancing services provided by Cloud Providers, it communicates with Ingress Controller, ingress controller routes traffic to appropriate Pods.

# Host file manipulation

adding the line 127.0.0.1 posts.com into C:\Windows\System32\drivers\etc

Trying to trick browser into accessing posts-clusterip-srv via posts.com

# Learning: Manuel Container Code change

Every time we make a change to our containerized code, we have to build and push the image again. Which is very tedious, there is a better way.

# Success!

I was having a lot of issues by the end of the blog tutorial, getting 502 errors at first, not knowing why my pods were erroring, I think eventually I found it to be the nodeports causing posts/event-bus/query pods to error out, which at first I didn't know why it was causing it. But the I had to trace my steps. Eventually after rebuilding my docker containers and pushing the images. I've finally gotten all my pods to function, communicating with each other via clusterip, getting my client pod to render the react app in client. After all my

Tricking local machine with editing HOST file adding posts.com as the equivalent of localhost as an ingress-nginx site.

# debugging

debugging has being really hard with constantly having to check if my pod is erroring out with "kubectl logs <name>", or restarting with "kubectl rollout restart deployment <name>", I did use "k describe <kind> <name>" but it wasn't too useful. If only there are ways to enter a pod and logging constantly. Need further reading.

# Skaffold

https://skaffold.dev/docs/references/yaml/

DEV software that handles the workflow for building, pushing and deploying your application in K8s

start: skaffold dev
cleanup: skaffold delete

# Meaning of Query Service

NOUN.
1: QUESTION, INQUIRY
2: a question in the mind : DOUBT
3: QUESTION MARK sense 2
VERB.
1: to ask questions of especially with a desire for authoritative information
2: to ask questions about especially in order to resolve a doubt
3: to put as a question
4: to mark with a query

# Why do we need a query service ?

When we want to get every posts and comments created, typically we can get it from the posts and comments services, but doing that means we are making two api calls, having the query service we have the data we need in just one go.
