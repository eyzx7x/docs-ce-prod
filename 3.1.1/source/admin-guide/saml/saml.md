## Gluu Server SAML Intro

## Overview
SAML is an XML-based, open-standard data format for exchanging authentication and authorization data between an identity provider 
(like the Gluu Server) and a service provider (like Dropbox, O365, etc.). 

The Gluu Server bundles software to achieve two SAML workflows:

### Outbound SAML 
Outbound SAML can also be called SP-initiated single sign-on (SSO) or traditional SAML. 

In an outbound SAML transaction a website or application (SP) redirects a user to a designated Identity Provider (IDP) for authentication and authorization. The IDP asks for the user's credentials and upon successful authentication redirects the user to the protected content. 

To achieve outbound SAML SSO, you can refer to the following docs: 

- [Shibboleth SAML IDP]()    
- [Shibboleth SAML SP]() 

### Inbound SAML     
Inbound SAML single sign-on enables an organization to offer SAML authentication as a front door to their digital service. Inbound SAML is a common requirement for SaaS providers that need to support the authentication requirements of large enterprise customers. 

In many ways, Inbound SAML was the precursor to what we now commonly refer to as "social login". 

A typical inbound SAML user flow is: 

1. User tries to access a protected resource at your application;    
2. User is redirected to a discovery page (presented by your IDP) that presents one or more external IDP's where the user may have an existing identity (their "home IDP");   
3. User selects their home IDP and is sent home for authentication;   
4. Upon successful authentication at their home IDP, user is redirected back to your service with access to the protected resource. 

The Gluu Server supports two software options for Inbound SAML: 

- [Passport.js SAML IDP]()      
- [Asimba SAML Proxy]()   

Both software components are optional during the Gluu Server installation and setup process. 

Of the two options, **we recommend using Passport.js SAML for inbound SAML SSO**. Key management is more complicated in Asimba, and using Asimba requires modifications to be made in configuration files.

!!! Note
    For a good realworld example of inbound SAML, check out the [Federated Login flow at EDUCAUSE](https://sso-users.educause.edu/?resumePath=%2Fidp%2FZQ4DF%2FresumeSAML20%2Fidp%2FstartSSO.ping&allowInteraction=true&reauth=false). 

## SAML vs. OpenID Connect  

SAML is stable and mature, and is well supported at many of the Internet's largest domains. 

However, the last major release of SAML was in 2005! So if your target application doesn't already support SAML, or for new application development, we recommend using [OpenID Connect](./openid-connect.md). 

The following bullet points might help you determine when to use SAML, and when to use OpenID Connect:

- If you have an application that already supports SAML, use SAML.
- If you need to support user login at an external IDP (like a customer or partner IDP), use SAML.
- If you have a mobile application, use OpenID Connect.
- If you are writing a new application, use OpenID Connect.



