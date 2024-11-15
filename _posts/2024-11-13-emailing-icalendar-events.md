---
layout: post
title:  "Emailing calendar invites using iCalendar"
date:   2024-11-13 10:00:00 +1:00
published: true
---

I was working on a system for booking online language classes.  When students book a class online, they get an email with a confirmation of the date,
time and how to connect to the class via Zoom, Google Meet etc.  

I wanted to send iCalendar attachments with the confirmation email so that if the student is
using a supported calendar client such as Google Calendar, Outlook or Apple Calendar, the appointment for the class is automatically added to their calendar.

I also wanted to be able to email updates (e.g. if the time of the class changes) and also cancellations.

As the most popular email address type with these students was Gmail, I initially focussed on getting this working with Google Calendar.

## The iCalendar format

The iCalendar format has lots of features, including support for recurring events, but for my use-case, the classes are just one-off and do not need this complexity.

Here's an example of an iCalendar event for a single event.

```
BEGIN:VCALENDAR
CALSCALE:GREGORIAN
METHOD:REQUEST
VERSION:2.0
PRODID:-//Simple ICalendar//Simple ICalendar//EN
BEGIN:VEVENT
UID:event-1234567@testcompany.com
DTSTAMP:20241106T100000Z
DTEND:20241106T203000Z
DTSTART:20241106T193000Z
ORGANIZER;CN=Test Company:mailto:donotreply@testcompany.com
STATUS:CONFIRMED
SEQUENCE:0
SUMMARY:German lesson with Wolfgang
LOCATION:https://us04web.zoom.us/j/foo
END:VEVENT
END:VCALENDAR
```

This is created as a text file (with extension `.ics`) and then sent as an attachment in the confirmation email.

The actual event details are enclosed within the `BEGIN:VEVENT` and `END:VEVENT` tags.

The important fields are:
* `METHOD` - This should be `REQUEST` when creating or updating an event, but `CANCEL` when sending a cancellation.
* `PRODID` - This can be set to what you want but is the name of the software generating the events.
* `UID` - A unique global ID for the event in the format of an email address.  This should be the same when sending updates for the same event.
   Keep to alphanumeric characters plus `-` and `_` to the left of the `@`.
* `DTSTAMP` - A date stamp for when the file was generated.  If an update or cancellation is sent this should be updated.
* `DTSTART`, `DTEND` - The start and end date of the event.  A time zone can be put here, but for simplicity I kept to UTC.
* `ORGANIZER` - This is in the format as shown, with the name (indicated by `CN`) and the email address with `mailto`.  This doesn't have to be the    
   attendee of the meeting, just the person sending the invites.  
   Note that the character after the `ORGANIZER` is a `;` and not a `:`.
* `STATUS` - This should be `CONFIRMED` for creating and updating or `CANCELLED` when sending a cancellation.
* `SEQUENCE` - This should start at `0` and for every subsequent update or cancellation should be incremented by `1`.
* `SUMMARY` - The summary that will appear in the event title.
* `LOCATION` - This optional field can either be a text, e.g. "Buckingham Palace", or a URL for a video call, in which case in Google Calendar it 
   will be clickable once added to the calendar.

## Attaching to an email

The iCalendar file should be sent as an email attachment with an `.ics` extension, e.g. `meeting.ics`.

The `Content-type` of the attachment should be set:
* For event creations and updates: `text/calendar; method=REQUEST charset="UTF-8"`
* For event cancellations: `text/calendar; method=CANCEL charset="UTF-8"`

Also the attachment should be sent inline and the `Content-id` set to `"cid:meeting.ics"`

## Appearance in Gmail

When an email with an iCalendar attachment is received, Gmail will add a panel to the top of the email like this:

![Booking confirmation gmail]({{ site.url }}/assets/icalendar/booking-confirmation-gmail.png){:style="border: 1px solid #ddd; width: 80%;"}

### Why the Yes/Maybe/No buttons?

Unfortunately the user **has** to click "Yes" or "Maybe" for the event to be added to their Google calendar.  If they do not respond it is not added. 
I don't believe there is a way to have it added automatically.

Unfortunately it's also not possible to attach some URL to the buttons so that if the user clicks "No" it would hit some endpoint on your
website to cancel the event.  You can put links in the `DESCRIPTION` field (see below) to do this e.g "To cancel please click here", but the
user might think clicking "No" is sufficient, but will just have the effect of not adding the event to the user's calendar.  You can configure it to send a RSVP response by email, but this requires automated processing email responses.

An alternative is to set the `METHOD:PUBLISH` rather than `METHOD:REQUEST` in the file, however in Gmail the panel in the email will look like this:

![Booking confirmation add to calendar]({{ site.url }}/assets/icalendar/booking-confirmation-gmail-add-to-calendar.png){:style="border: 1px solid #ddd; width: 80%;"}

In this case the user has to click "Add to calendar" link for it to be added.  Also events send with `METHOD:PUBLISH` are not tracked by Google Calendar, so if we send an update, it will just create a new duplicate event in the user's calendar.

It may also be that although `METHOD:REQUEST` was set, the "Add to calendar" still appears instead of "Yes/Maybe/No".  This can occur if Google 
Calendar thinks that you don't trust the sender sufficiently - e.g. you've not accepted invites before or not received emails from this address before.

In conclusion, using `METHOD:REQUEST` is the best option.

### Appearance in Google Calendar

If the user clicks "YES" in email panel, the event will appear in their Google calendar like this:

![Booking confirmation in Google calendar]({{ site.url }}/assets/icalendar/booking-confirmation-google-calendar.png){:style="border: 1px solid #ddd; width: 80%;"}

## Sending updates

By sending another mail to the user with another iCalendar attachment, the event can be automatically updated in their calendar without the user having to open the mail.

In addition to changing any fields relevant for the update, e.g. `LOCATION`, `DTSTART`/`DTEND`, the following fields need to be changed for an update:

* `UID` - This must be the same as in the original mail sent.
* `DTSTAMP` - This should be set with the current timestamp to allow the iCalendar client (e.g. Google Calendar) to recognise it is an update to 
    an existing event.
* `SEQUENCE` - This should be incremented for each update.

## Sending cancellations

The event can also be cancelled by sending a mail with an iCalendar attachment. This removes the event from the user's calendar without them having
to open the mail.

The following fields must also be changed:

* `METHOD` - Set to `CANCEL`
* `UID` - This must be the same as in the original mail sent.
* `DTSTAMP` - This should be set with the current timestamp.
* `SEQUENCE` - This should be incremented for each update.
* `STATUS` - Set to `CANCELLED`

N.B. The `METHOD` is `CANCEL` but the `STATUS` is `CANCELLED`.

# Setting the event description

An optional HTML description can be set to add the details to the event.  This could be the same as the email body sent for the invite,
but doesn't have to be.

In the screenshot above for showing how the event appears in Google Calendar, this is the text: *"Hi Mark, this is to confirm you lesson booking..."*

This can be set by using the `DESCRIPTION` attribute within the `VEVENT` tags.

However, it needs to:
1. Escape `,`, `:`, `;` and `\` with a preceding `\`.
2. Wrap the lines to be no more than 75 characters long (including the `DESCRIPTION:` tag, with the subsequent line starting with a space
   (plus up to 75 more characters).  Escape sequences cannot be split.
3. Trim any extra HTML whitespace otherwise on Google Calendar desktop (on Android it is fine), the whitespace is not compressed
   as it normally is with HTML, and new line characters are rendered as new lines like this: 

    ![Whitespace issue on Google calendar]({{ site.url }}/assets/icalendar/whitespace-issue-google-calendar.png){:style="border: 1px solid #ddd; width: 40%;"}

After applying escaping, wrapping and trimming the result should be something like this:

```
...
STATUS:CONFIRMED
SEQUENCE:1
SUMMARY:German lesson with Wolfgang
DESCRIPTION:<html><body>Hi Mark<br><br>This is to confirm your lesson booki
 ng for <strong>Friday 15/11/2024 19\:30</strong>. <br><br>If you need to <s
 trong>cancel or change the lesson </strong>please let us know before Thursd
 ay.
...
```

## Testing

In my case I wrote a lot of unit tests to verify that the format of the iCalendar file was correct and the escaping/wrapping/trimming of the
`DESCRIPTION` attribute was correct.

I also used an [online validator](https://icalendar.org/validator.html) to validate the files.  This is of limited use and this site contains
lots of ads.

However, to test on email clients, you have to really send emails which is not ideal.

## Next steps

The next step is to test on Outlook and Apple Calendars, where there will probably be some nuances.  Currently I have
the system configured to only sent to Google based email addresses. 

There's also a long-tail of other calendar clients, such as Yahoo, Zoho etc, but test on the main three
is probably enough to cover 80-90% of users.

## References

A lot of the information I got here was from looking at the content of iCalendar invites that I had received from people, asking ChatGPT (which
was not always correct of course) and from trial-and-error testing against Gmail.

I also looked the [iCalendar Elixir library](https://github.com/lpil/icalendar).  This is currently unmaintained, and missing support for some fields, but had some useful information.

