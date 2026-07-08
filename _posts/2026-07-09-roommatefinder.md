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
Block := make(map[string]*structs.Block, 0)

for _, row := range rows {
	if _, ok := Occupancies[row.Blockno]; ok {
		Occupancies[row.Blockno][row.Roomno] = Occupancies[row.Blockno][row.Roomno] + 1
	} else {
		Occupancies[row.Blockno] = make(map[string]int, 0)
		Occupancies[row.Blockno][row.Roomno] = 1
	}
}

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

I know.. it looks amazing. 

