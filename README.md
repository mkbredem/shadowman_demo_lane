# shadowman-demo-lane

Reservable, auto-named Datadog demo hosts for shadowman, kept on a separate
"demo lane" so nothing in the existing Datadog setup changes.
Every object in the demo lane uses the existing name plus the suffix `_v2`.

| Existing object | Demo lane object |
| --- | --- |
| Datadog tag `env:shadowman` | `env:shadowman_v2` |
| Datadog webhook `EDA_Webhook` | `EDA_Webhook_v2` |
| Monitor "Web Server is Down" | "Web Server is Down_v2" |
| Monitor "CPU Load is high" | "CPU Load is high_v2" |
| (no copy) "Host not reporting data" | none, so deleted demo hosts raise no alert |
| AAP event stream "Datadog" (Infrastructure) | "Datadog_v2" (Infrastructure) |
| Credential "Datadog Token" | "Datadog Token_v2" |
| Job template 842 "Install and Configure Datadog Agent" | "Install and Configure Datadog Agent_v2" |
| Five rulebooks on event stream "Datadog" | one rulebook `rulebooks/datadog_v2.yml`, activation "Datadog_v2" |

Host names are not suffixed: `<user>-<MMDD>-<flow>-datadog.shadowman.dev`
(hyphens, because underscores are not valid in DNS host names).

## Phase 1 setup, in order

### 1. AAP: credential and event stream (tracker step A17)
1. Automation Decisions > Infrastructure > Credentials > Create credential
   - Name: `Datadog Token_v2`
   - Organization: `Infrastructure`
   - Credential type: Token Event Stream
   - Token: generate a new random value; keep it for step 2
   - HTTP header key: `Authorization`
2. Automation Decisions > Event Streams > Create event stream
   - Name: `Datadog_v2`
   - Organization: `Infrastructure`
   - Event stream type: Token Event Stream
   - Credential: `Datadog Token_v2`
   - Leave **Test mode** on for now.
3. Copy the event stream URL.

### 2. Datadog: webhook and monitors (tracker step A16)
1. Integrations > Webhooks > New: name `EDA_Webhook_v2`, URL = the `Datadog_v2`
   event stream URL, payload identical to `EDA_Webhook`, custom header
   `{"Authorization": "<token from step 1>"}`.
2. Create the two monitors in `datadog/monitors_v2.json` (Monitors > New monitor >
   Import monitor from JSON), one at a time.
3. Send a test notification from the webhook page and confirm the event shows in
   `Datadog_v2` test mode. Then turn test mode off.

### 3. AAP: agent install copy (tracker step A18)
1. Credential type `Datadog API Key_v2` (Organization-independent):
   - Input: `{"fields":[{"id":"api_key","label":"Datadog API key","type":"string","secret":true}],"required":["api_key"]}`
   - Injector: `{"extra_vars":{"datadog_api_key":"{{ api_key }}"}}`
2. Credential `Datadog API Key_v2` of that type in Infrastructure, holding the
   same API key job template 842 uses today.
3. Project `Shadowman Demo Lane_v2` in Infrastructure, pointing at
   `https://github.com/mkbredem/shadowman-demo-lane`.
4. Job template `Install and Configure Datadog Agent_v2`: copy of 842 settings
   (inventory Shadowman Production, same execution environment, become on,
   Ansible Linux Service Account), project `Shadowman Demo Lane_v2`, playbook
   `playbooks/datadogagentinstall_v2.yml`, credentials add `Datadog API Key_v2`,
   **no extra variables**, prompt on launch for Variables (for `vm_name`).
   Job template 842 is not changed.

### 4. AAP: demo rulebook and activation (tracker step A19)
1. EDA project `Shadowman Demo Lane_v2` (Infrastructure), same repository.
2. Rulebook activation `Datadog_v2`:
   - Organization: Infrastructure
   - Rulebook: `datadog_v2.yml`
   - Source mapping: "Listen for alerts from Datadog v2" -> event stream `Datadog_v2`
   - Extra variables: `ao_webhook_path: c8c4e8c2-9a00-4ee2-a1cc-03c5f87a6798`
   - Decision environment and AAP credential: the same ones activation 46
     "Datadog Events to AO" uses (open it and copy them)
   - Restart policy: on failure
