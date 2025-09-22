# PSET 2

## Concept Questions

### 1. Contexts

The contexts ensure that the nonce generated is only unique for the provided context. This allows the same nonce to exist for 2 different contexts, allowing more possible combinations of contexts and nonces. In the URL shortening app, a context will end up being the URL base.

### 2. Storing used strings

NonceGeneration must store sets of used strings because when generating a new one, the concept can quickly look up to ensure the new nonce is not already used for this context.

Let's say our counter = `x`. Then `used = {str(0), str(1), str(2), ..., str(x-1)}`. When we generate a new nonce, we return `str(x)` and increment the counter to `x+1`. This way, we ensure that the nonce generated is unique for this context.

### 3. Words as nonces

One advantage of the scheme to use common dictionary words for nonce generation is that they’re easier to remember. If we’re thinking about real-life applications, I can simply tell you the base URL and the easy to remember dictionary word, and you will likely remember it for a while. When you’re ready to go to my targetURL, it’ll be easy to remember the short URL because it uses a common word.

One disadvantage with this scheme is that there is reduced security which can allow attackers to guess nonces and reach target URLs. Let’s say our target URL is a Google Slides link that a teacher is using to have her students collaborate on a class presentation. She uses a URL shortener to easily give students access to the Google Slides, with edit access. If the shortened URL’s nonce is an easy to guess word, any attacker can brute force their way to the Google Slides and ruin the teacher’s and the student’s work on the projects. Now we can only imagine the damage possible if we used a more security-nuanced example.

```
concept NonceGeneration [Context]
purpose generate common words within a context
principle each generate returns a common word not returned before for that context

state
  a set of Contexts with
    a used set of Strings
    a dictionary set of Strings

actions
  generate (context: Context) : (nonce: String)
    effect returns a nonce from the dictionary that is not already used by this context
```

## Synchronization Questions

### 1. Partial matching

To generate a nonce, you only need to provide it a `shortUrlBase` for its context, as that’s all it needs to create a unique nonce.

To register a URL shortening mapping to a targetURL, you need both the `targetUrl` and `shortUrlBase` to do this mapping.

### 2. Omitting names

The reason this strategy of omitting names isn’t used in every case is ambiguity. Two arguments with the same name could exist from different actions. This can get confusing, where we’re not sure which version of this argument we’re referring to. It’s best to omit names when there is no risk of naming collisions.

### 3. Inclusion of request

There’s a request action in the first two syncs but not the third one because the first two syncs are triggered by an external request from the user whereas the third sync is triggered internally by the system. In the first two syncs, the service only generates nonces and registers short URLs when a user explicitly asks to shorten a URL. Once a short URL has been created, the system can automatically set an expiry time for it. This action is a consequence of registration.

### 4. Fixed domain

```
sync generate
  when Request.shortenUrl (targetUrl)
  then NonceGeneration.generate (context: “bit.ly”)

sync register
  when
    Request.shortenUrl (targetUrl)
    NonceGeneration.generate (): (nonce)
  then UrlShortening.register (shortUrlSuffix: nonce, shortUrlBase: “bit.ly”, targetUrl)

sync setExpiry
  when UrlShortening.register (): (shortUrl)
  then ExpiringResource.setExpiry (resource: shortUrl, seconds: 3600)

```

### 5. Adding a sync

```
sync expire
  When ExpiringResource.expireResource (): (resource: shortUrl)
  Then UrlShortening.delete(shortUrl)
```

## Extending the design

### 1.

```
Concept Authentication [User]
Purpose allows user to sign in and authenticate themselves.
Principle after a user registers themselves, they are able to sign in and authenticate themselves wherever needed

state
  a set of Users with
    a username String
    a password String

  a set of sessions with
    a user User
    a secret String

actions
  register (username: String, password: String) : (user: User)
    requires no User exists with this username
    effects returns the user created

  signin (username: String, password: String) : (session: Session)
    requires username exists and password matches the password stored for username
    effects the session for this User is returned

  authenticate (user: User, secret: String) : (user: User)
    requires the secret passed in matches the secret stored in the session for the user
    effects returns the user

Concept Analytics [User, Resource]
Purpose allows the number of shortUrl accesses be viewable by person who created the shortUrl
Principle after confirming the user created this shortUrl, let them view how many people accessed the shortUrl

state
  a set of Resources with
    an owner User
    a viewCount Number

actions

  initAnalytics (owner: User, resource: Resource)
    requires resource is not already owned by owner or being viewed
    effects set resource’s viewCount to 0 and set its owner to be owner.

  incrementCount(resource: Resource)
    requires resource exists
    effects increment the viewCount of the resource by 1

  viewAnalytics (requester: User, resource: Resource) : (count: Number)
    requires the requester is the owner of the source
    effects returns the viewCount for this resource

```

### 2.

```
sync initAnalytics
when
  Request.shortenUrl (targetUrl, secret)
  Authentication.authenticate (user, secret) : (user: User)
  UrlShortening.register() : (shortUrl)
then Analytics.initAnalytics(user: owner, shortUrl: resource)

sync recordView
when UrlShortening.lookup(shortUrl)
then Analytics.incrementCount(shortUrl: resource)

sync viewAnalytics
when Authentication.authenticate(user, secret) : (user)
then Analytics.viewAnalytics(user: requester, shortUrl: resource)

```

### 3.

**Allowing users to choose their own short URLs**

This can be done by allowing an optional parameter called `customNonce` to `Request.shortenUrl`. In the register sync, `UrlShortening.register` will ensure this `customNonce` is valid and can be used.

**Using the “word as nonce” strategy to generate more memorable short URLs**

I would not recommend adding this feature. While it does sound nice in theory in order to help users remember urls better, it can have more negative effects than good. I would rather prioritize the safety of user’s urls and content in the urls over an easier method of remembering them.

**Including the target URL in analytics, so that lookups of different short URLs can be grouped together when they refer to the same target URL**

We can extend the `Analytics` concept and store the `targetUrl` in each resource. Then the `initAnalytics` sync is able to pass the `targetUrl` and we can do our desired search.

**Generate short URLs that are not easily guessed**

We can do all our updates in `NonceGeneration` by using advanced algorithms that generate strings which are complex and unique that they can’t be randomly guessed.

**Supporting reporting of analytics to creators of short URLs who have not registered as user**

I feel as if this is something we shouldn’t do. The modularity concept will all break if we implement this as it would require us to modify a lot of the code. We designed our code under the assumption that we would have users logged onto our site. This non-signed in practice would pose a lot of security risks and abandon our modular system.
