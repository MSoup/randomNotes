# Docker - Using Bind Mounts: error Couldn't find a package.json file in "/app"

Edit: This is more of a rant in a markdown file, I can't possibly call this documentation. Forgive me. 
Who would've thought my first real post would be on such an obscure topic?

Anyway... The docker tutorial is fairly hard to follow for people who are running on Windows 10, so I am writing this to address some of the potential issues that may arise when following the documentation on this specific docker tutorial page: [part 6 - using bind mounts](https://docs.docker.com/get-started/06_bind_mounts/). Think of it as an extended documentation page for Windows 10 users.

Turns out, a TON of people are having issues with this part of the tutorial. Here's how I know this:

This is what the average page looks like on the tutorial documentation. Easy to understand, and I love the transparency of this.
![This is what the average page in the tutorial documentation looks like](https://github.com/MSoup/randomNotes/blob/main/images/bind-mounts/Persisting%20data.JPG)

Oh...
![And this is actually what the Bint Mounts page looks like. Yikes!](https://github.com/MSoup/randomNotes/blob/main/images/bind-mounts/Mount.JPG)

I imagine the issues stem from a combination of two things:

1. The documentation is not all encompassing for different OS users
2. The documentation is too concise and assumes some things that the user may not be aware of.
3. <s>Maybe only I got stuck and I'm a baddie</s>

If you wish to follow along with my exact setup, you will need:

- The docker getting-started repo (available [here](https://github.com/docker/getting-started/)) 

- VSCode Bash (see [here](https://code.visualstudio.com/docs/editor/integrated-terminal))

- Windows 10 64-bit: Pro, Enterprise, or Education (Build 17134 or higher).

  For Windows 10 Home, see [System requirements for WSL 2 backend](https://docs.docker.com/docker-for-windows/install/#system-requirements-for-wsl-2-backend).

- [Hyper-V and Containers](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v) Windows features must be enabled (might be enabled by default, but please double check). 

Hopefully this can be of use to future readers. Let's follow along the documentation for now. I'll elaborate when I can to make the documentation even just a little bit clearer.

### Step 1: Make sure no other getting-started containers are running

There are two ways to check:

- "Container list" in Docker Desktop

  - If there's no green icon, you're probably in the clear!
  - If there is a green icon (that says "running"), you'll want to remove it. Click the trash bin icon to the right of the list.

- The following command

  ```bash
   docker ps
  ```

  - If this is empty, that means there aren't containers running.
  - If there is a getting-started container running, you want to [remove it](https://docs.docker.com/engine/reference/commandline/rm/#examples).



### Step 2: Run the following command

```bash
 docker run -dp 3000:3000 \
     -w /app -v "$(pwd):/app" \
     node:12-alpine \
     sh -c "yarn install && yarn run dev"
```

If it works for you, congrats! As far as I am aware, most people running Windows 10 get stuck here. Did you?

Here's the first issue I encountered:

```bash
docker: Error response from daemon: the working directory 'C:/Program Files/Git/app' is invalid, it needs to be an absolute path.
See 'docker run --help'.
```

With a bit of research, you'll see that $(pwd) is resolving to a location where your app isn't. 

If you are getting the same error, type in your git-bash

```bash
pwd
```

Is your getting-started/app located in C:/Program Files/Git/app? I bet it's different!

Fine, let's make it an absolute path.

```bash
docker run -dp 3000:3000 -w /app -v "/c/Users/charm/Desktop/getting-started:/app" node:12-alpine sh -c "yarn install && yarn run dev"
```

```bash
docker: Error response from daemon: the working directory 'C:/Program Files/Git/app' is invalid, it needs to be an absolute path.
```

Oh? But didn't I just specify an absolute path? Let's scan through the documentation a little deeper. With some further reading, we can begin to understand the cryptic commands we just typed in:

```bash
-dp 3000:3000 
// Run in detached (background) mode and create a port mapping

-w /app 
// Sets the “working directory” or the current directory that the command will run from

-v "$(pwd):/app"
// bind mount the current directory from the host in the container into the /app directory

node:12-alpine 
// the image to use. Note that this is the base image for our app from the Dockerfile

sh -c "yarn install && yarn run dev" 
// the command. We’re starting a shell using sh (alpine doesn’t have bash) and running yarn install to install all dependencies and then running yarn run dev. If we look in the package.json, we’ll see that the dev script is starting nodemon.
```

Oops, I messed up. The error above says that **the working directory 'C:/Program Files/Git/app' is invalid**, which means there is a problem with the working directory, which corresponds to the -w /app part of the command. 

But hold on, this means that **/app is resolving to the directory to which git bash exists in**! Of course there's no /app directory in there! This is totally not what I meant to do.

It turns out, <s>Windows hates us</s> Windows is trying to be smart by converting your path over to your path relative to where git bash is installed. 

To fix this, add another slash! (as in, //app), and revert the second part because I used the full pathname in the wrong location.

Let's try again:

```
docker run -dp 3000:3000 -w //app -v "$(pwd):/app" node:12-alpine sh -c "yarn install && yarn run dev"
```

And it returns a unique ID! Yay it works!

**Not so fast.**

It's not running when I look in Docker Desktop. In fact, when I click into it, it shows

```
yarn install v1.22.5

info No lockfile found.

[1/4] Resolving packages...

[2/4] Fetching packages...

[3/4] Linking dependencies...

[4/4] Building fresh packages...

success Saved lockfile.

Done in 0.08s.

yarn run v1.22.5

error Couldn't find a package.json file in "/app"

info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

So close, yet so far.

What can we learn from this? 

Apparently there's no package.json file in "/app", but I clearly see it. 

When I type into my console

```
ls app
```

I see

```
Dockerfile    package.json  src
node_modules  spec          yarn.lock
```

There it is! I see package.json! What do they mean by that message?

Back to google, reddit, discord, friends, and stackoverflow...

I stumbled upon [this thread](https://github.com/docker/getting-started/issues/76#issuecomment-811091222) where a lot of others were describing the very same issue. The original poster says that adding the additional step **cd app &&** into the commands fixes the issue. Alright, here we go...

```bash
docker run -dp 3000:3000 -w //app -v "$(pwd):/app" node:12-alpine sh -c "cd app && yarn install && yarn run dev"
```

Which returns...

 ```
sh: cd: line 1: can't cd to app: No such file or directory
 ```

That's strange. If docker can't cd to app, then how was I able to ls app? 

Turns out, <s>I lack a fundamental understanding of what I'm doing</s> I need to be in my /app directory within getting-started. Let's try that

```
$ pwd

/c/Users/charm/OneDrive/Desktop/getting-started/app

$ docker run -dp 3000:3000 -w //app -v "$(pwd):/app" node:12-alpine sh -c "cd app && yarn install && yarn run dev"

11eb513673f938f8b3ba836b708d355eb3bf29bd78455e983455a51b

​```docker
sh: cd: line 1: can't cd to app: No such file or directory
```

The issue was <s>AGAIN Windows hating us</s> that $(pwd) is again, resolving to a location that is different from where we actually are! To fix this, we add an extra / at the very start of the string. The final command looks like this

```bash
​```Run this from your getting-started/app directory

docker run -dp 3000:3000 -w //app -v "/$(pwd):/app" node:12-alpine sh -c "yarn install && yarn run dev"
```

Docker now runs with a green icon, and the innards say:

```
yarn install v1.22.5

[1/4] Resolving packages...

success Already up-to-date.

Done in 0.57s.

yarn run v1.22.5

$ nodemon src/index.js

[nodemon] 1.19.2

[nodemon] to restart at any time, enter `rs`

[nodemon] watching dir(s): *.*

[nodemon] starting `node src/index.js`

Using sqlite database at /etc/todos/todo.db

Listening on port 3000
```

I'm in tears. It has been a full 8 hour workday and some. I just finished two steps of a tutorial, but I figured it out. 
What errors will I run against next? Stay tuned for more.

MSoup.
