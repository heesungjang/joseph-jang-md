---
id = "spfx-oninit-async-await-blocking"
title = "Why Our SPFx Web Parts Randomly Disappeared (And How We Fixed It)"
abstract = "Some of our SharePoint web parts were randomly failing to render for certain users. The problem? An async/await call in onInit() that seemed harmless until it wasn't."
tags = ["sharepoint", "spfx", "javascript"]
date = 2025-08-18
status = "show"
---

We had this weird bug. Some users reported that our SharePoint web parts just wouldn't show up on their pages. Not all the time, not for everyone, just... randomly. The DOM would be empty where the component should be.

Sometimes there was a console error: `Failed to execute 'removeChild' on 'Node'` but it didn't make much sense. What was trying to remove what child? The web part wasn't even rendering.

Couldn't reproduce it locally. Took us months to figure out what was happening.

It might seem like an obvious silly mistake, but hopefully this helps somebody... because it took us way too long to figure out.

## Finding the Clue

I was scrolling through a SharePoint developer Discord when someone shared this exact problem. They had a web part with a "phone home" feature that tracked usage by pinging their API from the `onInit()` method. Their solution caught my attention:

> "Unfortunately, if the API service receiving these requests was down, the request would not complete or would take too long to complete, resulting in the onInit() method not returning and, subsequently, the web part not rendering.
>
> We fixed this by switching to the pattern fetch('https://....').then(...).catch(...) to ensure the PhoneHome component returns immediately, allowing onInit() to finish execution and the web part to render."

That's when it clicked. We had the same pattern, just with a different API call.

## Our License Check Problem

Our web parts checked licenses during initialization. Made sense - why render a component the user isn't licensed to use? The code looked something like this (simplified version):

```typescript
export interface ILicense {
  expired: boolean;
  plan: Plan;
  renewalDate?: string;
  message?: string;
  status?: StatusType;
}

export abstract class OurExtendedClientSideWebPart<
  T extends IWebPartProps
> extends BaseClientSideWebPart<T> {
  protected async onInit(): Promise<void> {
    try {
      const tenantId = this.context.pageContext.aadInfo.tenantId.toString();
      const userId = this.context.pageContext.aadInfo.userId.toString();
      const componentId = this.context.manifest.id;
      const version = this.context.manifest.version;

      // license check with licensing service
      this.properties.license = (await LicensingService.HasValidLicense(
        tenantId,
        userId,
        componentId,
        version,
        this.context
      )) as ILicense;
    } catch (error) {
      // do the error handling..
    }

    return Promise.resolve();
  }
}
```

Just another async/await code.. with try-catch. What could go wrong?

## The Promise Limbo

Apparently everything. But the tricky part was more subtle than we initially thought.

We had multiple layers of error handling - try-catch in `onInit()` AND try-catch inside our `LicensingService`. This gave us false confidence that everything was "safe." The licensing service would catch any network errors and just return `false` instead of throwing.

But that was exactly our mistake. When a network request hangs indefinitely, it never gets to the point where it would resolve or reject. The licensing service just sits there waiting for a response that never comes. No error is thrown because there's no error - just a promise stuck in "pending" state forever.

So our try-catch blocks never run. Not because they can't catch the error, but because the promise never resolves OR rejects. It's stuck in limbo.

And here's the thing about SPFx - `render()` doesn't get called until `onInit()` completes. I'm not entirely sure but it seems so if `onInit()` takes too long, SPFx eventually gives up and tries to clean up the DOM element, which is where that `removeChild` error comes from. The framework expected a web part to be there, but `onInit()` never finished, so there's nothing to remove.

## The SPFx Lifecycle Gotcha

SPFx has this lifecycle where `onInit()` has to complete before anything else happens. When you return a Promise from `onInit()`, the framework waits for it. Makes sense for critical initialization, but it also means any hanging promise will kill your component.

The Discord person's `.then().catch()` approach would work, but we realized we had a different problem. License validation wasn't optional - we actually needed to know if the user was licensed before rendering anything.

## Moving the Check

Instead of trying to make the async call non-blocking, we moved the license check out of `onInit()` entirely. We created a wrapper React component that handles licensing at the component level:

```typescript
// Clean onInit - no API calls
protected async onInit(): Promise<void> {
  // other simple setups for the webpart like pnp and etc.
  return Promise.resolve();
}

// Render method now uses the wrapper
public render(): void {
  const element = React.createElement(LicenseWrapper, {
    context: this.context,
    componentId: this.context.manifest.id
  });

  ReactDom.render(element, this.domElement);
}
```

The `LicenseWrapper` component handles the async license check after the web part has already rendered:

```typescript
const LicenseWrapper: React.FC<Props> = ({
  context,
  componentId,
  children,
}) => {
  const [license, setLicense] = useState<ILicense | null>(null);

  useEffect(() => {
    const checkLicense = async () => {
      try {
        const licenseService = new LicenseService();
        const licenseResult = (await licenseService.validateLicense({
          tenantId: context.pageContext.aadInfo.tenantId,
          componentId,
        })) as ILicense;
        setLicense(licenseResult);
      } catch (error) {
        console.warn("License check failed:", error);
        setLicense({ expired: true } as ILicense);
      }
    };

    checkLicense();
  }, [context, componentId]);

  //  some more logics like loading skeleton and etc.

  if (license.expired) {
    return <TrialCard />;
  }

  return <>{children}</>;
};
```

## Why This Works Better

Moving the license check to React component level means:

- `onInit()` completes immediately, so the web part always renders something
- Users see a loading state instead of nothing
- Network issues with the license API don't break the entire component
- The error handling is more explicit

The web part renders fast, then the license check happens. If it fails, users get a proper error message instead of a mysterious empty space.

## What We Learned

`async/await` syntax hides the blocking behavior. When you write `await someApiCall()`, it looks innocent, but in lifecycle methods like `onInit()`, that "await" can freeze your entire component.

The old `.then().catch()` pattern was more explicit about being asynchronous. With async/await, it's easy to accidentally create dependencies you didn't intend.

Framework lifecycle methods are usually the wrong place for optional API calls. Critical setup belongs in `onInit()`. Everything else should happen after the component renders.

## The Simple Rule

If your web part can render without the data, don't `await` it in `onInit()`. Move it to component level where users can see loading states and proper error handling.

Your users will thank you for showing them something instead of nothing.

---

**References:**

- [SPFx Basics: Constructor vs. onInit()](https://www.voitanos.io/blog/initialize-sharepoint-framework-components-constructor-oninit/)
- [SharePoint Framework onInit() Promises](https://sharepoint.stackexchange.com/questions/222515/sharepoint-framework-spfx-oninit-promises)
- [Async Render in SPFx Web Parts](https://blog.aterentiev.com/async-render-spfx-web-parts)
- [GitHub Issue: SPFx WebPart Loading Issue](https://github.com/SharePoint/sp-dev-docs/issues/9062)
