## Hearts

[hearts-repo](https://github.com/romainmenke/hearts)

[notifier-repo](https://github.com/romainmenke/universal-notifier)

At work we were talking about pushing bugs to a CI service and how an element of play could be introduced if each repository had an old-school health bar. So I decided to see if I could get it done over the weekend.

I needed three elements to make this work :

 - golang after-step for wercker
 - server listening to the after-step
 - data store to keep score


 ### Day One :

Apparently nobody writes steps in Go for Wercker although the documentation specifies that it is possible. The only clue I got was that I needed to deploy my Go code as a binary.

I started with a simple `fmt.Print("Hello World")` and tried to get Wercker to log that. First I needed to compile for linux since Wercker runs Docker containers.

teaching Go how to cross-compile :

```bash
brew install go --with-cc-common
```

or :

```bash
brew reinstall go --with-cc-common
```

Then I needed to know that exact architecture to be able to compile correctly, so I pushed a simple bash script that did nothing other than :

```bash
 echo uname -a
```

This told me that wercker containers ran on `amd64`.

My go build command now became :

```bash
GOOS=linux GOARCH=amd64 go build .
```

Wercker steps require a `run.sh` script as an entrypoint.
There is no way around it, but all it has to do is start up your go executable.
To get the path to your go executable use the `$WERCKER_STEP_ROOT` env variable.

```bash
#!/bin/bash

$WERCKER_STEP_ROOT/universal-notifier
```

Another requirement is the `wercker-step.yml`.
Versioning needs to be bumped for each deployment to the step registry.
The properties correspond to variables passed to this step when you use it in a `wercker.yml`.

```yml
name: your-step-name
version: 0.1.0
description: Your Awesome Step
keywords:
  - cool
  - handy
properties:
  var1:
    type: string
    required: true
  var2:
    type: string
    required: true
```

Finally add a `wercker.yml`.
This will them Wercker that you are pushing a step.

```yml
box: wercker/default
build:
  steps:
    - validate-wercker-step
```


After all this I got Wercker to say `Hello World` from an after-step.
Now it was time to gather as much info from Wercker as possible so I listed all the environment variables available to after-steps and wrote a `.proto` file as I would be using gRPC.

The after-step would act as a gRPC Client and send the Wercker data to a gRPC Server.

The after-step takes two variables :
 - port : the gRPC server port
 - host : the gRPC server host

 As it just sends all Wercker data to a variable service it can be easily used for many different purposes.

### Day 2:

The idea behind Hearts is that each failing push to Wercker makes the repository lose a life/heart. Each repository starts with three. If a repository was broken but is now fixed a life is added. So I needed to keep track of current life and if the previous build was a pass or fail. I also wanted to keep track of a user's performance. Each push to wercker add's one exp and each repository that drops to zero hearts adds to a deaths counter.

I did not want to add the complexity of a real database implementation so I decided to simply use Git. This would give me the benefit of only needed to service write ops. Reads would go straight to github. Not at all advisable but it would do for now. I wrote a simple scheme for a User and a Heart type. These would be written to .json files and pushed to Git. A shield with the number of hearts would be saved as a .svg file.

The path of the files would be my database structure and would correspond to source control paths. So the shield for `github.com/user/repo` would be `/github.com/user/repo.svg`. After pushing to Git the full path becomes `github.com/romainmenke/hearts/db/hearts/github.com/user/repo.svg`

All this was wrapped in a fakedb package and imported into the Hearts service. Hearts uses gRPC to setup a server and respond to calls from the Wercker step.
To deploy I dockerized the app and pushed to docker-cloud were it is added to an AWS EC2 instance.

All that was left to do was test it. I created another `Hello World` app and added `universal-notifier` to the `wercker.yml` with the ip from the EC2 instance and port from the Hearts app. Then I pushed a working version and saw Hearts come to life in the docker-cloud logs. A broken version quickly uncovered a fatal flaw in my plan. github aggressively caches images (also svg files) so the shield doesn't update for about an hour.

Maybe I'll fix that next week with a real db and and a real api call.
