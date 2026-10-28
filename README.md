# Email-to-WhatsApp Assistant

An n8n workflow that reads each new Gmail message, sorts it into one of seven categories with an LLM, and sends a short WhatsApp message only for urgent and action emails.

## The Problem

Checking email all day is a cost for people who need to stay reachable.

- Most mail is promotions, receipts, and newsletters, but the few urgent messages are buried in it.
- Gmail's own sorting does not know what matters to you.
- Push notifications fire for everything, so people mute them.
- Messages that mention a meeting have to be copied to a calendar by hand.

## The Solution

1. A Gmail trigger checks for new mail every minute.
2. A Gemini agent reads the subject, body, and sender, and returns one category: `!!Urgent`, `@Action`, `@Calendar`, `@Waiting`, `Read Later`, `myPromos`, or `myArchive`.
3. The workflow adds the matching Gmail label.
4. For Urgent and Action emails, a second agent writes a short WhatsApp message with key words in bold.
5. The message is sent through the Whapi API.
6. For Urgent, Action, and Calendar emails, a third agent extracts event details. If it finds a clear date, a Google Calendar event is created.

## How It Helps

- Only two categories reach your phone. The rest stay in Gmail, labelled.
- Each WhatsApp message states what is needed and by when, so you can decide without opening the email.
- Meetings mentioned in an email can land on your calendar without typing.
- Times in other time zones are converted to Gulf Standard Time (UTC+4) by the prompt. Change this for your zone.

## What Makes It Different

| Feature | How it works here |
|---|---|
| Sorting | A prompt with examples per category, not keyword rules |
| Output control | Agents return structured output through a parser |
| Phone alerts | Limited to `!!Urgent` and `@Action` |
| Calendar | The agent must return an error flag instead of guessing a missing date |
| Label setup | A manual branch creates the seven labels |

## How It Works

```
Gmail Trigger (every minute)
        |
   AI Agent: classify
        |
     Switch on category
   /  |  |  |  |  |  \
 label the email in Gmail (7 label nodes)

 Urgent / Action
        |
   AI Agent: write WhatsApp text --> Whapi --> WhatsApp

 Urgent / Action / Calendar
        |
   AI Agent: extract event --> error? --no--> Google Calendar event
```

## Tech Stack

| Part | Tool |
|---|---|
| Orchestration | n8n |
| LLM | Google Gemini (3 agents with structured output) |
| Mail | Gmail |
| Calendar | Google Calendar |
| WhatsApp | Whapi (`gate.whapi.cloud`) |

## Run It Locally

1. Import `workflow.json` into n8n.
2. Create credentials: Gmail OAuth, Google Calendar OAuth, Gemini, and a Whapi bearer token.
3. Run the manual branch once. It creates the seven Gmail labels.
4. Open each label node and select the new label ids. The file holds ids from the original account (`Label_3` to `Label_8`) that will not match yours.
5. Replace `WHATSAPP_NUMBER`, `YOUR_CALENDAR_ID`, and `YOUR_EMAIL@example.com`.
6. Edit the system prompts: the example emails, the time zone, and the "Sent to" address.
7. Activate the workflow.

## Warnings

- Full email text is sent to Gemini, and urgent text goes through Whapi. Do not use this on mail you cannot share with them.
- Gemini and Whapi may be paid. The trigger runs every minute.
- The model can misclassify. A real urgent email may be labelled `Read Later`.
- Keep keys out of the JSON. The error-alert node used a CallMeBot key in the URL query, now replaced by `CALLMEBOT_API_KEY`.
- The workflow does not archive or delete mail. It only adds labels.

## Status

Done: trigger, classification, label nodes, WhatsApp message, event extraction, calendar creation.

Known issues, from reading the nodes:
- The `Action` and `Urgent` label nodes have an empty or fixed message id, so those two labels are probably not applied.
- The error branch (`if error?`, `set error message`, CallMeBot alert) is disabled, so failures do not reach WhatsApp.
- The `Filter` node after the trigger is disabled and passes everything.

Next: fix those three, then add a test with sample emails.
