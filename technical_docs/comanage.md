# COmanage Plugin

Plan for writing a COmanage plugin to connect the registry to an OA4MP server to issue SciTokens.

## Update Oauth2 server

The current OAuth2 server code in 
[Model/Oauth2Server.php](https://github.com/Internet2/comanage-registry/blob/develop/app/Model/Oauth2Server.php)
hard wires some end points and doesn't currently allow dynamic client registration. I'll re-wire this using
https://github.com/thephpleague/oauth2-client to improve the OAuth2 plumbing in COmanage servers.

This will be done as a separate pull request, as it's independent of the OA4MP work.

## Update OA4MP App.

### Switch to proper OAuth2 interface

Starting from Scott's [Oa4mpClient](https://github.com/cilogon/Oa4mpClient), make the necessary changes so that this app can talk to both traditional OA4MP and SciTokens OA4MP.

First change the interface [oa4mpInitializeRequest](https://github.com/cilogon/Oa4mpClient/blob/master/Controller/Oa4mpClientCoOidcClientsController.php#L784) to use [OAuth2 dynamic client registration](https://tools.ietf.org/html/rfc7591) through the COmanage OAuth2 server plugin.

Documentation on this is [here.](http://grid.ncsa.illinois.edu/myproxy/oauth/server/manuals/dynamic-client-registration.xhtml)

### Add Correct Scopes for SciTokens
