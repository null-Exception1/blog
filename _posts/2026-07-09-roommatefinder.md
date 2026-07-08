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
Before you yell at me - **NO, I was not going to use raw admission numbers from our university**. That would defeat the point of their privacy, and since our university isn't linked to the site in any way officially, that would mean data exposition would lead to some shady shit.

Instead my plan was this - Let's just hash the admission number, before it even hits the database, or the backend.

>Now, I hear you asking, "Dumbass, that means your frontend contains your encryption method, if they know your pepper, attackers can just brute force your admission numbers against the hashes"

Here's where it gets interesting - We hash it twice. Once in the frontend, the other time in the backend. Why? Why not just hash it in the backend?

Well the simple explanation is - I don't want even a singular leak of someone's real admission number, that means not even a **single** request to the server should contain someone's private data.

Imagine this

>Attacker has finally managed to get a handle on my backend. He's snuck in, finally placed his malware that hooked the process and he's now reading the requests that come in.

Yeah, this is why.

So the reason why it was purely written as "Admnno" in the struct was because i hadn't quite got to hashing yet, and i was operating under admnno for the duration of testing.



## Version 2: Sessions and Login

>So the idea is simple, as soon as a user logs in or registers, give them a session ID cookie which for the duration of validity can be used to automatically stay logged in
