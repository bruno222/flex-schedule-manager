# Twilio Flex Schedule Manager

This solution provides a flexible, robust, and scalable way to manage open and closed hours for Twilio Flex applications.

![Schedule manager](screenshots/schedules.png)

## Disclaimer

**This software is to be considered "sample code", a Type B Deliverable, and is delivered "as-is" to the user. Twilio bears no responsibility to support the use or implementation of this software.**

## How it works

The schedule manager uses two main concepts: rules and schedules. Rules define a single or recurring event, such as "open Monday - Friday 8 AM - 5 PM," or "closed for holiday on 9/5/2022." Schedules are comprised of one or more rules (along with a time zone to apply the rules for). When checking the schedule from your application, such as a Studio flow, the status of the schedule will be returned (open or closed), and if it is closed, information from the matching rule (such as closed for holiday) will also be provided to allow for flexible handling.

To manage rules and schedules, a Flex plugin is provided which adds a Schedule Manager item to the side navigation for workers with the `admin` role. This allows viewing the current configuration, the status of each schedule, and publishing updates to the configuration.

![Rules view](screenshots/rules.png)

To allow for greater scalability than provided by Twilio Sync and some other solutions, configuration is stored within a Twilio Asset behind a Twilio Function. When updates to the configuration are being saved, a new asset version is generated and included in a new build, which is deployed when completed. This means that publishing schedules may take a few moments.

## Pre-requisites

Make sure you have [Node.js](https://nodejs.org) as well as [`npm`](https://npmjs.com) installed.

Next, please install the [Twilio CLI](https://www.twilio.com/docs/twilio-cli/quickstart). If you are using Homebrew on macOS, you can do so by running:

```bash
brew tap twilio/brew && brew install twilio
```

Finally, install the [Flex Plugin extension](https://www.twilio.com/docs/flex/developer/plugins/cli/install) and the [serverless plugin](https://www.twilio.com/docs/labs/serverless-toolkit/getting-started) for the Twilio CLI:

```bash
twilio plugins:install @twilio-labs/plugin-flex
twilio plugins:install @twilio-labs/plugin-serverless
```

## Installation

First, clone the repository, change to its directory, and install:

```bash
git clone https://github.com/twilio-professional-services/flex-schedule-manager.git

cd flex-schedule-manager
npm install
```

Then, copy `.env.example` to `.env` and configure your Twilio account SID and token:

```
ACCOUNT_SID=ACxxxxxx
AUTH_TOKEN=abc123

TWILIO_SERVICE_RETRY_LIMIT=5
TWILIO_SERVICE_MIN_BACKOFF=100
TWILIO_SERVICE_MAX_BACKOFF=300
```

Then, deploy the serverless functions:

```bash
twilio serverless:deploy
```

Note the domain name that is output when the deploy completes--this will be referenced throughout the rest of the readme.

**Note: If you need to re-deploy via CLI in the future, be sure to first update your local `serverless/assets/config.private.json` file with any configuration changes.**

Then, update the Flex UI configuration with the serverless function domain from above:

```
POST https://flex-api.twilio.com/v1/Configuration
Authorization: Basic {base64-encoded Twilio Account SID : Auth Token}
Content-Type: application/json

{
  "account_sid": "Enter your Twilio Account SID here",
  "ui_attributes": {
    ... include your existing ui_attributes here ...
    "schedule_manager": {
      "serverless_functions_domain": "Enter the serverless domain here"
    }
  }
}
```

Next, switch to the Flex plugin directory:

Flex UI version 1.x 
```bash
cd ../plugin-schedule-manager-v1
```

Flex UI version 2.x
```bash
cd ../plugin-schedule-manager-v2
```

Copy `public/appConfig.example.js` to `public/appConfig.js`:

```bash
cp public/appConfig.example.js public/appConfig.js
```

Install the dependencies:

```bash
npm install
```

Run the plugin locally:

```bash
twilio flex:plugins:start
```

Once you are happy with your plugin, you have to deploy then release the plugin for it to take affect on Twilio hosted Flex.

Run the following command to start the deployment:

```bash
twilio flex:plugins:deploy --major --changelog "Notes for this version" --description "Functionality of the plugin"
```

After your deployment runs you will receive instructions for releasing your plugin from the bash prompt. You can use this or skip this step and release your plugin from the Flex plugin dashboard here https://flex.twilio.com/admin/plugins

For more details on deploying your plugin, refer to the [deploying your plugin guide](https://www.twilio.com/docs/flex/plugins#deploying-your-plugin).

## Example configurations

Example configurations, such as pre-defined holidays, are available in the `examples` directory. To use one of these, paste the contents of the desired file into `serverless/assets/config.private.json`, replacing its contents. Then, deploy the service.

## Evaluating schedules

You can evaluate a schedule in your application by making an HTTP request to the `check-schedule` function:

```
https://my-serverless-domain/check-schedule?name=Schedule%20Name%20Here
```

or

```
POST https://my-serverless-domain/check-schedule
Content-Type: application/json

{
  "name": "Name of the schedule you want to evaluate"
}
```

The schedule's rules will then be evaluated against the current date and time. The logic is as follows:

- If the schedule is marked as manually closed, the schedule is closed, and the closed reason provided is `manual`.
- If one or more closed rules match, the schedule is closed, and the closed reason provided is from the topmost closed rule match in the list.
- If an open rule matches and no closed rules match, the schedule is open.
- If no rules match, the schedule is closed, and the closed reason provided is `closed`.

Here is an example response from a closed schedule:
```
{
  "isOpen": false,
  "closedReason": "closed"
}
```

Here is an example response from an open schedule:
```
{
  "isOpen": true,
  "closedReason": ""
}
```

## Simulating schedules

When performing testing, you may want to simulate a date/time when evaluating a schedule. To do so, provide an ISO 8601-formatted date/time in the `simulate` parameter in your request:

```
POST https://my-serverless-domain/check-schedule
Content-Type: application/json

{
  "name": "Name of the schedule you want to evaluate",
  "simulate": "2022-08-08T07:59"
}
```

## Get the status of all schedules

Schedule data may be useful within Flex, such as during queue transfers. The included utility classes in the plugin component provide an easy way to get the status of all schedules:

```js
import { loadScheduleData } from 'utils/schedule-manager';

...

const scheduleData = await loadScheduleData();
```

This calls the `admin/list` function, which requires the Flex user token. `scheduleData` will include a `data` object with a `schedules` array, containing each schedule. Each schedule will have a `status` object, which contains the same object you would get from calling the `check-schedules` function.

## Using within Studio

![Studio example](screenshots/studio.png)

1. Bring a Run Function widget into your flow, named `check_schedule`, configured to the `schedule-manager` service, and `/check-schedule` function.
2. Add to the function parameters: key: `name`, value: `Name of schedule to check` and save.
3. Bring a Split Based On... widget into your flow and set the variable to test: `widgets.check_schedule.parsed.isOpen`
4. Add a condition Equal To `true`, connect it to the step representing your open logic, and save.
5. Connect the 'No Condition Matches' to your closed logic.
6. If you'd like to say different text depending on the closed reason, here is an example you can use in a Say/Play widget:

```
{% case widgets.check_schedule.parsed.closedReason  %}

{% when 'manual' %}

We are currently closed due to unforseen circumstances. Please call back later.

{% when 'holiday' %}

We are currently closed due to the holiday. Please call back during our normal business hours.

{% else %}

We are currently closed. Please call back during our normal business hours.

{% endcase %}

Thank you for calling and have a great day.
```

## Preventing conflicting configuration updates

The Flex plugin loads the configuration interface for workers with the `admin` role, of which there may be more than one. Therefore, it is a possibility that multiple people may attempt to update the schedule configuration at the same time. To prevent workers overwriting each other's changes, a few guards have been put in place:

- When updating configuration with the `update-schedules` function, the `version` property must be provided with the same `version` that was retrieved from the `list-schedules` function which loaded the initial data. If this does not match, the request will fail. In the user interface, the following alert will be shown: `Schedule was updated by someone else and cannot be published. Please reload and try again.` This allows the worker to rescue the changes they were attempting to make, and merge them with the changes that were saved first.
- When retrieving configuration from this `list-schedules` function, a check is made that the latest build is what is deployed. The `versionIsDeployed` property is returned indicating whether this is the case. If it is not, this means another user is in the middle of publishing changes. In the user interface, the following alert will be shown: `Another schedule publish is in progress. Publishing now will overwrite other changes.` This allows the worker to wait for the publish to complete before making changes.

## Development

Run `twilio flex:plugins --help` to see all the commands we currently support. For further details on Flex Plugins refer to our documentation on the [Twilio Docs](https://www.twilio.com/docs/flex/developer/plugins/cli) page.

