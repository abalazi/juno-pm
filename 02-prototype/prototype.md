# Prototype · Juno

> Module 2 · Prototype the Idea Worth Building. The working build that tests your idea, made with your build tool of choice.

## Prototype link

https://abalazi.github.io/juno-customer-voice/

Code: https://github.com/abalazi/juno-customer-voice

## What it demonstrates

Juno's daily review loop. Juno reads new feedback, checks what happened at the event, and drafts a reply with a next step. At the end of the day I review each draft in Slack and approve it, edit it, send it to engineering or skip it. Nothing goes out without my click. On Monday Juno sums up what was replied, what went to engineering and what's still open.

## Debrief

- **What worked:** Clicking through makes the "human approves everything" idea easy to get. Checking the build against my skill file caught two gaps (Juno needed read access to more tools, and the daily Slack card had no format), so I updated the skill file. Showing the evidence next to each draft makes it feel trustworthy.
- **What broke / felt like a toy:** It's all sample data. Nothing is connected to our real tools yet, so it doesn't prove Juno finds the right signals or writes replies I'd actually send. The Slack buttons in the prototype are simulated.
- **What I'd change next pass:** Run it on last week's real feedback and compare Juno's drafts with the replies we actually sent, tracking how many I'd approve without edits. Also check that our help desk API can post replies and that every feedback has an event ID.
