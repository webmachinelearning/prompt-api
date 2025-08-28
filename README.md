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

The Chrome built-in AI team and the Web Machine Learning Community Group are exploring purpose-built APIs for some of these use cases (namely [translator / language detector](https://github.com/webmachinelearning/translation-api), [summarizer / writer / rewriter](https://github.com/webmachinelearning/writing-assistance-apis), and [proofreader](https://github.com/webmachinelearning/proposals/issues/7)). This proposal additionally exploring a general-purpose "prompt API" which allows web developers to prompt a language model directly. This gives web developers access to many more capabilities, at the cost of requiring them to do their own prompt engineering.

Currently, web developers wishing to use language models must either call out to cloud APIs, or bring their own and run them using technologies like WebAssembly and WebGPU. By providing access to the browser or operating system's existing language model, we can provide the following benefits compared to cloud APIs:

* Local processing of sensitive data, e.g. allowing websites to combine AI features with end-to-end encryption.
* Potentially faster results, since there is no server round-trip involved.
* Offline usage.
* Lower API costs for web developers.
* Allowing hybrid approaches, e.g. free users of a website use on-device AI whereas paid users use a more powerful API-based model.

Similarly, compared to bring-your-own-AI approaches, using a built-in language model can save the user's bandwidth, likely benefit from more optimizations, and have a lower barrier to entry for web developers.

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
const session = await LanguageModel.create();

// Prompt the model and wait for the whole result to come back.
const result = await session.prompt("Write me a poem.");
console.log(result);

// Prompt the model and stream the result:
const stream = session.promptStreaming("Write me an extra-long poem.");
for await (const chunk of stream) {
  console.log(chunk);
}
```

### System prompts

The language model can be configured with a special "system prompt" which gives it the context for future interactions. This is done using the `initialPrompts` option and the "chat completions API" `{ role, content }` format, which are expanded upon in [the following section](#n-shot-prompting).

```js
const session = await LanguageModel.create({
  initialPrompts: [{ role: "system", content: "Pretend to be an eloquent hamster." }]
});

console.log(await session.prompt("What is your favorite food?"));
```

The system prompt is special, in that the language model will not respond to it, and it will be preserved even if the context window otherwise overflows due to too many calls to `prompt()`.

If the system prompt is too large, then the promise will be rejected with a `QuotaExceededError` exception. See [below](#tokenization-context-window-length-limits-and-overflow) for more details on token counting and this new exception type.

### N-shot prompting

If developers want to provide examples of the user/assistant interaction, they can add more entries to the `initialPrompts` array, using the `"user"` and `"assistant"` roles:

```js
const session = await LanguageModel.create({
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

* Placing the `{ role: "system" }` prompt anywhere besides at the 0th position in `initialPrompts` will reject with a `TypeError`.
* If the combined token length of all the initial prompts is too large, then the promise will be rejected with a [`QuotaExceededError` exception](#tokenization-context-window-length-limits-and-overflow).

### Customizing the role per prompt

Our examples so far have provided `prompt()` and `promptStreaming()` with a single string. Such cases assume messages will come from the user role. These methods can also take arrays of objects in the `{ role, content }` format, in case you want to provide multiple user or assistant messages before getting another assistant message:

```js
const multiUserSession = await LanguageModel.create({
  initialPrompts: [{
    role: "system",
    content: "You are a mediator in a discussion between two departments."
  }]
});

const result = await multiUserSession.prompt([
  { role: "user", content: "Marketing: We need more budget for advertising campaigns." },
  { role: "user", content: "Finance: We need to cut costs and advertising is on the list." },
  { role: "assistant", content: "Let's explore a compromise that satisfies both departments." }
]);

// `result` will contain a compromise proposal from the assistant.
```

Because of their special behavior of being preserved on context window overflow, system prompts cannot be provided this way.

### Tool use

The Prompt API supports **tool use** via the `tools` option, allowing you to define external capabilities that a language model can invoke in a model-agnostic way. Each tool is represented by an object that includes an `execute` member that specifies the JavaScript function to be called. When the language model initiates a tool use request, the user agent calls the corresponding `execute` function and sends the result back to the model.

Here’s an example of how to use the `tools` option:

```js
const session = await LanguageModel.create({
  initialPrompts: [
    {
      role: "system",
      content: `You are a helpful assistant. You can use tools to help the user.`
    }
  ],
  tools: [
    {
      name: "getWeather",
      description: "Get the weather in a location.",
      inputSchema: {
        type: "object",
        properties: {
          location: {
            type: "string",
            description: "The city to check for the weather condition.",
          },
        },
        required: ["location"],
      },
      async execute({ location }) {
        const res = await fetch("https://weatherapi.example/?location=" + location);
        // Returns the result as a JSON string.
        return JSON.stringify(await res.json());
      },
    }
  ]
});

const result = await session.prompt("What is the weather in Seattle?");
```

In this example, the `tools` array defines a `getWeather` tool, specifying its name, description, input schema, and `execute` implementation. When the language model determines that a tool call is needed, the user agent invokes the `getWeather` tool's `execute()` function with the provided arguments and returns the result to the model, which can then incorporate it into its response.

#### Concurrent tool use

Developers should be aware that the model might call their tool multiple times, concurrently. For example, code such as

```js
const result = await session.prompt("Which of these locations currently has the highest temperature? Seattle, Tokyo, Berlin");
```

might call the above `"getWeather"` tool's `execute()` function three times. The model would wait for all tool call results to return, using the equivalent of `Promise.all()` internally, before it composes its final response.

Similarly, the model might call multiple different tools, if it believes they all are relevant when responding to the given prompt.

### Multimodal inputs

All of the above examples have been of text prompts. Some language models also support other inputs. Our design initially includes the potential to support images and audio clips as inputs. This is done by using objects in the form `{ type: "image", content }` and `{ type: "audio", content }` instead of strings. The `content` values can be the following:

* For image inputs: [`ImageBitmapSource`](https://html.spec.whatwg.org/#imagebitmapsource), i.e. `Blob`, `ImageData`, `ImageBitmap`, `VideoFrame`, `OffscreenCanvas`, `HTMLImageElement`, `SVGImageElement`, `HTMLCanvasElement`, or `HTMLVideoElement` (will get the current frame). Also raw bytes via `BufferSource` (i.e. `ArrayBuffer` or typed arrays).

* For audio inputs: for now, `Blob`, `AudioBuffer`, or raw bytes via `BufferSource`. Other possibilities we're investigating include `HTMLAudioElement`, `AudioData`, and `MediaStream`, but we're not yet sure if those are suitable to represent "clips": most other uses of them on the web platform are able to handle streaming data.

Sessions that will include these inputs need to be created using the `expectedInputs` option, to ensure that any necessary downloads are done as part of session creation, and that if the model is not capable of such multimodal prompts, the session creation fails. (See also the below discussion of [expected input languages](#multilingual-content-and-expected-input-languages), not just expected input types.)

A sample of using these APIs:

```js
const session = await LanguageModel.create({
  // { type: "text" } is not necessary to include explicitly, unless
  // you also want to include expected input languages for text.
  expectedInputs: [
    { type: "audio" },
    { type: "image" }
  ]
});

const referenceImage = await (await fetch("/reference-image.jpeg")).blob();
const userDrawnImage = document.querySelector("canvas");

const response1 = await session.prompt([{
  role: "user",
  content: [
    { type: "text", value: "Give a helpful artistic critique of how well the second image matches the first:" },
    { type: "image", value: referenceImage },
    { type: "image", value: userDrawnImage }
  ]
}]);

console.log(response1);

const audioBlob = await captureMicrophoneInput({ seconds: 10 });

const response2 = await session.prompt([{
  role: "user",
  content: [
    { type: "text", value: "My response to your critique:" },
    { type: "audio", value: audioBlob }
  ]
}]);
```

Note how once we move to multimodal prompting, the prompt format becomes more explicit:

* We must always pass an array of messages, instead of a single string value.
* Each message must have a `role` property: unlike with the string shorthand, `"user"` is no longer assumed.
* The `content` property must be an array of content, if it contains any multimodal content.

This extra ceremony is necessary to make it clear that we are sending a single message that contains multimodal content, versus sending multiple messages, one per each piece of content. To avoid such confusion, the multimodal format has fewer defaults and shorthands than if you interact with the API using only text. (See some discussion in [issue #89](https://github.com/webmachinelearning/prompt-api/pull/89).)

To illustrate, the following extension of our above [multi-user example](#customizing-the-role-per-prompt) has a similar sequence of text + image + image values compared to our artistic critique example. However, it uses a multi-message structure instead of the artistic critique example's single-message structure, so the model will interpret it differently:

```js
const response = await session.prompt([
  {
    role: "user",
    content: "Your compromise just made the discussion more heated. The two departments drew up posters to illustrate their strategies' advantages:"
  },
  {
    role: "user",
    content: [{ type: "image", value: brochureFromTheMarketingDepartment }]
  },
  {
    role: "user",
    content: [{ type: "image", value: brochureFromTheFinanceDepartment }]
  }
]);
```

Details:

* Cross-origin data that has not been exposed using the `Access-Control-Allow-Origin` header cannot be used with the prompt API, and will reject with a `"SecurityError"` `DOMException`. This applies to `HTMLImageElement`, `SVGImageElement`, `HTMLVideoElement`, `HTMLCanvasElement`, and `OffscreenCanvas`. Note that this is more strict than `createImageBitmap()`, which has a tainting mechanism which allows creating opaque image bitmaps from unexposed cross-origin resources. For the prompt API, such resources will just fail. This includes attempts to use cross-origin-tainted canvases.

* Raw-bytes cases (`Blob` and `BufferSource`) will apply the appropriate sniffing rules ([for images](https://mimesniff.spec.whatwg.org/#rules-for-sniffing-images-specifically), [for audio](https://mimesniff.spec.whatwg.org/#rules-for-sniffing-audio-and-video-specifically)) and reject with an `"EncodingError"` `DOMException` if the format is not supported or there is some error decoding the data. This behavior is similar to that of `createImageBitmap()`.

* Animated images will be required to snapshot the first frame (like `createImageBitmap()`). In the future, animated image input may be supported via some separate opt-in, similar to video clip input. But we don't want interoperability problems from some implementations supporting animated images and some not, in the initial version.

* For `HTMLVideoElement`, even a single frame might not yet be downloaded when the prompt API is called. In such cases, calling into the prompt API will force at least a single frame's worth of video to download. (The intent is to behave the same as `createImageBitmap(videoEl)`.)

* Attempting to supply an invalid combination, e.g. `{ type: "audio", value: anImageBitmap }`, `{ type: "image", value: anAudioBuffer }`, or `{ type: "text", value: anArrayBuffer }`, will reject with a `TypeError`.

* For now, using the `"assistant"` role with an image or audio prompt will reject with a `"NotSupportedError"` `DOMException`. (As we explore multimodal outputs, this restriction might be lifted in the future.)

Future extensions may include more ambitious multimodal inputs, such as video clips, or realtime audio or video. (Realtime might require a different API design, more based around events or streams instead of messages.)

### Structured output with JSON schema or RegExp constraints

To help with programmatic processing of language model responses, the prompt API supports constraining the response with either a JSON schema object or a `RegExp` passed as the `responseConstraint` option:

```js
const schema = {
  type: "object",
  required: ["rating"],
  additionalProperties: false,
  properties: {
    rating: {
      type: "number",
      minimum: 0,
      maximum: 5,
    },
  },
};

// Prompt the model and wait for the JSON response to come back.
const result = await session.prompt("Summarize this feedback into a rating between 0-5: "+
  "The food was delicious, service was excellent, will recommend.",
  { responseConstraint: schema }
);

const { rating } = JSON.parse(result);
console.log(rating);
```

If the input value is a valid JSON schema object, but uses JSON schema features not supported by the user agent, the method will error with a `"NotSupportedError"` `DOMException`.

The result value returned is a string that can be parsed with `JSON.parse()`. If the user agent is unable to produce a response that is compliant with the schema, the method will error with a `"SyntaxError"` `DOMException`.

```js
const emailRegExp = /^[a-zA-Z0-9.!#$%&'*+\/=?^_`{|}~-]+@[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$/;

const emailAddress = await session.prompt(
  `Create a fictional email address for ${characterName}.`,
  { responseConstraint: emailRegExp }
);

console.log(emailAddress);
```

The returned value will be a string that matches the input `RegExp`. If the user agent is unable to produce a response that matches, the method will error with a `"SyntaxError"` `DOMException`.

If a value that is neither a `RegExp` object or a valid JSON schema object is given, the method will error with a `TypeError`.

By default, the implementation may include the schema or regular expression as part of the message sent to the underlying language model, which will use up some of the [input quota](#tokenization-context-window-length-limits-and-overflow). You can measure how much it will use up by passing the `responseConstraint` option to `session.measureInputUsage()`. If you want to avoid this behavior, you can use the `omitResponseConstraintInput` option. In such cases, it's strongly recommended to include some guidance in the prompt string itself:

```js
const result = await session.prompt(`
  Summarize this feedback into a rating between 0-5, only outputting a JSON
  object { rating }, with a single property whose value is a number:
  The food was delicious, service was excellent, will recommend.
`, { responseConstraint: schema, omitResponseConstraintInput: true });
```

If `omitResponseConstraintInput` is set to `true` without `responseConstraint` set, then the method will error with a `TypeError`.

### Constraining responses by providing a prefix

As discussed in [Customizing the role per prompt](#customizing-the-role-per-prompt), it is possible to prompt the language model to add a new `"assistant"`-role response in addition to a previous one. Usually it will elaborate on its previous messages. For example:

```js
const followup = await session.prompt([
  {
    role: "user",
    content: "I'm nervous about my presentation tomorrow"
  },
  {
    role: "assistant"
    content: "Presentations are tough!"
  }
]);

// `followup` might be something like "Here are some tips for staying calm.", or
// "I remember my first presentation, I was nervous too!" or...
```

In some cases, instead of asking for a new response message, you want to "prefill" part of the `"assistant"`-role response message. An example use case is to guide the language model toward specific response formats. To do this, add `prefix: true` to the trailing `"assistant"`-role message. For example:

```js
const characterSheet = await session.prompt([
  {
    role: "user",
    content: "Create a TOML character sheet for a gnome barbarian"
  },
  {
    role: "assistant",
    content: "```toml\n",
    prefix: true
  }
]);
```

(Such examples work best if we also support [stop sequences](https://github.com/webmachinelearning/prompt-api/issues/44); stay tuned for that!)

Without this continuation, the output might be something like "Sure! Here's a TOML character sheet...". Whereas the prefix message sets the assistant on the right path immediately.

(Kudos to the [Standard Completions project](https://standardcompletions.org/) for [discussion](https://github.com/standardcompletions/rfcs/pull/8) of this functionality, as well as [the example](https://x.com/stdcompletions/status/1928565134080778414).)

If `prefix` is used in any message besides a final `"assistant"`-role one, a `"SyntaxError"` `DOMException` will occur.

### Appending messages without prompting for a response

In some cases, you know which messages you'll want to use to populate the session, but not yet the final message before you prompt the model for a response. Because processing messages can take some time (especially for multimodal inputs), it's useful to be able to send such messages to the model ahead of time. This allows it to get a head-start on processing, while you wait for the right time to prompt for a response.

(The `initialPrompts` array serves this purpose at session creation time, but this can be useful after session creation as well, as we show in the example below.)

For such cases, in addition to the `prompt()` and `promptStreaming()` methods, the prompt API provides an `append()` method, which takes the same message format as `prompt()`. Here's an example of how that could be useful:

```js
const session = await LanguageModel.create({
  initialPrompts: [{
    role: "system",
    content: "You are a skilled analyst who correlates patterns across multiple images."
  }],
  expectedInputs: [{ type: "image" }]
});

fileUpload.onchange = async (e) => {
  await session.append([{
    role: "user",
    content: [
      { type: "text", value: `Here's one image. Notes: ${fileNotesInput.value}` },
      { type: "image", value: fileUpload.files[0] }
    ]
  }]);
};

analyzeButton.onclick = async (e) => {
  analysisResult.textContent = await session.prompt(userQuestionInput.value);
};
```

The promise returned by `append()` will reject if the prompt cannot be appended (e.g., too big, invalid modalities for the session, etc.), or will fulfill once the prompt has been validated, processed, and appended to the session.

Note that `append()` can also cause [overflow](#tokenization-context-window-length-limits-and-overflow), in which case it will evict the oldest non-system prompts from the session and fire the `"quotaoverflow"` event.

### Configuration of per-session parameters

In addition to the `initialPrompts` option shown above, the currently-configurable model parameters are [temperature](https://huggingface.co/blog/how-to-generate#sampling) and [top-K](https://huggingface.co/blog/how-to-generate#top-k-sampling). The `params()` API gives the default and maximum values for these parameters.

_However, see [issue #42](https://github.com/webmachinelearning/prompt-api/issues/42): sampling hyperparameters are not universal among models._

```js
const customSession = await LanguageModel.create({
  temperature: 0.8,
  topK: 10
});

const params = await LanguageModel.params();
const conditionalSession = await LanguageModel.create({
  temperature: isCreativeTask ? params.defaultTemperature * 1.1 : params.defaultTemperature * 0.8,
  topK: isGeneratingIdeas ? params.maxTopK : params.defaultTopK
});
```

If the language model is not available at all in this browser, `params()` will fulfill with `null`.

Error-handling behavior:

* If values below 0 are passed for `temperature`, then `create()` will return a promise rejected with a `RangeError`.
* If values above `maxTemperature` are passed for `temperature`, then `create()` will clamp to `maxTemperature`. (`+Infinity` is specifically allowed, as a way of requesting maximum temperature.)
* If values below 1 are passed for `topK`, then `create()` will return a promise rejected with a `RangeError`.
* If values above `maxTopK` are passed for `topK`, then `create()` will clamp to `maxTopK`. (This includes `+Infinity` and numbers above `Number.MAX_SAFE_INTEGER`.)
* If fractional values are passed for `topK`, they are rounded down (using the usual [IntegerPart](https://webidl.spec.whatwg.org/#abstract-opdef-integerpart) algorithm for web specs).

### Session persistence and cloning

Each language model session consists of a persistent series of interactions with the model:

```js
const session = await LanguageModel.create({
  initialPrompts: [{
    role: "system",
    content: "You are a friendly, helpful assistant specialized in clothing choices."
  }]
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
const session = await LanguageModel.create({
  initialPrompts: [{
    role: "system",
    content: "You are a friendly, helpful assistant specialized in clothing choices."
  }]
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

const session = await LanguageModel.create({ signal: controller.signal });
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

* If the prompt is still queued behind other prompts in the session, then it will be removed from the queue, and the returned promise will be rejected with an `"AbortError"` `DOMException`.
* If the prompt is being currently responded to by the model, then it will be aborted, the prompt/response pair will be removed from the session, and the returned promise will be rejected with an `"AbortError"` `DOMException`.
* If the prompt has already been fully responded to by the model, then attempting to abort the prompt will do nothing.

Similarly, the `append()` operation can also be aborted. In this case the behavior is:

* If the append is queued behind other appends in the session, then it will be removed from the queue, and the returned promise will be rejected with an `"AbortError"` `DOMException`.
* If the append operation is currently ongoing, then it will be aborted, any part of the prompt that was appended so far will be removed from the session, and the returned promise will be rejected with an `"AbortError"` `DOMException`.
* If the append operation is complete (i.e., the returned promise has resolved), then attempting to abort it will do nothing. This includes all the following states:
  * The append operation is complete, but a prompt generation step has not yet triggered.
  * The append operation is complete, and a prompt generation step is processing.
  * The append operation is complete, and a prompt generation step has used it to produce a result.

Finally, note that if either prompting or appending has caused an [overflow](#tokenization-context-window-length-limits-and-overflow), aborting the operation does not re-introduce the overflowed messages into the session.

### Tokenization, context window length limits, and overflow

A given language model session will have a maximum number of tokens it can process. Developers can check their current usage and progress toward that limit by using the following properties on the session object:

```js
console.log(`${session.inputUsage} tokens used, out of ${session.inputQuota} tokens available.`);
```

To know how many tokens a prompt will consume, without actually processing it, developers can use the `measureInputUsage()` method. This method accepts the same input types as `prompt()`, including strings and multimodal input arrays:

```js
const stringUsage = await session.measureInputUsage(promptString);

const audioUsage = await session.measureInputUsage([{
  role: "user",
  content: [
    { type: "text", value: "My response to your critique:" },
    { type: "audio", value: audioBlob }
  ]
}]);
```

Some notes on this API:

* We do not expose the actual tokenization to developers since that would make it too easy to depend on model-specific details.
* Implementations must include in their count any control tokens that will be necessary to process the prompt, e.g. ones indicating the start or end of the input.
* The counting process can be aborted by passing an `AbortSignal`, i.e. `session.measureInputUsage(promptString, { signal })`.
* We use the phrases "input usage" and "input quota" in the API, to avoid being specific to the current language model tokenization paradigm. In the future, even if we change paradigms, we anticipate some concept of usage and quota still being applicable, even if it's just string length.

It's possible to send a prompt that causes the context window to overflow. That is, consider a case where `session.measureInputUsage(promptString) > session.inputQuota - session.inputUsage` before calling `session.prompt(promptString)`, and then the web developer calls `session.prompt(promptString)` anyway. In such cases, the initial portions of the conversation with the language model will be removed, one prompt/response pair at a time, until enough tokens are available to process the new prompt. The exception is the [system prompt](#system-prompts), which is never removed.

Such overflows can be detected by listening for the `"quotaoverflow"` event on the session:

```js
session.addEventListener("quotaoverflow", () => {
  console.log("We've gone past the quota, and some inputs will be dropped!");
});
```

If it's not possible to remove enough tokens from the conversation history to process the new prompt, then the `prompt()` or `promptStreaming()` call will fail with a `QuotaExceededError` exception and nothing will be removed. This is a proposed new type of exception, which subclasses `DOMException`, and replaces the web platform's existing `"QuotaExceededError"` `DOMException`. See [whatwg/webidl#1465](https://github.com/whatwg/webidl/pull/1465) for this proposal. For our purposes, the important part is that it has the following properties:

* `requested`: how many tokens the input consists of
* `quota`: how many tokens were available (which will be less than `requested`, and equal to the value of `session.inputQuota - session.inputUsage` at the time of the call)

### Multilingual content and expected input languages

The default behavior for a language model session assumes that the input languages are unknown. In this case, implementations will use whatever "base" capabilities they have available for the language model, and might throw `"NotSupportedError"` `DOMException`s if they encounter languages they don't support.

It's better practice, if possible, to supply the `create()` method with information about the expected input languages. This allows the implementation to download any necessary supporting material, such as fine-tunings or safety-checking models, and to immediately reject the promise returned by `create()` if the web developer needs to use languages that the browser is not capable of supporting:

```js
const session = await LanguageModel.create({
  initialPrompts: [{
    role: "system",
    content: `
      You are a foreign-language tutor for Japanese. The user is Korean. If necessary, either you or
      the user might "break character" and ask for or give clarification in Korean. But by default,
      prefer speaking in Japanese, and return to the Japanese conversation once any sidebars are
      concluded.
    `
  }],
  expectedInputs: [{
    type: "text",
    languages: ["en" /* for the system prompt */, "ja", "ko"]
  }],
  // See below section
  expectedOutputs: [{
    type: "text",
    languages: ["ja", "ko"]
  }],
});
```

The expected input languages are supplied alongside the [expected input types](#multimodal-inputs), and can vary per type. Our above example assumes the default of `type: "text"`, but more complicated combinations are possible, e.g.:

```js
const session = await LanguageModel.create({
  expectedInputs: [
    // Be sure to download any material necessary for English and Japanese text
    // prompts, or fail-fast if the model cannot support that.
    { type: "text", languages: ["en", "ja"] },

    // `languages` omitted: audio input processing will be best-effort based on
    // the base model's capability.
    { type: "audio" },

    // Be sure to download any material necessary for OCRing French text in
    // images, or fail-fast if the model cannot support that.
    { type: "image", languages: ["fr"] }
  ]
});
```

Note that the expected input languages do not affect the context or prompt the language model sees; they only impact the process of setting up the session, performing appropriate downloads, and failing creation if those input languages are unsupported.

If you want to check the availability of a given `expectedInputs` configuration before initiating session creation, you can use the `LanguageModel.availability()` method:

```js
const availability = await LanguageModel.availability({
  expectedInputs: [
    { type: "text", languages: ["en", "ja"] },
    { type: "audio", languages: ["en", "ja"] }
  ]
});

// `availability` will be one of "unavailable", "downloadable", "downloading", or "available".
```

### Expected output languages

In general, what output language the model responds in will be governed by the language model's own decisions. For example, a prompt such as "Please say something in French" could produce "Bonjour" or it could produce "I'm sorry, I don't know French".

However, if you know ahead of time what languages you are hoping for the language model to output, it's best practice to use the `expectedOutputs` option to `LanguageModel.create()` to indicate them. This allows the implementation to download any necessary supporting material for those output languages, and to immediately reject the returned promise if it's known that the model cannot support that language:

```js
const session = await LanguageModel.create({
  initialPrompts: [{
    role: "system",
    content: `You are a helpful, harmless French chatbot.`
  }],
  expectedInputs: [
    { type: "text", languages: ["en" /* for the system prompt */, "fr"] }
  ],
  expectedOutputs: [
    { type: "text", languages: ["fr"] }
  ]
});
```

As with `expectedInputs`, specifying a given language in `expectedOutputs` does not actually influence the language model's output. It's only expressing an expectation that can help set up the session, perform downloads, and fail creation if necessary. And as with `expectedInputs`, you can use `LanguageModel.availability()` to check ahead of time, before creating a session.

(Note that presently, the prompt API does not support multimodal outputs, so including anything array entries with `type`s other than `"text"` will always fail. However, we've chosen this general shape so that in the future, if multimodal output support is added, it fits into the API naturally.)

### Testing available options before creation

In the simple case, web developers should call `LanguageModel.create()`, and handle failures gracefully.

However, if the web developer wants to provide a differentiated user experience, which lets users know ahead of time that the feature will not be possible or might require a download, they can use the promise-returning `LanguageModel.availability()` method. This method lets developers know, before calling `create()`, what is possible with the implementation.

The method will return a promise that fulfills with one of the following availability values:

* "`unavailable`" means that the implementation does not support the requested options, or does not support prompting a language model at all.
* "`downloadable`" means that the implementation supports the requested options, but it will have to download something (e.g. the language model itself, or a fine-tuning) before it can create a session using those options.
* "`downloading`" means that the implementation supports the requested options, but will need to finish an ongoing download operation before it can create a session using those options.
* "`available`" means that the implementation supports the requested options without requiring any new downloads.

An example usage is the following:

```js
const options = {
  expectedInputs: [
    { type: "text", languages: ["en", "es"] },
    { type: "audio", languages: ["en", "es"] }
  ],
  temperature: 2
};

const availability = await LanguageModel.availability(options);

if (availability !== "unavailable") {
  if (availability !== "available") {
    console.log("Sit tight, we need to do some downloading...");
  }

  const session = await LanguageModel.create(options);
  // ... Use session ...
} else {
  // Either the API overall, or the expected languages and temperature setting, is not available.
  console.error("No language model for us :(");
}
```

### Download progress

For cases where using the API is only possible after a download, you can monitor the download progress (e.g. in order to show your users a progress bar) using code such as the following:

```js
const session = await LanguageModel.create({
  monitor(m) {
    m.addEventListener("downloadprogress", e => {
      console.log(`Downloaded ${e.loaded * 100}%`);
    });
  }
});
```

If the download fails, then `downloadprogress` events will stop being emitted, and the promise returned by `create()` will be rejected with a "`NetworkError`" `DOMException`.

Note that in the case that multiple entities are downloaded (e.g., a base model plus [LoRA fine-tunings](https://arxiv.org/abs/2106.09685) for the `expectedInputs`) web developers do not get the ability to monitor the individual downloads. All of them are bundled into the overall `downloadprogress` events, and the `create()` promise is not fulfilled until all downloads and loads are successful.

The event is a [`ProgressEvent`](https://developer.mozilla.org/en-US/docs/Web/API/ProgressEvent) whose `loaded` property is between 0 and 1, and whose `total` property is always 1. (The exact number of total or downloaded bytes are not exposed; see the discussion in [webmachinelearning/writing-assistance-apis issue #15](https://github.com/webmachinelearning/writing-assistance-apis/issues/15).)

At least two events, with `e.loaded === 0` and `e.loaded === 1`, will always be fired. This is true even if creating the model doesn't require any downloading.

<details>
<summary>What's up with this pattern?</summary>

This pattern is a little involved. Several alternatives have been considered. However, asking around the web standards community it seemed like this one was best, as it allows using standard event handlers and `ProgressEvent`s, and also ensures that once the promise is settled, the session object is completely ready to use.

It is also nicely future-extensible by adding more events and properties to the `m` object.

Finally, note that there is a sort of precedent in the (never-shipped) [`FetchObserver` design](https://github.com/whatwg/fetch/issues/447#issuecomment-281731850).
</details>

## Detailed design

### Instruction-tuned versus base models

We intend for this API to expose instruction-tuned models. Although we cannot mandate any particular level of quality or instruction-following capability, we think setting this base expectation can help ensure that what browsers ship is aligned with what web developers expect.

To illustrate the difference and how it impacts web developer expectations:

* In a base model, a prompt like "Write a poem about trees." might get completed with "... Write about the animal you would like to be. Write about a conflict between a brother and a sister." (etc.) It is directly completing plausible next tokens in the text sequence.
* Whereas, in an instruction-tuned model, the model will generally _follow_ instructions like "Write a poem about trees.", and respond with a poem about trees.

To ensure the API can be used by web developers across multiple implementations, all browsers should be sure their models behave like instruction-tuned models.

### Permissions policy, iframes, and workers

By default, this API is only available to top-level `Window`s, and to their same-origin iframes. Access to the API can be delegated to cross-origin iframes using the [Permissions Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/Permissions_Policy) `allow=""` attribute:

```html
<iframe src="https://example.com/" allow="language-model"></iframe>
```

This API is currently not available in workers, due to the complexity of establishing a responsible document for each worker in order to check the permissions policy status. See [this discussion](https://github.com/webmachinelearning/translation-api/issues/18#issuecomment-2705630392) for more. It may be possible to loosen this restriction over time, if use cases arise.

Note that although the API is not exposed to web platform workers, a browser could expose them to extension service workers, which are outside the scope of web platform specifications and have a different permissions model.

## Alternatives considered and under consideration

### How many stages to reach a response?

To actually get a response back from the model given a prompt, the following possible stages are involved:

1. Download the model, if necessary.
2. Establish a session, including configuring per-session options and parameters.
3. Add an initial prompt to establish context. (This will not generate a response.)
4. Execute a prompt and receive a response.

We've chosen to manifest these 3-4 stages into the API as two methods, `LanguageModel.create()` and `session.prompt()`/`session.promptStreaming()`, with some additional facilities for dealing with the fact that `LanguageModel.create()` can include a download step. Some APIs simplify this into a single method, and some split it up into three (usually not four).

### Stateless or session-based

Our design here uses [sessions](#session-persistence-and-cloning). An alternate design, seen in some APIs, is to require the developer to feed in the entire conversation history to the model each time, keeping track of the results.

This can be slightly more flexible; for example, it allows manually correcting the model's responses before feeding them back into the context window.

However, our understanding is that the session-based model can be more efficiently implemented, at least for browsers with on-device models. (Implementing it for a cloud-based model would likely be more work.) And, developers can always achieve a stateless model by using a new session for each interaction.

## Privacy and security considerations

Please see [the _Writing Assistance APIs_ specification](https://webmachinelearning.github.io/writing-assistance-apis/#privacy), where we have centralized the normative privacy and security considerations that apply to all APIs of this type.

## Stakeholder feedback

* W3C TAG: [w3ctag/design-reviews#1093](https://github.com/w3ctag/design-reviews/issues/1093)
* Browser engines and browsers:
  * Chromium: prototyping behind a flag
  * Mozilla: [mozilla/standards-positions#1213](https://github.com/mozilla/standards-positions/issues/1213)
  * WebKit: [WebKit/standards-positions#495](https://github.com/WebKit/standards-positions/issues/495)
* Web developers: positive
  * See [issue #74](https://github.com/webmachinelearning/prompt-api/issues/74) for some developer feedback
  * Examples of organic enthusiasm: [X post](https://x.com/mortenjust/status/1805190952358650251), [blog post](https://tyingshoelaces.com/blog/chrome-ai-prompt-api), [blog post](https://labs.thinktecture.com/local-small-language-models-in-the-browser-a-first-glance-at-chromes-built-in-ai-and-prompt-api-with-gemini-nano/)
  * [Feedback from developer surveys](https://docs.google.com/presentation/d/1DhFC2oB4PRrchavxUY3h9U4w4hrX5DAc5LoMqhn5hnk/edit#slide=id.g349a9ada368_1_6327)
