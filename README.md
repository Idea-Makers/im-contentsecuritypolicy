# IM.ContentSecurityPolicy

Content Security Policy (CSP) headers and per-request nonces for ASP.NET Core and Umbraco CMS 13 to 18.

The package:

- generates a random nonce for each request;
- writes a `Content-Security-Policy` or `Content-Security-Policy-Report-Only` header built from `CspOptions`;
- adds the nonce to `<script>`, `<style>` and `<link>` tags through a tag helper, HTML helpers or view components.

It targets net8.0 and runs on .NET 8, 9 and 10. It references `Umbraco.Cms.Core` 13 to 18 so that the Umbraco Marketplace lists it. The code does not call Umbraco, so it also works in plain ASP.NET Core apps.

> **Source availability:** the source repository is private. The package is distributed through
> [NuGet.org](https://www.nuget.org/packages/IM.ContentSecurityPolicy). Report bugs and request features in the
> [issue tracker](https://github.com/Idea-Makers/im-contentsecuritypolicy/issues) and ask questions in
> [Discussions](https://github.com/Idea-Makers/im-contentsecuritypolicy/discussions).

| Umbraco | .NET |
|---------|------|
| 13, 14  | 8    |
| 15, 16  | 9    |
| 17, 18  | 10   |

For Umbraco 10 to 12 use version 1.0.2.

## Installation

```bash
dotnet add package IM.ContentSecurityPolicy
```

## Setup

```csharp
using IM.ContentSecurityPolicy.Extensions;
using IM.ContentSecurityPolicy.Middleware;

builder.Services.AddNonceServices();

var app = builder.Build();

app.UseNonceGeneration();
app.UseContentSecurityPolicy();
```

Call `UseNonceGeneration` before `UseContentSecurityPolicy`. In the other order the header and the page get different nonces.

Register the tag helpers and the HTML helper namespace in `Views/_ViewImports.cshtml`:

```cshtml
@addTagHelper *, IM.ContentSecurityPolicy
@using IM.ContentSecurityPolicy.Extensions
```

## Adding the nonce to tags

### Tag helper

Add `asp-nonce="true"` to a `<script>`, `<style>` or `<link>` element:

```cshtml
<script src="~/js/site.js" asp-nonce="true"></script>

<script asp-nonce="true">
    console.log('inline script');
</script>

<link rel="stylesheet" href="~/css/site.css" asp-nonce="true" />
```

The `nonce` attribute is written only when the request already has a nonce, so the nonce middleware must run first.

### HTML helpers

| Method | Output |
|--------|--------|
| `Html.ScriptWithNonce(src, type = "text/javascript", attributes = null)` | `<script src="..." type="..." nonce="..."></script>` |
| `Html.InlineScriptWithNonce(content, type = "text/javascript")` | `<script type="..." nonce="...">content</script>` |
| `Html.StyleWithNonce(href, attributes = null)` | `<link rel="stylesheet" href="..." nonce="..." />` |
| `Html.InlineStyleWithNonce(content)` | `<style nonce="...">content</style>` |
| `Html.GetNonceAttribute()` | ` nonce="..."`, or an empty string when the request has no nonce |

`attributes` is an anonymous object such as `new { defer = "defer" }`. Property names are written in lower case.

The helpers do not HTML-encode their arguments. Pass only trusted values.

### View components

The view components put the nonce into a view model and render a view. The package does not include the views, so add one per component you use, for example `Views/Shared/Components/NonceScript/Default.cshtml`:

```cshtml
@model IM.ContentSecurityPolicy.ViewComponents.NonceScriptViewModel
<script src="@Url.Content(Model.Src)" type="@Model.Type" nonce="@Model.Nonce"></script>
```

```cshtml
@await Component.InvokeAsync("NonceScript", new { src = "~/js/site.js" })
```

| Component | Arguments | Model properties |
|-----------|-----------|------------------|
| `NonceScript` | `src`, `type = "text/javascript"`, `attributes = null` | `Src`, `Type`, `Nonce`, `Attributes` |
| `NonceInlineScript` | `content`, `type = "text/javascript"` | `Content`, `Type`, `Nonce` |
| `NonceStyle` | `href`, `rel = "stylesheet"`, `attributes = null` | `Href`, `Rel`, `Nonce`, `Attributes` |
| `NonceInlineStyle` | `content` | `Content`, `Nonce` |

The models are in the `IM.ContentSecurityPolicy.ViewComponents` namespace and are named after the component, for example `NonceInlineStyleViewModel`.

### Nonce service

Inject `INonceService` from `IM.ContentSecurityPolicy.Services` to read the nonce in your own code.

| Method | Description |
|--------|-------------|
| `GenerateNonce()` | Creates a new nonce and stores it in `HttpContext.Items`. |
| `GetNonce()` | Returns the stored nonce, or creates one if there is none. |
| `HasNonce()` | Returns `true` when the request has a nonce. |

A nonce is 32 bytes from `RandomNumberGenerator`, Base64-encoded.

## Configuration

`AddNonceServices` takes an optional `Action<CspOptions>`. The source `'nonce-{nonce}'` in any list is replaced with the request nonce.

```csharp
builder.Services.AddNonceServices(options =>
{
    options.ScriptSrc = new[] { "'self'", "'nonce-{nonce}'", "https://cdn.example.com" };
    options.ImgSrc = new[] { "'self'", "data:", "https:" };
    options.ReportOnly = true;
    options.ReportUri = "https://example.com/csp-report";
});
```

| Property | Directive | Default |
|----------|-----------|---------|
| `DefaultSrc` | `default-src` | `'self'` |
| `ScriptSrc` | `script-src` | `'self' 'nonce-{nonce}'` |
| `StyleSrc` | `style-src` | `'self' 'nonce-{nonce}'` |
| `FontSrc` | `font-src` | `'self'` |
| `ImgSrc` | `img-src` | `'self' data:` |
| `ConnectSrc` | `connect-src` | `'self'` |
| `FrameSrc` | `frame-src` | `'self'` |
| `ObjectSrc` | `object-src` | `'none'` |
| `BaseUri` | `base-uri` | `'self'` |
| `FormAction` | `form-action` | `'self'` |
| `FrameAncestors` | `frame-ancestors` | `'self'` |
| `CustomDirectives` | any other directive | empty |
| `ReportUri` | `report-uri` | not set |
| `UpgradeInsecureRequests` | `upgrade-insecure-requests` | `true` |
| `ReportOnly` | sends `Content-Security-Policy-Report-Only` | `false` |

An empty array leaves the directive out of the header.

### Builder

`CspOptionsBuilder` sets the same properties with `With...` methods. Pass it the options instance to change it in place:

```csharp
using IM.ContentSecurityPolicy.Options;

builder.Services.AddNonceServices(options => new CspOptionsBuilder(options)
    .WithScriptSrc("'self'", "'nonce-{nonce}'", "https://cdn.example.com")
    .WithConnectSrc("'self'", "https://api.example.com")
    .WithCustomDirective("worker-src", "'self'"));
```

There are two presets:

- `CspOptionsBuilder.CreateStrict()` is the default policy with `frame-src 'none'`.
- `CspOptionsBuilder.CreateDevelopment()` adds `'unsafe-inline'` and `'unsafe-eval'` to scripts, `'unsafe-inline'` to styles, `data:` to fonts, `https:` to images and `ws:` `wss:` to connections, and turns off `upgrade-insecure-requests`.

A preset returns a new builder, so copy the values you need:

```csharp
builder.Services.AddNonceServices(options =>
{
    if (builder.Environment.IsDevelopment())
    {
        var dev = CspOptionsBuilder.CreateDevelopment().Build();
        options.ScriptSrc = dev.ScriptSrc;
        options.StyleSrc = dev.StyleSrc;
        options.ConnectSrc = dev.ConnectSrc;
        options.UpgradeInsecureRequests = false;
    }
});
```

### Options on the middleware

`app.UseContentSecurityPolicy(options => { ... })` starts from the defaults and ignores the options registered with `AddNonceServices`.

## Umbraco

The default policy blocks the inline scripts of the Umbraco backoffice. Apply the middleware to front-end requests only, before `app.UseUmbraco()`:

```csharp
app.UseWhen(
    context => !context.Request.Path.StartsWithSegments("/umbraco"),
    branch => branch.UseNonceGeneration().UseContentSecurityPolicy());
```

## Security notes

- Start with `ReportOnly = true` and a `ReportUri`. Enforce the policy once the reports are clean.
- Nonces cover `<script>`, `<style>` and stylesheet `<link>` elements. They do not cover inline event handlers such as `onclick`, or `javascript:` URLs.
- Do not put a nonce on a script that contains user input.
- Cached markup keeps the nonce it was rendered with. Do not use output caching or Umbraco cached partials for markup that contains a nonce.

## License

Free for private and commercial use under the [IM.ContentSecurityPolicy License](https://github.com/Idea-Makers/im-contentsecuritypolicy/blob/main/LICENSE.md):
no modification, distribution only as part of your website or application, and a commercial
website or application shows "Uses IM.ContentSecurityPolicy by Idea Makers" with a link to https://ideamakers.pl.
