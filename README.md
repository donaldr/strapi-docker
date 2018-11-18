# [Strapi](https://github.com/strapi/strapi) containerized PLUS

This a fork of [strapi/strapi-docker](https://github.com/strapi/strapi-docker).

The main difference is that it pulls from a docker image that has a local strapi build (i.e. not a package from npm). Also, there is a process for you to create your own strapi build should you need custom functionality. 

In the following instructions in order to prevent confusion for when something is run on your host or in your docker instance I will preface each code block with the comment `# On host` or `# In instance`. 

## Normal Installation 

In most cases you just want to use the Strapi to make your own CMS. Here are instructions for how to do that for Windows, Mac and Linux. 

### Mac

#### Install Docker for Mac

Go to https://store.docker.com/editions/community/docker-ce-desktop-mac and click [Get Docker CE for Mac (Edge)](https://download.docker.com/mac/edge/Docker.dmg). There are certain performance fixes in the edge channel that make it worth it. 

#### Get Source 

Open up a terminal window. If you already had a terminal window make sure to close and re-open it so that the newly installed binaries are accessible. Do the following:

```
# On host
cd /path/to/projects
git clone https://github.com/donaldr/strapi-docker
```

#### Run the docker instance for the first time

```
# On host
docker-compose up --build
```
The --build flag makes sure you get the latest build.

Initially, it is recommended to run without the -d flag that puts the process in the background. In this way you can see the the progress of the build. When the instance is successfully built you'll see something like the following in the terminal:

```
api_1  | [2018-11-15T19:05:56.445Z] info ☄️   Admin panel: http://localhost:1337/admin
api_1  | [2018-11-15T19:05:56.445Z] info ⚡️ Server: http://localhost:1337
```

To stop the docker instances simply Ctrl-C. 

#### Run the docker instance while you're developing

If you've verified that things work you can run the docker instance in the background. No need to build it since the image has already been built.

```
# On host
docker compose up -d
```

To stop the docker instances do:

```
# On host
docker compose down
```

#### Developing on the docker instance

If you need to do something more complex than the admin tool content creator allows you to do then you'll probably need to code. The strapi instance lives in the strapi-app folder. You can modify files directly in this folder on your OS and use your preferred IDE. 

If you need to run scripts (say generate a controller) then you'll need to access a shell within the docker instance. For this you can run the `shell.sh` script:

```
#On host
./shell.sh
```

This script simply finds the container id of the docker instance and then executes the bash shell interactively:

```
#On host
docker exec -it `docker ps --filter name=strapi-docker_api -q` /bin/bash
```

#### Developing Strapi itself

Let's say you want to modify Strapi itself. There are only rare instances where you might want to do that. In a recent case I had to do this. Strapi has an auto-reload feature which restarts the server when modifications are made to the source. However, the functionality only worked for modified files and not for files that were added. So I decided to add this functionality to my fork.

Build the development version

```
# On host
docker-compose -f docker-compose.dev.yml build
```

Run the development version. You can run it in the foreground to make sure there are no errors:

```
# On host
docker-compose -f docker-compose.dev.yml up
```

Or in the background if you've already made sure it runs properly:

```
# On host
docker-compose -f docker-compose.dev.yml up -d
```

Shell into the docker instance...

```
# On host
./shell.sh
```

...and clone strapi into the `/usr/local/src/strapi` directory

```
# In instance
cd /usr/local/src
git clone https://github.com/donaldr/strapi
```

Substitute for wherever your source for strapi is. 

After you've clone strapi you can start by running a build. You can see strapi's documentation for more information on the various ways to develop strapi. I'll just run through the specific case for auto-reload issue mentioned above. 

```
# In instance
npm run setup:build
```

The build process takes about twenty minutes. Grab a coffee. 

The build process creates a number of symlinks that refer to the src that you've built. 

In the case of the auto-reload I first ran a strapi app I had built. Note that this strapi-app will only exist if you've built it as described in the **Run the docker instance for the first time** section.

```
# In instance
cd /usr/src/api/strapi-app
strapi start
```

Then I verified that the functionality I wanted was indeed not present. I went to the admin tool at `http://localhost:1337/admin` and then used the strapi cli tool to generate an api that I arbitrarily named "hello".

```
# In instance
strapi generate:api hello
```

If the auto-reloading worked as we desired there would be a new content type called "Hellos" in the admin tool. I verified this wasn't the case. 

I killed the dev server started, deleted the directory created by the strapi generator:

```
# In instance
cd /usr/src/api/strapi-app
strapi start
rm -rf api/hello
```

Then I modified the file `/usr/local/src/strapi/packages/strapi/bin/strapi-start.js` and made my changes for the auto-reload functionality. 

I then started the dev server again, checked to make sure there was no "hellos" in the admin tool, generated the hello api, and then reloaded page to see if the server had indeed restarted:

```
# In instance
cd /usr/src/api/strapi-app

#Run in background and don't output stdout or stderr, alternatively just run strapi start and open up another shell
strapi start > /dev/null 2>&1 &

#Check admin tool for lack of hellos

strapi generate:api hello

#Reload admin page in browser and check for hellos
```

Success!

Now you'll have to do two things. Ideally, you'll commit and push the source back to your repository so you don't lose the change. 

```
# In instance
cd /usr/local/src/strapi
git add . 
git commit -m "Added ability to reload server in dev mode when files are added"
git push
```

Now, you also want to be able to use your brand new amazing build for when you're using this docker instance during regular development. For that I've created a docker image called strapi-built. At the moment it is hosted on docker hub and I will be the only one able to push changes. The image will eventually be hosted on a private repository hub where multiple team members can commit changes. 

Here's the process for saving this image and pushing it to the docker hub. 

```
#On host

#Login to docker hub using your credentials
docker login 

./save-strapi-built.sh
```

After making a change to the strapi-built image you'll need to run the code in **Run the docker instance for the first time** again. This makes sure to rebuild the docker image with new base image. 
