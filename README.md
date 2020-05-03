# EdgeDB-cookbook
Shiny and new -- EdgeDB is a relational database that stores and describes the data as strongly typed objects and relationships between them.
Assuming you already have the database set up, here are some purposeful statements you could use:

See https://spectrum.chat/edgedb/general/personal-accomplishment-edgedb-authentication-from-svelte-js~ff16a60c-91b2-4e85-aa13-1aa3d22d9500

After annoying Elvis and Yury for a solid 7 months on various chat platforms, I've finally built a backend API server that authenticates against EdgeDB. Please be warned that this is a very tiny window into the project, and it does not cover network configuration, satisfying browser CORS policies, etc. If anyone is interested in building something similar for their web app, here are some of the high-level steps my application implements:

# 1) Svelte-JS makes an async POST request to the API server's url with the user's email and password.

```js
  let email = "";
  let password = "";
  let result;
  let text;
  let jD;
  let data;
  let combined;
  $: combined = {
    email: email,
    password: password
  };
  $: if(jD) { $session.self = JSON.parse(jD) }
  $: if(data) { jD = data; $session.text = data; }

  async function login() {
    let response = await fetch("/api/v1/login", {
      method: "post",
      mode: "cors",
      credentials: "same-origin",
      headers: {
        "Content-Type": "application/json"
      },
      redirect: "follow",
      referrerPolicy: "no-referrer",
      body: JSON.stringify(combined)
    });
    let text = await response.text();
    let data = text;
    return data;
  }

  async function submitHandler() {
    result = login();
    email = "";
    password = "";
  }
```

# 2) Nginx reverse-proxies the request to a backend Python API server (running on MagicStack's uvloop).

### Nginx config

```
location /api/v1/ {
        proxy_set_header Accept-Encoding "";
        proxy_pass http://<address>:<port>;
        proxy_http_version  1.1;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade; 
        proxy_set_header Host   "<domain>.com";
        proxy_buffering off;
        proxy_redirect off;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port  443;
        proxy_set_header X-Forwarded-Host  $host;
        limit_req zone=reqlimit burst=6;
        limit_req_status 460; # Status to send
        limit_conn connlimit 20;
}
```

### Backend / API

Here is a small look at what authentication logic could look like in an asynchronous context.

```python
import hashlib
import asyncio
import concurrent.futures as cf
from api.edgedb import fetchall_json, fetchone_json

def hashpass(password=""):
    # Business systems should use stronger encryption methods
    _hash = hashlib.md5()
    _hash.update(password.encode('UTF-8'))
    return _hash.hexdigest()

async def aiohashpass(password=""):
    loop = asyncio.get_running_loop()
    executor = cf.ThreadPoolExecutor()
    future_pass = loop.run_in_executor(executor, hashpass, password)
    _hash = await future_pass
    return _hash

async def create_new_session(uuid="", sessionEvent=None):
    query = 'select api::create_new_session(<uuid>"%s");' % (uuid)
    await fetchone_json(query)
    sessionEvent.set()

async def validate_credentials(email, password):
    password = await aiohashpass(password)
    query = '''select api::validate_credentials("%s", "%s") { id };''' % (email, password)
    response = await fetchone_json(query)
    return response

async def get_guest_user():
    query = '''select api::User { name, isGuest } filter .name = "Guest" LIMIT 1;'''
    response = await fetchall_json(query)
    return response

async def trash_collect():
    await fetchall_json('''select api::clear_expired();''')
```

In this context, the API server is responsible for encrypting the password, building a url-quoted query with the data supplied by the client, and shipping it off to the HTTP EdgeDB pool server, which implements four different routes:

```
-  '/' :                     Returns '{}' for validating that the server is up and running.
-  '/execute/<statement>':   Executes the statement provided and returns '{}'.
-  '/fetchone_json/<query>': Awaits a `fetchone_json` call for a given query, 
                              like `SELECT api::validate_credentials(...) { id, user: {...}, ... }`. 
- '/fetchall_json/<query>':  Awaits a `fetchall_json` call like the example above, except that his route returns a list of corresponding set
                              objects as JSON.
```

This service is managed by Systemd with a run script and is configured to compress and rotate logs via logrotate, a standard logging daemon.

```bash
#!/bin/bash
( python3.8 edbPoolServer.py > /var/log/app/access_edgedb.log 2> /var/log/app/error_edgedb.log & )
sleep 1
status="$( curl -s http://127.0.0.1:6565/ )" \
    && if [[ "$status" == "{}" ]]; then 
              echo -e "The EdgeDB API database connector is running...\n"; 
       else
            echo -e "Error.. Exiting.";
       fi
```

## 3) An EdgeDB module

Here are a few things I use in the API module on the database side:

```sql
function api::clear_expired() -> SET OF api::Session using (
        WITH MODULE api
        DELETE Session
        IF
        (
            <datetime>Session.createdAt + <duration>Session.allottedDuration <= datetime_current()
        )
        ELSE <Session>{}
        # Important to cast an empty set as a SET OF Session objects
        # so that `IF` and `ELSE` can match types, but also feed
        # something to `DELETE` in case no matches could be found
    );

function api::create_new_session(EMAIL: std::str, PASS: std::str) -> SET OF api::Session using (
        WITH MODULE api 
        FOR USER IN {api::validate_credentials(EMAIL, PASS)}
        UNION (
            INSERT Session {
                token := <str>(SELECT api::random_big_str_id()),
                sessionID := <str>(SELECT api::random_big_str_id()),
                user := USER
            }   
        )
     );

function api::validate_credentials(EMAIL: std::str, PASS: std::str) -> SET OF api::User using (
        WITH MODULE api
        SELECT
            User
        FILTER
            .email = EMAIL
            AND
            .password = PASS
        LIMIT 1
    );
```

but there are also some one-off utility functions for things are not immediately obvious. For example, what if you want to create a Session that will never be deleted by `api::clear_expired()` but works within the boundaries of the `std::duration` type? To satisfy this kind of need, you might implement a helper method that returns a `std::duration` value that is virtually infinite:

```sql
function api::forever_duration() ->  std::duration using (
        SELECT <duration>'867240 hours';
    );
```

Likewise, if you needed a random string of numbers, you might use the `std::re_replace` function with:

```sql
function api::rand_num_string() ->  std::str using (
        SELECT 
            std::re_replace(
                # This removes the space at the beginning:
                r'^[ \t]+',
                r'',
                (
                    SELECT std::to_str(random() * 10 ^ 13, '99999999999999')
                )
            )
    );
```

Finally, you might need a type to store sessions for easy lookup:

```sql
type api::Session {
    required single link user -> api::User;
    required single property allottedDuration -> std::duration {
        # I wasn't a fan of writing this part (and there probably is a better way)
        # but it was needed to satisfy the CLI.
        default := (WITH
            MODULE api
        SELECT
            <duration>'24 hours'
        );
    };
    required single property createdAt -> std::datetime {
        default := (WITH
            MODULE api
        SELECT
            datetime_current()
        );
    };
    required single property sessionID -> std::str;
    single property token -> std::str;
};
```

### Back at the API server

Depending on the EdgeDB pool server's response, the API server will subsequently perform any actions relevant to the data returned. Perhaps log any relevant behaviors, and then return a minimized JSON object that reflects the data passed in the original request body.

## 4) If the credentials were valid database records, Svelte will surgically ablate the shackles that bind 'guest-only' views to the DOM.

My current theory is that it will be possible to facilitate a continuous dialogue with Svelte and EdgeDB whereby EdgeDB acts as a template factory for DOM components.

### "But this is sounding an awful lot like React. Isn't the whole point of Svelte to get away from the bulkiness of a virtual DOM?"

Response: "Yes, that's true. But I would argue this is a different artifact entirely — because, in a cladistic and literal sense, React is not a database. What's more, Svelte doesn't currenly have a way to dynamically implement generic types."

## 5) (Under research development)

### Using EdgeDB as a meta-object generator for <insert thing here>

Yes, something like this is a better suited to a pure programming language, but it just sounds like more fun to have the figurative kitchen knives and swiss army knifes taken away to solve such a puzzle. Where would the fun be in solving a problem that has a million different solutions already?
Moreover, I think it opens up the possibility that a database can be more than just a SQL server. Maybe it can be an engine, a core, or a robotic heart that has peripherals and some external interfaces running on a network port.

### "Your scientists were so preoccupied with whether or not they could, they didn't stop to think if they should." -- Jurassic Park

A non-traditional database that offers more than just SELECT statements could be the start of a human-like brain that's written in bits rather than in blood. Whimsy aside, I think EdgeDB has the potential to serve AI Engineering, web development, server networking, and many other non-traditional arenas.

(Update: Additionally, the YouTube channel "Code Artistry" has a tutorial on how to integrate Svelte with GraphQL if you facilitate EdgeDB's direct integration with Svelte ) https://www.youtube.com/watch?&v=WqOLx2yuF3M
