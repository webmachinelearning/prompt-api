# Explainer for the Prompt API

_This proposal is an early design sketch by the Chrome built-in AI team to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome._

Browsers and operating systems are increasingly expected to gain access to a language model. ([Example](https://developer.chrome.com/docs/ai/built-in), [example](https://blogs.windows.com/windowsdeveloper/2024/05/21/unlock-a-new-era-of-innovation-with-windows-copilot-runtime-and-copilot-pcs/), [example](https://www.apple.com/apple-intelligence/).) Language models are known for their versatility. With enough creative [prompting](https://developers.google.com/machine-learning/resources/prompt-eng), they can help accomplish tasks as diverse as:

* Classification, tagging, and keyword extraction of arbitrary text;
* Helping users compose text, such as blog posts, reviews, or biographies;
* Summarizing, e.g. of articles, user reviews, or chat logs;
* Generating titles or headlines from article contents
* Answering questions based on the unstructured contents of a web page
* Translation between languages
* Proofreading

Although the Chrome built-in AI team is exploring purpose-built APIs for some of these use cases (e.g. [translation](https://github.com/webmachinelearning/translation-api), and perhaps in the future summarization and compose), we are also exploring a general-purpose "prompt API" which allows web developers to prompt a language model directly. This gives web developers access to many more capabilities, at the cost of requiring them to do their own prompt engineering.

Currently, web developers wishing to use language models must either call out to cloud APIs, or bring their own and run them using technologies like WebAssembly and WebGPU. By providing access to the browser or operating system's existing language model, we can provide the following benefits compared to cloud APIs:

* Local processing of sensitive data, e.g. allowing websites to combine AI features with end-to-end encryption.
* Potentially faster results, since there is no server round-trip involved.
* Offline usage.
* Lower API costs for web developers.
* Allowing hybrid approaches, e.g. free users of a website use on-device AI whereas paid users use a more powerful API-based model.

Similarly, compared to bring-your-own-AI approaches, using a built-in language model can save the user's bandwidth, likely benefit from more optimizations, and have a lower barrier to entry for web developers.

**Even more so than many other behind-a-flag APIs, the prompt API is an experiment, designed to help us understand web developers' use cases to inform a roadmap of purpose-built APIs.** However, we want to publish an explainer to provide documentation and a public discussion place for the experiment while it is ongoing.

## Goals

Our goals are to:

* Provide web developers a uniform JavaScript API for accessing browser-provided language models.
* Abstract away specific details of the language model in question as much as possible, e.g. tokenization, system messages, or control tokens.
* Guide web developers to gracefully handle failure cases, e.g. no browser-provided model being available.
* Allow a variety of implementation strategies, including on-device or cloud-based models, while keeping these details abstracted from developers.

The following are explicit non-goals:

* We do not intend to force every browser to ship or expose a language model; in particular, not all devices will be capable of storing or running one. It would be conforming to implement this API by always signaling that no language model is available, or to implement this API entirely by using cloud services instead of on-device models.
* We do not intend to provide guarantees of language model quality, stability, or interoperability between browsers. In particular, we cannot guarantee that the models exposed by these APIs are particularly good at any given use case. These are left as quality-of-implementation issues, similar to the [shape detection API](https://wicg.github.io/shape-detection-api/). (See also a [discussion of interop](https://www.w3.org/reports/ai-web-impact/#interop) in the W3C "AI & the Web" document.)

The following are potential goals we are not yet certain of:

* Allow web developers to know, or control, whether language model interactions are done on-device or using cloud services. This would allow them to guarantee that any user data they feed into this API does not leave the device, which can be important for privacy purposes. Similarly, we might want to allow developers to request on-device-only language models, in case a browser offers both varieties.
* Allow web developers to know some identifier for the language model in use, separate from the browser version. This would allow them to allowlist or blocklist specific models to maintain a desired level of quality, or restrict certain use cases to a specific model.

Both of these potential goals could pose challenges to interoperability, so we want to investigate more how important such functionality is to developers to find the right tradeoff.

## Examples

### Zero-shot prompting

In this example, a single string is used to prompt the API, which is assumed to come from the user. The returned response is from the language model.

```js
const session = await ai.languageModel.create();

// Prompt the model and wait for the whole result to come back.
const result = await session.prompt("Write me a poem.");
console.log(result);

// Prompt the model and stream the result:
const stream = await session.promptStreaming("Write me an extra-long poem.");
for await (const chunk of stream) {
  console.log(chunk);
}
```

### System prompts

The language model can be configured with a special "system prompt" which gives it the context for future interactions:

```js
const session = await ai.languageModel.create({
  systemPrompt: "Pretend to be an eloquent hamster."
});

console.log(await session.prompt("What is your favorite food?"));
```

The system prompt is special, in that the language model will not respond to it, and it will be preserved even if the context window otherwise overflows due to too many calls to `prompt()`.

If the system prompt is too large (see [below](#tokenization-context-window-length-limits-and-overflow)), then the promise will be rejected with a `"QuotaExceededError"` `DOMException`.

### N-shot prompting

If developers want to provide examples of the user/assistant interaction, they can use the `initialPrompts` array. This aligns with the common "chat completions API" format of `{ role, content }` pairs, including a `"system"` role which can be used instead of the `systemPrompt` option shown above.

```js
const session = await ai.languageModel.create({
  initialPrompts: [
    { role: "system", content: "Predict up to 5 emojis as a response to a comment. Output emojis, comma-separated." },
    { role: "user", content: "This is amazing!" },
    { role: "assistant", content: "❤️, ➕" },
    { role: "user", content: "LGTM" },
    { role: "assistant", content: "👍, 🚢" }
  ]
});

// Clone an existing session for efficiency, instead of recreating one each time.
async function predictEmoji(comment) {
  const freshSession = await session.clone();
  return await freshSession.prompt(comment);
}

const result1 = await predictEmoji("Back to the drawing board");

const result2 = await predictEmoji("This code is so good you should get promoted");
```

(Note that merely creating a session does not cause any new responses from the language model. We need to call `prompt()` or `promptStreaming()` to get a response.)

Some details on error cases:

* Using both `systemPrompt` and a `{ role: "system" }` prompt in `initialPrompts`, or using multiple `{ role: "system" }` prompts, or placing the `{ role: "system" }` prompt anywhere besides at the 0th position in `initialPrompts`, will reject with a `TypeError`.
* If the combined token length of all the initial prompts (including the separate `systemPrompt`, if provided) is too large, then the promise will be rejected with a `"QuotaExceededError"` `DOMException`.

### Customizing the role per prompt

Our examples so far have provided `prompt()` and `promptStreaming()` with a single string. Such cases assume messages will come from the user role. These methods can also take in objects in the `{ role, content }` format, or arrays of such objects, in case you want to provide multiple user or assistant messages before getting another assistant message:

```js
const multiUserSession = await ai.languageModel.create({
  systemPrompt: "You are a mediator in a discussion between two departments."
});

const result = await multiUserSession.prompt([
  { role: "user", content: "Marketing: We need more budget for advertising campaigns." },
  { role: "user", content: "Finance: We need to cut costs and advertising is on the list." },
  { role: "assistant", content: "Let's explore a compromise that satisfies both departments." }
]);

// `result` will contain a compromise proposal from the assistant.
```

Because of their special behavior of being preserved on context window overflow, system prompts cannot be provided this way.

### Emulating tool use or function-calling via assistant-role prompts

A special case of the above is using the assistant role to emulate tool use or function-calling, by marking a response as coming from the assistant side of the conversation:

```js
const session = await ai.languageModel.create({
  systemPrompt: `
    You are a helpful assistant. You have access to the following tools:
    - calculator: A calculator. To use it, write "CALCULATOR: <expression>" where <expression> is a valid mathematical expression.
  `
});

async function promptWithCalculator(prompt) {
  const result = await session.prompt(prompt);

  // Check if the assistant wants to use the calculator tool.
  const match = /^CALCULATOR: (.*)$/.exec(result);
  if (match) {
    const expression = match[1];
    const mathResult = evaluateMathExpression(expression);

    // Add the result to the session so it's in context going forward.
    await session.prompt({ role: "assistant", content: mathResult });

    // Return it as if that's what the assistant said to the user.
    return mathResult;
  }

  // The assistant didn't want to use the calculator. Just return its response.
  return result;
}

console.log(await promptWithCalculator("What is 2 + 2?"));
```

We'll likely explore more specific APIs for tool- and function-calling in the future; follow along in [issue #7](https://github.com/webmachinelearning/prompt-api/issues/7).

### Configuration of per-session parameters

In addition to the `systemPrompt` and `initialPrompts` options shown above, the currently-configurable model parameters are [temperature](https://huggingface.co/blog/how-to-generate#sampling) and [top-K](https://huggingface.co/blog/how-to-generate#top-k-sampling). The `params()` API gives the default, minimum, and maximum values for these parameters.

_However, see [issue #42](https://github.com/webmachinelearning/prompt-api/issues/42): sampling hyperparameters are not universal among models._

```js
const customSession = await ai.languageModel.create({
  temperature: 0.8,
  topK: 10
});

const params = await ai.languageModel.params();
const slightlyHighTemperatureSession = await ai.languageModel.create({
  temperature: Math.max(
    params.defaultTemperature * 1.2,
    params.maxTemperature
  ),
  topK: 10
});

// params also contains defaultTopK and maxTopK.
```

If the language model is not available at all in this browser, `params()` will fulfill with `null`.

### Session persistence and cloning

Each language model session consists of a persistent series of interactions with the model:

```js
const session = await ai.languageModel.create({
  systemPrompt: "You are a friendly, helpful assistant specialized in clothing choices."
});

const result = await session.prompt(`
  What should I wear today? It's sunny and I'm unsure between a t-shirt and a polo.
`);

console.log(result);

const result2 = await session.prompt(`
  That sounds great, but oh no, it's actually going to rain! New advice??
`);
```

Multiple unrelated continuations of the same prompt can be set up by creating a session and then cloning it:

```js
const session = await ai.languageModel.create({
  systemPrompt: "You are a friendly, helpful assistant specialized in clothing choices."
});

const session2 = await session.clone();
```

The clone operation can be aborted using an `AbortSignal`:

```js
const controller = new AbortController();
const session2 = await session.clone({ signal: controller.signal });
```

### Session destruction

A language model session can be destroyed, either by using an `AbortSignal` passed to the `create()` method call:

```js
const controller = new AbortController();
stopButton.onclick = () => controller.abort();

const session = await ai.languageModel.create({ signal: controller.signal });
```

or by calling `destroy()` on the session:

```js
stopButton.onclick = () => session.destroy();
```

Destroying a session will have the following effects:

* If done before the promise returned by `create()` is settled:

  * Stop signaling any ongoing download progress for the language model. (The browser may also abort the download, or may continue it. Either way, no further `downloadprogress` events will fire.)

  * Reject the `create()` promise.

* Otherwise:

  * Reject any ongoing calls to `prompt()`.

  * Error any `ReadableStream`s returned by `promptStreaming()`.

* Most importantly, destroying the session allows the user agent to unload the language model from memory, if no other APIs or sessions are using it.

In all cases the exception used for rejecting promises or erroring `ReadableStream`s will be an `"AbortError"` `DOMException`, or the given abort reason.

The ability to manually destroy a session allows applications to free up memory without waiting for garbage collection, which can be useful since language models can be quite large.

### Aborting a specific prompt

Specific calls to `prompt()` or `promptStreaming()` can be aborted by passing an `AbortSignal` to them:

```js
const controller = new AbortController();
stopButton.onclick = () => controller.abort();

const result = await session.prompt("Write me a poem", { signal: controller.signal });
```

Note that because sessions are stateful, and prompts can be queued, aborting a specific prompt is slightly complicated:

* If the prompt is still queued behind other prompts in the session, then it will be removed from the queue.
* If the prompt is being currently processed by the model, then it will be aborted, and the prompt/response pair will be removed from the conversation history.
* If the prompt has already been fully processed by the model, then attempting to abort the prompt will do nothing.

### Tokenization, context window length limits, and overflow

A given language model session will have a maximum number of tokens it can process. Developers can check their current usage and progress toward that limit by using the following properties on the session object:

```js
console.log(`${session.tokensSoFar}/${session.maxTokens} (${session.tokensLeft} left)`);
```

To know how many tokens a string will consume, without actually processing it, developers can use the `countPromptTokens()` method:

```js
const numTokens = await session.countPromptTokens(promptString);
```

Some notes on this API:

* We do not expose the actual tokenization to developers since that would make it too easy to depend on model-specific details.
* Implementations must include in their count any control tokens that will be necessary to process the prompt, e.g. ones indicating the start or end of the input.
* The counting process can be aborted by passing an `AbortSignal`, i.e. `session.countPromptTokens(promptString, { signal })`.

It's possible to send a prompt that causes the context window to overflow. That is, consider a case where `session.countPromptTokens(promptString) > session.tokensLeft` before calling `session.prompt(promptString)`, and then the web developer calls `session.prompt(promptString)` anyway. In such cases, the initial portions of the conversation with the language model will be removed, one prompt/response pair at a time, until enough tokens are available to process the new prompt. The exception is the [system prompt](#system-prompts), which is never removed. If it's not possible to remove enough tokens from the conversation history to process the new prompt, then the `prompt()` or `promptStreaming()` call will fail with an `"QuotaExceededError"` `DOMException` and nothing will be removed.

Such overflows can be detected by listening for the `"contextoverflow"` event on the session:

```js
session.addEventListener("contextoverflow", () => {
  console.log("Context overflow!");
});
```

### Multilingual content and expected languages

The default behavior for a language model session assumes that the input languages are unknown. In this case, implementations will use whatever "base" capabilities they have available for the language model, and might throw `"NotSupportedError"` `DOMException`s if they encounter languages they don't support.

It's better practice, if possible, to supply the `create()` method with information about the expected input languages. This allows the implementation to download any necessary supporting material, such as fine-tunings or safety-checking models, and to immediately reject the promise returned by `create()` if the web developer needs to use languages that the browser is not capable of supporting:

```js
const session = await ai.languageModel.create({
  systemPrompt: `
    You are a foreign-language tutor for Japanese. The user is Korean. If necessary, either you or
    the user might "break character" and ask for or give clarification in Korean. But by default,
    prefer speaking in Japanese, and return to the Japanese conversation once any sidebars are
    concluded.
  `,
  expectedInputLanguages: ["en" /* for the system prompt */, "ja", "kr"]
});
```

Note that there is no way of specifying output languages, since these are governed by the language model's own decisions. Similarly, the expected input languages do not affect the context or prompt the language model sees; they only impact the process of setting up the session and performing appropriate downloads.

### Testing available options before creation

In the simple case, web developers should call `ai.languageModel.create()`, and handle failures gracefully.

However, if the web developer wants to provide a differentiated user experience, which lets users know ahead of time that the feature will not be possible or might require a download, they can use the promise-returning `ai.languageModel.availability()` method. This method lets developers know, before calling `create()`, what is possible with the implementation.

The method will return a promise that fulfills with one of the following availability values:

* "`no`" means that the implementation does not support the requested options, or does not support prompting a language model at all.
* "`after-download`" means that the implementation supports the requested options, but it will have to download something (e.g. the language model itself, or a fine-tuning) before it can create a session using those options.
* "`readily`" means that the implementation supports the requested options without requiring any new downloads.

An example usage is the following:

```js
const options = { expectedInputLanguages: ["en", "es"], temperature: 2 };

const supportsOurUseCase = await ai.languageModel.availability(options);

if (supportsOurUseCase !== "no") {
  if (supportsOurUseCase === "after-download") {
    console.log("Sit tight, we need to do some downloading...");
  }

  const session = await ai.languageModel.create({ ...options, systemPrompt: "..." });
  // ... Use session ...
} else {
  // Either the API overall, or the expected languages and temperature setting, is not available.
  console.error("No language model for us :(");
}
```

### Download progress

In cases where the model needs to be downloaded as part of creation, you can monitor the download progress (e.g. in order to show your users a progress bar) using code such as the following:

```js
const session = await ai.languageModel.create({
  monitor(m) {
    m.addEventListener("downloadprogress", e => {
      console.log(`Downloaded ${e.loaded} of ${e.total} bytes.`);
    });
  }
});
```

If the download fails, then `downloadprogress` events will stop being emitted, and the promise returned by `create()` will be rejected with a "`NetworkError`" `DOMException`.

<details>
<summary>What's up with this pattern?</summary>

This pattern is a little involved. Several alternatives have been considered. However, asking around the web standards community it seemed like this one was best, as it allows using standard event handlers and `ProgressEvent`s, and also ensures that once the promise is settled, the session object is completely ready to use.

It is also nicely future-extensible by adding more events and properties to the `m` object.

Finally, note that there is a sort of precedent in the (never-shipped) [`FetchObserver` design](https://github.com/whatwg/fetch/issues/447#issuecomment-281731850).
</details>

## Detailed design

### Full API surface in Web IDL

```webidl
// Shared self.ai APIs

partial interface WindowOrWorkerGlobalScope {
  [Replaceable, SecureContext] readonly attribute AI ai;
};

[Exposed=(Window,Worker), SecureContext]
interface AI {
  readonly attribute AILanguageModelFactory languageModel;
};

[Exposed=(Window,Worker), SecureContext]
interface AICreateMonitor : EventTarget {
  attribute EventHandler ondownloadprogress;

  // Might get more stuff in the future, e.g. for
  // https://github.com/webmachinelearning/prompt-api/issues/4
};

callback AICreateMonitorCallback = undefined (AICreateMonitor monitor);

enum AICapabilityAvailability { "readily", "after-download", "no" };
```

```webidl
// Language Model

[Exposed=(Window,Worker), SecureContext]
interface AILanguageModelFactory {
  Promise<AILanguageModel> create(optional AILanguageModelCreateOptions options = {});
  Promise<AICapabilityAvailability> availability(optional AILanguageModelCreateCoreOptions options = {});
  Promise<AILanguageModelInfo?> params();
};

[Exposed=(Window,Worker), SecureContext]
interface AILanguageModel : EventTarget {
  Promise<DOMString> prompt(AILanguageModelPromptInput input, optional AILanguageModelPromptOptions options = {});
  ReadableStream promptStreaming(AILanguageModelPromptInput input, optional AILanguageModelPromptOptions options = {});

  Promise<unsigned long long> countPromptTokens(AILanguageModelPromptInput input, optional AILanguageModelPromptOptions options = {});
  readonly attribute unsigned long long maxTokens;
  readonly attribute unsigned long long tokensSoFar;
  readonly attribute unsigned long long tokensLeft;

  readonly attribute unsigned long topK;
  readonly attribute float temperature;
  readonly attribute FrozenArray<DOMString>? expectedInputLanguages;

  attribute EventHandler oncontextoverflow;

  Promise<AILanguageModel> clone(optional AILanguageModelCloneOptions options = {});
  undefined destroy();
};

[Exposed=(Window,Worker), SecureContext]
interface AILanguageModelParams {
  readonly attribute unsigned long defaultTopK;
  readonly attribute unsigned long maxTopK;
  readonly attribute float defaultTemperature;
  readonly attribute float maxTemperature;
};

dictionary AILanguageModelCreateCoreOptions {
  [EnforceRange] unsigned long topK;
  float temperature;
  sequence<DOMString> expectedInputLanguages;
}

dictionary AILanguageModelCreateOptions : AILanguageModelCreateCoreOptions {
  AbortSignal signal;
  AICreateMonitorCallback monitor;

  DOMString systemPrompt;
  sequence<AILanguageModelInitialPrompt> initialPrompts;
};

dictionary AILanguageModelInitialPrompt {
  required AILanguageModelInitialPromptRole role;
  required DOMString content;
};

dictionary AILanguageModelPrompt {
  required AILanguageModelPromptRole role;
  required DOMString content;
};

dictionary AILanguageModelPromptOptions {
  AbortSignal signal;
};

dictionary AILanguageModelCloneOptions {
  AbortSignal signal;
};

typedef (DOMString or AILanguageModelPrompt or sequence<AILanguageModelPrompt>) AILanguageModelPromptInput;

enum AILanguageModelInitialPromptRole { "system", "user", "assistant" };
enum AILanguageModelPromptRole { "user", "assistant" };
```

### Instruction-tuned versus base models

We intend for this API to expose instruction-tuned models. Although we cannot mandate any particular level of quality or instruction-following capability, we think setting this base expectation can help ensure that what browsers ship is aligned with what web developers expect.

To illustrate the difference and how it impacts web developer expectations:

* In a base model, a prompt like "Write a poem about trees." might get completed with "... Write about the animal you would like to be. Write about a conflict between a brother and a sister." (etc.) It is directly completing plausible next tokens in the text sequence.
* Whereas, in an instruction-tuned model, the model will generally _follow_ instructions like "Write a poem about trees.", and respond with a poem about trees.

To ensure the API can be used by web developers across multiple implementations, all browsers should be sure their models behave like instruction-tuned models.

## Alternatives considered and under consideration

### How many stages to reach a response?

To actually get a response back from the model given a prompt, the following possible stages are involved:

1. Download the model, if necessary.
2. Establish a session, including configuring per-session options and parameters.
3. Add an initial prompt to establish context. (This will not generate a response.)
4. Execute a prompt and receive a response.

We've chosen to manifest these 3-4 stages into the API as two methods, `ai.languageModel.create()` and `session.prompt()`/`session.promptStreaming()`, with some additional facilities for dealing with the fact that `ai.languageModel.create()` can include a download step. Some APIs simplify this into a single method, and some split it up into three (usually not four).

### Stateless or session-based

Our design here uses [sessions](#session-persistence-and-cloning). An alternate design, seen in some APIs, is to require the developer to feed in the entire conversation history to the model each time, keeping track of the results.

This can be slightly more flexible; for example, it allows manually correcting the model's responses before feeding them back into the context window.

However, our understanding is that the session-based model can be more efficiently implemented, at least for browsers with on-device models. (Implementing it for a cloud-based model would likely be more work.) And, developers can always achieve a stateless model by using a new session for each interaction.

## Privacy considerations

If cloud-based language models are exposed through this API, then there are potential privacy issues with exposing user or website data to the relevant cloud and model providers. This is not a concern specific to this API, as websites can already choose to expose user or website data to other origins using APIs such as `fetch()`. However, it's worth keeping in mind, and in particular as discussed in our [Goals](#goals), perhaps we should make it easier for web developers to know whether a cloud-based model is in use, or which one.

If on-device language models are updated separately from browser and operating system versions, this API could enhance the web's fingerprinting service by providing extra identifying bits. Mandating that older browser versions not receive updates or be able to download models from too far into the future might be a possible remediation for this.

Finally, we intend to prohibit (in the specification) any use of user-specific information that is not directly supplied through the API. For example, it would not be permissible to fine-tune the language model based on information the user has entered into the browser in the past.

## Stakeholder feedback

* W3C TAG: not yet requested
* Browser engines and browsers:
  * Chromium: prototyping behind a flag
  * Gecko: not yet requested
  * WebKit: not yet requested
  * Edge: not yet requested
* Web developers: positive ([example](https://x.com/mortenjust/status/1805190952358650251), [example](https://tyingshoelaces.com/blog/chrome-ai-prompt-api), [example](https://labs.thinktecture.com/local-small-language-models-in-the-browser-a-first-glance-at-chromes-built-in-ai-and-prompt-api-with-gemini-nano/))
