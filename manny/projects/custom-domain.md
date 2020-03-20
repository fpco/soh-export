# How to use a custom domain name with FP Haskell Center

To use a custom hostname that is **not** a subdomain of *fpcenter.io* (e.g., *myapp.mycompany.com*), you are responsible for configuring the Internet's [Domain Name System](http://en.wikipedia.org/wiki/Domain_name_system) (DNS) to translate your domain name to the IP address of the server running your app.  While a complete tutorial on DNS is beyond the scope, the requirements are:

  * Register your 2nd-level domain (e.g., *mycompany.com*) name with a [registrar](http://en.wikipedia.org/wiki/Domain_name_registrar) for your top-level domain (e.g. *.com*).
  
  * Have a [name server](http://en.wikipedia.org/wiki/Name_server).  Many registrars will include a name server with registration, or use a third-party DNS hosting service.
  
  * On your name server, create a resource record with fields:
  
      * Name: Your custom hostname (e.g., `myapp.mycompany.com`)
      * Type: `CNAME` (sometimes known as an *alias*)
      * Value: the **host name** of your app server as displayed on the deployment manager's server details page.  To reach the server details page, visit the [deployment manager](https://www.fpcomplete.com/deploy) and select your app server.  The host name will look like `26619930-b535-497e-a895-73d033cc251a.fpcenter.io` (for shared app servers) or `ec2-23-22-64-135.compute-1.amazonaws.com` (for dedicated EC2 app servers), with different random numbers.
    
Then, enter your custom hostname (e.g., `myapp.mycompany.com`) in the **hostname** field of the IDE's deployment configurations tab, and deploy your app.
  
### Subdomains of fpcenter.io

Using a custom hostname which is a subdomain of *fpcenter.io* is simple: enter the hostname you want (e.g. `myapp.fpcenter.io`) in the **hostname** field of the IDE's deployment configurations tab, and it will work automatically after you deploy.  Note that each customer is limited to five of these custom `fpcenter.io` subdomains at a time per app server license.
