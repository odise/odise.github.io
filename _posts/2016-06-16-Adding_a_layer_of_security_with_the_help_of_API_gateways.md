---
layout: post
title: "Adding a layer of security with the help of API gateways"
tagline: "API gateway thoughts"
category: 
tags: [ AWS, api gateway, security ]
---
{% include JB/setup %}
# Adding a layer of security with the help of API gateways

**tl;dr:**
How to create a unified API and implement additional business logic to orchestrate a micro-service infrastructure setup? What set of features an API gateway solution needs to provide in general? Is Amazon API Gateway ready to take over that role? 

![alt text](http://c7.staticflickr.com/2/1535/26580883726_e779f3a774_h.jpg "Gateway")
Image credits go to [Alexis Lewis](https://www.flickr.com/photos/136522029@N03/26580883726).

## Prologue

Building extendable web applications isn't an easy task to do. Especially with microservices environments it might start with domains of small responsibilities but tend to become a zoo of components accumulating clusters of functionality. Dependencies among those micro components can easily lead into complex architectures that can become hard to manage. 

At a certain point in time, security aspects coming along with complex systems arise on architecture board schedules. People start to realize that, let's say service A, B and C aren't customer facing but still accessible for everyone. Maybe service A and B are already prepared to restrict access with the help of authentication but C does not authenticate clients at all. Analyzing the details reveals that obfuscation is used instead of a following security patterns. Using hard coded API tokens as part of the URL might be one example. 

A couple of approaches exist for overcoming such problems. Moving backend services to private subnets and restricting access via network ACLs is a good choice but not always possible. Complexity gains with Javascript front-end application which are open to the user. APIs serving native mobile apps as well as web apps running in a browser at the same time can considered being a common scenario these days. So how do you secure such a setup and reduce the development effort as much as possible? Facing the facts, software developers tend to be paid to implement features whereas operations people are expected to take care of service and infrastructure security. So let's get practical.

## API gateways 

With the rise of micro service architecture [API gateways](http://microservices.io/patterns/apigateway.html) becoming more popular. The idea is to provide a single entry point for all the fine-grained APIs of the individual services. That way authentication and even more important authorization can be handled from a single spot avoiding the implementation of permission rules as part of the individual service components. In short, the following aspects I consider to be the key features that a API gateway solution should provide:

1. Configuration based user facing API definition. Basically I would expect reverse proxy functionality to the underlying services without the necessity to write code.
2. TLS secured endpoints on the front-end. Certificates should be manageable in a convenient way.
3. User authentication must be a part of the API request handling. Having said this validity checking of a [JSON web tokens](http://jwt.io) (JWT) will do. The token creation can be done by another service component whereas redirecting non-authenticated request should be forwarded to that service (HTTP 302 redirect to a login page). 
4. Authorization should be covered as part of the request handling. That way the business logic can be gathered in one place. Of course such logic must be maintainable (deployable) in a reasonable way.
5. Backend services behind the API gateway should be accessible from the API gateway only. 
6. The whole solution must scale to handle traffic peaks and be able to cope with potential service attacks.
7. Web application firewall functionality should be part of the solution. Integration should be easy. 
8. As part of the previous point logging and auditing would be very appreciated.

Enterprise flavored products like [Apigee](https://apigee.com), [CA API Gateway](http://www.ca.com/us/products/ca-api-gateway.html), [Oracle API Gateway](http://www.oracle.com/us/products/middleware/identity-management/api-gateway/overview/index.html), [Strongloop](https://strongloop.com) as well as open source solutions as [Tyk](https://tyk.io/) and [Kong](https://getkong.org/) aiming to shield and unify micro service environments. Most interesting for AWS backed infrastructures is the [Amazon API Gateway](https://aws.amazon.com/de/api-gateway/) (AGW). So let's have a closer look on that one.

## AWS API Gateway

At the first glance the feature set looks pretty good.

* REST API creation based on web UI and Swagger import 
* throttling rules for all HTTP methods and endpoints
* build in CloudFront distribution for API endpoints doing TLS termination and caching
* API versioning to handle stage, live etc. environments 
* client SDKs for JavaScript, iOS, and Android
* signature version 4 authentication
* IAM and access policies to authorize access for AWS resources

Let us dive into details and check which of the points mentioned above does AGW cover. 

![alt text](https://s3.amazonaws.com/awscomputeblogmedia/1_custom-authorizers-flow.png "architecture")


### API definition

AWS did a pretty good job on how you can define the user facing APIs. A reasonable UI gives a rich set of feature at hand. Moreover, it is possible to import and export the API as Swagger representation. Honestly, I didn't check how good that feature works. The feature set of Swagger definitions goes most probably far beyond the things AGW can handle. However, as we are limiting ourselves to reverse proxy like functionality this should be good enough to store AGW API definitions in a [VCS](https://en.wikipedia.org/wiki/Version_control). 
 

### Endpoint TLS termination

Yes this feature is available as mentioned earlier. However: as of today AWS doesn't support certificate management as part of [AWS Certificate Manager](https://aws.amazon.com/certificate-manager/) (ACM). This becomes essential whenever your API should be available under a customized CNAME saying api.myservice.example.com. What does that mean in detail? 

ACM provides the possibility to create and maintain TLS certificates and use it as part of CloudFront distributions and for ELBs without additional charge. This saves a bunch of maintenance overhead and costs (personally I think this is one of the best features AWS released during the last months). It got rolled out to a couple of AWS regions and I use it extensively. Without this key feature, all AGW APIs with a customized CNAME depend on a TLS certificate in PEM format along with certificate chain. Consequently one must buy or use TLS certificates and manage these manually (e.g. storing them on secure place and renew them when they are going to be expired). Additionally I'm unsure on how to renew such a certificate in production without taking the whole API service down. 

For further details have a look at the [AGW documentation](http://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-custom-domains.html).

### Authentication and authorization 

In order to decide whether a web request hitting your API has permission to be forwarded, AGW provides [API key functionality](http://docs.aws.amazon.com/apigateway/latest/developerguide/how-to-api-keys.html). This way specific routes of the API can be restricted. To implement more fine-grained access handling [custom authorization](http://docs.aws.amazon.com/apigateway/latest/developerguide/use-custom-authorizer.html) can be deployed as part of the API definition. This way JWT (or any other) can be implemented as Lambda functions. Have a look at [this post](https://aws.amazon.com/blogs/compute/introducing-custom-authorizers-in-amazon-api-gateway/) on how to do this.

### Backend service security 

IMHO one of the main aspects of why you would introduce a single point of failure like a API gateway is the ability to encapsulate under laying services and hiding them from the rest of the world (aka the wicked Internet). 

Maybe I'm a little bit over the edge here but as ACM provides the ability to manage TLS in sane way I introduced this to more or less all services possible (as part of CloudFront distributions or along with an ELB). Getting rid of certificate files deployed to EC2 instances I felt like a no-brainer from now on. Sadly I realized that AGW jeopardizes this huge step forward for the time being. So why is that?

In order to restrict EC2 server access to the AWS AGW one needs to implement [client-side X.509 authentication](https://en.wikipedia.org/wiki/Mutual_authentication) on EC2 instance level. This practically means that ACM in combination with ELB doesn't work anymore. Instead the ELB needs to forward HTTPS traffic as TCP to the underlying web server (deployed on the EC2 instance) which needs to authorize the request. The certificates to do this can easily be created within the AGW, that's true, but ... I definitely prefer network ACL approaches, security group based access management and ACM! For further details please have a look at the [documentation](http://docs.aws.amazon.com/apigateway/latest/developerguide/getting-started-client-side-ssl-authentication.html).

### Scalability 

As an AWS service, AGW should scale with the number of requests hitting the API. Again this hasn't been evaluated in detail yet but I think it is one of the main reasons why one would prefer a managed solution over a self-managed approach like Kong etc. 

There are soft-ish limits that restrict the number of requests per second to 1000 (steady-state) and 2000 requests per second on bursts. Check all the limits [here](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-limits.html).


### Web Application Firewall

Up to this point there is no [AWS WAF](https://aws.amazon.com/waf) integration in AGW. However AWS WAF is still one of "the new kids in town" as well so it wouldn't be surprising if that is going to be integrated into AGW (like ACM - hopefully). 

### Logging and auditing

The good news is that there is CloudWatch in place already. AGW integrates it and everything that comes along with that. A lot of third-party services support CloudWatch already and at least provides you with the possibility to establish alarms and logging. 

# Conclusion 

Is Amazon API Gateway worth trying? Yes and no. Apart from the missing certificate integration in ACM the product looks and feels good. I have no idea how a client side SSL authentication integrated into AWS could look like. Most probably there is a lot of work to do for the AWS engineering team in order to integrate it into the ELB configuration ([this](https://blog.cloudflare.com/protecting-the-origin-with-tls-authenticated-origin-pulls/) is how CloudFlare is doing it). Restricting the access with the help of security groups sounds like a sane idea to me but of course I don't have AWS insight to judge on the complexity of this.

The question on whether to use an API gateway solution at all strongly depends on the scenario. It definitely introduces another single point of failure. Missing auth functionality can be a good reason to go with API gateways as it reduces the effort of changing each and every component which can be very painful.

Personally, I wouldn't consider it without the certainty of rock solid scaling underneath. Self managed solutions like Kong deployed on EC2 in combination with an autoscaling group might work very well under the condition that the gateway software generates useful metrics which can be used to control the autoscaling group size. I didn't check the details mainly because there is another aspect that lowers my excitement about Kong: the dependencies it comes with (Cassandra, PostgreSQL). 

I really would like to encourage AWS to push ACM into other AWS products. This is definitely the way to go. Nobody wants to care for SSL/TLS certificates on instance level - it's pure pain and opens a large variety of security holes. Besides that network based access restrictions feels like a good practice. It is easy to understand and can be managed as part of the infrastructure automation (e.g. [CloudFormation](https://www.youtube.com/watch?v=kV4vHpqrj6E) or [Terraform](https://terraform.io)).
