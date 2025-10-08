# Nginx Proxy Manager (NPM) + Crowdsec -(minus) Authentik

> [!WARNING]
> The NPM fork used by this repository is currently (indefinitely) **unmaintained** - as is this project.
> While I can confirm it still works, **I'd advise against using it** - *until I have time to update it*.
This guide is an alteration of the [main guide](https://github.com/suckharder/NPM-Crowdsec-Authentik-Stack/blob/main/README.md) - it aims the remove the Authentik component from the stack, keeping only NPM and Crowdsec.

> **At this time - I mostly just removed a lot of the steps, and this currently stands untested - however, just omitting Authentik is very straightforward so it should work fine.**

**[NPM](https://nginxproxymanager.com/guide/)** is an easy-to-use reverse proxy built on Nginx, managed via a GUI. In this case we will be securing it with **[Crowdsec](https://docs.crowdsec.net/)** - to automatically ban malicious IPs.

>We will be using a [fork of NPM](https://github.com/LePresidente/docker-nginx-proxy-manager) that comes pre-packaged with a Crowdsec bouncer.

### The thought proccess:
- **1 ->** A user wants to access a protected service
- **2 ->** The user makes a request to the proxy (NPM)
- **3 ->** The proxy consults Crowdsec and serves the request (or Crowdsec bans the user *[403 response]*)

We'll be setting this up in **Docker** using ***Docker Compose***. I also included a *test container* for demonstration - we will be placing it inside an internal network, hidden behind the NPM reverse proxy, and protected by Crowdsec.
We will also **TEST** if everything is working order :)

***Bonus : You'll get a nice dashboard to monitor all the blocked threats :)***

## IMPORTANT: Limitations

See the [main guide](https://github.com/suckharder/NPM-Crowdsec-Authentik-Stack/blob/main/README.md), except you're losing authetication...

I offer **ABSOLUTELY ZERO WARRANTY**, and take **ABSOLUTELY NO RESPONSIBILTY** if you compromise your security with this setup.

## Requirements

- **GNU/Linux** *(please don't ask me how to deploy this on Windows I have no idea)*
- **Git**
- **Sudo**
- *(a working)* **Docker** Installation
- a **text editor** you like *(I use nano)*

>This guide will not be covering the absolute basics, i.e. I assume you can set-up your domain and point it where you want, or set-up your SSL/TLS certs.

# HOW TO: GUIDE

>On my systems, I usually just make a user, give the user sudo priviliges and then do everything as that user, adjust stuff accordingly if needed

1. First, **clone my repo**

`git clone https://github.com/suckharder/NPM-Crowdsec-Authentik-Stack`

2. *This is almost definitely unnecessary, but idk*

`sudo chown -R 1000:1000 NPM-Crowdsec-Authentik-Stack`

3. **Get inside the directory**

`cd NPM-Crowdsec-Authentik-Stack`

4. **MODIFY docker-compose.yml, .env, and delete the authentik-certs, authentik-custom-templates, authentik-media folders**

**Remove** lines shown below from ***docker-compose.yml***

```
##############_____Authentik_Postgresql_____###########################

  postgresql:
    image: docker.io/library/postgres:16-alpine
    container_name: authentik-postgresql
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
    volumes:
      - database:/var/lib/postgresql/data
    networks:
      - internal_network
    environment:
      # Set in .env
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      # Set in .env
      POSTGRES_USER: ${POSTGRES_USER}
      # Set in .env
      POSTGRES_DB: ${POSTGRES_DB}

##############_____Authentik_Redis_____################################

  redis:
    image: docker.io/library/redis:alpine
    container_name: authentik-redis
    command: --save 60 1 --loglevel warning
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping | grep PONG"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 3s
    networks:
      - internal_network
    volumes:
      - redis:/data

##############_____Authentik_Server_____###############################

  server:
    # Check Authentik Docs for recommended pinned version
    image: ghcr.io/goauthentik/server:2025.4.1
    container_name: authentik-server
    restart: unless-stopped
    command: server
    environment:
      # Set in .env
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY}
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      # Set in .env
      AUTHENTIK_POSTGRESQL__USER: ${POSTGRES_USER}
      # Set in .env
      AUTHENTIK_POSTGRESQL__NAME: ${POSTGRES_DB}
      # Set in .env
      AUTHENTIK_POSTGRESQL__PASSWORD: ${POSTGRES_PASSWORD}
      AUTHENTIK_ERROR_REPORTING__ENABLED: true
    volumes:
      - ./authentik-media:/media
      - ./authentik-custom-templates:/templates
    networks:
      npm:
      internal_network:
# Leave ports unpublished, access through reverse proxy. Use the commented-out ports for reference. Use 9000 for outpost.
### ports:
   #   - "9000:9000"
   #   - "9443:9443"
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy

##############_____Authentik_worker_____###############################

  worker:
    # USE THE SAME IMAGE AS SERVER!!!
    image: ghcr.io/goauthentik/server:2025.4.1
    container_name: authentik-worker
    restart: unless-stopped
    command: worker
    environment:
      # Set in .env
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY}
      AUTHENTIK_REDIS__HOST: redis
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      # Set in .env
      AUTHENTIK_POSTGRESQL__USER: ${POSTGRES_USER}
      # Set in .env
      AUTHENTIK_POSTGRESQL__NAME: ${POSTGRES_DB}
      # Set in .env
      AUTHENTIK_POSTGRESQL__PASSWORD: ${POSTGRES_PASSWORD}
      AUTHENTIK_ERROR_REPORTING__ENABLED: true
    user: root
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./authentik-media:/media
      - ./authentik-certs:/certs
      - ./authentik-custom-templates:/templates
    networks:
      internal_network:
    depends_on:
      postgresql:
        condition: service_healthy
      redis:
        condition: service_healthy
        

        
        
... (in the volumes section) ...

# authentik postgresql
  database:
    driver: local
# authentik redis
  redis:
    driver: local
```

**Remove** lines shown below from ***.env***

```
POSTGRES_PASSWORD=
POSTGRES_USER=
POSTGRES_DB=
AUTHENTIK_SECRET_KEY=
```

5. **Generate a root password for NPM's database**

`tr -dc A-Za-z0-9 </dev/urandom | head -c 32`

6. **Paste it into the .env file** as ROOT_DATABASE_PASSWORD=, so it looks like

```
DATABASE_PASSWORD=
ROOT_DATABASE_PASSWORD=yourpasswordhere
CROWDSEC_BOUNCER_APIKEY=
```

7. **Generate a password for NPM's database**

`tr -dc A-Za-z0-9 </dev/urandom | head -c 32`

8. **Paste it into the .env file** as DATABASE_PASSWORD=, so it looks like

```
DATABASE_PASSWORD=yourpasswordhere
ROOT_DATABASE_PASSWORD=yourpasswordhere
CROWDSEC_BOUNCER_APIKEY=
```

>*The bouncer API key will come later.*

13. Now, let's **compose** what we have so far

>I **STRONGLY** recommend reading the provided ***docker-compose.yaml*** file. I even included a few comments so it's *more easily* understandable.

`sudo docker compose up -d`

14. When all is done, we need to **add the Crowdsec NPM bouncer and get our API key**

`sudo docker compose exec crowdsec cscli bouncer add npm-bouncer`

> **DO NOT lose this key!** It's important.

15. We can **stop the stack** now

`sudo docker compose down`

16. **Paste the key into the .env file** as CROWDSEC_BOUNCER_APIKEY=, so it looks like

```
DATABASE_PASSWORD=yourpasswordhere
ROOT_DATABASE_PASSWORD=yourpasswordhere
CROWDSEC_BOUNCER_APIKEY=yourbouncerapikey
```

17. Everything Docker-related should be done now, we can **compose again**

`sudo docker compose up -d`

> If **Docker complains** about Crowdsec's *acquis.yaml* file, you need to remove the crowdsec volume and compose again ***sudo docker volume rm your_volume_name***

> Also, I advise that you check out the acquis.yaml file as well

18. **Set up NPM.** Default username:`admin@example.com` , default password: `changeme`

> This is where you might want to set up you domain and/or your certs as well.

> I **WOULDN'T** expose port 81 *(NPMs UI)* publically at all. If you're setting up a remote machine use an SSH tunnel or something. Ex.:  *ssh -L 9999:localhost:81 remoteuser@server-ip -p your-ssh-port*

19. **Make a proxy host in NPM pointing to the** ***hello-test*** **container**

> Use hello-test as the 'forward hostname' and port 80.

>Yes, we use Docker's internal DNS, so maybe don't screw around with the container names :)

20. **Verify you can reach hello-test.** *hello-test* should successfully open.

> At this point, unless you somehow magically got banned by Crowdsec already, you should be able to get to *hello-test* through the proxy.

21. **You're done. Congratz.**

> I **STRONGLY** recommend ***testing*** (next section).

22. **BONUS:** Connect your Crowdsec to a dashboard. Register at https://app.crowdsec.net/, choose *Add Security Engine*, and copy ***ONLY*** the key. Then run

`sudo docker exec crowdsec cscli console enroll -e context yourkeyhere`

Afterwards just approve the enroll request in the dashboard :)

# Testing

> **DO NOT TEST FROM THE SAME MACHINE RUNNING THE STACK** - it'll always work :)

> I removed the "follow redirects" curl commands, we shouldn't need them anymore.

1. **Open hello-test in a web browser**

> If you dont get to hello-test's page, something is wrong.

2. **Run this command**

`curl -I https://hello-test-url-here`

> This time the response should be *200* since we're not redirecting for authetication.

3. **Check your external/public IP address**

4. **Manually ban your IP in Crowdsec**

`sudo docker exec crowdsec cscli decisions add -i your-ip-here`

5. **Verify your IP is banned**

`sudo docker exec crowdsec cscli decisions list`

6.  **Open hello-test in a web browser**

> If you don't see a *"Blocked by Crowdsec"* screen something is **VERY** wrong.

7. **Run this command**

`curl -I https://hello-test-url-here`

> Since we're not using Authentik, the response should immediately be *403* - if it's not, something is **VERY** wrong.

> To unban yourself use ***sudo docker exec crowdsec cscli decisions delete -i your-ip-here***

## Issues

I just quickly went through the main guide and removed Authentik - this is not tested but should work fine. Feel free to open an **Issue**

## Special thanks

- *see the* [*main guide*](https://github.com/suckharder/NPM-Crowdsec-Authentik-Stack/blob/main/README.md)
