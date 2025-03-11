# Supporting Bearer Token Auth for Credential Providers

- Jonatan Gonzalez ([jgonz120](https://github.com/jgonz120))
- Issue [#12877](https://github.com/NuGet/Home/issues/12877)

## Summary

Update the NuGet CLI to allow for bearer token authentication when publishing packages.

## Motivation

NuGet.org wants to shift to using bearer token authentication for publishing packages, enabling the use of short lived, 15-minute, API keys for authentication.
This would reduce the security risk of a key leak, since currently the long-lived keys can be valid for up to a year.

Currently NuGet.org takes in the API key through the [X-NuGet-ApiKey](https://learn.microsoft.com/en-us/nuget/api/package-publish-resource#request-parameters) header.
The CLI for pushing provides a warning if the ApiKey is not specified causing feeds, [like Azure Artifacts](https://learn.microsoft.com/en-us/azure/devops/artifacts/nuget/publish?view=azure-devops#publish-packages), to tell users to pass a placeholder value into the command.

Additionally due to the [limitation in .NET](https://github.com/dotnet/runtime/issues/91867#issue-1889923702) we can’t send these types of credentials using the HttpClientHandler.
To circumvent this Azure Artifacts sends their key using basic auth, passing the key as a password in a network credential.
By working around the .NET limitation for bearer token usage, we can allow a more technically correct solution and allow other feeds to implement a credential provider that returns a bearer token.

## Explanation

### Functional explanation

The push command will only display the no API key warning when the API key is not specified and we were unable to authenticate with the source.

When the NuGet client receives an unauthorized response, it will pass the `WWW-Authenticate` header to the credential providers.
When credential providers return a bearer token, the request will be retried by adding the bearer token to the authentication header.
When we make a request with an expired token, the server should return a 401 Unauthorized respond, causing a new request for credentials.

### Technical explanation

[HttpSourceAuthenticationHandler.SendAsync](https://github.com/NuGet/NuGet.Client/blob/313eecd3af442ee2eeed2e6decf310858934ab21/src/NuGet.Core/NuGet.Protocol/HttpSource/HttpSourceAuthenticationHandler.cs?plain=1#L68) will be updated to check if the previous response contains a WWW-Authenticate: Bearer header.
If it does, then before sending the request the method will attempt to retrieve the token from the credentials provided and pass them in the authentication header if it’s available.

[GetAuthenticationCredentialsRequest](https://github.com/NuGet/NuGet.Client/blob/ab6d3b886b32e9b8f68a7f1b299c43b0c72487dc/src/NuGet.Core/NuGet.Protocol/Plugins/Messages/GetAuthenticationCredentialsRequest.cs?plain=1#L12C25-L12C60) will be updated to include a new field, `WwwAuthenticateHeader`, that will tell the credential providers which auth headers can be used.
This can be used by CredentialProviders to know if they’re communicating with a version of NuGet that supports Bearer tokens.
If `WwwAuthenticateHeader` is null, then the version of nuget they are communicating with does not support bearer tokens. Credential providers that want to use bearer auth will return a [GetAuthenticationCredentialsResponse](https://github.com/NuGet/NuGet.Client/blob/ab6d3b886b32e9b8f68a7f1b299c43b0c72487dc/src/NuGet.Core/NuGet.Protocol/Plugins/Messages/GetAuthenticationCredentialsResponse.cs?plain=1#L14C1-L14C61) that has “bearer” in the AuthenticationTypes list.

When deciding which protocol to use for authentication we will use a combination of the headers provided and the results from ICredentials.GetCredential.
If the header indicates the server accepts basic, and GetCredential returns a result for basic, we will use that for the request.
Similarly, if the header accepts bearer, and GetCredentials returns a result for bearer, we will use that.

To maintain the “PreAuthenticate” functionality, a cache can be used to keep track of URLs and the credential previously used to authenticate against it.

## Drawbacks

Even though we are moving away from sending the bearer token as a password during the request, the ICredential.GetCredential explicitly returns a NetworkCredential, which requires a username and password.

## Rationale and alternatives

### Alt - Use basic Authentication

This option would have NuGet.org implement the credential provider like Azure Artifacts does for authentication.
The client wouldn’t need to be updated, and the API Key would be sent over to the server using basic authentication.
The username field would be filled with a placeholder.
The client could still be updated to no longer require ApiKey.
While this solution has minimal work for the client, it requires using an authentication mechanism that expects a username and password.

We would need to verify that these changes do not negatively impact feeds which use basic authentication as a workaround.

## Prior Art

## Unresolved Questions

## Future Possibilities

Encourage .NET to implement [Issue #91867](https://github.com/dotnet/runtime/issues/91867#issue-1889923702), allowing for native support of bearer tokens.
This would allow us to remove the code required to pass the bearer token in the header and instead let the framework handle it.
This would require moving NuGet HTTP requests out of the devenv.exe into a service hub process or waiting for devenv.exe to move to .NET core.
