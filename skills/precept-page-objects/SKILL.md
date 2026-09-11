---
name: precept-page-objects
description: "Design and write page objects on Precept.Web: IPageContext injection, locator properties, scoped registration, tabs and second users, and when to move to screenplay. Use when creating or editing a page object, a reusable component, or the class that owns an app's persistent chrome."
---

# Page objects on Precept.Web

```csharp
using Microsoft.Playwright;
using Precept.Web;

public sealed class SignInPage(IPageContext context)
{
    public ILocator Email  => context.Page.GetByLabel("Email");
    public ILocator Submit => context.Page.GetByRole(AriaRole.Button, new() { Name = "Sign in" });
    public ILocator Error  => context.Page.GetByRole(AriaRole.Alert);

    public Task OpenAsync() => context.Page.GotoAsync("/login");          // relative to Web:BaseUrl

    public async Task SignInAsync(string email, string password)
    {
        await Email.FillAsync(email);
        await context.Page.GetByLabel("Password").FillAsync(password);
        await Submit.ClickAsync();
    }
}
```

```csharp
services.AddScoped<SignInPage>();                     // in IPreceptStartup.ConfigureServices: one per test and per retry
```

A `[Binding]` class takes it by constructor; a plain test resolves it from `PreceptTestContext.Current.Services`. The test then asserts on the locator: `await Assert.That(signIn.Error).ToBeVisibleAsync()`.

## Rules

1. **Depend on `IPageContext`, never on `IPage`, and read `context.Page` where it is used.** A page or locator copied into a field points at the tab it was when the object was built; the context is what lets a test move every page object to a popup or to another user with one assignment (`context.Page = popup`). Restore it in a `finally` when the move is temporary.
2. **Expose locators, not values, and never assert.** A locator is a query the assertion retries against the live page; a `string Text` or `bool IsVisible()` is a snapshot the test then has to wait for. `WaitFor…` calls and `Thread.Sleep` have no place here.
3. **Public methods are the user's actions in the domain's words**: `SignInAsync`, `AddToBasketAsync(sku)`, not `ClickSubmit` or `FillEmail`. An action that lands on another screen does not return that screen's object; the other page object is injected too and follows the same context.
4. **Locate by what the user sees**: `GetByRole`, `GetByLabel`, `GetByText`, then `GetByTestId`; a CSS or XPath selector is the last resort and gets a comment saying why. Selectors live only in the page object.
5. **One class per screen, named `*Page`.** A fragment used on several screens (a data grid, a date picker, a toast) is a component taking the same `IPageContext` plus the locator of its root, so it scopes its own queries. The persistent chrome — navigation, header, sign-out, notifications — is one class named `*Root` (`AppRoot`), not repeated on every page.
6. **No state beyond the context.** A page object is shared by every user the test acts as, so a field set for one user is read for the next; keep what a step needs to remember in the step class or `ScenarioContext`.
7. **A page object that grows a method per scenario is the signal to stop adding to it.** Screenplay (`Precept.Screenplay`: `Target`, `IInteraction`, `Actor`) composes the same locators into single-purpose interactions; the two styles can share a project. See `precept-modules`.

Done when the class is registered scoped, takes `IPageContext`, holds no page or locator in a field, every public member is a locator or a user action, and the test that uses it contains every assertion.
