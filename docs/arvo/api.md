---
navhome: /docs
next: true
sort: 12
title: API Connectors
---

# API Connectors

Most people have lots of data stored in online services, many of
which have APIs.  API connectors allow the user to access this
data from within their Urbit.

A security driver allows the user to make authenticated requests
to the service's API, but the user still needs to know the API
and send and receive JSON.  An API connector puts a layer of
porcelain over this to allow the user easier control over their
data.

Each connector should perform two basic functions.  First, it
should expose a tree of the data in the service that's accessible
through `.^` and FUSE.  Second, it should expose event streams as
actions occur in the service.  Let's take a look at how the `%gh`
app accomplishes both for Github.

After starting `%gh` (`|start %gh`), let's look at the root of
the tree that `%gh` exposes:

    ~your-urbit:dojo> .^(arch %gy /=gh=)
    [fil=~ dir={[p=~.issues q=~]}]

`%gh` is currently a skeleton -- it contains examples of all the
functionality necessary for a connector, but many endpoints
aren't implemented.  As we can see here, only issues are
implemented.  We can explore this tree:

    ~your-urbit:dojo> .^(arch %gy /=gh=/issues)
    [fil=~ dir={[p=~.by-repo q=~] [p=~.mine q=~]}]

And eventually:

    /-  gh
    .^(issue:gh %gx /=gh=/issues/by-repo/philipcmonktest/testing/7)
    <issue...>
    login.user:.^(issue:gh %gx /=gh=/issues/by-repo/philipcmonktest/testing/7)
    'philipcmonktest'

The other thing an API connector needs to do is expose event
streams.  One straightforward way to use this capacity is with
the `:pipe` app:

    :pipe|connect %gh /listen/philipcmonktest/testing/issues/'issue_comment' %public

Now creating an issue results in the following message in your
`%public` `:talk` channel:

    ~your-urbit[philipcmonktest@github]: opened issue #11: i found a bug!

You'll also get notifications of issue status changes and
comments.

As is often the case, the best way to write an API connector is
to copy an existing one and modify it.  We'll dissect the Github
connector here.  This code is in `/=home=/app/gh/hoon`, and it's
also reproduced at the bottom of this page (in case the code
drifts out of sync with this doc).

We'll go over the two parts to an API connector (one-time reading vs listening
for events) separately.

## Reading

A connector exposes a tree of data.  Every read request has a
`care`, which is generally either `%x` or `%y.  `%x` is a request
for a particular piece of data, while `%y` is a request for a
directory listing.  Roughly, `%x` means Unix `cat` and `%y` means
Unix `ls`.

> Sometimes you wish to expose a tree where a part of the path
> can't be enumerated.  For example, a `%y` of `/issues/by-repo`
> "should" produce a list of all Github users, but we don't do
> that because it's too long.  Instead, we just produce our own
> username (from the `web.plan` file).  You can still access
> repos from other users, you just don't see them in the
> directory listing.

The usual flow for implementing this tree of data makes heavy use
of the `connector` library.  This library is well documented in
the source, so check out `/=home=/lib/connector/hoon`.

The `connector` library needs to be initialized with definitions
of `move` and `sub-result` (all the types of data that can be
returned by a place).  This can just be a line at the top of the
main core:

    =+  connector=(connector move sub-result)  ::  Set up connector library

Most of the Github-specific logic is in `++places`, which is a
list of all the places we can request.  A place consists of:

    ++  place
      $:  guard/mold
          read-x/$-(path move)
          read-y/$-(path move)
          sigh-x/$-(jon/json (unit sub-result))
          sigh-y/$-(jon/json (unit arch))
      ==

-  `guard`, the type of the paths we should match.  For example,
   to match `/issues/<user>/<repo>` use `{$issues @t @t $~}`.

-  `read-x`, called when someone tries to read the place with
   `care` `%x`.  Should produce a single move, usually either a
   `%diff` response if we can immediately answer or a `%hiss`
   http request if we need to make a request to the api.  See the
   `++read-*` functions in `++helpers` for some common handlers.

-  `read-y`, same as `read-x` except with care `%y`.

-  `sigh-x`, called when an http response comes back on this
   place.  You're given the json of the result, and you should
   produce either a result or null.  Null represents an error.
   If you didn't create an http request in `read-x`, then this
   should never be called.  Use `++sigh-strange` from `++helpers`
   to unconditionally signal an error.

-  `sigh-y`, same as `sigh-x` except with care `%y`.  Note that a
   `%y` request must produce an arch, unlike a `%x` request,
   which may produce data of any mark.

Filling out the list of places is a lot of grunt work, but most
places are fairly straightforward, and the `++read-*` helper
functions are useful.  Check out the library source for more
information on those.

Besides the list of places, we just need to handle the flow of
control.  `++peek` and `++peer-scry` are the interface we expose
to the rest of the system while `++sigh-httr` and `++sigh-tang`
are used to handle responses when we make HTTP requests.

- `++peek` should usually just produce `~`.  If there are cases
  where we can respond directly to a `.^` request without
  blocking on anything, we could do it here, but it's generally
  not worth the hassle since the same logic should be duplicated
  in `++peer-scry`.

- `++peer-scry` is where the actual handling for a read request
  goes.  In general, we just need to call `++read` from the
  `connector` library with the bone, list of places, care, and
  path.  This will match the path to the appropriate place and
  run either `++read-x` or `++read-y`, depending on the care.

- If `++read-x` or `++read-y` made a successful API request, then
  the response will come back on `++sigh-httr`.  Here, we just
  need to parse out the wire and call `++sigh` from the
  `connector` library with the list of places, care, path, and
  HTTP result.  This will match the path to the appropriate place
  and run either `++sigh-x` or `++sigh-y`, depending on the care.

- If `++read-x` or `++read-y` made an API request that failed,
  then we'll get a stack trace in `++sigh-tang`.  Here, we just
  print it out and move on.

That's really all there is to the reading portion of API
connectors.

One of the most accessbile ways to jump into Arvo programming is
to just add more places to an existing API connector.  It's
useful, small in scope, and comes in bite-sized chunks since most
places are less than ten lines of code.

## Listening

Listening for events is fairly service-specific.  In some
services, we poll for changes.  The Twitter connector has an
example of this, but note that it predates the `connector`
library and is thus more complicated than it needs to be, and the
interface it exposes isn't standard.  In Github, we power our
event streams with webhooks.

For Github, when someone subscribes to
`/listen/<user>/<repo>/<events...>`, we want to produce
well-typed results when they occur.

We only want to create one webhook per event, so in our state we
have `hook`, which is a map of event names to the set of bones
which are subscribed to that event.  We must always keep this
up-to-date.

Flow of control starts in `++peer-listen`, where we just call
`++listen`.  In `++listen`, for each event in the list of events,
we check to see if we have that hook set up (by checking whether
it exists in `hook`).  If so, we call `++update-hook` to add the
current bone to the set of listeners.  Otherwise, we call
`++create-hook`, which sends a request to Github to set up the
new webhook.  We also create an entry in `hook` with the current
bone.

When we created the webhook, we told Github to send the event to
`/~/to/gh/gh-<event>.json?anon&wire=/`.  This turns into a poke.
Let's parse out this url.  The first `gh` is the app name to
poke.  The next portion, `gh-<event>` is the mark to convert the
JSON to.  For example, when the "issues" event fires, it'll get
converted to mark `gh-issues`.  This mark has a conversion
routine from `json` to the `issues` type, which is defined in the
`gh` structure (`/=home=/sur/gh/hoon`).  Once this conversion
happens, the `gh` app will get poked with the correctly marked
data in `++poke`.

In `++poke`, we check `hook` to find the subscribers to the event
that was fired, and we update them.

That's really all there is to it.  Webhook flow isn't well
standardized, so even if your chosen service is powered by
webhooks, your listening code might look rather different.  The
point is to expose the correct interface.


## Github API Connector Code

In case the code in `/=home=/app/gh/hoon` drifts out of sync with
this doc:

```
::  This is a connector for the Github API v3.
::
::  You can interact with this in a few different ways:
::
::    - .^({type} %gx /=gh={/endpoint}) to read data or
::      .^(arch %gy /=gh={/endpoint}) to explore the possible
::      endpoints.
::
::    - subscribe to /listen/{owner}/{repo}/{events...} for
::      webhook-powered event notifications.  For event list, see
::      https://developer.github.com/webhooks/.
::
::  This is written with the standard structure for api
::  connectors, as described in lib/connector.hoon.
::
/?  314
/-  gh, plan-acct
/+  gh-parse, connector
::
!:
=>  |%
    ++  move  (pair bone card)
    ++  card  
      $%  {$diff sub-result}
          {$them wire (unit hiss)}
          {$hiss wire {$~ $~} $httr {$hiss hiss}}
      ==
    ::
    ::  Types of results we produce to subscribers.
    ::
    ++  sub-result
      $%  {$arch arch}
          {$gh-issue issue:gh}
          {$gh-list-issues (list issue:gh)}
          {$gh-issues issues:gh}
          {$gh-issue-comment issue-comment:gh}
          {$json json}
          {$null $~}
      ==
    ::
    ::  Types of webhooks we expect.
    ::
    ++  hook-response
      $%  {$gh-issues issues:gh}
          {$gh-issue-comment issue-comment:gh}
      ==
    --
=+  connector=(connector move sub-result)  ::  Set up connector library
::
|_  $:  hid/bowl
        hook/(map @t {id/@t listeners/(set bone)})  ::  map events to listeners
    ==
::  ++  prep  _`.  ::  Clear state when code changes
::
::  List of endpoints
::
++  places
  |=  wir/wire
  ^-  (list place:connector)
  =+  (helpers:connector ost.hid wir "https://api.github.com")
  =>  |%                              ::  gh-specific helpers
      ++  sigh-list-issues-x
        |=  jon/json
        %+  bind  ((ar:jo issue:gh-parse) jon)
        |=  issues/(list issue:gh)
        gh-list-issues+issues
      ::
      ++  sigh-list-issues-y
        |=  jon/json
        %+  bind  ((ar:jo issue:gh-parse) jon)
        |=  issues/(list issue:gh)
        :-  `(shax (jam issues))
        (malt (turn issues |=(issue:gh [(rsh 3 2 (scot %ui number)) ~])))
      --
  :~  ^-  place                       ::  /
      :*  guard=$~
          read-x=read-null
          read-y=(read-static %issues ~)
          sigh-x=sigh-strange
          sigh-y=sigh-strange
      ==
      ^-  place                       ::  /issues
      :*  guard={$issues $~}
          read-x=read-null
          read-y=(read-static %mine %by-repo ~)
          sigh-x=sigh-strange
          sigh-y=sigh-strange
      ==
      ^-  place                       ::  /issues/mine
      :*  guard={$issues $mine $~}
          read-x=(read-get /issues)
          read-y=(read-get /issues)
          sigh-x=sigh-list-issues-x
          sigh-y=sigh-list-issues-y
      ==
      ^-  place                       ::  /issues/by-repo
      :*  guard={$issues $by-repo $~}
          read-x=read-null
          ^=  read-y
          |=  pax/path
          =+  /(scot %p our.hid)/home/(scot %da now.hid)/web/plan
          =+  .^({* acc/(map knot plan-acct)} %cx -)
        ::
          ((read-static usr:(~(got by acc) %github) ~) pax)
          sigh-x=sigh-strange
          sigh-y=sigh-strange
      ==
      ^-  place                       ::  /issues/by-repo/<user>
      :*  guard={$issues $by-repo @t $~}
          read-x=read-null
          read-y=|=(pax/path (get /users/[-.+>.pax]/repos))
          sigh-x=sigh-strange
          ^=  sigh-y
          |=  jon/json
          %+  bind  ((ar:jo repository:gh-parse) jon)
          |=  repos/(list repository:gh)
          [~ (malt (turn repos |=(repository:gh [name ~])))]
      ==
      ^-  place                       ::  /issues/by-repo/<user>/<repo>
      :*  guard={$issues $by-repo @t @t $~}
          read-x=|=(pax/path (get /repos/[-.+>.pax]/[-.+>+.pax]/issues))
          read-y=|=(pax/path (get /repos/[-.+>.pax]/[-.+>+.pax]/issues))
          sigh-x=sigh-list-issues-x
          sigh-y=sigh-list-issues-y
      ==
      ^-  place                       ::  /issues/by-repo/<user>/<repo>
      :*  guard={$issues $by-repo @t @t @t $~}
          ^=  read-x
          |=(pax/path (get /repos/[-.+>.pax]/[-.+>+.pax]/issues/[-.+>+>.pax]))
        ::
          read-y=(read-static ~)
          ^=  sigh-x
          |=  jon/json
          %+  bind  (issue:gh-parse jon)
          |=  issue/issue:gh
          gh-issue+issue
        ::
          sigh-y=sigh-strange
      ==
  ==
::
::  When a peek on a path blocks, ford turns it into a peer on
::  /scry/{care}/{path}.  You can also just peer to this
::  directly.
::
::  We hand control to ++scry.
::
++  peer-scry
  |=  pax/path
  ^-  {(list move) _+>.$}
  ?>  ?=({care *} pax)
  :_  +>.$  :_  ~
  (read:connector ost.hid (places %read pax) i.pax t.pax)
::
::  HTTP response.  We make sure the response is good, then
::  produce the result (as JSON) to whoever sent the request.
::
++  sigh-httr
  |=  {way/wire res/httr}
  ^-  {(list move) _+>.$}
  ?.  ?=({$read care @ *} way)
    ~&  res=res
    [~ +>.$]
  =*  style  i.way
  =*  ren  i.t.way
  =*  pax  t.t.way
  :_  +>.$  :_  ~
  :+  ost.hid  %diff
  (sigh:connector (places ren style pax) ren pax res)
::
::  HTTP error.  We just print it out, though maybe we should
::  also produce a result so that the request doesn't hang?
::
++  sigh-tang
  |=  {way/wire tan/tang}
  ^-  {(list move) _+>.$}
  %-  (slog >%gh-sigh-tang< tan)
  [[ost.hid %diff null+~]~ +>.$]
::
::  We can't actually give the response to pretty much anything
::  without blocking, so we just block unconditionally.
::
++  peek
  |=  {ren/@tas tyl/path}
  ^-  (unit (unit (pair mark *)))
  ~ ::``noun/[ren tyl]
::
::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
::         Webhook-powered event streams (/listen)            ::
::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
::
::  To listen to a webhook-powered stream of events, subscribe
::  to /listen/<user>/<repo>/<events...>
::
::  We hand control to ++listen.
::
++  peer-listen
  |=  pax/path
  ^-  {(list move) _+>.$}
  ?.  ?=({@ @ *} pax)
    ~&  [%bad-listen-path pax]
    [~ +>.$]
  (listen pax)
::
::  This core handles event subscription requests by starting or
::  updating the webhook flow for each event.
::
++  listen
  |=  pax/path
  =|  mow/(list move)
  =<  abet:listen
  |%
  ++  abet                                              ::  Resolve core.
    ^-  {(list move) _+>.$}
    [(flop mow) +>.$]
  ::
  ++  send-hiss                                         ::  Send a hiss
    |=  hiz/hiss
    ^+  +>
    =+  wir=`wire`[%x %listen pax]
    +>.$(mow [[ost.hid %hiss wir `~ %httr [%hiss hiz]] mow])
  ::
  ::  Create or update a webhook to listen for a set of events.
  ::
  ++  listen
    ^+  .
    =+  pax=pax  ::  TMI-proofing
    ?>  ?=({@ @ *} pax)
    =+  events=t.t.pax
    |-  ^+  +>+.$
    ?~  events
      +>+.$
    ?:  (~(has by hook) i.events)
      $(+>+ (update-hook i.events), events t.events)
    $(+>+ (create-hook i.events), events t.events)
  ::
  ::  Set up a webhook.
  ::
  ++  create-hook
    |=  event/@t
    ^+  +>
    ?>  ?=({@ @ *} pax)
    =+  clean-event=`tape`(turn (trip event) |=(a/@tD ?:(=('_' a) '-' a)))
    =.  hook
      %+  ~(put by hook)  (crip clean-event)
      =+  %+  fall
            (~(get by hook) (crip clean-event))
          *{id/@t listeners/(set bone)}
      [id (~(put in listeners) ost.hid)]
    %-  send-hiss
    :*  %+  scan
          =+  [(trip i.pax) (trip i.t.pax)]
          "https://api.github.com/repos/{-<}/{->}/hooks"
        auri:epur
        %post  ~  ~
        %-  taco  %-  crip  %-  pojo  %-  jobe  :~
          name+s+%web
          active+b+&
          events+a+~[s+event] ::(turn `(list ,@t)`t.t.pax |=(a=@t s/a))
          :-  %config
          %-  jobe  :~
            =+  =+  clean-event
                "http://107.170.195.5:8443/~/to/gh/gh-{-}.json?anon&wire=/"
            [%url s+(crip -)]
            [%'content_type' s+%json]
          ==
        ==
    ==
  ::
  ::  Add current bone to the list of subscribers for this event.
  ::
  ++  update-hook
    |=  event/@t
    ^+  +>
    =+  hok=(~(got by hook) event)
    %_    +>.$
        hook
      %+  ~(put by hook)  event
      hok(listeners (~(put in listeners.hok) ost.hid))
    ==
  --
::
::  Pokes that aren't caught in more specific arms are handled
::  here.  These should be only from webhooks firing, so if we
::  get any mark that we shouldn't get from a webhook, we reject
::  it.  Otherwise, we spam out the event to everyone who's
::  listening for that event.
::
++  poke
  |=  response/hook-response
  ^-  {(list move) _+>.$}
  =+  hook-data=(~(get by hook) (rsh 3 3 -.response))
  ?~  hook-data
    ~&  [%strange-hook hook response]
    [~ +>.$]
  ::  ~&  response=response
  :_  +>.$
  %+  turn  (~(tap in listeners.u.hook-data))
  |=  ost/bone
  [ost %diff response]
--
```
