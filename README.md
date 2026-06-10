# Tailscale Reverse Proxy
This is an example configuration for setting up a reverse proxy running on a Tailscale node in Docker Compose. The benefits of doing this are:
- You don't need to forward ports on your router and nothing you host this way is publicly routable
- You can protect individual services with VoidAuth and share the Tailscale node as you wish
- Your domains/subdomains remain consistent regardless of whether you're connected to your home network or any other
- Because no ports are mapped from containers onto the host, not even devices on your LAN can access your services unless you connect them to Tailscale (buy all the sketchy IoT stuff you want)

For that last item, note that *all* port mappings in this compose file are commented out, if they're present at all. You do not need the port mappings. Tailscale handles this. They're there for reference only. You can delete the comments, but I like having them. 

## General prep
This compose file **uses Cloudflare as a provider**. It would need modifications to work with others, but the gist of it should be the same.  Gather the following:
- Cloudflare DNS zone edit token
- Tailscale auth key, pre-approved is nice
- The GID that Docker runs as on your machine

To get the Docker GID, run ``grep docker /etc/group`` which will print something like ``docker:x:968:username``. In this case, the GID is 968. 

Populate these in the .env file along with the other items there. ``MY_DOMAIN`` is the root domain you're using, ``PROXY_CFG_DIR`` is where the persistent storage for the stack lives, adjust these to your needs. The Void keys and password can be any sufficiently long random strings, same for the Dozzle key; use ``openssl rand -hex 64 `` to generate them. Note, Dozzle is only here to show an example of how to set up the proxy and to confirm it's working. You can remove it if you like. 

In the main compose file, you may also want to change the Tailscale hostname defined in ``TS_HOSTNAME``. This is the name the container will get in your Tailnet. 

Finally, ensure you create `servicenet` by running `docker network create servicenet` or whatever other network name you decide to use. If you want to use a different name, ensure you update the compose file to reflect this. Find and replace is your friend here. 

## Traefik
This set up uses labels to tell Traefik what to proxy and how.  Traefik polls the Docker socket via the proxy included in this compose file to get the labels. This is the label block from Dozzle in the provided yaml:

```
labels:
	- "traefik.enable=true"
	- "traefik.http.routers.dozzle.entrypoints=websecure"
	- "traefik.http.routers.dozzle.rule=Host(`dozzle.${MY_DOMAIN}`)"
	- "traefik.docker.network=servicenet"
	- "traefik.http.services.dozzle.loadbalancer.server.port=8080"
	- "traefik.http.routers.dozzle.middlewares=voidauth@docker"
```
This can be reused for other containers. **When reusing it, you must change any instance of the word "dozzle" to another word**. It doesn't have to match the container name, but it does have to be consistent. Again, **find and replace is your friend**. 

The only place you can deviate is in the host rule where it says ``Host(`dozzle.${MY_DOMAIN}`)``. Here, you can set whatever you like as long as it matches one of your DNS records. Reminder, this expands to something like ``dozzle.mydomain.com`` or something similar.  

The first two lines just enable Traefik and SSL.

Because multiple networks are in use, the following line is necessary to tell Traefik where to find this container. It can be reused for other containers on other networks, just update the network name:
```
- "traefik.docker.network=servicenet"
```

This sets the port Traefik proxies. It will be different for every container and this is part of why I leave ports in the compose file, but commented out... because I won't remember later if I totally remove them:
```
- "traefik.http.services.dozzle.loadbalancer.server.port=8080"
```
Finally, this line enables the VoidAuth middleware. Include it for any containers you want to put auth in front of:
```
- "traefik.http.routers.dozzle.middlewares=voidauth@docker"
```


## Last steps
Make sure you have your DNS configured. At a minimum for this set up, you need to have a domain set up for Void Auth, otherwise you won't be able to complete set up. Bring the stack up, authenticate Tailscale if you didn't use an auth key/pre-approve, then run ``docker logs voidauth`` where it will print a link for you to set the admin password. Give some time for DNS propagation. You can check the Traefik logs to see when it completes; it shouldn't take too long. 

The last thing to do is enable forwarding for your domain in the Void Auth settings. Go to ``void.yourdomain.com`` and click the hamburger menu in the top left, go to "ProxyAuth Domains" and create an entry for your domain. I used a wildcard like ``*.mydomain.com`` but Void Auth is highly configurable and you may want something different. 

From here, you can use the label block with modifications to proxy any number of containers. They don't have to be in the same compose file or on the same network. Don't forget that you can comment out the ports section of any container config for added security. 
