<div id="top"></div>

# QNAPDocker
How to run Docker on QNAP NAS Devices


## NOTE: As of the recent Container station update, 'docker-compose' command no longer works. You must instead use 'docker compose'

This is a basic guide encompassing most use-cases of Docker on a QNAP NAS device.
Steps are the same for either QTS or QuTS Hero.

# Contents
 - [Pre-Reqs](#Prereqs)
 - [Creating first container](#CreateFirst)
 - [Persistent Storage](#Persistent_storage)
 - [Creating a full stack](#full_stack)
 - [Starting, stopping & updating](#start_stop_update)
 - [Anatomy of a compose file](#anatomy)
 - [Translating to docker-compose](#translating)


<div id="Prereqs"></div>

## Pre-reqs
Ensure Container Station is installed from AppCenter
![install_container_station](https://user-images.githubusercontent.com/600235/193766546-04a5f038-4333-45a8-942f-b61e26c5f635.png)

Optional, but recommended, enable SSH
Control Panel->Telnet/SSH
![enable ssh](https://user-images.githubusercontent.com/600235/193767374-f8b0160b-7b7a-4119-8c72-96b177c855e5.png)
Port 22 is fine

[Back to top](#top)

<div id="createfolder"></div>

Create a shared folder, how you name this is your own preference. This will be used to store your persistent data.
In this example, we shall be using 'docker-test'

![shared-folder1](https://user-images.githubusercontent.com/600235/193768264-a2de4f0c-d9e9-41ff-bb9b-d11822cdfb62.png)

[Back to top](#top)

<div id="CreateFirst"></div>

## Creating first container
Now we're ready to create a Docker container. Let's start with something simple, Openspeedtest.

First, open up Container Station, click the Create tab, then 'Create Application'

![container-station](https://user-images.githubusercontent.com/600235/193771209-b9a262d7-0b02-4043-a53b-0a8d1f71a4b9.png)

You'll be presented with a Create Application window, containing 3 pre-populated lines and an empty Application name field.

![application1](https://user-images.githubusercontent.com/600235/193771363-e38a833f-4823-4bb9-a9e4-f20471b5170b.png)


In Application name, enter openspeedtest, then copy & paste the following into the black portion of the window

    version: '3'
    services:
      openspeedtest:
        container_name: openspeedtest
        image: openspeedtest/latest
        ports:
          - 3000:3000
          - 3001:3001

![application2](https://user-images.githubusercontent.com/600235/193782024-13eabc4d-3bd9-4132-89d4-27c310c8b1ba.png)

Once done, click Validate YAML to ensure the formatting is correct (YAML is fussy about formatting)

![validate](https://user-images.githubusercontent.com/600235/193782653-ea7f72fb-d692-419b-b47d-0872bcdbadc9.png)

If you get a green tick, click apply and wait for the image to download, then the container to spin up.

Once running, you should see the container running in the Overview tab
![running container](https://user-images.githubusercontent.com/600235/193783307-d8042743-e7ff-4e7a-aa6b-48cbc66aa405.png)

Navigating to http://NAS-IP:3000 should show you an OpenSpeedTest page.

[Back to top](#top)

<div id="Persistent_storage"></div>

## Persistant Storage

We've now created a simple container. But what if we want something more advanced? Something that actually stores data?

This is where the [create folder](#createfolder) step comes in.

In this part, we'll be creating a MariaDB instance and storing the data in a folder, so it'll persist when the container is stopped.

First, we'll need to create a directory to contain the data. Go to your shared folder (we used Docker-test in this guide) and create another folder inside of that, name it something descriptive, such as 'database'

![folder create](https://user-images.githubusercontent.com/600235/193791387-67b451fc-e20d-4a2f-9899-321846b4ae6a.png)


As in [Creating first container](#CreateFirst), open up Container Station and hit the create tab, followed by Create Application.

Give it a name, for example database, then paste the following code, being sure to change docker-test to whatever you named your persistent docker folder:

    version: '2.4'
    services:
      database:
        image: lscr.io/linuxserver/mariadb
        container_name: database
        restart: unless-stopped
        ports:
          - 3306:3306
        environment:
          - MYSQL_ROOT_PASSWORD=kdFkdQZkhxy8n4dJbPA
          - TZ=Europe/London
        volumes:
          - /share/docker-test/database:/config

After a short wait whilst container station download the image and runs it for the first time, you should see some files appear in your persistent folder.


![running files](https://user-images.githubusercontent.com/600235/193793253-568e57df-33f9-4b41-b136-8e47267aa6fc.png)



[Back to top](#top)

<div id="full_stack"></div>

## Creating a full stack

Now we've gotten a container running, with persistent storage, let's tie all this together by adding another container into the same docker file.
This container will not start until the database is up & running, it will also talk directly to the database.

Open up Container Station, go to the Overview tab and click the edit button to the right of the 'database' entry

![edit stack](https://user-images.githubusercontent.com/600235/193794032-f02ebf3d-0ae4-49ae-a2a5-988d4e74f5bc.png)

We'll be adding PHPMyadmin to this stack so we can view, edit & delete databases.

In the edit window, add the following:

      phpadmin:
        image: phpmyadmin
        container_name: phpadmin
        restart: unless-stopped
        depends_on:
          - database
        ports:
          - 88:80
        environment:
          - PMA_HOST=database
          - PMA_PORT=3306
          - MYSQL_USER=root
          - MYSQL_PASSWORD=kdFkdQZkhxy8n4dJbPA
          
Your full stack should look like this:

    version: '2.4'
    services:
      database:
        image: lscr.io/linuxserver/mariadb
        container_name: database
        restart: unless-stopped
        ports:
          - 3306:3306
        environment:
          - MYSQL_ROOT_PASSWORD=kdFkdQZkhxy8n4dJbPA
          - TZ=Europe/London
        volumes:
          - /share/docker-test/database:/config
      phpadmin:
        image: phpmyadmin
        container_name: phpadmin
        restart: unless-stopped
        depends_on:
          - database
        ports:
          - 88:80
        environment:
          - PMA_HOST=database
          - PMA_PORT=3306
          - MYSQL_USER=root
          - MYSQL_PASSWORD=kdFkdQZkhxy8n4dJbPA

Once running, container station should show 2 containers running under the database stack

![stack running](https://user-images.githubusercontent.com/600235/193795748-dfe77949-ac48-40fb-a32a-e97797cd1cba.png)

Now, if you go to http://NAS-IP:88 you should be prompted with a logon page. Try it!
Enter root as the username and kdFkdQZkhxy8n4dJbPA as the password.


![phpmyadmin](https://user-images.githubusercontent.com/600235/193796251-3756f356-2a3a-4f88-a356-6ac14ca46326.png)


[Back to top](#top)

<div id="start_stop_update"></div>

## Starting, Stopping & Updating

This is where SSH access comes in handy.

Connect to your NAS using ssh username@IP replacing username with your username & IP with your NAS' IP or hostname.

Once in, there's several commands we can use.

     docker container ls
 
 Will show all running containers.
 
    docker image ls
    
 Will show all images saved locally.
 
 This is all very nice, however we want to learn how to start, stop & update.
 
 There's 2 ways of starting & stopping. You can do it via the Container Station GUI using the stop square or the play triangle. This is self-explanitory.
 The other way is to do this via the command line.
 
 First, you must navigate to your container station directoy. By default, this is
     /share/Container/container-station-data/application
 Once in there, typing ls will show each stack in its own folder, inside each folder is a docker-compose.yml file. In this example, we want to stop the database stack, so first we must type the following command:
 
   cd /share/Container/container-station-data/application/database
 
if you typed ls at this point, you'll see 2 files. docker-compose.yml & qnap.json. It's worth pointing out that if the docker-compose.yml file is edited outside of container station, it'll be marked as invalid and can only be edited externally from that point on.

Once in the desired directory, simply type

    docker-compose down
    
 To bring down the container stack. You can inspect Container Station to check if this has indeed stopped.
 
 If you type
 
     docker-compose up -d
     
 The container stack will start, then return you to a command prompt.
 
 If you type
 
     docker-compose pull
     
  Docker compose will grab the latest versions of each image in the stack. However, it won't restart the stack with the newest version. So, to do a complete update and restart the stack to ensure it's running on the latest version, type the following
  
      docker-compose pull && docker-compose down && docker-compose up -d
      
  That line will, in this order: pull the latest image version, stop the stack, start the stack on the latest version.
  
  If you've pulled a new version of an image, the docker image ls command will show you more than one version of that image. How do you clean up after yourself? Well that's simple. Simply type the following command
  
      docker image prune --all
      
  And type y when prompted. That will remove any images that are not attached to a running container.
  
  [Back to top](#top)
  
  <div id="anatomy"></div>
  
  ## Anatomy of a compose file
  
  It's worth showing how a docker-compose.yml file is formatted and what each part does.
  
  Firstly, Yaml is quite fussy in terms of formatting. Line indents must be spaces, not tabs. Certain parts of the file must be indented from others to form an array.
  
  Let's examine our database container:
   
    version: '2.4'
    services:
      database:
        image: lscr.io/linuxserver/mariadb
        container_name: database
        restart: unless-stopped
        ports:
          - 3306:3306
        environment:
          - MYSQL_ROOT_PASSWORD=kdFkdQZkhxy8n4dJbPA
          - TZ=Europe/London
        volumes:
          - /share/docker-test/database:/config
          
You'll notice database: is indented 2 spaces in from services. Then each argument for the container is indented a further 2 spaces. Everything else is on the same indent level. However, when we define arrays, such as ports, environment variables & volumes, they're indented another 2 spaces, then start with an -.
It's important to get this right, otherwise docker-compose will fail.

Under ports & volumes, you can see colons between the characters. This is more obvious in the ports array. It's actually quite simple. Left of the colon (:) is the host (in this case, the QNAP NAS) and to the right, is inside the container. So, 3306:3306 means that, when you hit port 3306 on the NAS, so http://NAS-IP:3306 you will be redirected to port 3306 inside the container. The left side of this can be modified. So, say for example you already had a service listening on port 3306, you could change this to read 3307:3306. That would mean you go to http://NAS-IP:3307 to achieve the same result.
Typically, for ports, you only change the number on the left side of the colon.

Volumes is very similar. Left of the colon refers to the host (NAS) and right refers to the container. So, what this does is present everything in /share/docker-test/database to /config inside the database container.
Similar to ports, most of the time you only change the entry to the left of the colon. However, exceptions can be made. Plex for example, you may want to add multiple folders.



[Back to top](#top)


<div id="translating"></div>

## Translating to docker-compose

Quite often, a site, or a projects documentation will only show the docker command to run to spin up a service and not a docker-compose.yml file. This can normally be translated quite easily (it gets even easier with experience)

Going back to our first container example, let's take a look at OpenSpeedTest.
Their [documentation](https://openspeedtest.com/speed-testing-application-for-your-website.php) shows the following

    docker run --restart=unless-stopped --name openspeedtest -d -p 3000:3000 -p 3001:3001 openspeedtest/latest
    
If we take a look at this, we can break it down. It's creating a container with the name 'openspeedtest' presenting ports 3000 & 3001 and passing them internally to the same ports. Finally, it's pulling the openspeedtest/latest image.


If we take a look at our first example, we can see how it can be translated into a compose file


    version: '3'
    services:
      openspeedtest:
        container_name: openspeedtest
        image: openspeedtest/latest
        ports:
          - 3000:3000
          - 3001:3001

[Back to top](#top)  
     














