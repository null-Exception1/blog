---
title: How I built RoommateFinder (and my first fullstack)
date: 2026-07-08 00:00:00 +0000
---
# Roommate Finder

So I initially got the inspiration when I saw a terribly vibe coded website from a senior from my university, at which time i was also looking for an excuse to somehow learn Next-JS framework and Go.
It's kinda important to me that frontend should look alright so i picked NextJS for the project, and Go was my natural choice because i wanted to learn an objectively high performant language i could build a backend on.


# Goals

Specifically I had some goals pertaining to my project - 

I wanted to learn
- **Go concurrency**, from scratch, then be able to effectively intuitively build schema that would leverage the abilities of Go. My general backend used to be a Python Flask API that usually was my goto if i wanted to script something fast, but hey, i gotta grow to better things.
- **Next-JS frontend** - I wanted to be able to build a minimal but good looking frontend with minimal effort, and learn React jsx as well as Tailwind
- **PostgresSQL** - Since I was working *primarily* with users, registration and sessions, what better way than to work on one of the most compatible, well known databases in the world. I have had some *MongoDB* and *Firebase DB* experience but since i've heard Postgres is highly optimized, i wanted to leverage that for my site.
- **Dockerization** - I once built a docker container for a pwn ctf. I know how exactly they work and for what reasons they're used - but easier said than done. I might as well learn everything for containerization including proper Dockerfile language and Docker Compose setups

## Frontend

> So, what all do I already know?

Well, i've been using pure HTML CSS JS for the majority of my years in webdev (yuck i know).

>The thing about using raw CSS is that it almost never turns out well, in fact majority of frontend is dependent on reusable assets, learning proper UI design, tricks, plugins and things that would probably befuddle a backend programmer like me.

This is why my primary target was to first learn JSX (which is the way to HTMX rendering) which meant learning **useState** as well as **useEffect**. Couldn't have been very hard except for the fact that I'm extremely ass at frontend design and the best i could do was like 2 rectangles and 2 buttons.

I made like a small really shitty project without any styling or tailwind just to get the hang of basic JSX functioning because it boggled my mind ever since 10th grade how people could just write JSX. But i got the hang of it anyway.

>Another thing I had to learn on my own was **flexbox** and **grid** for tailwind css. 

As you can assume, after learning about components and quickly realising i didnt have to style my own components (yuck), I installed the daisyUI plugin into the project and quickly realised - "Wow, I really do not know how to place components on a page even if i knew how to copy them"

I do plan to get the grip of making better frontends by looking over the homework of some well done sites with dynamic sexy looking front pages - but damn, are they mostly all vibecoded kinda kills my motivation. 

like brother seriously this was the best i could do for day 1 of building the frontend first

```html
import Navbar from  "./components/navbar";
export  default  function  Home() {
return (
<div>
<Navbar  />
<div  className="flex h-screen max-h-screen justify-center min-h-screen items-center gap-4 bg-blue-100">
	<div  className="card bg-white shadow h-1/2 w-1/3">
		<div  className="card-body flex flex-col justify-evenly text-center items-center">
				<h1  className="text-5xl">Add yourself</h1>
					<h5  className="text-center text-2xl">alright bro what do you wanna do add yourself?</h5>
				<div  className="justify-center card-actions">
				<a  className="btn btn-primary"  href="/register">Register</a>
		</div>
	</div>
</div>
<div  className="card bg-white shadow h-1/2 w-1/3">
		<div  className="card-body flex flex-col justify-evenly text-center items-center">
			<h1  className="text-5xl">Search</h1>
			<h5  className="text-center text-2xl">browse the blocks broski</h5>
			<div  className="justify-center card-actions">
				<a  className="btn btn-primary"  href="/blocks">Browse</a>
			</div>
		</div>
	</div>
</div>
</div >
);
}
```
## Backend Engineering 

Finally, something i can talk about without an inferiority complex.

So at first I tried implementing the Go backend on my first shitty project. I had mostly looked forward to using the Ratelimit function i had which was the most complex thing happening in the code at the moment.

```go

func Routine() {
	for  range ticker.C {
		select {
			case ratelimit <- time.Now():
			default:
		}
	}
}
```
```go
func Ratelimit(handlerfunc  http.HandlerFunc) http.HandlerFunc {
	return  func(w  http.ResponseWriter, req  *http.Request) {
		select {
			case  <-ratelimit:
				handlerfunc(w, req)
			default:
				fmt.Println("[RATELIMIT EXCEEDED] Packet left out")
			}
		}
	}
}
```
As you can see, absolutely baby shit. I have no excuse - i'm a lightweight at writing systems languages. The most programming i've ever done in systems is when i've written physics simulations with C.

Go itself is taking it easy on me by not a headache to deal with. In fact I had just discovered how to distribute my code into separate files (which i desperately needed)

But anyways onward to writing my raw API v1

## My raw API v1

>I'll explain it to you in a nutshell. 

Basically what i needed (to rip off the vibe coded website) was to pretty much these endpoints

- **/blocks**
- **/rooms?block=**
- **/registration**
- **/login**

Each one of them serves its own purpose, **/blocks** shows you exactly what blocks are available to click on, and how many are Fully filled or Partially filled -
```go
type  Block  struct {
	Partial int
	Full int
}
```

Each person has their own struct
```go
type  Person  struct {
	Admnno string
	Name string
	Social string
	Socialtype string
	Roomno string
	Blockno string
	Created_at string
}

type  Room  struct {
People []*Person
}
```

## Privacy 

Before you yell at me - **NO, I was not going to use raw admission numbers from our university**. That would defeat the point of their privacy, and since our university isn't linked to the site in any way officially, that would mean data exposition would lead to some shady shit.

Instead my plan was this - Let's just hash the admission number, before it even hits the database, or the backend.

>Now, I hear you asking, "Dumbass, that means your frontend contains your encryption method, if they know your pepper, attackers can just brute force your admission numbers against the hashes"

Here's where it gets interesting - We hash it twice. Once in the frontend, the other time in the backend. Why? Why not just hash it in the backend?

Well the simple explanation is - I don't want even a singular leak of someone's real admission number, that means not even a **single** request to the server should contain someone's private data.

Imagine this

>Attacker has finally managed to get a handle on my backend. He's snuck in, finally placed his malware that hooked the process and he's now reading the requests that come in.

Yeah, this is why.

So the reason why it was purely written as "Admnno" in the struct was because i hadn't quite got to hashing yet, and i was operating under admnno for the duration of testing.


## Block

```go
Occupancies := make(map[string]map[string]int, 0)
rows := db.Query("SELECT * FROM people", globals.Globaldb)

Block := make(map[string]*structs.Block, 0)

for _, row := range rows {
	if _, ok := Occupancies[row.Blockno]; ok {
		Occupancies[row.Blockno][row.Roomno] = Occupancies[row.Blockno][row.Roomno] + 1
	} else {
		Occupancies[row.Blockno] = make(map[string]int, 0)
		Occupancies[row.Blockno][row.Roomno] = 1
	}
}

Block := make(map[string]*structs.Block, 0)

for key := range Occupancies {
	Block[key] = &structs.Block{Partial: 0, Full: 0}
	for _, occupancy := range Occupancies[key] {
		if occupancy >= 2 { // remind me to check the occupancy max limit later
			Block[key].Full = Block[key].Full + 1
		} else {
			Block[key].Partial = Block[key].Partial + 1
		}
	}
}
```
Honestly one of the more simpler ways would've been to experiment with Querying methods, which could've been cleaner but I felt like i wanted to rely more on my Go backend for this particular project.

It is objectively faster to do this level of filtering with a well formed SQL query but ya know... there's other stuff i want to add.

## Version 2: Sessions and Login

>So the idea is simple, as soon as a user logs in or registers, give them a session ID cookie which for the duration of validity can be used to automatically stay logged in. You send a /verify to verify the token everytime you visit the site.

Easy to say, hard to manage.

The main problem was the CORS that absolutely refused the living hell out of placing a cookie on a god forbid user.

Infact, i forgot that the Response was also sending back headers and the browser actually checks them?

```go
func Verify(w http.ResponseWriter, req *http.Request) {
	w.Header().Set("Access-Control-Allow-Origin", "http://localhost:3000")
	w.Header().Set("Access-Control-Allow-Credentials", "true")
	w.Header().Set("Access-Control-Allow-Methods", "GET, OPTIONS")
	w.Header().Set("Access-Control-Allow-Headers", "Content-Type")
	w.Header().Set("Content-Type", "application/text")
```

If you don't do this, it throws CORS error, because browsers will allow you to send data to that resources but if it doesn't send the right headers back, it could be malicious. So browsers invented a genius way to protect the user - which involved blocking the site altogether, leading me to try to figure out what the hell is going wrong for an hour straight.

I've dealt with CORS while working on javascript projects before, but i hate the fact that some random headers that look kind of useless on the outside have massive implications on the browser.

Another thing while learning this realisation was that Server components in React are asynchronous, while Client components are synchronous. This was intuitive to me but came as a shock to me.

```jsx

useEffect(() => {
    fetch("http://localhost:8080/verify", { credentials: "include" })
      .then(res => res.text())
      .then(data => setValid(data === "valid"))
      .catch(() => setValid(false));
  }, []);
```
*Client module*


```jsx
const { id } = await params; // Await params directly on the server
const res = await fetch(`http://localhost:8080/rooms?block=${id}`);
const loaded_data = await res.json();
```
*Server module*

The main fact is that server components have the indispensable ability to just wait it out.

## Logging

Generally logging is not my specialty. Neither is keeping track of explaining my code. But i'm trying to level up my production level game, and that requires me to debug my code. Golang is notoriously hard to debug compared to Python simply because the errors make absolutely zero sense sometimes.

```go
func Blocks(w http.ResponseWriter, req *http.Request) {

	logrus.WithFields(logrus.Fields{
		"package":  "handlers",
		"endpoint": "/blocks",
		"method":   req.Method,
		"remote":   req.RemoteAddr,
	}).Info("requested /blocks by ", req.RemoteAddr)

	w.Header().Set("Access-Control-Allow-Origin", "http://localhost:3000")
	w.Header().Set("Access-Control-Allow-Credentials", "true")
	w.Header().Set("Access-Control-Allow-Methods", "GET, OPTIONS")
	w.Header().Set("Access-Control-Allow-Headers", "Content-Type")
	w.Header().Set("Content-Type", "application/text")

	Occupancies := make(map[string]map[string]int, 0)
	rows := db.Query("SELECT * FROM people", globals.Globaldb)

	logrus.WithFields(logrus.Fields{
		"package":  "handlers",
		"endpoint": "/blocks",
		"rows":     len(rows),
		"method":   req.Method,
		"remote":   req.RemoteAddr,
	}).Info("fetched rows from postgres db")

	for _, row := range rows {
		if _, ok := Occupancies[row.Blockno]; ok {
			Occupancies[row.Blockno][row.Roomno] = Occupancies[row.Blockno][row.Roomno] + 1
		} else {
			Occupancies[row.Blockno] = make(map[string]int, 0)
			Occupancies[row.Blockno][row.Roomno] = 1
		}
	}

	logrus.WithFields(logrus.Fields{
		"package":     "handlers",
		"endpoint":    "/blocks",
		"occupancies": len(Occupancies),
		"method":      req.Method,
		"remote":      req.RemoteAddr,
	}).Debug("made occupancies map")

	Block := make(map[string]*structs.Block, 0)

	for key := range Occupancies {
		Block[key] = &structs.Block{Partial: 0, Full: 0}
		for _, occupancy := range Occupancies[key] {
			if occupancy >= 2 { // remind me to check the occupancy max limit later
				Block[key].Full = Block[key].Full + 1
			} else {
				Block[key].Partial = Block[key].Partial + 1
			}
		}
	}

	logrus.WithFields(logrus.Fields{
		"package":  "handlers",
		"endpoint": "/blocks",
		"blocks":   len(Block),
		"method":   req.Method,
		"remote":   req.RemoteAddr,
	}).Debug("made blocks map")

	str, err := json.Marshal(Block)

	if err != nil {
		logrus.WithFields(logrus.Fields{
			"package":   "handlers",
			"endpoint":  "/blocks",
			"jsonified": len(str),
			"method":    req.Method,
			"remote":    req.RemoteAddr,
		}).Error("error in JSON.Marshal")
	}

	logrus.WithFields(logrus.Fields{
		"package":   "handlers",
		"endpoint":  "/blocks",
		"jsonified": len(str),
		"status":    http.StatusOK,
		"method":    req.Method,
		"remote":    req.RemoteAddr,
	}).Info("response sent")

	fmt.Fprintf(w, "%s", string(str))
}
```

There's different levels of logging as far as i could tell, i only needed 4 - **Info**, **Debug**, **Error**, **Fatal**

There was also **Trace** but i don't think i was coding anything that big in size for me to have to trace back to anything.

So now, our logs look like this -

```
INFO[2026-07-08T18:40:37Z] requested /blocks by [::1]:48478              endpoint=/blocks method=GET package=handlers remote="[::1]:48478"
DEBU[2026-07-08T18:40:37Z] fetch query recieved, making results..        function=query package=db
DEBU[2026-07-08T18:40:37Z] returning results from query...               function=query package=db results=7
DEBU[2026-07-08T18:40:37Z] fetched rows from postgres db                 endpoint=/blocks package=caching
DEBU[2026-07-08T18:40:37Z] made occupancies map                          endpoint=/blocks occupancies=5 package=caching
DEBU[2026-07-08T18:40:37Z] made blocks map                               blocks=5 endpoint=/blocks package=caching
DEBU[2026-07-08T18:40:37Z] updating the cache..                          blocks=5 endpoint=/blocks package=caching
DEBU[2026-07-08T18:40:37Z] updated the cache.                            cache=5 package=caching
DEBU[2026-07-08T18:40:37Z] CACHE MISS!                                  
INFO[2026-07-08T18:40:37Z] response sent                                 endpoint=/blocks jsonified=140 method=GET package=handlers remote="[::1]:48478" status=200
DEBU[2026-07-08T18:40:37Z] checking session in db                        endpoint=/verify method=GET package=handlers
INFO[2026-07-08T18:40:37Z] no matching session found in db               endpoint=/verify package=handlers token=a452f271b62de95d2bc7af2a3cb45d53
DEBU[2026-07-08T18:40:38Z] checking session in db                        endpoint=/verify method=GET package=handlers
INFO[2026-07-08T18:40:38Z] requested /blocks by [::1]:48476              endpoint=/blocks method=GET package=handlers remote="[::1]:48476"
DEBU[2026-07-08T18:40:38Z] CACHE HIT!                                   
INFO[2026-07-08T18:40:38Z] response sent                                 endpoint=/blocks jsonified=140 method=GET package=handlers remote="[::1]:48476" status=200
INFO[2026-07-08T18:40:38Z] no matching session found in db               endpoint=/verify package=handlers token=a452f271b62de95d2bc7af2a3cb45d53
```

I know.. it looks amazing. Simply being able to locate an error in processing or having a LOG stored of it is a huge advantage in debugging and reducing a LOT of debugging time.

## Testing

So the project is pretty big at this point. Debugging is getting still progressively difficult. At this point I had even more difficult things. I also needed a test to make sure at any point of the process, we didn't mess up what already worked.

```go

func TestSignupflow(t *testing.T) {
	godotenv.Load("../../.env")

	initiation.Database()

	// registration flow
	logrus.Info("/registration testing in progress..")

	req := httptest.NewRequest("GET", "/registration?admnno=69&name=Shaurya&social=discordusername&socialtype=Discord&blockno=16&roomno=123&created_at=now", nil)

	w := httptest.NewRecorder()

	handlers.RegistrationHandler(w, req)

	resp := w.Result()
	body := w.Body.String()

	if resp.StatusCode != http.StatusOK {
		t.Fatalf("expected 200, got %d", resp.StatusCode)
	}

	if body != "done" {
		t.Errorf("expected 'done', got %q", body)
	}
	cookies := resp.Cookies()
	if len(cookies) == 0 {
		t.Fatal("expected a cookie, got none")
	}
	token := cookies[0].Value

	logrus.Info("/registration testing in complete..")

	defer func() {
		globals.Globaldb.Exec("DELETE FROM people WHERE admn_hash='69'")
		globals.Globaldb.Exec("DELETE FROM sessions WHERE admnno='69'")
	}()

	logrus.Info("got token as (assume its not malformed) ", token)

	logrus.Info("/verify testing in progress..")

	req = httptest.NewRequest("GET", "/verify", nil)
	req.AddCookie(&http.Cookie{Name: "sess_id", Value: token})
	w = httptest.NewRecorder()

	handlers.Verify(w, req)

	resp = w.Result()
	body = w.Body.String()

	if resp.StatusCode != http.StatusOK {
		t.Errorf("expected 200, got %d", resp.StatusCode)
	}

	if body != "valid" {
		t.Errorf("expected 'valid', got %q", body)
	}

	logrus.Info("/verify ended..")

	// logout flow
	logrus.Info("/logout testing in progress..")

	req = httptest.NewRequest("GET", "/logout", nil)

	req.AddCookie(&http.Cookie{Name: "sess_id", Value: token})
	w = httptest.NewRecorder()

	handlers.Logout(w, req)

	resp = w.Result()
	body = w.Body.String()

	if resp.StatusCode != http.StatusOK {
		t.Errorf("expected 200, got %d", resp.StatusCode)
	}

	if body != "logged out" {
		t.Errorf("expected 'logged out', got %q", body)
	}

	logrus.Info("/logout testing complete..")

	// login flow
	logrus.Info("/login testing in progress..")

	req = httptest.NewRequest("GET", "/login?admn_hash=69&name=Shaurya", nil)

	w = httptest.NewRecorder()

	handlers.Login(w, req)

	resp = w.Result()
	body = w.Body.String()

	if resp.StatusCode != http.StatusOK {
		t.Errorf("expected 200, got %d", resp.StatusCode)
	}

	if body == "not found" {
		t.Errorf("expected token, got %q", body)
	}

	logrus.Info("/login testing complete..")

}
```

It's pretty elegant. It tests the entire sign up flow with log out, session validity and verification. This is just to make sure that when i add the **caching mechanisms**, i was going to not give up any more time trying to fix something in a project as big as this, especially in a language i spent only 3 days learning through gobyexample.com

*Registration handler*
```go

func TestRegistrationHandler(t *testing.T) {
	if err := godotenv.Load("../../.env"); err != nil {
		t.Fatalf("failed to load .env: %v", err)
	}
	initfuncs.Database()

	req := httptest.NewRequest("GET",
		"/registration?admnno=69&name=Shaurya&social=discordusername&socialtype=Discord&blockno=16&roomno=123&created_at=now",
		nil)
	w := httptest.NewRecorder()

	handlers.RegistrationHandler(w, req)

	resp := w.Result()
	if resp.StatusCode != http.StatusOK {
		t.Fatalf("expected 200, got %d", resp.StatusCode)
	}
	if w.Body.String() != "done" {
		t.Errorf("expected 'done', got %q", w.Body.String())
	}

	cookies := resp.Cookies()
	if len(cookies) == 0 {
		t.Fatal("expected a cookie, got none")
	}

	if _, err := globals.Globaldb.Exec("DELETE FROM people WHERE admn_hash=$1", "69"); err != nil {
		t.Logf("cleanup people failed: %v", err)
	}
	if _, err := globals.Globaldb.Exec("DELETE FROM sessions WHERE admnno=$1", "69"); err != nil {
		t.Logf("cleanup sessions failed: %v", err)
	}
}
```

*Login handler*

```go
package individualflows

import (
	"golang/globals"
	"golang/handlers"
	initfuncs "golang/init"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/joho/godotenv"
)

func TestLoginHandler(t *testing.T) {
	// Seed DB with a fake user first
	godotenv.Load("../../.env")

	initfuncs.Database()

	_, err := globals.Globaldb.Exec(`
    INSERT INTO people (admn_hash, name, social, socialtype, roomno, blockno)
    VALUES ($1, $2, $3, $4, $5, $6)
`, "69", "Shaurya", "discordusername", "Discord", 123, 16)

	if err != nil {
		t.Fatalf("insert failed: %v", err)
	}

	req := httptest.NewRequest("GET", "/login?admn_hash=69&name=Shaurya", nil)
	w := httptest.NewRecorder()

	handlers.Login(w, req)

	resp := w.Result()
	if resp.StatusCode != http.StatusOK {
		t.Fatalf("expected 200, got %d", resp.StatusCode)
	}
	if w.Body.String() == "not found" {
		t.Errorf("expected token, got %q", w.Body.String())
	}

	// cleanup
	globals.Globaldb.Exec("DELETE FROM people WHERE admn_hash='69'")
	globals.Globaldb.Exec("DELETE FROM sessions WHERE admnno='69'")
}
```

*Logout Handler*
```

func TestLogoutHandler(t *testing.T) {
	godotenv.Load("../../.env")

	initfuncs.Database()
	_, err := globals.Globaldb.Exec(`
        INSERT INTO sessions (id, admnno) VALUES ($1, $2)
    `, "a18fbd57f9bbfd0450659cb69333415f", "69")
	if err != nil {
		t.Fatalf("failed to insert session: %v", err)
	}

	req := httptest.NewRequest("GET", "/logout", nil)
	req.AddCookie(&http.Cookie{Name: "sess_id", Value: "a18fbd57f9bbfd0450659cb69333415f"})
	w := httptest.NewRecorder()

	handlers.Logout(w, req)

	resp := w.Result()
	if resp.StatusCode != http.StatusOK {
		t.Errorf("expected 200, got %d", resp.StatusCode)
	}
	if w.Body.String() != "logged out" {
		t.Errorf("expected 'logged out', got %q", w.Body.String())
	}

	// cleanup
	globals.Globaldb.Exec("DELETE FROM sessions WHERE admnno=$1", "69")
}
```

*Verify handler*
```go

func TestVerifyHandler(t *testing.T) {
	// Load env + init DB
	if err := godotenv.Load("../../.env"); err != nil {
		t.Fatalf("failed to load .env: %v", err)
	}
	initfuncs.Database()

	// Seed DB with a fake session
	fakeToken := "a18fbd57f9bbfd0450659cb69333415f"
	_, err := globals.Globaldb.Exec(`
        INSERT INTO sessions (id, admnno, expires_at)
        VALUES ($1, $2, NOW() + interval '1 hour')
    `, fakeToken, "69")
	if err != nil {
		t.Fatalf("failed to insert session: %v", err)
	}

	// Make request with cookie
	req := httptest.NewRequest("GET", "/verify", nil)
	req.AddCookie(&http.Cookie{Name: "sess_id", Value: fakeToken})
	w := httptest.NewRecorder()

	handlers.Verify(w, req)

	resp := w.Result()
	if resp.StatusCode != http.StatusOK {
		t.Errorf("expected 200, got %d", resp.StatusCode)
	}
	if w.Body.String() != "valid" {
		t.Errorf("expected 'valid', got %q", w.Body.String())
	}

	// Cleanup
	if _, err := globals.Globaldb.Exec("DELETE FROM sessions WHERE admnno=$1", "69"); err != nil {
		t.Logf("cleanup failed: %v", err)
	}
}
```

**These are all the handlers we need to test separately** - This is more of the better depth tests which allow me to narrow down the problems. The few times where my session handling failed, these tests helped me find the problem extremely fast and elegantly.


Although I did have to update the test at some point or another in the upgradation process, making them a bit of hindrance to my progress. But this is the price you pay for never having to go step by step for a whole day trying to figure out the problem with your code.

## Benchmarking

So the main problem I was actively looking forward to solving was the optimization for handling /blocks and /rooms fetch api. Obviously they're the most heavy duty workers, and they're doing the most complex tasks.

I made a small script to benchtest our workers by seeding a **test_db** with 1000 fake users - 

```py
import psycopg2
import random
import string
from datetime import datetime
from faker import Faker

fake = Faker()

def random_hash(length=8):
    return ''.join(random.choices(string.ascii_lowercase + string.digits, k=length))

def seed_users(n=1000):
    conn = psycopg2.connect(
        dbname="test_db",
        user="devuser",
        password="devpass",
        host="localhost",
        port="5432"
    )
    cur = conn.cursor()

    for i in range(n):
        admn_hash = random_hash(12)
        name = fake.name()
        social = fake.user_name()
        socialtype = random.choice(["Discord", "Twitter", "Instagram"])
        roomno = random.randint(1, 200)
        blockno = random.randint(1, 20)
        created_at = datetime.now()

        cur.execute("""
            INSERT INTO people (admn_hash, name, social, socialtype, roomno, blockno, created_at)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
        """, (admn_hash, name, social, socialtype, roomno, blockno, created_at))

    conn.commit()
    cur.close()
    conn.close()
    print(f"Seeded {n} users into people table.")

if __name__ == "__main__":
    seed_users(1000)
```

So now we'll go step by step in the optimizations I added.

## 1. Caching (for /blocks)

>Refresh the page 1000 times, or give 1000 people 1 refresh... no difference.

So everyone knows the concept of caching. I made the caching mechanism myself as a good way to get a good grasp on caching and it's use cases. 

Now fortunately this would allow me to dwelve deeply into sync.Mutex Locking which I was looking forward to since gobyexample.com had taught me well enough to know when I should use it. Cache is a thing that can be edited and read by multiple handlers at the same time so it'll always be need to be protected to prevent a **race condition**.

So my first approach was kind of a master-class in how NOT to approach Locking. This was partially due to the fact that i underestimated the bottleneck of JSON.Marshal that managed to overshadow our results.

```go

if time.Now().After(globals.CacheExpiry) {
	caching.CacheUpdate() // Cache Update
}
globals.CacheMutex.RLock()
str, err := json.Marshal(globals.CacheBlocks) // Retrieve the cache
globals.CacheMutex.RUnlock()

```

```go
func CacheUpdate(){
	// .. boring sql stuff to retrieve stuff from db to update our cache

	globals.CacheMutex.Lock()
	clear(globals.CacheBlocks) // clear cache
	globals.CacheBlocks = Block
	globals.CacheExpiry = time.Now().Add(30 * time.Second)
	globals.CacheMutex.Unlock()

	// ..exit 
}
```

You see this is not a good idea when you check benchmarks - time for cache fetching went **UP** instead because of the sheer amount of locking we started doing for reading, added on with our bottleneck that i discovered later to be **JSON.Marshal**

Environment
- **OS**: Linux  
- **Arch**: amd64  
- **CPU**: Intel(R) Core(TM) i7-9750H @ 2.60GHz  
- **Go pkg**: `golang/benchmarks/simplefetch`

---

### Without Caching

| Benchmark | Iterations | Time/op | Bytes/op | Allocs/op |
| :--- | :---: | :---: | :---: | :---: |
| **BlocksHandler** | 2,877 | 388,047 ns | 258,546 | 2,207 |
| **RoomsBlocksHandler** | 1,000,000,000 | 0.0009717 ns | 0 | 0 |

---

### With Caching

| Benchmark | Iterations | Time/op | Bytes/op | Allocs/op |
| :--- | :---: | :---: | :---: | :---: |
| **BlocksHandler** | 28 | 54,649,916 ns | 17,629,019 | 459,798 |
| **RoomsBlocksHandler** | 1,000,000,000 | 0.02232 ns | 0 | 0 |

---

Infact the first time i tried caching, the ns/op went up, although the time was still lower than going for a fetch.


## The JSON.Marshal optimization

Turns out, JSONifying a dictionary is not worth it when you have to do it everytime you recall a cache hit. That defeats the point of a cache hit, you can save on time by just saving the JSON string in the Cache instead. That's where i realised "Cache is a very flexible idea and applied in many ways, not just objects"

```go
if time.Now().After(globals.CacheExpiry) {
	caching.CacheUpdate()
}
globals.CacheMutex.RLock()
str, err := json.Marshal(globals.CacheBlocks) // We have to Marshal it everytime we retrieve the Cache (Bottleneck)
globals.CacheMutex.RUnlock()
```

TLDR; Turns out instead of doing

DB -> Cache -> JSON.Marshal(result) -> client

We shorten the run so the result is already marshalled so the end run becomes

DB -> Cache -> client

So the code becomes

```go
func CacheUpdate(){
	// ...
	globals.CacheMutex.Lock()
	bytes, err := json.Marshal(Block)
	globals.CachedBlocksJSON = string(bytes) // store it stringified, maximum preprocessing
	globals.CacheMutex.Unlock()
	//...
```

And then the thing took off like a rocket:


## Removed json.Marshal overhead (caching)

| Benchmark                  | Iterations | Time/op      | Bytes/op | Allocs/op |
|----------------------------|------------|--------------|----------|-----------|
| **BlocksHandler**          | 3780       | 287,733 ns   | 208,379  | 1723      |
| **RoomsBlocksHandler**     | 1,000,000,000 | 0.0003799 ns | 0        | 0         |

---

This number... 0.0003799 ns this is the kind of latency i expected from Go. 

Honestly felt kind of good about myself after that.

## 2. Worker Pools + Caching for /rooms

The most interesting part about /rooms is that each block number needs to have their own cache. This cache as we discussed earlier need be JSON.Marshal, but there was also something i noticed very early on while implementing caching - we can have multiple workers update the cache of different blocks.

Each block essentially acts as an independent cacher by itself so we keep track of those expiries in a dictionary.

Now the general case for our caching rooms looked like this

```go

// For caching rooms : block -> room number -> structs.Room
var CacheRooms map[string]map[string]*structs.Room = make(map[string]map[string]*structs.Room, 0)
var CachedRoomsJSON map[string]string = make(map[string]string, 0)

// Per block expiry time
var CacheBlocksExpiry map[string]time.Time = make(map[string]time.Time, 0)
var CacheRoomsMutex sync.RWMutex

```

The results of caching for /rooms wasn't very interesting - partially because they were small fetched results and i couldn't really figure out a way to stress test cacheing without a meaningful difference in the time saved.

So anyways onto the WorkerPool implementation -

```go
package structs

type RoomsJob struct { 
	Blockno string
}

type RoomsJobResult struct {
	Blockno string
	JSON    string
}
```

The essential de-mystified idea of having workers is having jobs to assign them to, so they can achieve tasks parallel-y. Now my initial bottleneck with doing workerpools was that there was VIRTUALLY NO DIFFERENCE IN THE TIME SAVE.

My initial idea was to have a worker that is given a Job to update the cache of a certain 'block number', and it would straight away update the cache when it was done generating the result from the DB.

The same sync Locks that it required to ensure that the cache data was safeguarded from a race condition - made it so that concurrent instances of workers had to actually line up enough to bottleneck on a single worker's work. So it essentially became a queue where the workers had to still sit in line and not pick up new jobs after being done with their previous ones.


Pretty crazy right?


### Fan-in/Fan-out implementation

>Give jobs, take results, only one goroutine is allowed to look at the results

This is a fucking amazing optimization right here - basically when our workers are done with a job, they no longer have to wait for anything to update the cache. They pass it to a channel called **results**.

The **results** channel is now taken care of by only ONE goroutine, so that means, severely less locking, and incredible cache refreshing times.

```go

func WorkersResults() {
	for result := range globals.CacheRoomsJobsResults {
		logrus.WithFields(logrus.Fields{
			"package":       "routine",
			"resultblockno": result.Blockno,
			"resultlen":     len(result.JSON),
		}).Debug("updating cache with result from worker")
		globals.CachedRoomsJSON.Store(result.Blockno, result.JSON)
		//<-globals.CacheRoomsJobsResults
	}
}
```

This is the even more insane part - Go comes with it's own library for sync Maps that handle locking and unlocking for you. In fact, they're well built to accomodate it.

So now our entire code has become

```go
// in func main()
for range globals.NumWorkers { // we started all our workers for infinite time.
		globals.CacheRoomsJobsWaitGroup.Go(func() {
			for job := range globals.CacheRoomsJobs {
				caching.CacheRoomsUpdate(job.Blockno)
			}
		})
	}

go goroutines.WorkersResults() // started our result cleanup at the same time

```
```go
// Cache update trigger
func AddCacheRoomsJob(blockno string) {
	globals.CacheRoomsJobs <- structs.RoomsJob{Blockno: blockno} // updating cache now means just adding a job to queue
}
```
```go

// Cache update result
rooms := FormRooms(blockno)

bytes, _ := json.Marshal(rooms)

globals.CacheRoomsJobsResults <- structs.RoomsJobResult{Blockno: blockno, JSON: string(bytes)}

globals.CacheBlocksExpiry.Store(blockno, time.Now().Add(globals.CacheRoomsSeconds*time.Second))
```

So overall our entire benchmark has dropped.. let us compare the worker wise distributions


`const NumWorkers = 1`

goos: linux
goarch: amd64
pkg: golang/benchmarks/simplefetch
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkRoomsBlocksHandler-12  2705 req/s           13353 B/op        124 allocs/op
PASS
ok      golang/benchmarks/simplefetch   28.848s

`const NumWorkers = 50`
goos: linux
goarch: amd64
pkg: golang/benchmarks/simplefetch
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkRoomsBlocksHandler-12  25360 req/s 85 allocs/op
PASS
ok      golang/benchmarks/simplefetch   14.656s


Anything beyond 50 the workers start to crowd, it becomes like that old saying

>Too many chefs spoil the broth


## Cryptography


Nothing special here but an honourable mention to our backend and frontend hashing

```go
admnno := q.Get("admn_hash")
admn_hash := globals.SecureHash(admnno, os.Getenv("PEPPER"))
```

```jsx
const encoder = new TextEncoder();
const data = encoder.encode(admnno); // concat admnno
const hashBuffer = await crypto.subtle.digest("SHA-256", data);
const hashArray = Array.from(new Uint8Array(hashBuffer));
const hashHex = hashArray.map(b => b.toString(16).padStart(2, "0")).join("");

const queryParams = new URLSearchParams({
	admn_hash: hashHex,
	name: name
});
```

## Dockerization & Deployment

So for the sake of it i learnt enough docker to launch containers for my db, go backend and frontend.

```
services:
  postgres:
    image: postgres:16
    container_name: pg-roommate
    restart: always
    environment:
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpass
      POSTGRES_DB: roommatefinder
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql

  golang:
    build: ./golang
    container_name: go-backend
    restart: always
    environment:
      DATABASE_URL: postgres://devuser:devpass@postgres:5432/roommatefinder?sslmode=disable
      LOG_FORMAT: text
      LOG_LEVEL: debug
      CACHING: true
      PEPPER: prabhansharularegaytogether
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      
  roomate-finder:
    build: ./roomate-finder
    container_name: nextjs-frontend
    restart: always
    command: npm run dev
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:8080
      SERVER_API_URL: http://go-backend:8080
    ports:
      - "3000:3000"
    depends_on:
      - golang
    volumes:
      - ./roomate-finder:/app 
      - /app/node_modules

volumes:
  pgdata:
```

Most learning the fact that nextjs is NOT easy to export.


My Postgres Instance right now is hosted on Neon, Go backend is on Render, and Next-JS is on Vercel. Overall i had 0 difficulties even trying to deploy, it seems like theyre almost built for fullstack apps like mine, but honestly i kind of wish i hadnt spread the workload between 3 different services. I would've liked to have my docker compose up and running if i had access to spin up some docker containers 24/7.


## Conclusion


yeah this project was dope lol

if anything i'm definitely going to try to implement sharding on my own or even cache sharding.

but for now i think this is enough.


alright bye.

