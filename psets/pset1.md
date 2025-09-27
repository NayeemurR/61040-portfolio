# PSET 1

## Exercise 1: Reading a concept

### 1. Invariants

The first invariant is that the request’s count variable has to be greater than or equal to 0. You can never have a negative count for a requested item. The second invariant is that the sum of all purchase counts for a specific item must be less than or equal to that item’s original request count. You can’t have more purchases than the requested count for an item.

The second invariant is more important because it ensures correctness for the system. Friends can’t overpurchase items, which is the whole purpose of GiftRegistration.

The most affected action is Purchase. It requires that the request exists and has a positive count for this item. It then creates a purchase record and decrements the count variable. This preserves the invariant because if the count of a request is 0, then no one else needs to purchase this item anymore, ensuring that you can’t have more purchases than the original requested count for an item.

### 2. Fixing an action

`removeItem` can potentially break this invariant. If an item already has purchases and you allow `removeItem` to delete its request, the requested total for that item effectively drops to 0 while purchases > 0 still exist. This problem can be fixed if you add a check to ensure that a request for this item exists in the registry and it has no purchases.

### 3. Inferring behavior

Yes, a registry can be opened and closed repeatedly using the `open` and `close` actions. The reason for this depends on the owner. They may want to pause public access to the registry in order to edit items or counts, fix mistakes, or handle any type of necessary change without allowing a purchase during this period.

### 4. Registry deletion

This would matter in practice. For example, owners may just be testing out the actions by creating a dummy registry, and they would want the option to delete it so it’s not cluttering up their view. Another example could be from the perspective of the company of this GiftRegistration concept. They may not want to waste computer memory on unused registries. The option to delete a registry would greatly help in this regard.

### 5. Queries

A common query executed by a registry owner could be to know who bought which items. A common query executed by a gift giver could be to know what items are remaining to buy. These match the two primary actors’ needs: the owner audits gifts and thanks givers; the giver sees only what’s still needed on an open registry.

### 6. Hiding purchases

To allow for a feature which lets the recipient to choose not to see purchases, I would add a flag state called hidden. Then 2 actions, hidePurchases and showPurchases, would toggle this flag. When hidden is true, the owner would not be able to see any purchases made on their registry. When hidden is false, the owner would be able to see all purchases made on their registry.

```
state
  a set of Registrys with
  an owner User
  an active Flag
  a set of Requests
  a hidden Flag

Actions
  hidePurchases(registry: Registry)
    requires registry exists and registry's hidden is False
    effects hides purchases in the registry

  showPurchases(registry: Registry)
	requires registry exists and registry's hidden is True
	effects reveals purchases in the registry
```

### 7. Generic types

It’s preferable to represent items by a SKU code because it acts as a stable key. Item names, descriptions, and prices can all change, but the SKU remains the same and still represents the original item. Furthermore, if we have a product that multiple brands sell, a SKU can easily help a gifter discover the exact version of the product the owner wants. Furthermore, SKUs are universally understood. A product name may be altered or punctuation can be changed, which can lead to issues.

## Exercise 2: Extending a familiar concept

### 1. Complete the definition of the concept state AND 2. Write a requires/effects specification for each of the two actions

```
concept PasswordAuthentication
purpose limit access to known users
principle after a user registers with a username and a password,
    they can authenticate with that same username and password
    and be treated each time as the same user

state
  a set of Users with
    a username String
    a password String


actions
  register (username: String, password: String): (user: User)
    requires a user with this username does not exist
	effects a new user is created with this username and password

  authenticate (username: String, password: String): (user: User)
    requires a user with this username exists and the password matches
    effects returns this user
```

### 3. What essential invariant must hold on the state? How is it preserved?

The essential invariant is that usernames are unique. At most one User is given any username. In the initial state, with no users, the invariant holds trivially. When the `register` action is called, a check is done to ensure a user with this username doesn’t exist, thus holding the uniqueness. When `authenticate` is called, no state change is made, therefore still holding the invariant.

### 4. One widely used extension of this concept requires that registration be confirmed by email. Extend the concept to include this functionality.

```
state
  a set of Users with
    a username String
    a password String
    a confirmed Flag

  a set of Confirmations with
	a username String
	a token Secret

actions
  register (username: String, password: String): (user: User, token: Secret)
    requires a user with this username does not exist
	effects a new user is created with this username and password and confirmed is set to False, a new confirmation is created with this username and token, and the token is emailed to the user.

  confirm (username: User, token: Secret)
    requires user with username exists, user’s confirmed flag is False, and the token matches the token in the set of confirmations for this user.
    effects user’s confirmed flag is set to True.

authenticate (username: String, password: String): (user: User)
  requires a user with this username exists, the password matches, and the user’s confirmed flag is True.
  effects returns this user
```

## Exercise 3: Comparing concepts

```
concept PersonalAccessToken [User, Scope]
purpose log into software using revocable, scoped secrets
principle a user can create a single or multiple tokens to log into a software;
when presented, each token may have access scopes and an optional expiration date;
tokens can be revoked so they no longer work.

state
  a set of Users with
    a username String
	a password String
	a set of Tokens

  a set of Tokens with
    an owner User
	a scopes set of Scope
	an active Flag
	an expiration optionalDate

actions
  create (owner: User, scopes: Scope, expiration: optionalDate) : (token: Token, secret: Secret)
    requires that the owner exists in Users
    effects creates a new token for owner, gives it scopes, sets the active flag to True, and assigns it an expiration if provided.

  revokeToken (owner: User, token: Token)
	requires that owner and token exists and that the token belongs to the owner
	effects token’s active Flag is set to False

  authenticate (owner: User, token: Token) : (user: User)
	requires that owner and token both exist and that the token matches one of the owner’s tokens
	effects returns the owner

```

A personal access token differs from a password in a few ways. For starters, users can have many tokens versus the typical one password. Tokens also carry scopes which give access and restrictions to certain access points. Passwords don’t have this feature. Tokens can also be revoked or set to expire without affecting the account password.

To improve the GitHub documentation to explain these differences better, I would add a simple table called “Passwords vs Personal Access Tokens (classic).” This table would outline how many per user are allowed, scopes, expiration, revocation, and typical usage. This makes the statement “treat tokens like passwords” clear while showing why they’re not the same thing.

## Exercise 4: Defining familiar Concepts

### 1. URL Shortener

```
concept URLShortener [User]
purpose map long URLs to short suffixes for easy sharing
principle user submits a url;
  user either submits a suffix or receives an autogenerated one;
  while the original url is active, anyone can visit the shortened url and it’ll resolve to the longer, original one.

state
  a set of Links with
	an owner User
	a suffix String
	a target URL

actions
  shortenAuto (owner: User, targetURL: URL) : (link: Link)
	requires targetURL to be a valid and working URL.
	effects creates and returns a new link with an unused suffix given the targetURL for the owner

  shortenCustom (owner: User, targetURL: URL, suffix: String) : (link: Link)
    requires targetURL to be a valid and working URL AND suffix is not being used by another owner.
	effects creates and returns a new link with this suffix given the targetURL for the owner

  resolve (suffix: String) : (targetURL: URL)
	requires a Link exists with this suffix
	effects user is redirected to the link’s target URL.
```

### 2. Billable Hours Tracking

```
concept BillableHoursTracking [User, Project]
purpose Track hours worked for a project per employee
principle employee starts a session by selecting a project to work on;
  employee enters a description of the work they’ll do;
  employee ends the session when they are done working.

state
  a set of Users with
	a name String

  a set of Projects with
	a name String

  a set of Sessions with
	an employee User
	a project Project
	a description String
	a startTime Time
	a endTime optionalTime

actions
  startSession (employee: User, project: Project, description: String, startTime: Time) : (session: Session)
	requires employee and project exist
    effects if there is already an existing session for this employee, the existing session’s endTime is set to startTime. Regardless if that clause occurred, now a new session is then created for the project with the description and startime

  endSession (employee: User, project: Project, endTime: Time)
	requires employee to exist with an active session denoted by a blank endTime.
	effects the employee’s active session’s endTime is set to endTime.
```

## 3 Conference Room Booking

```
concept RoomBooking [User, Room]
purpose efficient booking of rooms without overlap
principle a user books a room for a certain time interval if that room is not already booked at any period during that time interval;
  bookings can be canceled.

state
  a set of Users with
    a name String

  a set of Rooms with
	a name String

  a set of Reservations with
	an organizer User
	a startTime Time
	an endTime Time
	a room Room

actions
  reserveRoom (organizer: User, room: Room, startTime: Time, endTime: Time) : (reservation: Reservations)
    requires the user and room exists. startTime is before endTime. The room isn’t already reserved for any period in between startTime and endTime.
    effects a new reservation is created for this room for the organizer during [startTime, endTime].

  cancelRoom (user: User, reservation: Reservations)
    requires reservation to exist and the reservation’s organizer is the valid user who created the reservation.
	effects removes the reserved timeslot from the reservation and frees up that space.
```
