# Nginx Proxy Manager (NPM) + Crowdsec + Authentik

> [!WARNING]
> The NPM fork used by this repository is currently (indefinitely) **unmaintained** - as is this project.
> While I can confirm it still works, **I'd advise against using it** - *until I have time to update it*.
Are you looking to **protect your services** using *only* simple and easily manageable solutions?
This repo (*hopefully*) does exactly that. **[NPM](https://nginxproxymanager.com/guide/)** is an easy-to-use reverse proxy built on Nginx, managed via a GUI. In this case we will be securing it with **[Crowdsec](https://docs.crowdsec.net/)** - to automatically ban malicious IPs, and **[Authentik](https://docs.goauthentik.io/docs/)** - a feature rich self-hosted Auth solution.

>We will be using a [fork of NPM](https://github.com/LePresidente/docker-nginx-proxy-manager) that comes pre-packaged with a Crowdsec bouncer.

### The thought proccess:
- **1 ->** A user wants to access a protected service
- **2 ->** The user makes a request to the proxy (NPM)
- **3 ->** The proxy consults Crowdsec and serves the request (or bans the user *[403 response]*)
- **4 ->** The user first arrives at an authentication page instead of the service ([forward proxy authentication](https://docs.goauthentik.io/docs/add-secure-apps/providers/proxy/forward_auth))
- **5 ->** If the user successfully finishes the authentication, the proxy receives an OK response from Authentik and actually forwards the user to the service

We'll be setting this up in **Docker** using ***Docker Compose***. I also included a *test container* for demonstration - we will be placing it inside an internal network, hidden behind the NPM reverse proxy, protected by Crowdsec, and we will also secure it with Authentik - even though the test container does *NOT* support any form of authentication by default.
We will also **TEST** if everything is working order :)

***Bonus : You'll get a nice dashboard to monitor all the blocked threats :)***

## IMPORTANT: Limitations

This stack is **NOT a silver bullet** when it comes to security. It should be used in combination with many other practices *(such as network segmentation, correctly configured firewalls, proper maintenance etc.)* and ideally behind something like [Cloudflare Tunnels](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) or a different "external reverse proxy" ([Pangolin](https://github.com/fosrl/pangolin) is being *heavily* astroturfed on Reddit at the moment, I haven't looked into it but that should be a *good example*).

You ***can*** use just this setup to expose your services publically *(in fact, according to my testing it seems to work pretty well)* - but I **wouldn't recommend it**.  But if you like to live on the edge - hey, you do you.

Maybe if you're ***renting a VPS*** that you don't care about as much - in that case, it should be a *decent security solution*.
>**On-prem**, I'd only use this as a ***"middle-step"*** between an "external reverse proxy" (see above) and your services, or between a VPN and your services. Or just to ***secure your services on an internal network*** - if you don't need external/public access at all.

Of course, all of the above **ONLY applies** assuming that that everything is ***properely deployed and configured*** *(which it likely isn't - given the fact I made it)*, as well as ***properely maintained***.

I offer **ABSOLUTELY ZERO WARRANTY**, and take **ABSOLUTELY NO RESPONSIBILTY** if you compromise your security with this setup.
> If you actually want this setup to be properly done, and wish to learn more, you should realistically be reading the ***official documentation*** for each of the services used, not my GitHub :)

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

4. **Generate a root password for NPM's database**

`tr -dc A-Za-z0-9 </dev/urandom | head -c 32`

5. **Paste it into the .env file** as ROOT_DATABASE_PASSWORD=, so it looks like

```
DATABASE_PASSWORD=
ROOT_DATABASE_PASSWORD=yourpasswordhere
CROWDSEC_BOUNCER_APIKEY=
POSTGRES_PASSWORD=
POSTGRES_USER=
POSTGRES_DB=
AUTHENTIK_SECRET_KEY=
```

6. **Generate a password for NPM's database**

`tr -dc A-Za-z0-9 </dev/urandom | head -c 32`

7. **Paste it into the .env file** as DATABASE_PASSWORD=, so it looks like

```
DATABASE_PASSWORD=yourpasswordhere
ROOT_DATABASE_PASSWORD=yourpasswordhere
CROWDSEC_BOUNCER_APIKEY=
POSTGRES_PASSWORD=
POSTGRES_USER=
POSTGRES_DB=
AUTHENTIK_SECRET_KEY=
```

>*Even though we will be setting up Authentik last, we'll fill it's variables now, so that we can compose. The bouncer API key will come later.*

8. **Generate a Postgresql password for Authentik**

`openssl rand -base64 36`

9. **Paste it into the .env file** as POSTGRES_PASSWORD=, so it looks like

```
DATABASE_PASSWORD=yourpasswordhere
ROOT_DATABASE_PASSWORD=yourpasswordhere
CROWDSEC_BOUNCER_APIKEY=
POSTGRES_PASSWORD=yourpasswordhere
POSTGRES_USER=
POSTGRES_DB=
AUTHENTIK_SECRET_KEY=
```

10. **Choose** whatever **DB name and USER** name you want, or generate these as well

```
DATABASE_PASSWORD=yourpasswordhere
ROOT_DATABASE_PASSWORD=yourpasswordhere
CROWDSEC_BOUNCER_APIKEY=
POSTGRES_PASSWORD=yourpasswordhere
POSTGRES_USER=authentik_db_user
POSTGRES_DB=authentik_db
AUTHENTIK_SECRET_KEY=
```

11. **Generate the Authentik secret key**

`openssl rand -base64 60`

12. **Paste it into the .env file** as AUTHENTIK_SECRET_KEY=, so it looks like

```
DATABASE_PASSWORD=yourpasswordhere
ROOT_DATABASE_PASSWORD=yourpasswordhere
CROWDSEC_BOUNCER_APIKEY=
POSTGRES_PASSWORD=yourpasswordhere
POSTGRES_USER=authentik_db_user
POSTGRES_DB=authentik_db
AUTHENTIK_SECRET_KEY=yoursecretkey
```

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
POSTGRES_PASSWORD=yourpasswordhere
POSTGRES_USER=authentik_db_user
POSTGRES_DB=authentik_db
AUTHENTIK_SECRET_KEY=yoursecretkey
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

20. **Verify you can reach hello-test.** *hello-test* should successfully open, as we haven't set-up Authentik yet

> At this point, unless you somehow magically got banned by Crowdsec already, you should be able to get to *hello-test* through the proxy.

21. If everything works so far, **make a proxy host pointing to** ***authentik-server***

> hostname: authentik-server, port: 9443 (SSL port - use scheme: https)

> Authentik Server *uses* **WEBSOCKETS - SET TO ON**

22. Set up **Authentik**

>*Honestly*, just watch this video by **Christian Lempa:** ***https://youtu.be/N5unsATNpJk***, *skip the configuration, as he does not use NPM nor Crowdsec*. Create a new admin user like he does, give the user MFA, disable the default one.

>*The part starting at* ***33:32*** *is also going to be relevant in a moment*, you can watch that too, but we'll be making adjustments for NPM - **we will not even be touching YAML again**.

23. **Create a new provider in Authentik**. Choose ***proxy provider***
> Give it a name - for example *nginx-hello-test*. Click on **FORWARD AUTH (SINGLE APPLICATION) -> enter the URL of** ***hello-test***. Click *Finish*.

24. **Create a new application in Authentik**
> Give it a name - for example *nginx-hello-test* again. For slug, enter *hello-test*.
> Choose the provider you just created. Click *Finish*.

25. **Modify Authentik's Embedded Outpost.** Just **move the application** you made for *hello-test* **to the right side and hit** ***Update***.

26. ***IMPORTANT:*** **Open the provider you created in Authentik, and copy the Nginx (Proxy Manager) config.**

27. **Open the hello-test proxy host in NPM, click Advanced and paste the config.**

> **DO NOT SAVE YET!**

28.  **Modify proxy_pass with authentik-server** *(again with the Docker DNS)*

`proxy_pass              http://authentik-server:9000/outpost.goauthentik.io;`

> Keep port 9000.

30. **You're done. Congratz.**

> Once you log out of Authentik/clear cookies, *you should be greeted with an authentication page* upon visiting *hello-test*.

> I **STRONGLY** recommend ***testing*** (next section).

31. **BONUS:** Connect your Crowdsec to a dashboard. Register at https://app.crowdsec.net/, choose *Add Security Engine*, and copy ***ONLY*** the key. Then run

`sudo docker exec crowdsec cscli console enroll -e context yourkeyhere`

Afterwards just approve the enroll request in the dashboard :)

# Testing

> **DO NOT TEST FROM THE SAME MACHINE RUNNING THE STACK** - it'll always work :)

1. **Open hello-test in a web browser**

> If you dont get to an Auth page, something is wrong.

2. **Run this command**

`curl -I https://hello-test-url-here`

> If you don't get a *302* response, something is wrong.

3. **Run this command**

`curl -i -L https://hello-test-url-here`

> If the very last response is not a *200*, something is wrong.

4. **Check your external/public IP address**

5. **Manually ban your IP in Crowdsec**

`sudo docker exec crowdsec cscli decisions add -i your-ip-here`

6. **Verify your IP is banned**

`sudo docker exec crowdsec cscli decisions list`

7.  **Open hello-test in a web browser**

> If you don't see a *"Blocked by Crowdsec"* screen something is **VERY** wrong.

8. **Run this command**

`curl -I https://hello-test-url-here`

> **If the response is still 302 DO NOT PANIC! - in a properely working setup, the code here SHOULD still be 302.** *Due to the way our set-up works (forward proxy authentication)* ***even banned IPs*** *should get a *302* here*, because NPM wants you to authenticate - which you won't be able to anyway - *see next step*.

9. **Run this command**

`curl -i -L https://hello-test-url-here`

> You probably get it now :)

> In other words, if the very last response is not *403* something is **VERY** wrong.

> To unban yourself use ***sudo docker exec crowdsec cscli decisions delete -i your-ip-here***

## Issues

I actually successfully deployed this exact set-up, but it is possible I screwed something up, or forgot something when making the repo. Feel free to open an **Issue**.

Also feel free to open an issue if you *believe something about this set-up is wrong or misconfigured*.

## Special thanks

- **Crowdsec** for their [example-docker-compose](https://github.com/crowdsecurity/example-docker-compose), which served as the basis for this whole stack
- Crowdsec again, for their [documentation](https://docs.crowdsec.net/) *(and awesome software DUH)*
- **LePresidente** for his [NPM fork](https://github.com/LePresidente/docker-nginx-proxy-manager) *(I hope he keeps maintaining it)*
- **Authentik**, for their [docs](https://docs.goauthentik.io/docs/), and of course their software
- **Christian Lempa**, for introducing me to Authentik *(via his [YouTube channel](https://www.youtube.com/channel/UCZNhwA1B5YqiY1nLzmM0ZRg))*
- **IBRACORP**, for their [documentation](https://docs.ibracorp.io/ibracorp)
- *And many more...*
