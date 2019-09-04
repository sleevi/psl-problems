
# Public Suffix List Problems
Ryan Sleevi, September 2019

_©2019, Google, Inc. All rights reserved._

_(This is a collection of thoughts from a maintainer of the [Public Suffix List](https://publicsuffix.org/) (PSL) about the importance of avoiding new Web Platform features, security, or privacy boundaries assuming the PSL is a good starting point. This isn't anything stamped with the Google Seal of Approval or Official Policy, but comes up enough in design reviews to be worth publicly documenting and sharing.)_

## **Public Suffix List: Past to Present**

In the beginning was the cookie, and it was good. It was a time when the root zone was small, cookies were simple, and small furry creatures from Alpha Centauri were real small furry creatures from Alpha Centauri. With the exception that none of this was ever true.

When cookies were first introduced, the idea was simple: anybody who registered a domain could take advantage of this hot new storage/persistence layer. If you registered `example.com`, then you could do whatever you wanted and set whatever cookie you wanted, for any host in the `example.com` domain namespace - because it was your domain. This worked for the few generic TLDs (gTLDs) that were set up by [RFC 920](https://tools.ietf.org/html/rfc920), but was known to break down when it came to country code TLDs (ccTLDs), that did what they wanted and made their own rules. Whether this was `.us` which divided into states and cities, or `.uk`, which had its own subdivisions mirroring the gTLDs, such as `.co.uk` and `.net.uk`. 

This divergence between the gTLD and ccTLD set led to browsers to implement a [set of heuristics in order to mitigate security issues](https://bugzilla.mozilla.org/show_bug.cgi?id=252342), which were [originally introduced in the cookie spec](https://web.archive.org/web/19970101073348/http://www.netscape.com/newsref/std/cookie_spec.html). However, the fundamental assumption, at the time of introduction, was that the party who registered the domain was in control of all content that appeared on their domain, and below it. The cookie boundary was not even seen as a [security or privacy boundary](https://bugzilla.mozilla.org/show_bug.cgi?id=8743#c5) - it was merely indicative of a misconfigured or spammy server causing more work for itself.

Originally, this logic was only constrained to cookies, and implemented directly within the cookie parsing algorithm, as specified both by the Netscape spec and the later-proposed [RFC 2109](https://tools.ietf.org/html/rfc2109). As part of the work for Firefox 3, which introduced Places, Mozilla worked to [generalize how it handled recording the logic for ccTLDs](https://bugzilla.mozilla.org/show_bug.cgi?id=319643), introducing with it the notion of the “effective TLD.” By extracting the logic from the cookie processing algorithm itself, this allowed Mozilla to group cookies and other settings across more easily managed logical boundaries, without having to duplicate the logic or come up with new heuristics. The “effective TLD” took into consideration ccTLD’s registration policies, which were painstakingly [determined over the span of several months](https://bugzilla.mozilla.org/show_bug.cgi?id=342314), and which ultimately gave birth to what we now know as the [Public Suffix List (PSL)](https://publicsuffix.org).

Parallel to this work, the notion of the Same-Origin Policy (SOP) was evolving. Michal Zalewski’s [Browser Security Handbook](https://code.google.com/archive/p/browsersec/wikis/Part2.wiki#Same-origin_policy) best captures this evolution, which was not a single policy, but a number of similar policies, such as around `document.domain`, `XMLHttpRequest`, and various plugins. Unlike cookies, which considered the boundary to be the “registered domain,” the SOP focused specifically on the combination of scheme, host, and port being an “exact” match<sup>[1](#footnote1)</sup>.

Over the years, the divergence between how the SOP is applicable in different contexts has been reduced, such that there are, effectively, two main security boundaries in the Web today: the SOP boundary (e.g. as used by fetch()/XMLHttpRequest/CORS) and the cookie boundary (quasi-shared with `document.domain`). As a consequence, browser developers who want to introduce new features are faced with a question: Should that feature use the origin as the boundary (implying the SOP) or should it use the “registrable domain” as the boundary (implying the PSL)?

## Problems: Solutions Later

The PSL tends to be suggested when trying to solve questions of _attribution_ - who is the responsible party for the content and what it does?

*   It comes up in the context of **security**: is it safe to allow `customer1.example.com` to influence `customer2.example.com`?
*   It comes up in the context of **privacy**: should `customer1.example.com` know about `customer2.example.com`?
*   It comes up in the context of **usability**: When the user accesses `customer1.example.com`, is the user’s expectation that they are dealing with `customer1` or `example.com`?
*   It comes up in the context of **resource management**: given finite resources, such as bandwidth, storage, or CPU, should `customer1.example.com` share resource limits with `customer2.example.com`?

The problem, however, is that the Public Suffix List is an ineffective solution for each of these questions, with real and damaging tradeoffs.

When it comes to **security**, the PSL has a problem in that its default assumption is that all subdomains within a given domain share the same security boundary. Absent an entry on the Public Suffix List, the default is that it will “fail open” and allow aggregation on the nearest boundary, which will “hopefully” be a realistic registrable domain. For example, when App Engine was introduced, it was necessary to add `appspot.com` to the PSL, in order to ensure that `foo.appspot.com` would not be able to access the cookies of `bar.appspot.com`. This approach to management puts the burden on domain holders, by starting off as a **default insecure** configuration. Because the PSL is, in all major browsers (and programming languages and clients), a static list, this means domain holders can’t offer new services “securely” until all existing copies in the ecosystem have updated to recognize their service. This is bad for the Web: **_we want new features to be default secure_**.

When it comes to **privacy**, the same default assumptions as security bite here. The default assumption is to assume “sameness” at the registrable domain level. This means any new use of subdomains requires care to get privacy right, and even then, **can’t be assured** because of the update lag.

Worse, however, is when services don’t realize this, and thus have customers on the same subdomains as their ‘production’ domains. The same assumptions the browser is making that allows `login.example.com` to seamlessly work with `www.example.com` ends up allowing `customer.example.com` full access! The site operator that wants to protect against this, by adding themselves to the PSL, has to make a choice: Do they break their customer URLs or their URLs?

In addition to breaking site operators’ expectations, it also breaks browsers’ expectations. The PSL is commonly used to infer organizational boundaries; if there is “sameness” of the domain, it must be “first party” or the same organization, while if it’s a different domain, it must be a different organization, or “third party”. However, these assumptions break down, because there’s no guarantee that `login.example.com` and `logs.example.com` are the same entity. If the site operator is presumed malicious and trying to exchange information, it’s trivial for them to configure a CNAME such that `logs.example.com` points to `logging.example`, bypassing the normal restrictions that might apply to loading `logging.example` from `login.example.com`, because, to the browser, both `login.example.com` and `logs.example.com` are operated by `example.com`.

Recall that the PSL was originally introduced to solve a **usability** problem, by attempting to group settings and cookies for registrable domains together into a more manageable interface. It can be tempting to use that same logic in other places, to try and make the settings more accessible. The problem is that the default aggregation algorithm doesn’t align with the recommendations to [use the origin as the basis for security decisions](https://chromium.googlesource.com/chromium/src/+/master/docs/security/url_display_guidelines/url_display_guidelines.md), because it ends up collapsing disparate entities into one interface. It also leaves it ambiguous how to solve for hosting providers - should `appspot.com` be collapsed into a single entry, or should it display for all of them?

It’s tempting to use the PSL for **resource management** purposes, such as quotas or process limits. However, this approach either penalizes domain holders or is easily circumvented. Similar to the usability question, it opens a hard question about whether the limit should be based on `appspot.com` or its subdomains, each potentially operated by different entities.

If you limit based on the subdomains, **adversaries can easily bypass any limits** by simply adding themselves to the PSL and claiming different entities control their subdomains, even if that’s not the case. For example, if each registrable domain gets a 5KB quota for a given object, then the adversary just says that `evil.example` is a public suffix, allowing them to get unlimited quota by simply minting subdomains. Yet the alternative, to apply the limit based on the TLD settings, penalizes the hosting provider `good.example` by causing all of their customers to be limited to an ever decreasing slice of that 5KB pie.

Beyond these issues, though, there’s a more generalized issue: Since the Public Suffix List has [expanded beyond simply ccTLD policies](https://hg.mozilla.org/mozilla-central/rev/b7c6f08e35b3a8447270097ab916b19fe00b31fc) and the introduction ICANN gTLD program, growth of the list is no longer bounded by the number of countries in the world and how they might administer their ccTLDs. **The list will now grow indefinitely** with the number of new domains introduced and their desire to add subdomains. This creates a real problem, as there is some limit to be reached in which the utility of inclusion is outweighed by the cost to end users, in terms of memory and bandwidth. At some point, it will need to be replaced, and **the best way to avoid issues with that is to avoid the use of the list itself**.

Consider, for example, when Let’s Encrypt introduced rate limits based on the eTLD. The PSL saw an incredible and sustained growth in pull requests, as domain holders tried to get around those limits. Some of these may be legitimate, but others were not, highlighting how the PSL is ineffective at distinguishing good use cases from bad. Equally terrifying, however, is how many providers only discovered the existence of the PSL once LE was using it to rate limit - meaning that their users were able to influence cookies and other storage without restriction, until an incidental change (wanting to get more certs) caused the server operator to realize.

Beyond this growth, however, there is an issue that, unlike ccTLDs, which are “forever” (for some value of forever), and gTLDs, which are expected to be a bit more stable than they are today, domains themselves experience high churn, and **additions to the PSL aren’t bounded in time or continuously checked for applicability**. Functionally, this means that the growth rate of the PSL is unbounded, because of the potential harms caused by removing or culling existing entries.

Consider, for example, that `operaunite.com`, the first domain added to the PSL, is no longer an Opera service offering. Can it be removed from the PSL? For the PSL maintainers, it’s unknown; the domain is still registered, and Opera hasn’t requested removal, so should the fact that it’s no longer being used for new domains justify removing it? If Opera did request removal, and it were removed, what are the implications to browsers and software which previously recognized it as an eTLD? For example, if the browser had local state or cookies associated, what are the impacts if it was removed?

In practice, it’s impossible for the PSL maintainers to reason about the implications of removal, even in the presence of explicit requests from the domain holder, and these problems are even messier when contemplating domain expirations or transfers, which may or may not be safe to remove or possible to reliably authenticate.

This isn’t easy to retrofit on either - despite nearly a decade of engagement with ICANN, ccTLDs aren’t inherently bound in the way gTLDs are, and already go against best practices, so we’re unlikely to get ccTLDs to adopt a new scheme. And any change that affects (and potentially removes) the private domains can have security consequences to the user and be undetected by the domain operator - **meaning setting new policies would still have to grandfather in significant existing entries.**

## FAQ


### Should I use the Public Suffix List/eTLD+1 for ...

The answer is **no**. For anything new, you should **avoid the Public Suffix List**.

### Wait, I wasn’t finished!

That’s not a question! However, the use case of the Public Suffix List has traditionally fallen on one of three dimensions: trying to solve for **privacy**, **security**, or **usability**. The problem is that it under-delivers and over-promises on all three of these dimensions, leading to privacy, security, or usability _issues_, often worse than the ones it was trying to resolve.

### So what am I supposed to use instead?

In general, you’ll have far fewer security and privacy issues if you adopt the Same Origin Policy instead. While more restrictive, the consistency with the existing Web Platform, particularly Javascript, is far more desirable, in that it simplifies reasoning about the feature and any of its interactions with the Web Platform.


### Developers don’t like how restrictive the SOP is. Are you sure it's the right idea?

It’s true, the SOP is a mighty hammer to weild, and it’s far more restrictive than simply eTLD+1. As it’s used today, eTLD+1 is often trying to be a shorthand for “associated with the same organization”, and alternative expressions of such associations (such as explored by DBOUND or First Party Sets) may be stepping stones towards more flexible expressions. However, using eTLD+1 to try and approximate that does not work, because it defaults to an insecure state of assuming different origins are related, requiring sites to opt-out in order to maintain security or privacy boundaries, and can be easily circumvented through adding to the PSL or through the use of CNAMEs to bypass any intended restrictions.


### What are we going to do about cookies then?

Hope for the best, and that clever folks can find a path to deprecating cookies’ big assumption? It’s important to note that the cookie security policy already is a huge security and privacy gap, long known, and that provides greater incentives for more daring fixes to the platform. We need to figure out what the happy path looks like - how to support existing use cases, but with a more declarative syntax or safer defaults. The [HTTP Stake Tokens](https://mikewest.github.io/http-state-tokens/draft-west-http-state-tokens.html) explores one possible long-term effort, by fully replacing cookies. Alternatively, declarative syntaxes like [First Party Sets](https://github.com/mikewest/first-party-sets) might pave a path to deprecating cookies’ Domain attribute, while still facilitating the use cases that it enables.


### Are you really saying the strategy is “hope?”

Maybe? Rome wasn’t built in a day, and this is definitely not an easy task. Getting rid of the dependencies on the Public Suffix List, and breaking some of the assumptions, is a long-term effort with no easy answers. However, it’s only made harder when new features are built on assuming that the registrable domain is a reasonable boundary. That’s why it’s so important to avoid that for new features, and to make sure we’re dealing with the core security/privacy/usability issues, rather than papering them over. Declarative expressions of relationships, rather than implicitly assuming based on the domain hierarchy, better reflects the world as it is.

### If that doesn’t work?

Even if the Web Platform is never able to fully excise cookies-as-they-are, there is still considerable complexity in having new features depend on the PSL. No matter how “cookie-like” a thing is, using the PSL for a new feature makes it hard to reason about the interactions and relationship with the Same Origin Policy.

### Are you sure there are no good uses for the Public Suffix List?

Much of this document is focused on aspects specific to the browser and the Web Platform. While cookies straddle that line — being most commonly experienced in the context of the Web Platform, but also possible to use in purely HTTP-based exchanges (like APIs) — the fact that they have one (very large) foot in the Web means we have to do better.

However, other uses may not touch on the browser at all, and it might be reasonable to use the PSL. For example, some use cases make use of the PSL to determine whether or not to ‘linkify’ a given string, like `my.simple.example`. Strictly speaking, what these use cases are after is having a list of all valid TLDs - the Root Zone Database - and it just so happens that the PSL has a full copy of the RZD within it. Switching to _just_ the Root Zone Database would be far more preferable here, but using the PSL is not all doom and gloom compared to the Web.

### How am I supposed to figure out reputation for a domain?

Another use of the PSL in non-browser cases tends to be about associating reputation. For example, if malicious content is found on a site, what’s the appropriate level to show and display a warning — for that origin, or for all domains associated with that entity?

This is functionally an example of the **resource management** issue, which adversaries can game by adding themselves to the list, thus causing it to seem as if domains are different operating entities when they’re the same, but is crucial for having ‘good’ actors, such as `amazonaws.com` or `appspot.com` highlight/disclaim responsibility. It’s been known that the PSL syntax is not expressive enough to handle how entities manage their domains today, such as Amazon and S3 buckets, but it’s what people are used to. The solutions here are going to vary based on the specific problem, but solutions like [DNS-based assertions might be more viable](https://www.ietf.org/mailman/listinfo/dbound) and suited for the task. 


## Notes

<a name="footnote1">1</a>: This is slightly fuzzy, for example, when taking to account that the default port may be omitted from the URL but still be semantically equivalent.
