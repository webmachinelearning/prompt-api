# [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/)

> 01.  What information does this feature expose,
>      and for what purposes?

This feature exposes two large categories of information:

- The implicit behavior of the underlying language model, in terms of what responses it provides to given inputs.

- The availability information for various capabilities of the API, so that web developers know what capabilities are available in the current browser, and whether using them will require a download or the capability can be used readily.

The privacy implications of both of these are discussed, in general terms, [in the _Writing Assistance APIs_ specification](https://webmachinelearning.github.io/writing-assistance-apis/#privacy), which was written to cover all APIs with similar concerns.

> 02.  Do features in your specification expose the minimum amount of information
>      necessary to implement the intended functionality?

We believe so. It's possible that we could remove the exposure of the download status information. However, it would almost certainly be inferrable via timing side-channels. (I.e., if downloading a language model or fine-tuning is required, then the web developer can observe the creation of the `LanguageModel` object taking longer.)

> 03.  Do the features in your specification expose personal information,
>      personally-identifiable information (PII), or information derived from
>      either?

No. Although it's imaginable that the backing language model could be fine-tuned on PII to give more accurate-to-this-user outputs, we intend to disallow this in the specification.

> 04.  How do the features in your specification deal with sensitive information?

We do not deal with sensitive information.

> 05.  Does data exposed by your specification carry related but distinct
>     information that may not be obvious to users?

It is possible that the multimodal inputs support for this API could pass along metadata, such as image metadata, to the underlying language model. There are cases where this can be useful, so we're unsure whether requiring that the implementation strip such information is the right path.

> 06.  Do the features in your specification introduce state
>      that persists across browsing sessions?

Yes. The downloading of language models, and any collateral necessary to support various options, persists across browsing sessions.

> 07.  Do the features in your specification expose information about the
>      underlying platform to origins?

Possibly. If a browser does not bundle its own models, but instead uses the operating system's functionality, it is possible for a web developer to infer information about such operating system functionality.

> 08.  Does this specification allow an origin to send data to the underlying
>      platform?

Possibly. Again, in the scenario where the model comes from the operating system, such data would pass through OS libraries.

> 09.  Do features in this specification enable access to device sensors?

No.

> 10.  Do features in this specification enable new script execution/loading
>      mechanisms?

No.

> 11.  Do features in this specification allow an origin to access other devices?

No.

> 12.  Do features in this specification allow an origin some measure of control over
>      a user agent's native UI?

No.

> 13.  What temporary identifiers do the features in this specification create or
>      expose to the web?

None.

> 14.  How does this specification distinguish between behavior in first-party and
>      third-party contexts?

We intend to use permissions policy to disallow the usage of these features by default in third-party (cross-origin) contexts. However, the top-level site can delegate to cross-origin iframes.

Otherwise, some of the possible [anti-fingerprinting mitigations](https://webmachinelearning.github.io/writing-assistance-apis/#privacy-availability) involve partitioning information across sites, which is kind of like distinguishing between first- and third-party contexts.

> 15.  How do the features in this specification work in the context of a browser’s
>      Private Browsing or Incognito mode?

One possible area of discussion here is whether backing these APIs with cloud-based models make sense in such modes, or whether they should be disabled.

Otherwise, we do not anticipate any differences.

> 16.  Does this specification have both "Security Considerations" and "Privacy
>      Considerations" sections?

We don't yet have a specification, but when we do, we anticipate it delegating to the corresponding sections in _Writing Assistance APIs_:

* [Privacy considerations](https://webmachinelearning.github.io/writing-assistance-apis/#privacy)
* [Security considerations](https://webmachinelearning.github.io/writing-assistance-apis/#security)

> 17.  Do features in your specification enable origins to downgrade default
>      security protections?

No.

> 18.  What happens when a document that uses your feature is kept alive in BFCache
>      (instead of getting destroyed) after navigation, and potentially gets reused
>      on future navigations back to the document?

Ideally, nothing special should happen. In particular, `LanguageModel` objects should still be usable without interruption after navigating back. We'll need to add web platform tests to confirm this, as it's easy to imagine implementation architectures in which keeping these objects alive while the `Document` is in the back/forward cache is difficult.

(For such implementations, failing to bfcache `Document`s with active `LanguageModel` objects would a simple way of being spec-compliant.)

> 19.  What happens when a document that uses your feature gets disconnected?

The methods of the `LanguageModel` objects will start rejecting with `"InvalidStateError"` `DOMException`s.

> 20.  Does your spec define when and how new kinds of errors should be raised?

We do not yet have a specification, but we anticipate that we will follow the model in the [_Writing Assistance APIs_](https://webmachinelearning.github.io/writing-assistance-apis/). This includes some well-specified and also implementation-defined errors that reveal the limitations of the model. The user agent can instead use an `"UnknownError"` `DOMException` if necessary to protect privacy or security.

> 21.  Does your feature allow sites to learn about the user's use of assistive technology?

No.

> 22.  What should this questionnaire have asked?

Seems fine.
