## twain

I've long been dissatisfied with the limitations of basic authentication and reverse proxies in general, but whenever I tried to figure out a better way, the information I found was somewhat scattered and incomplete.

I managed to piece it together eventually, and realised that decoupling the reverse proxy from authentication and access control was exactly what I needed. I wrote an application to do just that, and decided to publish it, along with what I've learned.

This post is mostly about the concept of forward authentication, as that is a big piece of the puzzle and one about which there is limited information available.

### Reverse Proxies

Running a reverse proxy is a very useful way to provide secured access to the web-based applications you have running on your internal network. It takes incoming requests, and forwards them to another server, typically based on the host name and/or path of the originally requested resource.

Reverse proxies are typically just web servers which we have bent to our will, but my current preferred reverse proxy is one which was designed from the ground up to be a reverse proxy. It's called Traefik, and I like it because

1. it's a single binary, which makes for easy installation and updating on my Synology NAS
1. it automatically detects and provides access to any new apps I deploy in Docker
1. it automatically gets HTTPS certificates from Lets Encrypt
1. it's easy to configure

Options for authentication and authorisation are fairly limited though. Apache and nginx are more able, but still not as flexible as I'd like.

### Enter Forward Authentication

Forward authentication can give us a lot more flexibility. Perhaps the best way to explain forward authentication is to look at a typical authentication flow:

1. A client requests https://app.example.com/path
1. The reverse proxy makes a separate request to the forward auth url, including the details of the original request
1. The forward auth page returns 200 if the request should be allowed, and 401 or 403 if it should be refused.
1. The reverse proxy allows and proxies the request to the originally requested service, or rejects the query, based on the response from the forward auth page

It's that simple. The power of this approach comes from the fact that the forward auth page has a wealth of information it can use to determine whether an access attempt should be allowed.

Let's delve deeper into step 3 in the above flow.

### The forward auth page

The forward auth page uses all the information provided by the reverse proxy to determine how it should respond. This information can include:

* A username/password passed by basic authentication
* The IP address of the original client
* The originally requested protocol, host and URL (in this case, https://, application.domain.tld and /path)
* Any cookies provided by the client

(The current stable version of Traefik does not pass the path of the request, so you need to compile from source. It will be part of the 1.6 release.)

The authentication page then evaluates all the above information to determine whether access should be allowed.

This is quite a lot of information, which means we can be quite clever about whether or not we allow access. Already, we can see that forward authentication can offer much greater flexibility than simple basic authentication (which, by the way, we can still handle).

If the forward authentication page decides that the user is not authorised, it will return a 4xx page, and Traefik will deliver this rejection page to the client. This means we can immediately return a logon page as soon as we decide to deny access.

(This is one of the fundamental differences compared to nginx. Nginx will not deliver the rejection page, and has to be configured to redirect the browser at this point.)

### Logon page

Once access has been rejected, we may wish to provide the user some way to prove that they deserve access after all, in which case they should be presented with a login page of some kind. You could integrate with an SSO system, OAuth2, or simply have a username and password form. Whatever happens, the user will probably be sent along a path which includes a couple of different pages during the authentication process. For example:

1. User types their username and password and clicks Submit
1. The username and password get posted to a second page which verifies the provided credentials.

HTTP is stateless, which means that each request by the client will be processed in isolation, with no regard for previous queries or the fact that we’ve already authenticated. Every time we make a request, it's like we're connecting to the server for the very first time. Cookies are the standard way of carrying information between sessions so that we can preserve state, so, once the user's credentials have been verified, the authentication script should:

1. Set a cookie in the user's browser
1. Store a token on the server to indicate that said cookie is linked to an authorised user

Now that we’ve authenticated the user, we can send them back to the page which they originally requested. Remember that HTTP is stateless though, so we will need to have kept track of the original request so that we can send the user back there. Let’s adapt the above process accordingly.

1. The user types their username and password and clicks Submit
1. The username, password **and originally requested page** get posted to https://twain.example.com/login.php
1. The authentication page verifies the provided credentials
1. The authentication page sets a cookie in the user's browser
1. The authentication page stores a token on the server to indicate that said cookie is linked to an authorised user
1. **The authentication page redirects the user to the originally requested resource**

### The full process

The full process from beginning to end, for an initially unauthenticated user, thus looks like this.

1. A client requests https://app.example.com/path
1. The reverse proxy makes a separate request to the Forward Auth page, including the details of the original request
1. The forward auth evaluates the client IP, cookies, basic authentication credentials and the requested URL
1. The forward auth page returns a 4xx response, as the user is not currently allowed to access that page
1. The reverse proxy delivers the 4xx page to the client
1. The user types their username and password and clicks Submit
1. The username and password and originally requested page get posted to https://twain.example.com/login.php
1. The authentication page verifies the provided credentials
1. The authentication page sets a cookie
1. The authentication page stores a token on the server to indicate that said cookie is linked to an authorised user
1. The authentication page redirects the user to the originally requested resource

### Email authentication

I use a password manager, with complex passwords. They’re nigh-impossible to remember, and a pain to read from a phone screen while typing them into a PC. Ideally, if I don't fully trust a system, I don't want to enter my password anyway so, in addition to the above, I’ve set up another logon flow. It looks like this:

1. A client requests https://app.example.com/path
1. The reverse proxy makes a separate request to the Forward Auth page, including the details of the original request
1. The forward auth evaluates the client IP, cookies, basic authentication credentials and the requested URL
1. The forward auth page returns a 4xx response, as the user is not currently allowed to access that page
1. The reverse proxy delivers the 4xx page to the client
1. User types their username **(but not their password)** and clicks Submit
1. **The username and originally requested page get posted to https://twain.example.com/login.php**
1. **The authentication page sends the user an email and displays a message informing them of that fact, and keeps checking in the background if the link has been clicked
1. **The user clicks the link in their email**
1. **The authentication page redirects them to the originally requested resource**
1. The authentication page sets a **session** cookie
1. The authentication page stores a **very-short lived** token on the server to indicate that said cookie is linked to an authorised user
1. The authentication page redirects the user to the originally requested resource

Hopefully this example serves to illustrate the potential power of forward authentication.

### The authentication page itself

I’ve been vague about the nature of the authentication page so far, because it could be anything. I created twain in PHP, because I’m using a Synology NAS to host the authentication page, and it comes with a web server supporting PHP, but you can implement this any way you like, as long as it serves HTTP.

While I've settled on Traefik, the basic concept is supported by a number of reverse proxies/web servers. I personally tested it with nginx before I settled on Traefik, and it was fully functional. In nginx, the module providing this functionality is called http_auth_request.

twain is available [here](https://www.github.com/hjbotha/twain)
