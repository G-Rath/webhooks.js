# @octokit/webhooks

> GitHub webhook events toolset for Node.js

[![@latest](https://img.shields.io/npm/v/@octokit/webhooks.svg)](https://www.npmjs.com/package/@octokit/webhooks)
[![Test](https://github.com/octokit/webhooks.js/workflows/Test/badge.svg)](https://github.com/octokit/webhooks.js/actions?query=workflow)

<!-- toc -->

- [Usage](#usage)
- [Local development](#local-development)
- [API](#api)
  - [Constructor](#constructor)
  - [webhooks.sign()](#webhookssign)
  - [webhooks.verify()](#webhooksverify)
  - [webhooks.verifyAndReceive()](#webhooksverifyandreceive)
  - [webhooks.receive()](#webhooksreceive)
  - [webhooks.on()](#webhookson)
  - [webhooks.onAny()](#webhooksonany)
  - [webhooks.onError()](#webhooksonerror)
  - [webhooks.removeListener()](#webhooksremovelistener)
  - [webhooks.middleware()](#webhooksmiddleware)
  - [Webhook events](#webhook-events)
- [TypeScript](#typescript)
  - [`WebhookEvent`](#webhookevent)
  - [`EventNames`](#eventnames)
  - [`EventPayloads`](#eventpayloads)
- [License](#license)

<!-- tocstop -->

`@octokit/webhooks` helps to handle webhook events received from GitHub.

[GitHub webhooks](https://docs.github.com/webhooks/) can be registered in multiple ways

1. In repository or organization settings on [github.com](https://github.com/).
2. Using the REST API for [repositories](https://docs.github.com/rest/reference/repos#webhooks) or [organizations](https://docs.github.com/rest/reference/orgs#webhooks/)
3. By [creating a GitHub App](https://docs.github.com/developers/apps/creating-a-github-app).

Note that while setting a secret is optional on GitHub, it is required to be set in order to use `@octokit/webhooks`. Content Type must be set to `application/json`, `application/x-www-form-urlencoded` is not supported.

## Usage

```js
// install with: npm install @octokit/webhooks
const { Webhooks } = require("@octokit/webhooks");
const webhooks = new Webhooks({
  secret: "mysecret",
});

webhooks.onAny(({ id, name, payload }) => {
  console.log(name, "event received");
});

require("http").createServer(webhooks.middleware).listen(3000);
// can now receive webhook events at port 3000
```

## Local development

You can receive webhooks on your local machine or even browser using [EventSource](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) and [smee.io](https://smee.io/).

Go to [smee.io](https://smee.io/) and <kbd>Start a new channel</kbd>. Then copy the "Webhook Proxy URL" and

1. enter it in the GitHub App’s "Webhook URL" input
2. pass it to the [EventSource](https://github.com/EventSource/eventsource) constructor, see below

```js
const webhookProxyUrl = "https://smee.io/IrqK0nopGAOc847"; // replace with your own Webhook Proxy URL
const source = new EventSource(webhookProxyUrl);
source.onmessage = (event) => {
  const webhookEvent = JSON.parse(event.data);
  webhooks
    .verifyAndReceive({
      id: webhookEvent["x-request-id"],
      name: webhookEvent["x-github-event"],
      signature: webhookEvent["x-hub-signature"],
      payload: webhookEvent.body,
    })
    .catch(console.error);
};
```

`EventSource` is a native browser API and can be polyfilled for browsers that don’t support it. In node, you can use the [`eventsource`](https://github.com/EventSource/eventsource) package: install with `npm install eventsource`, then `const EventSource = require('eventsource')`

## API

1. [Constructor](#constructor)
2. [webhooks.sign()](#webhookssign)
3. [webhooks.verify()](#webhooksverify)
4. [webhooks.verifyAndReceive()](#webhooksverifyandreceive)
5. [webhooks.receive()](#webhooksreceive)
6. [webhooks.on()](#webhookson)
7. [webhooks.onAny()](#webhooksonany)
8. [webhooks.onError()](#webhooksonerror)
9. [webhooks.removeListener()](#webhooksremoveListener)
10. [webhooks.middleware()](#webhooksmiddleware)
11. [Webhook events](#webhook-events)

### Constructor

```js
new Webhooks({secret[, path, transform]})
```

<table width="100%">
  <tr>
    <td>
      <code>
        secret
      </code>
      <em>(String)</em>
    </td>
    <td>
      <strong>Required.</strong>
      Secret as configured in GitHub Settings.
    </td>
  </tr>
  <tr>
    <td>
      <code>
        path
      </code>
      <em>(String)</em>
    </td>
    <td>
      Only relevant for <a href="#webhooksmiddleware"><code>webhooks.middleware</code></a>.
      Custom path to match requests against. Defaults to <code>/</code>.
    </td>
  </tr>
  <tr>
    <td>
      <code>
        transform
      </code>
      <em>(Function)</em>
    </td>
    <td>
      Only relevant for <a href="#webhookson"><code>webhooks.on</code></a>.
      Transform emitted event before calling handlers. Can be asynchronous.
    </td>
  </tr>
</table>

Returns the `webhooks` API.

### webhooks.sign()

```js
webhooks.sign(eventPayload);
```

<table width="100%">
  <tr>
    <td>
      <code>
        eventPayload
      </code>
      <em>
        (Object)
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Webhook request payload as received from GitHub
    </td>
  </tr>
</table>

Returns a `signature` string. Throws error if `eventPayload` is not passed.

Can also be used [standalone](src/sign/).

### webhooks.verify()

```js
webhooks.verify(eventPayload, signature);
```

<table width="100%">
  <tr>
    <td>
      <code>
        eventPayload
      </code>
      <em>
        (Object)
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Webhook event request payload as received from GitHub.
    </td>
  </tr>
  <tr>
    <td>
      <code>
        signature
      </code>
      <em>
        (String)
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Signature string as calculated by <code><a href="#webhookssign">webhooks.sign()</a></code>.
    </td>
  </tr>
</table>

Returns `true` or `false`. Throws error if `eventPayload` or `signature` not passed.

Can also be used [standalone](src/verify/).

### webhooks.verifyAndReceive()

```js
webhooks.verifyAndReceive({ id, name, payload, signature });
```

<table width="100%">
  <tr>
    <td>
      <code>
        id
      </code>
      <em>
        String
      </em>
    </td>
    <td>
      Unique webhook event request id
    </td>
  </tr>
  <tr>
    <td>
      <code>
        name
      </code>
      <em>
        String
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Name of the event. (Event names are set as <a href="https://docs.github.com/developers/webhooks-and-events/webhook-events-and-payloads#delivery-headers"><code>X-GitHub-Event</code> header</a>
      in the webhook event request.)
    </td>
  </tr>
  <tr>
    <td>
      <code>
        payload
      </code>
      <em>
        Object
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Webhook event request payload as received from GitHub.
    </td>
  </tr>
  <tr>
    <td>
      <code>
        signature
      </code>
      <em>
        (String)
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Signature string as calculated by <code><a href="#webhookssign">webhooks.sign()</a></code>.
    </td>
  </tr>
</table>

Returns a promise.

Verifies event using [webhooks.verify()](#webhooksverify), then handles the event using [webhooks.receive()](#webhooksreceive).

Additionally, if verification fails, rejects the returned promise and emits an `error` event.

Example

```js
const { Webhooks } = require("@octokit/webhooks");
const webhooks = new Webhooks({
  secret: "mysecret",
});
eventHandler.on("error", handleSignatureVerificationError);

// put this inside your webhooks route handler
eventHandler
  .verifyAndReceive({
    id: request.headers["x-github-delivery"],
    name: request.headers["x-github-event"],
    payload: request.body,
    signature: request.headers["x-hub-signature"],
  })
  .catch(handleErrorsFromHooks);
```

### webhooks.receive()

```js
webhooks.receive({ id, name, payload });
```

<table width="100%">
  <tr>
    <td>
      <code>
        id
      </code>
      <em>
        String
      </em>
    </td>
    <td>
      Unique webhook event request id
    </td>
  </tr>
  <tr>
    <td>
      <code>
        name
      </code>
      <em>
        String
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Name of the event. (Event names are set as <a href="https://docs.github.com/developers/webhooks-and-events/webhook-events-and-payloads#delivery-headers"><code>X-GitHub-Event</code> header</a>
      in the webhook event request.)
    </td>
  </tr>
  <tr>
    <td>
      <code>
        payload
      </code>
      <em>
        Object
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Webhook event request payload as received from GitHub.
    </td>
  </tr>
</table>

Returns a promise. Runs all handlers set with [`webhooks.on()`](#webhookson) in parallel and waits for them to finish. If one of the handlers rejects or throws an error, then `webhooks.receive()` rejects. The returned error has an `.errors` property which holds an array of all errors caught from the handlers. If no errors occur, `webhooks.receive()` resolves without passing any value.

The `.receive()` method belongs to the `event-handler` module which can be used [standalone](src/event-handler/).

### webhooks.on()

```js
webhooks.on(eventName, handler);
webhooks.on(eventNames, handler);
```

<table width="100%">
  <tr>
    <td>
      <code>
        eventName
      </code>
      <em>
        String
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Name of the event. One of <a href="https://docs.github.com/developers/webhooks-and-events/webhook-events-and-payloads">GitHub's supported event names</a>.
    </td>
  </tr>
  <tr>
    <td>
      <code>
        eventNames
      </code>
      <em>
        Array
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Array of event names.
    </td>
  </tr>
  <tr>
    <td>
      <code>
        handler
      </code>
      <em>
        Function
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Method to be run each time the event with the passed name is received.
      the <code>handler</code> function can be an async function, throw an error or
      return a Promise. The handler is called with an event object: <code>{id, name, payload}</code>.
    </td>
  </tr>
</table>

The `.on()` method belongs to the `event-handler` module which can be used [standalone](src/event-handler/).

### webhooks.onAny()

```js
webhooks.onAny(handler);
```

<table width="100%">
  <tr>
    <td>
      <code>
        handler
      </code>
      <em>
        Function
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Method to be run each time any event is received.
      the <code>handler</code> function can be an async function, throw an error or
      return a Promise. The handler is called with an event object: <code>{id, name, payload}</code>.
    </td>
  </tr>
</table>

The `.onAny()` method belongs to the `event-handler` module which can be used [standalone](src/event-handler/).

### webhooks.onError()

```js
webhooks.onError(handler);
```

If a webhook event handler throws an error or returns a promise that rejects, an error event is triggered. You can use this handler for logging or reporting events. The passed error object has a .event property which has all information on the event.

Asynchronous `error` event handler are not blocking the `.receive()` method from completing.

<table width="100%">
  <tr>
    <td>
      <code>
        handler
      </code>
      <em>
        Function
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Method to be run each time a webhook event handler throws an error or returns a promise that rejects.
      The <code>handler</code> function can be an async function,
      return a Promise. The handler is called with an error object that has a .event property which hass all the information on the event: <code>{id, name, payload}</code>.
    </td>
  </tr>
</table>

The `.onError()` method belongs to the `event-handler` module which can be used [standalone](src/event-handler/).

### webhooks.removeListener()

```js
webhooks.removeListener(eventName, handler);
webhooks.removeListener(eventNames, handler);
```

<table width="100%">
  <tr>
    <td>
      <code>
        eventName
      </code>
      <em>
        String
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Name of the event. One of <a href="https://docs.github.com/developers/webhooks-and-events/webhook-events-and-payloads">GitHub’s supported event names</a>.
    </td>
  </tr>
  <tr>
    <td>
      <code>
        eventNames
      </code>
      <em>
        Array
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Array of event names.
    </td>
  </tr>
  <tr>
    <td>
      <code>
        handler
      </code>
      <em>
        Function
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      Method which was previously passed to <code><a href="webhookson">webhooks.on()</a></code>. If the same handler was registered multiple times for the same event, only the most recent handler gets removed.
    </td>
  </tr>
</table>

The `.removeListener()` method belongs to the `event-handler` module which can be used [standalone](src/event-handler/).

### webhooks.middleware()

```js
webhooks.middleware(request, response[, next])
```

<table width="100%">
  <tr>
    <td>
      <code>
        request
      </code>
      <em>
        Object
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      A Node.js <a href="https://nodejs.org/docs/latest/api/http.html#http_class_http_clientrequest">http.ClientRequest</a>.
    </td>
  </tr>
  <tr>
    <td>
      <code>
        response
      </code>
      <em>
        Object
      </em>
    </td>
    <td>
      <strong>Required.</strong>
      A Node.js <a href="https://nodejs.org/docs/latest/api/http.html#http_class_http_serverresponse">http.ServerResponse</a>.
    </td>
  </tr>
  <tr>
    <td>
      <code>
        next
      </code>
      <em>
        Function
      </em>
    </td>
    <td>
      Optional function which invokes the next middleware, as used by <a href="https://github.com/senchalabs/connect">Connect</a> and <a href="http://expressjs.com/">Express</a>.
    </td>
  </tr>
</table>

Returns a `requestListener` (or _middleware_) method which can be directly passed to [`http.createServer()`](https://nodejs.org/docs/latest/api/http.html#http_http_createserver_requestlistener), <a href="http://expressjs.com/">Express</a> and other compatible Node.js server frameworks.

Can also be used [standalone](src/middleware/).

### Webhook events

See the full list of [event types with example payloads](https://docs.github.com/developers/webhooks-and-events/webhook-events-and-payloads/).

If there are actions for a webhook, events are emitted for both, the webhook name as well as a combination of the webhook name and the action, e.g. `installation` and `installation.created`.

<!-- autogenerated via scripts/generate-types.ts -->

| Event                                                                                                                        | Actions                                                                                                                                                                                                                                             |
| ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`check_run`](https://developer.github.com/v3/activity/events/types/#checkrunevent)                                          | `completed`<br>`created`<br>`requested_action`<br>`rerequested`                                                                                                                                                                                     |
| [`check_suite`](https://developer.github.com/v3/activity/events/types/#checksuiteevent)                                      | `completed`<br>`requested`<br>`rerequested`                                                                                                                                                                                                         |
| [`code_scanning_alert`](https://developer.github.com/v3/activity/events/types/#codescanningalertevent)                       | `appeared_in_branch`<br>`closed_by_user`<br>`created`<br>`fixed`<br>`reopened`<br>`reopened_by_user`                                                                                                                                                |
| [`commit_comment`](https://developer.github.com/v3/activity/events/types/#commitcommentevent)                                | `created`                                                                                                                                                                                                                                           |
| [`content_reference`](https://developer.github.com/v3/activity/events/types/#contentreferenceevent)                          | `created`                                                                                                                                                                                                                                           |
| [`create`](https://developer.github.com/v3/activity/events/types/#createevent)                                               |                                                                                                                                                                                                                                                     |
| [`delete`](https://developer.github.com/v3/activity/events/types/#deleteevent)                                               |                                                                                                                                                                                                                                                     |
| [`deploy_key`](https://developer.github.com/v3/activity/events/types/#deploykeyevent)                                        | `created`<br>`deleted`                                                                                                                                                                                                                              |
| [`deployment`](https://developer.github.com/v3/activity/events/types/#deploymentevent)                                       | `created`                                                                                                                                                                                                                                           |
| [`deployment_status`](https://developer.github.com/v3/activity/events/types/#deploymentstatusevent)                          | `created`                                                                                                                                                                                                                                           |
| [`fork`](https://developer.github.com/v3/activity/events/types/#forkevent)                                                   |                                                                                                                                                                                                                                                     |
| [`github_app_authorization`](https://developer.github.com/v3/activity/events/types/#githubappauthorizationevent)             | `revoked`                                                                                                                                                                                                                                           |
| [`gollum`](https://developer.github.com/v3/activity/events/types/#gollumevent)                                               |                                                                                                                                                                                                                                                     |
| [`installation`](https://developer.github.com/v3/activity/events/types/#installationevent)                                   | `created`<br>`deleted`<br>`new_permissions_accepted`<br>`suspend`<br>`unsuspend`                                                                                                                                                                    |
| [`installation_repositories`](https://developer.github.com/v3/activity/events/types/#installationrepositoriesevent)          | `added`<br>`removed`                                                                                                                                                                                                                                |
| [`issue_comment`](https://developer.github.com/v3/activity/events/types/#issuecommentevent)                                  | `created`<br>`deleted`<br>`edited`                                                                                                                                                                                                                  |
| [`issues`](https://developer.github.com/v3/activity/events/types/#issuesevent)                                               | `assigned`<br>`closed`<br>`deleted`<br>`demilestoned`<br>`edited`<br>`labeled`<br>`locked`<br>`milestoned`<br>`opened`<br>`pinned`<br>`reopened`<br>`transferred`<br>`unassigned`<br>`unlabeled`<br>`unlocked`<br>`unpinned`                        |
| [`label`](https://developer.github.com/v3/activity/events/types/#labelevent)                                                 | `created`<br>`deleted`<br>`edited`                                                                                                                                                                                                                  |
| [`marketplace_purchase`](https://developer.github.com/v3/activity/events/types/#marketplacepurchaseevent)                    | `cancelled`<br>`changed`<br>`pending_change`<br>`pending_change_cancelled`<br>`purchased`                                                                                                                                                           |
| [`member`](https://developer.github.com/v3/activity/events/types/#memberevent)                                               | `added`<br>`edited`<br>`removed`                                                                                                                                                                                                                    |
| [`membership`](https://developer.github.com/v3/activity/events/types/#membershipevent)                                       | `added`<br>`removed`                                                                                                                                                                                                                                |
| [`meta`](https://developer.github.com/v3/activity/events/types/#metaevent)                                                   | `deleted`                                                                                                                                                                                                                                           |
| [`milestone`](https://developer.github.com/v3/activity/events/types/#milestoneevent)                                         | `closed`<br>`created`<br>`deleted`<br>`edited`<br>`opened`                                                                                                                                                                                          |
| [`org_block`](https://developer.github.com/v3/activity/events/types/#orgblockevent)                                          | `blocked`<br>`unblocked`                                                                                                                                                                                                                            |
| [`organization`](https://developer.github.com/v3/activity/events/types/#organizationevent)                                   | `deleted`<br>`member_added`<br>`member_invited`<br>`member_removed`<br>`renamed`                                                                                                                                                                    |
| [`package`](https://developer.github.com/v3/activity/events/types/#packageevent)                                             | `published`<br>`updated`                                                                                                                                                                                                                            |
| [`page_build`](https://developer.github.com/v3/activity/events/types/#pagebuildevent)                                        |                                                                                                                                                                                                                                                     |
| [`ping`](https://developer.github.com/v3/activity/events/types/#pingevent)                                                   |                                                                                                                                                                                                                                                     |
| [`project`](https://developer.github.com/v3/activity/events/types/#projectevent)                                             | `closed`<br>`created`<br>`deleted`<br>`edited`<br>`reopened`                                                                                                                                                                                        |
| [`project_card`](https://developer.github.com/v3/activity/events/types/#projectcardevent)                                    | `converted`<br>`created`<br>`deleted`<br>`edited`<br>`moved`                                                                                                                                                                                        |
| [`project_column`](https://developer.github.com/v3/activity/events/types/#projectcolumnevent)                                | `created`<br>`deleted`<br>`edited`<br>`moved`                                                                                                                                                                                                       |
| [`public`](https://developer.github.com/v3/activity/events/types/#publicevent)                                               |                                                                                                                                                                                                                                                     |
| [`pull_request`](https://developer.github.com/v3/activity/events/types/#pullrequestevent)                                    | `assigned`<br>`closed`<br>`converted_to_draft`<br>`edited`<br>`labeled`<br>`locked`<br>`opened`<br>`ready_for_review`<br>`reopened`<br>`review_request_removed`<br>`review_requested`<br>`synchronize`<br>`unassigned`<br>`unlabeled`<br>`unlocked` |
| [`pull_request_review`](https://developer.github.com/v3/activity/events/types/#pullrequestreviewevent)                       | `dismissed`<br>`edited`<br>`submitted`                                                                                                                                                                                                              |
| [`pull_request_review_comment`](https://developer.github.com/v3/activity/events/types/#pullrequestreviewcommentevent)        | `created`<br>`deleted`<br>`edited`                                                                                                                                                                                                                  |
| [`push`](https://developer.github.com/v3/activity/events/types/#pushevent)                                                   |                                                                                                                                                                                                                                                     |
| [`release`](https://developer.github.com/v3/activity/events/types/#releaseevent)                                             | `created`<br>`deleted`<br>`edited`<br>`prereleased`<br>`published`<br>`released`<br>`unpublished`                                                                                                                                                   |
| [`repository`](https://developer.github.com/v3/activity/events/types/#repositoryevent)                                       | `archived`<br>`created`<br>`deleted`<br>`edited`<br>`privatized`<br>`publicized`<br>`renamed`<br>`transferred`<br>`unarchived`                                                                                                                      |
| [`repository_dispatch`](https://developer.github.com/v3/activity/events/types/#repositorydispatchevent)                      | `on-demand-test`                                                                                                                                                                                                                                    |
| [`repository_import`](https://developer.github.com/v3/activity/events/types/#repositoryimportevent)                          |                                                                                                                                                                                                                                                     |
| [`repository_vulnerability_alert`](https://developer.github.com/v3/activity/events/types/#repositoryvulnerabilityalertevent) | `create`<br>`dismiss`<br>`resolve`                                                                                                                                                                                                                  |
| [`secret_scanning_alert`](https://developer.github.com/v3/activity/events/types/#secretscanningalertevent)                   | `created`<br>`reopened`<br>`resolved`                                                                                                                                                                                                               |
| [`security_advisory`](https://developer.github.com/v3/activity/events/types/#securityadvisoryevent)                          | `performed`<br>`published`<br>`updated`                                                                                                                                                                                                             |
| [`sponsorship`](https://developer.github.com/v3/activity/events/types/#sponsorshipevent)                                     | `cancelled`<br>`created`<br>`edited`<br>`pending_cancellation`<br>`pending_tier_change`<br>`tier_changed`                                                                                                                                           |
| [`star`](https://developer.github.com/v3/activity/events/types/#starevent)                                                   | `created`<br>`deleted`                                                                                                                                                                                                                              |
| [`status`](https://developer.github.com/v3/activity/events/types/#statusevent)                                               |                                                                                                                                                                                                                                                     |
| [`team`](https://developer.github.com/v3/activity/events/types/#teamevent)                                                   | `added_to_repository`<br>`created`<br>`deleted`<br>`edited`<br>`removed_from_repository`                                                                                                                                                            |
| [`team_add`](https://developer.github.com/v3/activity/events/types/#teamaddevent)                                            |                                                                                                                                                                                                                                                     |
| [`watch`](https://developer.github.com/v3/activity/events/types/#watchevent)                                                 | `started`                                                                                                                                                                                                                                           |
| [`workflow_dispatch`](https://developer.github.com/v3/activity/events/types/#workflowdispatchevent)                          |                                                                                                                                                                                                                                                     |
| [`workflow_run`](https://developer.github.com/v3/activity/events/types/#workflowrunevent)                                    | `completed`<br>`requested`                                                                                                                                                                                                                          |

<!-- /autogenerated via scripts/generate-types.ts -->

## TypeScript

`@octokit/webhooks` exports 3 types that can be used independent from the code.

Note that changes to the exported types are not considered breaking changes, as the changes will not impact production code, but only fail locally or during CI at build time.

### `WebhookEvent`

The `WebhookEvent` type is an object with the properties `id`, `name`, and `payload`. `name` must be one of the known event names. The type for `payload` be set using an optional type parameter, e.g. `WebhookEvent<MyPayloadType>`

### `EventNames`

The `EventNames` type is a module containing types for all known event names and event/action combinations. For example, `EventNames.CheckRunEvent` is a string enum for `"check_run" | "check_run.completed" | "check_run.created" | "check_run.requested_action" | "check_run.rerequested"`.

`EventNames.All` is an enum of all event/action combinations. `EventNames.StringNames` is an enum for the known event names only.

### `EventPayloads`

The `EventPayloads` type exports payload types for all known evens. For example `EventPayloads.WebhookPayloadCheckRun` exports the payload type for the `check_run` event.

## License

[MIT](LICENSE.md)
